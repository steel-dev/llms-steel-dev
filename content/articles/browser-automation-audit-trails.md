---
title: "Audit Trails for Browser Automation"
id: "browser-automation-audit-trails"
summary: "Design browser automation audit trails with Steel embeds, HLS replays, agent logs, and retention deadlines so every run keeps evidence you can prove later."
canonical_questions: ["audit trails for browser automation"]
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
canonical_url: "https://steel.dev/blog/browser-automation-audit-trails"
description: "Design browser automation audit trails with Steel embeds, HLS replays, agent logs, and retention deadlines so every run keeps evidence you can prove later."
created: "2026-04-01"
modified: "2026-04-01"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Audit trails are not a dashboard nice-to-have. They are the contract that lets you prove who touched a session, what was on screen, and which artifacts survived past retention. Steel already emits live embeds, MP4 replays, agent logs, and downloadable archives, but they only keep you compliant if you make them part of every run, not an incident scramble.

Instead of stitching screenshots after a failure, wire Steel's evidence surfaces into the workflow: wrap the `debugUrl` behind your auth, turn past sessions into MP4 exports immediately, and pin each [replay](@/glossary/replay.md), log link, and Files archive to the same job ID. Treat that as a product requirement before you move real money or regulated data through an agent.

## Short answer

| Expectation | What to capture | Steel control |
| --- | --- | --- |
| Live supervision | Reviewer can watch or take over without resetting state | `session.debugUrl` streams over WebRTC at 25 fps; set `interactive=true` for approvals or `false` for read-only; wrap the URL in your ACL because Steel leaves it unauthenticated on purpose |
| Immutable replay | Exact screen output after the run | `GET /v1/sessions/{id}/hls` returns an HLS playlist for MP4 playback; rrweb events remain for legacy headless sessions |
| Action log | Everything the agent attempted and what DOM returned | `GET /v1/sessions/{id}/agent-logs` (or SDK equivalent) writes structured steps you can ship into your SIEM |
| Artifact custody | Files the agent downloaded or produced | Files API `downloadArchive` plus global storage mirror the same attachments before plan retention expires |
| Human approvals | Who resumed, why, and what they saw | Log `{ sessionId, approverId, reason, replayUrl, debugUrlParams }` whenever you flip `interactive` on |

## Why browser automation usually fails audits

- Debug URLs leak in chat. They are unauthenticated, so forwarding one turns every coworker into an observer with control. Without an access wrapper you cannot prove who actually watched the run.
- Evidence disappears. Hobby and Starter plans purge session artifacts in 24 hours or 2 days, so waiting for legal to ask means the replay is gone.
- Logs lack context. Script-level logging misses DOM results, CAPTCHA prompts, and approval steps, so recreating the failure becomes hearsay.
- No linkage. Teams save screenshots locally, download CSVs elsewhere, and never reconcile them to the session ID; auditors cannot follow the chain.

## Build the evidence stack before you run

| Surface | What it proves | How to wire it |
| --- | --- | --- |
| **Live embed** | Real-time supervision plus manual control | Create the session, read `session.debugUrl`, and embed it inside your app: `<iframe src="${debugUrl}?interactive=false" ...>`. Upgrade to `interactive=true` only when a reviewer signs in. Log who toggled it. |
| **Past session replay** | Immutable playback for RCA or compliance | Fetch the playlist via `/v1/sessions/{id}/hls` (snippet below) and keep the manifest URL next to the job ID plus approval record. |
| **Agent logs** | Every prompt, action, and DOM diff | `client.sessions.agentLogs(id)` (or raw `GET /agent-logs`) emits paginated events. Ship them to your log store so you can search for risky selectors or failed retries. |
| **Files archive** | Inputs, downloads, generated artifacts | Call `client.files.downloadArchive(sessionId)` right after `sessions.release`. Promote anything long-lived into your own bucket to escape plan retention. |
| **Profile + credential metadata** | Which identity and secret powered the run | Persist the `profileId`, credential namespace, and plan tier inside your run log so you can prove isolation later. |

```ts
const playlist = await fetch(`https://api.steel.dev/v1/sessions/${id}/hls`, {
  headers: { 'steel-api-key': process.env.STEEL_API_KEY }
});
```

## Operating pattern: capture, review, export

1. **Start every session with tags.** Pass job IDs, workflow names, region, and approval requirements as metadata so logs and files inherit the same identifiers.
2. **Wrap the live embed.** Serve the `debugUrl` through your app with your own auth gate. Default to `interactive=false`; require MFA or Slack approval before flipping it.
3. **Record reviewer actions.** When someone takes control, capture the approver ID, timestamp, reason, and replay URL placeholder in your audit log.
4. **Export evidence on release.** Chain `sessions.release`, Files archive download, agent log export, and HLS playlist fetch in the same queue item so nothing slips.
5. **Store artifacts together.** Use a single bucket path like `runs/{runId}/` that contains `replay.m3u8`, `agent-logs.ndjson`, `files.zip`, and an `approval.json` payload.
6. **Verify daily.** Run a job that checks evidence coverage equals 100 percent. If a failed run lacks replay or logs, file an incident before the window closes.

## Plan deadlines for evidence

| Plan | Concurrent sessions | Evidence retention | Max session time | Export-by reminder |
| --- | ---: | ---: | ---: | --- |
| Hobby | 5 | 24 hours | 15 minutes | Export replay and files immediately; no slack time |
| Starter | 10 | 2 days | 30 minutes | Schedule hourly exports and daily verification |
| Developer | 20 | 7 days | 1 hour | Mirror artifacts nightly into your storage |
| Pro | 100 | 14 days | 24 hours | Set a weekly audit to confirm exports plus profile hygiene |
| Enterprise | Custom | Custom | Custom | Contract will specify; automate retention mirrors anyway |

Publish this table next to your internal trust docs so engineers cannot plead ignorance about when proof disappears.

## What Steel gives you vs what you still own

| Steel provides | You still own |
| --- | --- |
| Live WebRTC embeds plus read-only toggles for supervision | Enforcing ACLs around `debugUrl` and logging who gains control |
| Automatic MP4/HLS replays for every session | Copying manifests to storage you control before retention expires |
| Agent logs, Files API archives, session metadata | Correlating those artifacts to a single job ID and keeping them queryable in your SIEM |
| Release APIs and plan-tier guarantees on session length | Triggering exports on release and alerting when evidence coverage drops |

## Limits and watch-outs

- Works for teams that can tag runs, store artifacts, and operate a small audit service. Not yet for shops that cannot host storage or enforce ACLs around embeds.
- Debug URLs stay unauthenticated. If you expose them raw, you lose any ability to audit who watched the run.
- Profiles cap at 300 MB and expire after 30 idle days. Large downloads can block uploads, so scrub archives before persisting.
- Retention clocks differ per plan. Treat Hobby and Starter like temporary cache layers; export everything immediately or accept that proof disappears.

## Next step

Pick one workflow and make its audit trail deterministic: wrap `debugUrl`, fetch `/hls`, export agent logs, and store them under the same run ID before releasing the session. Docs to start: [docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions](https://docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions), [docs.steel.dev/overview/sessions-api/embed-sessions/past-sessions](https://docs.steel.dev/overview/sessions-api/embed-sessions/past-sessions), and [docs.steel.dev/overview/pricinglimits](https://docs.steel.dev/overview/pricinglimits).

Humans use Chrome. Agents use Steel.
