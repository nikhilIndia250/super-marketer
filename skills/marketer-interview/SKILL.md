---
name: marketer-interview
description: Use whenever delegating a task to the performance-marketer or martech-expert agents (project-local or from the super-marketer plugin) for an audit, optimization, growth-plan, or efficiency-target task — and especially the moment either agent's response contains blocking questions rather than a finished analysis. Converts those questions into an actual interactive exchange with the user instead of just relaying agent text, and governs how to resume the agent once answered.
---

# Marketer Interview — orchestration procedure

This skill exists to close a specific, confirmed gap: `performance-marketer`
and `martech-expert` are both instructed to surface genuinely blocking
ambiguity as explicit questions before building a full analysis (see
"For an audit or optimization task, ask first" in `performance-marketer.md`).
**But a subagent cannot show the user a dialogue box.** It can only hand
back one written report. Without this skill, an orchestrator's default
behavior is to just print that report — questions and all — as plain text,
which looks like an answer was given when actually nothing has been
asked yet. That happened in testing: the agent asked the right questions,
and the orchestrator just relayed them as text instead of actually asking.

## 1. Recognize a blocking-questions response

The agent's report is asking, not answering, when it does things like:
- States it's returning questions *before* building the analysis, rather
  than after one
- Lists 2+ explicit questions, often numbered, often under a heading like
  "Questions for you," "Blocking questions," or similar
- Says something to the effect of "I'll build the full plan once I have
  these answers" rather than presenting completed numbers

Don't require an exact phrase match — if the report is clearly asking
rather than concluding, treat it as a blocking-questions response.

## 2. Ask them one at a time, via `AskUserQuestion` — not as a dumped list

For each question in the report, call `AskUserQuestion` **individually**,
not as one call bundling all of them (even though the tool supports
multiple questions per call — the point is the interview *feel*, matching
how the agent's own report framed each question as changing the
recommendation differently). For each one:
- Write real, distinct options that capture the actual plausible answers
  the agent's report implies (not generic "yes/no" unless the question
  genuinely is binary) — the report usually states what the different
  answers would mean, use that to build the options
- Mark the option you'd actually recommend, if the report gives you
  enough to judge one as more likely correct, the way the agent's own
  reports usually do
- Always leave room for free text (the `AskUserQuestion` tool does this
  automatically) since your option-guessing may not cover what the user
  actually wants to say

Do not answer on the user's behalf by guessing at what they'd probably
want — that's the exact failure this whole mechanism exists to prevent,
just moved one level up the chain.

## 3. Resume the same agent with the answer — never start a fresh one

Once you have an answer (or several, if the agent returned multiple
questions), send it back to the **same subagent invocation** that asked —
using its agent id/name, not a new delegation. This preserves everything
it already pulled and reasoned through; starting fresh would waste the
work already done and risk re-deriving things inconsistently.

## 4. Repeat for as many rounds as it takes — this isn't a one-time gate

A resumed agent can come back with *another* round of blocking questions
if something new and similarly significant turns up mid-analysis (this is
explicit in `performance-marketer.md`'s own instructions — asking isn't
limited to the first pass). Treat every round the same way: recognize it,
ask one at a time, resume. Confirmed in real use: a single real analysis
needed four separate rounds of this before landing on a final plan.

## 5. Once there are no more blocking questions, relay normally

When the agent's response is a completed analysis (no more open blocking
questions), present it to the user as the finished deliverable — no
further `AskUserQuestion` calls needed for that round.

## What this skill does not change

This governs *your* behavior as the orchestrator — it does not add any
new tool to the subagents themselves, and it doesn't make them capable of
live dialogue. They still only ever hand back one report per invocation.
All the interactivity happens here, in how you handle that report.
