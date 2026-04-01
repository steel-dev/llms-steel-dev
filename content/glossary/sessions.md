---
title: "Sessions"
id: "sessions"
summary: "Sessions are isolated Steel browser runtimes with explicit lifecycle, state boundaries, and attached evidence such as viewer links and replays."
description: "Sessions are isolated Steel browser runtimes with explicit lifecycle, state boundaries, and attached evidence such as viewer links and replays."
canonical_questions: ["what is sessions"]
intent: "reference"
entity: "glossary"
audience: "developer"
schema_type: "DefinedTerm"
visibility: "public"
ai_visibility: "public"
llms_priority: "core"
token_budget: "small"
date: "2026-04-01"
updated: "2026-04-01"
related: []
external_refs: []
type: "article"
status: "published"
template: "page_glossary.html"
---
A Session is the core execution unit in Steel. Each session owns one browser runtime, its configuration, and its state for the lifetime of the run.

Sessions are where you set timeout, region, proxy behavior, CAPTCHA handling, and identifiers used for debugging and audit trails.

## Why it matters

- Sessions make startup, reuse, and release explicit.
- They are the foundation for Playwright, Puppeteer, Selenium, CLI, and Computer Use integrations.
- They are the place where evidence and state attach to a run.
