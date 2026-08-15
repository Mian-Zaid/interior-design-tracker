# Architecture Brief: Serverless Google-Sheets App (+ Drive uploads)

A reusable blueprint for building **static, backend-free web apps that use a Google Sheet
as their database**, with Google login, live data, and — new in this app — **image uploads
to the user's own Google Drive**. This repo's interior-design tracker is one implementation
of the pattern; hand this document to a new session to build a different app on it.

It is a direct descendant of [`expense-tracker`](https://github.com/Mian-Zaid/expense-tracker):
same auth, same parser strategy, same safe-write rules. The delta is Section
["Adding Drive file uploads"](#adding-drive-file-uploads).

## What it is

A **100% static single-page app** (`index.html`, inline CSS+JS, no build step, no server)
that uses a **Google Sheet as its database**, reads/writes it **directly from the browser**
via the Google Sheets API, authorized with **client-side Google OAuth**. Hosted free on
**GitHub Pages**.

## The core pattern (reuse this for any app)

```
Static page (GitHub Pages)
      │  Google Identity Services (GIS) → OAuth access token (in-memory)
      ├─────────────────────────────────────────────┐
      ▼                                             ▼
Google Sheets API v4 (gapi client)          Google Drive API v3 (fetch/XHR)
  values.get / update / batchUpdate           upload/files (multipart)
  values.clear                                files/{id}/permissions
      │                                             │
      ▼                                             ▼
 User's Google Sheet                          User's Google Drive
```

## Key technical decisions & why

1. **No backend at all.** Everything runs in the browser. The "server" is Google's own APIs.
2. **Auth = Google Identity Services token client**
   (`google.accounts.oauth2.initTokenClient`). NOT a share link — **a share link CANNOT
   authorize writes**; only an OAuth token can. This is the #1 thing people get wrong.
3. **Two Google scripts loaded:** `https://apis.google.com/js/api.js` (gapi, for the Sheets
   API) + `https://accounts.google.com/gsi/client` (GIS, for the token). Wait for both
   globals before init.
4. **gapi init** with `discoveryDocs:['https://sheets.googleapis.com/$discovery/rest?version=v4']`,
   then `gapi.client.setToken({access_token})` after each token grant. No API key needed
   when using OAuth. **Drive is not loaded into gapi** — see below.
5. **Token lifecycle:** tokens last ~1h, live only in memory (never localStorage). On load,
   try silent `requestAccessToken({prompt:''})`; on 401, silently re-request; if that fails,
   show a "Sign in" button (`prompt:'consent'`). Keep the raw `access_token` in a variable
   too — `fetch`/`XHR` calls to Drive need it as a bearer token.
6. **Config in localStorage:** OAuth Client ID, Spreadsheet ID (parsed from a full URL via
   regex `/\/spreadsheets\/d\/([a-zA-Z0-9-_]+)/`), tab name, plus app-specific settings.
7. **Live data:** optimistic UI update → API call → re-fetch to reconcile; plus **20s
   polling** (paused when `document.visibilityState !== 'visible'`, and refetch on tab
   focus). Also pause polling while a write modal is open or an upload is in flight, or the
   refresh will fight the optimistic state.
8. **Safe writes into human-maintained sheets:** never insert/delete whole rows (that shifts
   other content) and **never overwrite a non-empty cell** — re-read fresh, find an empty
   target row, verify it's blank, then `values.update` a specific A1 range. Delete = clear
   cells, not delete the row. This protects real user data.
9. **Guarded writes.** Before updating or clearing an existing row, re-read its anchor cell
   (the entry's identifying column) and abort if it no longer matches what the UI thought
   was there. Two devices editing the same sheet is the normal case, not the edge case.
10. **Derived values are never stored.** Progress percentages are recomputed client-side on
    every load and poll. Nothing to keep in sync, and no user formulas get clobbered.

## Sheets API calls used

| Purpose | Call |
|---------|------|
| Read all cells of a tab | `spreadsheets.values.get({range:"'TabName'", valueRenderOption:"FORMATTED_VALUE"})` |
| Read one guard cell | `spreadsheets.values.get({range:"'TabName'!A5"})` |
| Write a cell range | `spreadsheets.values.update({range, valueInputOption:"USER_ENTERED", resource:{values}})` |
| Write several ranges at once | `spreadsheets.values.batchUpdate({resource:{valueInputOption, data:[{range,values},…]}})` |
| Clear a region | `spreadsheets.values.clear({range, resource:{}})` |

Note there is no `values.append` and no `batchUpdate` with row/column requests anywhere —
both would shift a human's layout. Everything is a targeted `update` into verified-empty
cells.

## Handling messy, human-maintained sheets

Real sheets are rarely a clean flat table. Both apps in this family put **multiple
side-by-side tables on one tab**, each with a title cell above a header row, variable
lengths, and blank gaps. The parser **auto-detects each block** by scanning every cell for a
known header anchor, then reads the nearest non-empty cell above it as the block title.

- expense-tracker anchors on `Description` (→ `Description | Amount | Date`)
- this app anchors on `Title` (→ `Title | Description | Status | Image URL | Video URL`)

Two refinements worth carrying forward:

- **Confirm the anchor with a neighbour.** A single-cell match is fragile — a row whose data
  happens to read `Title` would spawn a phantom table. Require an adjacent header
  (`Description` or `Status`) before accepting a header row.
- **Blank rows are gaps, not terminators.** Deleting an entry clears its cells, so a table's
  data region legitimately contains holes. Walk to the end of the block (or until another
  header appears in the same column), not to the first empty row. Track `lastDataRow`
  separately so appends land after everything.

Design your parser to discover structure from anchors, not fixed cell coordinates.

## Adding Drive file uploads

The new capability in this app, and the part worth copying verbatim.

1. **Scope:** add `https://www.googleapis.com/auth/drive.file` alongside the Sheets scope,
   space-separated, in `initTokenClient`. This scope grants access **only to files the app
   itself creates** — so it is non-sensitive, needs no Google verification review, and the
   consent screen can stay in "Testing" forever. Do not reach for `auth/drive`.
2. **Enable the Drive API** in the Cloud project (Library → Google Drive API). Same project
   and same OAuth Client ID as Sheets; no new authorized origins needed.
3. **Don't load Drive into gapi.** The gapi discovery client can't do media uploads, and you
   want upload progress anyway. Call the REST endpoints directly with the raw access token:

   ```
   POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&fields=id,webViewLink
   Authorization: Bearer <token>
   Content-Type: multipart/related; boundary=<b>

   --<b>
   Content-Type: application/json; charset=UTF-8

   {"name":"photo.jpg","mimeType":"image/jpeg"}
   --<b>
   Content-Type: image/jpeg

   <binary>
   --<b>--
   ```

   Build the body as `new Blob([preamble, fileBlob, epilogue])` and send it with
   `XMLHttpRequest` so `xhr.upload.onprogress` can drive a progress bar. `fetch` has no
   upload progress.
4. **Make the file link-viewable, or the link only works for the uploader:**
   `POST /drive/v3/files/{id}/permissions` with `{"role":"reader","type":"anyone"}`.
   Treat a failure here as non-fatal but *warn loudly* — the file uploaded fine, it just
   won't render on other devices. And tell the user in-app what this permission means: link
   sharing is a real privacy trade-off, not an implementation detail.
5. **Store `https://drive.google.com/uc?id=FILE_ID`** in the sheet cell, but **render**
   `https://drive.google.com/thumbnail?id=FILE_ID&sz=w400` in the `<img>` — the thumbnail
   endpoint is the one that reliably hot-links. Parse the file ID back out of whatever URL
   form is in the cell (`/file/d/…`, `?id=…`, `/d/…`) so hand-pasted links work too.
6. **Resize before upload.** Draw to a canvas capped at ~1600px on the long edge and
   `toBlob(…, 'image/jpeg', 0.85)`. Phone photos are 4–8 MB; this app gets used on mobile
   data in a half-renovated room.
7. **Uploads are the one blocking write.** Every text field can be applied optimistically,
   but you cannot optimistically render a Drive URL that does not exist yet. Upload first,
   show a progress bar, and only then write the row.
8. **Files are never deleted.** Unlinking a photo, deleting a task, or cancelling the dialog
   leaves the file in Drive. Deleting user files automatically is worse than leaving strays —
   just document it.

## One-time Google Cloud setup (the only real config burden)

1. Cloud Console → new project → **enable Google Sheets API and Google Drive API**.
2. **OAuth consent screen** (External) → add both scopes → add yourself under **Test users**
   (no verification needed; the app can stay in "Testing").
3. **Credentials → OAuth client ID → Web application** → add **Authorized JavaScript
   origins** = the exact hosting origin, e.g. `https://<user>.github.io` (origin only, no
   path).
4. Paste the Client ID into the app's settings.

## Gotchas that cost time

- **Share link ≠ write access.** Must use OAuth.
- **Authorized JavaScript origins must match exactly** — GitHub Pages origin is
  `https://<user>.github.io`, NOT the `/repo/` path. Mismatch = sign-in fails.
- **Adding a scope to an existing Client ID doesn't retro-grant it.** Users already signed in
  keep their old token until it expires; they must re-consent. Expect "why does upload 403"
  reports right after you ship a new scope.
- **`drive.file` cannot see files it didn't create** — including files the same user uploaded
  through a different app. That's the point of the scope, but it means you can't "adopt" an
  existing Drive folder.
- **`drive.google.com/uc?id=…` is unreliable as an `<img src`** these days; `?/thumbnail` is
  the endpoint that works. Keep the two concerns separate: what you *store* vs. what you
  *render*.
- **Workspace accounts** (company Google accounts) often **block public link sharing** by
  admin policy — the `permissions.create(anyone)` call will fail. Personal Gmail accounts are
  fine.
- **`FORMATTED_VALUE`** returns display strings (numbers with commas, dates as text). Parse
  accordingly, or use `UNFORMATTED_VALUE` and convert date serials yourself.
- **Don't touch users' existing formulas** when writing; compute your own totals in-app.

## Recommended file layout

- `index.html` — entire app. Internal structure: helpers → parse → render → auth/API →
  Drive upload → modals → event handlers → polling → init.
- `README.md` — Google Cloud setup, sheet layout, hosting, troubleshooting.
- `ARCHITECTURE.md` — this file.
- No backend files.

## Testing a backend-free app

There's no server to integration-test against, but the write logic (A1 range arithmetic,
empty-row discovery, table placement) is exactly where bugs hide and is very cheap to cover:
drive the real `index.html` in headless Chromium, stub `window.gapi` / `window.google` with
an in-memory grid, intercept the Drive endpoints, and assert on the **ranges and values the
app tries to write**. That catches off-by-one column placement, gap handling, and clobbering
regressions without ever touching a real sheet.

## Reusable starting prompt for the next app

> Build a static single-page `index.html` (inline CSS+JS, no backend, no build) that uses a
> Google Sheet as its database via the Sheets API v4 with **client-side Google OAuth**
> (Google Identity Services token client, scopes `.../auth/spreadsheets` and — if you need
> file uploads — `.../auth/drive.file`). Load `api.js` + `gsi/client`. Store OAuth Client ID
> + Sheet ID + tab name in localStorage. Silent token on load, sign-in button on 401. Read
> the sheet, auto-detect side-by-side tables by scanning for a header anchor, render [MY UI],
> write back with optimistic updates + 20s polling paused when the tab is hidden. Writes must
> be safe: re-read fresh, only write into empty cells, never insert/delete rows, and guard
> every in-place update against the row having changed. Uploads go to Drive via multipart
> XHR, then `permissions.create(anyone, reader)`, storing the file URL in a cell. Mobile-first,
> light/dark, rounded cards. Host on GitHub Pages — remind me to add the Pages origin to
> Authorized JavaScript origins. Then write a README with the Google Cloud setup steps.
