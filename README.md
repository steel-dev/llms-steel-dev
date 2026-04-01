# Answers Steel Dev

This project was scaffolded by `ansorum init`.

It includes:

- answer-first starter content in `content/`
- a JSON-LD sidecar example at `content/refunds.schema.json`
- a curated pack definition in `collections/packs/billing.toml`
- deterministic eval fixtures in `eval/fixtures.yaml`

Run the full workflow:

```bash
ansorum build
ansorum serve
ansorum audit
ansorum eval
```

Use `ansorum eval --llm` only when `OPENAI_API_KEY` is set and you want OpenAI
Responses API rubric scoring.
