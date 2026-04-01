---
title: "A Production Checklist for Browser Agents"
id: "production-checklist-for-browser-agents"
summary: "Use this production checklist to harden browser agents: plan-tier gating, session controls, evidence logging, and recovery rituals that withstand scale."
canonical_questions: ["a production checklist for browser agents"]
intent: "reference"
entity: "browser-automation"
audience: "developer"
schema_type: "Article"
visibility: "public"
ai_visibility: "public"
llms_priority: "core"
token_budget: "medium"
date: "2026-04-01"
updated: "2026-04-01"
related: []
external_refs: []
type: "article"
status: "draft"
canonical_url: "https://steel.dev/blog/production-checklist-for-browser-agents"
description: "Use this production checklist to harden browser agents: plan-tier gating, session controls, evidence logging, and recovery rituals that withstand scale."
created: "2026-04-01"
modified: "2026-04-01"
tags: [checklist, production, browser, agents]
---
You are ready for production when [browser agents](@/glossary/browser-agents.md) survive concurrency, anti-bot pressure, and human review without losing state. That happens only after you wire evidence, approvals, recovery hooks, and plan-tier limits into the run, not when the happy path passes on localhost.

This checklist keeps teams honest: it groups the work into prepare, run, and operate stages so you can see what is missing before traffic spikes or an auditor asks for proof.

## Production checklist

### 1. Prepare your control plane (before traffic)

| Item | Why it matters | How to verify |
| --- | --- | --- |
| Plan tier sized for real load (Steel Local ~1 session, Steel Cloud 100+) | Keeps concurrency, RPS, and proxy budgets aligned with actual workflows instead of starving jobs on day one | Compare target parallel sessions and RPS with `Pricing/Limits`, bake alerts at 80 percent of plan cap |
| Session lifetime budget (24 hour ceiling, adjustable timeout per run) | Prevents zombie browsers while ensuring long flows finish under the allowed window | Store timeout per workflow, confirm Sessions API `timeout` matches longest job, add release reminder to job teardown |
| Proxy and CAPTCHA coverage | Most production sites rotate bot defenses faster than shipping cycles | List required regions, confirm managed proxies or BYO pool per region, toggle `solveCaptcha` only where policy allows |
| Secrets, profiles, and credentials policy | Auth reuse fails without profile storage and credential custody | Decide which runs need Profiles API vs stateless sessions, store credential owner, rotate tokens on schedule |
| Environment parity | Local Playwright stacks hide TLS, codec, and CPU limits that show up in Steel Cloud | Rebuild representative workflows against Steel sessions weekly, diff telemetry vs local |

### 2. Run workflows (during execution)

| Item | Why it matters | How to verify |
| --- | --- | --- |
| Stage timers for startup, first action, completion | Latency drift is the first sign you are about to breach SLAs | Emit timers per session lifecycle stage, export scoreboard daily |
| Selector and action contracts | Fuzzy selectors create nondeterministic failures that hide in retries | Track selectors in version control, gate releases on selector diff review |
| Manual approval schema (who, why, elapsed) | Every pause is a production incident without context | Store approvals alongside sessions (reason, operator, resumed session ID), alert when manual rate >5 percent |
| Evidence coverage (logs plus replay for every failure) | Without proof, you cannot debug or satisfy auditors | Require replay URL and agent log link before marking jobs complete, block deploys when evidence coverage <100 percent |
| Release discipline | Sessions bill until released even if the job failed early | Call `sessions.release` on success paths and `releaseAll` on incident cleanup; monitor orphan count |

### 3. Operate and recover (after execution)

| Item | Why it matters | How to verify |
| --- | --- | --- |
| Plan-cap saturation watch | Reliability issues often start as exhausted concurrency pools | Alert when any plan metric hits 90 percent, route to the owner who can upgrade or shift load |
| Replay and log review cadence | Keeps regressions visible instead of tribal memory | Run a 10 minute daily ritual: export segmented scoreboard, sample replays for every red slice, log actions |
| Retry strategy with idempotency keys | Production retries must not double-charge or re-open tickets | Store workflow ID and attempt count, refuse retries without idempotency metadata |
| Incident drill and on-call runbook | Agents touch money and accounts; recovery must be muscle memory | Keep an updated playbook with owner, escalation path, and checklist per failure mode |
| Dependency refresh (proxies, extensions, scripts) | Browser ecosystems drift; stale components are a silent outage | Track versions for anti-detect packages, extensions, and scripts, rotate on schedule |

## Red flags you cannot ignore

| Signal | Why it is dangerous | Action |
| --- | --- | --- |
| Failed session without replay or logs | You are blind during the postmortem and cannot prove what happened | Block deploys until evidence exports reach 100 percent again |
| Manual interventions above 5 percent of runs | Human approvals are masking automation gaps | Add approval reason taxonomy, route top reasons into engineering backlog |
| Selector churn on every release | Frontend changes are ahead of your maintenance loop | Freeze deploy until selectors are versioned, pair with DOM-change alerts |
| Idle sessions after workflow ends | You are paying for nothing and leaving state open | Run hourly `releaseAll` sweep, alert when orphan count exceeds 0 |
| Geo or proxy mismatch | Wrong data or blocked runs despite "success" status | Pin required regions per workflow and audit proxy inventory weekly |

## Minimum viable production setup

- Sessions API client with configurable timeout, proxy, and CAPTCHA params per workflow.
- Scoreboard export that captures startup, first action, completion, plan-tier utilization, approval rate, and evidence coverage.
- Structured manual approval and intervention log (operator, reason, resumed session ID, elapsed time).
- Replay plus agent log storage tied to every failed run before incident close.
- Release hooks: success path releases specific session, incident path calls `releaseAll` and posts orphan count.
- Weekly parity run that compares Steel Cloud vs local automation stats, recorded in the artifact folder.

## What Steel handles for you

- Managed Sessions that start in under a second on Steel Cloud plus Profiles API for state reuse when auth matters.
- Built-in managed proxies, CAPTCHA solving, and stealth layers so you do not maintain your own anti-bot stack.
- Live view, replays, and Agent Logs for every session so evidence coverage is achievable.
- Credentials and Files APIs so sensitive inputs stay vaulted while uploads and downloads remain scriptable.

## Next step

Run your workflows against the Sessions API lifecycle doc, confirm every checklist item above has an owner, then cut the first production review using the artifact template you just filled. [Read the session lifecycle](https://docs.steel.dev/overview/sessions-api/session-lifecycle) and wire the release hooks before you add more traffic.

Humans use Chrome. Agents use Steel.
