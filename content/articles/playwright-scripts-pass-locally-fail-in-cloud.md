---
title: "Your Playwright Script Passed Locally. It Still Failed in Production."
id: "playwright-scripts-pass-locally-fail-in-cloud"
summary: "Diagnose why a Playwright script that passes locally fails in Steel Cloud by aligning runtimes, anti-bot posture, and evidence loops."
canonical_questions: ["your playwright script passed locally. it still failed in production."]
intent: "reference"
entity: "integration"
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
canonical_url: "https://steel.dev/blog/playwright-scripts-pass-locally-fail-in-cloud"
description: "Diagnose why a Playwright script that passes locally fails in Steel Cloud by aligning runtimes, anti-bot posture, and evidence loops."
created: "2026-03-31"
modified: "2026-03-31"
tags: [playwright, troubleshooting, automation]
immutable: false
---
The usual culprit is not Playwright. It is the gap between your laptop runtime and the managed fleet you ship to: new GPU flags, stricter [proxies](@/glossary/proxies.md), CAPTCHA gates, or a missing session release. Treat the production run like a different surface area, reproduce it with a Steel session, and the failure gets boring again.

Do three things fast: capture the failing session artifact, align the runtime to the exact Steel session type you use in prod, then walk the checklist below. That loop turns "works on my machine" into a reproducible incident you can fix.

## Fast map of failures

| Symptom in prod | Likely break | Quick check | Steel move |
| --- | --- | --- | --- |
| Script never reaches first `goto` | `chromium.launch` stuck on localhost while prod expects CDP attach | Confirm `chromium.connectOverCDP` URL plus `sessionId` | Create the session via SDK, then attach using the returned ID so both envs share the same browser build |
| Element wait times out even though it rendered locally | Remote region or locale changed DOM timing | Compare Steel session viewer timeline against local HAR | Pin viewport, timezone, and locale when you create the session so Playwright waits for the right signal |
| Login loops forever | Auth state not persisted between retries | Check if the prod run started from a clean profile each time | Use Steel Profiles (`profileId`, `persistProfile`) to reuse cookies, MFA, and extensions |
| Soft-blocked data (HTML loads, values blank) | Anti-bot stack sees datacenter IP or missing input noise | Inspect proxy origin plus pointer events in trace | Run the session with `useProxy` and CAPTCHA solving, drive inputs through native Playwright helpers so traces look human |
| Session dies mid-run with timeout | Session exceeds default 5 minute lifetime or never released | Look at `releasedAt` versus expected duration | Set `timeout` explicitly, release via SDK when done, or bulk release with `sessions.releaseAll()` before a rerun |

## Why local success dies in cloud

**Runtime drift:** Local scripts launch a bundled Chromium. [Steel Cloud](@/glossary/steel-cloud.md) boots a managed browser in under one second and keeps it alive up to 24 hours. Different GPU flags, locales, or touch support will surface cracks immediately unless you pin them when creating sessions.

**Network and region shifts:** Production traffic often routes through managed proxies or a different geography. That changes pricing, cookies, and even which React bundle you receive. Explicitly choose `region` or proxies before the run so DOM contracts stay put.

**Anti-bot scrutiny:** Datacenter IPs, identical pointer patterns, and blank canvas entropy read like bots. Steel's managed sessions already rotate fingerprints, but you still need to emit real scrolls, hovers, and human-paced timing or you will land in soft blocks.

**State resets:** Rerunning from a blank context wipes MFA cookies, device trust, and local storage. [Profiles](@/glossary/profiles.md) store up to 300 MB of user data so you can resume exactly where the previous run stopped.

**Missing evidence:** Local tests let you watch the browser. Remote fleets need traces by default. Steel captures live viewer links plus HLS playback so you can freeze the failing step, copy selectors, and rerun confidently.

## Diagnostic checklist

1. **Freeze the failing session:** Open the Steel session viewer, export the replay, and save console plus network logs before retrying.
2. **Confirm you attached to Steel correctly:** Replace any lingering `chromium.launch()` calls with `chromium.connectOverCDP()` so local repro runs through the same wss endpoint as prod.
3. **Diff session configs:** Compare timeout, proxy, region, and profile arguments from the prod run against your local script parameters.
4. **Check auth state:** If prod runs rely on MFA or remembered devices, ensure the session started with the right profile and that `persistProfile` was true.
5. **Inspect anti-bot posture:** Look for CAPTCHA events, fingerprint mismatches, or proxy bans inside the trace; switch to Steel managed proxies when a site punishes datacenter IPs.
6. **Replay selector timing:** Use the recorded timeline to see whether the locator ever rendered. If not, add state-based waits rather than raw sleeps.
7. **Clean up sessions:** Call `sessions.release()` or `sessions.releaseAll()` so retries start from a clean slate instead of hitting minute-five timeouts.

## Fix sequences that actually stick

### Runtime mismatch

Instead of assuming your laptop Chromium matches prod, spin up a Steel session with the exact config and attach Playwright to it while you debug:

```ts
import { Steel } from "steel-sdk";
import { chromium } from "playwright";

const steel = new Steel({ apiKey: process.env.STEEL_API_KEY });

const session = await steel.sessions.create({
  region: "iad",
  persistProfile: true,
  timeout: 600000,
  useProxy: true,
  solveCaptcha: true,
});

const browser = await chromium.connectOverCDP(
  `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`
);
const page = browser.contexts()[0].pages()[0];
await page.goto("https://portal.example.com");

await browser.close();
await steel.sessions.release(session.id);
```

Now every environment runs the same browser build, [proxy](@/glossary/proxies.md) stack, and timeout floor.

### Anti-bot false positives

Instead of hammering `page.click()` with robotic timing, emit realistic input sequences and reuse session state. Small waits, pointer moves, and steel-managed proxies blunt soft blocks. When CAPTCHAs fire, pass `solveCaptcha: true` so the run waits for the solver rather than failing silently.

### Selector and data drift

Instead of relying on `waitForTimeout`, replay the failure and instrument selectors as contracts. Use `page.waitForSelector()` with domain-specific locators, and store sanitized HTML or Markdown artifacts in your run log. That makes DOM drift obvious before it ships.

### Session hygiene

Instead of letting the default 5 minute lifetime expire, set `timeout` per workflow and release in code when you are done. Bulk release before large reruns so you do not inherit half-dead sessions that still count against concurrency.

## Steel guardrails for Playwright

- Sessions start in under a second and can run for 24 hours, so you can reproduce the exact production runtime locally without waiting.
- Profiles preserve up to 300 MB of user data, which keeps authenticated Playwright flows stable between retries.
- Managed proxies, regional routing, and CAPTCHA solving reduce the anti-bot gaps between your laptop Wi-Fi and the sites you automate.
- The live viewer plus replay exports give you the evidence you need before you edit selectors or timeouts.
- Session lifecycle APIs let you release individual runs or the entire fleet, which prevents orphaned browsers from tripping future jobs.

## Limitations and next steps

Steel cannot fix vendor-side device binding or outages on the target site. If the portal whitelists specific hardware IDs or locks accounts after multiple login attempts, you still need a human checkpoint. Treat this page as the fast path to evidence, not a silver bullet.

Next step: rerun your script while attached to a Steel session with the same proxy and profile settings you expect in production, then watch the replay to trim the root cause. When it looks good, add a release call plus trace capture to your code so the next incident is even faster.

Humans use Chrome. Agents use Steel.
