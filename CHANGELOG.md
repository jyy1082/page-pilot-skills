# Changelog

All notable changes to this project are documented in this file, following
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.2.0]

### Fixed
- `buildSkillDraft` overwrote a `check` step's `checked` boolean with a
  literal `"{{name}}"` *string* placeholder — silently corrupting its
  type. Any code (like `PagePilot.check()`) expecting a real boolean would
  have received a truthy string instead, regardless of what value was
  actually meant. Now uses a separate `checkedParam` marker property,
  leaving `checked` as the originally recorded boolean (a safe fallback if
  a value isn't provided when filling the skill back in later).

### Added
- `fillSkillParameters(skill, values)` — substitutes real values back into
  a saved skill's `{{name}}` placeholders (and `checkedParam` markers),
  returning a fresh steps array ready to hand to `PagePilot.run()`. A
  missing value leaves the literal `{{name}}` text in place rather than
  silently blanking it out (much easier to notice something's wrong), and
  a missing checked value falls back to what was originally recorded.
  Never mutates the skill passed in. This is what actually makes a saved
  skill usable again — `buildSkillDraft`/`saveSkill` alone only get you as
  far as storing one.
- 6 new real-browser tests, including a full round trip: record → save as
  a skill → fill with values different from what was originally recorded
  → replay through a real `PagePilot.run()` → confirm the new values (not
  the recorded ones) actually land in the page.

## [0.1.0] — Initial release

### Added
- `detectParameters(steps)`: scans a page-pilot-recorder step array for
  `type`/`select`/`check` values worth turning into named parameters,
  suggesting a human-readable name for each by inspecting the field's
  `<label>` (both `for=` and wrapping), `aria-label`, `placeholder`, and
  `name` attribute, in that priority order. Long values (>200 chars) and
  checkbox/radio states default to unchecked (suggested as fixed, not
  parameterized); select values default checked.
- `hasFragileSteps(steps)` / `isHighRisk(steps)`: heuristic checks for a
  structural-path-fallback selector and for a click matching a common
  dangerous-action word (delete, submit, pay, transfer, etc., in English
  and Chinese), used to pre-fill (not enforce) warnings in the panel.
- `buildSkillDraft(description, steps, acceptedParams)`: builds a draft
  skill with each accepted parameter's value replaced by a `{{name}}`
  placeholder, working on a copy — never mutates the steps array passed in.
- `saveSkill` / `listSkills` / `getSkill` / `deleteSkill`: a small
  `localStorage`-backed storage API, scoped per domain
  (`location.hostname` by default). `saveSkill` strips everything but a
  parameter's `name` before writing to storage — example values are never
  persisted, deliberately, even if a draft object happens to carry them.
- `showArchivePanel(steps, options?)`: the full review UI — task
  description, detected parameter candidates (editable name, checkbox),
  the step list (each individually removable, for dropping recorded
  noise), a fragile-selector warning, and a high-risk checkbox
  (pre-checked based on `isHighRisk`, always overridable). Resolves with
  the saved record on "Save as skill" (saving itself, callers don't need
  to call `saveSkill` separately) or `null` on "One-time use". Marked
  with `data-ppr-ignore` so page-pilot-recorder never records interactions
  with the panel itself as part of a session.
- A real-browser test suite (`test/browser-test.mjs`, `npm test`, 40
  cases) covering label-priority detection across every supported hint
  type, the storage round trip, per-domain scoping, and the full archive
  panel flow including renaming a parameter, deleting a step, and both
  the save and skip paths.
