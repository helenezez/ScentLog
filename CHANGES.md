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
