---
title: "Scaling Browser Automation to Hundreds of Sessions"
id: "scaling-browser-automation-to-hundreds-of-sessions"
summary: "How to size queues, concurrency, and proxy fleets so browser automation runs hundreds of Steel sessions without flaking."
canonical_questions: ["scaling browser automation to hundreds of sessions"]
intent: "reference"
entity: "browser-infrastructure"
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
canonical_url: "https://steel.dev/blog/scaling-browser-automation-to-hundreds-of-sessions"
description: "How to size queues, concurrency, and proxy fleets so browser automation runs hundreds of Steel sessions without flaking."
created: "2026-03-31"
modified: "2026-03-31"
tags: [operations-guide, browser-automation, scaling]
immutable: false
---
## Short answer
Browser automation collapses past the first few dozen runs when concurrency, queue depth, and [proxy](@/glossary/proxies.md) rotation fight each other. [Steel Local](@/glossary/steel-local.md) tops out around one live session, and even the Hobby and Starter cloud plans cap concurrent sessions at 5 and 10 with 1 to 2 requests per second, so bursty workers back up immediately.

Instead of chasing flaky VMs, run [Steel Cloud](@/glossary/steel-cloud.md) like a session factory: pick a plan that matches the number of simultaneous sessions you owe, size your work queue to 70 to 80 percent of that cap, pre-warm [proxies](@/glossary/proxies.md) for each region, and release sessions aggressively so the next job can reuse the slot. Pro plans give you 100 concurrent sessions, 10 requests per second, 24 hour runtimes, managed residential proxies, and releaseAll controls so one engineer can keep hundreds of flows moving.

### Where scaling collapses first
| Symptom | Why it breaks | Steel move |
| --- | --- | --- |
| Local runner saturates after a handful of jobs | Steel Browser (self-hosted) is designed for single-session work and lacks managed stealth, proxies, and concurrency above ~1 | Shift high-volume queues to Steel Cloud which guarantees 100+ concurrent sessions plus managed infra so scaling is configuration, not hardware babysitting. |
| Queue depth outruns the plan cap | Hobby through Pro plans hard limit concurrent sessions (5, 10, 20, 100) and requests per second (1, 2, 5, 10). Extra work just waits and burns SLAs | Track active sessions per plan and throttle your dispatcher so it never exceeds the cap. Upgrade before queue depth stays above 80 percent for more than a few minutes. |
| Long sessions block new work | Lower tiers stop sessions after 15 to 60 minutes; Pro tops out at 24 hours. Jobs that overrun block concurrency even though they no longer do useful work | Pass per-job `timeout` values when you call `sessions.create` and reuse session IDs only when state is required. Release or `releaseAll` when a job stalls. |
| IP bans spike as you add seats | More workers share the same datacenter IP by default; anti-bot systems notice | Flip `useProxy: true` so every session gets a fresh managed residential IP, or attach Bring Your Own proxies with your own rotation logic. |

## Know your concurrency budget before you queue
| Plan | Concurrent sessions | Requests/sec cap | Max session time | When it fits |
| --- | --- | --- | --- | --- |
| Hobby | 5 | 1 | 15 minutes | Local prototyping and CI smoke tests |
| Starter | 10 | 2 | 30 minutes | Small agent pilots where you batch traffic |
| Developer | 20 | 5 | 1 hour | Production-ish runs that still fit inside short flows |
| Pro | 100 | 10 | 24 hours | Real fleets, long tail retries, human handoff loops |
| Enterprise | Custom | Custom | Custom | Regulated workloads or anything above 100 concurrency |

Pick the lowest tier whose request-per-second and session length match your median job. Steel Cloud plans let you burst up to 3x your prepaid browser-hour credits, so you can absorb short spikes without an upgrade mid-incident. When multi-region routing matters, add the `region` flag on session creation so your queue can split capacity by geography instead of fighting over one cluster.

## Run a session factory, not ad hoc launches
1. **Model every task as a job in a durable queue.** Each job carries required resources (profile ID, proxy need, region). The dispatcher app pulls from the queue only when the live session count is below the plan cap minus a buffer.
2. **Pre-warm sessions when workloads are steady.** Use `sessions.create` ahead of peak hours, store the IDs, and keep them alive with realistic `timeout` windows so the automation can attach instantly instead of paying startup time mid-SLA.
3. **Attach the right surface per job.** When connecting via Playwright, Puppeteer, or CDP, tag log lines with `session.id`, proxy choice, and queue job ID so operators can trace a failure back to the precise resource combination.
4. **Release aggressively.** Call `sessions.release` as soon as the job hands off evidence or files. If you suspect a leak, `sessions.releaseAll` resets the fleet and frees browser-hour credits before you hit the cap.
5. **Segment proxies and regions.** Create lightweight proxy pools (managed, BYO, or default) per queue partition and rotate them deliberately so retries do not hammer the same IP and trigger bans.

## Metrics and guardrails to watch
| Signal | Healthy range | Action when it drifts |
| --- | --- | --- |
| Queue depth vs concurrency | Depth stays under 0.8 x concurrent-session cap | If depth stays high, either add more Pro capacity or shed load by pausing ingest. |
| Requests per second | 70 to 90 percent of plan RPS | When you hit the wall, add jitter between job launches or upgrade to avoid server-side throttling. |
| Session age distribution | Most sessions finish before 50 percent of the plan's max duration | Audit jobs that run long, set tighter `timeout` values, or split workflows into multiple sessions. |
| Proxy bandwidth draw | Within the GB allocation implied by your credits | Monitor `proxyBandwidth / credits` so you are not surprised by residential overages; BYO if a single customer needs unusual geo spread. |
| Browser hours spend | Tracks completed jobs within your credit pool | Spikes without higher throughput mean hung sessions; trigger `releaseAll` or auto-scaling logic to recycle them. |

## Trade-offs and limits
- Steel Cloud enforces hard concurrency and RPS caps by plan. If the fleet must hold steady at 150+ live sessions, align with Enterprise early so the cap change is contractual, not a scramble.
- Managed proxies are billed by the GB. Keep large downloads on default datacenter IPs when stealth is not needed, and reserve residential IPs for the forms, checkouts, or OTP flows that gate your automation.
- Sessions still top out at 24 hours even on Pro. For multi-day automations, store state in profiles or files between sessions and spin up a fresh browser rather than fighting the ceiling.
- Long queues hide flaky logic. If a job retries more than twice, push it into a dead letter topic with the session ID so operators can inspect the replay while fresh work keeps moving.

## Next steps
- Map your workload against the [Pricing and Limits](https://docs.steel.dev/overview/pricinglimits) table and resize your plan before the queue becomes the bottleneck.
- Review the [Sessions API overview](https://docs.steel.dev/overview/sessions-api/overview) and [session lifecycle](https://docs.steel.dev/overview/sessions-api/session-lifecycle) docs so your automation actually releases slots when it finishes.
- Decide whether Steel-managed or Bring Your Own proxies fit each queue by reading the [proxy guide](https://docs.steel.dev/overview/stealth/proxies), then bake that routing into your dispatcher.

Humans use Chrome. Agents use Steel.
