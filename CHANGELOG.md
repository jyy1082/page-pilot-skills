# Changelog

All notable changes to this project are documented in this file, following
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
