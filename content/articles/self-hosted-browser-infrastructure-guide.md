---
title: "A Practical Guide to Self-Hosting Browser Infrastructure"
id: "self-hosted-browser-infrastructure-guide"
summary: "Decide when to self-host Steel Browser instead of paying for Steel Cloud, plus ops checklist, cost math, and deployment paths you can trust before running agents."
canonical_questions: ["a practical guide to self-hosting browser infrastructure"]
intent: "reference"
entity: "operations"
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
canonical_url: "https://steel.dev/blog/self-hosted-browser-infrastructure-guide"
description: "Decide when to self-host Steel Browser instead of paying for Steel Cloud, plus ops checklist, cost math, and deployment paths you can trust before running agents."
created: "2026-04-01"
modified: "2026-04-01"
tags: [decision-guide, self-hosting, steel]
immutable: false
---
## Short answer
Self-hosting Steel Browser is worth the operational cost only when you can name a person who owns Docker, Chrome patching, SSL, and alerting, and when your workflow stays under a handful of concurrent sessions. The [Docker self-hosting guide](https://docs.steel.dev/overview/self-hosting/docker) expects 4 GB of RAM, 10 GB of disk, and open ports 3000/5173/9223 plus a persisted `.cache` volume, so treat it like any other stateful service you must secure and back up.

The moment you need 5+ concurrent jobs, managed stealth, CAPTCHA solving, credentials, files, or region flags, Steel Cloud turns out cheaper than spending nights debugging your own fleet. The official [Steel Local vs Steel Cloud table](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud) caps self-hosted concurrency at roughly one session and leaves out managed proxies, Credentials API, Files API, and multi-region support, so default to Cloud unless compliance or customization forces you to own the stack.

| If this describes you | Self-host the open-source Steel Browser? | Why |
| --- | --- | --- |
| Need strict VPC or on-prem residency, <3 concurrent sessions, and can run Docker | **Yes** | Docker Compose or the single-image run command keep everything on your metal while exposing the Sessions API on port 3000 and the UI on 5173. |
| Need managed proxies, CAPTCHA solving, Credentials or Files APIs, or >5 sessions | **No** | Those surfaces only exist in Steel Cloud today, along with 100+ session concurrency and multi-region flags. |
| Want to patch Chrome builds or preload custom extensions | **Yes** | Self-hosted Steel lets you rebuild the image, change the Chrome version, and drop extensions into `api/src/extensions/` before boot. |
| Need instant observability, replay retention, and steady on-call coverage | **No** | Steel Cloud streams live view and replays automatically, while self-hosting means wiring health checks, storage, and alerting yourself. |

## What self-hosting actually entails
### Baseline requirements
- Reserve at least 4 GB RAM, 10 GB storage, and expose ports 3000 (API), 5173 (UI), and 9223 (Chrome debugging) per the Docker quick start. Persist the `.cache` directory to reuse Chrome profiles or extensions between restarts. ([Docker guide](https://docs.steel.dev/overview/self-hosting/docker))
- Budget for Chrome updates: if you need a different build, you must edit the Dockerfile and rebuild the image yourself before redeploying. ([Docker guide](https://docs.steel.dev/overview/self-hosting/docker))
- Cluster guidance is still TBD; the `clustering` doc is only a placeholder, so horizontal scaling is your responsibility until Steel ships an official recipe. ([Clustering doc](https://docs.steel.dev/overview/self-hosting/clustering))

### Deployment paths you can pick
| Path | Use it when | Notes |
| --- | --- | --- |
| Docker Compose | You want parity with development and separate API/UI containers | `docker compose up -d` launches API + UI, maps ports, and mounts `.cache`, but you must wire TLS and host firewalls yourself. |
| Single Docker image | You prefer one container | `docker run --rm -it -p 3000:3000 -p 9223:9223 ghcr.io/steel-dev/steel-browser:latest` exposes API + UI inside one image; great for CI or throwaway hosts. |
| Railway template | You want a managed PaaS but still own the instance | Railway adds automatic HTTPS, metrics, and scaling knobs; health-check `https://<domain>/v1/health` before handing traffic to it. ([Railway guide](https://docs.steel.dev/overview/self-hosting/railway)) |
| Render | You already run workloads on Render | The Render doc is a link stub only, so treat it like running the Docker image manually on Render rather than expecting a turnkey template. ([Render doc](https://docs.steel.dev/overview/self-hosting/render)) |

After boot, connect the same way you would to [Steel Cloud](@/glossary/steel-cloud.md):

```ts
import { chromium } from "playwright";
import Steel from "steel-sdk";

const steel = new Steel({ baseUrl: process.env.STEEL_BROWSER_URL });
const session = await steel.sessions.create();
const browser = await chromium.connectOverCDP(session.websocketUrl);
```

## Decision factors to vet before committing
| Factor | Questions to ask | Evidence |
| --- | --- | --- |
| Concurrency ceiling | Can you keep work under one or two simultaneous sessions? | Steel Local is documented as a ~1-session runner; Steel Cloud ships 100+ sessions plus lifecycle controls. ([Steel Local vs Steel Cloud](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud)) |
| Trust surfaces | Do you need Credentials API, Files API, managed proxies, or CAPTCHA solving? | None of those exist in Steel Browser today, so you'd build or buy substitutes. ([Steel Local vs Steel Cloud](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud)) |
| Cost of ownership | Who patches Docker and tracks Chrome CVEs? | Docker guide spells out manual rebuild steps, `.cache` persistence, and port exposure, which also means you own hardening. |
| Scaling plan | How will you add hosts or regions later? | There is no published clustering walkthrough yet, so plan your own load balancer, scheduler, and metrics stack. |
| Observability | How will operators replay failures or see live state? | Self-hosting exposes the UI on 5173 and CDP on 9223, but you still need to record replays, export logs, and enforce retention. |

## Operating checklist for self-hosted Steel
1. **Pin the image and config.** Keep explicit image tags and compose files in git so you can revert quickly after a Chrome or Node upgrade. Persist `.cache` and back it up when profiles matter.
2. **Harden the network.** Put reverse proxies in front of ports 3000 and 5173, lock down 9223, and require VPN or firewall rules before letting agents attach.
3. **Monitor health endpoints.** Poll `/v1/health` (Railway exposes this publicly) and alert when startup takes longer than expected or Chrome crashes.
4. **Track session utilization.** Emit metrics for active sessions and queue depth; the minute you hover near 80 percent of capacity, plan a second host or migrate to Steel Cloud.
5. **Automate patching.** Build a recurring job that rebuilds the image with the latest Chrome and Node versions, then redeploys after running smoke tests.
6. **Document handoffs.** Capture the `baseUrl`, auth model, and runbook so the same Playwright or Puppeteer scripts can swap `baseUrl` when you eventually graduate to Steel Cloud.

## Works for / Not yet
- **Works for:** Proof-of-concept agents, compliance-sensitive tests, teams that must inspect or patch Chrome builds, and anyone with a DevOps owner already running Docker services.
- **Not yet:** High-concurrency agent fleets, workflows that depend on managed credentials or files, or teams without 24/7 coverage for browser restarts.

## Trade-offs and limits
- Self-hosted Steel inherits Chrome limits and resource ceilings from your host, so session length, retries, and memory leaks are on you to investigate.
- Persistent storage only exists if you mount `.cache`; lose that volume and your auth context evaporates.
- Railway simplifies TLS and scaling but still leaves credentials, proxy policies, and CMDB integration to your team.
- Clustering and multi-region deployments lack official playbooks right now, so expect to build HA on top of your cloud provider if you need it.

## Next steps
- Follow the [Docker guide](https://docs.steel.dev/overview/self-hosting/docker) or [single-image command](https://docs.steel.dev/overview/self-hosting/docker#single-docker-image-deployment) to stand up a test host, then capture the cost and staffing impact.
- Trial the [Railway deployment](https://docs.steel.dev/overview/self-hosting/railway) if you need a managed PaaS but still want single-tenant browser infrastructure.
- Revisit the [Steel Local vs Steel Cloud matrix](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud) quarterly; as soon as concurrency, proxies, or credential needs expand, point the same SDK calls at Steel Cloud and keep building.

Humans use Chrome. Agents use Steel.
