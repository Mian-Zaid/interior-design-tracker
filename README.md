# Interior Design Tracker

Track home interior / renovation progress room by room — a plain checklist backed by
**your own Google Sheet**, with optional inspiration photos in **your own Google Drive**.
No server, no Apps Script, no build step. It's one self-contained `index.html`.

> First time here? Open the app, tap **⚙︎**, and follow the [Setup](#setup) section.

- 📱 Mobile-first, light **and** dark — follows your system, or tap ☀/🌙 in the header to
  override it (System / Light / Dark also lives in ⚙ settings)
- 🏠 **Rooms** hold **items**; each item is a checklist row with a status circle
- ⭕ Tap the circle to cycle **To Do → In Progress → Done**. One tap, no dialog, no photo needed
- 🏢 **Floors** — give a room any floor name you like (*Ground floor*, *Basement*, *Attic*…)
  and the home page groups and filters by it. Skip it and nothing changes
- 🗂️ **Board view** — every item across every room in three columns, filter by room,
  **drag** a card between columns or tap the **‹ ›** arrows
- 📊 Auto progress: one number for the whole project, a thin bar per room
  (`done ÷ total`), never written to the sheet
- 🖼️ **Optional** inspiration photos and/or video links per item — several of each, or
  none at all, now or any time later. Nothing ever requires media
- 🔍 Tap a photo for a **full-screen gallery** — swipe, arrow keys, or ‹ › to page through
- 🕒 Sorted **most recently changed first**; finished items sink to the bottom, capped at 10
  with **Show 10 more**
- ✎ Edit, 🗑 delete, 🔀 move items between rooms — destructive actions confirmed
- ➕ Create / edit / delete rooms — deleting never orphans items
- ⚡ Optimistic updates + 20-second polling to stay in sync across devices
- 👥 **Two people, two devices, one list** — share the sheet, send a setup link, each signs
  in with their own Google account
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
- The **floor** (optional) sits in the cell **immediately to its right**, on the same row.
- The **header row** is `Title | Description | Status | Image URL | Video URL | Updated`.
- Rows below are that room's items. Blank rows are fine — they're gaps left by deleted
  items, and the app keeps reading past them.
- **`Image URL` and `Video URL` can each hold more than one link** — separate them with line
  breaks (or spaces). The first is the item's cover / primary link.
- Tables are separated by one or more blank columns and can each be a different length.

Example (two rooms side by side, with a blank column between them):

| Kitchen | Ground floor | | | | | | Bedroom | First floor | | | | |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Title** | **Description** | **Status** | **Image URL** | **Video URL** | **Updated** | | **Title** | **Description** | **Status** | **Image URL** | **Video URL** | **Updated** |
| Install cabinets | Upper + lower units | Done | https://drive.google.com/uc?id=… | | 2026-08-11T10:00:00Z | | Paint walls | Matte white, 2 coats | Done | | | 2026-08-08T10:00:00Z |
| Tile the backsplash | Zellige, herringbone | In Progress | | https://youtu.be/… | 2026-08-14T10:00:00Z | | Wardrobe doors | Sliding, mirrored | In Progress | | | 2026-08-12T10:00:00Z |

The app **auto-detects every table** by finding each `Title` header (it also checks that
`Description` or `Status` sits next to it, so an item literally named "Title" doesn't create
a phantom room). You don't configure columns anywhere.

You don't have to build this by hand — connect an empty sheet and tap **＋ New** under
*Rooms*; the app writes the title cell and the full header row for you.

### Floors

Every room can carry a **floor** — a free-text label you type yourself. There is no fixed
list: *Ground floor*, *First floor*, *Basement*, *Attic*, *Guest annexe*, *Upstairs* are all
equally valid, and matching is case-insensitive so `ground floor` and `Ground Floor` group
together.

It lives in the cell **immediately right of the room name**, on the title row — inside the
table's own six columns, in a cell nothing else uses. Adding a floor therefore shifts nothing
and needs no new column. You can type it straight into the sheet if you prefer.

- Set it when you create a room, or any time later via **Edit room**.
- Leave it blank and the room is simply ungrouped — everything works exactly as before.
- On the home page, rooms are **grouped under floor headings** and a chip row lets you show
  one floor at a time. Rooms with no floor collect under **No floor**.
- **If no room has a floor, none of that chrome appears at all** — no chips, no headings. The
  feature is invisible until you use it.

Existing sheets need no migration: a room with an empty cell there is just a room with no
floor.

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
- **Edit room** just overwrites its title cell and the floor cell beside it; the items stay
  put beneath it.
- **Delete room** never deletes items silently. If the room has items you must first either
  **move all of them** to another room or **delete them** along with the room — each behind
  a confirmation dialog. Only then is the table cleared, in its own columns only.

---

## Using it

### Home

Deliberately quiet. At the top, one number — the whole project's progress — over a hairline
bar. Below it, **Rooms** with a **＋ New**, then the room tiles: the room's most recent photo,
a thin progress bar, and its name with `done of total` underneath. Rooms with no photos show
an empty well — that's normal, not an error.

- **Tap a tile** to open that room.
- **Tap the big number** to open the board.
- Once any room has a floor, a chip row appears: **All**, then one chip per floor, plus
  **No floor** if some rooms are unassigned. Under **All** the tiles are grouped beneath
  uppercase floor headings; pick a single floor and you get a plain grid of just those rooms.
  Your choice is remembered on the device.

### Room checklist

- **Tap the circle** to advance the status. That's the whole interaction — no dialog, no
  photo required, nothing to confirm.
- **Tap the name** to edit the item.
- The right edge shows the item's photo or video if it has one, or a quiet **＋** to add one.
  **Tap a photo** to see it full-screen; **tap a video** to open it in that app.
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

Tap **🖼 Photos** or **📷 Camera** in the item popup, or the quiet **＋** on a checklist row.
The photo picker takes **several files at once**, and every item can hold up to **20**.

1. Each image is resized in the browser to **1600px on the long edge** and re-encoded as
   JPEG, so a 6 MB phone photo uploads as a few hundred KB over mobile data.
2. It's uploaded to **your** Google Drive (`drive.file` scope, so the app can only ever see
   files it created itself).
3. The app then marks the file **"anyone with the link can view"** and stores
   `https://drive.google.com/uc?id=FILE_ID` in the Image URL cell.
4. The row shows a thumbnail with a small **count badge** when there's more than one.

### Several photos on one item

Pick as many as you like in one go — they upload **one after another**, each appearing in the
strip as it lands, with one progress bar across the whole batch. If one fails, the ones
already uploaded stay.

In the strip, every photo has a **✕** to remove it and a **★** to make it the cover. The
**first photo is the cover** — it's the one shown on the checklist row, the board card, and
the room tile on the home page. Tap any thumbnail to view it full-screen.

They all live in the item's **one `Image URL` cell**, separated by line breaks. No extra
column, no migration: a cell holding a single URL is simply an item with one photo, and one
you type by hand with URLs separated by spaces or newlines works too.

> ⚠️ **Photos are shared by link.** Step 3 is what makes a photo load on your other devices
> — without it the link only opens for whoever uploaded it. The trade-off is that **anyone
> who gets that URL can view that photo.** The links are long and unguessable, which is fine
> for a home renovation project, but don't upload anything private. The same warning is
> shown in ⚙ settings.

Two more things worth knowing:

- Photo upload is the one **blocking** write — Save is disabled while it runs, because
  there's no URL to write until the file exists. With a batch, that's until the last one
  finishes.
- **Removing a photo, or deleting an item or room, does not delete the file from Drive.**
  It only unlinks it. Tidy up in Drive if you want the file gone. Likewise, if you upload a
  photo and then hit Cancel, the file stays in your Drive unreferenced.

### Tapping media

- **Photos open full-screen in the app.** A larger rendition is fetched for the viewer, so it
  isn't the 46px thumbnail blown up. Tap the backdrop, press Escape or hit ✕ to close;
  **Open original ↗** goes to the file in Drive (and follows whichever photo you're on). If
  the image can't load you get a message and the link, not a broken icon.
- **With several photos the viewer is a gallery**: **swipe** left/right, tap the **‹ ›**
  buttons, press the **arrow keys**, or tap a dot to jump. It wraps at both ends, and a
  counter shows where you are. With a single photo none of that chrome appears.
- **Videos open in their own app.** The tile is a real link, so tapping it hands off to the
  Instagram / YouTube / TikTok app when it's installed, and falls back to the browser when
  it isn't. This is your phone's own Universal Links (iOS) / App Links (Android) doing the
  work — the app is never forced open, and on a desktop it just opens a tab.
  With several links the tile opens the **first** one and carries a count badge; the rest are
  in the item's **Edit** screen, each with its own **Open ↗**.

If an item has both a photo and a video, the tile shows the photo with a ▶ badge; tapping it
views the photo, and the videos are one tap further in via the item's **Edit** screen.

### Video links & previews

Tap **🔗 Video link** in the item popup, paste a link, and a preview appears under the field
— the same way a photo preview does. **＋ Another link** adds a row, so one item can hold up
to **10** links: the Reel that gave you the idea, the how-to video, the supplier's clip. Each
row has its own preview and its own **✕**.

They all live in the item's **one `Video URL` cell**, separated by line breaks — same as
photos, so there's no new column and nothing to migrate.

What each preview looks like depends on the platform:

| Link | Preview |
|---|---|
| **Instagram** Reel / post / IGTV | Embedded inline (portrait), public posts only |
| **YouTube** (`watch?v=`, `youtu.be`, `/shorts/`, `/embed`) | Thumbnail; tap ▶ to play in place |
| **Google Drive** video | Thumbnail; tap ▶ to play in place |
| **TikTok**, **Vimeo** | Embedded inline |
| Anything else | A card with the hostname and an **Open ↗** link |

In the checklist, an item with a video shows a tinted play tile — Instagram's gradient,
YouTube's red — so you can tell at a glance what kind of reference it is, with a small count
badge when there's more than one. If an item has both a photo and a video, the photo wins and
gets a ▶ badge.

**Several links don't load several embeds.** A poster image is one cheap request, so it
always shows; a platform that can only be embedded (Instagram, TikTok, Vimeo) gets a quiet
**▶ Show preview** strip instead, and loads its frame when you ask for it. With a single link
nothing changes — it previews straight away, as it always did.

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

## Sharing with a second person

Two (or more) people can use the same list from their own phones and their own Google
accounts. There's no shared password and no second copy of the data — everyone reads and
writes the same sheet, and the 20-second polling means a change shows up on the other device
within about half a minute.

**Three things have to be true.** Do them once:

### 1. Give them edit access to the Sheet

Open the Sheet in Google Sheets → **Share** → add their Google account as **Editor**.
This is what actually authorises their writes. Without it they get "Access denied (403)".

### 2. Add them as a Test user on the OAuth consent screen

[Google Auth Platform → Audience](https://console.cloud.google.com/auth/audience) → **Test
users** → **Add users** → their Google address.

This one is easy to miss. The consent screen is in **Testing** mode, which allows up to 100
test users; anyone not on that list gets *"Access blocked: this app has not completed
verification"* no matter how the sheet is shared.

> **Why not publish the app instead?** The `.../auth/spreadsheets` scope is classed as
> **sensitive** by Google, so leaving Testing would mean going through app verification. For
> a household, adding a test user is the right answer.

### 3. Send them a setup link

In ⚙ settings on the device that already works, use **Use this on another device** →
**Copy**, and send them the link however you like. Opening it on their phone fills in the
Client ID, Sheet ID and tab name — no typing a 72-character Client ID on a phone keyboard —
and prompts them to sign in with **their own** Google account.

The link contains only those three settings, **never a sign-in token**. None of them are
secrets: the Client ID is public by design, and the Sheet ID is useless to anyone you haven't
given access in step 1.

### What to expect once you're both in

- Everyone sees the same rooms, items and statuses.
- **Photos work across accounts.** They upload to the uploader's own Drive and are marked
  link-viewable so others can see them. The flip side: if the uploader deletes the file from
  their Drive, it disappears for everyone.
- **Simultaneous edits are safe.** Every write re-checks the row first, so if you both change
  the same item at once the slower one is refused with "Row changed — refresh and try again"
  rather than silently overwriting.
- Sign-in and other settings are per-device.

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
4. Tap **＋ New** beside *Rooms* to create your first room. The **Floor** field is optional.

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
  neighbouring `Description`/`Status` cell. Tap **＋ New** beside *Rooms* and let the app
  write one.
- **Only one of my photos shows on the row:** that's by design — the row, the board card and
  the room tile show the **cover** (the first one) with a badge counting the rest. Tap it for
  the full gallery.
- **Tapping a video tile always opens the same link:** it opens the first one. The tile has to
  stay a real link for the app handoff to work, and a link can only point at one place — open
  the item to reach the others.
- **A photo or video I typed into the sheet by hand is ignored:** each link must start with
  `http://` or `https://`, and links must be separated by a line break or a space. Anything
  else in those cells is skipped.
- **"Video links must start with http:// or https://":** one of the rows has something that
  isn't a full URL — the message names it. Fix or ✕ that row and save again.
- **No floor chips on the home page:** no room has a floor yet. Open a room → **Edit room**
  and type one, and the chips appear. Floor names are matched ignoring case and extra spaces,
  so `ground floor` and `Ground  Floor` land in the same group — the spelling of whichever
  room the app reads first is the one shown on the chip.
- **"Could not create room" / "exceeds grid limits":** a Google Sheet tab has a fixed grid —
  a new one is 1000 rows × 26 columns — and each room takes 6 columns plus a gap, so the 4th
  room needs column 27. The app now widens the tab automatically (appending columns at the
  far right, which shifts nothing). If you're on an older build, add columns to the tab by
  hand and it'll work.
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
- **The other person gets "Access blocked: app has not completed verification":** they're
  not on the **Test users** list — see [Sharing with a second person](#sharing-with-a-second-person).
- **The other person gets 403 but can sign in:** they have consent but not **Editor** access
  to the Sheet itself. Both are needed.
- **Privacy:** the app runs entirely in your browser and only ever contacts Google's own
  APIs. Your Client ID, Sheet ID, and tab name live in `localStorage`; access tokens live
  only in memory and expire automatically.
