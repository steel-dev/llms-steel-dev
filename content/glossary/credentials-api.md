---
title: "Credentials API"
id: "credentials-api"
summary: "The Credentials API stores secrets server-side and injects them into Steel sessions without exposing raw values to the model or operator."
description: "The Credentials API stores secrets server-side and injects them into Steel sessions without exposing raw values to the model or operator."
canonical_questions: ["what is credentials api"]
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
The Credentials API is Steel's secret-injection surface for browser workflows. Instead of handing passwords to prompts or scripts, you store credentials once and let Steel inject them into the page at runtime.

That keeps raw secrets out of logs, transcripts, and most human review paths.

## Why it matters

- It reduces secret sprawl in prompts and code.
- It supports namespacing and controlled injection into sessions.
- It pairs naturally with Profiles and session-level approvals.
