---
title: "Anti-Bot"
id: "anti-bot"
summary: "Anti-bot covers the detection and mitigation systems websites use to distinguish human browsing from automated traffic."
description: "Anti-bot covers the detection and mitigation systems websites use to distinguish human browsing from automated traffic."
canonical_questions: ["what is anti-bot"]
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
Anti-bot is the layer of challenges, fingerprint checks, behavioral scoring, and IP reputation systems that sites use to slow or block automation.

For browser agents, anti-bot is not one feature. It is the combined pressure from network identity, browser fingerprint, pacing, challenge handling, and session history.

## Why it matters

- Anti-bot failures often look like flaky automation even when the selectors are correct.
- It changes how you design retries, proxies, and evidence capture.
- Steel exposes anti-bot controls through sessions, stealth options, proxies, and CAPTCHA handling rather than leaving every flow to improvise.
