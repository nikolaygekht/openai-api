# Project Status

**Date:** 2026-05-21
**Skill:** `openai-api` (Claude Code skill)
**Phase:** Iteration 1 complete; awaiting human review before iteration 2.

---

## TL;DR

The skill is **functional and useful today**. All 19 skill files are written (~5,500 lines), the first round of evals ran cleanly (96 % with-skill vs 60 % baseline across 8 tasks), and two real-world experiments under `test-output/` show it works end-to-end on production-shaped problems. The next blocker is human review of the eval outputs to drive iteration 2.

---

## Done

### Skill content — complete

`openai-api/` contains 19 files:

| Area | Files | Lines |
|---|---|---|
| Entry point | `SKILL.md` | 274 |
| Shared (cross-cutting) | `shared/{models,prompt-caching,structured-outputs,tool-use-concepts,error-codes,secrets,live-sources}.md` | 1,420 |
| Python | `python/openai-api/{README,streaming,tool-use,audio}.md` | 1,240 |
| TypeScript | `typescript/openai-api/{README,streaming,tool-use,audio}.md` | 1,150 |
| C# | `csharp/openai-api.md` | 509 |
| Java | `java/openai-api.md` | 485 |
| cURL | `curl/examples.md` | 450 |
| Evals | `evals/evals.json` | (8 test prompts) |
| **Total** | **19 files** | **~5,500 lines** |

### Iteration 1 evals — run and aggregated

8 paired runs (with-skill + baseline) under `test-output/openai-api-workspace/iteration-1/`:

| # | Eval | With Skill | Baseline | Delta |
|---|------|-----------|----------|-------|
| 1 | python-basic-chat-with-secrets | 100 % | 57 % | +43pp |
| 2 | typescript-streaming-nextjs-route | 100 % | 62 % | +38pp |
| 3 | cost-optimization-prompt-caching | 83 % | 83 % | 0 *(weak assertions)* |
| 4 | python-tool-use-agent | 100 % | 50 % | +50pp |
| 5 | csharp-reasoning-with-key-from-env | 100 % | 57 % | +43pp |
| 6 | audio-transcription-long-file | 100 % | 100 % | 0 *(weak assertion)* |
| 7 | typescript-structured-extraction | 100 % | 12 % | **+88pp** |
| 8 | migration-from-chat-completions | 86 % | 57 % | +29pp |
| **avg** | | **96 %** | **60 %** | **+36pp** |

Bench artifacts: `benchmark.json`, `benchmark.md`, `review.html` (renderable side-by-side viewer), plus `grade.py` for re-running the assertion grader.

### Real-world experiments — both worked

- **test01 (carport drawing dimension extraction)** — single Responses-API call with vision input. Extracted title block, 5 dimensions, 4 materials. Cost ≈ $0.015. First try.
- **test02 (door drawing vs cutlist cross-validation)** — three-call pipeline (extract drawing → extract cutlist → cross-check). Found one genuine engineering concern (proposal revision mismatch: drawing rev `-7` vs cutlist rev `-0`). Cost ≈ $0.245 total. Full writeup in `test-output/test02/results.md`.

### Documentation — complete

- `README.md` — project intro, scope, evidence-it-works
- `INSTALL.md` — global / project / symlink install instructions across platforms
- `CLAUDE.md` — guidance for Claude Code when working inside this repo
- `LICENSE`
- `.gitignore` — excludes `.env/`, `test-output/`, `.claude/settings.local.json`

---

## In progress

### Iteration 2 — gated on human review

`test-output/openai-api-workspace/iteration-1/review.html` is built and ready. The next step is for **a human to open it, walk through the 16 outputs (8 with-skill + 8 baseline), leave per-eval feedback, and click "Submit All Reviews"** to download `feedback.json`. That file drives the iteration-2 rewrite.

Until that's done, I'm not editing the skill — there's no signal to act on beyond the assertions, and the assertions already report 96 %.

---

## Not started

### Description optimization

`scripts/run_loop.py` in the `skill-creator` skill drives a 5-iteration optimization loop over the SKILL.md `description:` field to maximize triggering accuracy on a held-out test set. We'd:

1. Generate ~20 trigger eval queries (mix of should-trigger / should-not-trigger).
2. Review them.
3. Run the loop with `--model claude-opus-4-7` (the model powering Claude Code).

This step needs the OpenAI key — to launch the same `claude -p` calls the loop uses. The key is at `.env/open-ai` so we have what we need. We should wait until skill content is locked (post-iteration-2) before optimizing the description.

### Packaging

`python3 -m scripts.package_skill openai-api` (from the skill-creator directory) produces `openai-api.skill` — a zip-format artifact you can hand to users for a one-click install. We haven't packaged yet; doing so before iteration 2 would mean shipping again immediately after.

---

## Open questions / decisions to make

1. **Should evals 3 and 6 get tighter assertions?** Both showed 0 delta in iteration 1, but the runs themselves visibly differed (baselines used wrong field names / wrong models). The grading patterns just didn't catch it. Worth fixing before iteration 2 so we can detect real regressions.
2. **Should the audio coverage stay where it is, or expand?** Right now we cover transcription, translation, and TTS. We don't cover Realtime (voice agents) — that was an explicit out-of-scope decision at scoping time. Stay or revisit?
3. **What's the bar for shipping iteration 1?** Possible answers: (a) ship as-is after user review, (b) only ship after iteration 2 with feedback applied, (c) ship after at least one external user has tried it. No decision yet.

---

## Known limitations

- **`platform.openai.com` returns 403 to Claude Code's WebFetch tool.** The skill documents the workaround (WebSearch + GitHub for SDK source), but it's a friction point for anyone trying to live-fetch newer docs.
- **Model and pricing data is cached at 2026-05-21.** OpenAI's catalog shifts roughly quarterly. `shared/models.md` will need refresh on a similar cadence; `shared/live-sources.md` documents how.
- **C# .NET SDK is experimental for Responses.** `OPENAI001` warning suppression is required. Single `ClientResultException` (no per-status subclasses) is a real ergonomic gap vs. Python / Node / Java — the skill documents this but can't fix it.
- **Iteration 1 evals were code-only** (subagents wrote files but didn't execute). We haven't actually run the generated code against the real OpenAI API yet to confirm it works. The key at `.env/open-ai` makes that possible in iteration 2 if we want to add a "code actually runs" assertion layer.

---

## File map of artifacts produced in this iteration

```
openai-api-skill/
├── README.md                                              ← shipping doc
├── INSTALL.md                                             ← shipping doc
├── CLAUDE.md                                              ← repo-level guide for Claude Code
├── STATUS.md                                              ← this file
├── LICENSE
├── openai-api/                                            ← THE SKILL (19 files, ~5,500 lines)
│   ├── SKILL.md
│   ├── shared/             (7 files)
│   ├── python/openai-api/  (4 files)
│   ├── typescript/openai-api/ (4 files)
│   ├── csharp/openai-api.md
│   ├── java/openai-api.md
│   ├── curl/examples.md
│   └── evals/evals.json
└── test-output/                                           ← gitignored
    ├── _research/          (4 briefs, ~2,350 lines)
    ├── openai-api-workspace/iteration-1/                  ← eval bench
    │   ├── benchmark.json, benchmark.md, review.html
    │   ├── grade.py, regrade.py
    │   └── eval-{01..08}-*/{with_skill,without_skill}/run-1/{outputs/, grading.json, timing.json}
    ├── test01/                                            ← carport drawing experiment
    │   ├── drawing.pdf
    │   ├── extract_dimensions.py
    │   └── output.json
    └── test02/                                            ← door + cutlist cross-validation
        ├── drawing.pdf, cutlist.pdf
        ├── cross_check.py
        ├── drawing.json, cutlist.json, alignment.json
        └── results.md
```

---

## Next concrete action

**A human (you) opens `test-output/openai-api-workspace/iteration-1/review.html` in a browser, walks through the 16 outputs, leaves feedback, and clicks "Submit All Reviews."** That produces `feedback.json` next to the review file. Drop that file into the workspace and tell me — I'll read it and start iteration 2.

If you'd rather skip the per-output review for now and trust the 96 %-vs-60 % benchmark, say so and I'll move on to either (a) tightening eval 3 and 6 assertions, (b) description optimization, or (c) packaging.
