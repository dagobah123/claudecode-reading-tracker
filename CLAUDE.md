# CLAUDE.md — Reading Tracker (Android App)

## Project overview
A mobile-first reading tracker with notes and flashcard review, shipped as a **native Android APK**. The UI is a single-file web app wrapped in a minimal Android WebView shell. Code is written in Termux (proot Ubuntu) on the phone; the APK is **built by GitHub Actions**, never locally.

## Environment constraints (important)
- Development happens inside Termux (proot Ubuntu) on an Android phone.
- NEVER attempt to install or run the Android SDK, Gradle builds, or emulators locally — they do not work in proot. All APK builds happen in GitHub Actions (`ubuntu-latest` runners have the Android SDK preinstalled).
- No npm, no bundlers, no JS frameworks. The web layer is ONE file: `app/src/main/assets/index.html` with inline CSS and vanilla JavaScript.
- Quick local testing of the web UI: `python3 -m http.server 8080` from the assets folder, open `localhost:8080` in the phone browser.

## Repository layout
```
reading-tracker/
├── CLAUDE.md
├── .github/workflows/build.yml     # CI: builds debug + release APK on push, uploads as artifact
├── settings.gradle
├── build.gradle
├── gradle/ + gradlew               # Gradle wrapper (committed so CI needs no setup)
├── logo.svg                        # master logo (book icon, see below)
└── app/
    ├── build.gradle                # minSdk 26, targetSdk 34, Kotlin, single Activity
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/.../MainActivity.kt   # WebView loading file:///android_asset/index.html
        ├── assets/index.html          # the entire app UI + logic
        └── res/                       # launcher icons generated FROM logo.svg
            └── drawable/ic_logo.xml   # logo as Android vector drawable
```

## Android shell rules
- One Kotlin `MainActivity` with a full-screen `WebView`; enable JavaScript and DOM storage (`domStorageEnabled = true`) so `localStorage` persists.
- Handle back-button to navigate the WebView, exit on double-press.
- No permissions in the manifest except none — the app is fully offline.
- Package name: `dev.local.readingtracker`.

## CI / GitHub Actions (`.github/workflows/build.yml`)
- Trigger on push to `main` and manual `workflow_dispatch`.
- Steps: checkout → set up JDK 17 → `./gradlew assembleDebug assembleRelease` → upload APKs as build artifacts.
- Sign release builds with a keystore stored in repo secrets (`KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`); if secrets are missing, still produce the debug APK.
- Workflow: push from Termux with git, download the APK artifact from the Actions page on the phone, install it.

## Logo
- Master file is `logo.svg`: an open book with rounded pages on a deep indigo circle, warm amber bookmark ribbon. Flat, modern, legible at 48px.
- Reference SVG (keep this as the source of truth):
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 128 128">
  <circle cx="64" cy="64" r="60" fill="#312e81"/>
  <path d="M64 42c-8-6-20-8-30-6v52c10-2 22 0 30 6 8-6 20-8 30-6V36c-10-2-22 0-30 6z" fill="#eef2ff"/>
  <path d="M64 42v52" stroke="#c7d2fe" stroke-width="3"/>
  <path d="M40 50c6-1 12 0 17 2M40 60c6-1 12 0 17 2M40 70c6-1 12 0 17 2M71 52c5-2 11-3 17-2M71 62c5-2 11-3 17-2M71 72c5-2 11-3 17-2" stroke="#818cf8" stroke-width="3" stroke-linecap="round" fill="none"/>
  <path d="M82 30v26l6-5 6 5V30z" fill="#f59e0b"/>
</svg>
```
- Convert `logo.svg` into `res/drawable/ic_logo.xml` (vector drawable) and adaptive launcher icons (`mipmap-anydpi-v26` with foreground/background layers). Also reuse it inline in `index.html` as the header logo.

## Web app features (in priority order)
1. Add/edit/delete books with title, author, total pages.
2. Update current page → progress bar and percent complete.
3. Daily reading log → streak counter and pages-per-day stat.
4. Notes & quotes per book.
5. Flashcard review mode: shows a note prompt, user self-grades (again/good/easy), simple spaced-repetition scheduling.

## Data
- All data in `localStorage` under one key: `readingTrackerData` (single JSON object).
- `books`: { id, title, author, totalPages, currentPage, status ("reading" | "finished" | "abandoned"), dateAdded, dateFinished }
- `logs`: { date (YYYY-MM-DD), bookId, pagesRead }
- `notes`: { id, bookId, text, isQuote, createdAt, srs: { interval, dueDate, ease } }
- Never clear localStorage without an explicit confirm dialog. Provide Export (download JSON) and Import (restore) buttons.

## Design rules
- Mobile-first: touch targets ≥ 44px, single-column layout, no hover-dependent UI.
- Dark theme by default (indigo/amber palette matching the logo), high contrast.
- Fully offline: no external fonts, no CDN requests, no network calls.

## Coding conventions
- Web layer: vanilla ES6+, small pure functions, one `render()` redrawing from state, all state changes through `saveData()`. Keep `index.html` under ~1500 lines.
- Kotlin layer: keep `MainActivity.kt` minimal (< 100 lines); no libraries beyond AndroidX core + WebKit.

## Workflow
- Make small, verifiable changes; describe how to manually test each change on the phone.
- Web UI changes: test via the local HTTP server first, then push to trigger a CI build.
- Never edit generated launcher icons by hand — regenerate them from `logo.svg`.
```
