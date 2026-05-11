# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code skills (and a matching plugin) that give coding agents deep MLflow expertise. The skills cover tracing, evaluation, debugging, and documentation search. The repo also includes a test framework that runs skills headlessly via `claude -p` and evaluates their output with MLflow judges.

## Skill Structure

Each skill is a directory at repo root containing `SKILL.md` (required) and optional subdirectories (`scripts/`, `references/`, `assets/`). The `SKILL.md` frontmatter defines `name`, `description`, `allowed-tools`, and optionally `disable-model-invocation`.

```
<skill-name>/
  SKILL.md          # Skill definition with YAML frontmatter
  scripts/          # Python templates referenced in SKILL.md
  references/       # Supporting reference files
```

The `.claude-plugin/plugin.json` manifest lists all skills and hooks for marketplace distribution.

## Test Framework

Tests live in `tests/` and are driven by YAML configs in `tests/configs/`.

**Run a test:**
```bash
cd tests
python test_skill.py configs/tracing_test.yaml
python test_skill.py configs/agent_evaluation.yaml
```

**Test config fields** (YAML):
- `project_dir` — subdirectory within the temp work dir where Claude Code runs
- `setup_script` — Python script that clones/prepares the test project
- `judges` — list of judge module paths; each exports `get_judges() -> list`
- `skills` — skill directory names copied into the test project's `.claude/skills/`
- `prompt` — the exact prompt sent to `claude -p`
- `mlflow_port` — local MLflow server port (must be available); set `tracking_uri` instead to use an external server
- `environment` — env vars passed to the setup script (e.g., `REPO_URL`, `OPENAI_API_KEY`)

**Test flow:**
1. `infra.py` checks prerequisites (skills exist, port free), starts local MLflow, creates experiments
2. `infra.py` runs the setup script (clones a repo into the work dir), installs skills, configures CC tracing
3. `test_skill.py` invokes `claude -p <prompt>` with allowed tools and captures output
4. `run_judges.py` (spawned as a subprocess) loads judge modules, fetches CC traces from MLflow, runs `mlflow.genai.evaluate()` on them
5. Exit code reflects which phase failed: 1=setup, 2=execution, 3=verification

**Writing judges** — each judge module exports `get_judges()` returning a list of judges created with `mlflow.genai.judges.make_judge()`. Judges check CC traces for evidence of correct skill behavior.

## Hook

`hooks/mlflow-suggest-hook.py` is a `UserPromptSubmit` hook that reads `{"prompt": "..."}` from stdin, matches keywords, and prints skill suggestions to stdout. Install in `~/.claude/settings.json` under `hooks.UserPromptSubmit`.

## Skill Authoring Conventions

- `SKILL.md` frontmatter must include `name` and `description`; `description` is the trigger text the agent uses to decide when to invoke the skill
- `mlflow-agent` is the master dispatcher (`disable-model-invocation: true`) — it routes to sub-skills without doing LLM work itself
- Skills reference `uv run` for all Python/MLflow commands
- Scripts in `scripts/` are templates the skill instructs the agent to copy into the user's project, not scripts run directly

## Requirements

- **MLflow 3.8+** for all skill APIs and test judge evaluation
- `pyyaml`, `mlflow`, `litellm` for the test framework (`pip install pyyaml mlflow litellm`)
- `claude` CLI in PATH for running tests
