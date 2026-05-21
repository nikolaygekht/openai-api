# Installing the `openai-api` skill into Claude Code

Two ways to install: **globally** (available in every Claude Code session) or **per-project** (only when you open a specific repo). Pick based on how often you use OpenAI in your work.

> The skill is the `openai-api/` directory in this repo — that's the entire artifact. Installation is just copying that directory into the right `.claude/skills/` location.

---

## Prerequisites

- **Claude Code** installed and working — see https://docs.claude.com/en/docs/claude-code.
- Read access to this repository (clone it, or download a tarball).

No other dependencies — the skill is plain markdown. The Python / Node / C# / Java *examples* inside the skill reference real SDKs, but those are only installed in your *projects* when you actually use them; the skill itself has no install-time dependencies.

---

## Option A — Global install (recommended)

A global install makes the skill available in every Claude Code session on your machine, regardless of which directory you're working in. This is what you want if you use OpenAI from multiple projects.

### Linux / macOS / WSL

```bash
# 1. Clone this repo somewhere convenient
git clone <repo-url> ~/src/openai-api-skill
cd ~/src/openai-api-skill

# 2. Copy the skill into Claude Code's global skills directory
mkdir -p ~/.claude/skills
cp -r openai-api ~/.claude/skills/

# 3. Verify
ls ~/.claude/skills/openai-api/SKILL.md
```

### Windows (PowerShell)

```powershell
# 1. Clone this repo
git clone <repo-url> $HOME\src\openai-api-skill
Set-Location $HOME\src\openai-api-skill

# 2. Copy the skill into Claude Code's global skills directory
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse openai-api "$HOME\.claude\skills\"

# 3. Verify
Test-Path "$HOME\.claude\skills\openai-api\SKILL.md"
```

### Windows (CMD)

```cmd
git clone <repo-url> %USERPROFILE%\src\openai-api-skill
cd %USERPROFILE%\src\openai-api-skill

if not exist "%USERPROFILE%\.claude\skills" mkdir "%USERPROFILE%\.claude\skills"
xcopy /E /I openai-api "%USERPROFILE%\.claude\skills\openai-api\"
```

### Verifying the global install

1. Start a new Claude Code session (in any directory).
2. Ask: *"Do you have a skill called openai-api available?"*
3. Claude should list it among `available_skills` in its system prompt and reply yes.

Or trigger it directly: *"Write a small Python script that uses the OpenAI API to answer a question."* Claude should invoke the skill and produce code that uses `client.responses.create()` with `gpt-5.2`, reads the key from env, and adds `.env` to `.gitignore`.

---

## Option B — Project-scoped install

A project-scoped install makes the skill available *only* when Claude Code is opened inside that specific project. Use this when:

- You don't want the skill cluttering Claude's available-skills list in unrelated projects.
- You want to pin a specific *version* of the skill to a project (so the project's behavior is reproducible even if the global skill changes).
- You're in an environment where `~/.claude/skills/` is read-only (some sandboxed setups).

### Steps

From inside your project's repo root:

```bash
# Linux / macOS / WSL
mkdir -p .claude/skills
cp -r /path/to/openai-api-skill/openai-api .claude/skills/
```

```powershell
# Windows PowerShell
New-Item -ItemType Directory -Force -Path .claude\skills | Out-Null
Copy-Item -Recurse C:\path\to\openai-api-skill\openai-api .claude\skills\
```

Then commit `.claude/skills/openai-api/` to version control if you want the whole team to share it (recommended for shared codebases) — or add `.claude/skills/` to `.gitignore` if it should stay local.

### Verifying the project install

Inside the project directory, start a new Claude Code session and ask the same verification question as above. Claude Code picks up both global and project skills; project skills override global ones if the name matches.

---

## Option C — Development install (symlink, for hacking on the skill)

If you want to edit the skill and have changes reflect immediately without re-copying:

```bash
# Linux / macOS / WSL
ln -s "$(pwd)/openai-api" ~/.claude/skills/openai-api
```

```powershell
# Windows (run as Administrator, or with Developer Mode enabled)
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\openai-api" -Target "$(Get-Location)\openai-api"
```

Now any edit you save in this repo's `openai-api/` is live in Claude Code on the next session start.

> Note: symlinks don't work the same across WSL ↔ Windows. If you edit on the Windows side but Claude Code runs in WSL (or vice versa), use a regular copy instead.

---

## Updating the skill

If you've installed via copy:

```bash
# Linux / macOS / WSL — global
cd ~/src/openai-api-skill
git pull
rm -rf ~/.claude/skills/openai-api
cp -r openai-api ~/.claude/skills/
```

```powershell
# Windows PowerShell — global
Set-Location $HOME\src\openai-api-skill
git pull
Remove-Item -Recurse -Force "$HOME\.claude\skills\openai-api"
Copy-Item -Recurse openai-api "$HOME\.claude\skills\"
```

For symlinked dev installs, `git pull` alone is enough — changes propagate automatically.

For project-scoped installs, run the same delete-then-copy inside the project's `.claude/skills/` directory.

---

## Uninstalling

### Global

```bash
# Linux / macOS / WSL
rm -rf ~/.claude/skills/openai-api
```

```powershell
# Windows
Remove-Item -Recurse -Force "$HOME\.claude\skills\openai-api"
```

### Project

```bash
rm -rf .claude/skills/openai-api
```

After uninstalling, restart Claude Code so it stops listing the skill in its available-skills set.

---

## Troubleshooting

**Claude doesn't seem to know about the skill.**
1. Confirm the file `SKILL.md` exists at the right path:
   - Global: `~/.claude/skills/openai-api/SKILL.md`
   - Project: `<project>/.claude/skills/openai-api/SKILL.md`
2. Restart your Claude Code session. Skills are loaded at session start.
3. Ask: *"List the skills you have available."* — `openai-api` should appear in the response.

**Claude triggers the wrong skill (e.g., `claude-api` instead).**
The skill's `description:` field is designed to trigger on OpenAI-specific cues (imports of `openai`, mentions of GPT-5, Responses API, Whisper, etc.) and explicitly *not* trigger on Anthropic / Claude work. If you see misfires, try a more explicit prompt, e.g., *"using the OpenAI API in Python …"*, or open an issue with the failing prompt so the description can be tuned.

**WSL + Windows paths get tangled.**
Pick one side and stick with it. If Claude Code runs in WSL, install to `~/.claude/skills/` inside WSL (Linux paths). If it runs natively on Windows, use `%USERPROFILE%\.claude\skills\`. Don't mix.

**`platform.openai.com` returns 403 inside Claude Code's WebFetch.**
That's not an install problem — OpenAI's docs site blocks the WebFetch tool. The skill handles this by directing Claude to use `WebSearch` instead, or to fetch the SDK source on GitHub. See `shared/live-sources.md` inside the installed skill.

---

## Multiple installs at once

You can have all three install modes coexisting — global + project + symlinked dev — and Claude Code will resolve in this order: **project > global**. The most specific install wins. Useful pattern during development:

- Global = stable shipping version (copied).
- One project = symlinked to your dev checkout for live testing.
- Other projects = use the global by default.

---

## Where to file issues

Open an issue on this repo if:

- The skill triggers when it shouldn't (false positive).
- The skill *doesn't* trigger when it should (false negative — most common after OpenAI ships new features).
- A code example is wrong or outdated.
- Model IDs, pricing, or API surfaces in `shared/models.md` are stale.

Include the prompt you used and the Claude Code version (`claude --version`).
