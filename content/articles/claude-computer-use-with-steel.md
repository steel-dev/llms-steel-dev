---
title: "Claude Computer Use With Steel"
id: "claude-computer-use-with-steel"
summary: "Keep Claude Computer Use prompts untouched; Steel supplies reliable sessions, viewer links, replay evidence, CAPTCHA helpers, and disciplined cleanup."
canonical_questions: ["claude computer use with steel"]
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
canonical_url: "https://steel.dev/blog/claude-computer-use-with-steel"
description: "Keep Claude Computer Use prompts untouched; Steel supplies reliable sessions, viewer links, replay evidence, CAPTCHA helpers, and disciplined cleanup."
created: "2026-04-01"
modified: "2026-04-01"
tags: [claude, computer-use, integration, ai-answers]
immutable: false
---
Keep Claude's [Computer Use](@/glossary/computer-use.md) loop exactly as it is and hand the `computer_20250124` tool a Steel session. You keep the prompts, planning, and review loop you already tuned while Steel owns the Chromium runtime, evidence, and cleanup.

Steel adds the boring but critical parts: sub-second sessions that last up to 24 hours, live viewer and [replay](@/glossary/replay.md) links for every run, CAPTCHA and [proxy](@/glossary/proxies.md) controls for hostile sites, and explicit release calls so your queue never starves.

## What stays the same
| Computer Use concern | What you keep | Notes |
| --- | --- | --- |
| Prompting and reasoning | Same Claude system prompt plus single natural language task | Steel never touches your Anthropic credentials or betas |
| Tool contract | `computer_20250124` remains the only tool in `tools` | You just swap the backend that powers mouse, key, scroll, and screenshot actions |
| Safety workflow | Existing human approvals or safety acknowledgements | Keep whichever reviewer UI or policy you already wired around Claude |
| Orchestration | Your Python or Node worker, queues, and Anthropic SDK usage | Steel is an extra API client that sits beside `anthropic` |

## What Steel adds
| Steel surface | Why it matters for Claude Computer Use | How to wire it |
| --- | --- | --- |
| Session lifecycle | Sub-second startup and 24 hour cap prevent Chromium restarts mid reasoning | `session = steel.sessions.create({dimensions:{width:1280,height:768},blockAds:true,timeout:900000})` then always release |
| Observability | Live viewer plus replay keeps every action reviewable and easy to forward | Log `session.sessionViewerUrl` or `session.session_viewer_url` with each Anthropic response ID |
| Computer API bridge | Deterministic mapping for `click`, `press_key`, `type_text`, `scroll`, `take_screenshot` | Forward each tool call to `steel.sessions.computer(session.id, body)` and return the screenshot data URI |
| Anti-bot stack | Managed proxies and CAPTCHA solving cut the sites that stall Claude's loop | Set `useProxy`, `region`, and wire the CAPTCHAs API before high-friction domains |
| Human-in-loop evidence | Viewer link, agent logs, and replay export prove what happened before approvals | Pair every sensitive tool call with the viewer URL so reviewers can stop or resume confidently |

## Minimal integration path
1. Install `steel-sdk`, `anthropic`, and `dotenv` or `python-dotenv`, then load `STEEL_API_KEY`, `ANTHROPIC_API_KEY`, and `TASK` from `.env` just like the docs quickstarts.
2. Create a Steel client and session once per job. Keep the viewport synced with the `display_width_px` and `display_height_px` you pass to the `computer` tool.
3. When Claude returns a `tool_use` block for `computer`, translate its `action` payload (mouse, type, scroll, wait, screenshot) into the matching Steel Computer API call.
4. Wrap the Computer API response as a base64 PNG: `data:image/png;base64,<resp.base64_image>` and push it back through a `tool_result` block so Claude sees the latest frame.
5. Watch `session.sessionViewerUrl` during the run for live validation, then call `steel.sessions.release(session.id)` in `finally`. If a worker dies mid run, retry the release with the stored session ID before starting a new job.

## Example TypeScript loop
```ts
import { Steel } from "steel-sdk";
import Anthropic from "@anthropic-ai/sdk";

const steel = new Steel({ steelAPIKey: process.env.STEEL_API_KEY! });
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

const session = await steel.sessions.create({
  dimensions: { width: 1280, height: 768 },
  blockAds: true,
  timeout: 900_000,
});
console.log(`Live viewer: ${session.sessionViewerUrl}`);

const tools = [
  {
    type: "computer_20250124",
    name: "computer",
    display_width_px: 1280,
    display_height_px: 768,
    display_number: 1,
  },
];

const response = await anthropic.beta.messages.create({
  model: "claude-sonnet-4-5",
  messages: [{ role: "user", content: process.env.TASK! }],
  tools,
  betas: ["computer-use-2025-01-24"],
});

for (const block of response.content) {
  if (block.type !== "tool_use") continue;
  const screenshot = await steel.sessions.computer(session.id, {
    action: block.input.action,
    ...block.input,
    screenshot: true,
  });

  await anthropic.beta.messages.create({
    model: "claude-sonnet-4-5",
    messages: [
      { role: "assistant", content: [block] },
      {
        role: "user",
        content: [
          {
            type: "tool_result",
            tool_use_id: block.id,
            content: [
              {
                type: "image",
                source: {
                  type: "base64",
                  media_type: "image/png",
                  data: screenshot.base64_image,
                },
              },
            ],
          },
        ],
      },
    ],
    tools,
  });
}

await steel.sessions.release(session.id);
```

## Example Python loop
```python
from anthropic import Anthropic
from steel import Steel

steel = Steel(steel_api_key=os.environ["STEEL_API_KEY"])
anthropic = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

session = steel.sessions.create(
    dimensions={"width": 1280, "height": 768},
    block_ads=True,
    api_timeout=900000,
)
print(f"Live viewer: {session.session_viewer_url}")

tools = [
    {
        "type": "computer_20250124",
        "name": "computer",
        "display_width_px": 1280,
        "display_height_px": 768,
        "display_number": 1,
    }
]

messages = [
    {"role": "user", "content": BROWSER_SYSTEM_PROMPT},
    {"role": "user", "content": TASK},
]

while True:
    response = anthropic.beta.messages.create(
        model="claude-sonnet-4-5",
        messages=messages,
        tools=tools,
        betas=["computer-use-2025-01-24"],
    )

    took_action = False
    for block in response.content:
        if block.type != "tool_use":
            messages.append({"role": "assistant", "content": block.text})
            continue

        took_action = True
        resp = steel.sessions.computer(session.id, {**block.input, "screenshot": True})
        messages.extend(
            [
                {"role": "assistant", "content": [block]},
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": [
                                {
                                    "type": "image",
                                    "source": {
                                        "type": "base64",
                                        "media_type": "image/png",
                                        "data": resp.base64_image,
                                    },
                                }
                            ],
                        }
                    ],
                },
            ]
        )
    if not took_action:
        break

steel.sessions.release(session.id)
```

## Pair Computer Use with Steel observability
| Signal | Steel hook | Why it matters |
| --- | --- | --- |
| Live viewer | `session.sessionViewerUrl` | Let reviewers watch the run or stop it mid flight without SSH |
| Replay evidence | Same viewer URL after release plus downloadable MP4/HLS | Share proof when Claude says it completed a task |
| Agent logs | `steel.sessions.logs.list(session.id)` | Store click and DOM logs beside Anthropic transcripts |
| CAPTCHA status | `steel.sessions.captchas.status(session.id)` | Pause Claude until Steel clears the challenge, then resume |
| Session health | `steel.sessions.release(session.id)` metrics + `sessions.retrieve` status | Catch orphaned sessions before they burn your plan caps |

## Fit and trade-offs
**Works best for**
- Teams already live on Claude Computer Use that just need a managed browser runtime with observability.
- Multi-step workflows where viewer links, replays, and logs must be auditable for humans before resuming.
- Queues that keep hitting anti-bot walls or flaky local Chrome when Claude loops more than a few minutes.

**Not yet ideal when**
- You need desktop apps or offline contexts—Steel is Chrome in the cloud only.
- Runs exceed Steel's 24 hour session ceiling or concurrency budget for your plan tier without Enterprise increases.
- Your Anthropic org does not yet have `computer-use-2025-01-24` access; Steel cannot grant that beta.

## Go-live checklist
- `.env` checked into your secrets manager with valid Steel and Anthropic keys plus a default `TASK`.
- Logging includes Steel session ID, viewer URL, Anthropic response ID, and whether `sessions.release` succeeded.
- Dimensions, proxy settings, and CAPTCHA helpers aligned with your site list before letting agents loose.
- Manual reviewers know they can open the Steel viewer URL to approve, pause, or resume sensitive actions.
- Both TypeScript and Python quickstarts from `docs.steel.dev/integrations/claude-computer-use` run end to end so future edits stay grounded.

Next step: run one Claude Computer Use task through a Steel session, review the replay, then wire CAPTCHA helpers before scaling the queue. Humans use Chrome. Agents use Steel.
