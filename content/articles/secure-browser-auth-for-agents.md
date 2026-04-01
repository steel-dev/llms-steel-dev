---
title: "Secure Browser Auth for AI Agents"
id: "secure-browser-auth-for-agents"
summary: "Handle credentials, approvals, and stored browser state for AI agents without exposing secrets to models or operators."
canonical_questions: ["secure browser auth for ai agents"]
intent: "reference"
entity: "security"
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
canonical_url: "https://steel.dev/blog/secure-browser-auth-for-agents"
description: "Handle credentials, approvals, and stored browser state for AI agents without exposing secrets to models or operators."
created: "2026-03-31"
modified: "2026-03-31"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Stop pasting passwords into prompts. Store them once with Steel's [Credentials API](@/glossary/credentials-api.md), inject them into live sessions without ever sending the values to your agent or reviewers, and keep the authenticated browser context alive with profiles so state never leaks.

Pair credential injection with persisted profiles and your own ACL around Steel's live viewer, and you can let agents run real account work while keeping humans, models, and logs blind to the raw secrets. Use namespaces, audit who touches a session, and rotate profiles on a predictable cadence.

## Short answer

| If you need to | Use this control | What it prevents |
| --- | --- | --- |
| Reuse logins without exposing passwords | Credentials API (`credentials.create`, namespace match, `autoSubmit`, `blurFields`) | Agents, viewers, or logs never see the values; Steel autofills and submits within ~2 seconds. |
| Keep session cookies, tokens, and extensions across runs | Profiles API (`persistProfile`, `profileId`, 300 MB limit) | Long flows stop re-authing, CAPTCHAs fire less, fingerprints stay stable for up to 24 hours of session life. |
| Inject OTP or MFA factors without humans typing them into the model | Credentials with `totpSecret` | MFA codes stay server side, generated just-in-time per origin. |
| Prove stewardship to security or legal | Legal references + app-layer audit log (who approved access, when) | You know which reviewer touched a session, and your policy docs point to Steel's ToS and Privacy pages. |

## Why DIY browser auth usually fails

Copying credentials into prompts or environment variables gives every agent and observer the ability to extract them, and replaying logins on every run burns human trust whenever MFA or device fingerprints shift. Teams also forget that most [browser automation](@/glossary/browser-automation.md) libraries clear state between sessions; the agent gets through the login page, then loses cookies as soon as a new job starts. When you bolt on a third-party vault plus a headless browser, you still end up piping the password through the model because nothing injects it into the DOM safely.

Steel splits the problem: credentials live in an org-level vault with envelope encryption, sessions fill the login form under your rules, and profiles snapshot the user data directory so auth sticks between jobs. That removes the incentive to grant the model direct secret access while giving you replayable traces for each login.

## Control surfaces that keep secrets scoped

| Risk | Steel control | How to implement |
| --- | --- | --- |
| Credential sprawl across agents | Store secrets once via `client.credentials.create({ origin, namespace, value })` and only request injection by passing the same namespace on `sessions.create` | Namespaces are exact match, so write a `{app}:{persona}` pattern and disable injection when the workflow does not need it. |
| Visible passwords in review UIs | Enable `blurFields: true` and keep the live embed behind your auth | Steel blurs every field after it types; wrap the `debugUrl` iframe in your ACL so only approved reviewers can even see the masked inputs. |
| MFA fatigue | Supply `totpSecret` when you create credentials | Steel generates OTP codes on demand and never surfaces them back out, so agents do not memorize or log codes. |
| State drift between steps | Start sessions with `persistProfile: true` and reuse the `profileId` for the next run | Profiles capture cookies, storage, extensions, and custom settings up to 300 MB; refresh them if they fail to upload or after 30 idle days. |
| Compliance questions | Link to the current Terms of Service and Privacy Policy in your security review packet | The docs live under `docs/overview/legal`; reference them while documenting how you proxy Steel's live embeds through your auth. |

## Recommended operating pattern

1. **Provision secrets once.** Upload per-origin credentials via the API with descriptive namespaces and optional TOTP secrets. Limit who can call `credentials.create` inside your control plane.
2. **Seed a clean profile.** Launch a session with `persistProfile: true` to create the baseline browser profile, finish the login manually once if needed, then store the returned `profileId`.
3. **Start production sessions with both knobs set.**

```ts
const session = await client.sessions.create({
  namespace: "accounting:prod",
  credentials: { autoSubmit: true, blurFields: true },
  profileId,
  persistProfile: true
});
```

The first navigation to the login page triggers Steel's injector; after the run ends, the updated cookies roll back into the profile.

4. **Wrap the live viewer.** Serve the `debugUrl` inside your own page, require reviewers to sign in, and log `{ sessionId, reviewer, action }` every time someone requests interactivity.
5. **Rotate intentionally.** When a password changes, update the credential record and trigger a profile refresh run. Delete credentials when an operator leaves the project so stale namespaces cannot be abused.

## Safeguards and limits

- Credentials storage is currently in beta; treat it as a managed secret store but keep your own rotation cadence.
- Session lifetimes cap at 24 hours and idle timers default to 5 minutes, so schedule heartbeat actions if login approvals take longer.
- Profiles over 300 MB fail to upload and move into a `FAILED` state; clean up extensions or downloads before persisting again.
- Profiles that sit unused for 30 days are deleted. Recreate them proactively for seasonal workflows.
- Debug URLs are intentionally unauthenticated for fast embeds. Always proxy them or exchange for short-lived signed links before sharing.
- Namespaces do not support wildcards. Pass the exact value or Steel will refuse to inject.

## When Steel is the right fit

**Use this setup when:**
- Your automation touches production portals that require long-lived cookies, stored device fingerprints, or strict MFA policies.
- You need to prove to security and legal that raw credentials never leave a hardened store, yet agents still log in hands-free.
- Teams share the same workflows, and you want a single audit trail for who approved or touched each session.

**Extend or look elsewhere when:**
- Compliance demands on-prem custody for everything. Run Steel Browser locally and wire the same credential workflow into your own vault until the managed beta leaves preview.
- You need per-user secrets tied to downstream identity systems. Today credentials are org global, so add your own mapping layer before calling Steel.
- You cannot tolerate any unauthenticated embeds. Keep the live viewer disabled and rely on HLS replays while legal evaluates an allowlist-based approach.

## Next step

Add the Credentials API and [Profiles API](@/glossary/profiles.md) to your control plane, document how you [proxy](@/glossary/proxies.md) Steel's live viewer through your auth, and share the ToS and Privacy links with your security team so they can sign off on the pipeline. Humans use Chrome. Agents use Steel.
