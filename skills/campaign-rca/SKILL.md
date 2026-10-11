---
name: campaign-rca
description: Use when a specific symptom has already been noticed (a metric spiked or dropped, something looks broken) and the question is "why." Reactive and narrow in scope (one symptom), not narrow in investigation — checks every relevant hypothesis rather than stopping at the first that fits. Not for a general health check with no specific trigger (use campaign-audit) or a routine periodic review (use campaign-analysis). Orchestrates performance-marketer and martech-expert to surface a primary cause plus any other confirmed contributing factors, each verified against real data, with a concrete action plan.
---

# Campaign RCA — orchestration procedure

This is a task-type command — it shapes the investigation and the output,
it doesn't itself carry platform knowledge. Delegate the actual diagnosis
to `performance-marketer`/`martech-expert`, which load the right
campaign-type playbook (e.g. `google-ads-uac`) and pull real data
themselves.

## 1. Require a specific symptom, not a vague concern

Don't start without: which client, which campaign/metric, what changed
(direction and rough magnitude), and roughly when it started. "Something
seems off" is a campaign-audit request, not an RCA — ask for the specific
symptom before proceeding rather than guessing at one.

## 2. Mix-shift check first — always

Before any other hypothesis, invoke `performance-marketer` with an explicit
instruction to rule out mix-shift first (this is already one of its own
standing rules: a blended metric moving because a cheap channel's spend
share fell is not the same bug as that channel actually getting worse).
Skipping this step and jumping straight to a campaign-level explanation is
the single most common way an RCA reaches the wrong conclusion.

## 3. Hypothesize from the campaign-type skill's known failure patterns

If a campaign-type playbook exists for what's in scope (e.g.
`google-ads-uac`'s "Common failure patterns, roughly in order of
frequency" list), use it as the ranked hypothesis list rather than
inventing one from scratch — these lists exist specifically because they're
already the common real-world causes for that campaign type. **Work
through the list, not just down to the first match** — see step 4.

## 4. Verify every hypothesis worth checking — don't stop at the first confirmed one

For each hypothesis in the ranked list (at minimum the top few; all of
them if the account/timeframe makes that cheap):
- If it's a **performance** question (budget, bidding, creative, audience),
  `performance-marketer` checks it against real data.
- If it's a **"is this actually tracked/configured correctly"** question,
  hand it to `martech-expert` — and where a live platform-side check would
  settle it (e.g. "does this account's conversion tracking show a real
  event" or "what does Google's own API say here"), use the API Explorer
  (`/connectors/explorer`) to query the platform directly rather than
  inferring from this dashboard's synced data alone.

**Finding one hypothesis that's confirmed by real data is not a stopping
condition.** A real symptom is very often driven by more than one true
factor at once (e.g. a budget cut *and* a creative that was already
softening before the cut). Keep checking the remaining hypotheses that are
still cheap to verify against real data even after the first one is
confirmed — reporting only the first cause found looks decisive but is
frequently incomplete. Record each hypothesis's outcome explicitly:
**confirmed contributor** (with the evidence), **ruled out** (with the
evidence that ruled it out), or **not checked** (and why, if time/data
genuinely didn't allow it) — don't silently drop a hypothesis without
saying which of the three it was.

## 5. Output shape

A primary cause plus any other confirmed contributing factors — not
artificially forced into a single winner, and not a vague list of maybes
either. Every factor named must have survived independent verification
against real data, not be included just because it's plausible:

- **Primary cause** — the single biggest driver, stated plainly, with the
  specific evidence that supports it (a number, a query result, a config
  fact).
- **Other confirmed contributing factors**, if any real ones survived
  verification, listed below the primary and ranked by evidence of actual
  impact — each with its own supporting evidence. If nothing else survived
  verification, say so plainly rather than manufacturing a second factor
  to look thorough.
- **A concrete action plan for each factor worth acting on** — not
  "check the budget," but the specific action, the specific number/target,
  who should actually execute it (media buyer, account owner, or flagged
  to `martech-expert`/`developer` if it's a tracking/config fix), and the
  metric + timeframe that would confirm it worked. See
  `performance-marketer`'s own output shape for the exact structure.
- If the evidence genuinely doesn't distinguish between two causes, say so
  explicitly and name the exact next check (a query, a screenshot, a
  config value) that would resolve it — don't paper over real ambiguity
  with a confident-sounding guess.
- A one-line note on what was ruled out and why (mix-shift, in particular,
  and any other hypothesis checked and rejected), so the reader doesn't
  re-litigate ground already covered.
