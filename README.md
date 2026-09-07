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

- **Home** — a "same sky, different hour" time zone bar, presence and the
  goodnight ritual, **pockets of time** (overlapping windows where you're
  both awake and free, claimable by either of you), a live shared
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
- **Move** — a GPS run tracker, a **live synced run** (watch their
  distance climb next to yours while you're both out there), and
  **closing the distance**: the real great-circle gap between your two
  cities, chipped away by every kilometre you both run. Manual logging
  for anyone who already tracks with Strava, Garmin or Nike.
- **Settings** — profile picture and details, time zone, wake/sleep and
  working hours, calendar import, and phone push via ntfy.

## Invite codes and identity

The invite code is the credential for a space, so it has to be hard to
guess. It is `ADJECTIVE-NOUN-XXXX` (e.g. `AMBER-HARBOR-K7P2`) drawn from
`crypto.getRandomValues`, where `XXXX` is base32 with `0`, `1`, `O`, `I`
and `L` removed so a code survives being read down the phone. That is
64 x 64 x 31^4 = **3,782,742,016** combinations.

The previous scheme was 12 words x 89 numbers = 1,068, which had two
serious consequences that are now fixed:

- **Collisions destroyed data.** Space creation did a blind `setDoc`, so
  generating a code another couple already held overwrote their entire
  space. At 1,068 combinations the birthday paradox put that at roughly a
  coin flip by the 38th couple. Creation now checks whether the document
  exists first and retries, and never writes over an existing space.
- **Anyone with a code could take over a partner.** Joining always
  assigned role `b` and overwrote `core.b`, so a third party who guessed
  a code silently replaced partner B and inherited the space. Each
  profile now records the anonymous-auth `uid` that owns it. Joining
  fills an empty slot or recognises a returning device; if both slots are
  held by other devices it asks *which one of you is this?* rather than
  overwriting anyone. Existing spaces adopt a uid on next launch.

Codes remain bearer credentials — anyone you give one to is in. This
raises the cost of guessing by about 3.5 million times; it is not a
substitute for real accounts.

## Calendars and pockets

**Pockets** scans the next three days in 15-minute slots and returns the
windows where *both* of you are awake, outside working hours, and free of
imported calendar blocks — all evaluated in each person's own time zone.
Either partner can claim one and it is pencilled in for both. Where a
window falls on a different calendar date for each of you, the partner's
side is tagged with their weekday, because "tomorrow" is not the same
word for both of you.

Free/busy comes from two sources:

1. **Waking and working hours**, set in Settings. No integration needed.
2. **A `.ics` import.** Export from Google Calendar (Settings →
   Import/Export), Apple Calendar (File → Export) or Outlook and drop the
   file in. Parsed entirely in the browser. Handles UTC and `TZID` times,
   all-day events, RFC 5545 line folding, and `DAILY`/`WEEKLY` recurrence
   including `BYDAY`, `COUNT` and `UNTIL`. `TRANSPARENT` (shown as free)
   and `CANCELLED` events are ignored. Monthly and yearly recurrence is
   skipped rather than guessed at. **Only start and end times are
   stored** — no titles, guests or locations ever reach the database.

### Live calendar sync is not built

- **Google Calendar** *is* reachable from a pure browser app: Google
  Identity Services issues access tokens without a client secret. But it
  needs your own OAuth client ID with authorised origins, and Calendar is
  a sensitive scope, so Google verification is required before more than
  a hundred people can use it. Its browser tokens also expire in about an
  hour with no refresh token, so reads only happen while the app is open
  — there is no background sync without a server.
- **Apple Calendar / iCloud** has no public API. CalDAV needs an
  app-specific password and is blocked by CORS from a browser.
- **Secret `.ics` subscription URLs** from Google/Outlook/Apple are also
  CORS-blocked, so they would need a proxy.

The `.ics` import exists because it needs none of that and works today.

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
| Run tracking, live synced runs, closing the distance | ✓ | ✓ |
| Saved route shapes | — | ✓ |

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

## Presence, sleep, and quiet hours

### What this app deliberately does not do

It does **not** detect when a phone is powered off, in airplane mode, or
on Do Not Disturb / Focus. No browser exposes any of that, by design —
device state like that is a fingerprinting and stalking vector. A native
app barely helps: iOS gates Focus status behind a special entitlement and
user permission and only yields a boolean, Android needs
notification-policy access, and *nothing* can report a powered-off phone
because the app on it is dead too.

More importantly, we don't want it. Inferring "they went to sleep" from
an absent signal is the shape of passive partner monitoring, and absence
of a heartbeat is equally well explained by a tunnel, a flat battery, or
a force-quit. So the app never guesses. It says "last had the app open
42m ago" — app activity, described as app activity.

### What it does instead

- **Presence heartbeat.** While the app is open and visible it writes a
  timestamp to `spaces/{code}/live/presence` every two minutes. That
  drives an honest "has the app open right now" / "last had the app open
  20m ago".
- **The goodnight ritual.** You *tap* goodnight; your partner sees it.
  Intentional and mutual rather than inferred. It clears itself if you're
  active again inside your own waking hours, so nobody stays "asleep" for
  days after forgetting.
- **Inferred quiet hours, with zero tracking.** Both partners already set
  their wake/sleep hours in Settings, so "it's 2:04 AM for Jamie" needs
  no monitoring at all.
- **The quiet-hours guard.** Try to send a love tap while they're asleep
  or outside their waking hours and the app stops you. For notes it
  offers **"leave it quietly"** — the note saves and is waiting when they
  wake, but their phone never buzzes. The notes composer shows the same
  warning up front, so the guard is never a surprise at send time.

## Running: how it works, and why not Strava (yet)

The tracker uses the browser's `navigator.geolocation.watchPosition`,
summing haversine distance between fixes. It filters out fixes vaguer
than 40m, steps under 3m (GPS jitter while standing still), and implied
speeds over 9 m/s (a GPS jump, not a runner). Rejected fixes keep the
previous anchor rather than resetting it, so a slow jogger's distance
accumulates across several fixes instead of being discarded.

Live run state is written to its own small doc (`spaces/{code}/live/run`)
every 5 seconds rather than into the main core document — otherwise a
30-minute run would rewrite the whole space document a few hundred times.

**Known limits:**

- A browser tab cannot track GPS once the phone locks. The app requests a
  screen Wake Lock where supported, and tells the user to keep the screen
  on. This is the honest ceiling for a web app; a native wrapper
  (Capacitor) or a real background-location API is the only fix.
- Home coordinates for the "distance between us" calculation are rounded
  to one decimal degree (~11km) before they are stored — city-level, so
  the feature works without keeping anyone's address in the database.
- Route traces are precise location history, so they are only saved for
  couples on the paid tier, which is the only tier that draws them.

### Strava

Not integrated, and it can't be from a static file. Strava uses OAuth2:
redeeming the authorization code for tokens requires your
`client_secret`, and anything in this HTML file is readable by anyone who
opens it. Shipping the secret client-side would let a stranger pull your
users' Strava data. Refresh tokens also need somewhere server-side to
live.

To do it properly you need a small backend (a Firebase Cloud Function
fits, since Firebase is already here):

1. Register an API application in Strava's developer settings for the
   `client_id` / `client_secret`.
2. Function 1 — `stravaCallback`: receives Strava's redirect, exchanges
   the code for access + refresh tokens using the secret, stores them
   under the space, keyed per partner.
3. Function 2 — `stravaWebhook`: subscribe to Strava's activity webhook
   so new activities are pushed to you; on each event, fetch the
   activity and write a run into `spaces/{code}/runs` in the same shape
   the tracker already produces.
4. Refresh the access token when it expires and write the new one back.

Because step 3 writes into the existing `runs` collection, everything on
the Move screen — the live gap, the totals, the history — keeps working
untouched.

**Check Strava's API Agreement before building this.** It places real
restrictions on displaying one athlete's data to another user, which is
exactly what a couples app does. That is a product risk, not a technical
one, and it is worth resolving before writing the functions.

Manual logging exists so Strava/Garmin/Nike users aren't shut out in the
meantime — the numbers count toward the shared totals identically.

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
