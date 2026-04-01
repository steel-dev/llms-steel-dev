---
title: "Why Browser Automation Gets Expensive at Scale"
id: "browser-automation-cost-at-scale"
summary: "Why browser automation spend spikes at scale and how to keep Steel session, proxy, and CAPTCHA costs predictable."
canonical_questions: ["why browser automation gets expensive at scale"]
intent: "concept"
entity: "operations"
audience: "developer"
schema_type: "Article"
visibility: "public"
ai_visibility: "public"
llms_priority: "core"
token_budget: "medium"
date: "2026-03-31"
updated: "2026-03-31"
related: []
external_refs: []
type: "article"
status: "draft"
canonical_url: "https://steel.dev/blog/browser-automation-cost-at-scale"
description: "Why browser automation spend spikes at scale and how to keep Steel session, proxy, and CAPTCHA costs predictable."
created: "2026-03-31"
modified: "2026-03-31"
tags: [decision-guide, browser-automation, cost]
immutable: false
---
## Short answer
Browser automation spend jumps when concurrency caps, browser-hour rates, and stealth add-ons collide. [Steel Local](@/glossary/steel-local.md) is effectively a single-session runner, while [Steel Cloud](@/glossary/steel-cloud.md) plans top out at 5, 10, 20, or 100 concurrent sessions with request-per-second limits plus browser-hour pricing between $0.10 and $0.05. If you keep jobs idle in those slots or retry blindly, you burn your entire credit pool without adding throughput.

Instead of asking "why is this bill so high" after the fact, treat Steel Cloud as a session factory: size your plan to the number of simultaneous jobs you truly need, keep queue depth below 70 percent of the cap, release sessions as soon as the evidence is stored, and move stealth extras (managed [proxies](@/glossary/proxies.md), CAPTCHA solves) to the handful of flows that prove they need it. Pro plans pair 100 concurrent sessions, 10 requests per second, 24 hour max runtimes, and cheaper per-hour rates so one engineer can run hundreds of flows while staying inside the credit budget.

### Cost drivers you cannot ignore
| Driver | Why it explodes | Steel control |
| --- | --- | --- |
| Idle browser hours | Every live session burns $0.05 to $0.10 per hour even if the agent stalls. Long-lived flows on Hobby or Starter hit 15 to 30 minute caps and block new work. | Pass per-job `timeout` values, release session IDs as soon as you have artifacts, and split workflows so only the steps that need continuity reuse a profile. |
| Concurrency caps | Plans enforce 5, 10, 20, or 100 live sessions plus 1, 2, 5, or 10 requests per second. Queues that ignore that ceiling stack up and burst into retry storms. | Track active sessions and throttle dispatchers before they exceed 80 percent of the cap. Upgrade before hitting the limit daily or negotiate Enterprise capacity for anything above 100. |
| Stealth surface area | Residential proxies run $10 to $5 per GB depending on plan, and CAPTCHA solves are $4 to $3 per 1k on paid tiers. Turning both on everywhere multiplies cost faster than success. | Route every session with a `stealthNeeded` flag. Use managed proxies only for OTP gates or checkout flows, keep static scraping on default datacenter IPs, and capture CAPTCHA counts per workflow. |
| Retries without evidence | Silent failures trigger duplicate work and double proxy or CAPTCHA spend. Without replays, operators kill browser hours guessing. | Pipe `session.id` into logs, store replays, and push any job with more than two retries into a dead letter queue for inspection before letting it run again. |

## Know what you are paying per run
| Component | Plan math | Keep it sane |
| --- | --- | --- |
| Browser hours | Charged per minute at $0.10 (Hobby, Starter) down to $0.05 (Pro). Credits refresh monthly and overages can reach 3x the subscription at the same rate. | Forecast minutes per workflow and compare to plan credits. If browser-hour spend rises faster than throughput, leak detection (`releaseAll`) is cheaper than bumping tiers. |
| Proxy bandwidth | Billed per GB at $10 (Starter) to $5 (Pro). High-res screenshots or file downloads chew through this silently. | Keep media-heavy downloads on default networking when stealth is optional. Store evidence in Files API instead of re-downloading from the source site. |
| CAPTCHA solves | Priced at $4 to $3 per 1k solves on paid plans. One botwall can consume thousands per day. | Log solver usage per domain and add detection logic so you only call the solver when the challenge truly fires. |
| Human time | Every manual inspection of broken runs costs more than the browser hour. | Give operators session logs, HLS replay, and job metadata so the first investigation solves the issue without re-running the entire workflow. |

## Decide who should own the fleet
| Scenario | Stay on Steel Browser (self-host) | Move to Steel Cloud |
| --- | --- | --- |
| Concurrency need | You run one to three sessions at a time and can tolerate VM babysitting. | You need more than five live sessions or plan to hit the 20 to 100 range with proper queueing. |
| Stealth + compliance | Sites trust your colo IPs and you do not need managed proxies, CAPTCHA solves, or regional placement. | You need managed residential IPs, CAPTCHA API, data retention windows up to 14 days, and multi-region flags. |
| Cost predictability | You pay with team time and infra, trading cash for engineering attention. | You pay transparent browser-hour, proxy, and CAPTCHA rates while the platform handles fleet health. |
| Lifecycle controls | Manual restarts or scripts reclaim stuck sessions. | `release`, `releaseAll`, and warm pools recycle slots so concurrency stays productive. |

If you already need stealth, long retention, or more than ten simultaneous runs, Steel Cloud ends up cheaper because it replaces bespoke retries, [proxy](@/glossary/proxies.md) contracts, and on-call toil with plan math you can model in a spreadsheet. Self-hosting stays attractive only while you can point to a single engineer owning each failure.

## Run a cost discipline loop
1. **Model job cost upfront.** Include estimated minutes, proxy GB, and CAPTCHA probability when enqueueing tasks so finance can tie top-of-funnel demand to browser spend.
2. **Keep queue health visible.** Emit metrics for `activeSessions`, `queuedJobs`, and `planCap` so alerts fire before you slam into RPS ceilings.
3. **Segment by trust level.** Give each queue its own proxy policy and retry budget, then tag logs with that policy to see which customers burn the most.
4. **Audit retries weekly.** Any workflow with more than two retries per success should either gain better selectors, a human approval step, or be paused until it pencils out.
5. **Rotate evidence storage.** Ship logs, files, and replays to your own storage as soon as the job ends so you can reduce on-platform retention unless compliance requires longer windows.

## Trade-offs and limits
- Steel Cloud still caps session length at 24 hours on Pro. Long-running portal harvesters should checkpoint work into Files API and resume in a fresh session.
- Managed proxies charge by bandwidth even when challenges fail, so do not leave high-bitrate downloads tunneled through stealth endpoints without reason.
- Retention tops out at 14 days on Pro. If audit requirements need more, export traces immediately.
- Overage mode lets you spend up to triple your monthly credits at plan rates, but it does not bypass concurrency caps. Treat it as surge handling, not a substitute for real capacity planning.

## Next steps
- Map your current queue depth, runtime, and proxy usage against the [Pricing and Limits](https://docs.steel.dev/overview/pricinglimits) table, then confirm whether the plan cap or budget is the real constraint.
- Revisit the [Steel Local vs Steel Cloud](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud) guide if you are still straddling both models and paying for fixes twice.
- Wire `session.id` and proxy usage into your monitoring so leaks show up before invoices. The [Sessions API overview](https://docs.steel.dev/overview/sessions-api/overview) and [proxy guide](https://docs.steel.dev/overview/stealth/proxies) cover the exact parameters.

Humans use Chrome. Agents use Steel.
