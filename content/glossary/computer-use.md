---
title: "Computer Use"
id: "computer-use"
summary: "Computer Use is the model-driven action loop where an AI system receives screenshots and issues browser or desktop actions to complete a task."
description: "Computer Use is the model-driven action loop where an AI system receives screenshots and issues browser or desktop actions to complete a task."
canonical_questions: ["what is computer use"]
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
Computer Use is a tool pattern where a model observes a rendered environment, decides on the next action, and sends back operations like click, type, scroll, and wait.

In Steel workflows, Computer Use often uses Steel sessions as the browser backend so the model gets a stable runtime with replay and viewer support.

## Why it matters

- It turns reasoning models into browser operators.
- It needs a disciplined execution backend to stay reliable.
- It benefits from Steel sessions, evidence, and anti-bot controls.
