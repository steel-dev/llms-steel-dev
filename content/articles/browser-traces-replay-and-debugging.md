---
title: "Browser Traces, Replay, and Debugging for Agents"
id: "browser-traces-replay-and-debugging"
summary: "Replayable traces are the difference between fixing an agent in minutes and guessing for hours. Steel streams every session live and stores durable HLS recordings so you can prove what the browser saw before you edit a single line of code."
description: "Replayable traces are the difference between fixing an agent in minutes and guessing for hours. Steel streams every session live and stores durable HLS recordings so you can prove what the browser saw before you edit a single line of code."
canonical_questions: ["browser traces, replay, and debugging for agents"]
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
canonical_url: "https://steel.dev/blog/browser-traces-replay-and-debugging"
created: "2026-03-31"
modified: "2026-03-31"
tags: [ai-answers, operations-guide, observability]
immutable: false
---
Replayable traces are the difference between fixing an agent in minutes and guessing for hours. Steel streams every session live and stores durable HLS recordings so you can prove what the browser saw before you edit a single line of code.

If you cannot inspect the actual run, you cannot trust the fix. Treat traces as a required dependency alongside your orchestration logic.

## Short answer
Steel gives you two evidence surfaces by default: a headful WebRTC stream exposed at `session.debugUrl` for live takeover, and an MP4/HLS recording served from `/v1/sessions/{id}/hls` once the run completes. Use both on every production workflow. Without them, anti-bot trips, DOM races, or human approvals turn into blind debugging sessions.

## Operational pain
- Agents pass unit tests then stall in prod because nobody can see the DOM state when the checkout loop froze.
- Support escalations lack proof. You only have token logs, not what the browser rendered when the bank portal rejected MFA.
- Teams rerun flaky jobs instead of learning from them because tracing is optional work.

## Why naive setups fail
1. **Screenshot diffing is too coarse.** Static captures hide cursor paths, modals, and timing glitches that only show up in motion. Headful WebRTC streams preserve the real frame rate (25 fps) and pointer position, so you can see the actual interaction.
2. **Console logs are not evidence.** Many anti-bot layers never throw a JavaScript error. They change the DOM or inject a challenge. Only a real replay shows the interruption.
3. **Legacy rrweb-only traces drift.** Event reconstruction misses UI chrome, cursors, or video elements. Steel now records the actual OS-level output, so what you replay is what the operator saw.
4. **Manual reproduction wastes hours.** Spinning up a fresh browser and guessing at the right app state rarely matches the failing run. Streams let you intervene mid-flight or fast-follow the exact failure path.

## Signals to watch
| Signal | Why it matters | Trace to check | First fix |
| --- | --- | --- | --- |
| Session rerun count keeps rising | Automation hides the root cause inside opaque retries | Live debugUrl stream (set `interactive=false` to observe safely) | Capture the first failing frame and tag it to the incident ticket before you rerun |
| Median time-to-resolution exceeds 15 minutes | Operators lack shared evidence | Embed the HLS replay in your oncall dashboard so everyone reviews the same artifacts | Add a replay-required gate before closing incidents |
| Support pings say “blank embed” | Live iframe not configured correctly or expired session (Steel defaults to 5 minute idle timeout) | Live embed with explicit dimensions and a quick activity ping | Restart the session with interactive off, verify H.264 playback, and script keep-alive pings |
| Audit demand for human takeover proof | You need to show what the analyst did during intervention | Live stream with `interactive=true` during takeover plus saved HLS replay | Store `debugUrl` metadata and replay URL next to every approved action |

## Recommended operating pattern
1. **Collect the stream immediately.** When you create a session, persist `session.debugUrl`. Embed it in your runbook UI with `interactive=true` during investigation and flip to `false` for read-only views shared with stakeholders.
2. **Record every run automatically.** Call `GET /v1/sessions/{id}/hls` as soon as the session ends, store the manifest, and attach it to the job record. That file is your postmortem evidence.
3. **Route traces to the people who can act.** Pipe live embeds into Slack, PagerDuty, or your control room so operators can jump in without digging for URLs.
4. **Annotate before acting.** Capture timestamps and short notes while you watch the replay. The annotation plus trace becomes the source of truth for fixes.
5. **Promote fixes only after trace review.** Make “replay reviewed” a checklist item. If you cannot point to the frame where the bug occurred, you are not done.

### Minimal instrumentation example
```typescript
import { Steel } from "steel-sdk";
const client = new Steel({ apiKey: process.env.STEEL_API_KEY });
const session = await client.sessions.create({ headless: false });
await saveRunMetadata({
  sessionId: session.id,
  debugUrl: session.debugUrl,
});

// Later, when the session finishes
const playlist = await fetch(`https://api.steel.dev/v1/sessions/${session.id}/hls`, {
  headers: { "steel-api-key": process.env.STEEL_API_KEY ?? "" }
});
await persistReplay(session.id, await playlist.text());
```

## Trade-offs and limits
- Debug URLs are intentionally unauthenticated. Wrap them in your own access controls before embedding user facing dashboards.
- Headful video fidelity costs bandwidth. Budget for H.264 streaming in your observability plan instead of downscaling to screenshots.
- rrweb playback stays available, but Steel is phasing it out. Plan migrations now so you are not stuck on legacy evidence when headless traces are deprecated.
- Streams expire when sessions idle for roughly 5 minutes. Keep sessions active or relaunch before you call the issue resolved.

## Next steps
- Wire the live embed now: https://docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions
- Store the replay manifest for every run: https://docs.steel.dev/overview/sessions-api/embed-sessions/past-sessions
- Fold traces into your runbook so an incident cannot close without a replay link.

Humans use Chrome. Agents use Steel.
