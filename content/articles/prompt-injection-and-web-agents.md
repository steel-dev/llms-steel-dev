---
title: "Prompt Injection Risks in Web Agents"
id: "prompt-injection-and-web-agents"
summary: "Show how Steel's sessions, credential injection, approvals, and evidence capture limit prompt injections without killing browser agent state or audits."
canonical_questions: ["prompt injection risks in web agents"]
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
canonical_url: "https://steel.dev/blog/prompt-injection-and-web-agents"
description: "Show how Steel's sessions, credential injection, approvals, and evidence capture limit prompt injections without killing browser agent state or audits."
created: "2026-04-01"
modified: "2026-04-01"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Prompt injection is inevitable any time an agent reads untrusted DOM. Attackers hide instructions inside support portals, in-app chat widgets, or seeded downloads, then wait for your model to repeat a secret or click a destructive button. Steel cannot stop a model from reading poisoned copy, but it does give you control surfaces that keep credentials out of the prompt, gate risky actions, and capture every frame for audit.

Treat prompt injection like an ops problem, not a novelty. Run sessions that behave like isolated incognito windows, inject credentials through a server-side vault, force humans to approve anything outside the runbook, and export evidence before retention windows expire. Those steps will not make the attack disappear. They do make each attempt observable, reversible, and far less profitable.

## Short answer

| If the injection step tries to | Steel control | Why it works |
| --- | --- | --- |
| Trick the agent into revealing passwords | Credentials API with namespaces, `autoSubmit`, `blurFields`, `exactOrigin`, optional `totpSecret` | Secrets never enter the prompt or live viewer; Steel decrypts per request using AES-256-GCM plus an org KMS key and fills the login form within ~2 seconds inside the session. |
| Reuse the hijacked session on another tenant | Sessions API + Profiles API (`persistProfile`, `profileId`, release discipline) | Each session is its own incognito browser capped at 24 hours with a 5 minute idle timeout. Profiles store up to 300 MB of context that you can rotate per tenant so injected instructions cannot jump sandboxes. |
| Push a destructive action without oversight | Wrapped `debugUrl` with `interactive=true` plus approval logging | The live WebRTC stream is intentionally unauthenticated, so you wrap it in your ACL, turn on interactivity only after recording `{sessionId, approver, action}`, and keep the same session paused until you resume automation. |
| Hide what happened | Files API, HLS replays, plan-tier retention table | Steel backs up session files and exposes replay endpoints so you can export MP4, downloads, and logs before your plan's retention (24 h to 14 days) expires. Attackers cannot erase that once you mirror it into your storage. |

## Why prompt injection hits browser agents harder

Browser agents read everything: marketing popovers, admin banners, random HTML comments. That gives attackers a large surface to slip payloads into the agent's short term memory. Simon Willison's "lethal trifecta" is a good mental model: give the agent instructions, hand it tools, then harvest the secrets it exposes. If you pipe raw credentials through a prompt, or let reviewers share the debug URL without access control, a single injected div can walk off with anything the session has permission to touch.

Unlike API automation, [browser agents](@/glossary/browser-agents.md) also inherit the user's identity. Long lived cookies, stored extensions, and live approvals make a compromised session far more valuable. That is why Steel leans on supervised autonomy: isolate identities, log every action, and make escalation obvious to the humans who operate the workflow. Prompt injection becomes less about clever prompt wording and more about whether your infrastructure left any high value target exposed.

## Control surfaces that actually move the risk

### 1. Keep secrets out of the token stream

Upload credentials once via `client.credentials.create({ origin, namespace, value, totpSecret })`, then request injection by passing `credentials: {}` on `sessions.create`. Steel stores each record with its own AES-256-GCM key, re-encrypts it with your org's private KMS key, and only decrypts inside the running session over Steel's private WireGuard backbone. Defaults enable `autoSubmit`, `blurFields`, and `exactOrigin`, which means credentials fill and submit quickly, fields blur before a vision model can read them, and injections only happen on the expected domain. Even if an injected prompt tries to coax the agent into echoing the password, there is nothing in memory to leak.

Pair credential injection with clear namespaces such as `{tenant}:{workflow}`. That pattern fences each login to the right persona and gives you a switch to disable injection when a job does not require secrets. Add `totpSecret` whenever a site enforces MFA. Steel generates codes just in time and never returns them, so an injected prompt cannot hoard the shared secret.

### 2. Constrain what an injection can touch after login

Steel sessions behave like headful incognito windows. They start in under a second, live up to 24 hours, and time out after 5 minutes of inactivity unless you send heartbeats. Always release sessions explicitly (`sessions.release` or `releaseAll`) once the job ends so you do not leave authenticated browsers around for attackers to target. When you need continuity, set `persistProfile: true` and reuse the returned `profileId`. [Profiles](@/glossary/profiles.md) hold up to 300 MB of cookies, extensions, and storage and expire after 30 idle days, which lets you rotate them per tenant or per workload.

Publish plan-tier guardrails so every team knows when to scale:

| Plan | Concurrent sessions | Retention window | Max session length |
| --- | ---: | ---: | ---: |
| Starter | 10 | 2 days | 30 minutes |
| Developer | 20 | 7 days | 1 hour |
| Pro | 100 | 14 days | 24 hours |
| Enterprise | Custom | Custom | Custom |

Treat those numbers like part of your threat model. [Steel Local](@/glossary/steel-local.md) is roughly one live session and keeps everything inside your VPC. [Steel Cloud](@/glossary/steel-cloud.md) handles the fleets that need managed stealth, proxies, and large credential stores. Split sensitive workflows accordingly so a compromised Local session cannot see Cloud secrets, and vice versa.

### 3. Put humans between injected instructions and production damage

Every session ships with a `debugUrl` that streams the live browser over WebRTC at 25 fps. The URL is unauthenticated by design, so front it with your own auth layer before anyone even previews it. When a workflow hits a risky action, pause the automation and hand reviewers an iframe pointing to `${debugUrl}?interactive=true&showControls=true`. Log the reviewer, the action they approved, and the timestamp before you flip interactivity on. If the reviewer rejects the step, call `sessions.release` so the attacker cannot reuse the authenticated state.

For read-only monitoring, pass `interactive=false`. That keeps observers from fat-fingering a click while still letting them watch for suspicious DOM instructions. Either way, keep idle timers in mind: send a lightweight action every few minutes if approvals take longer than the default five minute window.

### 4. Capture artifacts faster than attackers can cover their tracks

Combine the Files API and replay endpoints with your own storage so you always have an evidence trail. Every session lets you download single files or a full archive via `sessions.files.downloadArchive(sessionId)`. That archive moves from the session VM into global storage automatically when you release the session, but plan retention limits still apply. Mirror everything into your SIEM or object store before the clock runs out. Pull the HLS replay (`/v1/sessions/{id}/hls`) or MP4 and attach it to your incident record. Tie those artifacts to approval logs so you can prove whether an instruction came from a poisoned page or from your own agent logic.

Agent logs matter too. Fetch `GET /v1/sessions/{id}/agent-logs` when something looks suspicious and match DOM events to prompt history. Prompt injection loses most of its sting when you can replay the entire sequence verbatim.

## Operating pattern for a sane trust posture

1. **Classify workflows and secrets.** Decide which namespaces, profiles, and plan tiers each workflow needs before the agent runs anything.
2. **Seed profiles under supervision.** Create the baseline session with `persistProfile: true`, finish the login manually once, then record the `profileId` and the credential namespace together.
3. **Wrap every live view.** Serve the `debugUrl` through your own app, require sign-in, and log approvals before enabling interactivity.
4. **Export artifacts on release.** Download files, agent logs, and the replay immediately after `sessions.release` so retention windows never surprise you.
5. **Red-team your own agents.** Inject fake instructions into staging portals and confirm that credentials stay masked, approvals fire, and alerts trigger before anything leaks.

## What Steel gives you vs what you still own

| Steel handles | You still owe |
| --- | --- |
| Isolated sessions with defined lifetimes, WebRTC viewers, and release APIs | Enforcing who can create, pause, or resume a session inside your product |
| Managed credential storage, injection toggles, namespaces, and TOTP filling | Mapping namespaces to real tenants, rotating secrets, and revoking access when people leave |
| Profiles, Files, replays, and agent logs so you can inspect every run | Classifying data, mirroring evidence to compliant storage, and reviewing logs for anomalies |
| Plan-tier concurrency, proxy pools, stealth, and CAPTCHA solving | Scaling orchestration so one noisy workflow does not starve the rest of your fleet |

Prompt injection defenses fail when teams assume Steel can absolve them of LLM hygiene. You still need model-level guardrails, schema validation, and allowlisted tool use. Steel gives you the infrastructure hooks to make those policies stick.

## Safeguards and limits

- The Credentials API is still in beta. Treat it like production software but watch the changelog for updated behaviors and keep your own rotation cadence.
- Profiles over 300 MB fail to upload and move into a `FAILED` state. Trim downloads and extensions before persisting again, and recreate profiles every 30 idle days.
- Debug URLs remain unauthenticated. Never paste them directly into chat or tickets; proxy them immediately or disable them entirely for the most sensitive workflows.
- Idle sessions shut down quietly. If a review or holding pattern will take longer than five minutes, send periodic `page.waitForTimeout(0)` heartbeats or increase the `timeout` parameter when you create the session.
- Plan retention windows are real. Hobby plan logs disappear after 24 hours while Pro gives you 14 days. Mirror anything important to your own storage.
- Steel Local has roughly one concurrent headful session. If you need dozens of simultaneous approvals or red-team runs, move them to Steel Cloud so you do not lose coverage while Local idles.

## Next step

Pilot a high-risk workflow end to end: upload credentials with namespaces, seed a profile, run the session through a wrapped live embed, and export the replay plus Files archive before you release it. Keep the runbook handy for the next red-team drill. Humans use Chrome. Agents use Steel.
