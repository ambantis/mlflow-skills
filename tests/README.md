# Running the Tests

Tests run Claude Code headlessly against a real MLflow server, then evaluate the agent's behavior using MLflow judges.

## Prerequisites

**Claude Code** must be authenticated in your shell. The test runner invokes `claude -p` as a subprocess and inherits whatever auth mechanism you have configured (API key, enterprise SSO, Bedrock, Vertex, etc.).

**An LLM API key** is required for the LLM-based judges. MLflow uses `openai/gpt-4.1-mini` by default, or the Databricks managed judge model when `MLFLOW_TRACKING_URI` points at a Databricks workspace. Copy `.env.example` to `.env` at the repo root and fill in your credentials:

```bash
cp .env.example .env
```

The test runner loads `.env` automatically on startup.

## Running a Test

Each test requires a `REPO_URL` pointing at a real agent codebase. This is the project Claude will work on during the test — it gets cloned into a temp directory, the skill is installed into it, and Claude runs there following the test prompt. There is no default; you must supply one.

From the repo root:

```bash
uv run python tests/test_skill.py tests/configs/tracing_test.yaml \
  REPO_URL=https://github.com/your-org/your-agent

uv run python tests/test_skill.py tests/configs/agent_evaluation.yaml \
  REPO_URL=https://github.com/your-org/your-agent
```

Any values listed under `environment:` in the YAML config can be passed as `KEY=VALUE` arguments after the config path, overriding the values in the YAML.

## MLflow Server

By default the test runner starts a local MLflow server on the port specified in the YAML (`mlflow_port`). To use an existing server instead, set `tracking_uri` in the YAML or point `MLFLOW_TRACKING_URI` at it — Databricks workspaces are also supported.

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All judges passed |
| 1 | Setup failed (missing skills, port in use, etc.) |
| 2 | Claude Code execution failed |
| 3 | One or more judges failed |

## Adding a Test

1. Add a YAML config to `configs/`
2. Add a setup script to `scripts/` that clones or prepares the target project
3. Add judge modules to `judges/` — each must export `get_judges() -> list`
4. Reference the setup script and judge paths in the YAML
