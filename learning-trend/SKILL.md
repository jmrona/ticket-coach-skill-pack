---
name: learning-trend
description: Shows how the user's ticket-coach performance has evolved over time — per-ticket scorecards, technology proficiency growth, and seniority progression. Use this when the user asks things like "how am I improving?", "show my progress", "have I gotten better at Go?", "what's my trend", or any request to see evolution/history/improvement related to ticket-coach sessions. This skill is read-only and purely presentational — it never writes to the cognitive profile or scorecards, and only activates when the user explicitly asks to see their progress, never proactively.
---

# Learning Trend

This skill reads the scorecard history that `ticket-coach` has been quietly building up (one entry per completed ticket, via `cognitive-profile`'s FLOW D) and presents it back to the user as a readable picture of how their performance and proficiency have evolved.

This skill is **read-only**: it never writes to `profile.json` or `scorecards.jsonl`, and never asks `cognitive-profile` to change anything. It only consumes data via `cognitive-profile`'s FLOW E and presents it.

## When this activates

Only on an explicit request from the user to see their progress, history, or improvement (e.g. "how am I doing", "show me my trend", "am I getting better at Go"). Never activate this proactively or as a side effect of another skill's flow — `ticket-coach` does not call this skill, and finishing a ticket does not trigger it.

## Flow

1. **Fetch the data.** Call `cognitive-profile`'s FLOW E to get the full scorecard history.
   - If status is `no_scorecards_yet` → tell the user plainly that there's no ticket-coach history yet to show a trend from, and stop here. Don't fabricate a trend from nothing.
   - If status is `scorecards_found` → continue.
2. **Also fetch the current profile**, via `cognitive-profile`'s FLOW C, to report current `known_technologies` proficiency levels and `seniority_level` alongside the trend (these are the outcomes the scorecards feed into, so showing them together is more useful than scores in isolation). If no profile exists, skip this and just present the scorecard trend on its own.
3. **Decide what to show** based on what the user actually asked:
   - A general "how am I doing" → give an overview: how many tickets tracked, the rough direction (improving / steady / mixed) for each of the five criteria across the available history, and current proficiency levels per technology.
   - A request scoped to one technology (e.g. "have I gotten better at Go?") → filter to scorecards where that technology appears in `technologies_involved`, and focus the answer there.
   - A request scoped to one criterion (e.g. "am I asking for help less?") → focus on that criterion's trend across all tickets.
4. **Present it as a narrative, not a data dump.** Don't print the raw JSONL or a wall of numbers. Summarise direction and magnitude in plain language, and use the actual ticket IDs and dates where they make the point concrete (e.g. "Looking at your last 5 Go tickets, `code_without_help` has gone from averaging around 3 to consistently landing at 5 — PROJ-40 was the last time you needed real code rather than pseudocode"). A simple visual (e.g. a small chart or sparkline-style table) is welcome if the conversation surface supports it and there's enough data to make one meaningful (roughly 4+ data points) — don't force a chart out of 1-2 entries.
5. **If there isn't much data yet** (fewer than 3-4 scorecard entries), say so honestly rather than overstating a "trend" from a couple of data points — e.g. "You've only got two tickets tracked so far, so it's a bit early to call a trend, but here's what those two look like." Don't manufacture confidence the data doesn't support.
6. **Never show raw `cognitive_profile` axis scores** (e.g. `concreteness: 4.5`) even though FLOW C returns them — that's not what this skill is for, and showing them risks the same false-precision problem the cognitive-profile skill itself avoids by design. Stick to scorecard criteria, technology proficiency levels (`learning`/`intermediate`/`expert`), and `seniority_level` — these are the genuinely earned, behaviourally-grounded numbers, not self-reported ones.
7. **`proficiency_confidence` and `seniority_level_confidence` are fair to surface, in plain terms** — unlike the cognitive axis scores, these are accumulated directly from scorecard evidence (ticket outcomes), not from a handful of self-reported questionnaire answers, so they don't carry the same false-precision risk. If the user asks something like "how close am I to leveling up in Go?", translate the raw 0–1 value into a plain-language sense of distance rather than reporting the decimal directly (e.g. "you're most of the way there — a couple more strong Go tickets and it should tip over" rather than "your proficiency_confidence is 0.58"). Don't volunteer these numbers unprompted; they're an answer to "how close", not part of a general progress overview unless asked.

## Tone

This should feel like checking in on real progress, not a performance review. Lead with what's genuinely improved before mentioning anything that's flat or mixed. If a criterion has gotten worse recently, mention it plainly but without alarm — frame it as information, not a verdict (e.g. "design autonomy dipped a bit on the last couple of tickets" rather than "you're struggling with design now"). The user is doing this to learn, and the data reflects effort and growth over time more than it reflects any single ticket.