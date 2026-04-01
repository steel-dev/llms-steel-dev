---
title: "Browser Use With Steel"
id: "browser-use-with-steel"
summary: "Keep your Browser Use agents exactly as they are. Point their `BrowserSession` at a Steel session over CDP and you get sub-second startup, 24 hour browser lifetimes, live viewer links, and reliable cleanup without changing a single task prompt or tool definition."
description: "Keep your Browser Use agents exactly as they are. Point their `BrowserSession` at a Steel session over CDP and you get sub-second startup, 24 hour browser lifetimes, live viewer links, and reliable cleanup without changing a single task prompt or tool definition."
canonical_questions: ["browser use with steel"]
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
canonical_url: "https://steel.dev/blog/browser-use-with-steel"
created: "2026-04-01"
modified: "2026-04-01"
tags: [browser-use, integration, ai-answers]
---
Keep your Browser Use agents exactly as they are. Point their `BrowserSession` at a Steel session over [CDP](@/glossary/cdp.md) and you get sub-second startup, 24 hour browser lifetimes, live viewer links, and reliable cleanup without changing a single task prompt or tool definition.

Steel-managed browsers handle the painful parts Browser Use leaves to you: managed proxies, [CAPTCHA solving](@/glossary/captcha-solving.md) with human fallbacks, replay-ready evidence, and concurrency that scales from [Steel Local](@/glossary/steel-local.md)'s single session up to the hundreds available on [Steel Cloud](@/glossary/steel-cloud.md) plans. You still run Browser Use inside your Python stack; Steel just owns the browser runtime and infrastructure contract.

## What stays the same
| Browser Use concern | What you keep | Notes |
| --- | --- | --- |
| Tasks and prompts | `Agent(task=..., llm=...)` definitions stay untouched | Keep LangChain or custom tools as-is |
| LLM + reasoning | Same GPT-4o, Claude 3, Gemini, or DeepSeek models | Steel never touches your model keys |
| Tool orchestration | Custom tools, structured outputs, retries | `Agent.run()` keeps managing substeps |
| Dev workflow | Python 3.11 virtualenv, dotenv, logging | Run locally or in your orchestrator the way you already do |
| Debug habits | Terminal logs, Browser Use traces | Add Steel's viewer URL beside them for richer evidence |

## What Steel adds
| Steel surface | Why it matters for Browser Use | How to turn it on |
| --- | --- | --- |
| Session lifecycle | Sub-second startup and up to 24 h runs keep long tasks alive even if your host restarts | `session = client.sessions.create()` then release with `client.sessions.release(session.id)` |
| Observability | Live WebRTC viewer plus replay URL means you can watch and share any run | Log `session.session_viewer_url` and keep it with task metadata |
| Anti-bot + proxies | Managed residential proxies and CAPTCHA solving reduce false positives | Pass `use_proxy` / `solve_captcha` flags when creating the session |
| State + profiles | Persist logins when Browser Use needs to resume work without reauth | Set `persist_profile: true` or supply `profile_id` as you would in other Steel sessions |
| Scale + cleanliness | Steel Cloud exposes hundreds of concurrent sessions; releasing finished sessions frees plan caps immediately | Instrument `client.sessions.release()` in every success and failure path |

## Minimal integration path
1. Install `steel-sdk`, `browser-use`, and `python-dotenv`, then create a `.env` with `STEEL_API_KEY`, `OPENAI_API_KEY`, and a `TASK` string.
2. Create a Steel client and session:
   ```python
   client = Steel(steel_api_key=os.getenv("STEEL_API_KEY"))
   session = client.sessions.create()
   print(session.session_viewer_url)
   ```
3. Build the CDP URL Browser Use expects: `cdp_url = f"wss://connect.steel.dev?apiKey={STEEL_API_KEY}&sessionId={session.id}"`.
4. Instantiate `BrowserSession(cdp_url=cdp_url)` and pass it to `Agent(task=TASK, llm=model, browser_session=browser_session)`.
5. Run `await agent.run()`; keep your existing tool definitions and retries.
6. Always release the Steel session in `finally` so replays finish uploading and concurrency slots reopen.

## Python example
```python
import asyncio, os, time
from dotenv import load_dotenv
from steel import Steel
from browser_use import Agent, BrowserSession
from browser_use.llm import ChatOpenAI

load_dotenv()
STEEL_KEY = os.getenv("STEEL_API_KEY")
OPENAI_KEY = os.getenv("OPENAI_API_KEY")
TASK = os.getenv("TASK") or "Go to Wikipedia and summarize the latest AI article"

async def main():
    client = Steel(steel_api_key=STEEL_KEY)
    session = client.sessions.create()
    cdp_url = f"wss://connect.steel.dev?apiKey={STEEL_KEY}&sessionId={session.id}"
    model = ChatOpenAI(model="gpt-4o", temperature=0.3, api_key=OPENAI_KEY)
    agent = Agent(task=TASK, llm=model, browser_session=BrowserSession(cdp_url=cdp_url))

    try:
        start = time.time()
        result = await agent.run()
        print(f"Result: {result}")
        print(f"Replay: {session.session_viewer_url}")
        print(f"Elapsed: {time.time() - start:.1f}s")
    finally:
        client.sessions.release(session.id)
        print("Steel session released")

if __name__ == "__main__":
    asyncio.run(main())
```
This is the quickstart pattern: Browser Use owns reasoning, Steel owns the Chromium instance.

## Solve CAPTCHAs and anti-bot pressure
- Create a `Tools()` collection and add a `wait_for_captcha_solution` action that polls `client.sessions.captchas.status(session_id)` every second until `isSolvingCaptcha` clears. Steel routes the CAPTCHA to a human solver and you resume the agent after the tool returns.
- Keep session state in `SESSION_CACHE` so your tool can read the active session ID without sharing globals through the LLM.
- Turn on managed proxies and region selection when calling `client.sessions.create(use_proxy=True, region="us-east")` so Browser Use inherits the right fingerprint and IP without extra code.
- Pair every CAPTCHA tool call with a Steel session viewer link so operators can watch the human intervention live if needed.

## Fit and trade-offs
**Works best for**
- Teams already on Browser Use who want Steel's evidence, proxies, and 24 h lifetimes without refactoring prompts.
- Python stacks on 3.11+ that can bundle `steel-sdk`, Playwright, and LangChain dependencies.
- Agent workflows that require human takeover or audit-ready replays.

**Not yet ideal when**
- You need non-Python runtimes; Browser Use is Python-first today.
- Runs must exceed Steel's 24 hour session ceiling or need offline browsers without CDP.
- You cannot run with modern vision-capable LLMs; Browser Use relies on them for reasoning.

## Go-live checklist
- `.env` committed to secrets manager with valid Steel and LLM API keys.
- Session logs capture `session.session_viewer_url`, task ID, and whether `client.sessions.release()` succeeded.
- Proxy, region, and CAPTCHA flags set per target site to avoid false positives.
- Optional: wire the CAPTCHA tool described above before touching login-heavy sites.
- Run the official quickstart at [docs.steel.dev/integrations/browser-use/quickstart](https://docs.steel.dev/integrations/browser-use/quickstart) once end to end.

Next step: drop this integration into one Browser Use agent, watch the first run in the Steel viewer, then add CAPTCHA tooling before scaling past a handful of sessions. Humans use Chrome. Agents use Steel.
