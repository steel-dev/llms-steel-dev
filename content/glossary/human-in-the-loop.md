---
title: "Human-in-the-Loop"
id: "human-in-the-loop"
summary: "Human-in-the-loop means a workflow pauses for a person to review, approve, or take over before the browser session continues."
description: "Human-in-the-loop means a workflow pauses for a person to review, approve, or take over before the browser session continues."
canonical_questions: ["what is human in the loop"]
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
Human-in-the-loop is the control pattern where an automated browser run waits for a person at a risky or ambiguous step.

In Steel workflows, the key point is preserving the live session so the reviewer can see or continue the same browser state instead of replaying the run from scratch.

## Why it matters

- It keeps approvals inside the real workflow instead of outside it.
- It reduces rework when sign-off, MFA, or judgment calls are required.
- It creates a clearer audit trail for who intervened and when.
