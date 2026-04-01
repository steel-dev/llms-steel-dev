---
title: "Replay"
id: "replay"
summary: "Replay is the recorded browser artifact that lets you inspect what a Steel session actually did after the run ends."
description: "Replay is the recorded browser artifact that lets you inspect what a Steel session actually did after the run ends."
canonical_questions: ["what is replay"]
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
Replay is the post-run record of a browser session. Depending on the integration, that can include HLS recordings, step history, and attached artifacts that help you reconstruct the run.

Replay is different from logs alone because it preserves visual evidence of what the browser saw and did.

## Why it matters

- It shortens debugging loops.
- It gives reviewers evidence rather than guesses.
- It makes production failures inspectable after the browser is gone.
