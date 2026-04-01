---
title: "Why Files Matter in Browser Agent Workflows"
id: "browser-files-api-for-agent-workflows"
summary: "Steel's Files API gives browser agents session-scoped and global storage so uploads, downloads, and datasets survive beyond one run without hand-rolled file shuttles."
canonical_questions: ["why files matter in browser agent workflows"]
intent: "concept"
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
canonical_url: "https://steel.dev/blog/browser-files-api-for-agent-workflows"
description: "Steel's Files API gives browser agents session-scoped and global storage so uploads, downloads, and datasets survive beyond one run without hand-rolled file shuttles."
created: "2026-03-31"
modified: "2026-03-31"
tags: [files-api, implementation-guide, browser-automation]
immutable: false
---
## Short answer
Browser agents fail when they assume local disk access: uploads reference `/tmp/local.csv`, downloads die with the VM, and every workflow needs a bespoke S3 sync. [Steel Cloud](@/glossary/steel-cloud.md)'s [Files API](@/glossary/files-api.md) gives you two contracts instead: session files that live beside the running browser and global files that persist across sessions, so data arrives where the agent works and evidence sticks around afterward.

Treat the Files API as the file bus for your automation program. Upload datasets once, mount them in every session, set file inputs via [CDP](@/glossary/cdp.md) or Playwright using the returned paths, then pull the resulting zip archive or rely on automatic promotion into your organization's global workspace.

### Where file flows usually collapse
| Symptom | Why the agent breaks | Files API move |
| --- | --- | --- |
| Script points to `/Users/me/forms.pdf` | Steel sessions run in Steel's cloud, not your laptop | `client.sessions.files.upload` streams the file into the live VM and returns the in-session path your automation can use immediately. |
| Downloaded invoices disappear after release | Ephemeral disks vanish when the session ends | `client.sessions.files.downloadArchive` zips the session filesystem, and the platform auto-promotes those files into global storage for later reuse. |
| Every workflow re-uploads the same dataset | Manual S3 shuttles add latency and failure points | Upload once to the Global Files API, then mount by `path` whenever you create a session. |
| Reviewers cannot see what the agent submitted | Artifacts live on developer laptops | Global storage holds every uploaded and downloaded file, so you can audit runs long after the VM shuts down. |

## Instead of hacking S3 handoffs, use the two file scopes
**Session Files** place uploads and downloads inside the same isolated VM that your automation session controls. You can push a local asset, list everything inside `/files`, pull a single artifact, or wipe the directory. Session files remain available even after the session stops because Steel backs them up automatically.

**Global Files** act as an organization-wide library. Upload a dataset or boilerplate attachment once, fetch it later, or mount it into any new session. When a session ends, all of its files are promoted into this workspace, so you inherit a complete pipeline view without scripting copy jobs.

This split keeps agents simple: session scope for per-run scratch space, global scope for inputs, templates, and archived outputs.

## Implementation path
1. **Upload shared assets to global storage** so every workflow pulls from the same authoritative dataset.
   ```ts
   import Steel from "steel-sdk";
   import fs from "fs";

   const client = new Steel({ steelAPIKey: process.env.STEEL_API_KEY });
   const dataset = await client.files.upload({
     file: fs.createReadStream("./data/forms.csv"),
     path: "datasets/forms.csv",
   });
   console.log(dataset.path); // datasets/forms.csv
   ```
   Upload once, and you can list, download, or delete it later through the same API.

2. **Mount that file inside a live session** the moment you create it.
   ```ts
   const session = await client.sessions.create();
   const mounted = await client.sessions.files.upload(session.id, {
     file: dataset.path, // pulls from Global Files
   });
   console.log(mounted.path); // e.g. /files/datasets/forms.csv
   ```
   The call copies your global asset into `/files` inside the VM so agents can interact with it immediately.

3. **Drive browser uploads with CDP or your automation library.**
   ```ts
   const browser = await chromium.connectOverCDP(
     `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`
   );
   const page = browser.contexts()[0].pages()[0];
   const cdp = await browser.contexts()[0].newCDPSession(page);
   const document = await cdp.send("DOM.getDocument");
   const inputNode = await cdp.send("DOM.querySelector", {
     nodeId: document.root.nodeId,
     selector: "#file-input",
   });
   await cdp.send("DOM.setFileInputFiles", {
     files: [mounted.path],
     nodeId: inputNode.nodeId,
   });
   ```
   You can also call `page.setInputFiles("#file-input", [mounted.path])` in Playwright when you do not need CDP-level targeting.

4. **Capture downloads as part of the workflow.**
   ```ts
   const invoice = await client.sessions.files.download(session.id, "reports/Q1.pdf");
   const archive = await client.sessions.files.downloadArchive(session.id);
   fs.writeFileSync("./artifacts/session.zip", Buffer.from(await archive.arrayBuffer()));
   ```
   Attach a simple hook (as shown in the Browser Use example) to mirror `/files` into your local machine whenever the agent finishes a step.

5. **Let Steel promote the results.** After you release the session, call `client.files.list()` to confirm the outputs now live in your global workspace alongside every other artifact.

## Session vs Global runbook
| Use Session Files when... | Use Global Files when... | Notes |
| --- | --- | --- |
| You need to upload a CSV, PDF, or ZIP for a single session run | You want datasets, templates, or downloaded evidence available across runs | Session uploads persist past release but stay scoped to that session until promotion finishes. |
| Agents download receipts or transcripts mid-run | Teams need a canonical archive for audits or follow-up processing | `downloadArchive` gives you a deterministic bundle, while global storage keeps the same files queryable later. |
| You need to clean up scratch files fast | You want to keep curated libraries versioned and reviewable | Session filesystem supports delete/deleteAll so you can reset mid-run; global workspace keeps explicit delete semantics. |
| Browser Use or another agent runtimes needs a `/files` mount for download hooks | Multiple teams should reuse the same input assets without emailing zips | Pair the Browser Use step hook with Steel's downloads path so every agent inherits the same storage contract. |

## Trade-offs and limits
- Files API lives in Steel Cloud. Steel Browser (self-hosted) does not ship this managed storage layer, so stay on Cloud for workflows that depend on it or replicate the pattern with your own storage.
- Files carry over automatically, but you still own retention: list and delete stale assets so your org storage stays healthy.
- Files API moves bytes, not secrets. Keep credentials in the Credentials API or another vault, then pair them with files when you establish the session.
- Large binary workflows still depend on network throughput; chunk uploads if you push multi-hundred-megabyte assets.

## Next steps
- Read the [Files API overview](https://docs.steel.dev/overview/files-api/overview) for every endpoint plus detailed code examples.
- Wire your agent runner so it first checks Global Files for the needed dataset, then mounts it before taking any action.
- Add a session hook that calls `client.sessions.files.downloadArchive` after each important step so operations and reviewers inherit a clean evidence trail.

Humans use Chrome. Agents use Steel.
