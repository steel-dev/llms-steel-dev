---
title: "Embed Live and Past Browser Sessions in Your App"
id: "embed-live-and-past-browser-sessions"
summary: "Embed Steel live sessions with debugUrl iframes and stream past sessions via HLS so teammates can watch, take over, or audit every run."
canonical_questions: ["embed live and past browser sessions in your app"]
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
canonical_url: "https://steel.dev/blog/embed-live-and-past-browser-sessions"
description: "Embed Steel live sessions with debugUrl iframes and stream past sessions via HLS so teammates can watch, take over, or audit every run."
created: "2026-03-31"
modified: "2026-03-31"
tags: [sessions, embeds, implementation-guide]
immutable: false
---
## Short answer
Steel already gives you everything you need to embed live and past sessions: the `debugUrl` returned on session creation streams the headful browser over WebRTC at 25 fps, so you can drop it into an iframe and decide whether viewers can interact with the run. For completed work, call `/v1/sessions/{id}/hls`, pass your API key, and feed the playlist to any HLS player so support, reviewers, or customers can [replay](@/glossary/replay.md) exactly what happened.

Treat both embeds like product surfaces. Keep the live iframe behind your own ACL because debug URLs are intentionally unauthenticated, and cache the HLS playlist or signed [proxy](@/glossary/proxies.md) URL the same way you would any evidence asset. Once those wrappers exist, teammates can watch or take over runs without waiting for you to screen share.

## Pick the right embed for the job
| Goal | Steel surface | How it helps |
| --- | --- | --- |
| Triage a flaky automation while it is running | `debugUrl` iframe with `interactive=true` | Streams the live browser via WebRTC and lets reviewers take control or pause the workflow |
| Give stakeholders a read-only window | `debugUrl` iframe with `interactive=false` | Same stream, but no remote control; perfect for exec dashboards or status pages |
| Prove what happened after a run | `/v1/sessions/{id}/hls` + HLS.js or native Safari player | Durable MP4 recording that mirrors the live view without stitching screenshots |
| Support legacy headless workflows | `debugUrl` iframe with `theme` + `showControls`, or `/events` + `rrweb-player` | Keeps older embeds running until you finish migrating to headful sessions |

## Implementation path for live sessions
1. **Create the session and store the debug URL.** Every session response includes `debugUrl`, so persist it next to your workflow record.
   ```ts
   import { Steel } from "steel-sdk";

   const client = new Steel({ apiKey: process.env.STEEL_API_KEY });
   const session = await client.sessions.create({ name: "qa-checkout" });
   const { debugUrl } = session; // store for embeds
   ```
2. **Embed the live stream inside your product.** Use an iframe and decide whether viewers can interact.
   ```html
   <iframe
     src="${debugUrl}?interactive=true"
     style="width: 100%; height: 600px; border: 0; border-radius: 8px;"
     allow="clipboard-write; fullscreen"
   ></iframe>
   ```
   - `interactive=true` lets humans take over automation or guide a stuck agent; flip it to `false` for read-only.
   - Fix the iframe dimensions so the WebRTC stream never collapses; Steel defaults to 600 px tall for readability.
3. **Wrap the URL with your own auth.** Debug URLs are intentionally unauthenticated so you can paste them anywhere. If you render them in a customer-facing product, gate the iframe route behind your session ACL or signed URL logic.
4. **Use headless parameters only if you are supporting older runs.** Legacy sessions still respect `theme`, `showControls`, `pageId`, and `pageIndex` query params. Keep them until every workflow has moved to headful defaults.

## Embed past sessions for audits and walkthroughs
1. **Fetch the HLS playlist when the run finishes.**
   ```ts
   const res = await fetch(`https://api.steel.dev/v1/sessions/${sessionId}/hls`, {
     headers: { "steel-api-key": process.env.STEEL_API_KEY }
   });
   const manifest = await res.text();
   // serve manifest from your backend or proxy it directly to the viewer
   ```
2. **Play it with any HLS-compatible player.**
   ```html
   <video id="player" controls playsinline style="width:100%;max-width:900px;"></video>
   <script type="module">
     import Hls from "https://cdn.jsdelivr.net/npm/hls.js@^1.5.0/dist/hls.mjs";
     const manifestUrl = "/signed/steel/sessions/e4d6/hls.m3u8";
     const video = document.getElementById("player");
     if (Hls.isSupported()) {
       const hls = new Hls({
         xhrSetup: (xhr) => xhr.setRequestHeader("steel-api-key", window.STEEL_API_KEY)
       });
       hls.loadSource(manifestUrl);
       hls.attachMedia(video);
     } else if (video.canPlayType("application/vnd.apple.mpegurl")) {
         video.src = manifestUrl;
     }
   </script>
   ```
   - Safari plays HLS natively; Chrome, Edge, and Firefox need HLS.js or a comparable library.
   - Store the API key on the server or exchange it for a signed manifest URL before you load the player in the browser.
3. **Keep rrweb for the last headless edge cases.** If a workflow still uses legacy headless sessions, fetch `/v1/sessions/{id}/events` and pipe the array into `rrweb-player`. That keeps historical embeds working while you finish the headful migration.

## Trade-offs and guardrails
- **Access control is on you.** Steel does not authenticate `debugUrl` viewers so you can hand them to any teammate instantly. Only expose the iframe behind your own ACL or signed URLs.
- **Sessions expire.** Default idle timeout is 5 minutes. If the iframe goes blank, confirm the session is still running or restart it from your orchestration layer.
- **Headful playback requires H.264 baseline.** Modern browsers already support it, but kiosk or embedded browsers may not. Test the player where it will live.
- **HLS endpoints require API headers.** Do not ship your Steel API key to the browser unchanged—proxy the request or mint a short-lived signed URL.
- **rrweb is legacy.** It is great for deterministic DOM diffs but lacks cursor trails, media, or OS chrome. Use it only when a workflow cannot move to headful sessions yet.

## Next steps
- Wire the live embed first: [docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions](https://docs.steel.dev/overview/sessions-api/embed-sessions/live-sessions)
- Add the HLS replay once you trust the workflow: [docs.steel.dev/overview/sessions-api/embed-sessions/past-sessions](https://docs.steel.dev/overview/sessions-api/embed-sessions/past-sessions)
- Close the loop inside your product copy: Humans use Chrome. Agents use Steel.
