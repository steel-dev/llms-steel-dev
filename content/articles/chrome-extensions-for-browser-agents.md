---
title: "Chrome Extensions for Browser Agents"
id: "chrome-extensions-for-browser-agents"
summary: "Chrome extensions belong in browser-agent workflows when you need repeatable in-browser logic that Playwright or prompts cannot keep stable; Steel's Extensions API lets you upload once, inject per session, and keep the lifecycle sane."
canonical_questions: ["chrome extensions for browser agents"]
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
canonical_url: "https://steel.dev/blog/chrome-extensions-for-browser-agents"
description: "Chrome extensions belong in browser-agent workflows when you need repeatable in-browser logic that Playwright or prompts cannot keep stable; Steel's Extensions API lets you upload once, inject per session, and keep the lifecycle sane."
created: "2026-03-31"
modified: "2026-03-31"
tags: [chrome-extensions, browser-agents, implementation-guide]
immutable: false
---
## Short answer
Use Chrome extensions when your agent needs persistent, browser-side logic that scripts or prompts keep breaking. Steel's Extensions API (currently in beta) lets you upload an extension once for your organization—via `.zip`, `.crx`, or a Chrome Web Store URL—and inject it into any session by passing `extensionIds` or the shortcut `all_ext` when you create the session. That keeps shared DOM observers, UI overlays, and compliance helpers available to every workflow without rewriting [CDP](@/glossary/cdp.md) code each time.

Skip extensions when a lightweight Playwright step or content script edit will do. Extensions load only when the Steel session starts, they carry the permissions declared in `manifest.json`, and they add weight to the session. If the logic is simple, keep it inside your orchestrator; if it needs to run quietly beside the agent every time, make it an extension.

### When extensions belong inside the workflow
| Pain | Without extensions | Steel move | Evidence you gain |
| --- | --- | --- | --- |
| Agents repeatedly rebuild the same DOM hooks to read data or mutate UI | Prompted scripts re-query selectors differently every run | Upload the helper as an org extension and attach it with `extensionIds: ['helper_id']` | Consistent background script output plus replayable behavior tied to the session ID |
| Reviewers need a human overlay (redaction, watermark, approval toggles) inside the live session | You inject brittle JS via `evaluate` on every sensitive page | Ship a policy overlay extension, then run the workflow with `extensionIds: ['policy_overlay']` so it's always loaded | Live-view and replay both show the same guardrails, satisfying audit trails |
| You want instrumentation (performance timings, schema emitters) across all runs | Each script logs differently, and artifacts drift | Use `all_ext` so every managed session boots with the telemetry bundle | Every trace, HLS replay, and download already includes the custom metrics |
| You need a marketplace extension (password managers, translators) to finish the flow | Manual install on operators' machines, nothing for agents | Upload from the Chrome Web Store URL once, then keep using the assigned `extensionId` | Session evidence shows the same UI the marketplace add-on renders |

## Instead of patching scripts forever, treat extensions as shared instrumentation
Extensions turn one-off glue into an asset. Because Steel stores them at the org level, you upload once, list IDs with `client.extensions.list()`, and keep reusing them across agents, SDKs, and regions. That means the agent prompt can stay simple (“click @e4, read the card, return JSON”) while the extension handles the gnarly parts: injecting schema hints, suppressing pop-ups, or masking PII before snapshots land in your evidence folder. They also play nicely with human-in-the-loop steps because the same extension is visible in live embeds and HLS replays.

## Implementation path
1. **Package or pick your extension**
   - Zip your `manifest.json` plus source, or copy a Chrome Web Store URL.
   - Remember: the permissions you declare apply to every session that loads it; keep them as narrow as possible.
2. **Upload once per organization**
   ```typescript
   await client.extensions.upload({
     file: fs.readFileSync('extensions/redactor/redactor.zip')
   });
   // or
   await client.extensions.upload({
     url: 'https://chromewebstore.google.com/detail/...'
   });
   ```
3. **Discover the IDs you need to reference**
   ```typescript
   const extensions = await client.extensions.list();
   const redactorId = extensions.find((ext) => ext.name === 'Redactor').id;
   ```
4. **Attach extensions when you create sessions**
   ```python
   session = client.sessions.create(
       extension_ids=[redactorId, 'helper_metrics']  # or ['all_ext']
   )
   ```
   Extensions load and initialize when the session boots, so plan to start a new session when you change the set.
5. **Verify inside the run**
   - Use `steel browser live` or the debug URL to confirm the extension shows up in `chrome://extensions` and renders its UI.
   - Capture a `snapshot -i` so the evidence includes the extension's overlay or output.
6. **Update or retire as workflows evolve**
   ```typescript
   await client.extensions.update(redactorId, {
     file: fs.readFileSync('extensions/redactor/v2.zip')
   });
   await client.extensions.delete(redactorId);      // remove single
   await client.extensions.deleteAll();             // wipe everything
   ```
   Updates replace the stored artifact; deletions keep future sessions lean without touching past traces.

## Guardrails and trade-offs
| Works when... | Still limited when... |
| --- | --- |
| You need Chrome-only capabilities (content scripts, background workers) to run alongside every agent | You expect hot-reloading mid-session; extensions only load at startup |
| Your org wants one reviewed copy of a helper (schema extractor, visual diff) across many teams | You rely on APIs beyond what Chrome exposes; the extension framework cannot call Steel services directly without your glue |
| You must prove the agent saw the same UI humans do, including overlays | Simple DOM tweaks would be faster in Playwright—skip the extension to cut startup overhead |
| You need marketplace extensions (password manager, translator) without handing the agent your personal browser | Extensions are in beta; expect changes and keep a fallback path ready |

## Next steps
- Read the [Extensions API overview](https://docs.steel.dev/overview/extensions-api/overview) for every endpoint (`upload`, `list`, `update`, `delete`, `deleteAll`).
- Start a fresh Steel session with `extension_ids=['all_ext']` and confirm your helper loads before the agent runs its first step.
- Write down which workflows truly need an extension (shared overlays, instrumentation) and leave everything else to lightweight scripts.

Humans use Chrome. Agents use Steel.
