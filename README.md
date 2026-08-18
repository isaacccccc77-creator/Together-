# Us, apart

A quiet little space for a long-distance couple — one static HTML file, a
shared Firebase Firestore document per couple, and free push via
[ntfy.sh](https://ntfy.sh). No build step, no server to run.

## Setup

1. Create a free project at [firebase.google.com](https://firebase.google.com).
2. Enable **Firestore Database**.
3. Enable **Authentication → Sign-in method → Anonymous**. The app signs
   users in anonymously on load so Firestore rules can require
   `request.auth != null` — see the security note below.
4. In Firestore, open **Rules** and paste in the contents of
   [`firestore.rules`](./firestore.rules) from this repo.
5. Add a Web App to the project, copy its config object, and paste it into
   `FIREBASE_CONFIG` near the top of the `<script>` in `index.html`.
6. Open `index.html` (or host it — e.g. GitHub Pages) and create a space.

## Features

- **Home** — a "same sky, different hour" time zone bar, a live shared
  **vibe** (pick an ambient color/mood and it syncs to your partner's
  screen in real time, tinting the whole app), a streak, a virtual
  coffee-date session, and quick love-taps.
- **Notes** — text, photo, and voice notes, with a timeline and photo
  gallery view.
- **Us** — daily mood + a rotating daily question for both of you to
  answer.
- **Plans** — countdown to your next visit and a someday/bucket list.
- **Settings** — profile, time zone, wake/sleep hours (used to compute
  your best overlapping hours to talk), and phone push via ntfy.

## Security note

There's no per-couple authentication in this app — the invite code (e.g.
`PEACH-42`) is effectively the password to a space. `firestore.rules`
requires anonymous auth and blocks listing all spaces, which stops random
bots/scanners from reading every couple's data once Firestore's default
"test mode" rules expire. It does **not** stop someone who has your code
from reading your notes, photos, or voice memos — don't share invite codes
publicly.

## Known limits

- Firestore documents cap at 1MiB; a very long voice note can fail to
  save (the app now shows an error toast instead of silently dropping it,
  but consider moving large media to Firebase Storage if you extend this).
- Not yet installable as a PWA (no manifest/service worker) — phone push
  only works through the ntfy app, not a native install.
