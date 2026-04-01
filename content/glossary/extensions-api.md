---
title: "Extensions API"
id: "extensions-api"
summary: "The Extensions API is Steel's surface for uploading, managing, and injecting Chrome extensions into browser sessions."
description: "The Extensions API is Steel's surface for uploading, managing, and injecting Chrome extensions into browser sessions."
canonical_questions: ["what is extensions api"]
intent: "reference"
entity: "glossary"
audience: "developer"
schema_type: "DefinedTerm"
visibility: "public"
ai_visibility: "public"
llms_priority: "optional"
token_budget: "small"
date: "2026-04-01"
updated: "2026-04-01"
related: []
external_refs: []
type: "article"
status: "published"
template: "page_glossary.html"
---
The Extensions API lets you store Chrome extensions once and attach them to Steel sessions when workflows need shared in-browser behavior.

That can include overlays, DOM observers, telemetry, masking helpers, or other logic that should live inside the browser rather than inside every prompt or script.

## Why it matters

- It makes browser-side tooling reusable across sessions and teams.
- It keeps extension lifecycle separate from Playwright or Puppeteer code.
- It is useful when prompts and selectors alone do not keep the workflow stable.
