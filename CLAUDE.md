# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

"Us, apart" — a single-page web app for a long-distance couple to stay
connected: shared notes/photos/voice memos, a live "vibe" that tints both
partners' screens, a streak counter, time-zone overlap finder, a virtual
"coffee date" session timer, and free push notifications via ntfy.sh.

There is no build step, no package manager, no server-side code, and no test
suite. The entire application is one static HTML file. Development consists
of directly editing that file and opening it in a browser (or hosting it,
e.g. on GitHub Pages).

## Repository layout

```
index.html       The entire app: markup, CSS, and JS in one file (~2000 lines)
firestore.rules  Firestore security rules — paste into Firebase Console
README.md        Setup instructions, feature list, security/limits notes
```

That's it — there are no other source files, no `node_modules`, no config
files, no CI. Do not introduce a build toolchain, bundler, or framework
unless explicitly asked; the whole point of this project is that it's a
single file anyone can open and edit.

## Architecture of `index.html`

The file has three parts, in order:

1. **`<style>` (head)** — all CSS, using custom properties defined on `:root`
   (colors like `--rose`, `--honey`, `--dusk`; shadows; radius; fonts). Fonts
   are Google Fonts (`Fraunces` for display, `Libre Franklin` for body,
   `Caveat` for handwritten accents), loaded via `<link>`.
2. **`<body>` markup** — a sequence of screens, each an HTML block with an
   explanatory HTML comment marker:
   - `<!-- SETUP GATE -->` — shown if `FIREBASE_CONFIG` hasn't been filled in
   - `<!-- ONBOARDING -->` — create a space / join with an invite code
   - `<!-- MAIN APP -->` containing five tabs: `<!-- HOME -->`,
     `<!-- NOTES -->`, `<!-- TOGETHER -->` (the "Us" tab), `<!-- PLANS -->`,
     `<!-- SETTINGS -->`
   Navigation between screens/tabs is done by toggling CSS classes via JS
   (`showScreen()`), not routing — there's no URL/history involvement.
3. **`<script type="module">`** — all application logic, top to bottom:
   - Static data/config: `TIMEZONES`, `MOODS`, `TAPS`, `VIBES`, `PROMPTS`,
     `FIREBASE_CONFIG`
   - Global mutable `state` object (`{ code, role, core, notes }`) and a few
     other module-level `let`s for UI state (recording, lightbox, etc.)
   - Firebase bootstrap (`loadFirebase()`, dynamic `import()` of the Firebase
     v10 modular SDK from the `gstatic.com` CDN — no local install)
   - Data helpers (`localGet/Set/Remove` wrapping `localStorage`,
     `defaultCore()`, `updateCore()`)
   - Realtime sync (`subscribeSpace()` — Firestore `onSnapshot` listeners)
   - Per-feature render functions, one per screen/widget: `renderHome()`,
     `renderNotes()`, `renderTogether()`, `renderPlans()`, `renderSettings()`,
     `renderVibe()`, `renderStreak()`, `renderOverlap()`, `renderSession()`,
     `renderGallery()`, `renderLightbox()`, etc.
   - `renderAll()` calls every render function; it's the ambient "re-render
     everything after any state change" bulk update — there is no virtual DOM
     or reactivity, functions just re-set `textContent`/`innerHTML` directly.
   - `init()` runs at the bottom of the script and bootstraps the whole app.

### Data model

Two Firestore documents/collections per couple, keyed by the invite code
(`state.code`, e.g. `PEACH-42`):

- `spaces/{code}` — the "core" doc: both partners' profiles (`a`/`b`: name,
  tz, wake/sleep hours), start date, next visit date, mood, daily prompt
  answers, pings (love-taps), vibe, session state, streak, bucket list,
  ntfy topic. Read via `onSnapshot`, written via `updateCore(mutator)` which
  clones `state.core`, applies the mutator, optimistically updates local
  state, then `setDoc`s the whole document back (last-write-wins — there's no
  merge/transaction logic).
- `spaces/{code}/notes/{noteId}` — one doc per note (text/photo/voice),
  append-only (`create` only; rules block `update`/`delete`).

Per-device local state lives in `localStorage`: `my-role` (`{ code, role }`)
and `last-seen-{code}` (for unseen-ping/note badges). Nothing sensitive is
stored there; the invite code is the actual secret.

### Role model

Each partner is either role `"a"` or `"b"`. `me()`/`them()`/`myKey()`/
`theirKey()` are the small helpers used everywhere to avoid `if role === 'a'`
duplication — reuse them rather than re-deriving role logic inline.

## Conventions to follow when editing

- **Keep it a single file.** Don't split into modules, add a bundler, or
  pull in npm packages. External libraries are loaded via CDN `<script>`/
  `import()` only (currently just Firebase).
- **No build step.** Changes must work by opening `index.html` directly (once
  `FIREBASE_CONFIG` is set) — don't add anything that requires compiling,
  transpiling, or a dev server.
- **Render functions are dumb and idempotent.** They read from `state` and
  overwrite DOM content; they don't diff. After mutating `state.core` or
  `state.notes`, call the relevant `render*()` (or `renderAll()`) to reflect
  it — state changes are not automatically reactive.
- **All shared-state writes go through `updateCore()`** so streak
  recalculation, optimistic local update, and error-toast-on-failure stay
  consistent. Don't call `fb.setDoc`/`fb.updateDoc` directly for the core doc.
- **Firestore errors must surface to the user.** Per the existing pattern
  (see `updateCore`, `sendNote`), wrap writes in try/catch and call
  `showToast()` on failure rather than swallowing errors — a past bug class
  in this app was writes silently failing (e.g. oversized voice notes).
- **Match the existing visual language.** Use the CSS custom properties
  already defined on `:root` rather than hardcoding new colors; keep the
  warm/handwritten aesthetic (Fraunces/Libre Franklin/Caveat, soft shadows,
  rounded cards).
- **Security model is intentionally minimal.** The invite code is the only
  "auth" between partners (see README's Security note and the comments atop
  `firestore.rules`). If you change the data model, keep `firestore.rules` in
  sync — the current rules require anonymous auth, block `list` on
  `spaces` (no enumeration), and lightly validate note shape
  (`from in ['a','b']`, `ts is number`). Don't loosen these without reason.
- **Firestore's 1MiB document cap is real.** `spaces/{code}` is a single
  growing document (pings, notes-adjacent fields, etc.) — be mindful of
  unbounded arrays; voice notes already hit this limit in practice (see
  README "Known limits").

## Testing / verification

There is no automated test suite. To verify a change:

1. Fill in `FIREBASE_CONFIG` (near the top of the `<script>`, ~line 1008)
   with a real or test Firebase project's config, or use the "SETUP GATE"
   screen to confirm the gate itself still works when it's absent.
2. Open `index.html` in a browser (a local static server, e.g.
   `python3 -m http.server`, avoids any `file://` quirks with ES module
   imports).
3. Exercise the golden path manually: create a space, join it from a second
   browser/profile as the other role, and confirm realtime sync (notes,
   vibe, pings, session) shows up on both sides.
4. Check the browser console for errors — there's no linter or type checker,
   so console warnings/errors are the main signal something broke.

## Git workflow

- Default branch history is minimal (this app was scaffolded in one commit).
  Write clear, descriptive commit messages for changes.
- No CI is configured — there's nothing to pass automatically, so manual
  verification (above) before committing matters more than usual.
