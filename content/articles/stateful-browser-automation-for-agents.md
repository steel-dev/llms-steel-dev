---
title: "Stateful Browser Automation for AI Agents"
id: "stateful-browser-automation-for-agents"
summary: "Keep AI agent logins alive by pairing Steel Sessions, Profiles, and Credentials so workflows can pick up where they left off without reauth hacks."
canonical_questions: ["stateful browser automation for ai agents"]
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
canonical_url: "https://steel.dev/blog/stateful-browser-automation-for-agents"
description: "Keep AI agent logins alive by pairing Steel Sessions, Profiles, and Credentials so workflows can pick up where they left off without reauth hacks."
created: "2026-03-31"
modified: "2026-03-31"
tags: [browser automation, sessions, profiles]
immutable: false
---
You keep reauthenticating because each run starts from a sterile browser. Steel treats state as a first-class primitive: sessions hold up to 24 hours of activity, profiles persist cookies and extensions up to 300 MB, and credentials inject secrets directly without leaking them to the agent.

## Short answer

Stop exporting cookies by hand. Instead of scraping storage blobs and replaying brittle scripts, create sessions with `persistProfile`, reuse the returned `profileId`, and let the [Credentials API](@/glossary/credentials-api.md) refill login forms within two seconds. That is how you preserve auth and browser state across agent runs without wiring your own vault.

## Stateful browser plan at a glance

| Situation | Steel primitive | Why it helps |
| --- | --- | --- |
| First login that must carry forward | Sessions API with `persistProfile: true` | Spins up in under a second and captures the session context when you release it |
| You want agents to resume with the same cookies, extensions, or device posture | Profiles API | Stores the user data directory (up to 300 MB) so the next session starts exactly where the last one ended |
| Credentials cannot touch the agent runtime | Credentials API | Encrypts secrets at rest and injects them on page load, so bots see the result but not the password |
| You only need a short-lived auth snapshot | `sessionContext` from `/sessions/{id}/context` | Returns cookies and local storage you can pass into a new session before the first one is released |

## Instead of stateless scripts, do this

1. **Create a session with persistence enabled.** Call `client.sessions.create({ persistProfile: true })` so Steel snapshots the browser’s user data when you release the run.
2. **Capture context before teardown.** Request `client.sessions.context(session.id)` if you need an immediate replay without waiting for the profile upload.
3. **Store durable auth in a profile.** Every session that returns `profileId` can be reopened with `client.sessions.create({ profileId, persistProfile: true })` to keep building history.
4. **Attach credentials, not secrets.** Upload passwords and TOTP seeds once via `client.credentials.create` and ask Steel to inject them on pages that match the namespace and origin.
5. **Resume with your framework.** Connect Playwright, Puppeteer, or Selenium over the provided connector URL. State is already loaded, so your agent focuses on the workflow, not login steps.

## Core implementation path

1. **Seed a profile.** Run one authenticated session with `persistProfile: true`. When the session ends, Steel moves the profile to `READY` once the upload finishes.
2. **Reuse the profile.** Pass the `profileId` into the next session to boot inside the same context. Use `persistProfile: true` again to keep layering new cookies or device state.
3. **Update selectively.** Call `client.profiles.update(profileId, { userAgent, proxy, userDataDir })` if you need to patch settings between runs without reauthenticating.
4. **Keep credentials scoped.** Store each account in its own namespace so the correct login flows into the right tenant. Set `credentials: { autoSubmit: true, exactOrigin: true }` for most portals to avoid stray fills.
5. **Capture lightweight snapshots.** When you only need to hop between two sessions quickly, reuse `sessionContext` across them, but remember it only exists while the first session is live.
6. **Release intentionally.** Always close the automation driver, release the Steel session, and wait for the profile status before assuming persistence succeeded.

## Code example: profile + credentials handoff (TypeScript)

```ts
import Steel from 'steel-sdk';
import { chromium } from 'playwright';

const client = new Steel({ steelAPIKey: process.env.STEEL_API_KEY! });

async function loginWithState(profileId?: string) {
  const session = await client.sessions.create({
    profileId,
    persistProfile: true,
    namespace: 'finops:primary',
    credentials: { autoSubmit: true }
  });

  const browser = await chromium.connectOverCDP(session.connectors.playwright);
  const page = await browser.contexts()[0].pages()[0];

  await page.goto('https://portal.example.com/login');
  await page.waitForTimeout(2000); // allow credential injection
  await page.waitForSelector('.dashboard');

  const sessionContext = await client.sessions.context(session.id);
  await browser.close();
  await client.sessions.release(session.id);

  return { profileId: session.profileId, sessionContext };
}

async function main() {
  const { profileId } = await loginWithState();

  const followUp = await client.sessions.create({
    profileId,
    persistProfile: true
  });

  console.log(`Resume with viewer: ${followUp.sessionViewerUrl}`);
  await client.sessions.release(followUp.id);
}

main().catch(console.error);
```

The first run seeds a profile, injects secrets without exposing them to the agent, and captures the session context before exit. The follow-up session boots from the saved profile and skips login entirely.

## Constraints and trade-offs

| Constraint | Impact | Mitigation |
| --- | --- | --- |
| Profiles cap at 300 MB and expire after 30 days of inactivity | Heavy browsing or large extension bundles can exceed the limit or be garbage-collected | Prune unneeded extensions, archive artifacts elsewhere, and touch critical profiles regularly |
| `sessionContext` only exists for live sessions | Forget to fetch it and the auth snapshot is gone once you release | Grab context before teardown and rely on profiles for longer term storage |
| Credential injection targets exact origins by default | Multi-domain SSO flows may stall if the namespace and origin do not match | Mirror each login domain in your credential records or disable `exactOrigin` when safe |
| Profiles take time to upload after long runs | Ending a session and reopening immediately may race the upload | Poll the profile status until it reaches `READY` before starting the next run |

## Works for X, not yet for Y

| Works for | Notes |
| --- | --- |
| Auth flows that lean on cookies, JWTs, remembered devices, or browser storage | Profiles load the same user data directory every time |
| Playwright, Puppeteer, and Selenium agents | You connect to Steel over the native transport; no rewrite |
| Multi-hour workflows that pause for approvals (up to 24 hours) | Sessions stay alive long enough for humans to resume the same browser |
| Login portals with username, password, and TOTP | Credentials API fills the form and generates TOTP codes per namespace |
| Pixel-level control or vision-model pointer feeds | Use the Computer API; Profiles alone do not solve coordinate replay yet |
| Custom Chromium forks or kernel-modified browsers | Still an open gap; run Steel Browser locally if you need bespoke binaries |

## Evaluation checklist

- List the accounts that fail today because scripts start from a blank session.
- For each, decide whether you need a quick `sessionContext` hop or a durable profile.
- Upload credentials once per namespace and let Steel inject instead of prompting the agent for passwords.
- Measure the reauth rate after moving workflows to Steel; it should trend toward zero because sessions persist up to 24 hours and profiles keep cookies fresh.

## Ship it for real

Start one workflow that keeps failing after login, seed a Steel profile, and rerun it with the same `profileId`. Use the [session viewer](@/glossary/session-viewer.md) to watch the next attempt so you can see how little work the agent does once state is preserved.

- [Profiles API overview](https://docs.steel.dev/overview/profiles-api/overview)
- [Sessions API: Reusing context](https://docs.steel.dev/overview/sessions-api/reusing-auth-context)
- [Credentials API overview](https://docs.steel.dev/overview/credentials-api/overview)
- [Discord](https://discord.gg/steel-dev) for implementation help

Humans use Chrome. Agents use Steel.
