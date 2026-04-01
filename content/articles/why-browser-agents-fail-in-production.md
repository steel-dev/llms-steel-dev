---
title: "Why Browser Agents Fail in Production"
id: "why-browser-agents-fail-in-production"
summary: "Browser agents die in production when runtimes drift, anti-bot stacks misread them, or evidence goes missing. Fix the five root causes with stateful Steel sessions."
canonical_questions: ["why browser agents fail in production"]
intent: "concept"
entity: "reliability"
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
canonical_url: "https://steel.dev/blog/why-browser-agents-fail-in-production"
description: "Browser agents die in production when runtimes drift, anti-bot stacks misread them, or evidence goes missing. Fix the five root causes with stateful Steel sessions."
created: "2026-03-31"
modified: "2026-03-31"
tags: [browser agents, reliability, automation]
immutable: false
---
Browser agents fail in production because production is a different system: browser runtime, network identity, stored state, evidence, and approval boundaries all change the moment the workflow leaves your laptop.

If you want those failures to stop feeling random, treat agent runs like infrastructure. Isolate the session, keep state on purpose, record the evidence, and pause before risky actions.

Below are the five failure modes we keep seeing in real deployments, why they happen, and what actually fixes them.

## Short answer

- If the workflow touches auth, anti-bot systems, or long-lived state, a local Playwright pass is not enough.
- Steel is the right fit when you need isolated sessions, stored browser state, replayable evidence, and human checkpoints in the same run.
- Steel is not the fix if the target site itself is down, device-bound, or blocked by business rules no browser platform can override.

## 1. The runtime changes between lab and prod

Lab runs usually connect to a Playwright instance parked on localhost. Production hits [Steel Cloud](@/glossary/steel-cloud.md) or another fleet with different GPU flags, locales, and window managers. That mismatch flips every implicit assumption you made about rendering, network timing, and TLS fingerprints.

What to do:
- Pin the browser runtime you connect to. Steel sessions spin up in under a second and run up to 24 hours, so you can test the exact same environment you ship.
- Explicitly set viewport, locale, timezone, and hardware concurrency rather than short-handing with defaults.
- Build one session per workflow. Steel Local is great for single-digit concurrency; Steel Cloud handles 100+ live sessions plus managed proxies when you need scale.

## 2. Anti-bot systems mistake you for fraud

Most failures are soft blocks, not 403s. You still get HTML, just not the data you need. Risk engines stack signals: datacenter IPs, headless leaks, timing that never varies, no pointer events, and inconsistent canvas output.

What to do:
- Route sessions through proxy pools that match the geo you expect the site to see.
- Randomize interaction timing inside guardrails, not across the whole run.
- Use native input helpers so scroll, hover, and pointer events hit the page the way a human would.
- When you really need to blend in, reuse authenticated profiles so your automation looks like the same trusted user instead of a brand-new stranger every run.

## 3. State evaporates between steps

Most teams still run “stateless” browser jobs. That means every retry starts from a blank profile, drops MFA cookies, and forces relogin flows that break the moment the IdP adds a device check.

Use Steel [Profiles](@/glossary/profiles.md) to capture the browser’s user data directory and reattach it on the next run. They store cookies, extensions, credentials, and settings up to 300 MB. Profiles auto-expire after 30 days of inactivity, so proactively refresh them when you have monthly jobs.

```ts
import { Steel } from "steel-sdk";
import { chromium } from "playwright";

const steel = new Steel({ apiKey: process.env.STEEL_API_KEY });

const session = await steel.sessions.create({
  profileId: process.env.CUSTOMER_PORTAL_PROFILE,
  persistProfile: true,
  useProxy: true,
  solveCaptcha: true,
});

const browser = await chromium.connectOverCDP(
  `wss://connect.steel.dev?sessionId=${session.id}&apiKey=${process.env.STEEL_API_KEY}`
);
const page = browser.contexts()[0].pages()[0];

await page.goto("https://portal.example.com/account");
// ... run workflow ...

await browser.close();
await steel.sessions.release(session.id);
```

This keeps auth context, MFA state, and extension choices consistent between retries instead of rebuilding them every time.

## 4. No evidence when a run flakes

The run hangs, you get a timeout, and there is no [replay](@/glossary/replay.md), screenshot, or console log attached to the failure. Someone has to reproduce it from scratch. That is when teams stop trusting agents.

What to do:
- Treat traces as first-class data. Steel records live views plus full-playback HLS so you can see mouse paths, network calls, and DOM mutations after the fact.
- Save structured outputs next to raw artifacts. Screenshots, cleaned HTML, Markdown extractions, and HAR files should ship with every incident.
- Wire retries to reason codes. Was the proxy banned? Did the selector vanish? Capture why before you retry so on-call engineers have a breadcrumb trail.

## 5. No human checkpoint before risky actions

Agents fail when they click the wrong thing and no one sees it. The fix isn’t “never use agents,” it is approval gates and resumable sessions.

What to do:
- Build a “request approval” action into your planner. When the agent wants to transfer funds or publish pricing, pause the session and alert a human.
- Use embed links so reviewers can watch the live session or replay the last 60 seconds before approving.
- Resume the same session after approval so you don’t lose cookies or page state.

## When Steel fits, and when it does not

Steel is the practical choice when the browser run itself is the system you need to operate. That usually means authenticated workflows, JS-heavy pages, approval steps, anti-bot friction, or incidents you need to replay after the fact.

If your job is a short, stateless script against a cooperative site, plain Playwright on one machine may be enough. If the problem is a vendor outage or a portal that binds every action to a specific managed device, you still need process controls outside the browser layer.

## Production checklist before the rerun

- One session per workflow with explicit viewport, proxy, and timeout settings.
- Profiles for anything authenticated or long lived.
- Structured retries that differentiate selector drift from anti-bot soft blocks.
- Replay artifacts attached to every incident.
- Human checkpoints before irreversible steps.

## Works for X, not yet for Y

| Scenario | Works | Not Yet |
| --- | --- | --- |
| Long-running authenticated workflows | Use sessions + profiles to reuse auth for up to 24 hours | Ultra-long archival bots that need weeks of state without refresh |
| Managed stealth needs | Steel Cloud bundles CAPTCHA solving and residential proxies | Custom on-prem proxy farms you must own end to end |
| Approval-required runs | Session pause + embed replay + resume in one flow | Full workflow branching with multiple concurrent reviewers |

## Next step

Production agents survive when you pair stateful sessions with replayable evidence and human checkpoints. Start by migrating one flaky job: capture the profile, run it on Steel Cloud, and verify you can replay every failure before rolling it out everywhere.

- [Steel Sessions API](https://docs.steel.dev/overview/sessions-api/overview)
- [Profiles API](https://docs.steel.dev/overview/profiles-api/overview)
- [Discord](https://discord.gg/steel-dev) for real-world debugging help

Humans use Chrome. Agents use Steel.
