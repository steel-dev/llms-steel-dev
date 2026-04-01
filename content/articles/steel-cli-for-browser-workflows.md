---
title: "Steel CLI for Browser Workflows"
id: "steel-cli-for-browser-workflows"
summary: "Steel CLI keeps browser workflows fast by handling session lifecycle, passthrough commands, and agent skills so you debug less code and ship reliable runs."
canonical_questions: ["steel cli for browser workflows"]
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
canonical_url: "https://steel.dev/blog/steel-cli-for-browser-workflows"
description: "Steel CLI keeps browser workflows fast by handling session lifecycle, passthrough commands, and agent skills so you debug less code and ship reliable runs."
created: "2026-03-31"
modified: "2026-03-31"
tags: [steel-cli, browser-automation, implementation-guide]
immutable: false
---
## Short answer
Steel CLI replaces the hand-rolled scripts that slow [browser automation](@/glossary/browser-automation.md) down. It boots a named session in under a second on [Steel Cloud](@/glossary/steel-cloud.md), keeps that session permissioned, and forwards every familiar `agent-browser` command through one terminal contract, so you spend time finishing the workflow instead of stitching orchestration code.

If you already have Playwright or `agent-browser` muscle memory, the CLI adds the missing lifecycle: start, live-view, CAPTCHA solve, artifact capture, and stop, all behind `steel browser <command>`. Run it locally, against your own Steel Browser deployment, or in the managed cloud by flipping a single flag.

### Where the CLI actually speeds things up
| Bottleneck | Manual slog | Steel CLI move |
| --- | --- | --- |
| Session lifecycle | Write scripts around CDP to name, reuse, and tear down contexts | `steel browser start --session run-42` and `steel browser stop` handle lifecycle for you |
| Proof + debugging | Capture screenshots or HLS manually | `steel browser snapshot -i`, `steel screenshot`, and `steel browser live` stream the state instantly |
| CAPTCHA / stealth | Bolt third-party solvers onto every run | `steel browser start --stealth` or `--session-solve-captcha` keeps solving wired in |
| Endpoint juggling | Rebuild configs per environment | Switch between `--local`, `--api-url`, or default cloud without changing scripts |
| Agent instructions | Prompt glue for every project | Install the `steel-browser` skill so agents follow the same start -> work -> stop contract |

## Instead of patching wrappers, treat the CLI as the workflow contract
The CLI already knows how to call Steel's session API, sanitize live-view URLs, and keep stateful runs boring. That is what makes browser work move faster. Instead of writing brittle glue around Playwright or Puppeteer, teach your agents (or your shell scripts) to call `steel browser` commands directly. Every command is traceable, replayable, and consistent with the docs. The `steel-browser` skill ships with guardrails like "always run `snapshot -i` before you click" so large language models stick to the safe path without ad hoc prompts.

## Implementation path
1. **Install and log in**
   ```bash
   npm i -g @steel-dev/cli
   steel login
   ```
2. **Pick the right endpoint**
   - Default Cloud: do nothing. Steel Cloud sessions start in under 1 second and inherit managed stealth.
   - Local dev: `steel dev install && steel dev start`, then add `--local` when starting sessions.
   - Self-hosted: pass `--api-url https://steel.your-domain.dev/v1` so the CLI talks to your Steel Browser cluster.
3. **Start a named session**
   ```bash
   SESSION="workflow-$(date +%s)"
   steel browser start --session "$SESSION" --stealth
   steel browser live --session "$SESSION"   # grab the inspect link
   ```
4. **Drive the page with passthrough commands** (identical to `agent-browser` syntax):
   ```bash
   steel browser open https://example.com/checkout --session "$SESSION"
   steel browser snapshot -i --session "$SESSION"
   steel browser fill @e1 "Jane Doe" --session "$SESSION"
   steel browser click @e5 --session "$SESSION"
   steel browser wait --load networkidle --session "$SESSION"
   ```
5. **Collect artifacts and stop cleanly**
   ```bash
   steel screenshot https://example.com/checkout --full-page
   steel browser stop --session "$SESSION"
   ```
6. **Automate the ceremony for agents**
   - Install the `steel-browser` skill: `npx skills add steel-dev/cli --skill steel-browser`.
   - Restart your agent client so it discovers the skill manifest and enforces the same contract automatically.

## Patterns that make runs faster
| Signal | CLI move | Why it matters |
| --- | --- | --- |
| Need parity across clouds, self-hosting, and local | Use `--api-url`, `--local`, or environment overrides | One script handles every deployment target without copy-pasted config |
| Agent loops keep flaking mid-action | Force named sessions plus `snapshot -i` before actions via the skill | Agents stop hallucinating selectors and you get visual evidence per step |
| Human review blocks progress | Use `steel browser live` during the run, then share the `connect_url` for approvals | Reviewers can jump in without stopping the automation |
| You keep rebuilding starter repos | `steel forge playwright --name intake-bot` and `steel run browser-use --task ...` scaffold the workflow in minutes |
| Credential walls slow tests | Pair CLI sessions with the Credentials API (when you move to Cloud) so secrets never live in prompts |

## Works for X, not yet for Y
| Works when... | Still limited when... |
| --- | --- |
| You need deterministic browser control with evidence for every step | You expect DOM-level hooks beyond what `agent-browser` exposes; CLI mirrors that runtime |
| You want to run the same script against Steel Cloud, Steel Browser, or localhost | You need native mobile gestures beyond viewport emulation; use Steel mobile mode separately |
| You need CAPTCHAs or stealth proxies handled inside the workflow | You require bespoke proxy pools or anti-bot tricks not yet surfaced as flags |
| You want agent skills to follow a contract instead of bespoke prompts | You expect skills to manage secrets; pair CLI with Credentials API for storage |

## Next steps
- Skim the [Steel CLI docs](https://docs.steel.dev/overview/steel-cli) for every command and flag.
- Install the `steel-browser` skill and force your agent environment to restart so it respects the lifecycle contract.
- Ship one workflow this week: run `steel browser start`, capture evidence with `snapshot -i`, stop the session, and compare how much glue code disappeared.

Humans use Chrome. Agents use Steel.
