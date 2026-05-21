# openai-api-skill — Development Repo

This repo is the **development source** for a Claude Code skill named **`openai-api`** that teaches Claude how to write OpenAI API code correctly (Responses API, audio, streaming, tools, structured outputs, prompt caching) across Python, TypeScript, C#, Java, and raw cURL.

The skill itself lives under `openai-api/`. It is consumed elsewhere via the packaged `.skill` artifact — it is **not installed into `~/.claude/skills/` from this repo**. End-users install from the `.skill` package produced by `scripts/package_skill.py` (in `skill-creator/`).

---

## Layout

```
openai-api-skill/
├── CLAUDE.md                    — this file
├── LICENSE
├── .gitignore                   — excludes .env/, test-output/, .claude/settings.local.json
├── .env/                        — local secrets (gitignored). Contains the openai key used for ad-hoc testing
│
├── openai-api/                  — THE SKILL. This is what gets packaged and shipped.
│   ├── SKILL.md                 — entry point; triggering, defaults, language detection, reading guide
│   ├── shared/                  — language-agnostic concepts
│   │   ├── models.md
│   │   ├── prompt-caching.md
│   │   ├── structured-outputs.md
│   │   ├── tool-use-concepts.md
│   │   ├── error-codes.md
│   │   ├── secrets.md
│   │   └── live-sources.md
│   ├── python/openai-api/       — Python SDK reference (README, streaming, tool-use, audio)
│   ├── typescript/openai-api/   — Node / TypeScript SDK reference (same four)
│   ├── csharp/openai-api.md     — single-file C# / .NET reference
│   ├── java/openai-api.md       — single-file Java reference
│   ├── curl/examples.md         — raw HTTP for unsupported languages
│   └── evals/evals.json         — test prompts driving the skill's evals
│
└── test-output/                 — ALL generated test / eval / research output (gitignored)
    ├── _research/               — research briefs from initial scoping
    └── openai-api-workspace/    — eval runs
        └── iteration-1/
            ├── eval-NN-name/{with_skill,without_skill}/run-1/
            ├── benchmark.json
            ├── benchmark.md
            ├── review.html      — open in a browser to grade outputs
            ├── grade.py         — re-runs the assertion grader
            └── regrade.py       — one-shot migration from the original layout
```

---

## Rule: All test / eval / experiment output goes in `test-output/`

**Any time you produce intermediate artifacts** — research briefs, eval workspaces, throw-away scripts, generated code we're using to inspect skill behavior, screenshots of the eval viewer, anything that is NOT the skill itself — they go under `test-output/`. Never at the repo root, never inside `openai-api/`.

`test-output/` is gitignored. Treat its contents as scratch space.

**Why:** Keeps the deliverable (`openai-api/`) clean and obvious; keeps experiments out of version control; lets us blow away `test-output/` and rerun from scratch when needed.

**How to apply:**
- When spawning eval subagents, set their output paths to `test-output/openai-api-workspace/iteration-N/...`.
- When writing research briefs, scratch experiments, or one-off analysis scripts, save under `test-output/<descriptive-subfolder>/`.
- When packaging the skill (`python3 -m scripts.package_skill openai-api`), the resulting `.skill` file is OK to put at the repo root briefly, but consider moving it to `test-output/` if you're iterating.
- The skill's own evals (`openai-api/evals/evals.json`) stay inside the skill — they're part of what we ship. Only the *run results* go to `test-output/`.

---

## Secret handling (`.env/open-ai`)

The local OpenAI API key for ad-hoc testing lives at `.env/open-ai`. The `.env/` directory is gitignored. **Never** print, log, echo, or commit that value. **Never** paste a key on the command line; load it via env var only.

The skill itself enforces the same discipline for the code it teaches Claude to write — see `openai-api/shared/secrets.md` and the "Secrets & API Keys" section of `openai-api/SKILL.md`.

---

## Developing the skill

The high-level loop:

1. **Edit files under `openai-api/`** — primarily `SKILL.md` and the language-specific files.
2. **Run the evals** to see how Claude does with vs. without the skill:
   - Spawn paired subagents (with-skill + baseline) per eval. Save outputs to `test-output/openai-api-workspace/iteration-N/eval-NN-name/{with_skill,without_skill}/run-1/outputs/`.
   - Capture timing into `timing.json` per run dir.
   - Grade with `test-output/openai-api-workspace/iteration-N/grade.py` (or write a new grader for iteration N).
   - Aggregate: `python3 -m scripts.aggregate_benchmark test-output/openai-api-workspace/iteration-N --skill-name openai-api` (run from the `skill-creator` skill dir).
   - Generate the HTML viewer: `python3 <skill-creator>/eval-viewer/generate_review.py test-output/openai-api-workspace/iteration-N --skill-name openai-api --benchmark .../benchmark.json --static .../review.html`.
3. **Review** by opening `review.html` and filling in per-eval feedback. Submit-all-reviews downloads `feedback.json`.
4. **Iterate.** Apply feedback to `openai-api/`, then re-run as `iteration-N+1/`.

### Description optimization

After skill content is locked, run the description optimizer (`scripts/run_loop.py` in `skill-creator/`) to tune the SKILL.md `description:` field for triggering accuracy. That step uses real `claude -p` calls against a trigger eval set — needs the openai key.

### Packaging

`python3 -m scripts.package_skill openai-api` (run from the `skill-creator/` dir, or with the path adjusted) produces `openai-api.skill`. That's the shippable artifact.

---

## Conventions for Claude when working in this repo

- Don't create files at the repo root unless they're part of the skill or the repo's metadata (CLAUDE.md, LICENSE, .gitignore, README if added later).
- Don't install the skill into `~/.claude/skills/` from this repo — this is the dev source, not the installation.
- When asked to "test the skill," that means: run the eval workflow (paired subagents → grade → aggregate → review viewer) into `test-output/`, NOT install + invoke.
- When the user mentions an existing iteration ("iteration-1"), look under `test-output/openai-api-workspace/`.
- Models in any example code default to `gpt-5.2` (matching what the skill itself recommends).
- Never write a `sk-...` value to disk, even as a placeholder.
