# Changes

Tracking decisions made under the PRD's "latitude to improve" clause (§6.5),
plus a log of what's shipped so far. App lives in `index.html` — still a
single self-contained file, no build step, no dependencies.

## This pass: light/dark mode (§2.4) + Phase 1, data durability (§3, fixes D1)

### Light/dark mode

- Kept the v1 dark palette as the `:root` default and added a
  `@media (prefers-color-scheme: light)` override block. No manual toggle
  (per §2.4.5) — it follows the OS setting only.
- The light palette is a deliberate redesign, not an inversion. Gold and
  wine were both darkened/desaturated until text set in them cleared
  4.5:1 against the light card/background (verified programmatically,
  not eyeballed): gold `#dba85f → #8a5a1c`, wine `#ea6b86 → #a52f52`,
  sage `#93ab84 → #4f7a4a`.
- Split the single `--on-accent` token into `--on-gold` and `--on-wine`.
  A gold/wine dark enough to work as *light-mode text* is too dark for
  the old near-black `on-accent` to read against as a *button fill* —
  so light mode flips those to near-white while dark mode keeps them
  near-black, matching the PRD's "check per accent, not globally" note.
- `--line` (borders) went from `#322c29` to `#9c8d7c` in light mode —
  the original value inverted to near-invisible (contrast ~1.4:1) on
  white. New value clears 3:1 against both card and background.
- Added `--gold-border`, `--wine-border`, `--gold-soft`, `--focus-ring`,
  and `--ink-tint` tokens so every remaining hardcoded accent-tinted
  `rgba(...)` in the stylesheet (wishlist dashed border, profile-tag
  borders, the input focus ring, section-icon chips) resolves per-mode
  instead of silently keeping its dark-mode color in light mode.
- `apple-mobile-web-app-status-bar-style` changed from
  `black-translucent` to `default`. Translucent assumes the page under
  the status bar is reliably dark; with a light theme that would put
  white status-bar icons on a white page. iOS doesn't support a
  per-scheme value for this tag, so `default` is the safer static
  choice.
- Manifest `background_color`/`theme_color` stay dark (`#121012`). Same
  one-value-only constraint — a brief dark splash before first paint
  reads as less jarring than a bright white one, so dark fails more
  gracefully.
- Added two `<meta name="theme-color">` tags (one per `media` query)
  instead of the old single static tag, so browser/PWA chrome matches
  the active mode.
- Contrast pairs were checked with a script (WCAG relative-luminance
  formula), not by eye — every text/background and border pair listed
  above clears its 4.5:1 / 3:1 threshold in both modes.

### Phase 1 — data durability

- **Export**: "Export my data" bundles all four `localStorage` keys
  into one JSON object (`{ schemaVersion, exportedAt, data }`) and
  downloads it as `scent-log-backup-YYYY-MM-DD.json` via
  `Blob` + `URL.createObjectURL` + a programmatic anchor click, per the
  PRD's iOS Safari note.
- **Import**: "Import backup" opens a `.json` file picker, validates
  shape (schemaVersion present, the three array keys are arrays, the
  library key is a plain object) before touching anything, shows the
  current-vs-backup counts in a confirmation, and only overwrites on
  explicit confirm. Malformed input leaves existing data untouched and
  shows an inline error instead of throwing.
- Used the native `confirm()` dialog for the import confirmation rather
  than a custom modal — it works inside an installed iOS PWA and keeps
  the app dependency-free; a styled modal was more surface area than
  the decision needed.
- **Storage warning**: a one-time dismissible banner, shown on first
  load (state tracked in `localStorage`), explains data is device-only.
  Where `navigator.storage.estimate()` is available it's used to show
  current usage; where it isn't, the banner still shows with the
  default copy rather than being skipped.
- **Backup reminder**: an inline prompt in the Data section appears
  when the last export is >30 days old (or has never happened) and
  there are more than 10 total records (entries + collection +
  wishlist). Exporting clears the condition immediately.
- Added a `scent-meta` localStorage key (`{ schemaVersion }`) set once
  on first load if absent, so future migrations have a version to
  branch on locally — separate from the backup file's own
  `schemaVersion` field, which versions the *export format*, not the
  live data.
- Verified in Chromium (both color schemes): banner display + dismiss
  persistence, export producing a well-formed download, invalid-JSON
  import safely rejected, and a valid import correctly restoring data.

### Not in this pass (superseded — see next section)

Phases 2–6 from the PRD (checkpoint notifications, preference
intelligence/signed weighting, bottle management, context capture,
usability debt — search/filter/edit/undo) are unbuilt. Shipping them
independently per §3's instruction, not batched into this change.

## Next pass: v2 shell — new IA, palette, and Phases 2/3/5/6 (partial)

Implemented the "Scent Log v2 Shell" design (Claude Design handoff),
direction chosen: **warm amber palette, sharpened treatment, tab-bar
navigation**, dark side using the handoff's revised-dark variant. This
replaces the single-scroll v1 layout and light/dark palette from the
previous pass — the whole app was rebuilt around it, still as one
self-contained `index.html`.

### What changed structurally

- **Navigation**: sticky header + 4-tab bottom bar (Test / Shelf /
  Insights / Data), replacing the single long scroll. Section order
  within each tab follows the handoff's reasoning: the two questions
  people actually open the app for ("I need to log a checkpoint" /
  "what do I wear") come before the profile, which is a payoff to read
  rather than a task to do.
- **Palette**: full re-pick per scheme rather than reuse, per the
  handoff — dark: bg `#0c0a0c`, gold `#e0ab5e`, wine `#d2687c`, sage
  `#7f9873`; light: bg `#f4eee3` (warm paper, not white), gold
  `#8c5c16`, wine `#ab3a56`, sage `#4c6a3e`. Re-verified every
  text/fill/border pairing against WCAG AA with the same contrast
  script as the previous pass — all clear 4.5:1 (text) / 3:1 (borders,
  UI components) in both modes. Elevation in light mode comes from a
  border + soft shadow instead of a lightness step, since a
  near-white card on a warm-white page would otherwise disappear.
- **Schema v2**: `gravitate` → `reach` (still 1–10, now a 10-segment
  tap bar instead of a slider — no drag needed, no live-badge/resort
  dance). Collection notes restructured from prose-only checkpoints to
  `{ notes: {top, heart, base}, prose: {cp0..cp6h} }`, matching PRD
  §3.3. **The structured arrays are derived at load time from the
  existing prose with the same note-vocabulary matcher the profile
  already used (`extractNotes`) — never hand-authored.** This is true
  for both the built-in ~71-fragrance library and any of a user's own
  saved custom entries, so nothing here can introduce a note that
  wasn't already verified in the original checkpoint text.
- **Migration**: idempotent, gated on `scent-meta.schemaVersion`,
  reshapes the prior collection/entries/wishlist in place —
  `gravitate` copies straight to `reach`; each old entry's four
  checkpoint fields plus its free-text notes are concatenated into one
  labeled `dry` string (nothing is dropped, just consolidated to match
  the new entry shape); `hours` is backfilled from the old longevity
  band's midpoint since prior entries never stored a precise duration.
  Nothing is deleted or overwritten before the new shape is written.

### Fixes to gaps in the source shell

The shell is explicitly a navigation/visual pass (its own CHANGES notes
edit sheets, a working import, notifications, fill projection, sample
tracking, and the layering log as deliberately unbuilt). Implementing
it as a real app surfaced a few gaps that would have made shipped
sections permanently inert or, worse, silently fabricated data — fixed
rather than carried forward:

- **Finishing a live test now actually files an entry.** In the shell,
  "Finish" just cleared the active test with a toast; the checkpoint
  work was never rolled into the Tested log, which contradicts PRD
  §2.2 directly. Finish now computes real elapsed hours, bands them
  into a longevity tier, joins the recorded checkpoint notes into the
  entry's dry-down text, and pushes a real entry — with undo.
- **Weather is entered, not hardcoded.** The shell's `onStartTest`
  always stored the same fixed 21°C/58% on every test. Shipping that
  unchanged would have quietly fabricated data feeding the
  longevity-by-weather chart — the one feature that most depends on
  weather being real. Start Test now has two optional manual number
  fields (temperature, humidity) instead; blank stays `null` rather
  than a guess, and the weather chart correctly excludes entries
  without a reading.
- **My shelf and Passed on can actually be added to.** The shell has no
  add affordance for either — only the seeded sample rows exist, and a
  real user would have no way to ever add a bottle or file a pass. Added
  a one-line "Add a bottle you own" input (matching the pattern already
  used for Want to try) and a name + reason-chip row for Passed on, so
  those sections work for someone starting from zero.
- **Kept the already-working Export/Import from the previous pass**
  instead of the shell's stub (which only ever reports counts and
  says "replacing is not wired up"). Validate-before-overwrite,
  confirmation with current-vs-backup counts, and the storage
  warning banner / backup reminder all carried forward, updated for
  the new schema and the five keys the export now bundles
  (`collection`, `entries`, `wishlist`, `passed`, `activeTest`).

### What's real vs. honest-empty-state

Matching the shell's own restraint rather than overbuilding past it:

- **Recommended** stays a from-data explanation of why it has nothing
  to show (signal-bottle count, shared-note overlap across rated
  bottles, verified-library size) rather than a working recommendation
  list. The previous v1 build did surface real matches from the
  built-in library; this intentionally reverts to the honest-blocker
  version per the handoff's own critique of D4 (recommending mostly
  bottles the user already named has low discovery value, and it's
  a worse outcome than admitting the feature isn't there yet).
- **What should I wear** ranks by season/occasion tag match first,
  reach only breaks ties, and states plainly when nothing is tagged
  rather than silently falling back to a reach-only order.
- Profile bars are suppressed (in favor of a flat, honestly-labeled
  list) whenever every note would render as the same-length bar, so a
  one-bottle "profile" can't imply a ranking that isn't there.
- Fill level, cost-per-wear, and season/occasion tags render only when
  set — since no edit sheet exists yet to set them, real shelf items
  show the same honest "add a price to see cost per wear" states the
  seeded ones would.

### Explicitly not built (unchanged from the shell, deferred)

~~Edit sheets for existing entries/bottles (§6.1), a note editor for
unverified bottles~~ — built in the next pass, see below. Push
notifications for checkpoints (§2.3 — overdue checkpoints still
surface in-app on open, just not proactively), fill-depletion
projection (§4.3, editing the fill level itself is now possible, the
depletion-date estimate from fill history over time is not),
sample/decide-by tracking (§4.4), the layering log (§5.3), and
note-pairing detection (§3.4, needs ≥3 supporting bottles) remain
deferred. Each is a reasonable next independent phase.

## Fix: autocomplete regression from the v2 rebuild

v1 had live autocomplete against the full ~71-fragrance verified
library when typing a name into any add flow. The v2 tab-bar rebuild
dropped this from **Add a bottle you own** and **Want to try** — typing
a name did nothing until you hit Add, then it silently matched (or
didn't) in the background. Restored: both inputs now show a live
dropdown of matching name/house pairs as you type (`searchLibrary`,
name matches ranked above house-only matches), and picking one fills
the field. The Start Test suggestion list — which the v2 design
intentionally scoped to bottles already on the shelf or wishlist — now
also folds in full-library matches, so typing a fragrance you don't
own yet still surfaces it instead of coming up empty.

## Next pass: edit sheets (§6.1)

Filled the gap flagged above and in the v2 shell pass itself (which
explicitly shipped every "Edit" affordance as a toast stub).

- **Bottle edit** (My shelf → expand → Edit bottle): house,
  concentration, family, price paid, size (ml), fill level (%),
  season and occasion tags — all editable inline, saved back to
  `scent-collection`. This is what actually unlocks cost-per-wear,
  the fill bar, and tag-based ranking in **What should I wear** for a
  real bottle a user adds themselves, none of which had any input path
  before this pass.
- **Notes stay split by provenance.** A bottle matched to the built-in
  library keeps its verified `top`/`heart`/`base` notes locked — the
  edit form says so and points at the actual way to change it (remove
  and re-add under a different name) rather than silently allowing an
  edit that would drift from the verified source. An **unverified**
  bottle gets three plain comma-separated inputs (top/heart/base)
  instead; whatever the user types is tagged `notesSource: 'user'`,
  which keeps it editable on every later visit — once entered, a
  user's own words are never re-locked. Nothing here can insert
  invented notes into a verified library entry.
- **Entry edit** (Tested → Edit): date, site, longevity band, hours,
  temperature, humidity, and the dry-down text are all editable in
  place, so a typo or a forgotten weather reading doesn't mean
  delete-and-relog.
- Chip toggles (season/occasion) inside an open edit form sync the
  surrounding text fields from the DOM before re-rendering, so
  clicking a season chip can't silently discard whatever was just
  typed into price/size/fill/notes above it — verified in both color
  schemes with a Playwright pass that types into a field, toggles two
  chips, and confirms the typed value survived before saving.

## Phase 2 — checkpoint notifications (§2, fixes D5)

Active test state (§2.1) and the live test card (§2.2) were already in
place from the v2 shell pass. This phase is specifically §2.3:
notifications, plus the fallback the PRD calls out as load-bearing.

**How the iOS constraint was actually handled.** PRD §2.2 flags two
separate problems, and they need two separate answers, not one:

1. *No Notification API at all* outside an installed home-screen PWA
   on iOS Safari — `typeof Notification === 'undefined'` in a plain
   tab. Detected once via `notificationStatus()` and treated as its
   own state (`'unsupported'`), not lumped in with "denied."
2. *Unreliable delivery while the app is closed*, even once installed
   — a `setTimeout` armed at minute 0 has no way to fire at minute 30
   if the tab or app process isn't alive to run it.

For (1), the app never assumes the API exists — every call site checks
`notificationStatus()` first. For (2), the fix isn't a better
scheduling trick (there isn't one available to a plain web app); it's
routing around the assumption that a *push notification* is what
tells the user a checkpoint is due. What actually tells them is
**timestamp math against `activeTest.startedAt`**, computed fresh
every time `checkpointRows()` runs (on any render, `state.activeTest`
in hand) — a checkpoint is `overdue` the moment `now > dueAt`, full
stop, no timer or permission involved. That computation already
existed before this phase (it's what puts the "N due" badge in the
header and marks a checkpoint "Missed. Record it now, approximate is
fine." in the live test card) and needed no changes — it's what
`fireCheckpointDue()` piggybacks on rather than replacing.

**What's new is the in-session layer on top:**
- `Notification` permission is requested from `ensureNotificationPermission()`,
  called only from the start-test action — never on load, never on
  reopen, matching §2.3 exactly. `Notification.permission` only reads
  `'default'` before a first answer, so this is a no-op prompt (no
  browser dialog) on every start after the first.
- On start, and again on app load if a test is already active (a
  fresh page load has no memory of the previous session's timers),
  `scheduleCheckpointTimers()` arms one `setTimeout` per checkpoint
  that's still genuinely in the future — never for one already
  overdue, since that's the fallback's job, not a timer's. Recording a
  checkpoint early, or finishing the test, invalidates or clears the
  relevant timers so a stale one can't fire after the fact.
- When a timer fires: if permission is `'granted'`, show a real
  `Notification`; independent of that, always re-render the header
  badge and (if the Test tab is open) the live card, so the UI catches
  up the instant a checkpoint comes due even if the OS notification is
  unavailable or the user has the tab focused and wouldn't see an OS
  banner anyway.
- **Not silent on denied/unsupported**: a one-time toast ("Notifications
  are blocked / not available here — you'll still see missed
  checkpoints marked overdue when you reopen the app") fires the first
  time either state is hit, gated by a localStorage flag so it doesn't
  nag on every subsequent test. The live test card's own status line
  is never static copy — it reads real `Notification.permission` on
  every render, so "Notifications are on" / "blocked" / "not available
  in this browser" always matches what's actually true.

Verified in Chromium with the permission API mocked both ways: granted
(3 timers arm on a fresh test; a due, unrecorded checkpoint fires a
real `Notification`; an early-recorded one does not; Finish clears all
timers) and denied (one-time toast, correct card copy, no repeat toast
on a second test). Separately verified the fallback needs none of this
machinery: a test seeded 3 hours in the past with two checkpoints past
due shows the header badge and both "Missed" rows correctly on a
fresh page load, with zero timers armed for the two already-overdue
ones and exactly one for the checkpoint still genuinely ahead.

## Phase 3 — preference intelligence (fixes D2, D3)

Three of the four sub-requirements turned out to already exist, built
during the v2 shell pass (which front-loaded parts of several PRD
phases). This pass verified those for real rather than trusting the
earlier build note, fixed one real defect in the migration, and built
the one genuinely missing piece.

**3.1 signed weighting — already correct, now verified live, not just
read.** `weightForReach()` already matched the PRD formula exactly
(1-3 → -(4-reach), 4-6 → 0, 7-10 → reach-6) and the profile already
rendered both "You gravitate toward" and "You tend to avoid." Built a
Playwright case that rates a real collection item (Delina) 2/10 and
confirms its notes (iris, lychee, rhubarb, rose, peony) land in
`avoids` with negative weight and **none** of them leak into `likes`
— the failure mode a naive implementation could still have even with
the right formula, if the two lists were built independently instead
of from one signed sum per note.

**3.2 dislike bucket — already correct.** Passed on (the PRD's "Tried
and passed") already existed with the six reason chips, and
`buildProfile()` already applies a fixed -2 to a passed item's notes.
Verified live: filing a passed-on fragrance with real library notes
measurably pushes those notes into `avoids`. Passed items were never
readable by the collection-only Recommended blockers logic, so "excluded
from recommendations permanently" was already structurally true.

**3.3 structured notes + migration — verified, and one real defect
fixed.** Structured `{top, heart, base}` notes derived from prose via
the vocabulary matcher already existed for both the library and every
migrated collection item. What didn't hold up: `migrateIfNeeded()`
branched on `scent-meta.schemaVersion` **and** a secondary check that
inspected the collection's actual shape (`c[0].reach !== undefined`)
as a defensive backstop. That's exactly the "guessing at shape"
this pass was told not to do, and it wasn't just redundant — it was a
latent data-loss bug: if `schemaVersion` were ever already current
while a shape-check somehow disagreed, the shape-check would win and
skip writing the version tag straight, masking the real signal. Now
`migrateIfNeeded()` branches on `schemaVersion` alone. This is safe
because `schemaVersion` is *only* ever written by this same function
(unconditionally, every call) or by a successful import — both of
which only fire once the data is actually in the current shape — so
by construction the flag can't be stale relative to the data as long
as nothing outside this file edits `localStorage` directly.

Verified: (1) a v1-shaped collection (`gravitate`, no `reach`) migrates
correctly and picks up real structured notes; (2) `scent-custom-library`
is read during migration but never written or deleted, so a user's
existing custom entries survive untouched; (3) a fragrance matching a
custom-library entry gets its *real* prose-derived notes when added to
the shelf, not fabricated ones; (4) idempotency — booting fresh against
already-migrated, hand-edited data (`reach` changed from a UI action)
does not re-run the transform and does not reset the edit. (Testing
this with a plain page reload initially gave a false failure: Playwright
re-fires its own `addInitScript` fixture on every navigation, which
re-seeded the old v1 data underneath an already-`schemaVersion: 2` meta
flag — a test-harness artifact, not an app bug. The real test boots a
fresh browser context directly from a snapshot of already-migrated
storage, which is what "reopening the app later" actually looks like.)

**3.4 note pairing detection — built for real.** This was the one
placeholder: the Pairings card always showed static "not enough yet"
copy regardless of data. `computeNotePairings()` now looks only at
positively-weighted (reach ≥ 7) bottles — "high-rated" per the PRD's
own phrasing — counts every note pair's co-occurrence across them, and
surfaces a pair once **at least 3** distinct bottles carry both notes,
exactly the PRD's threshold. It goes one step further than a bare
count: the PRD's own example phrasing ("less so on its own") is a
*comparison* claim, not just a co-occurrence one, so for each
qualifying pair it also checks bottles that carry the first note
*without* the second — if those average a lower weight than the
paired bottles, it earns that phrasing; if every bottle with the first
note also has the second, it says so plainly instead of implying a
comparison that isn't there; otherwise it states the pairing without
overclaiming. Verified against the actual verified library (not
synthetic data): adding all 71 built-in fragrances at reach 9 finds
real qualifying pairs (musk+vanilla, amber+musk, etc.) and the
Insights tab renders the computed sentence in place of the
placeholder.

Not touched: Phase 4, per instruction.

## Library import/export (Data section)

Added a second import/export pair to the Data tab, alongside the
existing backup import/export, for the verified fragrance *library*
specifically (house/concentration/family/checkpoints/longevity/source
per fragrance) rather than a user's own bottles, tests, and ratings.
The two are deliberately kept separate and behave differently on
purpose:

- **Backup import** (existing) replaces the whole app state wholesale —
  it's a restore, not a merge.
- **Library import** (new) merges into `scent-custom-library` by `key`:
  an existing key gets its fields overwritten with the imported row,
  a new key gets added, and every key not present in the imported file
  is left exactly as it was. Nothing about a user's own collection,
  tests, wishlist, or passed-on list is touched by this at all.

**Accepted formats.** Both `.json` (`{schemaVersion, library: [...]}`,
matching the attached format guide/export shape) and `.csv` (column
headers matching the guide's field names, case-insensitive, any
order) are accepted. Format is detected from the file extension, with
a fallback content sniff (does the trimmed text start with `{`) for a
misnamed file.

**A real CSV parser, not `split(',')`.** Several checkpoint fields in
the attached library data have commas inside a single field (e.g.
`"Warm, skin-like amber and cedar, less sweet than the opening"` for
Baccarat Rouge 540's `cp2h`) — a naive comma-split corrupts these into
extra phantom columns and shifts every field after them. Writing a
minimal CSV state machine by hand (`parseCSV()`) was the right call
here over pulling in a parsing library: the grammar this app actually
needs is small and fixed (comma-delimited, double-quote-quoted fields,
`""` as an escaped quote inside a quoted field, `\r\n` or `\n` line
endings) and the PRD's hard constraint is zero runtime dependencies in
a single HTML file, so a dependency was never on the table regardless
of complexity. The parser is a straightforward character-by-character
state machine (in-quotes vs. not, per RFC 4180) that builds rows of
raw string cells; a second pass (`csvRowsToObjects()`) turns those
into field-keyed objects using the header row, so column order in the
source file doesn't matter as long as the required column names are
present.

**Validate everything before writing anything.** `validateLibraryRows()`
runs entirely in memory against the full parsed set — no row is
written to `localStorage` until every row has been checked — then
`handleLibraryImportFile()` performs exactly one `localStorage.setItem`
merging every valid row into the existing custom library object at
once. This means a mid-import crash or a bad row 60 rows in can't ever
leave the stored library half-updated. Per row, a required column
being empty (`key`, `name`, `house`, `conc`, `family`, all four
checkpoints, `longevity`, `source`), a `family` not matching one of
the 13 values in `VALID_FAMILIES`, a `longevity` not matching one of
the 4 bands in `VALID_LONGEVITY`, or a `key` repeated elsewhere in the
same import file (checked against keys seen earlier in the same file,
not against what's already stored — those are legitimate updates) all
disqualify that one row without aborting the rest. A CSV missing a
required column entirely is checked before any row parsing and aborts
the whole import (there's no per-row recovery from a column that
doesn't exist).

**Reporting.** Every rejected row is reported by position, not
silently dropped: `"Imported 34 entries. 3 rows skipped: row 12
invalid family, row 19 missing house, row 40 duplicate key."` — the
exact pattern asked for. CSV positions are true spreadsheet row
numbers (header is row 1, so the first data row is row 2); JSON
positions are reported as `entry N` (1-indexed into the `library`
array) since a JSON array has no header row to offset against, and
calling it a "row" there would be misleading.

**Malformed file: change nothing, say so.** Unparseable JSON syntax,
a JSON body without a `library` array, or a CSV with fewer than 2
rows or a missing required column all stop before touching storage
and report the problem — same pattern as the existing backup import's
"that doesn't look like a Scent Log backup" handling. This is
different from the bad-row case above: a structurally malformed file
can't be partially trusted, but a well-formed file with some invalid
rows can still deliver its valid rows.

**The `source` field.** Added to the library schema
(`structureLibraryEntry()`, both `BUILTIN_LIBRARY` and
`scent-custom-library`) so it round-trips through import and export
untouched, per the format guide's provenance requirement (never trust
a note pyramid whose origin can't be pointed to). It is not surfaced
anywhere in the UI, exactly as asked — it exists purely so a future
export carries forward where each entry's data actually came from.
`BUILTIN_LIBRARY`'s 71 entries predate this field and don't carry it
inline; `buildLibrary()` defaults them to `'v1 built-in (verified)'`
at load time so an export of an untouched install still reproduces
the same shape as the attached reference file, with every entry
correctly attributed.

**Display names for imported entries.** Existing custom-library
entries had no path to a properly-cased display name — the app's
`DISPLAY_NAMES` map is hand-authored only for the 71 built-in
fragrances, and every other lookup fell back to the raw (lowercase)
storage key. A newly imported fragrance would otherwise show up as
`new custom scent` instead of `New Custom Scent` everywhere its name
is displayed. `buildLibrary()` now also writes `DISPLAY_NAMES[key] =
custom[key].name` for every custom-library entry it loads, so this
works automatically for anything that comes in through the new
import — not scope creep, just what "merge in real data" requires to
actually render correctly.

**Library export** (`doLibraryExport()`) writes the current combined
library (built-ins + custom, by key) back out as
`{schemaVersion, library: [...]}` in the exact shape of the attached
reference file — verified directly against it: round-tripping the
71-entry file back out reproduces every field, including `source`.

Verified with Playwright against the attached 71-entry JSON and a CSV
generated from the same data (every field quoted, matching a real
spreadsheet export): full-file import both ways, a merge that updates
one existing key's checkpoint text and adds one brand-new key while
leaving 70 others untouched, a mixed-validity file (2 good rows, 1
each of missing-field / bad-family / bad-longevity / duplicate-key,
reported individually), a syntactically-broken JSON file, a
structurally-wrong JSON file (no `library` array), and a CSV missing
a required column — all behave as specified, and the export round
trip reproduces `source` correctly for both built-in-default and
freshly-imported entries. Screenshots confirm the new Library card
renders correctly in both light and dark mode, including the error
state.

## Phase 4 — bottle management

Implements PRD 4.1–4.4 (`Phase 4 is done when: a bottle with a price
and three logged wears displays an accurate cost per wear`) in full.
4.1's price/size/fill and 4.2's cost-per-wear already existed from
earlier work on the bottle edit sheet; this pass added the remaining
4.1 fields, and built 4.3 and 4.4 from scratch.

**4.1 extended fields.** Added `purchaseDate` and `batchCode` to the
collection item schema and the bottle edit sheet, alongside the
existing price/size/fill row. Neither is surfaced anywhere but the
edit sheet — the PRD doesn't ask for them on the card, and there's no
room on it that isn't already earning its place.

**4.2 cost per wear.** Verified rather than rebuilt: `price / wears`
(wears = count of Tested-log entries referencing the bottle) already
renders on the shelf card once both are present, exactly as specced.

**4.3 fill tracking + depletion estimate.** Every time a saved fill
value actually changes, a `{date, fill}` point is appended to the
item's `fillHistory` (same-day edits collapse into one point, so
re-checking a bottle twice in one sitting can't distort the trend).
`estimateDepletion()` fits a linear regression across every recorded
point (not just first/last, so one off reading doesn't swing the
result), projects forward to where the line crosses 0%, and returns a
date plus days-from-now — or `null` when there are fewer than two
points, or when the trend isn't actually declining (a fill that went
up, e.g. a corrected typo, must never produce a countdown). Bottles
projected to empty within 60 days get a "running low" badge inline
on the shelf card (colors the fill bar and text danger-red) and are
also rolled up into a standalone callout above the ranked list —
tested that a flat or single-point history correctly produces no
estimate, and that a genuinely declining history produces one in a
sane range.

**4.4 sample tracking.** Collection items gained `isSample` and an
optional `decideBy` date. Samples are a parallel item type sharing
the same storage array and fields (so a sample can still be tested,
rated, and carry real notes toward the profile) but are filtered out
of the ranked "My shelf" list and the "What should I wear" picks —
those are for bottles already committed to, not a decant still being
evaluated. They get their own "Samples" section instead: sorted
overdue-first, each with an editable decide-by date, and a nudge
("Past your decide-by date. Log a test or pass on it.") plus a
danger-colored border once that date has passed, matching the PRD's
wording almost verbatim. "Log a test" hands off to the Test tab the
same way Want to try's "Test it" does; "Add to shelf" clears
`isSample` and the item joins the ranked list; "Pass on it" moves it
into Passed on with an undo toast, reusing the same removal pattern
as everywhere else in the app.

Verified with Playwright: purchase date/batch code round-trip through
the edit sheet; fill history accumulates correctly across edits and
feeds a depletion estimate that lands in the expected range, while a
flat or single-reading history correctly abstains; a sample is
excluded from both the ranked shelf list and What should I wear but
still shows up correctly once converted; the overdue nudge renders;
Pass on it and Add to shelf both mutate state correctly; and the
existing backup export/import carries every new field through
untouched, since both already serialize the collection generically.
Screenshots confirm the Running Low callout, Samples section
(including the overdue state), and the extended edit sheet all render
correctly in both color schemes.

## Fix: longevity missing from shelf card, observed-vs-library value

Longevity was technically still on the card (buried in the meta line,
abbreviated, mixed in with house/conc/family) but not in any sense the
core, at-a-glance stat the PRD frames it as, which is what "no longer
shows longevity" meant in practice. Three changes:

1. **Longevity is now its own prominent line on the card face** — full
   band text ("All day (10+ hrs)", not the abbreviated "All day"),
   color-coded (sage/wine/gold/faint by band, reusing the same color
   scheme the Tested log already used for its entry cards — extracted
   that into a shared `longevityColor()` instead of leaving two copies
   of the same ternary to drift). Removed from the house/conc/family
   meta line, where it read as one fact among four instead of the
   headline.

2. **Cost per wear moved into the expanded detail view.** Still
   computed the same way (`price / wears`), just no longer competing
   with longevity for the same line on the collapsed card.

3. **Observed longevity overrides the library value once a bottle has
   logged tests.** `observedLongevity()` averages the `hours` field
   across every Tested-log entry that references the bottle (hours
   rather than each entry's own longevity band, since hours is the
   more precise number and the band is just `bandFromHours()` applied
   to it — averaging hours first and reducing once avoids compounding
   rounding across multiple already-rounded bands) and reduces that
   back to a band with the same `bandFromHours()` used everywhere
   else. Falls back to the library's stated value when there are no
   tests yet. The distinction is visible via a small pill next to the
   value reading "observed" (filled sage) or "library" (neutral), and
   the expanded detail spells out the actual numbers ("Longevity above
   is observed from 3 logged tests, averaging 11.0h. The library
   states All day (10+ hrs).") so the override is never a surprise.

Verified with Playwright: a bottle with logged tests shows the
"observed" tag and a band that reflects its actual average hours
(tested with a case where the observed band differs from the
library's, so the override is unambiguous); a bottle with no tests
shows "library" and the library's stated value; cost-per-wear no
longer renders on the collapsed card and does render in the expanded
detail. Screenshots confirm both states in both color schemes.

## Fix: rating a bottle below 4 with nothing liked yet hid the avoids

`buildProfile()` was computing `avoids` correctly the whole time — the
bug was purely in `render_Profile()`'s gate: `hasProfile = signalBottles
>= 1 && likes.length > 0` required at least one *liked* bottle to exist
before rendering any profile content at all, avoids included. Rate a
single bottle below 4 with nothing yet rated above 6, and the real,
correctly-computed avoids list sat behind the generic "rate two
bottles" placeholder instead of showing.

Changed the gate to `signalBottles >= 1 && (likes.length > 0 ||
avoids.length > 0)` — either side is a real profile on its own — and
gave "You gravitate toward" a matching empty state ("Nothing rated
above 6 yet...") for the mirror case, instead of falling through to
text that assumed at least one like existed.

Verified with Playwright: a single bottle rated 2/10 (nothing liked)
now shows "You tend to avoid" with its real notes and the correct
empty state on the likes side; the reverse (liked-only) and
both-present cases were re-checked to confirm neither regressed.
Screenshot confirms the avoids-only state renders correctly in dark
mode.

## Phase 5.1 — weather

Implements PRD 5.1 in full. This is the one deliberate exception to
the app's "must work fully offline" rule (PRD §6), so every failure
mode falls back to manual entry rather than blocking anything.

**Capture.** A "Use my location" button next to the temp/humidity
fields on the live-test start screen calls the Geolocation API, then
fetches current conditions from Open-Meteo
(`api.open-meteo.com/v1/forecast`) — chosen because it needs no API
key, is CORS-enabled for direct browser calls, and has no paid tier to
worry about at this volume, which matters for a single static HTML
file with no backend to hide a key behind. Every failure mode
degrades to manual entry with a visible message instead of a silent
stall: no Geolocation support, denied permission, a timeout, or a
network/API failure. A field the fetch filled shows "From your
location just now" until the user edits either field by hand, at
which point the attribution clears — editing after a fetch is always
an active choice to override it, so the label needs to disappear
immediately, not just internally; the temp/humidity inputs were added
to the same live-re-render allowlist the search and name inputs
already use so that happens without losing focus.

**Store + display.** `weatherSource: 'auto' | null` travels with the
active test through to the finished entry (cleared automatically if
the entry's temperature is later hand-edited, same reasoning as
above) and renders as a small "gps" tag next to the reading, both on
the live test card and in the Tested log — the same source-tag
pattern already established for observed-vs-library longevity.

**Profile note, per fragrance.** The existing aggregate "Longevity by
weather" chart only ever compared cool vs. hot across every logged
test combined. PRD 5.1 also asks for this per fragrance ("note when
*a fragrance's* longevity varies meaningfully by temperature band"),
which the aggregate chart can't surface — a bottle's own swing can
get diluted into the average. `fragranceWeatherNotes()` reuses the
same `tempBandsFor()` band logic per bottle instead of across
everything, holds itself to the same bar the aggregate insight
already uses (at least 2 tests in both the cool and hot band before
claiming a difference), and lists up to 5 fragrances with the largest
swings under a "By fragrance" heading in the same card.

Verified with Playwright, mocking both the Geolocation permission and
the Open-Meteo response (route interception) so the test is
deterministic regardless of the sandbox's own network access: a
successful fetch fills both fields and carries `weatherSource: 'auto'`
through to the finished entry and its Tested-log tag; hand-editing a
fetched value clears the attribution live; a denied-permission context
shows the fallback message without breaking manual entry; and a
bottle with 2 cool + 2 hot logged tests produces the expected
per-fragrance variance sentence. Screenshots confirm the fetch,
active-test, error, and per-fragrance states in both color schemes.

## Phase 5.2, 5.3, 6.1, 6.2 — remaining PRD phases

Closes out everything left in the PRD's phased roadmap: the rest of
context capture (5.2, 5.3) and the rest of usability debt (6.1, 6.2).

**5.2 — season/occasion tags on entries.** Collection items already
had these; Tested-log entries didn't, so a wear couldn't be tagged
independently of the bottle it came from (you might wear the same
fragrance differently in different seasons). Added `seasons`/
`occasions` arrays to entries, editable via the same chip-row pattern
already used on the bottle edit sheet, shown as tags on the collapsed
entry card, and wired into the new season filter in 6.2 below.

**5.3 — layering log.** A genuinely new record type, not a variant of
a Tested-log entry: two or more fragrance references, a rating (reuses
the same 1-10 segment picker as reach, for visual consistency), and
notes. Kept separate from `state.entries` because forcing a
combination into the single-fragrance entry shape would corrupt every
per-fragrance calculation that assumes `ref` points at exactly one
bottle (observed longevity, weather variance, cost per wear). Lives at
the bottom of the Test tab, its natural home next to the Tested log.
References resolve to collection items by name at save time, same
pattern as everywhere else names get linked; a combo logged before a
bottle was added still displays and still matches once the bottle
shows up, since surfacing also falls back to a case-insensitive name
match. Editing is scoped to rating and notes — re-picking the
fragrances isn't supported, since fixing a rating or a typo covers the
realistic edit case without a multi-name-editing UI (see the "one deep
insight beats twelve shallow ones" tradeoff below). "Surface successful
combinations on the relevant collection cards" (rating 7+, the same
positive threshold `weightForReach()` uses elsewhere) renders as a
"Layers well with X (rated Y/10)" line in the shelf card's expanded
detail, alongside cost-per-wear and the observed-longevity
explanation.

**6.1 — wishlist edit path.** The last of the three record types PRD
6.1 names (`entry, collection item, and wishlist item`) without an
edit path. Tapping a wishlist item's name/note now opens an inline
edit form (name + note, Save/Cancel) in place of the row, matching how
every other edit-in-place flow in the app already works.

**6.2 — Tested log search/filter upgrade.** Text search already
covered "filter by fragrance"; the longevity-band chips already
existed. Added the rest behind a "More filters" disclosure (collapsed
by default, so the already-dense Tested log header doesn't get
permanently busier, with a `•` marker when a hidden filter is active
and a "Clear these filters" link once one is): a family filter
(`entryFamily()` resolves a family the same way `houseFor()` resolves
a house — prefer the linked collection item, fall back to a library
lookup by name), the new per-entry season tags from 5.2, a date range,
and a sort control (date newest-first, the existing default, or
longevity longest-first).

Verified with Playwright across all four: entry season/occasion tags
save, display, and correctly narrow the Tested log when filtered;
family/season/date-range filters each narrow to the expected subset
and "Clear these filters" resets them; sorting by longevity reorders
correctly; a layering combo saves with resolved refs, displays in the
log, surfaces on the linked shelf card's detail, stops surfacing the
moment its rating is edited below 7, and removes with a working undo;
the add-slot control caps at 5 and the remove-slot control floors at
2; wishlist editing saves a new name and note. Screenshots confirm the
filters panel, tagged entry, entry edit chips, layering log (list and
add form), the shelf-card surfacing line, and the wishlist edit form,
all in both color schemes.

This closes every phase in the PRD's roadmap (Phases 1 through 6).

## Fix: iOS Safari auto-zoom on every input focus

iOS Safari zooms the whole page in whenever a focused form field's
font-size computes under 16px, and doesn't zoom back out on blur —
the base `input, textarea, select` rule was `font-size: 12.5px`,
so this fired on essentially every field in the app.

Raised that rule to `font-size: 16px` and dropped vertical padding
from `10px` to `8px` with `line-height: 1.3` to keep fields close to
their previous visual height rather than shrinking the font back down
to compensate, per the instruction to use padding/line-height as the
lever, never font-size. Also found and fixed the one field with an
inline override bypassing the shared rule — the per-sample decide-by
date input on the Shelf tab's Samples section, which had its own
`font-size: 11.5px` — down to matching padding/line-height for the
same reason.

Added `maximum-scale=1, user-scalable=no` to the viewport meta tag as
the second half of the fix, so even a field that somehow computes
under 16px in the future can't trigger a runaway zoom.

Checked every input, select, and textarea in the app for stray
font-size overrides before concluding the shared rule plus the one
inline fix was complete coverage — not just the obvious name/search
fields, but the autocomplete inputs (shelf/wishlist/sample/layering
name fields), the checkpoint note textarea, every edit-sheet field
(bottle, entry, wishlist, layering), the Tested log's new filter
controls (family/season/date-range/sort), and the weather temp/
humidity fields. Verified programmatically with Playwright — swept
computed `font-size` across every form field on all four tabs, with
edit sheets and the filters panel expanded — rather than trusting the
CSS rule alone, since an inline style already once quietly bypassed
it. Screenshots confirm the layout still holds at the larger size in
the app's tightest multi-column rows (the bottle edit sheet's price/
size/fill and purchase-date/batch-code rows) in both color schemes;
a couple of longer placeholders (e.g. "Humidity % (optional)") now
truncate a bit more in narrow fields, an acceptable, expected
side effect of prioritizing the font-size fix over placeholder length,
exactly as instructed.
