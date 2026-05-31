---
title: "Claude Code Project Init Spec"
author: "Suraj Jaiswal"
date: "2026-05-31"
date-modified: "2026-05-31"
categories: [Agentic AI, Claude Code, MCP, LLM]
description: "Executable setup spec for Claude Code: scope-correct MCP installs (user vs project), plugin installs, project-type detection, scaffold templates, and dependency setup. Companion spec to The Ultimate Claude Code Setup — run this, not the blog post."
image: images/claude_logo_img.png
toc: true
toc-depth: 2
---

# Claude Code Setup Spec

> **Canonical URL:** `https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md`

> **Companion blog with rationale:** `https://jaiswalsuraj487.github.io/blogs/blogsData/claude_code_plugins_mcp_skills.html`
> **Last verified:** 2026-05-31 against npm, PyPI, github/github-mcp-server official docs, and `anthropics/claude-plugins-official`.

> **Claude — read this entire document, then execute it on the current project. This is not a blog post. Every section is mandatory. Print a one-line checkpoint after each phase (e.g. `✓ Phase 1a complete (5 added, 0 skipped)`). End with the Phase 5 report in the exact format specified.**
>
> **Idempotency:** every step is safe to re-run. Before any install or file creation, check existence and skip or ask before overwriting.
>
> **Scope rule:** generic tools → `--scope user`. Project-specific tools → `--scope project` (writes to `<repo>/.mcp.json`, gets committed). Personal overrides → `.claude/settings.local.json` (gitignored).

---

## Phase 0 — Sanity checks

Run these. If any fail, stop and tell the user to install:

```bash
claude --version
node --version          # need >= 20
uv --version            # `pipx install uv` if missing
docker --version        # GitHub MCP runs via docker
jq --version            # used by the pytest hook
```

`✓ Phase 0 complete` checkpoint.

---

## Phase 1a — User-scope MCP servers (universal)

Skip any already in `claude mcp list --scope user`.

```bash
# Context7 — anonymous mode works fine for personal use. If you hit rate limits,
# get a free API key at context7.com and append: -- npx -y @upstash/context7-mcp --api-key YOUR_KEY
claude mcp add --scope user context7 -- npx -y @upstash/context7-mcp

# GitHub — official binary (ghcr.io). The npm @modelcontextprotocol/server-github
# is DEPRECATED and shows "no longer supported".
# The -e flag passes the env var through to docker. Requires $GITHUB_PAT in your shell.
claude mcp add --scope user github -e GITHUB_PERSONAL_ACCESS_TOKEN=$GITHUB_PAT -- \
  docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN \
  ghcr.io/github/github-mcp-server

# Alternative if Docker isn't available — use the remote HTTP transport:
# claude mcp add --transport http github https://api.githubcopilot.com/mcp \
#   -H "Authorization: Bearer $GITHUB_PAT"

claude mcp add --scope user fs -- npx -y @modelcontextprotocol/server-filesystem "$HOME/code"

claude mcp add --scope user memory -- npx -y @modelcontextprotocol/server-memory

claude mcp add --scope user seq -- npx -y @modelcontextprotocol/server-sequential-thinking
```

`✓ Phase 1a complete (X added, Y skipped)` checkpoint.

---

## Phase 2 — Project-type detection

Inspect the repo and match against this table. Pick the dominant signal. If nothing matches, **ask the user** with project-type options as buttons. Print the detected type and your reasoning before continuing.

| Signals | Project type | Phase 1b servers | Phase 4 eval framework |
|---|---|---|---|
| `langgraph` / `langchain` in deps, `agents/` or `graph.py` | **LangGraph agent** | langfuse (opt), sentry (opt) | `langsmith agentevals` |
| `chromadb` / `pgvector` / `qdrant` / `weaviate`; `rag/` or `retriever.py` | **RAG** | postgres, langfuse (opt), sentry (opt) | `ragas` |
| `ultralytics` / `torch` + `opencv` / image dirs | **CV / YOLO** | aws-api+docs (if S3), sentry (opt) | `pycocotools imagehash supervision` |
| `transformers` + training script (`train.py`, HF Trainer) | **Fine-tuning** | aws-api+docs (if S3) | `lm-eval` |
| `fastapi` / `uvicorn`, no LLM deps | **Plain backend** | postgres (if DB), sentry (opt) | *(skip)* |
| Any project that scrapes / tests / drives a browser | **+ browser automation** | playwright OR agent-browser (pick one — see Phase 1b) | — |
| Empty repo or none of the above | **Ask the user** | — | — |

`✓ Phase 2 complete (detected: <type>, reason: <signal>)` checkpoint.

---

## Phase 1b — Project-scope MCP servers (only what the detected type needs)

Install with `--scope project` so they write to `<repo>/.mcp.json` and teammates inherit them. Skip any already in `claude mcp list --scope project`. Reference env vars — never bake real secrets.

```bash
# Postgres — RAG and DB-backed plain-backend projects.
# DO NOT use @modelcontextprotocol/server-postgres (archived after SQLi CVE).
# Defaults to read-only; add --access-mode=unrestricted only if you genuinely need writes:
claude mcp add --scope project postgres -- uvx postgres-mcp "$DATABASE_URL"
# Or for read+write:
# claude mcp add --scope project postgres -- uvx postgres-mcp --access-mode=unrestricted "$DATABASE_URL"

# Browser automation — pick ONE of these, not both. They overlap.
#
# Option A: Playwright MCP (Microsoft, most mature, ~114K tokens of context tax in full sessions)
claude mcp add --scope project playwright -- npx -y @playwright/mcp@latest
#
# Option B: agent-browser (Vercel Labs, Rust CLI, claims 93% smaller context footprint).
# Younger project (Jan 2026), 23K+ stars, 109K weekly npm downloads. Installed as a SKILL, not an MCP.
# npm install -g agent-browser
# agent-browser install     # downloads Chrome for Testing
# npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y    # project scope
# npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y -g # global scope

# AWS — CV / fine-tuning / AWS-deploy projects. Use awslabs.* (no @aws/mcp-server).
claude mcp add --scope project aws-api -- uvx awslabs.aws-api-mcp-server@latest
claude mcp add --scope project aws-docs -- uvx awslabs.aws-documentation-mcp-server@latest

# Langfuse — LLM observability. HTTP transport (no npm package).
claude mcp add --scope project --transport http langfuse \
  https://cloud.langfuse.com/api/public/mcp \
  --header "Authorization: Basic $(printf '%s:%s' "$LF_PUBLIC_KEY" "$LF_SECRET_KEY" | base64)"

# Sentry — production error tracking.
claude mcp add --scope project --transport http sentry https://mcp.sentry.dev/mcp
```

`✓ Phase 1b complete (X added, Y skipped)` checkpoint.

---

## Phase 3 — Plugins (user scope, inside Claude Code)

Skip any in `/plugin list`. All verified in the `claude-plugins-official` marketplace.

```text
# Core workflow plugins — recommended for every project
/plugin install ralph-loop@claude-plugins-official
/plugin install feature-dev@claude-plugins-official
/plugin install code-simplifier@claude-plugins-official
/plugin install code-review@claude-plugins-official
/plugin install pr-review-toolkit@claude-plugins-official
/plugin install claude-md-management@claude-plugins-official
/plugin install pyright-lsp@claude-plugins-official
/plugin install coderabbit@claude-plugins-official

# Optional — install per project need:
# /plugin install codspeed@claude-plugins-official              # performance benchmarking + MCP
# /plugin install 42crunch-api-security-testing@claude-plugins-official  # OpenAPI security audits

# superpowers is in a separate marketplace (Jesse Vincent's, not Anthropic's).
# Adds workflow slash commands like /brainstorming, /tdd, /systematic-debugging.
/plugin marketplace add obra/superpowers
/plugin install superpowers@obra-superpowers

/reload-plugins
```

**Notes:**
- If `ralph-loop` install fails, try `ralph-wiggum@claude-plugins-official` — some sources document the plugin under that name.
- Adding any third-party marketplace (`obra/superpowers` above) is a trust decision — plugins can register hooks that intercept tool calls. Inspect the source before installing if you don't already trust the author.
- `pyright-lsp` has a known config-propagation bug on Windows (anthropics/claude-code#16219). If LSP features don't appear, hand-configure or use `ruff` + manual `mypy` instead.

`✓ Phase 3 complete (X added, Y skipped)` checkpoint.

---

## Phase 4 — Scaffold files

Create these in the project root. **Ask before overwriting any that exist.** Templates are inline below.

```
CLAUDE.md
.claude/settings.json
.claude/hooks/pytest-changed.sh    # chmod +x after creating
.claude/agents/test-writer.md
.claude/agents/eval-runner.md
.claude/skills/tdd-loop/SKILL.md
.env.example                       # only if Phase 1b installed servers using env vars
```

Append to `.gitignore` (create if missing):

```
.claude/settings.local.json
.env
```

### 4.1 `CLAUDE.md`

Fill in placeholders from the detected project. Don't leave `<NAME>` etc. unfilled.

```markdown
# Project: <NAME>

## What this is
<one paragraph — what it does, who uses it, what stack>

## Stack
- Python 3.12, uv for env management
- <FastAPI | LangGraph | RAG | YOLO | ...>
- <Postgres + pgvector | etc.>

## Commands
- Install: `uv sync`
- Run: `uvicorn app.main:app --reload`   # or project-appropriate
- Test (fast): `uv run pytest -x --no-cov`
- Test (full): `uv run pytest --cov=app --cov-report=term-missing`
- Lint: `uv run ruff check . && uv run ruff format --check .`
- Type-check: `uv run mypy src/`
- Eval: `@eval-runner`

## Repo layout
- `src/` — application code
- `tests/` — pytest suite
- `evals/` — eval datasets and harness; do not modify without explicit ask
- `.claude/` — subagents, skills, hooks (committed)

## Testing rules (apply to production code in src/app/core, agents, rag, api)
1. TDD via the `tdd-loop` skill for any feature in production paths.
2. No live external API in CI. Wrap LLM/HTTP calls with `@pytest.mark.vcr()`.
3. Async tests use `httpx.AsyncClient` + `ASGITransport`.
4. LLM outputs: assert structure (Pydantic), properties, or syrupy snapshots — never literal strings.
5. Coverage: 80% line, 100% on `app/core/` and `app/agents/`.
6. Property-based (Hypothesis) for every pure utility.
7. Contract tests (Schemathesis) on every PR.
8. Prompt/RAG/agent changes require passing `@eval-runner` against `evals/thresholds.yaml`.

## Conventions
- `uv add <pkg>`, not `pip install`
- `uv run <cmd>`, not bare `python`
- Avoid `print` — use `loguru`
- Pydantic v2 syntax
- All public functions get type hints

## Don't
- Don't commit unless I explicitly say "commit".
- Don't run `terraform apply` — only `plan`.
- Don't touch `evals/datasets/`.
- Don't add top-level dependencies without asking.
- Don't bypass `@pytest.mark.vcr()` in LLM-touching tests.
- Don't suggest the deprecated `@modelcontextprotocol/server-postgres` — use Postgres MCP Pro.
```

### 4.2 `.claude/settings.json`

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | grep -E \"(rm -rf /|sudo rm|:(){ :|:&)\" && echo BLOCKED && exit 2 || exit 0'"
        }]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | grep -E \"(AKIA[0-9A-Z]{16}|sk-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36})\" && echo SECRET && exit 2 || exit 0'"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{
          "type": "command",
          "command": "bash \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/pytest-changed.sh"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "uv run ruff format . && uv run ruff check --fix . 2>&1 | tail -5" },
          { "type": "command", "command": "uv run pytest -q 2>&1 | tail -10" }
        ]
      }
    ]
  },
  "permissions": {
    "allow": [
      "Bash(uv run pytest *)",
      "Bash(uv run ruff *)",
      "Bash(uv run mypy *)",
      "Bash(uv add *)",
      "Bash(uv sync *)",
      "Bash(gh *)",
      "Edit(*)",
      "Write(*)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(docker run *)",
      "Bash(terraform plan *)",
      "Bash(aws *)",
      "mcp__playwright__*",
      "mcp__postgres__execute_sql",
      "mcp__github__create_*",
      "mcp__github__merge_*",
      "mcp__aws-api__*"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Bash(terraform apply *)",
      "Bash(terraform destroy *)",
      "Bash(kubectl delete *)",
      "mcp__github__delete_*",
      "mcp__postgres__drop_*",
      "mcp__postgres__truncate_*"
    ]
  }
}
```

### 4.3 `.claude/hooks/pytest-changed.sh`

```bash
#!/usr/bin/env bash
# PostToolUse hook: runs pytest on the matching test file when source changes.
# Handles src/, app/, and flat layouts. Never blocks the session.
set -u
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
[[ -z "$FILE" || "$FILE" != *.py ]] && exit 0

ROOT="$(git -C "$(dirname "$FILE")" rev-parse --show-toplevel 2>/dev/null)"
[[ -z "$ROOT" ]] && exit 0
REL="${FILE#$ROOT/}"
BASENAME=$(basename "$REL" .py)
DIRNAME=$(dirname "$REL")

if [[ "$BASENAME" == test_* || "$BASENAME" == *_test ]]; then
  TARGET="$REL"
else
  STRIPPED="${DIRNAME#src/}"
  STRIPPED="${STRIPPED#app/}"
  CANDIDATES=(
    "tests/${STRIPPED}/test_${BASENAME}.py"
    "tests/test_${BASENAME}.py"
    "${DIRNAME}/test_${BASENAME}.py"
  )
  TARGET=""
  for c in "${CANDIDATES[@]}"; do
    [[ -f "$ROOT/$c" ]] && { TARGET="$c"; break; }
  done
fi

[[ -z "$TARGET" ]] && exit 0
cd "$ROOT"
timeout 30 uv run pytest "$TARGET" -x --tb=short --no-header -q 2>&1 | tail -20 >&2
exit 0
```

Then: `chmod +x .claude/hooks/pytest-changed.sh`

### 4.4 `.claude/agents/test-writer.md`

```markdown
---
name: test-writer
description: Writes pytest unit, integration, and property-based tests for the file or function the user names. Generates fixtures, mocks external services with respx/responses, uses Hypothesis for pure functions. Use when the user says "write tests for", "add tests for", or "test this function/file/module".
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
---
You are a senior Python test engineer. When invoked:

1. Read the target file and inspect any sibling `tests/` patterns.
2. Detect the framework: FastAPI → `httpx.AsyncClient` + `ASGITransport`; LangGraph → `agentevals` + `InMemorySaver`; RAG → `ragas.evaluate`; plain Python → pytest + Hypothesis.
3. Generate in order: happy path, edge cases, error paths, Hypothesis properties for pure functions, syrupy snapshots for stable serialized outputs.
4. Mock externals: HTTP via `respx`/`responses`, time via `freezegun`, LLM/HTTP via `@pytest.mark.vcr()`, DB via fixtures with rollback.
5. Write the file. Run `uv run pytest -x` on it. Fix if it fails.
6. Output a one-paragraph coverage summary listing paths NOT covered.

Hard rules:
- Never hit live external APIs without VCR cassettes.
- Never assert on literal LLM output strings — use syrupy, properties, or LLM-as-judge.
- Never use `time.sleep()` — freezegun or asyncio mock-time.
- Test files in `tests/` mirroring source layout.
- Use parametrize for variations, not copy-paste.
```

### 4.5 `.claude/agents/eval-runner.md`

```markdown
---
name: eval-runner
description: Runs the project's eval suite (DeepEval / Ragas / agentevals / lm-eval) and produces a structured pass/fail report with per-metric scores, deltas vs the last run, and a list of failing examples. Use after any prompt change, retrieval change, model swap, or anything that touches app/prompts, app/agents, or app/rag.
tools: Read, Grep, Bash, Edit
model: sonnet
---
You are the project's evaluation operator.

1. Detect framework from `pyproject.toml` / `requirements*.txt`:
   - `ragas`               → `uv run python -m evals.ragas_run`
   - `deepeval`            → `uv run deepeval test run evals/`
   - `agentevals`/`langsmith` → `uv run pytest evals/ -m langsmith`
   - `lm-eval`             → `uv run lm_eval --tasks <project_tasks> ...`
2. Capture JSON to `evals/runs/<ISO-timestamp>.json`.
3. Diff vs `evals/runs/baseline.json`. Flag metrics regressed > threshold in `evals/thresholds.yaml` (default 2% absolute).
4. Print markdown table: metric | baseline | current | Δ | verdict.
5. Exit non-zero if any blocker metric regressed (read blockers from `evals/thresholds.yaml`).
6. For each failing example: ID, input, expected, got, metric score.

Hard rules:
- Never run live LLM evals on every CI step — use cassettes.
- Schedule expensive judge-LLM evals nightly, not per-PR.
- If `baseline.json` missing, treat current as new baseline and notice the user.
- If suite > 10 minutes, run subset (`--sample 50`) and say so.
- Never modify `evals/datasets/` — frozen for regression comparison.
```

### 4.6 `.claude/skills/tdd-loop/SKILL.md`

```markdown
---
name: tdd-loop
description: Disciplined red-green-refactor loop for production code. Use when the user asks to add a feature, build, implement an endpoint, fix a bug, or refactor a function in production code paths (src/app/core, src/app/agents, src/app/rag, src/app/api). Forces writing a failing test first, then minimal code to pass, then refactor. Triggers on "add", "build", "implement", "fix bug", "refactor" combined with a production code path.
---

Use ONLY for production-code work in `src/app/core/`, `src/app/agents/`, `src/app/rag/`, `src/app/api/`. Skip for prompt tweaks (use `@eval-runner` instead), notebooks, docs, CI/infra YAML.

## Procedure

1. **Red.** Smallest failing test in `tests/`. Match type to code: pure → Hypothesis; FastAPI → `httpx.AsyncClient`+`ASGITransport`; LangGraph → `agentevals`+`InMemorySaver`; RAG step → Q-A from `evals/datasets/`. Run `uv run pytest -x -k <new>` and SHOW the failure.
2. **Green.** Minimum implementation. No extras. Run test. Show it passes.
3. **Refactor.** Clean only. Run full suite. Show it still passes.
4. **Commit.** `feat(<area>): <one-line> + test`

## Gotchas

- Don't stub functions just to make tests pass — fake green.
- LLM outputs: assert structure/property, never literal strings.
- If test needs >50 lines of fixtures, design is wrong — stop and ask.
- Record external API calls with `@pytest.mark.vcr()` from first run.
- One test, one behavior. Test names with "and"/"or" split into two.
```

`✓ Phase 4 complete (X created, Y skipped)` checkpoint.

---

## Phase 5 — Project dependencies (uv)

Install the universal testing stack, then ONLY the eval framework for the detected project type.

```bash
# Universal — install for every project
uv add --dev pytest pytest-asyncio pytest-cov pytest-recording httpx \
            hypothesis schemathesis syrupy freezegun pytest-mock respx \
            ruff mypy
```

Then pick ONE based on Phase 2 detection:

```bash
# LangGraph agent
uv add --dev langsmith agentevals

# RAG
uv add --dev ragas

# CV / YOLO
uv add --dev pycocotools imagehash supervision

# Fine-tuning
uv add --dev lm-eval

# Plain backend — skip eval framework
```

`✓ Phase 5 complete (deps installed)` checkpoint.

---

## Phase 6 — Ready-to-work report (REQUIRED FORMAT)

Print EXACTLY this format. Do not improvise. If a section is empty, write `(none)`. Do not skip the box characters.

```
✅ Ready to work.

Project type detected: <one of: LangGraph agent | RAG | CV/YOLO | Fine-tuning | Plain backend>
                       <reason: e.g., "found langgraph in pyproject.toml and src/app/agents/">

MCPs:
  Phase 1a (user scope):     added: <list> | skipped: <list>
  Phase 1b (project scope):  added: <list> | skipped: <list>
                             writes to .mcp.json — commit this file

Plugins (user scope):        added: <list> | skipped: <list>

Files:
  Created:                   <list>
  Skipped (existed):         <list>
  Gitignored:                .claude/settings.local.json, .env

Dependencies installed:      <list of packages added in this phase>

Status:
  /context used:             NN%
  pytest:                    PASS / FAIL / NOT RUN (N tests)

Next command:                <ONE specific actionable command>
```

End of spec. Do not output anything after the report.
