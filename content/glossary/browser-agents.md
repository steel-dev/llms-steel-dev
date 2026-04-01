---
title: "Browser Agents"
id: "browser-agents"
summary: "Browser agents are AI or rules-based systems that complete tasks by operating a real web browser instead of calling a direct API."
description: "Browser agents are AI or rules-based systems that complete tasks by operating a real web browser instead of calling a direct API."
canonical_questions: ["what is browser agents"]
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
Browser agents are automation systems that read pages, click controls, fill forms, upload files, and navigate workflows through a browser runtime.

They matter when the real system of record lives behind web UI, authentication, anti-bot checks, or approval steps that an API does not expose.

## Why it matters

- Browser agents can interact with the same workflows humans use.
- They need stronger state, evidence, and recovery controls than simple API clients.
- In Steel, browser agents usually run on top of Sessions, Profiles, Credentials, and Files.
