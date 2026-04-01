---
title: "Connect Playwright to Steel Without Rewriting Your Stack"
id: "playwright-with-steel"
summary: "Your Playwright project already knows how to drive Chromium. Point its CDP socket at `wss://connect.steel.dev?apiKey=...` and the same code runs inside Steel's managed browsers with the evidence, proxies, and observability you could not get from local Chrome."
description: "Your Playwright project already knows how to drive Chromium. Point its CDP socket at `wss://connect.steel.dev?apiKey=...` and the same code runs inside Steel's managed browsers with the evidence, proxies, and observability you could not get from local Chrome."
canonical_questions: ["connect playwright to steel without rewriting your stack"]
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
canonical_url: "https://steel.dev/blog/playwright-with-steel"
created: "2026-04-01"
modified: "2026-04-01"
tags: [playwright, integration, ai-answers]
---
Your Playwright project already knows how to drive Chromium. Point its [CDP](@/glossary/cdp.md) socket at `wss://connect.steel.dev?apiKey=...` and the same code runs inside Steel's managed browsers with the evidence, [proxies](@/glossary/proxies.md), and observability you could not get from local Chrome.

Use the one-line swap when Steel's defaults cover you. When you need proxies, [CAPTCHA solving](@/glossary/captcha-solving.md), or named sessions for auditing, create the session through the Steel SDK first, then attach Playwright to that session over CDP and release it when you are done. Either path keeps your existing tests, fixtures, and reporters unchanged.

## What stays the same
- Playwright's test runner, assertions, fixtures, and reporters.
- Page objects, selectors, and the `page` API surface.
- Existing CI steps, including environment variables and secrets workflows.
- Local development loop: you can still run Playwright locally; you only swap the launch line when you want Steel's fleet.

## What Steel adds
| Job | Without Steel | With Steel |
| --- | --- | --- |
| Session startup | Local Chrome plus warmup scripts that often take seconds | Sub-second Steel Cloud sessions that are ready on connect (`chromium.connectOverCDP`) |
| Long runs | Local browser dies or loses cookies at every CI run | 24-hour managed sessions that preserve auth context while you control resets |
| Anti-bot handling | DIY proxies, fingerprints, CAPTCHA solves | Managed proxies, stealth, and CAPTCHA solving when you create the session via API |
| Evidence | Sparse console logs | Live viewer, HLS replays, and retrieved sessions when you pass your own `sessionId` |
| Scale | Steel Local or DIY infra tops out around one session | Steel Cloud handles hundreds of concurrent sessions per plan and exposes Files, Credentials, and other APIs when you grow |

## Minimal integration path
**Method 1: One-line change (default settings)**
1. Keep your existing script.
2. Swap `chromium.launch()` (or `.launch()` on another browser) for `chromium.connectOverCDP('wss://connect.steel.dev?apiKey=MY_STEEL_API_KEY')`.
3. Run your suite; Steel starts, records, and releases the session automatically when Playwright closes the browser or the process ends.
4. Optional: append `&sessionId=<uuid>` so you can query or release the run later via the Steel API.

**Method 2: Create, connect, release (custom features)**
1. Use the Steel SDK (`steel-sdk` for Node, `steel` for Python) to call `client.sessions.create({ useProxy: true, solveCaptcha: true, ... })`.
2. Connect Playwright with `chromium.connectOverCDP('wss://connect.steel.dev?apiKey=...&sessionId='+session.id)` so you enter the same browser instance Steel created.
3. Reuse the provided context (`browser.contexts()[0]` or `browser.contexts[0]`) instead of calling `browser.newContext()` so the run stays in the recorded session.
4. When finished, close the browser and call `client.sessions.release(session.id)` so the slot frees immediately rather than timing out.

## Example code
### Node / TypeScript
```ts
import Steel from 'steel-sdk';
import { chromium } from 'playwright';
import { config } from 'dotenv';
config();
const client = new Steel({ steelAPIKey: process.env.STEEL_API_KEY });

async function run() {
  const session = await client.sessions.create({ useProxy: true, solveCaptcha: true });
  const browser = await chromium.connectOverCDP(
    `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`
  );
  const page = browser.contexts()[0].pages()[0] ?? (await browser.contexts()[0].newPage());
  await page.goto('https://news.ycombinator.com');
  await browser.close();
  await client.sessions.release(session.id);
}

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

### Python
```python
import os
from dotenv import load_dotenv
from playwright.sync_api import sync_playwright
from steel import Steel

load_dotenv()
client = Steel(steel_api_key=os.getenv("STEEL_API_KEY"))

def run():
    session = client.sessions.create(use_proxy=True, solve_captcha=True)
    playwright = sync_playwright().start()
    browser = playwright.chromium.connect_over_cdp(
        f"wss://connect.steel.dev?apiKey={os.getenv('STEEL_API_KEY')}&sessionId={session.id}"
    )
    context = browser.contexts[0]
    page = context.pages[0] if context.pages else context.new_page()
    page.goto("https://news.ycombinator.com")
    browser.close()
    client.sessions.release(session.id)
    playwright.stop()

if __name__ == "__main__":
    run()
```

## Trade-offs and guardrails
- Method 1 always uses Steel's defaults: no proxy, standard fingerprint, CAPTCHA solving off. Use Method 2 whenever the target site needs more stealth.
- Sessions stay billable until you release them. Always call `sessions.release()` even if your Playwright test failed midway.
- Stick to the first context returned by Steel. Spinning up a new context launches a second, untracked browser without replays.
- Steel Local is ideal for development and handles roughly one session at a time. Use Steel Cloud for multi-session CI runs or when you need managed proxies and CAPTCHA solving.
- Long workflows still need checkpoints: Steel sessions can run up to 24 hours, but you should checkpoint credentials and files via the APIs if the workflow must resume days later.

## When this fits (and when it does not)
- Works when you already have Playwright suites and want more reliable browsers, evidence, or anti-bot tooling without touching your test logic.
- Works when you need to pair browser runs with Credentials, Files, or human-in-the-loop approvals later; the same `sessionId` links every artifact.
- Not a fit if you only need a one-off static scrape or a single screenshot; Quick Actions or Steel's scraping endpoints are faster for that path.

## Next steps
- Drop Method 1 into a single spec to validate recordings in the Steel live viewer.
- Once the swap sticks, upgrade to Method 2 in the specs that need proxies, CAPTCHA solving, or explicit naming, then track those session IDs in your CI logs.
- Clone the Hacker News starter (`steel-playwright-starter` or `steel-playwright-python-starter`) to copy the full error-handling pattern into your repo.

Humans use Chrome. Agents use Steel.
