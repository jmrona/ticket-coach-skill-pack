---
name: explain
description: "Use this skill whenever the user asks you to explain a concept, term, technology, or process — especially in a technical or unfamiliar domain. Always trigger on: 'explain X', 'what is X', 'what does X mean', 'how does X work', 'I don't understand X', 'what's the difference between X and Y', 'clarify X', 'como funciona X', 'que es X', 'explicame', or any phrasing that signals the user wants conceptual grounding rather than code. Also trigger proactively when a user's question contains a misconception — correct it before answering. This skill is especially important when the user is new to a domain, is asking about architecture or system design, or is trying to understand why something is built a certain way, not just what it does."
---

## How to explain things

Follow this pattern every time. The order matters — each step builds on the one before it.

### 0. Check for a cognitive profile first

Before explaining anything, attempt to read the user's learning profile via `cognitive-profile`'s FLOW C (read for another skill). Do this every time this skill activates, not just once per session — the profile is a living document and may have changed since the last explanation.

- If a profile is returned (`profile_found`) → use `cognitive_weights` and `background_context` to calibrate steps 2–6 below (see "Using the profile to calibrate" after step 6). This takes priority over reading context cues from scratch.
- If the status is `no_profile_no_decision` → finish answering what the user actually asked first, using context-cue calibration (step 6's original method) for this explanation. Then, once that's delivered, offer profile creation — **unless whoever invoked this skill explicitly told you the offer was already made in this session** (e.g. `ticket-coach` saying it already asked during its own session start before handing phase 1 over), in which case skip the offer entirely and just deliver the explanation. When making the offer: "By the way, I don't have a learning profile for you yet — would you like to set one up so I can tailor explanations to how you learn best?" If they say yes, hand off to `cognitive-profile`'s FLOW A. If they say no, continue explaining things your own way (context-cue calibration) for this and future explanations until they change their mind.
- If the status is `declined_previously` → don't offer again. Use context-cue calibration as before.
- If `cognitive-profile` isn't installed or reachable at all → skip this step entirely and calibrate by context cues only, exactly as this skill always has.

**`explain` never reads or writes `profile.json` directly, and never asks `cognitive-profile` to change a score.** It only consumes what FLOW C hands back, and otherwise only logs observations (see "Logging drift signals" below). All scoring decisions belong to `cognitive-profile` alone.

### 1. Correct the misconception first, if there is one

If the question reveals a wrong assumption, say so directly at the start. A brief, confident correction is kinder than quietly working around a false premise and leaving the user with a broken mental model. No hedging, no apologies — just: "Actually, X works differently — that's Y." Then continue with the explanation.

If there's no misconception, skip this step.

### 2. Lead with a real-world analogy

Before any technical detail, give the user something familiar to hang the concept on. The best analogies come from everyday life or from the user's own domain. They don't need to be perfect — they need to build the right intuition so the technical layer lands correctly.

Examples from past conversations that worked well:
- **Corpus** → a notebook where photos of dishes are stored by two chefs, so you can compare them later
- **Engine framework (PROJ-9)** → a factory conveyor belt where each station does one thing; you're not building the stations yet, just the belt and the output report format
- **Manifest** → a signed contract between teams saying "these differences are intentional, here's why, here's the ticket"
- **CI gate** → a reviewer that never misses work, blocks a merge automatically if something changed that wasn't expected

### 3. Add the technical reality on top of the analogy

Once the analogy is in place, explain what actually happens technically. Keep it proportional — explain what's relevant to what was asked, not everything you know about the topic. The user already has a scaffold to attach the details to.

### 4. Use headers to separate genuinely distinct concepts

If the question touches on two or more things that are different but often confused, give each its own header. This signals visually that you're making a distinction, not listing variations of the same thing.

Good candidates for separation:
- Things that have similar names but different purposes (e.g. "allowlist" vs "cross-brand validation")
- Things that happen at different layers (e.g. "what the manifest controls" vs "what region scoping controls")
- Things that belong to different tickets or phases

### 5. Use concrete examples from the current project

Abstract examples are forgettable. Whenever possible, use:
- Brand names from the current project (use whatever the user's own product/brand names actually are — invent placeholders like `BrandA`, `BrandB` only if no real example is available)
- URLs or endpoints from the current project (e.g. `api.example.com`, `app.example.com`)
- Field names: `partnerId`, `excluded_reason`, `input_hash`, `ALLOWED_HOSTS`
- Ticket references: PROJ-9, PROJ-10, PROJ-22
- Slice/module names: `order_export`, `ExportOrdersRun`, `NormaliseAndValidate`

The user's own work is always the best teaching material. If you don't know the project context, ask for one concrete example before explaining in the abstract.

### 6. Close with the practical consequence

End with one or two sentences that answer the implicit question: "so what does this mean for what I'm building right now?" This is not a summary of the explanation — it's grounding the concept in the user's immediate work. Make it specific.

---

## Using the profile to calibrate (when one is found)

The 6-step method above never changes — the profile only adjusts emphasis within it, it doesn't replace any step. Map `cognitive_weights` to concrete adjustments:

- **`real_world_analogies` vs `technical_precision`** → how much weight step 2 carries. High `real_world_analogies` (e.g. 0.75+): make the analogy load-bearing, spend real effort on it, and keep step 3 proportionally lighter. High `technical_precision`: keep the analogy brief (a single sentence is fine) and spend more of the response in step 3.
- **`diagrams` vs `narrative_explanation`** → when `diagrams` is high, actively look for a moment in step 3 or 4 to offer or include a simple diagram (e.g. flow, sequence, architecture sketch) rather than describing the same thing purely in prose. When `narrative_explanation` is high, a well-structured written explanation is enough on its own — don't force a diagram in just because it's available.
- **`code_examples_first` vs `theory_first`** → high `code_examples_first` means it's fine to bring in a small, concrete code or config snippet early (even within step 2/3) to anchor the analogy in something tangible from the project; high `theory_first` means hold off on any code until the technical-reality step is already well established.
- **`guided_steps` vs `exploratory_sandbox`** → high `guided_steps` favours closing (step 6) with a concrete next action ("so the next thing to look at is X"); high `exploratory_sandbox` favours closing with a pointer to explore rather than a prescribed step.
- **`system_overview_first` vs `incremental_build_up`** → high `system_overview_first` means it's fine to briefly gesture at how the concept fits into the wider system before diving into specifics; high `incremental_build_up` means start from the smallest concrete piece and only zoom out if asked.

Also use `background_context` as before: `seniority_level` and `known_technologies` still inform how much can be assumed, `preferred_terminology_style` decides domain-specific vs generic CS terms, and `is_english_native` calibrates vocabulary complexity within whichever language the user is using (never which language to respond in — that's always the user's own language).

None of this is exposed to the user. Don't mention weights, scores, or that a profile was consulted — the calibration should simply be felt in how the explanation is shaped, the same way a good teacher doesn't narrate their own pedagogy.

## Logging drift signals

While using a profile, watch for signs that the current calibration isn't landing. Two kinds of signal count, and both are logged the same way:

- **Implicit**: the user asks for the same explanation reformulated two or more times in a row (e.g. they keep asking follow-ups that suggest the original framing isn't sticking, not just asking a deeper follow-up question on a point that did land).
- **Explicit**: the user says something like "I don't get it like that", "can you explain it differently", or otherwise directly signals the current approach isn't working for them.

When either happens, append one line to `~/.config/opencode/data/cognitive-profile/drift-log.jsonl`:
```json
{"date": "<today>", "axis": "<best-guess axis, e.g. concreteness>", "signal": "<brief description of what happened>", "source_skill": "explain"}
```
Pick the axis that best fits what actually seemed to go wrong (e.g. they wanted code first, not the analogy → `concreteness`; they wanted a diagram, not prose → `modality`). If you're unsure which axis fits, log it anyway with your best guess rather than skipping the entry — `cognitive-profile` only acts once a consistent pattern builds up across several signals, so an occasional imperfect guess doesn't cause harm.

**Never write to `profile.json` itself, and never change your own calibration immediately in response to a single signal within the same explanation** — just adjust your approach for the very next attempt within this conversation as normal responsive behaviour (e.g. if asked to explain differently, obviously do so), log the signal, and let `cognitive-profile` decide separately, over time, whether the profile itself should change.

---

## Tone and language

- **Respond in the user's language.** If they write in Spanish, explain in Spanish. Don't switch languages mid-explanation.
- **Be direct.** Confident explanations build trust. Hedge only when genuine uncertainty exists.
- **Match formality to the user.** Conversational unless they're being formal.
- **Never make the user feel bad for not knowing something.** Correcting a misconception is helpful; implying they should have known is not.
- **Calibrate depth to the user's background.** If no cognitive profile was found (step 0), this is your primary calibration method: a developer who just joined the team gets more analogy and less jargon than a senior architect asking a specific technical question — read context cues such as vocabulary they use, how precisely they describe the problem, and whether they reference specific internals. If a profile was found, this still applies as a sanity check alongside the profile-driven calibration in "Using the profile to calibrate" — the profile sets the main dial, but obvious context cues in the moment (e.g. the user clearly already knows the term they're asking about) should still nudge the response, the same way they would for anyone.
