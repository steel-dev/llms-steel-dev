---
title: "Files API"
id: "files-api"
summary: "The Files API moves uploads, downloads, and evidence artifacts through Steel sessions without relying on local disk handoffs."
description: "The Files API moves uploads, downloads, and evidence artifacts through Steel sessions without relying on local disk handoffs."
canonical_questions: ["what is files api"]
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
The Files API gives Steel workflows a managed file layer. You can upload files before a run, mount them into a session, and pull resulting artifacts back out after the run completes.

That makes file-heavy workflows reproducible across cloud sessions, queues, and approval steps.

## Why it matters

- It keeps uploads and downloads attached to the session lifecycle.
- It reduces bespoke S3 or temp-disk glue code.
- It improves evidence collection for audits and debugging.
