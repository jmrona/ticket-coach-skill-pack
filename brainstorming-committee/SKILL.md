---
name: brainstorming-committee
description: "Turns a design brief into an approved, reviewed design through autonomous deliberation by three subagents with different perspectives, who debate decisions and converge on consensus without user involvement until the final design is ready for review. Used as the design phase for a ticket-coach session: the approved design becomes the silent internal reference plan ticket-coach checks the user's own design thinking against."
---

# Brainstorming (Committee)

Hands-off design dialogue. The caller (typically `ticket-coach`, during its own design phase) or the user directly provides a brief, then three subagents with deliberately different perspectives deliberate on each design decision and converge on consensus. The user only sees the final design — never the deliberation transcripts.

The non-negotiable rules on subagent dispatch (context isolation, no git/worktrees/elevated privileges, type selection, parallel dispatch) live in `../_shared/subagent-rules.md`. The design review step is delegated to the `design-review` skill.

## Load Required Files First

This skill depends on two sibling/shared files that are **not** always in context:

- `committee-member-prompt.md` (sibling) — prompt templates paste-filled in step 4
- `../_shared/subagent-rules.md` — the dispatch rules referenced throughout this procedure

Before running the procedure below, read both files using the read tool if you have not already read them in this session.

## Preconditions

- Use this skill only when there are at least two meaningful design decisions to make. For a single trivial decision (a config change, a one-line fix), this skill is overkill — just decide it directly.
- When invoked from `ticket-coach`'s design phase, the brief is the ticket itself (plus whatever phase 1 already established about the requirement) rather than a fresh user-provided brief — skip step 1's confirmation in that case, since `ticket-coach` has already confirmed understanding of the ticket in its own phase 1.

## Procedure

1. **Confirm the brief** (skip if invoked from `ticket-coach` — see Preconditions). This is the only point where you ask the user a question during deliberation. Restate what you understand they want to build, the scope as you see it, and any assumptions, then confirm directly with the user. Once confirmed, the user is hands-off until step 8.
2. **Explore project context.** Read the area of the codebase relevant to the brief. Stay narrow: every file you read here gets passed to every committee member in every round, so unnecessary context dilutes focus and wastes capacity. Identify the project's primary language and framework now because the Pragmatist subagent type depends on it (see Pragmatist Type Mapping below). If the feature touches more than ~3 subsystems, or you cannot determine which code is relevant, surface what you have found and ask for guidance (the user, or — if invoked from `ticket-coach` — hand the ambiguity back to it rather than guessing). This is the only other point where a question may be needed.
3. **Identify design decisions.** List the key decisions: architecture, approach, data flow, error handling, testing strategy, integration. Group related decisions so each committee round addresses a coherent set, not individual micro-choices.
4. **Run committee deliberation rounds.** For each decision group, dispatch three committee members in parallel (single message, multiple `task` calls — see "Parallel dispatch" in `../_shared/subagent-rules.md`) using the templates in `committee-member-prompt.md`. Each member receives identical context: the brief, the relevant codebase context, the specific decisions, and any decisions already made. Each returns a recommendation, reasoning, and concerns.
5. **Synthesise consensus** after each round using the table below. Record the consensus before moving to the next round.
6. **Assemble the design summary.** Combine all consensus decisions into a single coherent summary covering goal, architecture, components, interfaces, data flow, error handling, testing strategy, and integration points. Scale each section to its complexity. Follow existing patterns; do not propose unrelated refactoring.
7. **Invoke `design-review`** against the consolidated summary. If the review surfaces issues that require new decisions (not just wording fixes), feed the affected decisions back into a targeted committee round, update the summary, and re-invoke `design-review`. If `design-review` escalates after three iterations, stop and surface the remaining issues plus the current summary to whoever invoked this skill (the user, or `ticket-coach`).
8. **Return the result — never show the deliberation itself.**
   - **If invoked directly by the user**: present the reviewed summary, a brief note on any decisions where the committee was split (with the reasoning for the chosen direction), and any risks the committee or reviewer flagged. **MUST NOT** dump the deliberation transcripts — the user wants the result, not the process. Then ask for explicit approval; "looks fine" is not approval.
   - **If invoked from `ticket-coach`'s design phase**: return the reviewed summary directly to `ticket-coach` as the internal reference plan — **do not show it to the user yourself**. `ticket-coach` is the one that decides what the user sees and how the design phase plays out from there (see `ticket-coach`'s own SKILL.md, "Phase 2" handling), per the rule that the committee's plan is never shown to the user directly. `ticket-coach` is responsible for persisting this plan into its own session file once it receives it.

## The Three Perspectives

Each committee member uses a specialised subagent type that reinforces its perspective. The full prompt templates live in `committee-member-prompt.md`.

| Role        | What it prioritises                                                                  | Subagent type           |
|-------------|--------------------------------------------------------------------------------------|-------------------------|
| Pragmatist  | Simplicity, maintenance cost, shipping quickly, reusing what already exists          | Stack-specific (below)  |
| Architect   | Patterns, separation of concerns, well-defined interfaces, testability               | `architect-reviewer`    |
| Advocate    | Correctness, edge cases, robustness under failure, hard-to-misuse interfaces         | `qa-expert`             |

For the general rule that specialised subagent types are preferred over `general-purpose`, see `../_shared/subagent-rules.md`. The mapping above is the committee-specific instance of that rule.

### Pragmatist Type Mapping

Map the project's primary language or framework (identified during step 2) to the most specific available subagent type:

| Project stack          | Subagent type        |
|------------------------|----------------------|
| TypeScript             | `typescript-pro`     |
| Python                 | `python-pro`         |
| Go                     | `golang-pro`         |
| Rust                   | `rust-engineer`      |
| React frontend         | `react-specialist`   |
| PHP / Laravel          | `laravel-specialist` |
| Java / Spring          | `java-architect`     |
| C# / .NET              | `csharp-developer`   |
| Ruby / Rails           | `rails-expert`       |
| Multi-language / other | `general-purpose`    |

## Synthesising Consensus

| Outcome             | Action                                                                            |
|---------------------|-----------------------------------------------------------------------------------|
| All three agree     | Take that approach.                                                               |
| Two of three agree  | Take the majority view; record the dissenting concern if it is substantive.       |
| All three disagree  | Run a tiebreaking round (see below).                                              |

Concerns flagged by two or more members **MUST** be addressed in the design, even if the specific recommendation differs.

## Tiebreaking

If a round produces irreconcilable disagreement (not just preference differences), dispatch a single tiebreaking round. All three positions are passed to a single `architect-reviewer` subagent that must converge to a final decision. The tiebreaking template is in `committee-member-prompt.md`. Do not run more than one tiebreaking round per decision; if the tiebreaker still cannot converge, surface the unresolved decision (to the user, or to `ticket-coach` if that's the caller).

## Working in Existing Codebases

- Follow existing patterns. Where existing code has problems that affect the work, include targeted improvements in the design.
- Do not propose unrelated refactoring. Committee members are explicitly instructed in their prompts to stay within scope; the synthesiser must reject any recommendation that exceeds it.

## Completion Gate

This skill **MUST NOT** return a final result until all of these are true:

- Every identified decision group has a recorded consensus
- The design summary was assembled from those consensuses into a single text block
- `design-review` returned Approved
- If invoked directly by the user: the user gave explicit final approval against the reviewed summary

## Common Mistakes

| Mistake                                                                | Why it is wrong                                                                                                                |
|------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Asking the user clarifying questions during deliberation               | The committee replaces user questioning. The only allowed user questions are step 1 (when applicable) and the optional step 2 escape hatch. |
| Dispatching committee members sequentially                             | They **MUST** run concurrently. One message, multiple tool calls.                                                              |
| Passing session history to committee members                           | Violates context isolation from `../_shared/subagent-rules.md`. Pass the brief, the codebase context, the decisions, and prior consensus only. |
| Using `general-purpose` when a specialised type is available           | Violates the type-selection rule in `../_shared/subagent-rules.md`.                                                              |
| Showing the user the full deliberation transcripts, or showing the user anything at all when invoked from `ticket-coach` | The user wants the design, not the process. Surface only the result, splits, and flagged risks — and when invoked from `ticket-coach`, not even that; the plan stays internal. |
| Skipping `design-review` because the committee already reviewed itself | The committee deliberates, it does not review. `design-review` is a separate, holistic check.                                  |
| Running more than one tiebreaking round                                | If one tiebreaker cannot converge, the decision is genuinely unresolved and belongs to the user/caller.                        |
| Letting the synthesiser quietly drop a concern flagged by two members  | Two members flagging a concern is signal. Address it in the design or the round did not really converge.                       |
