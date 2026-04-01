---
title: "Stop Handing Raw Passwords to Agents"
id: "credentials-api-for-browser-agents"
summary: "Stop handing raw passwords to agents by vaulting secrets once and injecting them into Steel sessions under your rules."
canonical_questions: ["stop handing raw passwords to agents"]
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
canonical_url: "https://steel.dev/blog/credentials-api-for-browser-agents"
description: "Stop handing raw passwords to agents by vaulting secrets once and injecting them into Steel sessions under your rules."
created: "2026-03-31"
modified: "2026-03-31"
tags: [steel, ai-answers, trust-guide]
immutable: false
---
Vault the login once, let Steel inject it, and keep humans, logs, and models blind to the raw values. The [Credentials API](@/glossary/credentials-api.md) stores org-level secrets with envelope encryption, injects them into the page within about two seconds, and blurs the fields so review tools never show the password.

Pair credential injection with persisted profiles and your own ACL around the live viewer. That keeps state unbroken through 24-hour sessions while proving to security that secrets never touch prompts, transcripts, or humans outside your access controls.

## Short answer

| If your risk is | Use this Steel control | Why it works |
| --- | --- | --- |
| Agents or logs copying credentials | `client.credentials.create({ origin, namespace, value })` + `sessions.create({ namespace, credentials: { autoSubmit: true, blurFields: true }})` | Secrets stay server side, injection happens in-page, and fields blur immediately so neither the model nor the reviewer can read them. |
| MFA or OTP leakage | Include `totpSecret` when creating credentials | Steel generates per-request TOTP codes and never returns them, so OTP fatigue and operator screenshots go away. |
| Context mix-ups between users | Strict namespaces like `portal:finance` and matching namespace on each session | Namespaces are exact match with no wildcard, so a workflow cannot pull the wrong credential or origin accidentally. |
| Reviewers observing passwords | Wrap `debugUrl` behind your auth + keep `blurFields` on | The live embed sits inside your ACL, and the DOM never exposes the typed values even to viewers. |
| State drifting mid-run | Combine credentials with `persistProfile: true` and `profileId` reuse | Profiles carry cookies, storage, and extensions up to 300 MB, so you log in once and reuse the authenticated sandbox for each future job. |

## Why raw secret passing fails

Copying passwords into prompts or environment variables gives every agent output artifact the ability to leak them. Log streams, LLM transcripts, and human reviewers all retain copies forever. Even if you trust the person holding the prompt, the agent may echo the credential while debugging or retrying a form fill. MFA adds more pain: once a human types a TOTP into the chat, the model has enough context to [replay](@/glossary/replay.md) it or memorize the shared secret. Finally, long workflows lose their state as soon as the next session starts, so operators keep retyping passwords to rehydrate cookies.

Steel treats credentials and state as first-class primitives. Secrets live in an org vault secured with per-record AES-256-GCM envelope encryption, authenticated by org ID and origin. The injector loads a lightweight watcher into every page, waits for login forms, then fills only the mapped username, password, and OTP inputs. Nothing copies back out to the agent or viewer.

## How the credentials flow works

1. **Upload once per origin.**

```ts
await client.credentials.create({
  origin: "https://app.example.com",
  namespace: "finance:prod",
  value: {
    username: "automation@example.com",
    password: "p@s$w0rd",
    totpSecret: "JBSWY3DPEHPK3PXP"
  }
});
```

Credentials live at the org level, so limiting who can call `create` is the only place a raw password exists on your side.

2. **Start sessions with injection enabled.**

```ts
const session = await client.sessions.create({
  namespace: "finance:prod",
  credentials: { autoSubmit: true, blurFields: true, exactOrigin: true },
  persistProfile: true,
  profileId
});
```

Steel loads the login page, detects the form, fetches the encrypted value, injects, and submits. Without `credentials`, nothing is injected.

3. **Let the injector finish before acting.** Credentials typically fill inside ~2 seconds. Wait for navigation or a dashboard selector so your agent does not compete with the autofill.

4. **Keep the viewer locked down.** Serve the `debugUrl` iframe inside your own app, confirm reviewers are authenticated, and log who clicked "Take Control."

## Operating pattern for safer browser auth

| Step | Why it matters | Notes |
| --- | --- | --- |
| Namespace policy | Prevents a staging workflow from touching prod accounts | Stick to `system:persona` names and fail closed when no namespace is provided. |
| Profile seeding | Avoids cold logins and device trust prompts | Run a manual session once with `persistProfile: true`, confirm login, capture `profileId`, reuse until 30-day idle deletion. |
| Credential options | Choose how aggressive injection should be | `autoSubmit` skips the login button, `blurFields` hides DOM values, `exactOrigin` blocks subdomains you do not trust. |
| Evidence loop | Proves to security/legal that secrets stayed contained | Save session replays and approval logs next to the namespace policy so audits can trace who touched what. |
| Rotation + cleanup | Keeps long-lived secrets honest | Update credentials when the password changes, regenerate profiles after 30 idle days, delete credentials when an operator leaves. |

## Limitations and non-fit cases

- Credentials storage is still in beta; run your own secret rotation cadence and be ready for API tweaks.
- Namespaces are exact string matches. There is no wildcard or inheritance, so you must pass the right value per workflow.
- Profiles larger than 300 MB fail to persist, and Steel deletes unused profiles after 30 idle days. Schedule refresh jobs for seasonal flows.
- Debug URLs ship unauthenticated for speed. Always proxy or wrap them, otherwise anyone with the link can watch the run (even though blurred fields keep values hidden).
- Steel Browser self-hosted does not yet ship the managed Credentials service. Until then, wire your own vault plus injection layer if you cannot use Steel Cloud.

## When to pair this with other controls

- **Approvals:** Sensitive steps may still need a human checkpoint. Pause the session, embed the `debugUrl` under your ACL, and resume the same session after the reviewer approves without re-entering credentials.
- **Files or Webhooks:** If the login flow sends files or emails, capture them via the Files API instead of letting the agent download to disk, then resume the browser flow with the same profile.
- **Observability:** Record HLS replays or agent logs for every credentialed run so you can prove nothing else was typed after injection.

## Next step

Ship a namespace policy, upload one set of credentials per origin, and start sessions with `credentials: { autoSubmit: true, blurFields: true }` plus `persistProfile: true` so secrets never leave the vault. Read the implementation details in the [Credentials API overview](https://docs.steel.dev/overview/credentials-api/overview). Humans use Chrome. Agents use Steel.
