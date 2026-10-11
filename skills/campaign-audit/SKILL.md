---
name: campaign-audit
description: Use for a comprehensive, no-specific-trigger health check across a client's campaigns or channels — "how healthy is everything right now." Not for a single reported symptom (use campaign-rca) or a routine periodic review (use campaign-analysis). Orchestrates performance-marketer (and martech-expert when a data-integrity gap surfaces) into a structured scorecard.
---

# Campaign Audit — orchestration procedure

This is a task-type command, not a source of platform knowledge itself — it
defines *what to ask for and in what shape*, and delegates the actual
diagnosis to `performance-marketer`/`martech-expert`, which already know how
to pull real data and load the right campaign-type playbook (e.g.
`google-ads-uac`). Don't re-derive platform-specific reasoning here.

## 1. Confirm scope

If not already stated, ask for: client slug, which campaigns/channels (or
"everything active"), and a lookback window (default: last 30 days). Don't
guess a client slug — this codebase never mixes client data, and neither
should this command.

## 2. Run the audit via `performance-marketer`

Invoke the `performance-marketer` agent with an explicit audit framing, not
a generic "how's this doing" prompt — the difference is coverage. Ask it to
review **every** active campaign/channel in scope, not just the ones that
look interesting, and for each: spend trend, CPI/CAC, CTR, ROAS, ad-health
flags, and whether a campaign-type playbook exists for it (per its own
"Campaign-type playbooks" rule — if none exists yet for something in scope,
that must be stated explicitly, not silently skipped).

## 3. Escalate data-integrity findings separately

If the audit surfaces something that smells like a tracking/pipeline
problem rather than a performance problem (a metric that's structurally
zero, an attribution gap, a platform number that doesn't reconcile) —
invoke `martech-expert` for that specific finding on its own. Don't let
`performance-marketer` guess at a plumbing diagnosis; that's not its job
per its own scope note.

## 4. Output shape

A scorecard, not a narrative report:

- One line per campaign/channel: **Healthy** / **Needs attention** /
  **Insufficient data**, each with the one specific number or fact driving
  that verdict.
- A ranked top-3 list of what actually needs action across everything
  reviewed — ranked by expected impact, not by how many campaigns are
  affected.
- An explicit "not evaluated" line for anything skipped because no
  campaign-type playbook exists yet for it, or because a martech-expert
  question is still open and blocking a real verdict.
- No individual-campaign narrative unless it's one of the top-3 actions —
  an audit that reads like a full report on every campaign defeats the
  point of a scorecard.
