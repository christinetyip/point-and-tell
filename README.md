# Point-and-Tell Skill

*A Spatial Feedback Layer for Coding Agents - an installable agent skill*

**Point at what's wrong. Tell what to change. Let your agent fix it.**

## Install

Run the command for your agent. The `-g` installs machine-wide (all your
projects); drop it to install into the current project only.

**Claude Code:**

```bash
npx skills add https://github.com/christinetyip/point-and-tell --skill point-and-tell -a claude-code -g -y
```

**Codex:**

```bash
npx skills add https://github.com/christinetyip/point-and-tell --skill point-and-tell -a codex -g -y
```

**Any/most coding agents** (Cursor, Cline, Amp, Windsurf, …):

```bash
npx skills add https://github.com/christinetyip/point-and-tell --skill point-and-tell
```

Once installed, your agent applies the pattern when you're building something
interactive and ask for a feedback mode.

## The problem

When you build something visual or spatial with a coding agent — a game, an
editor, a map, a dashboard, an AR scene — the feedback loop breaks down at the
human: you see the bug instantly, but describing it in chat is slow and lossy.

> "The button — no, the other one, top-right-ish, under the timer — looks wrong
> when I'm carrying an item and the round is almost over."

You spend your testing time writing prose; the agent spends its time guessing
which element you meant and what state the app was in; reproduction becomes
archaeology.

## The fix: don't describe. Point.

Build a lightweight feedback mode into the app itself, from the start of the project. Tap
the thing that's wrong, tell what to change (a preset chip or a short note), and
the app captures everything the agent needs at that exact moment — one tap, one
**ping**, one JSON document:

```json
{
  "affordance": "DropItemButton",
  "note": "this button should not be at the center of my screen",
  "rect": { "x": 311, "y": 76, "w": 150, "h": 56 },
  "vmState": { "visible": true, "enabled": true, "state": "Ready", "text": "DROP" },
  "appClock": 399.0
}
```

The element's identity by its code-level name, its on-screen geometry, its live
view state, and when in the app's lifecycle it happened. The agent reads the
pings back later and turns them into code changes — no reproduction guesswork.

**The state snapshot is the killer field.** A screenshot shows the agent pixels;
the snapshot shows the *inputs to the rendering* — which is where bugs live.
Real example: a tester reported "the scale panel doesn't appear when I tap
Scale." The ping's state showed `sheet: "Scale", toolScaleSelected: true` — the
view-model was provably *correct*, so the bug had to be in the render layer.
One field cut the search space in half before the agent opened a single file.

## The loop

1. **Point & tell in the app.** Toggle feedback mode on, tap the thing that's
   wrong, pick a chip or type a short note. The element's identity, geometry,
   live view state, and app context are captured automatically along with your
   comment. Repeat for everything you spot.
2. **Tell your agent.** Back in the chat window, one sentence: *"I left
   point-and-tell pings."* 
3. **The agent fixes.** It pulls the pings, triages each one into a tracked
   issue, and quotes the captured state as evidence in the fix.

## Where it comes from

The pattern was extracted from a solo-dev + coding-agent mobile game where every
playtest happened on a phone, far away from the chat window. It became the
project's single highest-leverage dev tool: bug reports arrive carrying the
machine's own answer to *"which element, in what state, at which moment."*

## Does it fit your app?

One question: **when something looks wrong, would you struggle to describe
where and in what state?** If yes, install it. It earns the most in apps whose
bugs are *seen* rather than *logged* — games, editors, maps, dashboards, AR/VR,
anything drag-and-drop. It adds little to pure CLIs, APIs, and batch pipelines,
where the exact failing input and output can already be pasted into chat
losslessly.

## What's in the skill

[`skills/point-and-tell/SKILL.md`](skills/point-and-tell/SKILL.md) is the full
build spec your agent follows: the five components, the ping contract, the
questions the agent should ask you first, and the triage ritual. Prefer to
use it without installing? Hand your agent that file and say *"build this
into my app."*

## License

[MIT](LICENSE)
