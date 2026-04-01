---
title: "Human-in-the-Loop Browser Agents Without Losing State"
id: "human-in-the-loop-browser-agents"
summary: "Add human approvals to browser agents without restarting the session that already holds state."
canonical_questions: ["human-in-the-loop browser agents without losing state"]
intent: "reference"
entity: "browser-automation"
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
canonical_url: "https://steel.dev/blog/human-in-the-loop-browser-agents"
description: "Add human approvals to browser agents without restarting the session that already holds state."
created: "2026-03-31"
modified: "2026-03-31"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Keep the original session alive, hand reviewers the exact same viewport, and resume automation in place. Steel's live `debugUrl` embeds stream the running browser over WebRTC, so approvals happen inside the workflow instead of in a detached screenshot or rebuilt session.

That means no more clearing cookies, replaying logins, or cloning state between tools. You pause the automation, surface the interactive embed with `interactive=true&showControls=true`, log who touched it, then resume the same session ID that was already authenticated.

## Short answer

| If this is true | Do this inside Steel | State impact |
| --- | --- | --- |
| Humans must enter credentials, OTP, or payment info | Pause the session, share the `debugUrl` with `interactive=true` so they type directly into the live browser | Cookies, storage, and network stack stay exactly where the automation left them (<1s startup, up to 24h lifespan). |
| An agent needs approval before risky actions (refunds, deletes) | Gate the action on an approval event, keep the session open, and require a reviewer banner + log entry before resuming | You never recycle the session, so DOM state and auth survive the pause. |
| You owe audit evidence after handoff | Save the same session's `hls` replay and attach the approval log | Replay shows what the reviewer saw, plus who resumed and when. |

## Why approvals usually break state

Most teams still punt approvals to a different tool: send a screenshot to Slack, ask a human to replicate the steps, then rerun the job. That nukes the state three times over: cookies expire, CAPTCHAs reappear, and long forms lose unsaved data. When the workflow touches production accounts, starting a fresh session also means handing real credentials to a person or a vault you don't trust yet.

Steel treats approvals like any other session lifecycle step. [Sessions](@/glossary/sessions.md) cold start in under a second and can stay alive for 24 hours, so you can halt automation without releasing or recreating the browser. Instead of replaying your way back to the risky step, you stream the live browser to the reviewer and give them temporary control.

## Control surfaces that keep humans in the loop

| Surface | Steel primitive | What it solves |
| --- | --- | --- |
| Interactive live view | `session.debugUrl` + `interactive=true` (WebRTC, 25 fps) | Reviewer sees the real DOM, not static captures, and can scroll, click, and enter URLs with full fidelity. |
| Guided navigator | `showControls=true` (legacy headless UI) or your own chrome | Gives reviewers forward/back controls so they don't ask automation to rewind. |
| Scoped takeover window | Application-layer ACL that wraps the unauthenticated debug URL | Anyone with the raw URL can act on the session; wrap it in your auth and track `approved_by` before enabling interactivity. |
| Evidence capture | `GET /v1/sessions/{id}/hls` for MP4/HLS replay | Attach the approval to a replay so you can prove what happened later. |
| Read-only fallback | `interactive=false` when you just need someone to watch | Keeps observers from mutating state when they only need visibility. |

## Recommended operating pattern

1. **Detect the checkpoint.** Instrument your agent to emit an approval-required event (login, payment, destructive action, CAPTCHA). Do *not* release the Steel session yet.
2. **Freeze automation, keep the ID.** Store the session ID and context in your queue. Steel leaves the browser running (default idle timeout is 5 minutes; send a heartbeat if the review might take longer).
3. **Notify the reviewer with a live embed.** Render an iframe like the snippet below, wrapped in your auth. Display a banner so they know they're touching production state.

```tsx
const ApprovalEmbed = ({ debugUrl }: { debugUrl: string }) => (
  <iframe
    src={`${debugUrl}?interactive=true&showControls=true`}
    title="Steel approval session"
    style={{ width: '100%', height: '640px', border: 'none' }}
    allow="clipboard-read; clipboard-write"
  />
);
```

4. **Track who took control.** Before you flip `interactive` on, require the reviewer to confirm identity. Save `{ sessionId, approver, action, timestamp }` to your audit log.
5. **Resume or roll back.** Once the reviewer clicks “Continue,” your agent reconnects to the same session via Playwright/Puppeteer and finishes the workflow. If they decline, call `sessions.release()` so credentials don't linger.

## Safeguards and limits

- **Guard the debug URL.** It is intentionally unauthenticated for fast embeds. Always proxy it through your app or sign short-lived URLs so only reviewers gain access.
- **Remember idle timeouts.** Headful sessions stay alive for 24h but idle timers default to 5 minutes. Send lightweight actions (`page.waitForTimeout(0)`, heartbeats) if an approval might take longer.
- **Plan for concurrency.** Steel Local handles roughly one live session; managed approvals across teams usually need Steel Cloud so 100+ concurrent sessions, Credentials API, and Files API are available.
- **Snapshot before risky edits.** Grab an HLS replay pointer or page screenshot before handing off so you can diff what changed.
- **Annotate trust boundaries.** Interactive embeds mean another human can run arbitrary navigation. Restrict clipboard permissions, mask sensitive UI, or drop to read-only when the reviewer only needs to watch.

## When Steel fits and when it does not

**Use this pattern when:**
- You already rely on Steel sessions for automation and need occasional human intervention without scrapping context.
- The workflow mixes bots and humans on sensitive actions (finance ops, legal filings, vendor portals) and you need a single audit trail.
- You want reviewers to see exactly what the agent sees, including anti-bot prompts or vendor UI bugs.

**Look elsewhere or extend it when:**
- Compliance forbids unauthenticated video streams entirely, so run the embed inside your own zero-trust frame or keep everything on Steel Local inside your VPC until legal approves a managed plan.
- You need fully custom reviewer tooling (annotations, chat, multi-user co-browsing). Steel streams the browser; collaboration UI is still on you.
- You cannot afford an operator to babysit approvals; in that case build deterministic policies instead of half-hearted checkpoints.

## Next steps

- Follow the docs walkthrough on interactive embeds so reviewers can take control safely: [Human-in-the-loop controls](https://docs.steel.dev/overview/sessions-api/human-in-the-loop)
- Wire up live and replay embeds so every approval has matching evidence: [Embed sessions overview](https://docs.steel.dev/overview/sessions-api/embed-sessions)
- When you outgrow single-session prototypes, move the same code to Steel Cloud for managed credentials, files, and concurrency: [Steel Local vs Steel Cloud](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud)

Humans use Chrome. Agents use Steel.
