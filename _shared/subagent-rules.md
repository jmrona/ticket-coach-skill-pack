# Shared Subagent Rules

Used by `brainstorming-committee` and `design-review`. These are the non-negotiable rules ("iron laws") for any skill in this pack that dispatches subagents via `task`. Both skills reference this file instead of repeating these rules inline.

## Context isolation

Each subagent call **MUST** receive only what it needs to do its job: the brief, relevant codebase context, the specific decisions/text under review, and prior decisions if relevant. **MUST NOT** pass session/conversation history to a subagent. A subagent that inherits the full conversation ends up grading or reasoning about the conversation itself rather than the artefact it was actually asked to evaluate.

## No git, no worktrees, no elevated privileges

Subagents dispatched by these skills **MUST NOT** run `git` commands, create or switch worktrees, or be granted any privilege beyond reading the codebase context they were given and reasoning about it. These skills are for deliberation and review, not for making changes to the repository. Any subagent prompt that asks a dispatched subagent to modify files, commit, or branch is out of scope for this pack.

## Subagent type selection

Prefer the most specific available subagent type over a generic one. A specialised type carries built-in domain knowledge, so the dispatch prompt can focus on the actual decision or review content instead of teaching a persona from scratch. Fall back to `general-purpose` only when no specialised type is available for the role in question.

## Parallel dispatch

When a step calls for dispatching multiple subagents that don't depend on each other's output (e.g. the three committee perspectives in `brainstorming-committee`), dispatch them in a single message with multiple `task` calls so they run concurrently. Dispatching them one at a time, waiting for each to finish before starting the next, wastes time for no benefit.

## Model selection (where applicable)

For skills in this pack that involve a model choice for a dispatched subagent (currently only `design-review`'s reviewer dispatch): pick one concrete default model rather than reasoning about "the best model for this" each time, and escalate only on clear evidence that the task needs deeper reasoning (e.g. the default model's output is shallow, contradictory, or visibly missed something material). Don't pre-escalate "just in case" — that defeats the purpose of having a cheap default tier at all.
