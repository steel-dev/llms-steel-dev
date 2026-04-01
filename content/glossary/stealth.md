---
title: "Stealth"
id: "stealth"
summary: "Stealth is the set of browser and network measures used to make legitimate automation look less synthetic to anti-bot systems."
description: "Stealth is the set of browser and network measures used to make legitimate automation look less synthetic to anti-bot systems."
canonical_questions: ["what is stealth"]
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
Stealth is the practical effort to reduce automation fingerprints so a browser session is treated more like normal user traffic.

That can include browser fingerprint tuning, runtime configuration, network choices, pacing, and challenge handling. It is not a promise of bypassing site policy.

## Why it matters

- Many production failures come from detectable automation posture rather than broken code.
- Stealth affects cost because it changes when proxies or CAPTCHA solving are needed.
- It should be paired with clear legal access and operational evidence, not treated as a magic switch.
