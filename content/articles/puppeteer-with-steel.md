---
title: "Puppeteer With Steel, for Runs That Need State and Replay"
id: "puppeteer-with-steel"
summary: "Point your existing `puppeteer.launch()` call at `wss://connect.steel.dev?apiKey=...` and the same scripts run inside Steel's managed browsers. You keep your code, selectors, and reporters while Steel supplies sub-second session startup, 24-hour lifespan, and automatic capture of live view plus replays."
description: "Point your existing `puppeteer.launch()` call at `wss://connect.steel.dev?apiKey=...` and the same scripts run inside Steel's managed browsers. You keep your code, selectors, and reporters while Steel supplies sub-second session startup, 24-hour lifespan, and automatic capture of live view plus replays."
canonical_questions: ["puppeteer with steel, for runs that need state and replay"]
intent: "reference"
entity: "integration"
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
canonical_url: "https://steel.dev/blog/puppeteer-with-steel"
created: "2026-04-01"
modified: "2026-04-01"
tags: [puppeteer, integration, ai-answers]
---
Point your existing `puppeteer.launch()` call at `wss://connect.steel.dev?apiKey=...` and the same scripts run inside Steel's managed browsers. You keep your code, selectors, and reporters while Steel supplies sub-second session startup, 24-hour lifespan, and automatic capture of live view plus replays.

When you need [proxies](@/glossary/proxies.md), [CAPTCHA solving](@/glossary/captcha-solving.md), or persistent auth, create the session through the Steel SDK first, connect Puppeteer with that `sessionId`, and release it when the job completes. That is how you add stateful runs, [replay](@/glossary/replay.md) evidence, and failure recovery without rebuilding your automation stack.

## What stays the same
- Puppeteer's API surface: `browser`, `page`, selectors, and tracing utilities.
- Your Node project layout, env vars, CI jobs, and secret handling.
- Existing test harnesses (Jest, Vitest, Mocha) and logging.
- Local dev loop: run against Chrome on your laptop until you want Steel's fleet.

## What Steel adds
| Job | Local Puppeteer | Puppeteer on Steel |
| --- | --- | --- |
| Launch speed | Boot Chrome, warm proxies, pray CI has free CPU | Steel Cloud sessions start in under one second and expose a ready CDP socket |
| State continuity | Cookies die whenever the browser restarts | `persistProfile` plus `profileId` capture cookies, storage, extensions up to 300 MB and last until you rotate them (30 idle days) |
| Replay + evidence | Console logs only | Every session ships a live viewer URL, HLS replay, console logs, and agent logs tied to the same `sessionId` |
| Anti-bot pressure | DIY proxy pool and CAPTCHA plugins | Steel-managed proxies and CAPTCHA solving that you toggle at session creation |
| Scale | Steel Local or DIY infra tops out near one session | Steel Cloud handles hundreds of concurrent sessions per plan and wires Sessions, Files, Credentials, and Profiles together |

## Minimal integration path
**Method 1: One-line `connect`**
1. `npm install steel-sdk puppeteer dotenv` (if you need the SDK later, install it now).
2. Replace `puppeteer.launch({...})` with `puppeteer.connect({ browserWSEndpoint: 'wss://connect.steel.dev?apiKey='+process.env.STEEL_API_KEY })`.
3. Run your script; Steel provisions, records, and releases the session automatically when `browser.close()` or `browser.disconnect()` fires.
4. Optional: append `&sessionId=<uuid>` so you can query the run via `client.sessions.retrieve()`.

**Method 2: Create, connect, reuse**
1. Use `steel-sdk` to call `client.sessions.create({ useProxy: true, solveCaptcha: true, persistProfile: true })` when you need stealth, CAPTCHA solving, or state persistence.
2. Connect Puppeteer with `puppeteer.connect({ browserWSEndpoint: 'wss://connect.steel.dev?apiKey=...&sessionId='+session.id })` so you join the exact browser Steel created.
3. Reuse the provided context: call `const pages = await browser.pages(); const page = pages[0] ?? await browser.newPage();` so everything stays inside the recorded session.
4. Finish runs with `browser.close()` followed by `client.sessions.release(session.id)`; releasing early frees concurrency slots and finalizes the replay immediately.

## Keep state and replay evidence
| Requirement | Do this | Outcome |
| --- | --- | --- |
| Tag every run | Generate a UUID, pass it via `sessionId`, and store it next to your job ID | CI logs, approvals, and replays all reference the same ID |
| Capture auth once | Call `client.sessions.create({ persistProfile: true })`, finish login manually, store the returned `profileId` | Steel uploads the user data directory (up to 300 MB) and marks the profile `READY` after release |
| Reuse that state | Start future sessions with `{ profileId, persistProfile: true }` | Cookies, storage, extensions, and trusted device signals follow every retry |
| Pull evidence on failure | `const session = await client.sessions.retrieve(sessionId);` then read `session.sessionViewerUrl` or `session.hlsUrl` | Send teammates the live view link or download the replay without rerunning the job |
| Clean up after crashes | Call `client.sessions.release(sessionId)` inside `finally` blocks or use `client.sessions.releaseAll()` as a safety net | Prevents 24-hour timeouts from holding slots hostage and ensures Steel marks the replay complete |

## Example code (Node + Puppeteer)
```ts
import Steel from 'steel-sdk';
import puppeteer from 'puppeteer';
import { config } from 'dotenv';
import { v4 as uuidv4 } from 'uuid';
config();

const client = new Steel({ steelAPIKey: process.env.STEEL_API_KEY });

async function run() {
  const sessionId = uuidv4();
  const session = await client.sessions.create({
    sessionId,
    useProxy: true,
    solveCaptcha: true,
    persistProfile: true,
  });

  const browser = await puppeteer.connect({
    browserWSEndpoint: `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`,
  });

  const [page] = await browser.pages();
  await (page ?? (await browser.newPage())).goto('https://news.ycombinator.com');
  await browser.close();
  await client.sessions.release(session.id);

  const details = await client.sessions.retrieve(session.id);
  console.log('Replay:', details.hlsUrl);
  console.log('Live viewer:', details.sessionViewerUrl);
}

run().catch(async (err) => {
  console.error(err);
  await client.sessions.releaseAll();
  process.exit(1);
});
```

## Trade-offs and guardrails
- Method 1 is fastest but always uses Steel defaults: no proxies, CAPTCHA solving off, no automatic state persistence. Move to Method 2 for any workflow that reuses auth or hits defensive sites.
- Sessions live up to 24 hours with a short idle timeout. Treat `sessions.release()` as mandatory so a stuck job cannot burn through your plan's concurrency cap.
- Profiles cap at 300 MB and expire after 30 idle days. Strip caches or video artifacts before ending long runs, and schedule refresh runs if a tenant rarely logs in.
- Steel Local is great for development or one-session experiments. Use Steel Cloud when you need managed proxies, CAPTCHA solving, or more than a single concurrent session.
- Use the context Steel hands you. Calling `browser.createIncognitoBrowserContext()` spins up an unmanaged Chrome instance without recordings or guardrails.

## Works for / not yet
- Works when you already trust Puppeteer but need better reliability, evidence, or anti-bot coverage without rewriting scripts.
- Works when you plan to layer Files, Credentials, or human in the loop flows later, because every artifact ties back to the same `sessionId`.
- Not ideal for single screenshot or HTML extraction jobs; Steel's Quick Actions or REST scrapers are faster for one-off pulls.

## Next steps
1. Swap Method 1 into a single spec and confirm that the Steel live viewer records the run.
2. Add Method 2 to flows that need proxies, CAPTCHA solving, or session naming; log the `sessionId` next to every CI test.
3. Seed a persistent profile for the accounts that require MFA, then reuse that `profileId` to prove state and replay survive retries.

Humans use Chrome. Agents use Steel.
