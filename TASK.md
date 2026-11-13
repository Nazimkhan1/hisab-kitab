# Hisab Kitab: what the app uses and how to run it

A daily expense, income and len-den (lend/borrow) app. It is a free **PWA** (web app that installs like a phone app), hosted on **GitHub Pages**. Each user's data stays on their own phone, or in **their own Google Drive** if they sign in. There is no server and no database.

- Live app: https://nazimkhan1.github.io/hisab-kitab/
- Share link: https://tinyurl.com/hisab-kitab (opens the app directly, no ad page)
- Code: https://github.com/nazimkhan1/hisab-kitab
- Pick a language with a link: add `?lang=hinglish`, `?lang=en`, `?lang=hi`, `?lang=bn` or `?lang=ur`

> Old bit.ly links (bit.ly/hisab-kitab-app, bit.ly/4jaivAy) still work, but the free Bitly plan shows an ad page first. Share the TinyURL link instead.

---

## 1. Files in this repo

| File | What it does |
|---|---|
| `index.html` | The whole app: HTML, CSS and JavaScript in one file. No build step and no npm. |
| `sw.js` | Service worker: lets the app open offline and be installed. |
| `manifest.json` | PWA details: name, colours, icons, `display: standalone`. |
| `icons/` | App icons: `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` (Android), `apple-touch-icon.png` (iPhone). |
| `privacy.html` | Privacy policy, also linked in the Google OAuth consent screen. |
| `terms.html` | Terms of use: how the app is secure, and that everything is in the user's hands. |
| `TASK.md` | This file. |

---

## 2. What the app uses

### Frontend
- Plain **HTML + CSS + vanilla JavaScript**, with no framework and no libraries.
- **Google Fonts**: Baloo 2 and Inter, plus language fonts that load only when that language is picked: Noto Sans Devanagari (हिन्दी), Baloo Da 2 / Noto Sans Bengali (বাংলা), Noto Nastaliq Urdu (اردو).
- Light and dark mode follow the phone setting.
- Urdu uses right-to-left layout (`dir="rtl"`).

### Storage on the phone
- Browser **localStorage**.
  - Without login, entries are in `hisab-kitab-entries-v1`.
  - With login, entries are in `hisab-kitab-acct:<google user id>`, so each Google account on a phone has its own copy.
- Other keys:
  - `hisab-lang`: chosen language.
  - `hisab-tab`: last tab.
  - `hisab-install-dismissed`: when "Baad mein" was tapped on the install popup.
  - `hisab-kitab-welcome-skipped`: when the login prompt was skipped.
  - `hisab-kitab-last-account`: the account logged in last time.

### Google login and Drive
- **Google Identity Services** (`https://accounts.google.com/gsi/client`), using the token client (`initTokenClient`).
- Scopes:
  - `openid email profile`: name, email and photo shown in the header.
  - `https://www.googleapis.com/auth/drive.file`: the **only** Drive permission. The app can use only the files and folder it creates itself. It cannot see the user's other Drive files.
- The access token stays **in memory only** and expires in about 1 hour. It is never saved or sent anywhere except to Google.
- **Google Drive REST API v3**: folder find/create, file create (multipart upload), update (PATCH), list and delete.

### What gets saved in the user's Drive
A folder named **"Hisab Kitab"** in the user's **My Drive** holds:
- `_sync-data.json`: the full copy used to sync phones (all entries plus deleted ids).
- `YYYY-MM-DD.json`: one file per date (for example `2026-11-12.json`) that the user can open and read:
  ```json
  {
    "app": "hisab-kitab",
    "date": "2026-11-12",
    "updatedAt": "...",
    "totals": { "kharcha": 0, "aamdani": 0, "udhaarDiya": 0, "udhaarLiya": 0, "wapasMila": 0, "wapasDiya": 0 },
    "entries": [ { "id": "...", "type": "kharcha", "typeName": "Expense", "amount": 450, "category": "Khana-peena", "note": "sabzi", "createdAt": "..." } ]
  }
  ```

### How sync works
- **Local first.** Every entry is saved on the phone at once. About a second later it is written to Drive.
- Entries are never edited, only added or deleted, so merging two devices is simple: take everything from both, and drop the ids listed in `deleted`.
- Date files are rewritten only for dates whose entries changed. The app keeps a small hash per date to know this.
- If the first entries were made before login, they are merged into the account when the user logs in.

### PWA and install
- `sw.js` caches the app shell (page, manifest, icons) and Google Fonts. Google login and Drive calls always go to the network and are never cached.
- The page loads network-first, so updates show up; it falls back to the cached copy offline.
- Install popup:
  - **Android Chrome:** uses the `beforeinstallprompt` event and shows an **"Install karo"** button.
  - **iPhone Safari:** shows the 3 steps (Share → Add to Home Screen → Add), because iOS has no install button for web apps.
  - **Inside WhatsApp or another in-app browser:** tells the user to open the link in Chrome or Safari.
  - After "Baad mein" it stays hidden for 3 days. It never shows once the app is installed.
- A loading screen (₹ logo + spinner) shows while the app opens.

### Short link
- **TinyURL** at https://tinyurl.com/hisab-kitab, made without an account, so it can't be edited. It is a direct 301 redirect.

---

## 3. App features

- **Nayi entry:** Kharcha, Aamdani, Udhaar diya, Udhaar liya, Wapas mila, Wapas diya. Each entry has an amount, a date, a category or person, and a note.
- **Voice entry (🎤 Bol kar likho):**
  - The user speaks, e.g. "Sabzi 250", "Ramesh ko 2000 udhaar diya" or "kal petrol 300", and the form fills itself (type, amount, person, category, date, note).
  - The user checks the form and taps "Entry jodo". Nothing is saved automatically.
  - Uses the browser's Web Speech API (`SpeechRecognition`), with the speech language following the app language (hi-IN, en-IN, bn-IN, ur-IN).
  - The sentence is understood by the `VOICE.parse()` keyword parser in `index.html`. It handles Latin, Devanagari, Bangla and Urdu text.
  - The mic button is hidden where the browser has no speech support (for example Firefox).
  - **Several items in one sentence:** "Pyaj 1 kg 60, aalu 1 kg 30, bhindi 1 kg 70" opens a review list.
    - Each item gets its own name, quantity, amount and category, and the user can edit any of them before saving.
    - "N entry jodo" adds them all as separate kharcha entries (`VOICE.parseList()`).
    - This works only for voice, and only for kharcha.
- **Saved message:** a green toast ("✓ Kharcha ₹250 jud gaya") shows for about 2.5 seconds after every save.
- **Roz ka hisab:** entries for each month grouped by date, with a "Kharcha kahan gaya" category breakdown.
- **Len-den:** each person's balance (Lena hai / Dena hai) with quick buttons to record money returned.
- **Statement (bank style):**
  - Pick a month, a year or a date range, and filter by all / daily / len-den.
  - Shows opening balance, money in, money out and closing balance.
  - The table has Date | Details | Debit | Credit | Balance, with a running balance.
  - **Excel (CSV) download** and **PDF / Print**.
- **Top cards:** this month's kharcha, aamdani, lena hai, dena hai.
- **Languages:** Hinglish, English, हिन्दी, বাংলা, اردو.
- **Login with Google** saves to the user's own Drive and shows their photo in the header. A sync dot shows ✓ saved / … saving / ! error.
- **Without login:** works on that phone only. "Backup lo" and "Backup se wapas lao" in the ⋮ menu save or restore a `.json` file. These buttons are hidden after login, because Drive saves automatically.
- **Login prompt:** shown after the user saves their first entry ("Entry save ho gayi! Ab login kar lijiye…"). After "Abhi nahi" it shows again 3 days later.
- **Mobile:** bottom tabs for Nayi entry, Roz ka hisab, Len-den and Statement.
- **⋮ menu:** language, install app, sync, backup, terms, privacy, logout.

---

## 4. One-time setup that is already done (keep for reference)

### GitHub Pages
1. Create the repo `nazimkhan1/hisab-kitab` (Public).
2. Upload all the files.
3. Go to **Settings → Pages → Source: Deploy from a branch → `main` / `(root)` → Save**.
4. After about 1 minute the site is live at `https://nazimkhan1.github.io/hisab-kitab/`.

### Google Cloud (personal project `hisab-kitab-514509`)
1. https://console.cloud.google.com → create the project.
2. **APIs & Services → Library → Google Drive API → Enable**.
3. **Google Auth Platform → Branding:**
   - App name: Hisab Kitab
   - Support email
   - Logo: the 120px logo
   - App home page: `https://nazimkhan1.github.io/hisab-kitab/`
   - Privacy policy: `.../privacy.html`
   - Terms: `.../terms.html`
   - Authorized domain: `nazimkhan1.github.io`
4. **Audience:** External, then **Publish app → In production**, so any Gmail user can log in without being added as a test user.
5. **Data access → Add scopes:** `.../auth/drive.file`, `openid`, `email`, `profile`. All are non-sensitive, so **no Google verification is needed**.
6. **Clients → Create client → Web application**:
   - Authorized JavaScript origin: `https://nazimkhan1.github.io`
   - Copy the **Client ID** into `GOOGLE_CLIENT_ID` in `index.html`.
7. Branding verification (to show the logo on the Google screen) is optional. It needs domain ownership in Google Search Console.

---

## 5. How to update the app (every time)

1. Edit `index.html`, or whichever file needs the change.
2. **Bump `VERSION` in `sw.js`** (for example `hk-v12` → `hk-v13`). Without this, phones keep showing the old app.
3. Upload the changed files to GitHub. Either:
   - **Add file → Upload files → Commit changes**, or
   - `gh` CLI after a one-time `gh auth login` with the Nazimkhan1 account.
4. Wait 1–2 minutes for GitHub Pages to update.
5. Check that it is live:
   ```sh
   curl -s "https://nazimkhan1.github.io/hisab-kitab/sw.js?nc=$RANDOM" | grep VERSION
   ```
6. On the phone, close the app fully and open it again. The second open shows the new version.

If the site URL or domain ever changes, add the new origin in the Google OAuth client (**Clients → Authorized JavaScript origins**), or login will stop working.

---

## 6. Testing on a computer

- Open `index.html` directly in Chrome. Everything except Google login works. Login needs the real `https://nazimkhan1.github.io` origin.
- Check the phone layout in Chrome DevTools → device toolbar (375px wide).
- Check languages with `?lang=ur` (right-to-left) and `?lang=bn`.

---

## 7. Security and privacy (in short)

- No server and no database. The app maker has no admin panel, no user list and no access to anyone's data.
- Data is on the user's phone, or in the user's own Drive "Hisab Kitab" folder.
- Only the `drive.file` permission, which covers the app's own files only.
- Login happens on Google's own screen. The app never sees the password.
- The token is kept in memory only and expires in about 1 hour.
- HTTPS everywhere. No ads, analytics or tracking. The code is public.
- To delete data, the user can delete the "Hisab Kitab" folder in Drive. To remove the app's access, they can go to https://myaccount.google.com/permissions.

---

## 8. Ideas / pending

- [ ] Root short link: a repo named `nazimkhan1.github.io` with a redirect `index.html` (file ready at `~/www/nazimkhan1.github.io/index.html`), so `nazimkhan1.github.io` opens the app.
- [ ] Optional Google branding verification (Search Console domain ownership).
- [ ] Android fallback install popup: if Chrome doesn't offer install in a few seconds, show the manual "⋮ → Install app" steps.
