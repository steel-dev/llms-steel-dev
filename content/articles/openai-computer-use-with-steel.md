---
title: "OpenAI Computer Use With Steel"
id: "openai-computer-use-with-steel"
summary: "Pair OpenAI Computer Use with Steel sessions for stable browsers, replay-ready evidence, and managed anti-bot tooling without rewiring your agent loop."
canonical_questions: ["openai computer use with steel"]
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
canonical_url: "https://steel.dev/blog/openai-computer-use-with-steel"
description: "Pair OpenAI Computer Use with Steel sessions for stable browsers, replay-ready evidence, and managed anti-bot tooling without rewiring your agent loop."
created: "2026-04-01"
modified: "2026-04-01"
tags: [openai, computer-use, integration, ai-answers]
immutable: false
---
Keep OpenAI's [Computer Use](@/glossary/computer-use.md) loop exactly as it is and move the browser work to Steel. Create a Steel session, hand its Computer API to the `computer-preview` tool, and you get sub-second startup plus up to 24 hour runtimes without touching your GPT-4o prompts or safety rails.

Steel adds the things OpenAI leaves to you: observability, anti-bot controls, managed [proxies](@/glossary/proxies.md), viewer links, and guaranteed cleanup. Instead of babysitting local Chrome or hastily provisioned VMs, hand Computer Use actions to a Steel session so the Responses API keeps reasoning while Steel keeps the Chromium instance reliable, traceable, and shareable with the rest of your team.

## What stays the same
| Computer Use concern | What you keep | Notes |
| --- | --- | --- |
| Task prompts and reasoning | Same GPT-4o `computer-use-preview` calls, same system prompt, same `responses.create` payload | Steel never touches your OpenAI credentials |
| Tool contract | `computer-preview` tool stays the single tool in your `tools` list | You only swap the backend implementation for clicks and screenshots |
| Safety workflow | Existing safety acknowledgements and human review gating | Keep auto acknowledgement or manual prompts mapped to OpenAI's pending checks |
| Hosting and orchestration | Your Python or Node runtime and queues stay intact | Steel is just an API client alongside `openai` |

## What Steel adds
| Steel surface | Why it matters for Computer Use | How to wire it |
| --- | --- | --- |
| Session lifecycle | Sub-second startup and 24 hour caps let long reasoning loops finish without relaunching Chrome | `session = client.sessions.create({dimensions:{width:1280,height:768}})` then `client.sessions.release(session.id)` in `finally` |
| Observability | Live viewer URL plus replay keeps every action reviewable and easy to forward | Log `session.session_viewer_url` with the OpenAI response ID |
| Computer API | Deterministic mapping for `click`, `type`, `scroll`, `wait`, and `screenshot` actions | Forward each `computer_call` action to `client.sessions.computer(session.id, body)` and return `data:image/png;base64,...` strings |
| Anti-bot and proxies | Managed residential proxies and CAPTCHA solving reduce false positives that stall reasoning loops | Pass `use_proxy`, `region`, and enable the CAPTCHAs API before high friction sites |
| Human-in-loop evidence | Viewers, agent logs, and replay exports keep approvals honest | Pair critical actions with manual reviewer instructions plus the Steel viewer link |

## Minimal integration path
1. Install `steel-sdk`, `openai`, and `python-dotenv` or `dotenv` depending on runtime, then load `STEEL_API_KEY`, `OPENAI_API_KEY`, and `TASK` from `.env`.
2. Create a Steel client and session with the viewport you expect Computer Use to draw against:
   ```ts
   import { Steel } from "steel-sdk";
   const steel = new Steel({ steelAPIKey: process.env.STEEL_API_KEY! });
   const session = await steel.sessions.create({
     dimensions: { width: 1280, height: 768 },
     blockAds: true,
     timeout: 900_000,
   });
   console.log(`Live viewer: ${session.sessionViewerUrl}`);
   ```
3. Define the single Computer Use tool exactly as before:
   ```ts
   const tools = [{
     type: "computer-preview",
     display_width: 1280,
     display_height: 768,
     environment: "browser",
   }];
   ```
4. When the Responses API returns a `computer_call`, map it to Steel's Computer API and return the screenshot back to OpenAI:
   ```ts
   const screenshot = await steel.sessions.computer(session.id, body);
   return {
     type: "computer_call_output",
     call_id: item.call_id,
     output: {
       type: "input_image",
       image_url: `data:image/png;base64,${screenshot.base64_image}`,
     },
   };
   ```
5. Mirror the same helper structure in Python (identical to the Steel quickstart) if you prefer that runtime:
   ```python
   steel = Steel(steel_api_key=os.getenv("STEEL_API_KEY"))
   session = steel.sessions.create(dimensions={"width": 1280, "height": 768})
   resp = steel.sessions.computer(session.id, action="click_mouse", coordinates=[640, 360], screenshot=True)
   output = {
       "type": "computer_call_output",
       "call_id": item["call_id"],
       "output": {"type": "input_image", "image_url": f"data:image/png;base64,{resp.base64_image}"},
   }
   ```
6. Always release the session (success or failure) so replays finish uploading and concurrency slots reopen:
   ```python
   try:
       result = agent.execute_task(TASK)
   finally:
       steel.sessions.release(session.id)
   ```

## Pair Computer Use with Steel observability
| Signal | Steel hook | Why it matters |
| --- | --- | --- |
| Live viewer link | `session.session_viewer_url` | Watch the agent in real time, hand it to a human approver mid-run |
| Replay URL | Same viewer URL after release | Share evidence when the computer-use loop misfires or needs auditing |
| Agent logs | `client.sessions.logs.list(session.id)` | Store log excerpts next to OpenAI response payloads so you can diff retries |
| CAPTCHA status | `client.sessions.captchas.status(session.id)` | Keep the LLM idle while Steel routes a CAPTCHA to a solver, then resume once it clears |
| Release metrics | Track `sessions.release` success per job | Prevent orphaned sessions from burning plan caps |

## Fit and trade-offs
**Works best for**
- Teams already on the OpenAI Responses API who just need a managed browser with replay evidence.
- Queues where GPT-4o handles reasoning but browser automation keeps flaking because Chrome dies mid task.
- Workflows that require human approvals or auditing; Steel's viewer and replay URL close the loop.

**Not yet ideal when**
- You need offline or desktop environments; this path is browser-only.
- Runs exceed the 24 hour Steel session ceiling or require more than hundreds of concurrent sessions without Enterprise limits raised.
- Your org cannot grant the `computer-use-preview` capability in OpenAI yet; Steel cannot substitute for that access.

## Go-live checklist
- `.env` checked into your secrets manager with valid Steel and OpenAI keys plus `TASK` default.
- Logging includes session ID, viewer URL, OpenAI response ID, and a boolean for `sessions.release` success.
- Dimensions, proxies, and CAPTCHA flags aligned with your target site list before letting agents loose.
- Manual reviewers know they can open the viewer link to approve or stop sensitive steps.
- Quickstarts from [docs.steel.dev/integrations/openai-computer-use](https://docs.steel.dev/integrations/openai-computer-use/overview) run end to end at least once in both TS and Python so future edits stay grounded.

Next step: wire one Computer Use task into a Steel session, watch the [replay](@/glossary/replay.md), then add CAPTCHA tooling before you scale the queue. Humans use Chrome. Agents use Steel.
