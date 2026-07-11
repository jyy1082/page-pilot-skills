# page-pilot-skills

[中文](./README.zh-CN.md) · **English**

**Version 0.2.0** · see [CHANGELOG.md](./CHANGELOG.md) for release history

Turns a [page-pilot-recorder](https://github.com/jyy1082/page-pilot-recorder)
step array into a reusable, parameterized "skill": a short description, a
list of named parameters, and the original steps with concrete values
swapped for `{{parameter}}` placeholders — so the same recording can be
run again later with different values, instead of being locked to
whatever was typed during recording.

This is step 1 of a larger plan: recording and tagging now (this repo);
AI-driven retrieval and automatic parameter fill-in on top of it later.
**Nothing here talks to an AI at all** — it's a review UI a person uses
right after stopping a recording, plus a small local storage API. A
person still picks what becomes a parameter, still names it, still
confirms before saving.

## Demo

Record something with [page-pilot-recorder](https://github.com/jyy1082/page-pilot-recorder),
then instead of running or copying the steps directly, show the archive
panel:

```js
import { PagePilotRecorder } from 'page-pilot-recorder'
import { showArchivePanel } from 'page-pilot-skills'

const recorder = new PagePilotRecorder()
recorder.start()
// ...person interacts with the page...
const steps = recorder.stop()

const skill = await showArchivePanel(steps)
// A panel appears: task description, detected parameter candidates (with
// suggested names, checked/unchecked), the step list (each removable),
// and a high-risk checkbox. Picking "Save as skill" saves it (this
// function does the saving itself) and resolves with the saved record;
// picking "One-time use" resolves with null and saves nothing.
```

Later, to actually use a saved skill again with new values:

```js
import { listSkills, fillSkillParameters } from 'page-pilot-skills'
import { PagePilot } from 'page-pilot'

const skill = listSkills()[0] // or however you pick which one
const steps = fillSkillParameters(skill, { 'Last Name': 'Tanaka', 'Department': 'Engineering' })

const cursor = new PagePilot()
await cursor.run(steps)
```

## What gets detected as a parameter candidate

Every `type` step's `text`, every `select` step's `value`, and every
`check` step's `checked` state — each with a suggested human-readable name
tried in this order: a `<label for="...">` pointing at the field, a
wrapping `<label>`, `aria-label`, `placeholder`, then the `name` attribute.
If none of those are found, the person gets a generic "参数N" placeholder
name and has to name it themselves before it can be saved as a parameter.

- `select` values are suggested **checked** by default (usually the whole
  point of re-running a flow with different inputs).
- `check` (checkbox/radio) values are suggested **unchecked** by default —
  usually a fixed part of the flow, not something worth re-parameterizing.
- Values longer than 200 characters are suggested **unchecked** by
  default — more likely free-form text (a note, a description) that
  should stay fixed per run, not a value someone would swap out each time.

Password fields never appear here at all — page-pilot-recorder already
refuses to record them, so there's nothing to detect in the first place.

## What gets saved, and what deliberately doesn't

A saved skill record looks like:

```json
{
  "id": "skill_1720000000000_ab12cd",
  "domain": "admin.example.com",
  "description": "Add a new employee",
  "steps": [
    { "type": "click", "target": "#add-btn" },
    { "type": "type", "target": "#lastName", "text": "{{Last Name}}" },
    { "type": "select", "target": "#department", "value": "{{Department}}" }
  ],
  "parameters": [{ "name": "Last Name" }, { "name": "Department" }],
  "fragile": false,
  "highRisk": false,
  "createdAt": "2026-...",
  "updatedAt": "2026-..."
}
```

- **Example values are never persisted** — only parameter *names*. Even if
  a draft object happens to carry an example value alongside a parameter,
  `saveSkill()` strips everything but the name before writing to storage.
  This is deliberate: names, ids, and other real values typed during
  recording shouldn't end up sitting in `localStorage` indefinitely just
  because they were used as an example once.
- **Skills are scoped per domain** (`location.hostname` by default) — a
  skill saved on one site never shows up when working on another.
- **`fragile`** is set if any step's selector had to fall back to a
  structural path (see page-pilot-recorder's selector generation) — a
  heads-up to review before relying on it long-term, not a reason it
  can't be saved.
- **`highRisk`** starts pre-checked in the panel if any click's target
  text or selector matches a common dangerous-action word (delete,
  submit, pay, transfer, etc. — in English and Chinese) — the person can
  always override the checkbox either way before saving. This is a
  heuristic nudge, not a guarantee; nothing in this repo enforces
  anything based on it (that belongs to whatever executes the skill
  later, e.g. requiring a confirmation before running a high-risk one).

## API

| Function | Description |
|---|---|
| `detectParameters(steps)` | Scan a step array, return candidate parameters with suggested names |
| `hasFragileSteps(steps)` | True if any step's selector is a structural-path fallback |
| `isHighRisk(steps)` | True if any click's target/option looks like a dangerous action |
| `buildSkillDraft(description, steps, acceptedParams)` | Build a draft skill object with placeholders substituted in |
| `saveSkill(draft, domain?)` | Save a draft (or update an existing one by id); strips everything but parameter names |
| `listSkills(domain?)` | List saved skills for a domain, most recently updated first |
| `getSkill(id, domain?)` | Get a single skill by id |
| `deleteSkill(id, domain?)` | Delete a skill by id |
| `fillSkillParameters(skill, values)` | Substitute real values back into a saved skill's placeholders, returning steps ready for `PagePilot.run()` |
| `showArchivePanel(steps, options?)` | The full review UI; saves itself and resolves with the saved record, or `null` for "one-time use" |

`domain` defaults to `location.hostname` everywhere it's an optional
parameter.

## What this does NOT do (by design, for now)

- **No AI.** Nothing here calls out to any model. Detected parameter
  names are suggestions from DOM inspection, not language understanding;
  `fillSkillParameters` does plain string substitution with values you
  provide — it doesn't figure out what those values should be.
- **No retrieval/matching.** There's no "find the skill that matches this
  new instruction" — that's the next layer, built on top of what this
  repo stores.
- **No natural-language parameter extraction.** `fillSkillParameters`
  handles substituting values back in once you have them (e.g. from a
  form someone filled in), but turning a sentence like "add Jane Tanaka to
  Engineering" into `{ "Last Name": "Tanaka", "Department": "Engineering" }`
  is the next layer's job, not this one's.
- **No cross-device sync.** Storage is plain `localStorage`, scoped to
  the browser it was saved in.

## Testing

```bash
npm install
npm test
```

Runs a real-browser suite (Playwright + Chromium via `@sparticuz/chromium`
— see [page-pilot-recorder's README](https://github.com/jyy1082/page-pilot-recorder#testing)
for why that specific detour exists), covering parameter detection's label
priority order, the storage round trip, domain scoping, and the full
archive panel UI flow (renaming a parameter, deleting a noise step, the
high-risk checkbox, and both the "save" and "one-time use" paths).

## License

MIT
