# Interior Design Tracker

Track home interior / renovation progress room by room — a plain checklist backed by
**your own Google Sheet**, with optional inspiration photos in **your own Google Drive**.
No server, no Apps Script, no build step. It's one self-contained `index.html`.

> First time here? Open the app, tap **⚙︎**, and follow the [Setup](#setup) section.

- 📱 Mobile-first, light **and** dark — follows your system, or tap ☀/🌙 in the header to
  override it (System / Light / Dark also lives in ⚙ settings)
- 🏠 **Rooms** hold **items**; each item is a checklist row with a status circle
- ⭕ Tap the circle to cycle **To Do → In Progress → Done**. One tap, no dialog, no photo needed
- 🗂️ **Board view** — every item across every room in three columns, filter by room,
  **drag** a card between columns or tap the **‹ ›** arrows
- 📊 Auto progress: a ring per room, a bar overall (`done ÷ total`), never written to the sheet
- 🖼️ **Optional** inspiration image and/or video link per item — add either, both, or neither,
  now or any time later. Nothing ever requires media
- 🕒 Sorted **most recently changed first**; finished items sink to the bottom, capped at 10
  with **Show 10 more**
- ✎ Edit, 🗑 delete, 🔀 move items between rooms — destructive actions confirmed
- ➕ Create / rename / delete rooms — deleting never orphans items
- ⚡ Optimistic updates + 20-second polling to stay in sync across devices
- 🔐 Signs in with **your** Google account — data never leaves your Sheet and Drive
- 🔓 **Stays signed in.** Reloading the page never asks again; the session is renewed
  silently in the background, with a **Sign out** in ⚙ settings

---

## Why sign-in (and not just a share link)?

Google **does not allow writing to a Sheet from JavaScript using only a share link** —
every write must be authorized. So instead of a backend, this app uses Google's browser
OAuth: you click **Sign in with Google** once, and the app gets permission to edit the
sheet and to create files in your Drive on your behalf. The whole thing stays a static
page you can host on GitHub Pages.

You need two things (both free, one-time):

1. A **Google OAuth Client ID** (created in Google Cloud, ~5 min).
2. A **Google Sheet** you have edit access to.

> There is **no API key** anywhere in this app. API keys cannot authorize writes to a
> private sheet — only an OAuth token can. If you went looking for an "API key" field, you
> want the **OAuth Client ID** instead.

### Scopes the app asks for

| Scope | Why |
|-------|-----|
| `.../auth/spreadsheets` | Read and write your tracker sheet |
| `.../auth/drive.file` | Upload inspiration photos. This scope **only** grants access to files the app itself creates — it cannot see the rest of your Drive, and it needs no Google verification review |

---

## Sheet layout

The app reads a **single tab** (default name `Interior`) that holds **one or more room
tables laid out side by side**. Each table is one **room**:

- The **room name** sits in a cell above the table's header row, in the Title column.
- The **header row** is `Title | Description | Status | Image URL | Video URL | Updated`.
- Rows below are that room's items. Blank rows are fine — they're gaps left by deleted
  items, and the app keeps reading past them.
- Tables are separated by one or more blank columns and can each be a different length.

Example (two rooms side by side, with a blank column between them):

| Kitchen | | | | | | | Bedroom | | | | | |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Title** | **Description** | **Status** | **Image URL** | **Video URL** | **Updated** | | **Title** | **Description** | **Status** | **Image URL** | **Video URL** | **Updated** |
| Install cabinets | Upper + lower units | Done | https://drive.google.com/uc?id=… | | 2026-08-11T10:00:00Z | | Paint walls | Matte white, 2 coats | Done | | | 2026-08-08T10:00:00Z |
| Tile the backsplash | Zellige, herringbone | In Progress | | https://youtu.be/… | 2026-08-14T10:00:00Z | | Wardrobe doors | Sliding, mirrored | In Progress | | | 2026-08-12T10:00:00Z |

The app **auto-detects every table** by finding each `Title` header (it also checks that
`Description` or `Status` sits next to it, so an item literally named "Title" doesn't create
a phantom room). You don't configure columns anywhere.

You don't have to build this by hand — connect an empty sheet and tap **＋ New Room**; the
app writes the title cell and the full header row for you.

### The `Updated` column

Holds an ISO-8601 UTC timestamp (`2026-08-14T10:00:00Z`), rewritten every time an item
changes. It's what makes "most recently changed first" mean anything — sheet order is
*creation* order, which would bury an item you finished today under one you added last
month.

**Upgrading an older 5-column sheet:** on first load the app adds the `Updated` header for
you, but **only if the whole column beside that table is empty**. If something is already
there it leaves your data alone and says so in a banner; that room then falls back to sheet
order. You can also just type `Updated` into the cell yourself.

### Status values

The Status column is read leniently, so you can type in the sheet directly:

| Written in the sheet | Read as |
|---|---|
| `Done`, `Complete`, `Completed`, `Finished` | **Done** |
| `In Progress`, `Doing`, `WIP`, `Ongoing`, `Started` | **In Progress** |
| `To Do`, anything else, **or blank** | **To Do** |

The app always writes back the canonical `To Do` / `In Progress` / `Done`.

### Progress

Per-room progress is `done ÷ total items` (In Progress counts as not done). Overall
progress is every done item ÷ every item. Both are recalculated in the browser on each
load and poll — **nothing is written to the sheet**, so no formulas of yours get touched.

### How adding & deleting works (safe by design)

- **Add** writes the new item into the first fully-empty row under the room's existing
  items. It **never inserts or deletes whole rows** (that would shift the neighbouring
  tables) and **never overwrites a non-empty cell**.
- **Delete** just clears that item's cells, leaving a blank gap — no row shifting.
- **Move** (changing an item's room) writes it into the destination table first, then
  clears the old cells.
- **Every write is guarded**: the app re-reads the item's Title cell and aborts if someone
  changed that row from another device in the meantime ("Row changed — refresh and try
  again").
- **All writes are `RAW`**, so an item named `=1+1` is stored as text and never becomes a
  formula.
- **Create room** adds a new table in fresh columns to the right of everything already in
  the sheet — nothing shifts.
- **Rename room** just overwrites its title cell; the items stay put beneath it.
- **Delete room** never deletes items silently. If the room has items you must first either
  **move all of them** to another room or **delete them** along with the room — each behind
  a confirmation dialog. Only then is the table cleared, in its own columns only.

---

## Using it

### Home

Room tiles, each showing a progress ring and its most recent photo as the cover (rooms with
no photos show a tinted tile — that's normal, not an error). Tap a tile to open that room.
Tap the **big progress card** to open the board.

### Room checklist

- **Tap the circle** to advance the status. That's the whole interaction — no dialog, no
  photo required, nothing to confirm.
- **Tap the name** to edit the item.
- The right edge shows the item's photo or video if it has one, or a quiet **＋** to add one.
- Active items sit on top, most recently changed first. Finished items sink below, newest
  first, **10 at a time** with **Show 10 more**.

### Board

- Three columns matching the three statuses, colour-matched to the circles.
- **Filter chips** narrow to one room. Under "All rooms" each card shows which room it's in.
- **Drag a card** between columns, or tap the **‹ ›** arrows on the card. The arrows are
  there on purpose: drag is a hidden gesture with no affordance, and the arrows work with a
  keyboard and a screen reader.
- The Done column is capped at 10 with its own **Show more**.

Home, room and board are real URLs (`#/`, `#/room/Kitchen`, `#/board`), so the browser back
button and Android's back gesture work as expected.

---

## Photos & video (both optional)

Nothing in this app ever requires media. An item with no picture is a complete, normal item.

Tap **🖼 Photo** or **📷 Camera** in the item popup, or the quiet **＋** on a checklist row.

1. The image is resized in the browser to **1600px on the long edge** and re-encoded as
   JPEG, so a 6 MB phone photo uploads as a few hundred KB over mobile data.
2. It's uploaded to **your** Google Drive (`drive.file` scope, so the app can only ever see
   files it created itself).
3. The app then marks the file **"anyone with the link can view"** and stores
   `https://drive.google.com/uc?id=FILE_ID` in the Image URL cell.
4. The row shows a thumbnail; tapping it opens the full image in a new tab.

> ⚠️ **Photos are shared by link.** Step 3 is what makes a photo load on your other devices
> — without it the link only opens for whoever uploaded it. The trade-off is that **anyone
> who gets that URL can view that photo.** The links are long and unguessable, which is fine
> for a home renovation project, but don't upload anything private. The same warning is
> shown in ⚙ settings.

Two more things worth knowing:

- Photo upload is the one **blocking** write — Save is disabled while it runs, because
  there's no URL to write until the file exists.
- **Removing a photo, or deleting an item or room, does not delete the file from Drive.**
  It only unlinks it. Tidy up in Drive if you want the file gone. Likewise, if you upload a
  photo and then hit Cancel, the file stays in your Drive unreferenced.

### Video links & previews

Paste a link into **Video link** and a preview appears under the field, the same way a photo
preview does. What you get depends on the platform:

| Link | Preview |
|---|---|
| **Instagram** Reel / post / IGTV | Embedded inline (portrait), public posts only |
| **YouTube** (`watch?v=`, `youtu.be`, `/shorts/`, `/embed`) | Thumbnail; tap ▶ to play in place |
| **Google Drive** video | Thumbnail; tap ▶ to play in place |
| **TikTok**, **Vimeo** | Embedded inline |
| Anything else | A card with the hostname and an **Open ↗** link |

In the checklist, an item with a video shows a tinted play tile — Instagram's gradient,
YouTube's red — so you can tell at a glance what kind of reference it is. If an item has
both a photo and a video, the photo wins and gets a small ▶ badge.

**Why Instagram embeds rather than showing a thumbnail:** Instagram's oEmbed API has required
a Facebook app token since 2020, so there is no way for a page with no server to fetch a
Reel's poster image. The `/embed` endpoint needs no token, so that's what's used. Two
consequences worth knowing:

- **It only works for public posts.** A private or deleted Reel renders blank. The **Open ↗**
  link beside every preview always works, so nothing is lost.
- **It loads a frame from Instagram**, which means their cookies and scripts. If you'd rather
  not, the link still works fine without opening the item.

Nothing ever autoplays.

---

## Light & dark

The app follows your phone's light/dark setting out of the box. Tap the **sun / moon button**
in the header to override it — the icon shows what you'll switch *to*. Your choice is
remembered on that device and applied before the page paints, so there's no white flash on a
dark phone.

⚙ Settings has the full control: **System / Light / Dark**. Picking **System** clears the
override and goes back to following the phone, including live if you change it while the app
is open.

---

## Staying signed in

Once you've signed in, **reloading the page doesn't ask again.** The app keeps your access
token on the device and reuses it, renewing it quietly in the background while the app is
open. You'll only see the Google screen again when the token can't be renewed — typically
after a long gap, or if you signed out of Google.

There's a **Sign out** link in ⚙ settings that clears it from this device.

**Why not forever?** Google's browser sign-in issues *access tokens* — about an hour each,
with no refresh token. Refresh tokens require a client secret, which a page with no server
can't keep. So the app extends your session as far as a static page can, but a truly
permanent login would need a backend.

> **Worth knowing before you deploy:** the token is stored in this browser's `localStorage`,
> which on GitHub Pages is shared across **every** site under `https://<your-username>.github.io`.
> Your own apps sharing it is fine. If you ever host something you don't control there, that
> page could read this token — it's scoped to Sheets and `drive.file` and expires within the
> hour, but it's a real consideration. A custom domain, or a private repo you fully control,
> avoids it entirely.

---

## Setup

### 1. Create a Google Cloud project + OAuth Client ID

1. Go to the [Google Cloud Console](https://console.cloud.google.com/) and create (or pick)
   a project.
2. **Enable two APIs:** APIs & Services → **Library** → enable
   [*Google Sheets API*](https://console.cloud.google.com/apis/library/sheets.googleapis.com)
   **and** [*Google Drive API*](https://console.cloud.google.com/apis/library/drive.googleapis.com).
   Drive is the one people forget — without it everything works except photo upload, which
   fails with a 403.
3. **Configure the consent screen** — now called **Google Auth Platform** in the console
   (older consoles: APIs & Services → OAuth consent screen):
   - **Branding**: app name, user support email, developer contact.
   - **Audience**: User type **External**, publishing status **Testing**, and add your
     Google account under **Test users**.
   - **Data Access**: add both scopes —
     `https://www.googleapis.com/auth/spreadsheets` and
     `https://www.googleapis.com/auth/drive.file`.
4. **Create the Client ID:** **Clients** → **Create client**.
   - Application type: **Web application**.
   - Under **Authorized JavaScript origins**, add the origin(s) you'll open the app from:
     - `http://localhost:8000` (if testing locally)
     - `https://<your-username>.github.io` (for GitHub Pages)
   - Leave **Authorized redirect URIs** empty — this flow doesn't use them.
   - Create, then copy the **Client ID** (ends in `.apps.googleusercontent.com`).

> **Important:** the origin must match exactly where the page is served from. GitHub Pages
> origin is `https://<username>.github.io` — origin only, **no path, no trailing slash**.

**Reusing an existing Client ID** (e.g. from another Sheets app): enable the Drive API, add
the `drive.file` scope, then **sign out and back in**. Adding a scope doesn't retroactively
grant it — your existing token is still Sheets-only, so photo upload will 403 until you
re-consent.

### 2. Prepare your Sheet

1. Create/open a Google Sheet you can edit.
2. Rename a tab to `Interior` (or anything — you'll type the name in settings).
3. Leave it empty; the app creates the first room table for you.
4. Copy the sheet URL or ID.

### 3. Connect the app

1. Open the app, tap **⚙︎**.
2. Paste your **OAuth Client ID**, your **Sheet link/ID**, the **Tab name** (default
   `Interior`), and **Save**.
3. Click **Sign in with Google** and allow access to Sheets and Drive. Accept the
   "Google hasn't verified this app" screen — that's expected in Testing mode.
4. Tap **＋ New Room** to create your first room.

---

## Hosting on GitHub Pages

It's a single static file, so hosting is free:

1. Push this repo to GitHub.
2. **Settings → Pages → Build and deployment → Source:** *Deploy from a branch*.
3. Branch **`main`**, folder **`/ (root)`** → **Save**.
4. Your app goes live at `https://<your-username>.github.io/<repo-name>/`.
5. Make sure that **origin** (`https://<your-username>.github.io`) is listed under your
   OAuth client's **Authorized JavaScript origins**.

Settings are stored per-browser, so enter them once on each device.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — inline HTML, CSS, and JS |
| `README.md` | This file |
| `ARCHITECTURE.md` | The reusable "Google Sheet as a database" blueprint |

---

## Notes & troubleshooting

- **"Access denied (403)":** confirm your account has **edit** access to the sheet, and that
  **both** the Sheets API and the Drive API are enabled in your Cloud project.
- **Photo upload 403s but everything else works:** the Drive API isn't enabled, or
  `drive.file` wasn't added to the consent screen — or you added it and haven't signed out
  and back in yet.
- **Photos look broken on another device:** the "anyone with the link" permission didn't
  apply (the app warns when this happens). Share the file by link manually in Drive, or
  re-upload.
- **Sign-in popup blocked / origin error:** the exact origin you're serving from must be in
  **Authorized JavaScript origins**. For GitHub Pages that's `https://<username>.github.io`
  — not the `.../repo/` path.
- **"Sheet or tab not found (404)":** re-check the Sheet link/ID and Tab name in ⚙ settings.
- **"No room tables found":** the tab has no `Title` header row, or the header is missing its
  neighbouring `Description`/`Status` cell. Tap **＋ New Room** and let the app write one.
- **"Row changed — refresh and try again":** someone (or you, on another device) edited that
  item's row between your last sync and your write. The app refuses to clobber it; the next
  poll pulls in their change and you can redo yours.
- **Items sort oddly:** they're missing timestamps. Rows written before the `Updated` column
  existed sort by sheet position until you next touch them.
- **It asks me to sign in on every reload:** that's the bug this app fixes — make sure
  you're on a build that includes the "Staying signed in" behaviour above. If it persists,
  your browser is likely clearing site data on close, or blocking storage for the origin.
- **Signed in on one device but not another:** expected. Sign-in is per-browser, like the
  settings.
- **Privacy:** the app runs entirely in your browser and only ever contacts Google's own
  APIs. Your Client ID, Sheet ID, and tab name live in `localStorage`; access tokens live
  only in memory and expire automatically.
