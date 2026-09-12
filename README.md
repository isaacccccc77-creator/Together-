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
- **Plans** — countdown to your next visit, **the fund** (a shared pot
  for the flight, with a weekly plan that re-plans itself), a
  someday/bucket list, and **time capsules**: letters that stay sealed
  until a date you choose.
- **Move** — a daily **metabolic circuit** you can both do in a flat
  with no kit, a GPS run tracker, a **live synced run** (watch their
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

**Pockets** scans the next three days in 15-minute slots and shows the
windows where *both* of you are awake, outside working hours, and free of
calendar blocks — all evaluated in each person's own time zone. It is
purely something to read: no claiming, no buttons, nothing to negotiate.
Where a window falls on a different calendar date for each of you, the
partner's side is tagged with their weekday, because "tomorrow" is not
the same word for both of you.

Free/busy comes from three sources.

### 1. Waking and working hours

Set in Settings. No integration needed, works immediately.

### 2. Google Calendar (one tap)

Uses Google Identity Services' token flow, which is designed for browser
apps and needs **no client secret** — so the app still has no server.

Two deliberate choices:

- It calls the **`freeBusy`** endpoint, not `events.list`. That returns
  nothing but busy intervals, so event titles, guests and locations never
  reach the app or the database, even in memory.
- The access token lives in a variable and is **never persisted**. GIS
  tokens last about an hour with no refresh token, so "connected" is a
  stored preference, not a stored credential. On launch the app quietly
  tries to re-mint a token; if that fails, the busy blocks already synced
  are still there.

To enable it:

1. In Google Cloud Console, create an **OAuth 2.0 Client ID** of type
   *Web application*.
2. Add the origin you serve the app from to **Authorised JavaScript
   origins**. It must be `https://…` — Google will not accept a
   `file://` page, so the app has to be hosted (GitHub Pages is fine).
3. Paste the client ID into `GOOGLE_CLIENT_ID` in `index.html`.

Note that Calendar scopes are classed **sensitive** by Google, so an app
serving the public needs to pass OAuth verification. Until it does, the
project stays in testing mode and only test users you add explicitly can
connect. The `Connect Google Calendar` button only appears once
`GOOGLE_CLIENT_ID` is filled in; without it the card falls back to file
import.

### 3. A `.ics` import

Needs no setup at all. Export from Google Calendar (Settings →
Import/Export), Apple Calendar (File → Export) or Outlook and drop the
file in. Parsed entirely in the browser: UTC and `TZID` times, all-day
events, RFC 5545 line folding, and `DAILY`/`WEEKLY` recurrence including
`BYDAY`, `COUNT` and `UNTIL`. `TRANSPARENT` (shown as free) and
`CANCELLED` events are ignored. Monthly and yearly recurrence is skipped
rather than guessed at. Only start and end times are stored.

**Apple Calendar** still has no public API — CalDAV needs an
app-specific password and is CORS-blocked from a browser — and secret
`.ics` subscription URLs are CORS-blocked too, so both remain
file-import only.

## The fund

A shared pot for the thing that ends the distance. You set what it's
for, a target, a currency and a date; both of you pay in; the app keeps
the plan honest.

The plan is arithmetic done carefully rather than a model:

- **what's left**, and **how many weeks** remain
- **per person, per week** to land on the date
- **ahead or behind the line** you'd need to be on by now, with the
  weekly figure already absorbing any shortfall rather than quietly
  failing
- **where your actual pace lands you**, which is often the more useful
  number

It re-plans on every payment. Crossing 25/50/75/100% fires confetti
once — on the crossing, not on every payment — and notifies the other
person. Paying in counts toward the daily streak.

## Staying power

Two changes aimed squarely at the app still being open in six months:

- **120 daily questions**, up from 15. Fifteen repeated every fortnight,
  which is roughly when a daily ritual starts feeling like a chore.
- **Streak forgiveness.** A missed day used to send a 60-day streak to
  zero, which is exactly the moment people stop opening an app like
  this. A streak now survives one missed day, at most once a fortnight,
  and the app says so out loud — "we covered yesterday for you" — because
  a silent lie would be worse than the reset.

## Design system notes

**Bento grids.** The "Us, so far" stats are laid out on a four-column
bento: hero tiles spanning the full width, some tiles spanning two
columns, some rows carrying three figures side by side. The uneven
rhythm is the point — a wall of identical squares reads as a
spreadsheet, while mixed spans give the eye somewhere to land first.

**Animated arrows**, used where an arrow means something rather than as
decoration:

- Two arrows creep *toward each other* along the closing-the-distance
  bar — the entire feature expressed in one gesture.
- A nudging chevron on primary calls to action.
- A turn arrow linking your side of a pocket to your partner's.

All of them stop under `prefers-reduced-motion`.

## Typography

- **Cormorant Garamond** for display — a high-contrast old-style serif,
  the elegant and romantic end of luxury rather than the loud
  fashion-magazine end.
- **Jost** for the interface — a quiet geometric sans that stays out of
  the serif's way.
- **Parisienne** for handwritten moments — kept to the sealed capsule
  envelopes alone, where a script reads as a letter. The daily question
  used it too and was too loud for a line of interface copy; it is now
  Cormorant's italic, which is the elegant version of the same intent.

Two things this needed. Cormorant has a small x-height, so everything set
in it is sized up about 12% to hold the same optical weight. It also
defaults to **old-style figures**, which drop 9s and 8s below the
baseline — fine in a sentence, wrong in a clock or a counter — so every
numeric display forces `lining-nums tabular-nums`.

Filled buttons take their colour from `--btn-bg` rather than `--rose`.
`--rose` is a decorative fill and failed WCAG AA under white text in
three of the four themes (paper 3.9:1, sakura 2.78:1, sage 3.63:1).
Each theme now sets a button ground measured against its own text
colour; all four clear 4.5:1.

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

## Metabolic circuits

Circuits are **generated per day**, not picked from a fixed list. A
seeded PRNG keyed to the UTC day number means both partners produce the
identical circuit from the date alone, with nothing to sync between
them — and it flips for both at the same instant regardless of time
zone.

**The library** is 33 bodyweight exercises, all doable in the space
beside a bed with no kit: squats, forward and reverse lunges, side
lunges, squat pulses, glute bridges, wall sit, calf raises, push ups,
incline push ups, tricep dips, superman holds, arm pulses, Russian
twists, flutter kicks, windmills, 4 point shoulder taps, plank, bicycle
crunches, sit ups, dead bug, bear hold, toe touches, jumping jacks,
jogging on the spot, mountain climbers, high knees, skaters, fast feet,
shadow boxing, inchworms, squat to reach and burpees. Each is tagged
with a category and an impact level, so the generator can balance a
circuit and mark the quiet ones for anyone with neighbours below.

**Generation** fills a six-slot pattern (e.g.
`cardio → lower → core → upper → lower → cardio`) drawn from six
shapes, so you never get six core moves in a row. Moves never repeat
within a circuit. Warm-up and cool-down are two each, also seeded.
Intensity varies between 40s/20s × 2, 30s/15s × 3 and 45s/15s × 2, so
sessions land between roughly 15 and 16 minutes.

**Rest days are planned, not random.** Every week gets exactly two. A
free shuffle was measured producing a **9-day unbroken active streak**
and **3 rest days in a row**, because independent weeks can stack their
rests at opposing edges. Rest pairs are now drawn from spacings 2–4
days apart, the first landing by midweek and the second from midweek
on, which bounds any run at **6 active days and 2 rest days** — verified
across 156 consecutive weeks. A rest day isn't an empty card: it offers
a six-minute mobility flow, and says plainly that doing nothing counts.

Verified over a year of generated days: 260 active and 104 rest, zero
repeated moves inside a circuit, every circuit carrying both cardio and
core, all 33 exercises reached, and identical output on repeat calls.

### Spotify

The workout screen embeds a shared playlist. This is an **embed, not an
integration**: `open.spotify.com/embed/...` in an iframe needs no API
key, no OAuth and no client ID, so there is nothing to set up and
nothing to verify. Paste a link once and you both get the same music.

Playback depth is Spotify's call, not ours — an embedded player behaves
differently for a logged-in Premium listener than for an anonymous one,
so treat full-track playback as something to confirm on your own
account rather than something this app guarantees.

Going further would mean the Web Playback SDK, which requires OAuth
*and* a Premium subscription on both sides. Authorization Code with
PKCE would keep it serverless, but demanding two Premium accounts
before a couple can do twenty squats is a bad trade.

Link parsing accepts everything people actually paste: `spotify:` URIs,
`open.spotify.com` links with or without a scheme, `?si=` tracking
params, and Spotify's regional `/intl-xx/` prefixes.

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
