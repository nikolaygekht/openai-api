# openai-api — a Claude Code skill

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to write **OpenAI API** code correctly across multiple languages, targeting OpenAI's current primary endpoint — the **Responses API** (`POST /v1/responses`).

Out of the box, Claude tends to reach for `chat.completions.create()` and `gpt-4o`-class models. Those still work, but they're not what OpenAI recommends for new code as of May 2026. This skill closes that gap.

> **Status: development.** Iteration-1 evals show **96 % pass rate with the skill vs. 60 % baseline** across 8 representative tasks (Python chat, Next.js streaming, cost optimization, tool use, C# reasoning, audio transcription, structured extraction, migration from Chat Completions). The skill is functional and useful today; iteration 2 (incorporating user review feedback) is pending.

## What this skill teaches Claude

- **Use the Responses API.** `client.responses.create()` / `client.responses.stream()` / `client.responses.parse()` — not `chat.completions`.
- **Pick the right model.** `gpt-5.2` is the default for new code (matches the official SDK README). The skill documents when to switch to `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.4-nano`, `gpt-5.2-codex`, or `o3` / `o4-mini`.
- **Handle secrets correctly.** Read `OPENAI_API_KEY` from env. Never hardcode, never echo, never commit. `.env` gitignored, `.env.example` committed.
- **Use the right shapes.** Tools are flat (`{type, name, description, parameters, strict}` — no `function: {...}` envelope). Structured outputs go under `text.format` (flat, no `json_schema:` wrapper).
- **Get prompt caching right.** OpenAI's caching is automatic but prefix-fragile — `usage.input_tokens_details.cached_tokens` (not `prompt_tokens_details`), stable system prompts at module scope, dynamic content trailing the input, `prompt_cache_key` for multi-tenant routing. ~90% discount on gpt-5.x cached input.
- **Stream when it matters.** `responses.stream()` helper or `stream=true` raw iteration; full event taxonomy documented.
- **Function calling and built-in tools.** Function tools with `strict: true`, plus the hosted built-ins (`web_search`, `file_search`, `code_interpreter`, `image_generation`, `computer`, `mcp`, `apply_patch`).
- **Reasoning models.** `reasoning.effort` and `reasoning.summary`, encrypted reasoning for stateless mode, `output_tokens_details.reasoning_tokens` accounting.
- **Audio.** `gpt-4o-transcribe` + `gpt-4o-mini-tts` (steerable via `instructions`), the 25 MB byte limit, voices including `marin` and `cedar`.
- **Migrate from Chat Completions.** Step-by-step field mapping table.

## Languages covered

| Language       | SDK / install                          | Coverage                                            |
| -------------- | -------------------------------------- | --------------------------------------------------- |
| **Python**     | `pip install openai` (≥ 2.37)          | Full: README, streaming, tool-use, audio            |
| **TypeScript** | `npm install openai` (≥ 6.38)          | Full: README, streaming, tool-use, audio            |
| **C# / .NET**  | `dotnet add package OpenAI` (≥ 2.10)   | Single-file reference; experimental Responses flag  |
| **Java**       | `com.openai:openai-java:4.36`          | Single-file reference                               |
| **cURL / HTTP**| n/a                                    | Raw-HTTP examples for languages without SDKs        |

## Skill layout

```
openai-api/
├── SKILL.md                    — entry point: triggering, defaults, language detection, reading guide
├── shared/                     — language-agnostic concepts
│   ├── models.md               — full model catalog + pricing (May 2026)
│   ├── prompt-caching.md       — caching architecture, invalidators, multi-tenant routing
│   ├── structured-outputs.md   — text.format flat shape, strict-mode subset, refusals
│   ├── tool-use-concepts.md    — function tools + built-in tools (web/file/code/MCP/etc.)
│   ├── error-codes.md          — HTTP status table, retry guidance, headers
│   ├── secrets.md              — .env / .gitignore / per-language secret patterns + CI
│   └── live-sources.md         — WebSearch + GitHub URLs (platform.openai.com blocks WebFetch)
├── python/openai-api/          — Python deep coverage
├── typescript/openai-api/      — Node / TypeScript deep coverage
├── csharp/openai-api.md
├── java/openai-api.md
├── curl/examples.md
└── evals/evals.json            — the test prompts that drive skill iteration
```

About 5,500 lines of skill content across 19 files.

## Scope and non-scope

**In scope:**
- The Responses API and everything it covers (text, vision, tools, structured outputs, reasoning, conversation state, streaming).
- The audio endpoints (`/v1/audio/transcriptions`, `/v1/audio/translations`, `/v1/audio/speech`).
- Prompt caching strategy across Responses calls.
- Cross-SDK error handling.

**Out of scope** (by design — keeps the skill focused):
- Legacy Chat Completions (mentioned only in the migration guide).
- Embeddings (`/v1/embeddings`) — covered briefly in `shared/models.md` for context.
- Batch API.
- Assistants API.
- Realtime API (voice agents).
- Fine-tuning.

## Installation

See **[INSTALL.md](./INSTALL.md)** for global and project-scoped install instructions.

## Repository layout

This repo is the **development source** for the skill, not an installation. The skill itself is the `openai-api/` directory; everything else is dev / test infrastructure.

```
openai-api-skill/
├── README.md          — this file
├── INSTALL.md         — installation guide for Claude Code
├── CLAUDE.md          — guidance for Claude Code when working inside this repo
├── LICENSE
├── .gitignore         — excludes .env/, test-output/, .claude/settings.local.json
├── openai-api/        — THE SKILL (shipped artifact)
└── test-output/       — eval workspaces and ad-hoc experiments (gitignored)
    ├── _research/     — research briefs that informed the skill
    ├── openai-api-workspace/iteration-1/   — paired with-skill / baseline eval runs
    ├── test01/        — carport drawing dimension extraction (PDF + Python script + results)
    └── test02/        — door drawing vs cutlist cross-validation (PDFs + script + results.md)
```

## Evidence the skill works

`test-output/openai-api-workspace/iteration-1/`:

- **8 paired eval runs** (with-skill vs no-skill baseline) across Python / TypeScript / C# / cost optimization / audio / structured outputs / migration scenarios.
- **96 % pass rate with the skill vs 60 % baseline.** Biggest gaps were in test cases where the baseline used `chat.completions`, legacy SDK helpers, or hardcoded keys — exactly the things the skill teaches Claude to avoid.
- See `review.html` in that folder for the visual side-by-side review.

`test-output/test02/results.md` shows an applied use case: cross-validating an engineering drawing (6 pages) against its cutlist (4 pages) using three Responses-API calls with vision + structured outputs. Three calls, ≈$0.25 total, full cross-check report with severity-tagged findings.

## Acknowledgements

Built with the [skill-creator](https://github.com/anthropics/claude-code) workflow from Anthropic. Structure inspired by the official `claude-api` skill.

## License

See [LICENSE](./LICENSE). The skill content is authored from publicly available OpenAI documentation and the official SDK source on GitHub.
