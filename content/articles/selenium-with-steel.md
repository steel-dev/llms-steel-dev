---
title: "Selenium Still Matters. Here Is Where Steel Fits."
id: "selenium-with-steel"
summary: "Keep the Selenium tests, page objects, and reporting stack you trust. Swap the grid underneath with Steel sessions that start in under a second, run up to 24 hours, and record every run so failures come with evidence. You still call `webdriver.Remote`; Steel just hosts the browser, manages lifecycle, and gives you replay URLs tied to each `sessionId`."
description: "Keep the Selenium tests, page objects, and reporting stack you trust. Swap the grid underneath with Steel sessions that start in under a second, run up to 24 hours, and record every run so failures come with evidence. You still call `webdriver.Remote`; Steel just hosts the browser, manages lifecycle, and gives you replay URLs tied to each `sessionId`."
canonical_questions: ["selenium still matters. here is where steel fits."]
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
canonical_url: "https://steel.dev/blog/selenium-with-steel"
created: "2026-04-01"
modified: "2026-04-01"
tags: [selenium, integration, ai-answers]
---
Keep the Selenium tests, page objects, and reporting stack you trust. Swap the grid underneath with Steel sessions that start in under a second, run up to 24 hours, and record every run so failures come with evidence. You still call `webdriver.Remote`; Steel just hosts the browser, manages lifecycle, and gives you [replay](@/glossary/replay.md) URLs tied to each `sessionId`.

Steel is not trying to replace WebDriver. It supplies the managed browser runtime when your language mix or compliance rules mean Selenium is staying put. That means a custom connection class for Steel headers, explicit session creation with `is_selenium: true`, and predictable release calls once the driver quits.

## What stays the same
- `selenium.webdriver` API surface, from capabilities to waits.
- Test harnesses: pytest, JUnit, NUnit, or bespoke runners.
- Page objects, selectors, and assertion utilities.
- CI wiring: you still export `STEEL_API_KEY`, set env specific URLs, and fan out suites the same way.
- Local debugging: you can run against Chrome on your laptop until you point the job at Steel.

## What Steel adds
| Job | DIY grid or localhost | Selenium on Steel |
| --- | --- | --- |
| Startup | Wait for EC2 or an on-prem VM to warm Chrome | Steel Cloud sessions start in under one second and hand you a ready endpoint |
| Session length | Browsers restart frequently to keep hosts clean | Steel keeps sessions alive for 24 hours so long flows finish without relaunching |
| Evidence | Console logs and screenshots if you wired them yourself | Each `sessionId` maps to logs plus replay exports so failures are reviewable and shareable |
| Anti-bot | Patchy proxy rotation and brittle stealth flags | Steel Cloud supplies managed proxies, fingerprints, and CAPTCHA solving once Selenium parity lands |
| Scale & ops | Maintain Selenium Grid, Docker images, shared storage | Steel handles fleet health; you just connect and release |

## Minimal integration path
1. **Install the SDK** (`pip install steel-sdk selenium`) so you can create and release sessions explicitly.
2. **Create a session for Selenium**:
   ```python
   session = client.sessions.create(is_selenium=True, session_name="nightly-checkout", persistProfile=True)
   ```
3. **Wrap Selenium’s RemoteConnection** to add Steel headers:
   - `steel-api-key`: proves you can attach to the session
   - `session-id`: routes the WebDriver calls to the browser Steel already provisioned
4. **Connect your existing test** with `webdriver.Remote(command_executor=CustomRemoteConnection(...), options=webdriver.ChromeOptions())`.
5. **Release sessions on exit** using `client.sessions.release(session.id)` or `client.sessions.releaseAll()` in a `finally` block so concurrency slots free up fast.

### Example code (Python)
```python
import os
from selenium import webdriver
from selenium.webdriver.remote.remote_connection import RemoteConnection
from steel import Steel

STEEL_WS = "http://connect.steelbrowser.com/selenium"
client = Steel(steel_api_key=os.environ["STEEL_API_KEY"])

class SteelRemote(RemoteConnection):
    def __init__(self, url: str, session_id: str):
        super().__init__(url)
        self._session_id = session_id

    def get_remote_connection_headers(self, parsed_url, keep_alive=False):
        headers = super().get_remote_connection_headers(parsed_url, keep_alive)
        headers.update({
            "steel-api-key": os.environ["STEEL_API_KEY"],
            "session-id": self._session_id,
        })
        return headers

def run(url: str):
    session = client.sessions.create(is_selenium=True, session_name="nightly-checkout")
    driver = webdriver.Remote(
        command_executor=SteelRemote(STEEL_WS, session.id),
        options=webdriver.ChromeOptions(),
    )

    try:
        driver.get(url)
        # ... your assertions ...
    finally:
        driver.quit()
        client.sessions.release(session.id)
        details = client.sessions.retrieve(session.id)
        print("Replay:", details.get("hlsUrl"))

if __name__ == "__main__":
    run("https://news.ycombinator.com")
```

## Where Steel fits vs Selenium Grid
| Scenario | Keep Selenium Grid | Move to Steel |
| --- | --- | --- |
| Language coverage across Python, Java, C#, Ruby | Already works locally; no managed runtime needed | Same story, just point WebDriver at Steel and stop tending grid nodes |
| Long authenticated flows that time out mid run | Sessions restart unless you hand roll persistence | Steel sessions run 24 hours and can persist profiles up to 300 MB |
| Compliance and audit pressure | Evidence depends on homegrown logging | Steel records every run and keeps replay URLs plus agent logs per session |
| Anti-bot escalation | DIY scripts juggling proxies and CAPTCHA plugins | Steel handles proxies and CAPTCHA solving elsewhere in the product stack, so parity arrives without bespoke work |
| Burst scaling for nightly suites | Add more servers or wait in queue | Reserve higher Steel Cloud tiers for hundreds of concurrent sessions |

## Trade-offs and guardrails
- CAPTCHA solving, proxy settings, and live session viewer features are still rolling out for Selenium, so keep a backup plan for highly protected targets until parity lands.
- You must add the header wrapper because Selenium does not expose a direct way to inject Steel’s auth headers. Without it, Steel drops the connection.
- Sessions idle out if you forget to release them. Treat release calls as mandatory so you do not hit plan caps.
- Profiles support up to 300 MB and expire after 30 idle days. For heavy Selenium flows, trim downloads or export files through the Files API before quitting.
- Steel Local is fine for laptops or small CI agents when you want to keep everything on-prem. Use Steel Cloud once you need managed proxies, SOC 2 controls, or high concurrency.

## Works for / not yet
- Works when your org has thousands of Selenium specs and no appetite for a framework rewrite, but you still want reliable cloud browsers.
- Works when you need Python or Java bindings side by side with TypeScript-based agents because everything connects over the same Sessions API.
- Not yet ideal if your workflow depends on Steel’s CAPTCHA or proxy knobs right now; they ship first for Playwright and Puppeteer and will follow for Selenium.
- Not yet ideal when you need the live session viewer inside incident review; stick to recorded replays until the viewer supports Selenium sessions.

## Next steps
1. Drop the `SteelRemote` helper into your Selenium utilities module and point one suite at Steel Cloud.
2. Tag every session you create with your existing run or build ID so logs, replays, and approvals line up.
3. Capture the limitations above in your test README so the team knows which flows can move today and which need proxy or CAPTCHA parity.

Humans use Chrome. Agents use Steel.
