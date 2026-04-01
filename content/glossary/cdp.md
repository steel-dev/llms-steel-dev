---
title: "CDP"
id: "cdp"
summary: "CDP, the Chrome DevTools Protocol, is the control channel many frameworks use to attach to and drive a Chromium browser instance."
description: "CDP, the Chrome DevTools Protocol, is the control channel many frameworks use to attach to and drive a Chromium browser instance."
canonical_questions: ["what is cdp"]
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
CDP stands for Chrome DevTools Protocol. It is the protocol used to inspect and control Chromium-based browsers for automation, debugging, and tooling.

Many Steel integrations expose a CDP endpoint so Playwright, Puppeteer, and related tools can attach to the managed browser session you created.

## Why it matters

- It lets existing browser frameworks connect to Steel without a rewrite.
- It is often the transport behind managed browser integrations.
- It is different from your app logic; it is the control plane into the browser.
