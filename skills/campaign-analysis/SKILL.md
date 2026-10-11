---
name: campaign-analysis
description: Use for a routine, on-demand or periodic performance review of a client's campaign(s) over a period — "how did this do," with no specific reported symptom and no requirement to cover every campaign the way an audit does. Not for a no-trigger full health check (use campaign-audit) or investigating a specific reported problem (use campaign-rca). Thin orchestration around performance-marketer's own standard diagnosis shape, with an explicit period-over-period comparison.
---

# Campaign Analysis — orchestration procedure

The lightest of the three task-type commands — this is close to
`performance-marketer`'s own default behavior, formalized so it's
consistent every time rather than shaped ad hoc per request.

## 1. Confirm scope

Client, which campaign(s)/channel(s) (can be narrower than an audit — a
single campaign is a completely normal ask here), and the period (default:
last complete week or month, whichever the client's own review cadence
uses — check `dashboard.mbr` config if unsure).

## 2. Run via `performance-marketer`

Invoke it for its standard four-part output: what's happening, why (as far
as the data supports), ranked recommendations, and what NOT to touch. This
is its default shape — no special audit-style full-coverage framing needed
here, and no RCA-style single-symptom narrowing either.

## 3. Always frame against the prior period

Pull the immediately preceding period of the same length for comparison
(this week vs. last week, this month vs. last month) so the analysis is
trend-framed rather than a snapshot — and apply `performance-marketer`'s
own noise-vs-trend rule (one period's movement is not a trend; say so when
the sample is too short to act on).

## 4. Output shape

`performance-marketer`'s standard four-part diagnosis, with the
period-over-period comparison folded into part 1 ("what's happening")
rather than presented as a separate section — the point is one coherent
read, not a report bolted onto a diagnosis.
