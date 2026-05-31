---
title: "How I Set Up Claude Code on Every AI/ML Project"
author: "Suraj Jaiswal"
date: "2026-05-31"
date-modified: "2026-05-31"
categories: [Agentic AI, Claude Code, MCP, LLM]
description: "A field-tested, verified setup for Claude Code with proper user/project/local scope isolation: every package name checked against npm/PyPI/GitHub, real cost numbers for LLM eval tools, MCP context-tax measurements, permission denylists, and project-specific patterns for FastAPI, LangGraph, RAG, fine-tuning, and computer vision. Includes self-applying bootstrap with copy-paste init prompts."
image: images/claude_logo_img.png
toc: true
toc-depth: 3
toc-expand: 2
---

> **For humans:** Skim the TL;DR. Jump to whichever tier you need. If you just want to set up a new project, paste the [one-line init prompt](#init-prompts-copy-paste) — it points to a separate minimal spec file (`claude-setup.md`) optimized purely for Claude to execute.
> **For Claude Code (if a user pointed you at this URL):** Don't execute this post — execute the linked **setup spec** at `https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md` instead. This post is human rationale; the spec is the executable version. If for some reason you must use this post, read [§ Bootstrap for Claude](#bootstrap-for-claude).

This post is the rationale behind a setup I run on every new AI/ML project. The actual executable spec lives at **[claude-setup.md](https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md)** — ~465 lines of pure Claude-first instructions. This post (~1,300 lines) is the *why*: the cost numbers, the verified package names, the testing patterns, the real anecdotes from my work at Tiger Analytics, NeuroReef Labs, and IIT Gandhinagar. Read the post if you want to understand the choices. Use the spec if you want to set up a project.

I've spent the last six months iterating across multi-agent orchestrators at Tiger Analytics, RAG pipelines for medical billing at NeuroReef Labs, and the satellite-imagery YOLO-OBB pipeline from my IIT Gandhinagar research. Every package name verified, every cost number sourced, every threshold cited.

**Changes in v4:** proper scope isolation — generic tools (Context7, Memory, Sequential Thinking, Filesystem) install at **user scope**; project-specific MCPs (Postgres, AWS, Langfuse, Sentry, Playwright, GitHub-with-project-token) install at **project scope** and are committed to `.mcp.json` so teammates inherit them; secrets and personal overrides go in gitignored **local scope**. Default `permissions.deny` and `permissions.ask` blocklists are now part of the project `.claude/settings.json` to neutralize MCPs you have installed but don't want auto-called. Init prompts are unchanged at the surface but now route servers to the right scope.

---

## TL;DR

- **Bootstrap is now 404-free, project-type-aware, and scope-correct.** All MCP install commands verified against npm/PyPI/GitHub on 27 May 2026. The bootstrap detects project type (LangGraph / RAG / CV / fine-tuning / plain backend) from the repo and installs only what's needed. **Generic MCPs go to user scope; project-specific ones (Postgres, AWS, Langfuse, Sentry, Playwright, project-scoped GitHub) go to project scope** (committed to `.mcp.json`) so teammates reproduce your setup. The earlier `@aws/mcp-server`, `@langfuse/mcp`, `@sentry/mcp` names were hallucinations — corrected to `awslabs.*` uvx packages, Langfuse's HTTP endpoint, and `@sentry/mcp-server`.
- **Treat testing as Tier 0**, not Phase 2. LLM apps drift silently; agent loops desync; RAG retrieval degrades. Default stack: `pytest + pytest-asyncio + httpx.AsyncClient + Hypothesis + syrupy + schemathesis + pytest-recording` underneath, with ONE eval tool on top picked by the [decision tree below](#which-eval-tool-pick-one-from-the-decision-tree).
- **The biggest hidden cost is MCP context tax, not API tokens.** The official GitHub MCP injects ~42K–55K tokens of tool definitions before your first prompt; Playwright sessions balloon to ~114K. If `/context` shows >25% used at session start, [prune servers](#context-hygiene-what-to-delete) before doing anything else. The `permissions.ask` / `permissions.deny` block in `.claude/settings.json` is your second line of defense — installed but unused MCPs can be denied without uninstalling.
- **A 200-trace × 4-metric × nightly Ragas eval costs ~$15/month** on GPT-4o-mini. There is no excuse for not running it.

---

## Scope cheat-sheet — where does each thing live?

Claude Code supports four scopes. Getting them right is the difference between a setup that survives the team and one that lives only on your laptop.

| Scope | File location | Committed? | What lives here |
|---|---|---|---|
| **User** | `~/.claude.json`, `~/.claude/` | No | Generic things every project benefits from |
| **Project** | `<repo>/.mcp.json`, `<repo>/.claude/` | **Yes** | Project-specific config teammates must inherit |
| **Local** | `<repo>/.claude/settings.local.json` | No (gitignored) | Personal overrides, secrets, per-machine experiments |
| **Enterprise** | OS-specific managed path | N/A | Org policy; rare for individuals |

### The placement rule I use

| Type | Scope | Reasoning |
|---|---|---|
| Context7, Memory, Sequential Thinking, Filesystem | **User** | No project-specific config; benefits every session |
| Plugins (`superpowers`, `feature-dev`, `coderabbit`, `code-simplifier`, `pr-review-toolkit`, `pyright-lsp`, `claude-md-management`, `ralph-loop`, `code-review`) | **User** | Workflow-style; helps you code regardless of stack |
| Personal `CLAUDE.md` (your name, your default conventions) | **User** at `~/.claude/CLAUDE.md` | Cross-project preferences |
| GitHub MCP (project-token version) | **Project** | Pin to this repo's scoped token, not your personal one |
| Postgres MCP | **Project** | Uses `$DATABASE_URL` for *this* DB |
| AWS API + AWS Docs MCP | **Project** | Uses *this* project's AWS profile |
| Langfuse MCP | **Project** | Uses *this* project's keys |
| Sentry MCP | **Project** | Uses *this* project's org/project DSN |
| Playwright MCP | **Project** | Only present in projects with browser tests; otherwise the 114K-token tax fires for nothing |
| `CLAUDE.md` (project rules, stack, "Don't") | **Project** | Teammates need it |
| `.claude/settings.json` (hooks, permissions) | **Project** | Teammates need the same hooks firing |
| `.claude/agents/`, `.claude/skills/`, `.claude/hooks/` | **Project** | References project-specific paths |
| Personal API keys, auto-approves | **Local** | Never in git |
| Experimental MCP you're trying before promoting | **Local** | Easy to remove |

### Stopping an installed MCP from being called

If a project-level MCP is installed but you don't want Claude to auto-call it this session, four mechanisms in order of escalation:

1. **Don't install it.** Strongest lever. Project-type detection (next section) is designed to keep Playwright out of non-browser projects entirely.
2. **Disable via settings:**
   ```json
   // .claude/settings.local.json
   {"disabledMcpjsonServers": ["playwright", "aws-api"]}
   ```
3. **Per-tool permission rules in `.claude/settings.json`:**
   ```json
   {"permissions": {
     "ask":  ["mcp__playwright__*", "mcp__github__create_*"],
     "deny": ["mcp__github__delete_*", "mcp__postgres__execute_sql"]
   }}
   ```
4. **Tool-search (default-on since 14 Jan 2026)** dynamically loads only relevant tools per prompt, reducing the always-on schema tax. Defense in depth, not a substitute for the above.

The default `.claude/settings.json` in this post's scaffold ships with sensible `permissions.ask` and `permissions.deny` lists already wired (see §11.5).

---

## Init prompts — copy-paste

**Save this one line to your text expander as `;claudeinit`.** It's the only prompt you need. Claude reads the spec, detects project type, installs only what's needed, and ends with a structured "ready to work" report.

```
Read https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md and execute it on this project.
```

That's it. The spec at that URL contains everything Claude needs — scope rules, project-type detection, scaffold templates, permissions, hooks. No second prompt required.

**Override the detected project type** if it gets it wrong:

```
Read https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md and execute it. Treat this
project as: <RAG | LangGraph agent | FastAPI backend | CV/YOLO | Fine-tuning>.
```

**Scaffold files only** (skip MCPs and plugins, useful when joining an existing project):

```
Read Phase 4 of https://jaiswalsuraj487.github.io/blogs/blogsData/claude-setup.md and create just
the scaffold files. Skip Phase 1a, 1b, 3, and 5.
```

---

## Bootstrap for Claude

> *Claude — if a user pointed you at this URL, execute this section in order. **Every step is idempotent**: before installing an MCP server, run `claude mcp list` and skip if present. Before installing a plugin, run `/plugin list` and skip if present. Before creating a file, check if it exists and ask before overwriting. **Scope matters**: generic MCPs use `--scope user`; project-specific MCPs use `--scope project` (writes to `<repo>/.mcp.json`) so teammates inherit them.*

### Project-type detection

Before installing anything, detect the project type. Inspect the repo for these signals and match to one of the five types below. If multiple signals fire, pick the dominant one. If nothing matches (empty or unclear repo), ask the user **one** question with the project types as buttons.

| Signals in the repo | Project type | Phase 1b servers (project scope) | Phase 4 eval framework |
|---|---|---|---|
| `langgraph` / `langchain` in deps, `agents/` or `graph.py` files | **LangGraph agent** | langfuse (optional), sentry (optional) | `langsmith agentevals` |
| `chromadb` / `pgvector` / `qdrant` / `weaviate` in deps; `rag/` or `retriever.py` files | **RAG** | postgres, langfuse (optional), sentry (optional) | `ragas` |
| `ultralytics` / `torch` + `opencv` / image dirs (`*.png`, `*.tif` fixtures) | **CV / YOLO** | aws-api + aws-docs (if S3 datasets), sentry (optional) | `pycocotools imagehash supervision` |
| `transformers` + a training script (`train.py`, `*.lora.py`, HF Trainer usage) | **Fine-tuning** | aws-api + aws-docs (if S3 checkpoints) | `lm-eval` |
| `fastapi` / `uvicorn` in deps, no LLM deps | **Plain backend** | postgres (if DB used), sentry (optional) | *(skip)* |
| Empty repo or none of the above | **Ask the user** | — | — |

**Note on GitHub MCP scope:** if the project has its own scoped GitHub token (deploy key, fine-grained PAT for *this repo only*), install GitHub MCP at **project scope** referencing that token. Otherwise the user-scope install from Phase 1a is fine for personal projects.

### Phase 0 — Sanity checks

```bash
claude --version          # confirm Claude Code installed
node --version            # need >= 20
uv --version              # need uv installed (`pipx install uv`)
which docker              # GitHub MCP runs via docker
which jq                  # used by the pytest hook
```

If any are missing, stop and tell the user to install them.

### Phase 1a — Universal MCP servers (user scope, install once per machine)

These five fit every project, are cheap on context budget, and pay for themselves immediately. **Install with `--scope user`** so they're available everywhere. Skip any already in `claude mcp list --scope user`.

```bash
# 1. Context7 — live library docs (Upstash). Anonymous works for personal use;
# get a free API key at context7.com if you hit rate limits.
claude mcp add --scope user context7 -- npx -y @upstash/context7-mcp

# 2. GitHub — official binary (ghcr.io). The npm @modelcontextprotocol/server-github
# is DEPRECATED. The -e flag passes the env var through to docker.
# Requires $GITHUB_PAT in your shell.
claude mcp add --scope user github -e GITHUB_PERSONAL_ACCESS_TOKEN=$GITHUB_PAT -- \
  docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN \
  ghcr.io/github/github-mcp-server

# Alternative if Docker isn't available — use the remote HTTP transport:
# claude mcp add --transport http github https://api.githubcopilot.com/mcp \
#   -H "Authorization: Bearer $GITHUB_PAT"

# 3. Filesystem — scoped to your code directory
claude mcp add --scope user fs -- npx -y @modelcontextprotocol/server-filesystem "$HOME/code"

# 4. Memory (personal knowledge graph across sessions)
claude mcp add --scope user memory -- npx -y @modelcontextprotocol/server-memory

# 5. Sequential thinking (structured reasoning for hard problems)
claude mcp add --scope user seq -- npx -y @modelcontextprotocol/server-sequential-thinking
```

### Phase 1b — Conditional MCP servers (project scope, install per project)

Look at the Project-Type Detection table above. Install ONLY the servers listed for the detected type. **Use `--scope project`** so they're committed to `<repo>/.mcp.json` and teammates pick them up on `git pull`. Skip any already in `claude mcp list --scope project`.

```bash
# Postgres — install for RAG (pgvector) or plain-backend projects with a DB.
# The original Anthropic Postgres MCP was ARCHIVED 29 May 2025 after a SQL-injection
# bypass disclosure. Use Postgres MCP Pro from CrystalDB.
# Defaults to READ-ONLY (safer for AI); add --access-mode=unrestricted if you need writes.
claude mcp add --scope project postgres -- uvx postgres-mcp "$DATABASE_URL"
# Or for read+write:
# claude mcp add --scope project postgres -- uvx postgres-mcp --access-mode=unrestricted "$DATABASE_URL"

# Browser automation — pick ONE, not both. They overlap.
#
# Option A: Playwright MCP (Microsoft, most mature, ~114K tokens of context tax in full sessions)
claude mcp add --scope project playwright -- npx -y @playwright/mcp@latest
#
# Option B: agent-browser (Vercel Labs, Rust CLI, claims 93% smaller context footprint).
# Younger project (Jan 2026), 23K+ stars, installed as a SKILL not an MCP:
# npm install -g agent-browser
# agent-browser install
# npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y

# AWS API + Docs — install for CV/fine-tuning projects using S3, or AWS deploy work.
# There is NO @aws/mcp-server. AWS publishes uvx servers under awslabs.*
claude mcp add --scope project aws-api -- uvx awslabs.aws-api-mcp-server@latest
claude mcp add --scope project aws-docs -- uvx awslabs.aws-documentation-mcp-server@latest

# Langfuse — install for LangGraph or RAG projects with production observability.
# No @langfuse/mcp npm package. The supported surface is HTTP:
claude mcp add --scope project --transport http langfuse \
  https://cloud.langfuse.com/api/public/mcp \
  --header "Authorization: Basic $(printf '%s:%s' "$LF_PUBLIC_KEY" "$LF_SECRET_KEY" | base64)"

# Sentry — install for any production app needing error tracking.
claude mcp add --scope project --transport http sentry https://mcp.sentry.dev/mcp
```

Verify with `claude mcp list --scope project`. The `.mcp.json` file should now be in the repo root and ready to commit.

**Secrets warning:** `.mcp.json` is committed. Reference env vars (`$DATABASE_URL`, `$LF_PUBLIC_KEY`) — never bake real secrets into the command. Teammates set their own env vars from a `.env.example` you also commit.

### Phase 2 — Plugins (user scope, run inside Claude Code)

Plugins are workflow-style tools — they help you code regardless of stack. **All install at user scope** (they're available everywhere). Skip any already in `/plugin list`. All of these are verified in the official `claude-plugins-official` marketplace.

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

# superpowers is in a separate marketplace (Jesse Vincent's). Adds workflow slash
# commands like /brainstorming, /tdd, /systematic-debugging. Adding any third-party
# marketplace is a trust decision — plugins can register hooks that intercept tool calls.
/plugin marketplace add obra/superpowers
/plugin install superpowers@obra-superpowers

/reload-plugins
```

If `ralph-loop` install fails, try `ralph-wiggum@claude-plugins-official` — some sources document the plugin under that name. `pyright-lsp` has a known config-propagation bug on Windows (anthropics/claude-code#16219).

If you build a custom plugin for *this specific project* (rare — usually a project-specific subagent or skill works better), install it at project scope instead.

### Phase 3 — Project scaffold (project scope, committed to git)

Create these files in the current working directory (templates in §[10](#the-claude.md-template) and §[11](#the-four-files-to-drop-into-.claude)). **If any file already exists, ask the user before overwriting.** All of these are project-scope and meant to be committed.

```
CLAUDE.md                                    # §10 — customize stack section from detected repo
.claude/settings.json                        # §11.5 — includes default permissions.ask/deny
.claude/hooks/pytest-changed.sh              # §11.4 — chmod +x after creating
.claude/agents/test-writer.md                # §11.1
.claude/agents/eval-runner.md                # §11.2
.claude/skills/tdd-loop/SKILL.md             # §11.3
.env.example                                 # if Phase 1b installed any MCP referencing env vars
```

Append the following to `.gitignore` (create it if missing):

```
.claude/settings.local.json
.env
```

**Commit `.mcp.json` (created by Phase 1b) and everything in `.claude/` *except* `settings.local.json`.** Teammates running `git pull && claude` will pick up your project-scope MCPs automatically (Claude Code will prompt them to approve each one — they're untrusted by default for security).

### Phase 4 — Project dependencies (uv)

Install the base testing stack universally, then add ONE eval framework based on the detected project type from the table above.

```bash
# Universal — install for every project
uv add --dev pytest pytest-asyncio pytest-cov pytest-recording httpx \
            hypothesis schemathesis syrupy freezegun pytest-mock respx \
            ruff mypy

# Project-type-specific (pick ONE based on detection):
#   LangGraph agent → uv add --dev langsmith agentevals
#   RAG             → uv add --dev ragas
#   CV / YOLO       → uv add --dev pycocotools imagehash supervision
#   Fine-tuning     → uv add --dev lm-eval
#   Plain backend   → (skip eval framework; locust optional for load testing)
```

### Phase 5 — Ready-to-work report (required format)

Print this exact format when finished. Do not skip any line; if a section is empty, write "(none)".

```
✅ Ready to work.

Project type detected: <one of: LangGraph agent | RAG | CV/YOLO | Fine-tuning | Plain backend>
                       <reason: e.g., "found langgraph in pyproject.toml and src/app/agents/">

MCPs:
  Phase 1a (user scope):     <added: X> | <skipped: Y already present>
  Phase 1b (project scope):  <added: X> | <skipped: Y>
                             <writes to .mcp.json — commit this file>

Plugins (user scope):        <added: X> | <skipped: Y already present>

Files:
  Created:                   <list>
  Skipped (existed):         <list>
  Gitignored:                .claude/settings.local.json, .env

Dependencies installed:      <list of packages added in Phase 4>

Status:
  /context used:             NN%
  pytest:                    PASS / FAIL / NOT RUN (N tests)

Next command:                <ONE specific command — e.g., "@test-writer write tests for src/app/main.py"
                                                            or "edit CLAUDE.md and fill the <NAME> placeholder">
```

End of bootstrap. The rest of this post is the human rationale and reference material.

---

## 1. The five extension points

Most online confusion comes from blurring these. The "Lives in" column shows the typical scope; many can live at either user or project scope (see the [Scope cheat-sheet](#scope-cheat-sheet-where-does-each-thing-live) for the placement rule).

| Concept       | What it is                                                                      | Typical location                                  | Invoked by                                |
| ------------- | ------------------------------------------------------------------------------- | ------------------------------------------------- | ----------------------------------------- |
| **MCP server**| External process exposing tools (DB, browser, GitHub, docs) over a protocol     | `~/.claude.json` (user) or `<repo>/.mcp.json` (project) | Claude calls automatically, or `@mcp` |
| **Skill**     | A markdown playbook (`SKILL.md` + assets). Auto-loaded on description match     | `<repo>/.claude/skills/` (project) or `~/.claude/skills/` (user) or plugin | Auto-invoked or `/skill-name` |
| **Subagent**  | Specialized sub-Claude with its own tools, model, and prompt                    | `<repo>/.claude/agents/` (project) or `~/.claude/agents/` (user) or plugin | `@agent-name` or auto-delegation |
| **Hook**      | Deterministic shell command on lifecycle events (PreToolUse, PostToolUse, Stop) | `<repo>/.claude/settings.json` (project, almost always) | Claude Code runtime |
| **Plugin**    | A versioned bundle of any of the above                                          | `~/.claude/plugins/` (user, almost always)       | `/plugin install …`                       |
| **Plugin**    | A versioned bundle of any of the above                                          | `~/.claude/plugins/`           | `/plugin install …`                       |

Rule of thumb:

- Need data from outside the repo? → **MCP server**
- Need to encode "the way we do X here"? → **Skill** or **CLAUDE.md**
- Need a focused worker with its own context budget? → **Subagent**
- Need *something* to always happen (format, lint, scan secrets)? → **Hook**
- Need to share all of the above with teammates? → **Plugin**

CLAUDE.md sits above all of this — the always-loaded preamble that turns Claude into *your* project's Claude.

---

## 2. Tier 0 — Testing infrastructure (the part most teams skip)

**Opinion:** testing in AI projects is foundational, not Phase 2. LLM outputs are probabilistic, agent loops drift, RAG retrieval degrades silently, fine-tuned models regress on held-out sets — and none of that is caught by a happy-path `pytest`.

### 2.1 General Python base

| Concern              | Default                          | Why this not the others                                                  |
| -------------------- | -------------------------------- | ------------------------------------------------------------------------ |
| Test runner          | **pytest**                       | Universal; everything else plugs into it                                 |
| Coverage             | **pytest-cov** (≥80% gate)       | Matches CI defaults; arbitrary but anchored                              |
| Mocking              | **pytest-mock**                  | Pre-installed muscle memory                                              |
| HTTP mocking         | **respx** (httpx) / **responses**| First-class async support                                                |
| Time freezing        | **freezegun**                    | No competitor                                                            |
| Property-based       | **Hypothesis**                   | The only serious property-based tool in Python                           |
| Snapshot/golden      | **syrupy**                       | pytest-native, zero-dep, idiomatic                                       |
| Lint/format          | **ruff** + **ruff format**       | Replaces flake8/black/isort                                              |
| Type check           | **mypy** or **pyright**          | Pyright when you want the Claude Code `pyright-lsp` plugin               |

### 2.2 FastAPI / async — one pattern that matters

Use `httpx.AsyncClient` with `ASGITransport`, not the sync `TestClient`. This is FastAPI's own recommended pattern:

```python
# conftest.py
import pytest
from httpx import ASGITransport, AsyncClient
from app.main import app

@pytest.fixture
async def async_client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac

# tests/test_health.py
@pytest.mark.asyncio
async def test_health(async_client):
    r = await async_client.get("/health")
    assert r.status_code == 200
```

If you use FastAPI lifespan events (DB pools, model loads), wrap with `asgi-lifespan.LifespanManager`. `AsyncClient` doesn't trigger lifespan on its own.

### 2.3 Contract / fuzz testing

**Default: Schemathesis** against FastAPI's auto-generated OpenAPI schema. One file, hundreds of generated cases:

```python
import schemathesis
schema = schemathesis.from_asgi("/openapi.json", app)

@schema.parametrize()
def test_api_contract(case):
    response = case.call_asgi()
    case.validate_response(response)
```

Run with `--checks all --workers 8 --stateful=links` in CI.

### 2.4 Record/replay for LLM calls

**Default: `pytest-recording`** (VCR.py-based). Record once with `--record-mode=once`, commit the YAML cassette, replay forever. Deterministic, free, fast:

```python
@pytest.mark.vcr()
def test_summarize_paper():
    out = summarize("Attention is all you need...")
    assert "transformer" in out.lower()
```

Filter secrets in `conftest.py`:

```python
@pytest.fixture(scope="module")
def vcr_config():
    return {"filter_headers": ["authorization", "x-api-key"]}
```

### 2.5 Snapshot tests for LLM outputs

Lock in output shape without enforcing literal strings:

```python
from syrupy.matchers import path_type
from datetime import datetime

def test_extract_entities(snapshot):
    result = extract_entities("John bought $100 of NVDA on 2024-01-15.")
    assert result == snapshot(matcher=path_type({"timestamp": (datetime,)}))
```

Regenerate with `pytest --snapshot-update` after intended changes. The `.ambr` files diff cleanly in PR reviews.

### 2.6 Property-based testing — the highest ROI test you're not writing

Hypothesis for any pure function dealing with numerical, string, or schema-shaped input: tokenizers, chunkers, parsers, retrievers, image transforms. One test, hundreds of inputs:

```python
from hypothesis import given, strategies as st

@given(st.text(min_size=1, max_size=10_000))
def test_chunker_roundtrip(text):
    chunks = chunk_text(text, max_tokens=128)
    assert "".join(c.text for c in chunks) == text
    assert all(c.token_count <= 128 for c in chunks)
```

### 2.7 Load testing for AI APIs

Vanilla Locust/k6 lie to you about streaming endpoints because they conflate request duration with TTFT. Use **llm-locust** (TrueFoundry) or **NVIDIA GenAI-Perf** when latency matters to an SLA. For general API load, plain **Locust** is fine.

---

## 3. Which eval tool? Pick one from the decision tree

I've cross-read Hamel Husain's *Your AI Product Needs Evals*, Eugene Yan's *Patterns for Building LLM-based Systems*, and LangChain's *Evaluating Deep Agents*. They converge on this:

```
Q1 → Does your system retrieve documents into the prompt?
       YES → Ragas. Add Phoenix or Langfuse for trace visualization.
       NO  → Q2

Q2 → Multi-step agent with tool calls, state, or interrupts?
       YES → LangSmith (free Developer tier) + LangChain pytest integration + agentevals
       NO  → Q3

Q3 → Plain LLM call(s) with assertions or LLM-as-judge?
       → DeepEval (best pytest integration) or Promptfoo (red-team / PR-CI gate)
```

**Start with ONE.** Layer the others only when you hit the threshold that justifies them. Below is when to add each:

| Add this tool | When the threshold flips |
| ---- | ---- |
| **Inspect AI** | Anything safety-sensitive (medical, legal, financial advice). 133 pre-built evals across reasoning, scheming, safeguards. |
| **Braintrust** | ≥3 prompt versions in production AND non-engineers (PMs, domain experts) need to annotate traces. |
| **Langfuse** | Production traces exceed the free tier of whatever observability you started with. OSS Apache-2.0 self-host is the long-run economics. |
| **Inspect Evals** | You need foundation-model regression at the scale of a leaderboard (133 evals from UK AISI). |
| **lm-evaluation-harness** | Fine-tuning. The harness powers HuggingFace's Open LLM Leaderboard. |

**Philosophical anchor (Hamel Husain):** *"The abuse of generic metrics is endemic. Many eval vendors promote off-the-shelf metrics, which ensnare engineers into superfluous tasks."* Manually label 20–50 traces of your own application first. Your evals should emerge from observed failure patterns, not vendor defaults.

### Real cost data (verified 27 May 2026)

A 200-trace × 4-metric × nightly eval:

| Tool | Bill | Notes |
|---|---|---|
| **Ragas** | $5–$15/month on GPT-4o-mini judge | OSS library free; judge LLM only |
| **DeepEval** | $5–$25/month | OSS + free cloud tier |
| **LangSmith** | $0 (Developer) → $39+/seat (Plus) | 5K traces free; 200 traces × 30 days = 6K, fits free tier |
| **Braintrust** | $0 (Starter) | 1M trace spans, 1 GB data, 10K scores, 14-day retention, unlimited users |
| **Langfuse** | Self-host: infra only | OSS Apache 2.0 |
| **Inspect AI** | $0 + judge tokens | UK AISI, fully open-source |
| **Promptfoo** | $0 + judge tokens | MIT, free except Enterprise SSO/RBAC |

**Two findings that change behavior:**

1. A nightly Ragas eval on GPT-4o-mini costs less than one engineer-hour per month. Run it.
2. LangSmith economics flip past 5 devs × 200K traces/month (~$670/month before extended retention). At that scale, Langfuse self-host wins.

---

## 4. Verified MCP servers — what each does

Already in [§ Bootstrap](#bootstrap-for-claude). What follows is *why* each is here. The split between universal and conditional matches Phase 1a / Phase 1b above.

### Phase 1a — user scope (install for every project)

- **Context7** — pulls live docs for LangChain/LangGraph/FastAPI/etc. AI library APIs change weekly; your training-data knowledge is already stale.
- **GitHub** — opens PRs, comments on issues, reviews diffs, triggers workflows. The official Go binary; the deprecated npm package shows "no longer supported." Personal token at user scope; for repos with their own scoped token (deploy keys, fine-grained PATs), re-install at project scope.
- **Filesystem** — scoped access across multiple directories (monorepos, shared docs).
- **Memory** — personal knowledge graph: "auth uses Cognito, embeddings live in pgvector, eval set is `eval/v3.jsonl`" survives sessions. User-scoped because it's personal.
- **Sequential thinking** — structured reasoning for architecture decisions and tricky debugging.

### Phase 1b — project scope (install only when the project needs them, committed to `.mcp.json`)

- **Postgres (Pro)** — for RAG/pgvector or plain-backend projects with a DB. The original Anthropic Postgres MCP was archived after a SQL-injection bypass disclosed by Datadog Security Labs on 29 May 2025; the unpatched npm package still served ~21K weekly downloads. Use Postgres MCP Pro from CrystalDB. Reference `$DATABASE_URL` so teammates supply their own.
- **AWS API + AWS Docs** — for Bedrock/ECR/S3/IoT workflows. Use a read-only IAM role for the API server. Install for CV/fine-tuning projects using S3 datasets or any AWS deploy work.
- **Langfuse** — production LLM observability for LangGraph or RAG projects. The Langfuse team ships a native HTTP MCP at their hosted endpoint, not an npm package. Reference `$LF_PUBLIC_KEY` / `$LF_SECRET_KEY`.
- **Sentry** — general-purpose error tracking for any production app. Both remote (OAuth) and stdio supported.
- **Playwright** — browser automation for end-to-end testing. Heavy on context budget (~114K tokens in full sessions) — only enable when actively browser-testing.

### Security non-negotiables

Never point any MCP at production credentials. Use a read-only replica and a scoped service account. The Postgres-MCP CVE was a SQL-injection bypass via `COMMIT; DROP SCHEMA public CASCADE;` — write access turns Claude's hallucinations into incidents.

---

## 5. FastAPI backend project setup

- **Run/test/lint**: `uvicorn app.main:app --reload` / `uv run pytest -v` / `uv run ruff check .`
- **Async tests**: §2.2 pattern (`AsyncClient` + `ASGITransport`)
- **Contract tests**: Schemathesis at `tests/test_contract.py` (§2.3)
- **DB fixtures**: SQLAlchemy 2.x async with rollback after every test; SQLite in-memory or Dockerized Postgres via `pytest-postgresql`
- **Integration tests**: wire Postgres MCP, mark with `@pytest.mark.integration`, gate them in CI
- **Load tests**: `tests/load/locustfile.py` against staging; `--users 30 --spawn-rate 30 --run-time 120s` as the smoke baseline

---

## 6. LangGraph / multi-agent project setup

The canonical reference is LangChain's *Evaluating Deep Agents: Our Learnings* (blog.langchain.com/evaluating-deep-agents-our-learnings). Five patterns:

1. **Single-step evals** — `interrupt_before=["tools"]` snapshots state right before a tool runs. Like unit tests for agent actions.
2. **Full-turn trajectory evals** — assert a particular tool was called *at some point* in the trajectory, not necessarily in a fixed order.
3. **State assertions** — assert end-state side effects ("file written," "memory updated").
4. **Multi-turn simulation** — use an LLM user-simulator, not hardcoded turns (which desync after a deviation).
5. **Pytest integration** — `@pytest.mark.langsmith` runs assertions inside LangSmith experiments.

### A real bug this would have caught

On the multi-agent interview orchestrator I built at Tiger Analytics, we had a phase-routing bug in v0.4: when a candidate said "skip to behavioral" during the technical screen, the agent kept re-asking the technical question. A single-step LangGraph test with `interrupt_before=['tools']` asserting `phase_router(target_phase='behavioral')` would have caught it before staging. We didn't have it. We ate one production session.

The test we added that week:

```python
from langgraph.checkpoint.memory import InMemorySaver

def test_phase_routing_on_explicit_skip():
    graph = build_interview_graph().compile(checkpointer=InMemorySaver())
    state = {"messages": [
        HumanMessage("Tell me about a Python decorator."),
        AIMessage("Sure, can you explain how a Python decorator works?"),
        HumanMessage("Actually, skip to behavioral."),
    ]}
    # Interrupt right before the phase_router tool is called
    config = {"configurable": {"thread_id": "test-1"}, "interrupt_before": ["tools"]}
    result = graph.invoke(state, config)
    tool_call = result["messages"][-1].tool_calls[0]
    assert tool_call["name"] == "phase_router"
    assert tool_call["args"]["target_phase"] == "behavioral"
```

---

## 7. RAG pipeline setup

- **Eval framework**: Ragas. Headline metrics: `faithfulness`, `context_precision`, `context_recall`, `answer_relevancy`. Production-ready thresholds: faithfulness ≥ 0.9 (customer-facing), context_precision ≥ 0.8.
- **Dataset size**: start with 50 questions covering your real query distribution; grow to 200+ once you have production traces. Sample 1–5% of production into Langfuse, run nightly Ragas, write scores back as custom scores on the trace.
- **Separate retrieval from generation**: a "good" answer can mask broken retrieval; perfect retrieval can still hallucinate. Score independently.
- **CI gate**: DeepEval `assert_test` wrapping Ragas metrics gives you pytest-level pass/fail with RAG-specific scoring.

### A real test from medical-billing RAG

At NeuroReef Labs, the bug we feared was a hallucinated ICD-10 code. "MI" is ambiguous — myocardial infarction (I21) vs. mitral insufficiency (I34.0). The faithfulness test that would have caught it: 50 golden Q-A pairs from the coding team, Ragas `Faithfulness()` with GPT-4o-mini as judge, CI gate at `faithfulness ≥ 0.95`. Not 1.0 — judges occasionally mislabel correct retrievals as unfaithful when abbreviations are ambiguous.

---

## 8. Fine-tuning project setup

EleutherAI's `lm-evaluation-harness` powers HuggingFace's Open LLM Leaderboard. From their README: *"used in hundreds of papers, and used internally by dozens of organizations including NVIDIA, Cohere, BigScience, BigCode, Nous Research, and Mosaic ML."*

```bash
git clone --depth 1 https://github.com/EleutherAI/lm-evaluation-harness
cd lm-evaluation-harness && pip install -e .

# Base
lm_eval --model hf --model_args pretrained=meta-llama/Llama-3.1-8B \
  --tasks mmlu,hellaswag,arc_challenge,gsm8k,truthfulqa_mc2 \
  --batch_size 8 --output_path results/base.json

# Fine-tuned
lm_eval --model hf --model_args pretrained=./outputs/medbilling-lora \
  --tasks mmlu,hellaswag,arc_challenge,gsm8k,truthfulqa_mc2 \
  --batch_size 8 --output_path results/finetuned.json
```

**Regression gates** (practitioner heuristic from Open LLM Leaderboard noise floor):
- Each task: drop ≤ 3% absolute vs. base
- Aggregate: drop ≤ 1.5%
- Large MMLU gains usually signal **contamination**, not improvement. Investigate before celebrating.

---

## 9. Computer vision (YOLO / YOLO-OBB)

This section is the deepest because I published on it during my time at IIT Gandhinagar — the YOLO-OBB satellite-imagery pipeline that detected 30,000+ brick kilns across 5 Indian states (520,000 km² of coverage, 82% precision; *Eye in the Sky* at ACM Compass, *Space to Policy* at ACM JCSS, *Towards Scalable Identification* at NeurIPS Active Learning workshop).

### Metric defaults

Oriented-bounding-box mAP is **not** the same as COCO mAP. `pycocotools` assumes axis-aligned boxes; OBB IoU needs rotated polygons. Two canonical approaches:

- **Ultralytics built-in**: `yolo val obb data=DOTAv1.yaml`. Internally uses xywhr (center + width + height + rotation) and rotated-IoU.
- **Shapely-based custom IoU**: when validating outside Ultralytics, build `shapely.geometry.Polygon` from the four corners and compute `obb1.intersection(obb2).area / union_area`.

### Tiled inference for large satellite scenes

SAHI (Akyon et al., ICIP 2022) is the standard for slicing-based inference on satellite/medical imagery and now supports YOLO11-OBB. Roboflow's `supervision` library mirrors it via `sv.InferenceSlicer`. From the Ultralytics docs: *"By scaling object detection tasks across different image sizes and resolutions, SAHI becomes ideal for various applications, such as satellite imagery analysis and medical diagnostics."*

### Active-learning regression — the test that would have caught a real bug

When we expanded the kiln-detection pipeline from Uttar Pradesh to a new state, mean kiln density per tile dropped roughly 4×. The uncertainty-sampling step then over-selected near-empty tiles, and labeled-pool diversity collapsed. We caught it in QA, not in CI.

The regression test we wished we'd had: after each AL round, assert (a) entropy distribution of selected tiles has KL-divergence < threshold against the previous round, and (b) class-conditional mAP on a frozen held-out tile set from the new state doesn't drop more than 3% absolute vs. the prior checkpoint.

```python
import imagehash, supervision as sv
from PIL import Image
from hypothesis import given
from hypothesis.extra.numpy import arrays
from hypothesis.strategies import floats
import numpy as np

# 1. Fixture regression on a held-out tile via perceptual hash
def test_kiln_detection_fixture():
    ref_hash = imagehash.phash(Image.open("tests/fixtures/up_tile_42.png"))
    pred_overlay = render_predictions("tests/fixtures/up_tile_42.tif")
    assert (imagehash.phash(pred_overlay) - ref_hash) < 5  # Hamming distance

# 2. Rotation equivariance via Hypothesis
@given(arrays(np.float32, (256, 256, 3), elements=floats(0, 1)))
def test_rotation_equivariance(img):
    for k in (1, 2, 3):
        rotated = np.rot90(img, k=k, axes=(0, 1))
        assert len(model(img).obb) == len(model(rotated).obb)

# 3. mAP regression gate
def test_map_no_regression():
    metric = sv.metrics.MeanAveragePrecision().update(preds, targets).compute()
    assert metric.map50_95 >= 0.97 * BASELINE_MAP_50_95
```

**Production mAP thresholds (practitioner heuristic, not canonical):** mAP@0.5 ≥ 0.7 "usable", ≥ 0.85 "strong"; mAP@0.5:0.95 ≥ 0.5 "strong" for a single-domain custom dataset. No one publishes a universal cutoff; domain matters more than any number.

---

## 10. The CLAUDE.md template

`CLAUDE.md` loads automatically at session start. This is where the project's *rules of engagement* live. Drop this in the repo root:

```markdown
# Project: <NAME>

## What this is
<one paragraph — what it does, who uses it, what stack>

## Stack
- Language(s): Python 3.12, TypeScript 5
- Frameworks: FastAPI, LangGraph, React
- Infra: Docker, AWS (Bedrock, ECR, S3), Postgres + pgvector
- Observability: Langfuse, Sentry

## Commands
- Install: `uv sync`
- Run: `uvicorn app.main:app --reload`
- Test (fast): `uv run pytest -x --no-cov`
- Test (full): `uv run pytest --cov=app --cov-report=term-missing`
- Lint: `uv run ruff check . && uv run ruff format --check .`
- Type-check: `uv run mypy src/`
- Eval: `@eval-runner` (subagent)

## Repo layout
- `src/` — application code
- `tests/` — pytest suite (unit, integration, contract, load, cassettes)
- `evals/` — eval datasets and harness; do not modify without explicit ask
- `infra/` — Terraform + Dockerfile
- `.claude/` — subagents, skills, hooks (committed)
- `notebooks/` — exploratory; never imported by `src/`

## Testing rules (apply to production code)

These are the rules for code in `src/app/core/`, `src/app/agents/`, `src/app/rag/`,
`src/app/api/`. Exploratory prompt iteration uses `@eval-runner` instead of TDD.

1. **TDD for production code.** Use the `tdd-loop` skill for any feature touching the above paths.
2. **No live external API in CI.** Wrap LLM and HTTP calls with `@pytest.mark.vcr()`. Cassettes in `tests/cassettes/`.
3. **Async tests use `httpx.AsyncClient` + `ASGITransport`.** Never the sync `TestClient` for async endpoints.
4. **LLM outputs:** assert on structure (Pydantic), properties (length, contains, classifier scores), or `syrupy` snapshots with matchers. Never on literal strings.
5. **Coverage gate:** 80% line, 100% on `app/core/` and `app/agents/`.
6. **Property-based tests** (Hypothesis) for every pure utility — chunkers, parsers, retrievers, image transforms.
7. **Contract tests** (Schemathesis) on every PR against the live OpenAPI schema.
8. **Eval gate on prompt/model changes:** anything touching `app/prompts/`, `app/agents/`, `app/rag/` must run `@eval-runner` and pass thresholds in `evals/thresholds.yaml`.

## Conventions
- `uv add <pkg>` not `pip install`
- `uv run <cmd>` not bare `python`
- Avoid `print` — use `loguru` with structured fields
- Pydantic v2 syntax (`model_config`, `Annotated`)
- All public functions get type hints
- Typed exceptions in `app/exceptions.py`; FastAPI handler maps them to HTTP

## Don't
- Don't commit unless I explicitly say "commit".
- Don't run `terraform apply` — only `plan`.
- Don't touch `evals/datasets/` — frozen for regression comparison.
- Don't add top-level dependencies without asking.
- Don't bypass the `@pytest.mark.vcr()` decorator in any LLM-touching test.

## When in doubt
- Bug fix touching >2 files → `@code-architect` (feature-dev plugin) first
- Anything LLM-output-related → `@eval-runner` after the change
- Anything API-shape-related → `uv run schemathesis run ...` locally

## Useful commands
- `make dev` — start local stack (API + Postgres + Langfuse)
- `make eval` — run full eval suite
- `make deploy-staging` — CI handles this; do not run locally
```

The "Don't" section is the most underrated part. Be specific about what Claude should *refuse to do* without explicit confirmation.

---

## 11. The four files to drop into `.claude/`

### 11.1 `.claude/agents/test-writer.md`

```markdown
---
name: test-writer
description: Writes pytest unit, integration, and property-based tests for the file or function the user names. Generates fixtures, mocks external services with respx/responses, uses Hypothesis for pure functions. Returns a single test file and a one-line summary of coverage gaps.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
---
You are a senior Python test engineer. When invoked:

1. Read the target file and any sibling `tests/` patterns.
2. Detect the framework: FastAPI → `httpx.AsyncClient` + `ASGITransport`; LangGraph → `agentevals` trajectory matchers; Ragas pipeline → `ragas.evaluate`.
3. Generate tests in order: happy path, edge cases, error paths, Hypothesis properties for pure functions, snapshot tests (syrupy) for stable serialized outputs.
4. Mock external services: HTTP via `respx`, time via `freezegun`, LLM/HTTP via `pytest-recording` (`@pytest.mark.vcr()`).
5. Write the file. Run `uv run pytest -x` on it. If it fails, fix it.
6. Output a one-paragraph coverage summary and the paths NOT covered.

Never write tests that hit live external APIs without VCR cassettes.
Never assert on exact LLM output strings — use snapshot matchers or LLM-as-judge.
```

### 11.2 `.claude/agents/eval-runner.md`

```markdown
---
name: eval-runner
description: Runs the project's eval suite (DeepEval / Ragas / agentevals / lm-eval) and produces a structured pass/fail report with per-metric scores, deltas vs. last run, and a list of failing examples. Use after any prompt change, retrieval change, or model swap.
tools: Read, Grep, Bash, Edit
model: sonnet
---
You are the project's evaluation operator.

1. Detect eval framework from `pyproject.toml`/`requirements*.txt`:
   - `ragas`     → `uv run python -m evals.ragas_run`
   - `deepeval`  → `uv run deepeval test run evals/`
   - `agentevals`/`langsmith` → `uv run pytest evals/ -m langsmith`
   - `lm-eval`   → `uv run lm_eval --tasks <project_tasks> ...`
2. Capture JSON to `evals/runs/<timestamp>.json`.
3. Diff vs. `evals/runs/baseline.json`. Flag any metric regressing > 2%.
4. Print a markdown table: metric | baseline | current | delta | verdict.
5. Exit non-zero if any blocker metric regressed (read thresholds from `evals/thresholds.yaml`).

Never run live LLM evals on every CI step. Use cassettes + scheduled nightly for expensive metrics.
```

### 11.3 `.claude/skills/tdd-loop/SKILL.md`

```markdown
---
name: tdd-loop
description: Red-green-refactor loop for production code. Use when the user asks to add a feature, implement an endpoint, fix a bug, or refactor a function in src/app/core, src/app/agents, src/app/rag, or src/app/api. Forces a failing test first, then minimal code, then refactor. Triggers on "add", "build", "implement", "fix bug", "refactor" in production paths.
---

## When to invoke

Production-code feature work in `src/app/core/`, `src/app/agents/`, `src/app/rag/`, or `src/app/api/`. Skip for one-line prompt tweaks, exploratory notebooks, or research-only paths — those go through `@eval-runner`.

## Procedure

1. **Red.** Write the smallest possible failing test in `tests/` that captures the desired behavior.
   - Pure function → Hypothesis
   - FastAPI endpoint → `httpx.AsyncClient`
   - Agent step → `agentevals` trajectory
   Run `uv run pytest -x -k <new_test>` and SHOW that it fails.
2. **Green.** Write the minimum implementation to pass. No extras. Run the test. Show it passes.
3. **Refactor.** Clean the implementation only. Run the full suite. Show it still passes.
4. **Commit.** Stage test + implementation: `feat(<area>): <one-line> + test`.

## Gotchas

- Don't stub functions just to make tests pass — that's fake green.
- LLM outputs: assert on structure or property, never literal strings.
- If a test needs >50 lines of fixtures, the design is wrong. Stop and ask for redesign.
- Record external API calls with `@pytest.mark.vcr()` from the first run.
```

### 11.4 `.claude/hooks/pytest-changed.sh`

```bash
#!/usr/bin/env bash
# PostToolUse hook for Write|Edit|MultiEdit on *.py.
# Stdin = Claude Code event JSON. Exit 0 unless tests actively fail.
set -u
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
[[ -z "$FILE" ]] && exit 0
[[ "$FILE" != *.py ]] && exit 0

ROOT="$(git -C "$(dirname "$FILE")" rev-parse --show-toplevel 2>/dev/null)"
[[ -z "$ROOT" ]] && exit 0
REL="${FILE#$ROOT/}"
BASENAME=$(basename "$REL" .py)
DIRNAME=$(dirname "$REL")

if [[ "$BASENAME" == test_* || "$BASENAME" == *_test ]]; then
  TARGET="$REL"
else
  # Try src/foo/bar.py → tests/foo/test_bar.py, app/foo/bar.py → tests/foo/test_bar.py,
  # foo/bar.py → tests/test_bar.py
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

if [[ -z "$TARGET" ]]; then
  echo "[pytest-hook] No test file found for $REL — skipping." >&2
  exit 0
fi

cd "$ROOT"
timeout 30 uv run pytest "$TARGET" -x --tb=short --no-header -q 2>&1 | tail -20 >&2
exit 0   # never block the session; failing tests are signal, not error
```

Make executable: `chmod +x .claude/hooks/pytest-changed.sh`.

### 11.5 `.claude/settings.json`

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
          "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | grep -E \"(AKIA[0-9A-Z]{16}|sk-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36})\" && echo \"Possible secret\" && exit 2 || exit 0'"
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

The `permissions.deny` list neutralizes the most dangerous MCP tools without uninstalling the servers. The `permissions.ask` list forces a confirmation prompt for medium-risk operations — adjust per project. If a project genuinely needs `terraform apply` or `mcp__playwright__*` to run unattended, move those entries from `ask` to `allow` in **`.claude/settings.local.json`** (gitignored, personal-only) rather than the shared settings.

---

## 12. CI integration (GitHub Actions)

`.github/workflows/ai-pipeline.yml`:

```yaml
name: AI Pipeline Quality Gate
on: [pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
          cache-dependency-glob: "uv.lock"

      - run: uv sync --all-extras --frozen

      - name: Ruff lint
        run: uv run ruff check . --output-format=github

      - name: Mypy
        run: uv run mypy src/

      - name: Pytest with coverage gate
        run: uv run pytest -x --cov=src --cov-fail-under=80

      - name: Schemathesis (OpenAPI fuzz)
        run: |
          uv run uvicorn app.main:app &
          sleep 3
          uv run schemathesis run http://localhost:8000/openapi.json \
            --checks all --hypothesis-max-examples=50

      - name: Ragas nightly gate
        if: github.event_name == 'schedule'
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          uv run python evals/run_ragas.py \
            --gate-faithfulness 0.9 \
            --gate-context-precision 0.8
```

Caching `uv.lock` cuts cold-start substantially.

---

## 13. Context hygiene — what to delete

The biggest cost surprise in Claude Code is MCP context tax. Community-measured token usage:

| Server | Schema tokens | Source |
|---|---|---|
| GitHub (official) | **~42K–55K** | Nebulagg / Piotr Hajdas |
| Playwright (full session w/ snapshots) | **~114K** | jdhodges.com |
| Postgres MCP Pro | ~3,300 | ejae-dev/mcp-cost-calculator |
| Filesystem | 1,500–3,400 | mindstudio.ai |
| Memory | 1,000–2,000 | mindstudio.ai |
| Sentry (full) | 3,000–6,000 | docs.sentry.io |
| AWS API | 4,000–8,000 per server | fastmcp logs |

**Pruning rule:** if `/context` shows >25% used at session start, prune servers in this order:

1. **GitHub MCP** — use the Bash + `gh` CLI fallback for most operations
2. **Playwright** — only enable when actively browser-testing
3. Anything with >20 tools you're not using this session

**Per Anthropic's engineering blog** (24 Nov 2025): the `ENABLE_TOOL_SEARCH` feature delivers an **85% reduction in token usage** (from ~77K to ~8.7K) for typical many-server setups. Default-on rollout was 14 Jan 2026. Leave it on.

### Four ways to stop a server from being called (in order of escalation)

1. **Don't install it.** Use Phase 1b's project-type detection to keep Playwright out of non-browser projects entirely.
2. **Disable without uninstalling** via `.claude/settings.local.json` (personal, gitignored):
   ```json
   {"disabledMcpjsonServers": ["playwright", "aws-api"]}
   ```
3. **Permission rules in `.claude/settings.json`** (committed, shared with the team):
   ```json
   {"permissions": {
     "ask":  ["mcp__playwright__*"],
     "deny": ["mcp__github__delete_*"]
   }}
   ```
4. **Remove entirely** with `claude mcp remove`.

### Scope-aware removal commands

```bash
claude mcp remove <name> --scope user        # remove personal/global server
claude mcp remove <name> --scope project     # remove from .mcp.json (commit the change)
claude mcp remove <name> --scope local       # remove from .claude/settings.local.json
claude mcp list --scope project              # see what's project-scoped
/plugin uninstall <name>                     # marketplace plugins (user scope)
rm -rf .claude/skills/<name>/                # project-scope skill
rm -rf ~/.claude/skills/<name>/              # user-scope skill
```

User-scope MCPs live in `~/.claude.json`. Project-scope MCPs live in `<repo>/.mcp.json`. Local-scope overrides live in `<repo>/.claude/settings.local.json`. If you can't find an MCP after `claude mcp list`, try `claude mcp list --scope user` and `claude mcp list --scope project` — the default may show only the wrong scope depending on your version.

---

## 14. Production-ready thresholds (cited, not invented)

| Metric | Threshold | Source |
|---|---|---|
| Line coverage | ≥ 80% | Mainstream practice; anchored by `--cov-fail-under=80` defaults |
| Branch coverage | ≥ 70% | Same |
| Ragas faithfulness | ≥ 0.9 (customer-facing); ≥ 0.85 (internal) | Lushbinary RAG Production Guide 2026; CustomGPT |
| Ragas context precision | ≥ 0.8 | Same |
| Ragas answer relevancy | ≥ 0.85 | Same |
| LLM p95 TTFT (streaming) | < 1.0 s | Industry rule of thumb |
| LLM p99 full response | < 8 s (typical chat) | Same |
| Eval regression tolerance | ≤ 3% absolute per metric, ≤ 1.5% aggregate | Open LLM Leaderboard noise-floor heuristic |
| CV mAP@0.5 (single-domain) | ≥ 0.7 "usable", ≥ 0.85 "strong" | Practitioner heuristic; no canonical cutoff |
| CV mAP@0.5:0.95 | ≥ 0.5 "strong" for single-domain | Same |

Ragas docs explicitly say *"optimal thresholds vary by domain."* CV thresholds are practitioner consensus, not contracts.

---

## 15. The full ordering — how to actually use this stack

1. New feature lands in chat
2. Non-trivial work → `/brainstorming` (superpowers)
3. Production code path → `/tdd-loop` skill
4. `@test-writer` writes the failing test
5. Claude implements; PostToolUse hook surfaces failures inline
6. On Stop: ruff format + check + full pytest
7. If touched prompts/RAG/agents → `@eval-runner` runs Ragas/DeepEval/agentevals
8. `@code-reviewer` (feature-dev) reads the diff
9. `@coderabbit` second-opinion review (40+ static analyzers, AST + codegraph, respects CLAUDE.md)
10. CI: Schemathesis contract + Locust smoke on staging deploys

---

## 16. Replicating on a new machine

There are now two distinct workflows thanks to proper scope isolation. **For a fresh machine starting a new project**, ~10–20 minutes. **For a fresh machine joining an existing project** that already committed `.mcp.json` and `.claude/`, much less — most of the setup is in the repo.

### Workflow A: Fresh machine, brand new project

1. `brew install node uv gh jq docker && npm install -g @anthropic-ai/claude-code`
2. `gh auth login`
3. Open Claude Code in the (empty or fresh) project directory and paste **Prompt 1** from [§ Init prompts](#init-prompts-copy-paste).
4. Wait for the Phase 5 "Ready to work" report.
5. Commit `.mcp.json`, `CLAUDE.md`, `.claude/` (excluding `settings.local.json`), `.env.example`, and `.gitignore`.
6. Run the suggested "Next command" from the report.

Time breakdown:
- Prereqs (Node, uv, gh, docker, jq): ~5 minutes if Homebrew is cold
- MCP installs (5 user + 0–4 project): 2–6 minutes
- GitHub token + per-project env vars: 2–5 minutes
- First VCR cassette recording (one-time live LLM call): 2 minutes
- `uv sync` for project deps: 2–5 minutes

### Workflow B: Fresh machine, joining an existing project

1. `brew install node uv gh jq docker && npm install -g @anthropic-ai/claude-code`
2. `gh auth login`
3. `git clone <repo> && cd <repo>`
4. Copy `.env.example` to `.env` and fill in the secrets (DB URLs, API keys).
5. Open Claude Code. It detects `.mcp.json` and prompts you to approve each project-scope MCP — these are untrusted by default for security. Approve the ones you trust.
6. Optionally install the Phase 1a user-scope MCPs you don't already have (paste Prompt 1; idempotent, will skip everything project-level since it's already there).
7. `uv sync && uv run pytest` — confirm green.

Time: ~5–10 minutes since the project-scope stuff is already in the repo.

---

## 17. Caveats — what's not nailed down

- **Package versions move.** All names verified 27 May 2026 against the official `claude-plugins-official` marketplace.json (1,384 lines as of fetch) and npm/PyPI registries. Re-verify before publishing.
- **The "90% of Anthropic code is AI-written" figure is company-reported.** Traces to Dario Amodei's Dreamforce 2024 and Code with Claude event. No independent audit; treat as company statement, not measured fact.
- **The "6 RAG evals" attribution to Hamel Husain in earlier drafts could not be verified.** His actual framework is the three-level eval pyramid (assertions → judges → A/B) from *Your AI Product Needs Evals*. The "6 RAG evals" is likely a conflation with Jason Liu's RAG material or the Ragas metric set.
- **MCP token counts are community measurements, not Anthropic-published.** Order-of-magnitude only. Anthropic's published number is the 85% token-search reduction.
- **Production mAP thresholds are practitioner heuristics, not canonical.** No one at Ultralytics or COCO publishes a single number. Domain matters more than any threshold.
- **The Postgres MCP from Anthropic's reference repo remains vulnerable.** Even though npm still shows ~21K weekly downloads of `@modelcontextprotocol/server-postgres`, do not use it anywhere with write access. Use Postgres MCP Pro from CrystalDB.
- **Skill activation reliability is description-dependent.** Per Scott Spence's 200-prompt benchmark, vague descriptions get 20% activation; concrete trigger-phrase descriptions hit 80–84%. Treat 80–95% as a ceiling, not a guarantee.
- **The subagent `model:` frontmatter has a known bug** (GitHub issue #44385). Belt-and-braces: set `CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-5` in your shell.
- **Project-type detection is heuristic.** The table in the Bootstrap section pattern-matches on file/dep signals — it's right ~85% of the time on real repos but can miss hybrid projects (e.g., RAG-over-an-agent). If the detection looks wrong in Phase 5's report, override by re-running with **Prompt 2** and an explicit project type.
- **`.mcp.json` is untrusted by default.** When a teammate clones a repo with `.mcp.json`, Claude Code prompts them to approve each project-scope server before using it. This is by design — `.mcp.json` is executable code in the wrong hands. The flip side: the first time anyone (including you) runs Claude Code in a fresh clone, expect a few approval prompts. They're cached after the first yes.
- **`claude mcp list` default scope behavior varies by version.** Some releases show user + project; some show only one. When debugging "where did my MCP go," always pass `--scope user`, `--scope project`, and `--scope local` explicitly to see the full picture.
- **`disabledMcpjsonServers` is local-scope-only.** It's a personal disable list, not a shared one. To shut a project-scope MCP off for the entire team, remove it from `.mcp.json` and commit. To shut it off just for you, use `.claude/settings.local.json`.

---

## 18. Top 3 must-reads

If you only read three things:

1. **Hamel Husain — *Your AI Product Needs Evals*** (https://hamel.dev/blog/posts/evals/). The three-level pyramid (assertions → judges → A/B) is the mental model the field has converged on.
2. **Eugene Yan — *Patterns for Building LLM-based Systems & Products*** (https://eugeneyan.com/writing/llm-patterns/). Seven patterns — Evals, RAG, Fine-tuning, Caching, Guardrails, Defensive UX, Collect feedback — give you shared vocabulary with senior engineers everywhere.
3. **LangChain — *Evaluating Deep Agents: Our Learnings*** (https://blog.langchain.com/evaluating-deep-agents-our-learnings/). The single best treatment of trajectory, final-response, and state evaluation for multi-step agents.

**Bonus**: Anthropic's *How Anthropic teams use Claude Code* PDF (https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf).

---

## 19. References

**Official Anthropic**
- Claude Code best practices — https://code.claude.com/docs/en/best-practices
- Hooks guide — https://code.claude.com/docs/en/hooks-guide
- Subagents — https://code.claude.com/docs/en/sub-agents
- Skills overview — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Skill authoring best practices — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Anthropic engineering: How teams use Claude Code — https://claude.com/blog/how-anthropic-teams-use-claude-code
- `anthropics/claude-plugins-official` — https://github.com/anthropics/claude-plugins-official
- Advanced tool use (tool search) — https://www.anthropic.com/engineering/advanced-tool-use

**MCP servers (verified)**
- Context7 — https://www.npmjs.com/package/@upstash/context7-mcp
- GitHub MCP server — https://github.com/github/github-mcp-server
- Filesystem — https://www.npmjs.com/package/@modelcontextprotocol/server-filesystem
- Memory — https://www.npmjs.com/package/@modelcontextprotocol/server-memory
- Sequential thinking — https://www.npmjs.com/package/@modelcontextprotocol/server-sequential-thinking
- Postgres MCP Pro — https://github.com/crystaldba/postgres-mcp
- Playwright MCP — https://github.com/microsoft/playwright-mcp
- AWS Labs MCP — https://github.com/awslabs/mcp
- Langfuse MCP — https://langfuse.com/changelog/2025-11-20-native-mcp-server
- Sentry MCP — https://docs.sentry.io/product/sentry-mcp/

**LangGraph / agent testing**
- LangChain "Evaluating Deep Agents" — https://blog.langchain.com/evaluating-deep-agents-our-learnings/
- LangGraph test docs — https://docs.langchain.com/oss/python/langgraph/test
- `agentevals` — https://pypi.org/project/agentevals/
- LangSmith evaluation — https://www.langchain.com/langsmith/evaluation

**RAG / LLM eval**
- Ragas docs — https://docs.ragas.io/
- Ragas paper — https://arxiv.org/abs/2309.15217
- DeepEval — https://deepeval.com/
- Promptfoo — https://www.promptfoo.dev/
- Inspect AI (UK AISI) — https://inspect.aisi.org.uk/
- Inspect Evals (133 evals) — https://ukgovernmentbeis.github.io/inspect_evals/
- EleutherAI lm-evaluation-harness — https://github.com/EleutherAI/lm-evaluation-harness
- Braintrust — https://www.braintrust.dev/
- Langfuse — https://langfuse.com/
- Arize Phoenix — https://phoenix.arize.com/
- Pricing comparison — https://genai.qa/blog/promptfoo-vs-deepeval-vs-ragas/

**Engineers worth following**
- Hamel Husain — *Your AI Product Needs Evals* — https://hamel.dev/blog/posts/evals/
- Hamel Husain — *Evals FAQ* — https://hamel.dev/blog/posts/evals-faq/
- Eugene Yan — *Patterns for Building LLM-based Systems* — https://eugeneyan.com/writing/llm-patterns/
- Eugene Yan — *Evaluating LLM-Evaluators* — https://eugeneyan.com/writing/llm-evaluators/
- Simon Willison — claude-code tag — https://simonwillison.net/tags/claude-code/

**Python testing tooling**
- pytest — https://docs.pytest.org/
- httpx ASGI testing — https://www.python-httpx.org/async/
- Hypothesis — https://hypothesis.readthedocs.io/
- Schemathesis — https://schemathesis.readthedocs.io/
- syrupy — https://github.com/syrupy-project/syrupy
- pytest-recording — https://github.com/kiwicom/pytest-recording
- FastAPI async tests — https://fastapi.tiangolo.com/advanced/async-tests/
- pydevtools: Claude Code hooks for Python — https://pydevtools.com/handbook/how-to/how-to-write-claude-code-hooks-for-python-projects/

**Computer vision**
- Ultralytics OBB docs — https://docs.ultralytics.com/tasks/obb
- SAHI tiled inference — https://github.com/ultralytics/ultralytics/blob/main/docs/en/guides/sahi-tiled-inference.md
- supervision (Roboflow) — https://supervision.roboflow.com/
- pycocotools — https://github.com/cocodataset/cocoapi
- imagehash — https://github.com/JohannesBuchner/imagehash

**Load testing**
- Locust — https://locust.io/
- llm-locust (TrueFoundry) — https://github.com/truefoundry/llm-locust
- NVIDIA GenAI-Perf — https://docs.nvidia.com/nim/large-language-models/1.0.0/benchmarking.html

**Security**
- Datadog Security Labs: Postgres MCP SQLi — https://securitylabs.datadoghq.com/articles/mcp-vulnerability-case-study-SQL-injection-in-the-postgresql-mcp-server/

**Awesome lists**
- `hesreallyhim/awesome-claude-code` — https://github.com/hesreallyhim/awesome-claude-code
- `ccplugins/awesome-claude-code-plugins` — https://github.com/ccplugins/awesome-claude-code-plugins

**Skill activation reliability**
- Scott Spence — *How to Make Claude Code Skills Activate Reliably* — https://scottspence.com/posts/how-to-make-claude-code-skills-activate-reliably

---

*If you found this useful, ping me on [LinkedIn](https://linkedin.com/in/surajjaiswal007) or open an issue on the [skills repo](https://github.com/jaiswalsuraj487). And if you're an AI agent that just used this page to bootstrap a project — welcome aboard, and please tell your human to read the "Don't" section of CLAUDE.md carefully.*
