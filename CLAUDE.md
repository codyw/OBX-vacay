# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

A private, single-file web app: a trip planner/itinerary for one specific family
vacation — **"Morse · OBX 2026"**, an Outer Banks, North Carolina trip running
**Saturday Jul 11 – Sunday Jul 26, 2026**. It is not a general product or
framework; the content (dates, addresses, confirmation numbers, activity ideas)
is specific to this trip and gets hand-edited as plans firm up.

The whole app is designed to be added to an iPhone home screen (apple-touch-icon
generated at runtime, safe-area insets, `-webkit-` touch tuning) and used as a
lightweight offline-ish reference during the trip.

## Repository structure

There is exactly **one file**: `index.html` (~2000 lines). There is no
`package.json`, no build tool, no test suite, no linter, and no CI config.
Git history shows edits made via GitHub's "upload files" web flow rather than
a local dev loop — treat this as a hand-edited static file, not a scaffolded
project.

`index.html` has three parts, in order:

1. **`<style>` (top of file)** — a full design-token system in `:root`, then
   component CSS, organized in commented `───` banner sections (TOKENS, NAV,
   CALENDAR PAGE, DRIVE PAGE, HOUSE PAGE, IDEAS PAGE, EAT & BOOK PAGE,
   REFERENCE PAGE, etc.). Each banner corresponds to one page/feature below.
2. **`<body>`** — a sticky `<nav>` with a brand name and a native `<select>`
   styled as a dropdown page-picker, a countdown `.strip`, and a `<main>`
   containing five `<section class="page" data-page="...">` blocks: `trip`,
   `calendar`, `ideas`, `eat`, `reference`. Exactly one page has class
   `.active` at a time; the rest are `display: none`.
3. **`<script>` (bottom of file)** — vanilla JS, no dependencies, organized in
   the same banner-comment style: home-screen icon generation, the token gate,
   the countdown ticker, calendar rendering, page switching, ideas sub-tab
   filtering, the collapsible house-details toggle, and swipe gestures.

## Development workflow

There's no install or build step. Edit `index.html` directly and open it in a
browser (or serve it with any static file server) to preview changes. Because
everything lives in one file, keep changes self-contained there rather than
introducing new files/tooling unless the user explicitly asks for a
restructure (e.g., splitting CSS/JS out, adding a bundler).

## The token gate — security-sensitive, read before touching

Near the top of the `<script>` block:

```js
const EXPECTED_HASH = '86e67b91a6366f1ec7d4b39fa4a9d620924f1e60b2839dbb3593a2f80cb898e4';
```

The page starts `<body class="locked">` (nav/strip/app hidden). On load, JS
reads a `?k=` query param, SHA-256-hashes it in the browser, and only removes
`.locked` if the hash matches `EXPECTED_HASH`. **The real token never lives in
source** — it only exists in the bookmark URL the family uses
(`.../index.html?k=<token>`).

- Never commit an actual token value, only its hash.
- To generate a new token/hash pair, use the one-liner already in the code
  comment directly above `EXPECTED_HASH`:
  ```
  node -e "const c=require('crypto'); const t=c.randomBytes(24).toString('hex'); console.log('token:',t,'hash:',c.createHash('sha256').update(t).digest('hex'));"
  ```
  Put the resulting hash in `EXPECTED_HASH` and give the resulting token/URL
  to the user out-of-band (chat reply) — don't write the token itself into
  the repo.
- This is a client-side obscurity gate, not real access control (the full
  markup ships to anyone with the URL, hash included) — treat it as
  "keep it off search engines / casual stumbling," not as protecting secrets.

## Trip data & the most common edits

These are the places a change to trip logistics or content actually needs to
happen:

- **Trip-wide date constants** (`<script>`, "CONSTANTS" section): `TARGET`
  (departure instant, drives the countdown), `TRIP_S` / `TRIP_E` (trip date
  range, drives the "Day N / 14" badge), and separately `CAL_START` in the
  "CALENDAR RENDER" section (first day shown in the calendar grid). The hero
  text (`.hero-dates`), the countdown strip text (`.dates`), and page-foot
  copy also hardcode the same dates in the markup. **If the trip dates ever
  change, update all of these together** — there is no single source of truth
  for the date range.
- **`DAY_DATA`** (object keyed by day-index-from-`CAL_START`, e.g. `0`, `1`,
  `15`) — drives the small emoji/text items shown on each day in the calendar
  grid. Add an entry here for any day that gets a new confirmed plan.
- **Per-page content blocks** — each page's markup follows a repeating card
  pattern; match the existing shape rather than inventing new structure:
  - Trip page: `.journey-card` (a leg of the drive, with `.jc-row` time/detail
    rows and a `.jc-maps` Apple Maps deep link) and the collapsible
    `.house-details` block for the rental house.
  - Ideas page: `.act` cards inside `.cat` groups (one `.cat[data-cat=...]`
    per category, filtered by the `.subtabs` buttons — a new category needs a
    matching subtab button `data-filter` + `.cat data-cat` pair), and
    `.group` cards under `.groupings` for suggested activity pairings.
  - Eat & Book page: `.eat-cat` cards of `<ul class="eat-list">` entries.
  - Reference page: `.ref-card` / `.ref-row` key-value rows, `.rainy-list`,
    and `.phone-list` / `.phone-row` (tap-to-call/email/maps links).

## Styling conventions

- Design tokens are CSS custom properties in `:root`: a sand/cream/paper
  neutral scale, `--navy`/`--teal`/`--gold`/`--coral`/`--sage` accent colors,
  and two font stacks — `--serif` (Fraunces, for display/headings/italic
  accents) and `--sans` (DM Sans, for UI/labels/body). Reuse existing tokens
  rather than hardcoding new colors/fonts.
- Component classes are kebab-case with a short per-component prefix (`.jc-*`
  for journey-card, `.ref-*` for ref-card, `.day-*` for calendar days, etc.).
  Follow this prefix convention for any new component.
- This is a mobile-first, iOS-tuned page: preserve `env(safe-area-inset-*)`
  padding, `dvh` units, `-webkit-tap-highlight-color`, and
  `touch-action: manipulation` when touching layout-affecting CSS.

## Adding a new page

1. Add an `<option>` to `#page-select` in the nav.
2. Add the page's key to the `PAGES` array and a label in `PAGE_LABELS`
   (`<script>`, "CONSTANTS" section) — this drives swipe navigation order and
   the dropdown label sync.
3. Add a new `<section class="page" data-page="yourkey">` in `<main>`,
   following the `.section-head` (num/title/sub) + content pattern used by
   the existing pages.
4. Style new components under a new banner-commented block in `<style>`,
   reusing existing tokens and the card patterns above.

## What's absent (don't guess)

There is no README, no automated tests, no linter, no CI, and no deployment
config in this repo. If asked how/where this is hosted or deployed, say that
isn't defined in-repo rather than assuming a host (e.g., GitHub Pages,
Netlify) — the user hosts/shares this file some other way not captured here.
