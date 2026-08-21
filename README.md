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
  coffee-date session, quick love-taps, and **On this day** memory
  resurfacing.
- **Notes** — text, photo, and voice notes, with a timeline and photo
  gallery view.
- **Us** — daily mood + a rotating daily question for both of you to
  answer.
- **Plans** — countdown to your next visit, a someday/bucket list, and
  **time capsules**: letters that stay sealed until a date you choose.
- **Settings** — profile, time zone, wake/sleep hours (used to compute
  your best overlapping hours to talk), and phone push via ntfy.

## Tiers

**Together (free)** — the entire daily heartbeat, with no caps:
love taps, unlimited text/photo/voice notes, the shared live vibe, moods,
the daily question, time zones and overlap, coffee-date sessions, streaks,
the visit countdown, and the someday list. Your full note archive is
always readable, forever, on the free tier.

**Forever (paid)** — the memory and ritual layer:

| Feature | Free | Forever |
| --- | --- | --- |
| Time capsules (sealed letters) | 1 sealed at a time | Unlimited |
| "On this day" resurfacing | — | ✓ |
| Custom love taps (inside jokes) | — | Up to 6 |
| "Us, so far" stats | — | ✓ |
| Themes (Midnight / Sakura / Sage) | Paper only | All |

Two deliberate design rules behind that split:

1. **We never lock a couple out of words they already wrote.** The free
   tier keeps full access to its own archive — only the *work done on top
   of it* (resurfacing, Wrapped) is paid. Holding someone's own memories
   hostage would be a bad way to make $5.
2. **Entitlement lives on the space, not the person.** One subscription
   covers both partners. Splitting a couple across two tiers makes no
   sense, and "your partner unlocked this for you" is a better moment
   than a second checkout.

### Billing is NOT implemented

`core.plan` is currently a plain client-side field, and `upgradeDemo()`
just flips it — good enough to demo the tier split, but anyone could set
it from devtools. To take real money you need:

1. Stripe Checkout (or RevenueCat for app stores) on the upgrade button.
2. A webhook → Cloud Function that writes `core.plan` **server-side**.
3. A Firestore rule forbidding clients from writing the `plan` field —
   the rules in this repo do not yet enforce that, because with no server
   there is nothing trustworthy to enforce it against.

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
