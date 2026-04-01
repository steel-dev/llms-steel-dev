---
title: "Session Lifecycle"
id: "session-lifecycle"
summary: "Session lifecycle is the sequence of creating, connecting to, operating, observing, and releasing a browser session."
description: "Session lifecycle is the sequence of creating, connecting to, operating, observing, and releasing a browser session."
canonical_questions: ["what is session lifecycle"]
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
Session lifecycle is the operational flow around a browser session: create it, connect your framework or agent, run the work, capture evidence, and release it cleanly.

The concept matters because browser reliability problems often come from lifecycle mistakes, not from selectors alone.

## Why it matters

- Lifecycle discipline determines cost, cleanup, and concurrency recovery.
- It is where evidence, approvals, profiles, and files attach to the run.
- Steel is opinionated about lifecycle so teams do not leave orphaned sessions or missing artifacts behind.
