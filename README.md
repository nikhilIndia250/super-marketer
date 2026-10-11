# Super Marketer

Performance-marketing agents and skills for Google/Meta Ads campaign diagnosis, audit, root-cause analysis, and growth planning — built and verified against real account data across multiple live accounts, not generic platform advice.

## What's in here

**2 agents:**
- `performance-marketer` — interprets real campaign performance data (spend, CPI, CAC, CTR, ROAS, ad-set/ad-level breakdowns) and turns it into a concrete diagnosis and action plan. Read-only; never mutates data.
- `martech-expert` — diagnoses tracking/API/attribution plumbing (pixel/SDK setup, conversion event configuration, join-key mismatches, API integration issues). Also read-only.

**10 skills:**
- 3 task-orchestration skills that route to the agents above: `campaign-analysis` (routine review), `campaign-audit` (full health check, scorecard output), `campaign-rca` (single-symptom root cause, multi-hypothesis)
- 1 deeply-verified campaign-type playbook: `google-ads-uac` (Google App Campaigns — ACI/ACE), covering attribution caveats, confirmed failure patterns, bidding-goal verification procedures, and a standing in-app-actions-vs-postclick gap check
- 6 platform/API reference skills: `google-ads`, `google-ads-api`, `meta-ads`, `meta-ads-api`, `mobile-marketing-platforms`, `web-analytics`

## Install

```bash
claude plugin marketplace add nikhilIndia250/super-marketer
claude plugin install super-marketer@nikhilIndia250/super-marketer
```

Or test locally before installing from GitHub:
```bash
claude --plugin-dir ./super-marketer
```

## A note on portability

This was extracted from a specific marketing dashboard's codebase, and some content references that dashboard's own internals directly — specific file paths (e.g. `services/connectors/transformService.js`), Supabase table/column names, and query patterns particular to its own data pipeline. Those references are illustrative ("this is how one real dashboard does it"), not something this plugin can act on in a different codebase — the agents expect to be told how *your* system stores and queries campaign data, or adapted to reference it.

What travels fully intact regardless of codebase: the platform knowledge itself — Google Ads/Meta Ads API mechanics, UAC campaign attribution behavior, GAQL query gotchas, the diagnostic procedures and failure patterns in `google-ads-uac`, and the behavioral disciplines baked into both agents (ask-first on genuine ambiguity, funnel-first analysis, network-volume-before-sizing-a-lever, credential-handling rules). That's the hard-won part, and it's universal.

## The question-asking pattern — read this before using these agents for an audit or optimization task

These agents are built to work in the spirit of an interview, not a one-shot answer — but they **cannot** hold a live back-and-forth with you directly. Here's why, and how it actually works.

### The architectural fact

A subagent (anything invoked via Claude Code's `Agent`/`Task` tool) runs to completion and hands back **one written report per invocation**. It does not have access to interactive UI tools like `AskUserQuestion`, and even if it did, it has no mechanism to pause mid-execution, render a dialogue box, block on your answer, and resume within that same run. Only the **top-level session you're actually talking to** — the orchestrator — can hold a live dialogue with you.

### The pattern that actually works

1. **The subagent surfaces ambiguity as plain text**, not as a dialogue box. Both agents in this plugin are instructed (see "For an audit or optimization task, ask first" in `performance-marketer.md`) to do a fast first pass aimed at identifying anything genuinely ambiguous — scope, metric definitions, which baseline a target applies to — and return with just those questions *before* investing in the full analysis, rather than guessing and burying the assumption in a footnote. This isn't limited to the start of the task either: if something new and similarly blocking turns up mid-analysis, the same instruction applies — stop and hand back there too.
2. **The orchestrator (you're talking to it right now) reads those questions and presents them to the user** — ideally one at a time via `AskUserQuestion`, matching the actual interview experience, rather than dumping the whole list as text.
3. **Once you've answered, the orchestrator resumes the *same* subagent** (not a fresh one) with your answers. This preserves everything the subagent already knows — its prior data pulls, its reasoning so far — so you're not paying to re-derive context, and the subagent picks up exactly where it left off.
4. **This repeats as many times as needed.** A single real analysis in testing needed four separate rounds of this — account scope, a baseline definition, a metric-scope decision, and a mid-task finding that reversed a planned sequencing — each one a full stop-question-answer-resume cycle, all using the same mechanism.

### Why this is the right design, not a workaround

This was deliberately chosen over giving the agents direct `AskUserQuestion` access, for reasons that matter more as usage scales, not less:

- **Multi-agent coordination.** With multiple subagents potentially active, a single coordination point (the orchestrator) can sequence questions from different agents sensibly. Direct per-agent dialogue access would risk uncoordinated, overlapping prompts with no one managing the order.
- **Context stays compartmentalized and cheap.** A subagent's own (often large) context — every tool call, every data pull — never has to enter the orchestrator's context; only its final report does. The resume pattern preserves this: resuming a subagent continues *its* context, not the orchestrator's. This is what makes long, many-round analyses affordable at all.
- **Portability.** This pattern depends on nothing but the orchestrating session having `AskUserQuestion` — true of any Claude Code session, in any environment. A dependency on subagents holding live blocking dialogue is unverified and environment-sensitive; it could work in one setup and silently fail in another.

### What this means practically, if you're the one orchestrating

When one of these agents hands back a report containing questions (it'll usually be obvious — explicit "before I build this, I need to know..." framing), don't just relay the text. Pull the questions out, ask them one at a time (via `AskUserQuestion` if you have it, or plain dialogue otherwise), and once answered, resume that same agent with the answers rather than starting a fresh one or trying to extrapolate the answer yourself. The agent is designed around getting resumed with real answers, not around you guessing on its behalf.

## What these agents will never do

Both agents are strictly read-only against campaign/business data. They query live platform APIs only through sanctioned credential functions (never a self-minted token, never routing around an auth check), and the only write either of them is allowed to make is an incidental auth-token refresh triggered by those sanctioned functions — never campaign data, config, or any business table. Diagnosis and recommendations only; a human applies any actual change.
