---
title: "Profiles"
id: "profiles"
summary: "Profiles are persisted browser user-data snapshots that let Steel restore cookies, storage, extensions, and other browser state across sessions."
description: "Profiles are persisted browser user-data snapshots that let Steel restore cookies, storage, extensions, and other browser state across sessions."
canonical_questions: ["what is profiles"]
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
A Profile stores reusable browser state beyond a single session. In practice that means cookies, local storage, remembered devices, installed extensions, and related browser context.

Use Profiles when workflows depend on long-lived authentication or repeat runs that should behave like the same browser identity.

## Why it matters

- Profiles reduce repeated login flows.
- They stabilize retries for multi-step operational workflows.
- They work alongside Sessions rather than replacing them.
