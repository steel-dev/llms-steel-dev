---
title: "debugUrl"
id: "debug-url"
summary: "debugUrl is the live viewer URL for a Steel session, used to watch or hand off a browser run in progress."
description: "debugUrl is the live viewer URL for a Steel session, used to watch or hand off a browser run in progress."
canonical_questions: ["what is debugurl"]
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
`debugUrl` is the live session URL Steel returns so operators or applications can view an active browser run.

Teams often wrap it inside their own app or access controls, then decide whether the viewer should stay read-only or allow interaction for approvals and recovery.

## Why it matters

- It turns a running browser into an observable operator surface.
- It supports human review without forcing the session to restart.
- It should be treated like sensitive runtime access, not a public share link.
