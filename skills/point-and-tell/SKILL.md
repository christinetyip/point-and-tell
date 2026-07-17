---
name: point-and-tell
description: Add a dev-only, in-app "point at what's wrong" feedback mode to a visual or spatial app — the user taps the broken element, the app captures its identity, geometry, and live view state as a JSON ping, and the agent reads pings back to fix bugs without reproduction guesswork. Use when the user is building something interactive with visual/spatial feedback (a game, editor, map, dashboard, AR/VR scene) and wants a feedback mode, playtest feedback tool, or in-app bug reporting, or says "point and tell".
metadata:
  author: christinetyip
  version: "1.0.0"
---

# Point-and-Tell — build spec

You are adding a **dev-only, in-app feedback mode** to this project: the user
taps any UI element or world object that looks wrong, optionally adds a note,
and the app records a **ping** — the element's identity, geometry, and live
view state at that moment. You (the agent) read pings back later and turn them
into code changes. This replaces the user describing visual/spatial bugs in
chat, which is slow and lossy.

## 0. Before building: ask the user

One short batched round of questions — these decisions vary per project and
this spec cannot make them for you:

1. **Who pings?** Just the user, or external testers? → drives identity
   capture, rate limits, and the storage choice.
2. **What input modalities does the app have?** Touch, mouse+keyboard,
   controller, VR? → drives the picker design (touch must not be the silent
   default).
3. **What feedback matters most?** Feel/timing, layout, content, correctness?
   → write the preset chips in *that* vocabulary (see the two example sets
   below).
4. **How will pings be read back?** Confirm a retrieval path exists from
   where you run — and for *whoever's agent triages*, not one person's
   machine. If you can't pull pings yourself, the loop is broken.
5. **How much now?** A solo dev testing locally may not need the trusted-side
   sanitizer on day one; a public playtest does.

**v0 in an hour:** registry + toggle + picker + chips, with accepted pings
simply printed as JSON console lines — no store; you read the app's logs.
That is a complete working loop for a solo dev (it's how the reference
project started). Add the sanitizer and session store when pings need to
outlive the console or testers go beyond the dev's own machine.

## 1. Components

Five pieces. Keep each one boring; the whole system is ~500 lines in the
reference implementation.

```
┌──────────────┐  publish(name, props)   ┌───────────────┐
│ your UI code │ ───────────────────────▶│   Registry    │  (a dictionary)
└──────────────┘   one line per element  └──────┬────────┘
                                                │ snapshot(name)
┌──────────────┐   tap → resolve target  ┌──────▼────────┐
│  PnT toggle  │ ───────────────────────▶│    Picker     │ → note sheet → ping
└──────────────┘                         └──────┬────────┘
                                                │ send
                                         ┌──────▼────────┐
                                         │  Sanitizer    │  (trusted side)
                                         └──────┬────────┘
                                                │ append
                                         ┌──────▼────────┐     pull via API/CLI
                                         │ Session store │ ◀── agent reads later
                                         └───────────────┘
```

### Registry
A module-level dictionary: `publish(name, props)` / `snapshot(name)`. UI code
publishes its latest view state under the element's name at every existing
props-compute site — one added line per element, and the UI code never knows
the feedback tool exists (passive registry: no events, no coupling). Make the
`publish()` line a convention for all future UI so coverage stays total for
free.

### Picker + PnT toggle
- A small always-visible **PnT toggle** that arms picking mode. Make it
  visually obvious (outline + fill, distinct armed state) and position it
  clear of platform chrome — system UI, notches and rounded device corners,
  browser toolbars, app nav bars. Testers must never hunt for it.
- **Dev-only is a hard gate, not a habit:** tie the toggle's existence to a
  build flag / dev-environment check that production provably fails, so
  shipping never requires remembering to remove anything. Verify the gate once
  before production — a leftover feedback tool in prod is a data-collection
  surface nobody meant to ship.
- While armed: tint the screen faintly (the mode must be unmistakable) and
  swallow app input (feedback picks must not trigger app actions). A tap/click
  resolves to the nearest registered affordance under the pointer. Adapt to
  the app's input modality (§0 q2): on desktop, a keyboard shortcut can arm
  the mode and a hover-highlight should preview the affordance before the
  click confirms the pick; on controller/VR, cursor or gaze select.
- **Echo the resolved affordance's name in the note sheet** so a mis-pick is
  caught before submit, not during triage.
- If the pick lands on a **non-registry surface** (the app's world, canvas, or
  data area): capture the *domain identity* of what was picked, at whatever
  granularity the app can name — in a game, a 3D raycast hit (object
  name/path/tags, hit point, camera pose); in a canvas editor, the hit-tested
  canvas object; in a grid or chart, the row/cell/data-point identity under
  the pointer. Identity, not screen coordinates — identity survives layout
  changes.

### Note sheet
After a pick, show one small sheet: **~6 preset chips + an optional text box**.
Presets beat typing — most feedback is one tap/click. Chips are per-domain
(§0 q3): a game might use `too slow` · `too fast` · `feels wrong` ·
`move this` · `confusing` · `love this`; a dashboard might use `wrong data` ·
`too slow` · `bad layout` · `confusing` · `missing info` · `love this`.
Include a positive chip — it tells you what not to break and keeps testers
pinging.

### Sanitizer
Never trust the client: validate on the trusted side before storage. Reference
caps (adopt as defaults): note ≤ 300 chars, snapshot depth ≤ 4, ≤ 250 total
keys, ≤ 200 chars per string value, per-user rate limit, per-session ping cap
(400). Reject with a logged reason. Also **log every accepted ping as one JSON
console line** — the log is a free second transport, visible in device logs
during live testing.

### Session store + read-back
Append pings to one document per app session, keyed sortably by time
(`session-<unix-ts>-<job-id>`). The session header carries `buildVersion` and
(optionally) `environment`. Use whatever persistence the platform has (data
store, table, JSON file, tiny endpoint) — but when more than one developer (or
a CI-run agent) will read pings, prefer a **shared store/endpoint** over one
machine's local file: the retrieval path must work for whoever's agent
triages. **Buffer and debounce writes; a failed flush is logged, never
silently dropped.** Define a retention policy — the store grows forever by
default. Write a **one-command read-back script** (list sessions;
dump one) and verify you can run it. Platform mappings, if useful: web =
overlay div + `elementFromPoint`, POST to a JSONL endpoint; mobile = overlay
view + hit test; game engine = GUI hit test + scene raycast, platform data
store.

## 2. The ping contract

These are **semantic slots, not required field names** — the table is the
contract; name each field in the host app's own vocabulary. (The JSON below
is one project's dialect: a game, where `playerName` fills the reporter slot
and a round clock fills the lifecycle slot. A dashboard's dialect might be
`userEmail` and `route + activeFilters`.)

| Slot | Contents | Why it's required |
|---|---|---|
| `affordance` | The picked element's registered name | Identifies the element by the same name the code uses |
| `note` | Chip text or typed note | The user's intent |
| `reporter` | Who filed it — whatever stable identity the app already has | Only needed once more than one person pings (§0 q1) |
| `rect` + `viewport` | Element's on-screen rectangle at pick time, plus the viewport dimensions it was measured in | Makes layout bugs provable from data; in responsive layouts a rect is meaningless without the window size |
| `vmState` | Snapshot of the element's live view state | Bugs live in the inputs to rendering, not the pixels; this is the highest-value field |
| `appClock` | In-app **lifecycle context** — a round timer in a game; the route, active filters, date range, or selected tenant in a dashboard | Answers "in what state was the app"; separates "always broken" from "broken in this state" |
| `buildVersion` | Build/deploy identifier (session header is fine; `environment` optional) | Dates every report against a build — it's how pings get marked superseded; load-bearing in the reference project, more so under continuous deployment |
| `at`, `index` | Timestamp + per-session sequence number | Ordering and cross-referencing with logs |
| `world` | Non-registry picks only: domain identity + hit point (+ camera pose where one exists) | Gives spatial/data-area bugs the same precision as UI bugs |

Reference example from a real project (illustrative, not normative):

```json
{
  "affordance": "DropItemButton",
  "note": "this button should not be at the center of my screen",
  "rect": { "x": 311, "y": 76, "w": 150, "h": 56 },
  "viewport": { "w": 832, "h": 384 },
  "vmState": { "visible": true, "enabled": true, "state": "Ready", "text": "DROP" },
  "appClock": 399.0,
  "at": 1784200954,
  "index": 1,
  "playerName": "tester-1",
  "world": null
}
```

(In the reference project the session header carries the build version and a
server job id; per-ping is equally valid.)

## 3. Your standing ritual (add to project agent instructions)

Pings rot: in the reference project a bug report sat unread for four sessions
and the tester had to hit the same bug twice. The tool is worthless without
this loop — add it to CLAUDE.md / AGENTS.md / equivalent:

1. **Triage at every session start.** List ping sessions, diff against a
   committed **triage ledger** (session-key → disposition), read anything new,
   file each ping into the issue tracker before other work, add the ledger row
   in the same commit. Many pings on the same affordance in the same build
   triage as ONE issue with a count, not N issues.
2. **Never fix silently.** Every ping becomes a tracked issue, a ruling, or an
   explicit "superseded by X" ledger entry, so the tester can always find out
   what happened to their report.
3. **Quote the ping's data in the fix.** `vmState` and `rect` are evidence —
   use them to prove the diagnosis, not re-derive it.

## 4. Acceptance

- [ ] User's answers to §0 are reflected in chips, identity, storage, scope.
- [ ] Production build provably contains no trace of the tool (§1 hard gate).
- [ ] Every existing UI element publishes to the registry; convention noted
      for future elements.
- [ ] Toggle → arm → pick → chip → stored ping works end to end **with the
      app's actual input modality, on the target device class** (§0 q2);
      non-registry picks work if the app has a world/canvas/data area.
- [ ] Stored pings carry the build version.
- [ ] Sanitizer caps + rate limit enforced; accepted pings echoed to console
      as JSON lines.
- [ ] Read-back script exists and you have successfully run it.
- [ ] Triage ritual + ledger file added to the project's agent instructions.
- [ ] Sanitizer and picker-resolution logic are platform-free and unit-tested
      headlessly.
