# Ticket Coach Skill Pack

A set of skills for [Claude Code](https://docs.claude.com/en/docs/claude-code) and [opencode](https://opencode.ai) that turn a Jira ticket into a guided, hands-on learning session — instead of an AI that just writes the code for you, this pack acts as a senior engineering coach who walks you through it.

## What's in here

| Skill | What it does |
|---|---|
| **`ticket-coach`** | The orchestrator. Splits a ticket into phases (understand → design → break down → tests → implement → validate), guides you through each one with actions rather than code, keeps the task list in your editor's todo panel, verifies your "done" claims against the actual repo state (git diff, test runs), and only hands you real code after a genuine blocker — pseudocode first, always. Any phase except tests, implementation and validation can be skipped if you already have it covered. |
| **`explain`** | Explains concepts in plain language with real-world analogies before any technical detail. Used by `ticket-coach` for its first phase (understanding the ticket), and usable on its own any time you want something explained. |
| **`cognitive-profile`** | Maintains a living profile of *how* you learn best (not what you know) — built from a short onboarding questionnaire, refined over time from behavioural evidence. Other skills read it to adapt explanations, never to gatekeep them. You never see or edit raw scores directly. |
| **`learning-trend`** | Read-only. Shows how your ticket-coach performance has evolved over time — only activates when you ask for it (e.g. "how am I improving?", "have I gotten better at Go?"). |
| **`brainstorming-committee`** | Three subagents with different perspectives (Pragmatist, Architect, Advocate) independently deliberate on design decisions for a ticket and converge on a plan. Used by `ticket-coach` as the engine behind its design phase — you're never shown the committee's plan directly, it's used as a private reference to check your own design thinking against. |
| **`design-review`** | A holistic second-pass review of the committee's consolidated design (completeness, consistency, clarity, scope, YAGNI) before it's accepted as the reference plan. Invoked automatically by `brainstorming-committee`, not something you call directly. |

## How they fit together

```
You: "Coach me through PROJ-42"
        │
        ▼
┌────────────────┐  reads/writes   ┌────────────────────┐
│  ticket-coach   │◄───────────────►│  cognitive-profile  │
│  (orchestrator) │                 │  (how you learn)    │
└───────┬─────────┘                 └──────────┬──────────┘
        │                                       │
        │ Phase 1: Understand                  │ scorecards.jsonl
        ▼                                       │ (per-ticket signal)
┌────────────────┐                              │
│    explain      │                             │
└───────┬─────────┘                             │
        │                                       ▼
        │ Phase 2: Design                ┌──────────────────┐
        ▼                                │  learning-trend   │
┌──────────────────────┐                 │  (you ask: "how   │
│ brainstorming-       │                 │   am I doing?")   │
│ committee             │                 └──────────────────┘
│  ├─ Pragmatist        │
│  ├─ Architect         │
│  └─ Advocate          │
│        │              │
│        ▼              │
│  design-review        │
└──────────┬────────────┘
           │ reference plan (never shown to you directly)
           ▼
   Phases 3-6: Break down → Tests → Implement → Validate
   (ticket-coach guides directly, no further delegation;
    the task list is mirrored into your editor's todo panel)
```

`ticket-coach` is the only skill that talks to all the others. `explain` and `brainstorming-committee`/`design-review` are delegated to for specific phases and hand their result back; `cognitive-profile` is consulted for personalisation and is the only thing that ever writes profile data; `learning-trend` only activates when you explicitly ask to see your progress.

## Installing

Each skill is a folder. Drop the folders you want into your skills directory:

- **Claude Code**: `~/.claude/skills/`
- **opencode**: `~/.config/opencode/skills/`

So, for example, `ticket-coach/SKILL.md` ends up at `~/.config/opencode/skills/ticket-coach/SKILL.md` (or the Claude Code equivalent), keeping the rest of that skill's files alongside it.

`_shared/subagent-rules.md` is used by `brainstorming-committee` and `design-review` — keep it one level up from both, i.e. install it as a sibling of those two skill folders (`~/.config/opencode/skills/_shared/subagent-rules.md`), not inside either one.

`agents/coach.md` is **not** a skill and doesn't go in the skills folder. On opencode, copy it to `~/.config/opencode/agents/coach.md`. See "The Coach agent" below for why it matters.

### The Coach agent (opencode)

`ticket-coach` refuses to run outside the `coach` agent. Starting a coaching session from `build` or `plan` gets you a hard stop and a message telling you to press **Tab** and switch — it won't read the ticket, create a session file, or start a phase.

That's deliberate rather than fussy. The coach agent carries the permissions the pedagogy depends on: read-only on the project repo (so the "coach" can't quietly write your code for you), the `question` permission that makes choices render as a selectable list instead of free text, and write access to the session and profile data outside the repo. Run the skill from `build` and you get something that looks like coaching but isn't — the agent can edit the repo, choices degrade to typed answers, and every data write hits a permission prompt. A half-started session from the wrong agent is worse than none, because it leaves a session file behind that a later coach session will resume as though it were sound.

The agent identifies itself to the skill with a marker line in its prompt (`TICKET_COACH_AGENT_MARKER: coach-v1`). If you rename the agent or write your own, keep that line.

**On Claude Code** there are no switchable primary agents, so there's nothing to enforce: the gate detects this and skips itself silently. Everything else in the pack works the same way.

The model in `agents/coach.md` is set to `openai/gpt-5.6` as a default — change it to whatever you use. Coaching leans on judgement rather than throughput (reading a diff, deciding whether an answer actually landed, choosing how hard to hint), so a reasoning-capable model earns its keep here.

### Minimum viable install

You don't need all six. Reasonable subsets:

- **Just want the coaching loop, nothing fancy**: `ticket-coach` + `explain` on their own. Phase 2 (design) falls back to `ticket-coach`'s own step-by-step guidance if `brainstorming-committee` isn't installed.
- **Add personalisation**: + `cognitive-profile` (+ `learning-trend` if you want to see progress over time later).
- **Add the design committee**: + `brainstorming-committee` + `design-review` + `_shared/subagent-rules.md`. These two depend on each other and on the shared rules file; install all three together. They also depend on your environment supporting subagent dispatch (Claude Code's `task` tool, or opencode's equivalent) — without that, skip these two and `ticket-coach` will fall back gracefully.

On opencode, install `agents/coach.md` whichever subset you pick — `ticket-coach` won't run without it.

One optional integration isn't shipped here: if you happen to have a `requesting-code-review` skill installed, `ticket-coach` uses it for the quality-review pass after each task is verified. Without it, it forms the same judgement itself.

### Permissions (opencode)

If you installed `agents/coach.md`, most of this is already handled — the agent carries its own permissions, including the `question` and `todowrite` grants the coaching loop needs. The block below still matters for the data directories, since those are read and written by skills that may also run outside the coach agent.

`cognitive-profile` and `ticket-coach` write to data directories outside your project folder (`~/.config/opencode/data/...`), which opencode treats as `external_directory` and prompts for approval on by default. Each of those two skills ships a `permissions-snippet.json` with the exact block to add to your `opencode.json` — copy both in:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "external_directory": {
      "~/.config/opencode/data/cognitive-profile/**": "allow",
      "~/.config/opencode/data/ticket-coach/**": "allow"
    }
  }
}
```

(If you already have a `permission` block, merge the `external_directory` entries into it rather than replacing the whole thing.)

Claude Code doesn't have this prompt-per-path system, so no equivalent step is needed there.

## `cognitive-profile`'s data files

`cognitive-profile` manages two data files of its own, kept outside this repo entirely (in `~/.config/opencode/data/cognitive-profile/`, regenerated the first time you use it):

- `profile.json` — the profile itself, created from a short onboarding questionnaire, refined automatically over time from how you actually work.
- `scorecards.jsonl` — one line per ticket completed via `ticket-coach`, feeding both the profile's automatic updates and `learning-trend`.

A third file lives in the same directory but belongs to `ticket-coach`, not to `cognitive-profile`:

- `coaching-notes.jsonl` — free-form memos about *how to coach you*, written and read by `ticket-coach` alone: pacing preferences, constraints in your setup, things you've clearly stopped needing explained, phases you tend to skip. It exists because the per-ticket session file is deleted at close-out, so anything learned about you would otherwise vanish with it.

The separation is deliberate. Scores are governed, confirmed with you before they change, and schema-validated; coaching notes are working memory for one skill and never influence a score. `cognitive-profile` never reads them.

You'll never see raw scores from either file — `cognitive-profile` deliberately avoids surfacing them, since a number like "concreteness: 4.5/5" implies more precision than a couple of 1–5 questionnaire answers actually support. Here's a trimmed example of what `profile.json` looks like, for reference (not something you ever need to write yourself):

```json
{
  "engineer_id": "your-id",
  "schema_version": "1.0",
  "last_updated_date": "2026-06-22",

  "cognitive_profile": {
    "concreteness": { "score": 4, "label": "concrete-leaning", "confidence": "self-reported" },
    "abstraction_bridge": { "score": 5, "label": "analogical-leaning", "confidence": "self-reported" }
  },

  "cognitive_weights": {
    "theory_first": 0.25,
    "code_examples_first": 0.75,
    "technical_precision": 0.0,
    "real_world_analogies": 1.0
  },

  "background_context": {
    "is_english_native": false,
    "known_technologies": [
      { "name": "Go", "type": "language", "proficiency": "intermediate", "proficiency_confidence": 0.3, "context": "Temporal" }
    ],
    "seniority_level": "senior",
    "seniority_level_confidence": 0.15
  },

  "update_history": [
    { "trigger": "initial_creation", "method": "full_questionnaire", "date": "2026-06-22", "changes": null }
  ]
}
```

`proficiency_confidence` and `seniority_level_confidence` are accumulators, not scores you set — `cognitive-profile` nudges them up (+0.15) or down (-0.10) after each completed ticket based on how strong that ticket's scorecard signal was, and fires the next upgrade once one crosses 0.70 (then resets it to 0). This means a genuine improving streak isn't lost just because it happens to land awkwardly between arbitrary checks, and a single off ticket only dents progress rather than wiping it out.

The full schema (all five axes, all ten weights, every enum) lives in `cognitive-profile/schema.json`.

## Typical session

```
You: Coach me through PROJ-42, I want to learn this, don't give me code unless I'm really stuck.

ticket-coach: [checks it's running as the `coach` agent — if not, hard stop]
              [reads your ticket, checks for a saved session, none found]
              [reads your cognitive-profile and coaching notes if they exist]
              Splits PROJ-42 into phases: Understand → Design → Break down
              into tasks → Write tests → Implement → Validate.

              "If you've already got any of these covered, say so and I'll
              skip it — tests, implementation and validation we'll always
              work through."

              Phase 1 — hands off to `explain`, which explains the ticket in
              plain language with an analogy, calibrated to how you learn.

              "Once that makes sense: tell me in your own words what you
              think needs to change."

You: [explains it back]

ticket-coach: Good. Phase 2 — runs `brainstorming-committee` quietly in the
              background (Pragmatist/Architect/Advocate debate, then
              `design-review`), gets back an approved internal plan, saves it
              to the session file, never shows it to you.

              "How do you want to approach the design — figure it out
              yourself and I'll check it after, want me to guide you toward
              it step by step, or just hand you the plan directly?"

You: I'll try it myself.

ticket-coach: [you propose an approach; it diverges from the internal plan on
              one edge case]

              "Are you sure about that? What happens if the response comes
              back empty?"

You: [iterate, eventually land close enough to the reference plan]

ticket-coach: Phase 3 — "Now break it into small tasks: which ones, and
              which files does each touch?" The agreed list is persisted and
              mirrored into your editor's todo panel, ticking over as you go.

              Phases 4 onwards — tests first, then implementation, one task
              at a time. Each "done" claim checked against actual git diff /
              test runs before the checklist moves; pseudocode before real
              code, real code only after a genuine blocker.

              [final phase: runs `just api check` and `just api test`, both
              pass] Ticket closed automatically — scorecard logged quietly to
              cognitive-profile, session file deleted, no prompting needed.

You (any time later): How's my Go coming along?

learning-trend: [reads the scorecard history] "Looking at your last 5 Go
                tickets, code_without_help has gone from averaging around 3
                to consistently landing at 5 — PROJ-40 was the last time you
                needed real code rather than pseudocode."
```

## A note on how these are written

These are instruction-following skills for an LLM agent, not enforced code. The rules inside (never show the committee's plan, verify before trusting a "done" claim, pseudocode before real code, and so on) are followed because the model reads and applies them each time, not because anything mechanically blocks deviation. In practice this works well, but in long sessions it's worth occasionally checking that the behaviour described here still matches what you're actually seeing.