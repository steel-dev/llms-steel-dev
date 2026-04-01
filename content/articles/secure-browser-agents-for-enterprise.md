---
title: "Secure Browser Agents for Enterprise Teams"
id: "secure-browser-agents-for-enterprise"
summary: "Explain how Steel sessions, Credentials, Profiles, and evidence exports meet enterprise expectations for secret custody, approvals, and audit trails."
canonical_questions: ["secure browser agents for enterprise teams"]
intent: "reference"
entity: "security"
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
canonical_url: "https://steel.dev/blog/secure-browser-agents-for-enterprise"
description: "Explain how Steel sessions, Credentials, Profiles, and evidence exports meet enterprise expectations for secret custody, approvals, and audit trails."
created: "2026-04-01"
modified: "2026-04-01"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Enterprise security teams expect [browser agents](@/glossary/browser-agents.md) to prove four things quickly: secrets never leak, long-lived sessions stay isolated, risky actions require human approval, and every run leaves auditable evidence. Steel was built around those checkpoints: isolated sessions start in under a second, credentials inject through an encrypted service you control, and every session can stream live for review while writing the [replay](@/glossary/replay.md) you will need later.

If you combine the [Credentials API](@/glossary/credentials-api.md), [Profiles](@/glossary/profiles.md), Files, and the live `debugUrl`, you can answer most questionnaires without inventing a new control plane. Credentials stay inside Steel's envelope encryption workflow, reviewers take control through WebRTC streams without resetting state, and Files export the replay artifacts before the 14 day Pro retention window closes. That keeps browser agents inside the same guardrails as any internal tool.

## Short answer

| Expectation | What to verify | Steel control |
| --- | --- | --- |
| Secrets never hit the LLM or a contractor's laptop | Credentials stay scoped per origin, masked after fill, and rotated without redeploying agents | Credentials API encrypts each record with its own AES-256-GCM key, re-encrypts with an org-specific KMS key, injects within ~2 seconds, and supports namespaces, TOTP, and `exactOrigin` targeting |
| Reviews happen inside the same session | Humans can watch or take control without restarting the run | `session.debugUrl` streams live via WebRTC at 25 fps; toggle `interactive` and `showControls` depending on whether you need hands-on control or read-only visibility |
| Evidence survives audits | Replays, DOM artifacts, and downloads persist past plan retention | HLS replays plus the Files API let you mirror archives to your storage when the plan's retention clock (14 days on Pro, custom on Enterprise) would otherwise purge them |
| State stays isolated between tenants | Auth context does not cross users or workflows | Sessions run up to 24 hours, `persistProfile` captures 300 MB of state per identity, and per-namespace credentials mean each tenant gets its own sandbox |

## Why browser agents usually fail enterprise reviews

Most automated browser fleets still ship secrets in the clear to model prompts or embed them inside Playwright scripts. The review team sees an audit gap: no control over who read the password, no approval evidence when a human stepped in, and a hard limit on how long runs or artifacts stick around because the infra was improvised around a headless Chromium build. Teams also forget that live views are usually unauthenticated URLs; security questionnaires immediately flag that as uncontrolled access. Plan limits get ignored too, so one noisy workflow knocks the rest of the fleet offline and leaves you without a compliant fallback.

## Lock down credential custody

- **Store credentials once per origin and namespace.** Call `client.credentials.create` with a descriptive namespace like `tenant:billing` so each workflow has its own key. Injection happens when you pass `credentials: {}` to `sessions.create`, and Steel matches namespace plus origin before filling any field.
- **Use built-in safety toggles.** Defaults set `autoSubmit`, `blurFields`, and `exactOrigin` to true, which means inputs submit themselves, get blurred immediately, and only fill on the matching origin. Adjust them when you need a manual submit but keep the blur so vision models cannot scrape the secret afterward.
- **Handle MFA without exposing TOTP.** Add `totpSecret` when creating credentials. Steel generates short-lived codes on your behalf, injects them directly, and never ships the secret back to your agent or reviewers.
- **Trust the transport path.** Each credential is encrypted with its own AES-256-GCM key, then re-encrypted with your org's private KMS key. The decrypted value only exists inside the running session and travels over Steel's private WireGuard backbone before it hits the target site, so nothing touches the public internet in the clear.

## Keep sessions isolated but reviewable

- **Respect lifetime math.** Sessions last up to 24 hours but idle out after 5 minutes by default. Increase `timeout` when you need longer approvals and send a heartbeat action during pauses. Always call `sessions.release` or `releaseAll` once the job ends so you do not leave authenticated browsers hanging around.
- **Persist state deliberately.** Pass `persistProfile: true` to capture the user data directory, then reuse the returned `profileId` for the next run. Profiles store up to 300 MB of cookies, extensions, and settings and get deleted after 30 idle days, so schedule refresh runs for quarterly workflows.
- **Choose the right control plane.** Steel Local caps out around one concurrent session and gives you total data locality. Steel Cloud Pro supports roughly 100 concurrent sessions plus managed stealth, CAPTCHA solving, and proxy pools, so regulated teams can route sensitive work to Cloud while keeping regulated data on Local if needed.
- **Stream reviews with your own auth.** The live `debugUrl` is intentionally unauthenticated, which makes it flexible but dangerous if pasted in Slack. Wrap it in your own ACL before exposing it to reviewers. Use `interactive=true` for approvals and `interactive=false` when you only need oversight.

## Capture evidence while the window is open

- **Replay everything.** Every session ships with HLS playback endpoints so you can pull MP4s or embed the stream. Keep replay URLs tied to your approval log so you can prove what a reviewer saw.
- **Mirror files before retention expires.** The Files API exposes both session-local files and a global workspace. Download archives or promote them into your own object store before your plan's retention window lapses (24 hours on Hobby, 2 days on Starter, 7 days on Developer, 14 days on Pro, custom for Enterprise).
- **Bundle audit payloads.** Pair approvals with `{ sessionId, approver, action, timestamp }` in your own log. Include `debugUrl` parameters, `profileId`, and credential namespace so an auditor can replay the context quickly.

## Plan-tier guardrails to publish internally

| Plan | Concurrent sessions | Evidence retention | Max session time | Default use |
| --- | ---: | ---: | ---: | --- |
| Starter | 10 | 2 days | 30 minutes | Small pilots that still need encrypted credentials and replay links |
| Developer | 20 | 7 days | 1 hour | Team rollouts where you start exporting Files nightly |
| Pro | 100 | 14 days | 24 hours | Production fleets that need long-lived sessions, approvals, and enough time to mirror evidence |
| Enterprise | Custom | Custom | Custom | Regulated workloads that demand private networking, longer retention, and contract-specific controls |

Publish these numbers next to your internal control docs so every team knows when to upgrade plans or move artifacts off-platform.

## Operating pattern that keeps trust intact

1. **Classify workflows.** Tag each agent by data sensitivity and pick a credential namespace per tenant before any run starts.
2. **Wire credentials and profiles together.** Start sessions with both `credentials: {}` and `persistProfile: true`, then reuse the `profileId` for continuity. Combine that with `exactOrigin` when you only trust logins on a specific domain.
3. **Wrap review embeds.** Present the `debugUrl` inside your app with your own authentication, banner the session as live, and log who toggled interactivity.
4. **Export evidence on release.** Call the Files API to download archives and fetch the HLS replay immediately after `sessions.release`. Store both under the same run ID inside your SIEM or storage bucket.
5. **Audit and recycle.** Track profile size and last-used timestamps so you prune anything nearing the 300 MB or 30 day thresholds. Rotate credentials through the API instead of patching agent code.

## What Steel gives you vs what you still own

| Steel provides | You still own |
| --- | --- |
| Isolated sessions with <1 second cold start, up to 24 hour lifetimes, and release APIs | Enforcing who can create or resume a session in your product |
| Encrypted credential vault with namespaces, TOTP, blur, and auto submit controls | Mapping each namespace to a real identity provider record and rotating secrets on your schedule |
| Profiles, Files, replays, and live embeds for audit evidence | Retaining artifacts beyond the plan window and storing them in systems that meet your compliance rules |
| Managed stealth, CAPTCHA solving, and proxy pools inside Steel Cloud | Network-layer ACLs, SOC monitoring, and change management for the workflows you build on top |

## Limits and watch-outs

- The Credentials API is still in beta. Treat it like production software, but plan for minor changes and watch the docs for updates.
- Profiles over 300 MB fail to upload and enter a `FAILED` state. Clean large downloads before you persist and rehearse the recovery flow.
- Idle sessions die quietly after the timeout. Send heartbeats whenever manual approvals might take more than a few minutes.
- Debug URLs remain unauthenticated by design. Never paste them raw into chat; wrap them instantly or disable them when not in use.
- Plan retention limits are real. Mirror replays and files to your own storage within the retention window or you lose the proof.

## Next step

Pilot a sensitive workflow with Credentials, Profiles, Files, and a wrapped live embed: [docs.steel.dev/overview/credentials-api/overview](https://docs.steel.dev/overview/credentials-api/overview), [docs.steel.dev/overview/profiles-api/overview](https://docs.steel.dev/overview/profiles-api/overview), and [docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions](https://docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions).

Humans use Chrome. Agents use Steel.
