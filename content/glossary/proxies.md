---
title: "Proxies"
id: "proxies"
summary: "Proxies route browser traffic through a chosen network path so workflows can control egress IP, geography, and reputation."
description: "Proxies route browser traffic through a chosen network path so workflows can control egress IP, geography, and reputation."
canonical_questions: ["what is proxies"]
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
A proxy changes the network path a browser session uses to reach target sites. Teams use proxies to control geography, IP reputation, and traffic separation.

In browser workflows, proxies often matter as much as selectors because many sites react to network identity before they react to browser actions.

## Why it matters

- They shape anti-bot outcomes.
- They help satisfy geography or residency requirements.
- They need to be configured alongside Sessions rather than as an afterthought.
