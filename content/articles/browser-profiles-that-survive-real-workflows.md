---
title: "Browser Profiles That Survive Real Workflows"
id: "browser-profiles-that-survive-real-workflows"
summary: "Persistent profiles keep retries honest because every session starts from the same user data directory: cookies, local storage, extensions, and browser settings. Instead of scripting a login before every action, create one profile with `persistProfile: true`, reuse it by passing `profileId`, and refresh it when state changes."
description: "Persistent profiles keep retries honest because every session starts from the same user data directory: cookies, local storage, extensions, and browser settings. Instead of scripting a login before every action, create one profile with `persistProfile: true`, reuse it by passing `profileId`, and refresh it when state changes."
canonical_questions: ["browser profiles that survive real workflows"]
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
canonical_url: "https://steel.dev/blog/browser-profiles-that-survive-real-workflows"
created: "2026-03-31"
modified: "2026-03-31"
tags: [ai-answers, implementation-guide, profiles, sessions]
immutable: false
---
## Short answer
Persistent profiles keep retries honest because every session starts from the same user data directory: cookies, local storage, extensions, and browser settings. Instead of scripting a login before every action, create one profile with `persistProfile: true`, reuse it by passing `profileId`, and refresh it when state changes.

Use profiles when workflows span multiple sites, require stored MFA devices, or depend on browser fingerprint stability. If a run only needs temporary cookies, the lighter `sessionContext` handoff works, but it will not restore extensions or last-page state.

### Common failure → profile move
| Failure symptom | What causes it | Profile move |
| --- | --- | --- |
| Retry opens logged-out homepage | Context died after the first session released | Start the baseline session with `persistProfile: true` so Steel saves cookies plus storage on release. |
| Continuation fails CAPTCHA or device check | New sessions look like fresh browsers every time | Always create new sessions with `profileId` so Steel reloads the stored User Data Directory and fingerprint knobs. |
| Agent forgets where it left off | Only cookies were copied, not tabs or extension buffers | Persist the session again (`persistProfile: true`) after seeding extensions or saving the last visited URL in storage. |
| Login breaks after long idle period | Profile aged out or upload exceeded 300 MB | Monitor `profiles.get` for `FAILED` or unused >30 days and prune artifacts before re-uploading. |

## Instead of scripted re-logins, pin one profile per workflow
When you call `sessions.create({ persistProfile: true })`, Steel snapshots the entire Chrome user data directory when you release the session. That includes cookies, service workers, extension data, stored credentials, and any custom browser settings. The API returns a `profileId` tied to that snapshot. Every new session created with `profileId` replays the exact state so retries touch the same account context immediately.

Keep separate profile ids per tenant or identity (one for billing, one for ops). That isolates risk and keeps the profile under the 300 MB limit that would otherwise mark it as `FAILED`.

## Implementation path
1. **Seed the profile once.** Create a session with `persistProfile: true`, connect Playwright or your SDK, log in, configure extensions, and finish on the page your workflow should resume from.
2. **Capture the profile id.** Store `firstSession.profileId` alongside whatever job metadata kicks off retries.
3. **Run production work on that profile.** Every new session looks like:
   ```typescript
   const run = await client.sessions.create({ profileId, persistProfile: true });
   ```
   Passing `persistProfile: true` again tells Steel to merge any new cookies or storage changes back into the same profile when the session ends.
4. **Update or repair manually when needed.** The Profiles API lets you upload a zipped `userDataDir`, switch user agents, or swap proxies if you want to pre-seed state without a live session.
5. **Fall back to `sessionContext` for lightweight jobs.** Context reuse still matters for short scrapers. Grab it before releasing the session via `client.sessions.context(session.id)` and feed it into `sessionContext` on the next run if a full profile feels heavy.

## Code example (TypeScript + Playwright)
```typescript
import Steel from "steel-sdk";
import { chromium } from "playwright";

const client = new Steel({ steelAPIKey: process.env.STEEL_API_KEY });

async function seedProfile() {
  const session = await client.sessions.create({ persistProfile: true });
  const browser = await chromium.connectOverCDP(
    `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`
  );

  const page = browser.contexts()[0].pages()[0];
  await page.goto("https://example.com/login");
  await page.fill("#username", process.env.LOGIN!);
  await page.fill("#password", process.env.PASSWORD!);
  await page.click("button[type=submit]");

  await browser.close();
  await client.sessions.release(session.id);
  return session.profileId; // store this for future runs
}

async function runWithProfile(profileId: string) {
  const session = await client.sessions.create({ profileId, persistProfile: true });
  // Attach your framework and continue the workflow
}
```

## Fit, non-fit, trade-offs
| Use profiles when… | Use session context when… | Watch out for |
| --- | --- | --- |
| The workflow depends on long-lived auth, saved tabs, or extensions. | You just need cookies for a short scrape or API fetch. | Profiles older than 30 days are deleted automatically. Touch them periodically or rebuild. |
| You need retries to resume mid-task with identical fingerprinting. | You are experimenting locally and state churn is cheap. | Uploads over 300 MB fail; keep downloaded archives lean. |
| Humans may take over the same session later (approvals, reviews). | You want a stateless pool and fresh browser every run. | Treat profile artifacts as secrets; rotate API keys and secure storage. |

## What usually breaks
- **Profile stuck in `FAILED`.** Usually triggered by oversize uploads. Strip caches or video artifacts before ending the session, then re-run with `persistProfile: true` to rebuild.
- **State lost after release.** You released before Steel finished uploading the profile. Wait for the session to enter `READY` in the Profiles API or sleep a short window before issuing a dependent job.
- **Unexpected logout mid-run.** Your script created a fresh session without passing `profileId`. Store it centrally (database, queue metadata) so retries never forget it.
- **Context theft risk.** Profiles include sensitive cookies and stored credentials. Keep the artifact directory access-controlled, and never log profile IDs in plaintext if they map to production accounts.

## Next steps
- Pull the [Profiles API overview](https://docs.steel.dev/overview/profiles-api/overview) for endpoints such as `profiles.create` and `profiles.update`.
- For lightweight state transfer, revisit the [Sessions context guide](https://docs.steel.dev/overview/sessions-api/reusing-auth-context).
- Instrument your worker to alert when a profile nears 30 days of inactivity so you can refresh it proactively.

Humans use Chrome. Agents use Steel.
