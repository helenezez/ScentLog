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

### Not in this pass

Phases 2–6 from the PRD (checkpoint notifications, preference
intelligence/signed weighting, bottle management, context capture,
usability debt — search/filter/edit/undo) are unbuilt. Shipping them
independently per §3's instruction, not batched into this change.
