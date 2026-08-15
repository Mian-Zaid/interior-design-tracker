# Interior Design Tracker

Track home interior / renovation progress room by room — reads and writes **your own
Google Sheet** directly from the browser, and uploads task photos to **your own Google
Drive**. No server, no Apps Script, no build step. It's one self-contained `index.html`.

> First time here? Open the app, tap **⚙︎**, and follow the [Setup](#setup) section to
> connect your Google Sheet.

- 📱 Mobile-first, light **and** dark mode (follows your system)
- 🏠 **Rooms** hold **tasks** — each task has a title, description, status, photo, video link
- 🟢 3-state status: **To Do → In Progress → Done** — tap the chip to cycle it
- 📊 Auto **progress bars**: per room and overall (`done ÷ total`), never stored in the sheet
- 📷 **Add a photo** from your gallery or camera — resized, uploaded to your Drive, thumbnail
  on the task card, tap to open full size
- ▶ **Video links** — paste any URL; YouTube links get a thumbnail and a "Watch" button
- ✎ Edit, 🗑 delete, and 🔀 move tasks between rooms — destructive actions are confirmed
- ➕ Create / rename / delete rooms — deleting never orphans tasks
- ⚡ Optimistic updates + 20-second polling to stay in sync across devices
- 🔐 Signs in with **your** Google account — data never leaves your Sheet and Drive
- 💾 Client ID, Sheet ID, and tab name saved in your browser (`localStorage`)

---

## Why sign-in (and not just a share link)?

Google **does not allow writing to a Sheet from JavaScript using only a share link** —
every write must be authorized. So instead of a backend, this app uses Google's browser
OAuth: you click **Sign in with Google** once, and the app gets permission to edit the
sheet and to create files in your Drive on your behalf. The whole thing stays a static
page you can host on GitHub Pages.

You need two things (both free, one-time):

1. A **Google OAuth Client ID** (created in Google Cloud, ~3 min).
2. A **Google Sheet** you have edit access to.

### Scopes the app asks for

| Scope | Why |
|-------|-----|
| `.../auth/spreadsheets` | Read and write your tracker sheet |
| `.../auth/drive.file` | Upload task photos. This scope **only** grants access to files the app itself creates — it cannot see the rest of your Drive, and it needs no Google verification review |

---

## Sheet layout

The app reads a **single tab** (default name `Interior`) that holds **one or more room
tables laid out side by side**. Each table is one **room**:

- The **room name** sits in a cell above the table's header row, in the Title column.
- The **header row** is `Title | Description | Status | Image URL | Video URL` (in that order).
- Rows below are that room's tasks. Blank rows are fine — they're gaps left by deleted
  tasks, and the app keeps reading past them.
- Tables are separated by one or more blank columns and can each be a different length.

Example (two rooms side by side, with a blank column between them):

| Kitchen | | | | | | Bedroom | | | | |
|---|---|---|---|---|---|---|---|---|---|---|
| **Title** | **Description** | **Status** | **Image URL** | **Video URL** | | **Title** | **Description** | **Status** | **Image URL** | **Video URL** |
| Install cabinets | Upper + lower units | Done | https://drive.google.com/uc?id=… | | | Paint walls | Matte white, 2 coats | In Progress | | https://youtu.be/… |
| Tile backsplash | Zellige, herringbone | In Progress | | | | Wardrobe doors | Sliding, mirrored | To Do | | |

The app **auto-detects every table** by finding each `Title` header (it also checks that
`Description` or `Status` sits next to it, so a task literally named "Title" doesn't create
a phantom room). You don't configure columns anywhere.

You don't have to build this by hand — connect an empty sheet and tap **＋ New Room**; the
app writes the title cell and header row for you.

### Status values

The Status column is read leniently, so you can type in the sheet directly:

| Written in the sheet | Read as |
|---|---|
| `Done`, `Complete`, `Completed`, `Finished` | **Done** |
| `In Progress`, `Doing`, `WIP`, `Ongoing`, `Started` | **In Progress** |
| `To Do`, anything else, **or blank** | **To Do** |

The app always writes back the canonical `To Do` / `In Progress` / `Done`.

### Progress

Per-room progress is `done ÷ total tasks` (In Progress counts as not done). Overall
progress is every done task ÷ every task. Both are recalculated in the browser on each
load and poll — **nothing is written to the sheet**, so no formulas of yours get touched.

### How adding & deleting works (safe by design)

- **Add** writes the new task into the first fully-empty row under the room's existing
  tasks. It **never inserts or deletes whole rows** (that would shift the neighbouring
  tables) and **never overwrites a non-empty cell**.
- **Delete** just clears that task's five cells, leaving a blank gap — no row shifting.
- **Move** (changing a task's room in the edit popup) writes it into the destination table
  first, then clears the old cells.
- **Every write is guarded**: the app re-reads the task's Title cell and aborts if someone
  changed that row from another device in the meantime ("Row changed — refresh and try
  again").
- **Create room** adds a new `title + Title | Description | Status | Image URL | Video URL`
  table in fresh columns to the right of everything already in the sheet — nothing shifts.
- **Rename room** just overwrites its title cell; the tasks stay put beneath it.
- **Delete room** never deletes tasks silently. If the room has tasks you must first either
  **move all of them** to another room (existing or newly created) or **delete them** along
  with the room — each behind a confirmation dialog. Only then is the table cleared, in its
  own columns only.

---

## Photos

Tap **🖼 Add Photo** (gallery / file picker) or **📷 Camera** in the task popup.

1. The image is resized in the browser to **1600px on the long edge** and re-encoded as
   JPEG, so a 6 MB phone photo uploads as a few hundred KB over mobile data.
2. It's uploaded to **your** Google Drive (`drive.file` scope, so the app can only ever see
   files it created itself).
3. The app then marks the file **"anyone with the link can view"** and stores
   `https://drive.google.com/uc?id=FILE_ID` in the task's Image URL cell.
4. The task card shows a thumbnail; tapping it opens the full image in a new tab.

> ⚠️ **Photos are shared by link.** Step 3 is what makes an uploaded photo load on your
> other devices (and for anyone else using the same sheet) — without it the link only
> opens for whoever uploaded it. The trade-off is that **anyone who gets that URL can view
> that photo.** The links are long and unguessable, which is fine for a home renovation
> project, but don't upload anything private. The same warning is shown in ⚙ settings.

Two more things worth knowing:

- Photo upload is the one **blocking** write — the task row isn't added until the file
  exists in Drive, because there's no URL to write optimistically.
- **Removing a photo from a task, or deleting a task or room, does not delete the file from
  Drive.** It only unlinks it. Tidy up in Drive itself if you want the file gone. Likewise,
  if you upload a photo and then hit Cancel, the file stays in your Drive unreferenced.

## Video links

Paste any URL into **Video link** (YouTube, a Drive video, Vimeo, anything). It renders as
a **▶ Watch** button on the task card. YouTube URLs (`watch?v=`, `youtu.be/`, `/shorts/`,
`/embed/`) additionally get a small thumbnail from `img.youtube.com`. Nothing is embedded
or auto-played.

---

## Setup

### 1. Create a Google Cloud project + OAuth Client ID

1. Go to the [Google Cloud Console](https://console.cloud.google.com/) and create (or pick)
   a project.
2. **Enable two APIs:** APIs & Services → **Library** → enable *Google Sheets API* **and**
   *Google Drive API*.
3. **Configure the consent screen:** APIs & Services → **OAuth consent screen**.
   - User type: **External** → Create.
   - Fill the required app name / email fields.
   - Under **Scopes**, add `.../auth/spreadsheets` and `.../auth/drive.file`.
   - Add your Google account under **Test users** (this lets you use the app while it's in
     "Testing" — no Google verification needed, because `drive.file` is not a sensitive
     scope).
4. **Create the Client ID:** APIs & Services → **Credentials** →
   **Create credentials → OAuth client ID**.
   - Application type: **Web application**.
   - Under **Authorized JavaScript origins**, add the origin(s) you'll open the app from:
     - `http://localhost:8000` (if testing locally)
     - `https://<your-username>.github.io` (for GitHub Pages)
   - Create, then copy the **Client ID** (ends in `.apps.googleusercontent.com`).

> **Important:** the origin must match exactly where the page is served from. GitHub Pages
> origin is `https://<username>.github.io` (no path).

**Already have the Client ID from the expense tracker?** Reuse it. Just enable the Drive
API, add the `drive.file` scope to the consent screen, and you're done — the GitHub Pages
origin is already authorized.

### 2. Prepare your Sheet

1. Create/open a Google Sheet you can edit.
2. Rename a tab to `Interior` (or anything — you'll type the name in settings).
3. Leave it empty; the app can create the first room table for you.
4. Copy the sheet URL or ID.

### 3. Connect the app

1. Open `index.html` (locally or via GitHub Pages — see below).
2. Tap the **⚙︎** icon (top right).
3. Paste your **OAuth Client ID**, your **Sheet link/ID**, the **Tab name** (default
   `Interior`), and **Save**.
4. Click **Sign in with Google**, choose your account, and allow access to Sheets and Drive.
5. Tap **＋ New Room** to create your first room.

Rooms load and tasks sync straight to your sheet. 🎉

---

## Hosting on GitHub Pages

It's a single static file, so hosting is free:

1. Push this repo to GitHub.
2. **Settings → Pages → Build and deployment → Source:** *Deploy from a branch*.
3. Branch **`main`**, folder **`/ (root)`** → **Save**.
4. Your app goes live at `https://<your-username>.github.io/<repo-name>/`.
5. Make sure that **origin** (`https://<your-username>.github.io`) is listed under your
   OAuth client's **Authorized JavaScript origins** (step 1.4 above).

Open the URL on any device, connect it once in ⚙ settings, and sign in. Settings are stored
per-browser, so enter them once on each device.

> You do **not** need a custom or "verified" domain — the free `github.io` origin works fine.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — inline HTML, CSS, and JS |
| `README.md` | This file |
| `ARCHITECTURE.md` | The reusable "Google Sheet as a database" blueprint |

---

## Notes & troubleshooting

- **"Access denied (403)":** confirm your signed-in Google account has **edit** access to
  the sheet, and that **both** the Sheets API and the Drive API are enabled in your Cloud
  project.
- **Photo upload fails with 403:** the Drive API isn't enabled, or `drive.file` wasn't added
  to the consent screen. Enable it, then sign out and back in so a fresh token is issued
  with the new scope.
- **Photos show as broken images on another device:** the "anyone with the link" permission
  didn't get applied (the app warns when this happens). Open the file in Drive and share it
  by link manually, or re-upload.
- **Sign-in popup blocked / origin error:** the exact origin you're serving from must be in
  **Authorized JavaScript origins**. For GitHub Pages that's `https://<username>.github.io`
  — not the `.../repo/` path.
- **"Sheet or tab not found (404)":** re-check the Sheet link/ID and Tab name in ⚙ settings.
- **"No room tables found":** the tab has no `Title` header row, or the header is missing its
  neighbouring `Description`/`Status` cell. Tap **＋ New Room** and let the app write one.
- **"Row changed — refresh and try again":** someone (or you, on another device) edited that
  task's row between your last sync and your write. The app refuses to clobber it; the next
  poll pulls in their change and you can redo yours.
- **Stuck on "Testing" consent screen:** add your Google account as a **Test user** on the
  OAuth consent screen. Personal use never needs Google's app verification.
- **Privacy:** the app runs entirely in your browser and only ever contacts Google's own
  APIs. Your Client ID, Sheet ID, and tab name live in `localStorage`; access tokens live
  only in memory and expire automatically.
