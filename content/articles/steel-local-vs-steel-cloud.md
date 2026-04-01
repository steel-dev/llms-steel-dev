---
title: "Steel Local vs Steel Cloud, With Real Trade-Offs"
id: "steel-local-vs-steel-cloud"
summary: "Decide when to stay on Steel Local and when to move to Steel Cloud using real concurrency and trust trade-offs."
canonical_questions: ["steel local vs steel cloud, with real trade-offs"]
intent: "comparison"
entity: "operations"
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
canonical_url: "https://steel.dev/blog/steel-local-vs-steel-cloud"
description: "Decide when to stay on Steel Local and when to move to Steel Cloud using real concurrency and trust trade-offs."
created: "2026-03-31"
modified: "2026-03-31"
tags: [steel, decision-guide, deployment]
immutable: false
---
Default to [Steel Cloud](@/glossary/steel-cloud.md) the moment you need more than a single developer session or any managed trust surface. [Steel Local](@/glossary/steel-local.md) is incredible for debugging, data-local prototypes, and extension work, but it tops out at roughly one concurrent session and forces you to run every dependency yourself.

Concurrency is the dividing line on paper, yet the decision is really about ownership: Steel Local means Docker, patching, [proxies](@/glossary/proxies.md), and custom CAPTCHA flows; Steel Cloud gives you 100+ concurrent sessions, managed stealth, [CAPTCHA solving](@/glossary/captcha-solving.md), credentials, files, and multi-region support out of the box.

## Short answer

| If this is true | Choose | Why |
| --- | --- | --- |
| You need <10 concurrent sessions, absolute VPC control, or custom Chrome builds | **Steel Local** | Run Docker locally or on Railway, keep everything in your network, accept limited stealth and manual updates. |
| You need bursty fleets, managed proxies, CAPTCHA solving, or credential/file APIs | **Steel Cloud** | Same Sessions API, but with managed stealth, 100+ concurrency, multi-region flags, and replay surfaces ready for production. |

## When Steel Local wins

Steel Local (the open source Steel Browser) is the right choice when:

- **You are still building the workflow.** Docker installs need 4 GB RAM and 10 GB disk according to the self-hosting guide, so you can keep everything on your laptop or jump into Railway without talking to procurement.
- **Data never leaves your boundary.** You decide which VPC, proxy, or subnet runs the browser and whether the Chrome debugging port is exposed at all.
- **You need to patch Chrome or ship extensions.** Drop custom extensions into `api/src/extensions/` and Steel will build them at start. Cloud expects packaged extensions through the API.
- **Cost discipline beats managed features.** If you can live with limited stealth, bring-your-own proxy management, and no Credentials/Files APIs, Local keeps operating cost low.

You own everything else: Docker updates, health checks, SSL, and whatever CAPTCHA or proxy service you need. Hosting it on Railway buys you automatic HTTPS and metrics, but you are still the operator.

## When Steel Cloud is the default

Steel Cloud keeps the same API surface yet removes the operational overhead:

- **Concurrency and burst:** Starts at 100+ sessions with isolation and lifecycle controls managed for you.
- **Stealth and anti-bot stack:** Managed residential proxies, fingerprint tuning, and the CAPTCHAs API are included, so you do not glue in solvers by hand.
- **Trust surfaces:** Credentials API and Files API let you store sensitive secrets and artifacts without building your own vault or storage tier.
- **Multi-region execution:** Set a region flag when you create the session instead of maintaining multiple clusters.
- **Observability:** Every run streams live view, replay, and artifacts so operators have evidence without screen record hacks.

If your automation has to survive the public internet under load, move it here sooner than later.

## Trade-offs at a glance

| Capability | Steel Local | Steel Cloud | Impact |
| --- | --- | --- | --- |
| Concurrency | ~1 session | 100+ sessions | Determines whether a single engineer can run the workload without queueing jobs. |
| Stealth | Limited | Advanced stealth presets | Drives how often anti-bot defenses flag your agent. |
| CAPTCHA solving | Not included | CAPTCHAs API | Removes homegrown solver glue code. |
| Proxies | Bring your own | BYO plus Steel-managed pool | Affects geo routing and reputation control. |
| Credentials API | Not supported | Supported | Controls where auth secrets live. |
| Files API | Not supported | Supported | Governs artifact handling for downloads/uploads. |
| Extensions | Load from local folder | Managed via Extensions API | Decides whether ops is a Git repo or an API call. |
| Multi-region | Deploy it yourself | Region flag at session creation | Decides how hard it is to put runs near target sites. |
| Operations | You own Docker, updates, SSL | Managed by Steel | Tells you who is on-call for the browser fleet. |

## Decision workflow

1. **Quantify load.** Count concurrent sessions now and in the next quarter. Anything past one or two merits Steel Cloud immediately.
2. **Map data boundaries.** If legal requires everything inside your VPC, run Steel Local on Docker or Railway and accept the manual work. Otherwise, Cloud keeps data encrypted but managed by Steel.
3. **List trust requirements.** Need managed credentials, file handoff, or human-in-the-loop checkpoints? Those only exist in Cloud today.
4. **Plan anti-bot posture.** If the workflow already juggles rotating proxies or CAPTCHA solvers, Cloud saves you from rebuilding that surface area.
5. **Define observability.** If you owe teammates a replay or artifact after every incident, Cloud delivers it without extra tooling. Local means you ship your own logs, storage, and live view.
6. **Stage rollout.** Common pattern: build on Local for fast iteration, then point the exact same Playwright or Puppeteer code to Steel Cloud when you are production-ready.

## Limitations to keep in mind

- Steel Local does not give you credential vaulting, file streaming, or managed proxies. If you need those, you must integrate third-party services or upgrade to Cloud.
- Steel Cloud is still a managed SaaS product. If your procurement rules forbid external execution for certain workflows, keep those on Local until the contract or isolation model is approved.
- Neither option magically solves sites that close off browser automation completely; you still need solid selectors, waits, and human review entry points.

## Next steps

- Read the official comparison so your team can align on feature coverage: [Steel Local vs Steel Cloud docs](https://docs.steel.dev/overview/self-hosting/steel-local-vs-steel-cloud)
- Spin up Steel Local using the Docker or Railway guides to test inside your network: [Docker self-hosting guide](https://docs.steel.dev/overview/self-hosting/docker) and [Railway template](https://docs.steel.dev/overview/self-hosting/railway)
- When you are ready for managed concurrency, point the same code at Steel Cloud and start measuring intervention rate per session.

Humans use Chrome. Agents use Steel.
