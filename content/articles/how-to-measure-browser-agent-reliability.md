---
title: "How to Measure Browser Agent Reliability"
id: "how-to-measure-browser-agent-reliability"
summary: "Track startup p95, first-action success, completion without retries, manual interventions, and evidence coverage to judge browser agent reliability with Steel."
canonical_questions: ["how to measure browser agent reliability"]
intent: "reference"
entity: "reliability"
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
canonical_url: "https://steel.dev/blog/how-to-measure-browser-agent-reliability"
description: "Track startup p95, first-action success, completion without retries, manual interventions, and evidence coverage to judge browser agent reliability with Steel."
created: "2026-03-31"
modified: "2026-03-31"
tags: [metrics-guide, reliability, browser-agents]
immutable: false
---
## Short answer
Browser agent reliability is not a vibe check; it is four numbers you can audit every hour: session startup success and p95 latency, first action success, completion rate without retries, and how often humans or missing evidence stall the run. Steel’s [remote benchmark](https://steel.dev/blog/remote-browser-benchmark) hits 0.89 s average startup, 1.09 s p95, and 1.34 s p99 across 5,000 runs with 100 percent success; that is 1.70×–8.95× faster than the other providers we tested, so treat 1.2 s p95 and 99.5 percent success as the first red lines. When a metric breaks, check which plan tier hit it: Steel Local is effectively a single-session control plane while Steel Cloud Starter, Pro, and Enterprise tiers stretch into the tens or hundreds of concurrent sessions, so annotate every scoreboard row with the plan and region before trusting an average.

Instead of counting how many jobs eventually finished, tag each Steel session with your queue ID, emit stage timers from `sessions.create` through `release`, and record whether each failure had a [replay](@/glossary/replay.md), logs, and an operator intervention. Publish those metrics with plan-tier utilization so alerts fire before startup p95 drifts past benchmark values; the loop shows whether you should fix [proxies](@/glossary/proxies.md), selectors, profiles, or human approval policies before touching model prompts.

## What to measure

### Steel benchmark baselines
| Signal | Steel baseline | Why it matters |
| --- | --- | --- |
| Startup success & p95 | 100% success, 0.89 s average, 1.09 s p95, 1.34 s p99 across 5,000 runs. | Treat 1.2 s p95 as your red line; exceeding it means the control plane is starving queues before automation even touches the DOM. |
| Control-plane tax | 229 ms total for create + release (25.6% of lifecycle). | When this number doubles, retries and warm pools burn your concurrency budget instead of shipping work. |
| First action (connect + first `goto`) | 665 ms average for connect plus the first DOM-ready navigation. | Spikes here mean anti-bot controls or layout drift is choking agents before any useful work happens. |

These baselines are the default scoreboard targets until you log fresher numbers in your own environment.

### Reliability metrics that matter
| Metric | Why it matters | How to calculate | Steel signal |
| --- | --- | --- | --- |
| Startup success & p95 | Control plane health determines whether you ever reach the page; queue cascades begin when creations fail or p95 crosses 1.2 s. Steel’s benchmark baseline is 0.89 s avg, 1.09 s p95 with 0 failures over 5,000 runs. | `successful_creates / total_creates` plus percentile on `(session_created_at → first_connect_at)` per region and plan. | Sessions API create/release telemetry, Pricing/Limits caps, remote-browser-benchmark reference runs. |
| First action success | Anti-bot walls and selector drift usually appear on the first DOM mutation, not the final output. Catching it here prevents wasted browser hours. | Count navigation or `page.goto` completions on the first action per session, split by proxy, UA, and mobile mode flags. | `agentLogs` timeline + Playwright event hooks + `sessionContext` metadata. |
| Workflow completion vs retries | Reliability is “done on first run”; retries mask failure costs and double proxy, CAPTCHA, and human review spend. | `completed_without_retry / total_runs` and `avg_retries_per_success`, segmented by workflow. | Job queue + `session.id` + `releaseAll` to ensure cleanup before re-queueing. |
| Manual intervention rate | Every approval drains latency budgets and exposes UX issues. Keep the rate low by diagnosing root causes early. | `(human_interventions / total_runs) * 100`, recorded via embed approvals or ops dashboards. | `debugUrl` embed callbacks + approval logs + operator queue metadata. |
| Plan-tier saturation | Steel Local is effectively one concurrent session while Steel Cloud Starter/Pro stretch into the tens or hundreds; once a tier hits 90 percent of its concurrency or RPS cap, startup p95 will spike before success drops. | Track `(active_sessions / plan_cap)` and `(requests_per_second / plan_cap)` per plan tier; alert when either crosses 0.9 for 5 minutes. | Pricing/Limits table for caps + Sessions API `list` to count active sessions; annotate scoreboard rows with plan tiers. |
| Evidence coverage | If a failure lacks replay/logs/files, it cannot improve reliability. Evidence coverage should be 100 percent for failed runs. | `failures_with_replay / total_failures` plus storage lag for logs/files exports. | HLS replay URLs, MP4 exports, Files API attachments, `agentLogs` pagination. |

### Segment every metric
A blended reliability number hides which region, plan tier, or [proxy](@/glossary/proxies.md) pool poisoned the curve. Slice the scoreboard before you trust it.
| Dimension | Why it matters | Implementation hook |
| --- | --- | --- |
| Region & data center | ISPs throttle differently; a noisy geography can sink your p95 without impacting other customers. | Tag sessions with `region` on create, then chart startup and first-action metrics per region before aggregating. |
| Plan tier & warm pool size | Steel Local (~1 concurrent session) vs Steel Cloud (hundreds), plus Starter vs Pro caps, control how many sessions can boot before the queue saturates. | Join session metrics with plan metadata from Pricing/Limits and alert when any tier crosses 90% of its concurrency or RPS ceiling. |
| Proxy or fingerprint class | Headless vs headful, mobile mode vs desktop, or residential vs datacenter proxies fail in different ways. | Store `proxyPool`, `fingerprintMode`, and `mobileMode` flags on each session and pivot success rates accordingly. |
| Workflow or domain | Selectors and approvals are domain-specific; hiding them inside an average keeps flaky flows invisible. | Derive a `workflow` tag from the job queue and publish per-domain slices in the same dashboard. |
| Human intervention reason | Approvals are not free; trending “KYC stuck” vs “OTP missing” guides fixes faster than knowing the blended rate. | Require structured reason codes in your approval UI and count them alongside the intervention metric. |

#### Plan-tier guardrails (from Pricing/Limits)
Use the current [Pricing/Limits](https://docs.steel.dev/overview/pricinglimits) table to annotate each slice with the right ceiling so alerts trigger before a plan starves the queue. These are the published caps today; if they change, update the scoreboard in lockstep.
| Plan tier | Concurrent sessions cap | Requests per second cap | Reliability response |
| --- | --- | --- | --- |
| Hobby ($0) | 5 | 1 | Treat it as a single-developer sandbox; if startup success or p95 drift, move production jobs off this tier immediately. |
| Starter ($29) | 10 | 2 | Alert as soon as a workflow holds ≥9 sessions or 90% of RPS for 5 minutes; pre-warm Steel Cloud pools or stage upgrades before approvals pile up. |
| Developer ($99) | 20 | 5 | Slice metrics per region to avoid burning the cap on one geography, and sweep stuck sessions with `releaseAll` when utilization sticks above 90 percent. |
| Pro ($499) | 100 | 10 | Publish per-region dashboards; most regressions here come from evidence export lag or manual approvals blocking releases rather than raw concurrency. |
| Enterprise | Custom | Custom | Mirror the contracted ceiling and stop trusting “unlimited” without on-call automation; negotiate burst headroom so alerts stay meaningful. |

## Why completion rates lie
A green “workflow succeeded” light usually hides the 30 seconds you spent waiting for a retry, the three CAPTCHA solves it consumed, or the operator who had to click Continue inside a `debugUrl`. Reliability falls apart when you only measure the last stage:
- Queues can drain minutes before a session ever boots when concurrency caps (5, 10, 20, 100) are saturated.
- Anti-bot blocks often kill the first navigation, so counting final output means you miss the step where you should rotate proxies or switch to headful mode.
- Retries reset session IDs, which nukes the evidence trail unless you store it per attempt.

Measure every stage. If startup is healthy but first actions fail, you need better fingerprints. If first actions pass but completion tanks, explore selectors, credentials, or human approvals. If completion looks perfect but evidence coverage drops below 100 percent, you are flying blind during postmortems.

## Instrument the loop
1. **Tag sessions with job context.** Store `session.id`, workflow name, proxy mode, and profile ID alongside your queue metadata so metrics can pivot on any dimension.
2. **Capture stage timers.** Record `(create → connect)`, `(connect → first action)`, `(first action → completion)`, and `(completion → release)` durations; export them to Prometheus or whatever time-series system you already trust.
3. **Log agent actions.** Call `/v1/sessions/{id}/agent-logs` (or stream live logs) to align model steps, CDP commands, and human inputs with timestamps.
4. **Store evidence immediately.** Save HLS URLs, MP4 downloads, screenshots, and scraped files using the Files API so every failure has replayable proof even after the 14-day retention window.
5. **Track interventions as events.** Your approval UIs should emit structured telemetry when someone takes control via `debugUrl` or a live embed, including reason codes and elapsed time.
6. **Automate cleanup.** Use `release` and `releaseAll` after every run, and record whether cleanup succeeded; stuck sessions distort both cost and perceived reliability.
7. **Segment by region and plan.** Break out success, latency, and intervention metrics by region, proxy type, and plan tier so a noisy geography or price plan cannot hide systemic regressions.
8. **Publish a live scoreboard.** Pipe these numbers into a shared dashboard (Grafana, Metabase, spreadsheets), pin the target + alert threshold for each metric, and assign a single owner so reliability drift is visible without digging through traces.
9. **Wire plan caps to alerts.** Pull concurrency and RPS ceilings straight from [Pricing/Limits](https://docs.steel.dev/overview/pricinglimits) and annotate every scoreboard row with the plan tier; alert early when startup p95 creeps upward while a plan is within 10 percent of saturation so ops can flip workloads from Steel Local (~1 concurrent session) to Steel Cloud (100+), or between Starter/Pro tiers, before prompts waste browser slots.

#### Stage instrumentation cheat sheet
| Stage | Fields to capture | Metric unlocked |
| --- | --- | --- |
| Create → connect | `session.id`, queue/job ID, region, plan tier, proxy pool, timestamps for `sessions.create` and first CDP connect | Startup success %, p95 latency, plan saturation alerts |
| First connect → first action | `agentLogs` event ID, action type (`goto`, selector click), DOM target, fingerprint mode (headful/mobile), proxy label | First-action success %, domain-by-domain anti-bot diagnosis |
| First action → completion | Workflow name, profile ID, credentials namespace, retry counter, navigation + form checkpoints | Single-pass completion %, retry rate, flaky selectors or auth drift |
| Completion → release | Cleanup duration, release status, `releaseAll` result, stuck-session flag | Control-plane tax %, stranded session detection, concurrency reclaim rate |
| Failure evidence | Replay URL, MP4 export path, Files API payload IDs, log export status | Evidence coverage %, audit readiness, incident reproducibility |
| Manual intervention | Approver ID, reason code, elapsed review time, embed/`debugUrl` path, resumed session ID | Intervention rate, approval backlog trends, human-time cost per workflow |

#### Session tag dictionary
Standardize the tags you emit on every session so slicing never turns into regex archaeology later.
| Tag | Purpose | Example values |
| --- | --- | --- |
| `jobId` | Join queue records, retries, and metrics without guessing which run maps to which artifact. | `claims-sync-2026-03-31-00012` |
| `workflow` | Segment reliability per business flow or domain and keep alert routing clean. | `kyc-verify`, `billing-export`, `claims-sync` |
| `region` | Compare startup and first-action health per data center before trusting blended numbers. | `us-east-1`, `eu-west-2`, `ap-south-1` |
| `planTier` | Watch for starter/pro plan saturation and know when to upgrade or shard workloads. | `starter`, `pro`, `enterprise` |
| `proxyPool` | Spot when one proxy vendor or residency class tanks first-action success. | `steel-resi`, `byo-dc`, `headful-lithuania` |
| `fingerprintMode` | Track headless/headful/mobile toggles so anti-bot fixes have evidence. | `headful`, `stealth`, `mobile` |
| `profileId` | Tie completion and retry metrics to the underlying auth state. | `merchant-portal-prod`, `issuer-sandbox` |
| `mobileMode` | Confirm when mobile DOMs were requested so DOM selectors and evidence stay aligned. | `true`, `false` |
| `approvalReason` | Trend manual interventions by root cause rather than by gut feel. | `otp-missing`, `kyc-review`, `compliance` |
| `evidenceStatus` | Ensure replay/log exports completed; alert when anything ships without proof. | `complete`, `missing-replay`, `missing-logs` |

### Retire fake reliability proxies
Vanity dashboards hide the root cause. Kill these summary metrics so the scoreboard reflects reality.
| Proxy signal | Why it lies | What to track instead |
| --- | --- | --- |
| Jobs completed per hour | Retries, manual approvals, and stuck cleanup still count as “success,” so you congratulate a workflow that burned double the time and proxies. | Startup success, single-pass completion, and manual intervention rate per workflow and region. |
| Queue depth or wait time | A long queue tells you demand beats supply, not why sessions fail to boot. | Startup p95 per region/plan plus first-action success segmented by proxy pool. |
| CPU or RAM usage | Host metrics improve even when workflows die on OTP or CAPTCHA steps; ops fix VMs instead of selectors. | Evidence coverage, approval reason codes, and retry rate; these point straight to automation bugs. |
| Prompt/response count | LLM turn counts fluctuate with copy tweaks but say nothing about browser survival. | Completion latency per attempt and retries per success so you see wasted browser hours. |

#### Manual intervention event schema
Audit every approval the same way you log incidents: structured fields keep reviews auditable and help you replay what happened inside `debugUrl` or live embeds.
| Field | Why it matters | Example |
| --- | --- | --- |
| `approverId` | Tie the decision back to a human so you can audit access and follow up on outliers. | `ops-lead-07` |
| `reasonCode` | Buckets interventions into actionable categories instead of Slack anecdotes. | `otp-missing`, `kyc-handoff`, `layout-shift` |
| `sessionId` | Keeps the approval linked to the evidence, retries, and cleanup events. | `sess_a12bc3` |
| `entryTimestamp` | Shows how long the run waited before someone took control. | `2026-03-31T12:04:22Z` |
| `elapsedMs` | Helps quantify the human-time tax per workflow and set staffing thresholds. | `18500` |
| `resumeAction` | Tells on-call whether the same session resumed, restarted, or was killed. | `resume_same_session` |
| `embedPath` | Records whether the takeover came from `debugUrl`, an iframe, or a custom review UI. | `embed/live/claims` |

#### Evidence artifact log schema
Treat replay, log, and file exports as first-class records; missing any of them should block your reliability publish just like a failed metric.
| Artifact field | Purpose | Example |
| --- | --- | --- |
| `replayUrl` | Links reviewers to the HLS playlist or MP4 so they can see the failure without rerunning it. | `https://hls.steel.dev/.../master.m3u8` |
| `mp4Path` | Guarantees you can keep the proof past Steel’s retention window by mirroring to your storage. | `s3://agents/runs/claims-sync-12.mp4` |
| `logsExportId` | Confirms agent logs finished exporting; without it you cannot align model actions to DOM state. | `log_exp_92ff` |
| `filesAssetIds` | Tracks downloaded spreadsheets, PDFs, or screenshots tied to the run. | `file_123`, `file_124` |
| `status` | Shows whether the evidence bundle is complete, partial, or missing to trigger alerts. | `complete` |
| `expiresAt` | Forces teams to pull artifacts before the 14-day default window closes. | `2026-04-14T00:00:00Z` |

#### Example: tag and emit metrics in code
```ts
import { SteelClient } from "@steeldev/sdk";
import { chromium } from "playwright";

const metricsTags = { workflow, region, plan, proxyPool };

const createdAt = Date.now();
const client = new SteelClient({ apiKey: process.env.STEEL_API_KEY });
const session = await client.sessions.create({ tags: { jobId, workflow } });

const connectStart = Date.now();
const browser = await chromium.connectOverCDP(session.cdpUrl);
metrics.observe("session_startup_ms", connectStart - createdAt, metricsTags);

const page = await browser.newPage();
await page.goto(firstUrl, { waitUntil: "domcontentloaded" });
metrics.observe("first_action_ms", Date.now() - connectStart, metricsTags);

try {
  await runWorkflow(page);
  metrics.increment("workflow_completion", { ...metricsTags, result: "success" });
} catch (error) {
  metrics.increment("workflow_completion", { ...metricsTags, result: "failure" });
  await uploadReplayEvidence(session.id); // Files API helper keeps evidence coverage at 100%.
  throw error;
} finally {
  await client.sessions.release(session.id);
}
```
Tag the session the moment it exists, emit each stage duration as soon as it finishes, and treat replay uploads plus releases as part of the same function. This keeps every metric synchronized with artifacts and prevents retries from erasing the evidence trail.

#### Scoreboard payload example
Ship one normalized record per workflow slice so alerting and dashboards stay consistent.
```json
{
  "metric": "startup_success",
  "value": 0.997,
  "windowMinutes": 5,
  "tags": {
    "workflow": "claims-sync",
    "region": "us-east-1",
    "planTier": "pro",
    "proxyPool": "steel-resi"
  },
  "evidence": {
    "replayUrl": "https://hls.steel.dev/runs/claims-sync-12/master.m3u8",
    "logsExportId": "log_exp_92ff",
    "missingEvidence": false
  },
  "alert": {
    "threshold": 0.995,
    "action": "scale_control_plane"
  }
}
```

##### Scoreboard payload fields
| Field | Why it exists | Notes |
| --- | --- | --- |
| `metric` | Names the signal so Grafana, Metabase, or alerting routers can fan out without guessing column names. | Keep it stable (`startup_success`, `first_action_success`, etc.) so downstream joins stay simple. |
| `value` | Stores the aggregated measurement you are comparing to the alert threshold. | Use ratios (0.997) or milliseconds depending on the metric; mixing units muddies alerts. |
| `windowMinutes` | Documents the roll-up window so humans know whether this was a 1-minute spike or a 1-hour regression. | Match your alert windows (5, 15, 30) and update it when playbooks change. |
| `tags.*` | Provide segmentation context (workflow, region, plan tier, proxy pool) so you can page the right owner. | Emit every tag defined in the dictionary; dashboards stay filterable without custom SQL. |
| `evidence.*` | Links the metric to replay/log artifacts so responders can inspect proof before muting alarms. | Set `missingEvidence` to `true` anytime a replay/log export failed; that alone should trigger an incident. |
| `alert.threshold` | Shows the contract baked into the runbook so nobody debates severity mid-incident. | Pair with the same target you published in the reliability scoreboard table. |
| `alert.action` | Tells on-call owners exactly what to do (scale pool, rotate proxies, pause queue) when the alert fires. | Keep it short and imperative so it can be pasted directly into PagerDuty or Slack. |

### Reliability scoreboard example
Keep one aggregated view for executives, but publish per-region, per-proxy, and per-plan slices so a noisy geography cannot hide a failing control plane.
| Metric | Target | Alert threshold | Owner & response |
| --- | --- | --- | --- |
| Startup success & p95 | ≥99.5%, p95 <1.2 s (Steel benchmark: 0.89 s avg / 1.09 s p95 / 1.34 s p99) | Success <99% or p95 ≥1.3 s for 5 min | Control-plane on-call: scale plan or warm pool, inspect control plane logs, fail open to Steel Cloud. |
| First action success | ≥97% | Drops <95% per domain | Anti-bot owner: rotate proxies, test headful/mobile mode, refresh selectors. |
| Completion without retry | ≥90% single-pass | Retries ≥1 per success | Workflow maintainer: split flows, persist profiles, fix flaky inputs. |
| Manual intervention rate | ≤5 per 100 runs | ≥10 per 100 runs or 3 consecutive spikes | Trust desk lead: pair approval reason with bug tickets; fix automation before adding staff. |
| Plan-tier saturation | <90% of concurrency or RPS cap per tier | ≥90% for 5 consecutive minutes | Control-plane + growth owners: shift jobs off Steel Local (~1 session) onto Steel Cloud (100+), or move Starter workloads to Pro/Enterprise tiers before queues stall. |
| Evidence coverage | 100% of failures | Any miss | Observability owner: halt deploys, patch replay/log exporters, treat as incident. |

### Alert wiring checklist
Tie each metric to an automated action so nobody has to discover regressions in Slack.
| Condition | Trigger window | Automated action |
| --- | --- | --- |
| Startup success <99% or p95 ≥1.3 s | 5-minute rolling window per region/plan | Page control-plane on-call, auto-scale warm pools, and temporarily cap queue fan-out. |
| Plan tier ≥90% of its concurrency or RPS cap | 5-minute rolling window per plan tier | Auto-scale the pool or shift workloads to the next Steel Cloud tier before saturation drags startup p95 above the benchmark, then notify owners to rebalance queue fan-out. |
| First action success dips below 95% for a domain | 15-minute rolling domain slice | Flip impacted workflows to headful or mobile mode and force proxy rotation until green again. |
| Manual intervention rate ≥10 per 100 runs | 30-minute rolling workflow slice | Pause the workflow queue, create a bug ticket with the last replay, and require incident review before resuming. |
| Evidence coverage <100% for failures | Immediate detection | Block deploys, open a Sev-2, and backfill missing replays/log exports before allowing new releases. |

## Run a daily reliability review
A scoreboard only matters if someone reads it. Hold a 10-minute cadence that keeps metrics, evidence, and owners honest:
1. Export the last 24 hours of scoreboard rows per workflow, region, and proxy slice; highlight any metric that crossed its alert threshold even once.
2. Pull the top three manual intervention reason codes and pin their replays plus agent logs to the bug tracker before the next queue resumes.
3. Verify evidence coverage stayed at 100 percent; if any run is missing replay or logs, open an incident before green-lighting deployments.
4. Compare startup success by plan tier against [Pricing/Limits](https://docs.steel.dev/overview/pricinglimits); if a cap is within 10 percent, scale warm pools or fail workloads over to Steel Cloud before the next spike.

| Ritual | Cadence | Questions to answer | Output |
| --- | --- | --- | --- |
| Scoreboard export | Daily (10 min) | Which workflow-region slices crossed targets or alert thresholds? | Highlighted table ready for owners to triage. |
| Intervention triage | Daily | Which reason codes consumed the most human time, and do they map to known bugs? | Ticket list with replay/log links per incident. |
| Evidence audit | Daily | Did any failed run miss replays, logs, or files? | Incident report plus export backlog if any gap exists. |
| Plan-cap check | Daily | Which plan tiers or warm pools exceeded 90% of concurrency/RPS? | Action list to shift jobs between Steel Local vs Cloud or across Starter/Pro tiers. |

## Interpret and act on the numbers
| Metric | Healthy range | Escalate when | First move |
| --- | --- | --- | --- |
| Startup success & p95 | ≥99.5% success, p95 <1.2 s, p99 <1.4 s | Success dips, p95 rises, or one region lags | Check plan caps, warm pool size, or proxy health; fall back to Steel Cloud if self-hosted control plane stalls. |
| First action success | ≥97% of sessions clear the first navigation | Drops below 95% or spikes per domain | Switch to headful, enable mobile mode, or attach managed proxies/CAPTCHA solving only on affected flows. |
| Completion without retry | ≥90% single-pass completion, retries ≤0.3 per success | Retries exceed 1 per success or certain flows hog retries | Instrument selectors and form states, persist profiles, or split risky flows into smaller, checkpointed steps. |
| Manual intervention rate | ≤5 per 100 runs for sensitive actions | Hits double digits or trends upward week over week | Patch automation logic, add pre-flight validation, or introduce staged approvals with better evidence to reduce escalations. |
| Evidence coverage | 100% of failed runs have replay + logs | Any gap appears | Fail open by shipping logs/replays to your own storage automatically; treat missing evidence as sev-worthy. |

Use these thresholds to trigger automation: alert when startup success falls for 5 minutes, pause a queue when manual approvals blow past the limit, or ship an incident review when evidence coverage dips at all.

## Steel telemetry map
| Signal | Where it lives | Notes |
| --- | --- | --- |
| Session lifecycle | `sessions.create`, `sessions.get`, `sessions.release`, `releaseAll`, plan caps from Pricing/Limits | Gives raw success/latency plus concurrency saturation data. |
| Agent decisions | `GET /v1/sessions/{id}/agent-logs` | Aligns prompts, actions, and DOM results for both automation and audit trails. |
| Live + replay evidence | `debugUrl` live viewer, HLS playback, MP4 downloads | Embed in ops tools for real-time approvals; store MP4 for forensic review. |
| Artifacts + files | Files API uploads/downloads | Persist structured outputs, DOM snapshots, and attachments without re-running sessions. |
| Context + profiles | `sessionContext`, Profiles API, Credentials API | Show whether failures correlate with stale auth or profile drift; helps size retries vs repairs. |

## Trade-offs and limits
- Steel Cloud caps a single session at 24 hours; jobs longer than that must checkpoint progress into Files API or storage and resume in a fresh session.
- Session replays, logs, and artifacts stick around for up to 14 days on Pro plans. Export anything compliance-sensitive before that window closes.
- Managed proxies and CAPTCHA solving carry per-GB and per-1k charges; reliable metrics should separate baseline runs from stealth-heavy ones so cost spikes are obvious.
- Human approvals still require your own ACLs around `debugUrl` and embeds; Steel provides URLs, not full access control.
- Steel Local is roughly a single-session runway while Steel Cloud plans ship managed stealth plus hundreds of concurrent browsers, so plan-cap alerts should always state which side of that boundary triggered.

## Works for X, not yet for Y
- Works for teams already tagging each queue job with `session.id`, so startup, action, and completion metrics match the evidence you review.
- Works for ops groups layering approvals on top of Steel Cloud because interventions become structured events rather than Slack screenshots.
- Not yet for shops that cannot emit unique job IDs or store replays; the loop depends on correlating metrics to artifacts you can audit later.
- Not yet for workflows that routinely exceed the 24-hour session cap or run disconnected from the network; you will lose telemetry before the run ends.

## Next steps
- Wire your metrics sink to the [Pricing and Limits](https://docs.steel.dev/overview/pricinglimits) table so alarms fire before you exceed concurrency or RPS caps.
- Revisit the [Sessions API overview](https://docs.steel.dev/overview/sessions-api/overview) to confirm which lifecycle events you emit and how to tag them.
- Download the [Remote Browser Benchmark](https://steel.dev/blog/remote-browser-benchmark) raw results so your scoreboard targets inherit the same evidence before you tune thresholds.
- Pair this page with [Why Browser Agents Fail in Production](https://steel.dev/blog/why-browser-agents-fail-in-production) so every failed run stays tied to concrete failure modes and evidence expectations.
- Grab a Steel Cloud API key at https://app.steel.dev so you can tag sessions with these metrics on your next run.

Humans use Chrome. Agents use Steel.
