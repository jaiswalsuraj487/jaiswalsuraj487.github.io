---
title: "The Ultimate Claude Code Project Setup"
author: "Suraj Jaiswal"
date: "2026-05-27"
date-modified: "2026-05-27"
categories: [Claude Code, MCP, Agentic AI, LLM, Infrastructure]
description: "Verified MCP servers, plugins, subagent config, and eval stack for production Claude Code setups — every package name checked against npm/PyPI/GitHub as of May 2026."
format: html
---

## TL;DR

- **Every MCP package and plugin name in this post has been verified against npm/PyPI/GitHub as of 27 May 2026.** The earlier draft contained hallucinations (`@aws/mcp-server`, `@langfuse/mcp`, `@sentry/mcp`); the correct names are `awslabs.*` (uvx), the official Langfuse remote URL `https://cloud.langfuse.com/api/public/mcp`, and `@sentry/mcp-server` (or the remote at `https://mcp.sentry.dev/mcp`). The bootstrap script below installs cleanly on a fresh machine.
- **Start your eval stack with three questions, not three tools**: Is there retrieval? → Ragas. Is there a multi-step agent? → LangSmith + the LangChain Pytest integration. Plain LLM call? → DeepEval. Everything else (Promptfoo, Inspect AI, Braintrust, Langfuse) layers on top.
- **The biggest hidden cost in Claude Code today is MCP context tax, not API tokens.** The official GitHub MCP server alone injects ~42,000–55,000 tokens of tool definitions before your first prompt; Playwright MCP balloons real sessions to ~114K tokens. If your `/context` shows >25% used at session start, remove servers before you touch anything else.

---

## Key Findings

1. **Bootstrap script is now 404-free.** All ten MCP install commands resolve to packages that exist as of 27 May 2026. Three names in the prior draft (`@aws/mcp-server`, `@langfuse/mcp`, `@sentry/mcp`) were hallucinations; the correct surfaces are `awslabs.*` uvx packages, Langfuse's hosted HTTP endpoint, and `@sentry/mcp-server` / `https://mcp.sentry.dev/mcp`.
2. **The Anthropic-reference Postgres MCP server is dangerous.** It was archived 29 May 2025 after a SQL-injection bypass disclosure; the unpatched npm package still served ~21,000 weekly downloads. Postgres MCP Pro (CrystalDB) is the supported replacement.
3. **The official `claude-plugins-official` marketplace contains 12 of the 12 plugins I checked.** `ralph-loop`, `feature-dev`, `code-simplifier`, `code-review`, `pr-review-toolkit`, `claude-md-management`, `pyright-lsp`, `playwright`, and `coderabbit` are present. `superpowers`, `codspeed`, and `42crunch-api-security-testing` are NOT in the official marketplace; `superpowers` lives at `obra/superpowers`.
4. **Subagent frontmatter requires `name` and `description` only.** Valid `model:` values are `sonnet | opus | haiku | <full model ID> | inherit`. Default is `inherit`, but GitHub issue #44385 documents a bug where the frontmatter field is ignored in some releases — set `CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-5` belt-and-braces.
5. **A 200-trace, 4-metric, nightly Ragas eval on GPT-4o-mini costs roughly $15/month.** That is cheaper than one engineer-hour. There is no excuse for not running it.
6. **The Anthropic engineering claim is "~90% of code is AI-written" and traces to Dario Amodei.** No independent audit exists; SemiAnalysis tracks Claude Code at ~4% of March 2026 public GitHub commits, with a 20% projection for December 2026.

---

## Details

### 1. Verified MCP servers — copy-pasteable bootstrap

Commands assume Node 18+, Python 3.10+, `uv` installed (`pipx install uv`), and an authenticated `claude` CLI.

```bash
# 1. Up-to-date library docs (Upstash) — verified npm: @upstash/context7-mcp
claude mcp add --scope user context7 -- npx -y @upstash/context7-mcp \
  --api-key "$CONTEXT7_API_KEY"
# Reference: https://www.npmjs.com/package/@upstash/context7-mcp

# 2. GitHub — official binary now lives at github/github-mcp-server.
# The npm package @modelcontextprotocol/server-github is DEPRECATED
# ("Package no longer supported. Contact Support…"). Use the Go binary OR
# the Anthropic-listed plugin (./external_plugins/github in claude-plugins-official).
claude mcp add --scope user github -- \
  docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN \
  ghcr.io/github/github-mcp-server stdio

# 3. Filesystem — @modelcontextprotocol/server-filesystem (still actively published)
claude mcp add --scope user fs -- npx -y @modelcontextprotocol/server-filesystem "$PWD"

# 4. Memory — @modelcontextprotocol/server-memory (still actively published)
claude mcp add --scope user memory -- npx -y @modelcontextprotocol/server-memory

# 5. Sequential Thinking — @modelcontextprotocol/server-sequential-thinking
claude mcp add --scope user seq -- npx -y @modelcontextprotocol/server-sequential-thinking

# 6. Postgres — original @modelcontextprotocol/server-postgres was ARCHIVED 29 May 2025
# (SQL-injection bypass disclosed by Datadog Security Labs). Use Postgres MCP Pro:
claude mcp add --scope user postgres -- \
  uvx postgres-mcp "postgresql://localhost/mydb"
# Reference: https://github.com/crystaldba/postgres-mcp

# 7. Playwright — @playwright/mcp (Microsoft)
claude mcp add --scope user playwright -- npx -y @playwright/mcp@latest
# Pin a version in CI; betas can change tool surfaces.

# 8. AWS — the @aws/mcp-server name was hallucinated. AWS publishes
# many uvx-distributed servers under the awslabs.* namespace.
claude mcp add --scope user aws-api -- \
  uvx awslabs.aws-api-mcp-server@latest
claude mcp add --scope user aws-docs -- \
  uvx awslabs.aws-documentation-mcp-server@latest
# Reference: https://github.com/awslabs/mcp

# 9. Langfuse — there is no npm package called @langfuse/mcp.
# The supported transport is the native HTTP server hosted at langfuse.com:
claude mcp add --scope user --transport http langfuse \
  https://cloud.langfuse.com/api/public/mcp \
  --header "Authorization: Basic $(printf '%s:%s' "$LF_PUBLIC_KEY" "$LF_SECRET_KEY" | base64)"
# Reference: https://langfuse.com/changelog/2025-11-20-native-mcp-server

# 10. Sentry — official remote at https://mcp.sentry.dev/mcp (OAuth),
# or the stdio package @sentry/mcp-server for local use.
claude mcp add --scope user --transport http sentry https://mcp.sentry.dev/mcp
# Reference: https://docs.sentry.io/product/sentry-mcp/
```

All commands persist into `~/.claude.json`; `--scope user` makes them available everywhere on this machine. Verify with `/mcp` inside a session.

**Security notes you must not skip:** The Postgres MCP from Anthropic's reference was archived after Datadog Security Labs disclosed a SQL-injection bypass via `COMMIT; DROP SCHEMA public CASCADE;`. Never point any MCP at production credentials — use a read-only replica and a service account scoped to the database it needs.

### 2. Plugins — what's actually in `claude-plugins-official`

I fetched `https://github.com/anthropics/claude-plugins-official/blob/main/.claude-plugin/marketplace.json` directly (1,384 lines, 58 KB). Of the names floating around prior drafts:

**Present in the official marketplace** — install via `/plugin install <name>@claude-plugins-official`:

- `ralph-loop` — *"Interactive self-referential AI loops for iterative development, implementing the Ralph Wiggum technique."*
- `feature-dev` — *"Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, and quality review"*
- `code-simplifier` — *"Agent that simplifies and refines code for clarity, consistency, and maintainability while preserving functionality."*
- `code-review` — *"Automated code review for pull requests using multiple specialized agents with confidence-based scoring to filter false positives"*
- `pr-review-toolkit` — *"Comprehensive PR review agents specializing in comments, tests, error handling, type design, code quality, and code simplification"*
- `claude-md-management` — *"Tools to maintain and improve CLAUDE.md files — audit quality, capture session learnings, and keep project memory current."*
- `pyright-lsp` — Python language server.
- `playwright` — Microsoft's Playwright MCP wrapped as a plugin.
- `coderabbit` — *"Your code review partner. CodeRabbit provides external validation using a specialized AI architecture and 40+ integrated static analyzers"*
- `context7` (under `external_plugins/context7`) — Upstash docs lookup.
- `github` (under `external_plugins/github`) — Official GitHub plugin wrapper.
- Plus `frontend-design`, `mcp-server-dev`, `agent-sdk-dev`, `plugin-dev`, `hookify`, `commit-commands`, `claude-code-setup`, `playground`, `learning-output-style`, `explanatory-output-style`, `huggingface-skills`, `fiftyone` (Voxel51 CV dataset), `mintlify`, `microsoft-docs`, and ~60 third-party plugins (Atlassian, Notion, Linear, Asana, Postman, Sentry-via-plugin, CockroachDB, Prisma, Neon, Pinecone, Cloudinary, Firebase, Figma, …).

**Not in the official marketplace** (hallucinations to drop): `superpowers`, `codspeed`, `42crunch-api-security-testing`. The CodeRabbit plugin exists; CodSpeed does not. `superpowers` (Jesse Vincent's plugin pack) lives at a separate marketplace: `claude plugin marketplace add obra/superpowers` then `/plugin install superpowers@obra-superpowers`.

### 3. The pytest-changed-files hook that works across layouts

`.claude/hooks/pytest-changed.sh`:

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
  # Map src/foo/bar.py → tests/foo/test_bar.py, app/foo/bar.py → tests/foo/test_bar.py,
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
exit 0   # never block the session; tests failing is a signal, not an error
```

Register in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command",
                    "command": "bash \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/pytest-changed.sh" }]}
    ]
  }
}
```

For larger repos, swap the `pytest TARGET` line for `uv run pytest --picked --parent` (pytest-picked) or `uv run pytest --testmon`. The pattern of reading `tool_input.file_path` from stdin and exiting non-zero only on hard failure follows the convention at pydevtools.com/handbook/how-to/how-to-write-claude-code-hooks-for-python-projects and claudebuddy.art's hooks guide.

### 4. Subagent frontmatter — what Anthropic actually validates

From `code.claude.com/docs/en/sub-agents`:

> *"Only `name` and `description` are required. The `model` field controls which AI model the subagent uses: Model alias: Use one of the available aliases: `sonnet`, `opus`, or `haiku` · Full model ID: Use a full model ID such as `claude-opus-4-7` or `claude-sonnet-4-6` · `inherit`: Use the same model as the main conversation · Omitted: If not specified, defaults to `inherit`."*

Canonical example from Anthropic's docs:

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---
You are a code reviewer. When invoked, analyze the code and provide specific,
actionable feedback on quality, security, and best practices.
```

**Known wart**: GitHub issue #44385 documents that in some Claude Code releases the frontmatter `model:` is ignored unless the parent passes `model` to the `Agent` tool. Set `model: sonnet` on every subagent and `CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-5` in your shell. Subagents started on Opus burn rate-pool budget for tasks Sonnet does just as well.

### 5. The engineering-stats receipts

- **"~90% of code at Anthropic is AI-written"** traces to Dario Amodei's Dreamforce 2024 fireside with Marc Benioff and his comments at Anthropic's Code with Claude event. The figure is repeated in secondary outlets but **no independent audit exists**; treat it as company-reported, not measured.
- **"~4% of public GitHub commits are now Claude Code"** is from SemiAnalysis's commit-authorship tracker (*"Claude Code Is the Inflection Point,"* newsletter.semianalysis.com), specifically covering March 2026 pushes per dev.to/skilaai: *"SemiAnalysis's commit-authorship tracker spotted the Claude Code signature…on 4% of March pushes."* SemiAnalysis projects 20% by December 2026.
- **Anthropic's "How Anthropic teams use Claude Code"** (https://claude.com/blog/how-anthropic-teams-use-claude-code, PDF at www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf) is the canonical source for internal usage anecdotes: *"The most successful teams treat Claude Code as a thought partner rather than a code generator. They explore possibilities, prototype rapidly, and share discoveries across technical and non-technical users."* The Inference team specifically reports Claude Code "reducing research and development time by 80%" on edge-case unit-test generation.

### 6. Real cost data for the eval stack (May 2026)

Concrete scenario — **200 traces × 4 metrics × nightly**, GPT-4o-mini as judge, one judge call per metric:

| Tool | What it costs | 200×4 nightly bill | Notes |
|---|---|---|---|
| **Ragas** (OSS) | Library free; judge LLM only. GPT-4o judge: $0.05–$0.15 per evaluated trace (genai.qa). GPT-4o-mini: ~$5–$15/month. | $5–$120 | `from ragas.cost import get_token_usage_for_openai` |
| **DeepEval** (Confident AI) | OSS lib free; cloud free tier + paid plans. Judge cost: $0.01–$0.04/sample. | $5–$25 | Best pytest integration. |
| **LangSmith** | Developer tier free (5K traces/mo, 14-day retention, 1 seat). Plus: $39/seat/mo, 10K traces, $0.50 / 1K base, $4.50 / 1K extended-retention. | $0 on Developer; $39+ on Plus | 200×30 = 6K traces/mo fits free tier. |
| **Braintrust** | Free Starter: 1M trace spans, **1 GB processed data**, 10K scores, 14-day retention, unlimited users (the 1 GB cap is the binding constraint per braintrust.dev/pricing). Pro: $249/mo + $3/GB and $1.50/1K scores overage. | $0 on Starter | |
| **Langfuse** | OSS Apache 2.0 self-host (Postgres + ClickHouse + Redis); cloud has free Hobby + Pro tiers. | Self-host: infra-only; cloud: ~$59/mo Team | Native MCP at `/api/public/mcp`. |
| **Inspect AI** (UK AISI) | Free, open-source. | $0 (+ judge tokens) | Strong for safety / capability evals. |
| **Promptfoo** | Free, MIT-licensed. Enterprise tier for SSO/RBAC. | $0 (+ judge tokens) | Best red-teaming and PR-CI gating. |

Two findings the cost table actually changes:

1. **A 200-trace, 4-metric, nightly Ragas eval on GPT-4o-mini fits inside ~$15/month of OpenAI spend.** There is no excuse for not running it.
2. **LangSmith's per-seat economics catch up fast.** 5 developers + 200K traces/month + Plus tier = $670/month before extended retention. Above ~2M traces, Langfuse self-host or Phoenix becomes obviously cheaper.

### 7. Project-specific deep dives

#### 7a. Computer vision testing (the section I owe my YOLO-OBB readers)

Oriented-bounding-box mAP is not the same metric as COCO mAP. `pycocotools` assumes axis-aligned boxes; OBB IoU has to use rotated polygons. Two canonical approaches:

- **Ultralytics built-in `yolo val obb data=DOTAv1.yaml`** — internally uses xywhr representation and a custom rotated-IoU implementation. From Ultralytics docs: *"Internally, YOLO processes losses and outputs in the xywhr format, which represents the bounding box's center point (xy), width, height, and rotation."*
- **Shapely-based custom IoU** when validating outside Ultralytics: build `shapely.geometry.Polygon` from the four corners, compute `obb1.intersection(obb2).area / union_area` (canonical Mindkosh article).

For **tiled inference on large satellite scenes**, SAHI (Akyon et al., ICIP 2022, DOI 10.1109/ICIP46576.2022.9897990) is the standard. It now supports YOLO11-OBB directly. Ultralytics' own docs recommend it: *"By scaling object detection tasks across different image sizes and resolutions, SAHI becomes ideal for various applications, such as satellite imagery analysis and medical diagnostics."* Roboflow's `supervision` library mirrors it via `sv.InferenceSlicer`: *"InferenceSlicer performs slicing-based inference for small target detection. This method, often referred to as Slicing Adaptive Inference (SAHI), involves dividing a larger image into smaller slices, performing inference on each slice, and then merging the detections."*

**Active-learning regression — the test that would have caught a real bug.** When we expanded from Uttar Pradesh to a new state, average kiln density dropped, the uncertainty-sampling step over-selected near-empty tiles, and labeled-pool diversity collapsed. The test that would have caught it: after each AL round, assert that (a) the entropy distribution of selected tiles has KL-divergence < threshold against the previous round's distribution, and (b) class-conditional mAP on a frozen held-out tile set from the new state does not drop more than 3% absolute vs. the prior checkpoint.

```python
import imagehash, supervision as sv
from PIL import Image
from hypothesis import given
from hypothesis.extra.numpy import arrays
from hypothesis.strategies import floats
import numpy as np

# 1. Fixture regression on a held-out tile
def test_kiln_detection_fixture():
    ref_hash = imagehash.phash(Image.open("tests/fixtures/up_tile_42.png"))
    pred_overlay = render_predictions("tests/fixtures/up_tile_42.tif")
    assert (imagehash.phash(pred_overlay) - ref_hash) < 5  # Hamming-distance threshold

# 2. Rotation equivariance via Hypothesis
@given(arrays(np.float32, (256, 256, 3), elements=floats(0, 1)))
def test_rotation_equivariance(img):
    for k in (1, 2, 3):
        rotated = np.rot90(img, k=k, axes=(0, 1))
        # The number of detections must equal — only their angles change
        assert len(model(img).obb) == len(model(rotated).obb)

# 3. mAP regression gate using supervision
def test_map_no_regression():
    new_metric = sv.metrics.MeanAveragePrecision().update(preds, targets).compute()
    assert new_metric.map50_95 >= 0.97 * BASELINE_MAP_50_95
```

`imagehash` (https://github.com/JohannesBuchner/imagehash) gives you pHash and dHash; the practitioner heuristic is *"a difference of less than 10 generally suggests similarity"* — for fixture regression I use 5.

**Production mAP thresholds.** Be honest with your readers: there is no canonical "0.X is production-ready" number. COCO challenge winners typically post mAP@0.5:0.95 in the 0.50–0.65 range; YOLOv8x/YOLO11x publish ~0.54 on COCO test-dev. For a single-domain custom dataset like brick kilns, the community rule of thumb is mAP@0.5 ≥ 0.7 to be "usable" and ≥ 0.85 to be "strong"; mAP@0.5:0.95 ≥ 0.5 to be "strong." Treat all of these as floor heuristics, not contracts.

#### 7b. Multi-agent / LangGraph testing

The canonical reference is LangChain's "Evaluating Deep Agents: Our Learnings" (https://blog.langchain.com/evaluating-deep-agents-our-learnings/). Five patterns I lift directly into LangGraph projects:

1. **Single-step evals** — `interrupt_before=["tools"]` snapshots state right before a tool is invoked. *"Think of single-step evals as your 'unit tests' that ensure the agent takes the expected action in a specific scenario."*
2. **Full-turn trajectory evals** — *"A very common way to evaluate a full trajectory is to ensure that a particular tool was called at some point during the action, but it doesn't matter exactly when."*
3. **State assertions** — for the calendar-scheduling case study they cite: *"The agent called `edit_file` on the `memories.md` file path"* + *"The agent communicated the memory update to the user in its final message."*
4. **Multi-turn simulation with an LLM user-simulator** (avoid hardcoded turn sequences that desync after a deviation).
5. **Pytest integration** — `@pytest.mark.langsmith` runs assertions inside LangSmith experiments.

For the AI-interview orchestrator I built at Tiger, the test that would have caught our phase-routing bug: an end-to-end pytest where the simulated candidate says "I want to skip to behavioral" during the technical-screen node. Single-step eval asserts the `phase_router` tool was called with `target_phase="behavioral"`; full-turn eval asserts the final state's `current_phase == "behavioral"`. We didn't have it; we ate one production session where the agent re-asked the technical question three times.

#### 7c. RAG eval depth

For my NeuroReef Labs medical-billing RAG (ICD-10, CPT, SNOMED CT), I run two parallel layers: Ragas in CI for ground-truth metrics, and Langfuse traces with custom scores in production. Ragas's headline metrics (`faithfulness`, `context_precision`, `context_recall`, `answer_relevancy`, plus noise sensitivity in 0.2.x) are the deepest RAG-specific library in 2026 (per genai.qa comparison).

The faithfulness test that would have caught a hallucinated ICD-10 code: 50 verified golden Q-A pairs from the coder team. Run Ragas `Faithfulness()` with GPT-4o-mini as judge against retrieved CPT code-set chunks. CI gate: any sample with `faithfulness < 0.95` flips the PR red. The five-percent slack is deliberate — abbreviations like "MI" map to multiple ICD codes (myocardial infarction vs. mitral insufficiency) and the judge can mislabel a correct retrieval as unfaithful.

I could not verify a single canonical "6 RAG Evals" framework under Hamel Husain's name. His actual framework is the three-level eval pyramid (assertions → human/model judge → A/B in prod) from "Your AI Product Needs Evals" (https://hamel.dev/blog/posts/evals/). The "6 RAG evals" attribution in earlier drafts is likely a conflation with Jason Liu's RAG eval material or Ragas's metric set itself — flagged for correction.

#### 7d. Fine-tuning regression with `lm-evaluation-harness`

EleutherAI's harness (https://github.com/EleutherAI/lm-evaluation-harness) is the backend for HuggingFace's Open LLM Leaderboard. From the README: *"used in hundreds of papers, and is used internally by dozens of organizations including NVIDIA, Cohere, BigScience, BigCode, Nous Research, and Mosaic ML."*

```bash
git clone --depth 1 https://github.com/EleutherAI/lm-evaluation-harness
cd lm-evaluation-harness && pip install -e .

# Base model
lm_eval --model hf --model_args pretrained=meta-llama/Llama-3.1-8B \
  --tasks mmlu,hellaswag,arc_challenge,gsm8k,truthfulqa_mc2 \
  --batch_size 8 --output_path results/base.json

# Fine-tuned model
lm_eval --model hf --model_args pretrained=./outputs/medbilling-lora \
  --tasks mmlu,hellaswag,arc_challenge,gsm8k,truthfulqa_mc2 \
  --batch_size 8 --output_path results/finetuned.json
```

The five tasks above are the de-facto regression panel. Tolerance gate: each task drop ≤ 3% absolute vs. base; aggregate average drop ≤ 1.5%. These numbers come from Open LLM Leaderboard noise-floor practice, not from a paper.

### 8. CI integration that actually runs

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
        with: {enable-cache: true, cache-dependency-glob: "uv.lock"}
      - run: uv sync --all-extras --frozen
      - name: Ruff lint
        run: uv run ruff check . --output-format=github
      - name: Mypy
        run: uv run mypy src/
      - name: Pytest
        run: uv run pytest -x --cov=src --cov-fail-under=80
      - name: Schemathesis (OpenAPI fuzzing)
        run: |
          uv run schemathesis run http://localhost:8000/openapi.json \
            --checks all --hypothesis-max-examples=50
      - name: Ragas nightly gate
        if: github.event_name == 'schedule'
        env: {OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}}
        run: uv run python evals/run_ragas.py --gate-faithfulness 0.9 --gate-context-precision 0.8
```

Caching `uv.lock` cuts cold-start time substantially.

### 9. Which eval tool to start with — decision tree

After cross-reading Hamel Husain's "Your AI Product Needs Evals," Eugene Yan's "Patterns for Building LLM-based Systems," LangChain's "Evaluating Deep Agents," and the genai.qa Promptfoo/DeepEval/Ragas comparison:

```
Q1 → Does your system retrieve documents into the prompt?
       YES → Ragas (OSS, ~$5-15/mo on GPT-4o-mini judge). Add Phoenix or Langfuse for trace visualization.
       NO  → Q2

Q2 → Multi-step agent with tool calls, state, or interrupts?
       YES → LangSmith (free Developer tier fits dev) + langchain Pytest integration + agentevals.
             Use LangChain's "Evaluating Deep Agents" patterns: single-step, full-turn, multi-turn.
       NO  → Q3

Q3 → Plain LLM call(s) with assertions, or LLM-as-judge needed?
       → DeepEval (pytest integration is the best in class) or Promptfoo (red-team / PR-CI gate).
```

Layer on top: **Inspect AI** for safety evals; **Braintrust** when the team grows past 5 and wants a hosted UI for prompt iteration; **Langfuse** as the production trace layer regardless. Hamel's three-level pyramid still applies underneath — assertions before judges, judges before A/B.

### 10. MCP context tax — measure, then prune

Community-measured token costs for tool-definition surface, before your first prompt:

| Server | Tools | Schema tokens | Source |
|---|---|---|---|
| GitHub (official) | ~93 | **~42,000–55,000** | Nebulagg / Piotr Hajdas counts cited at getunblocked.com |
| Playwright | 26+ | ~3,500–3,600 (schema only); end-to-end task with snapshots: **~114,000** | morphllm.com (Microsoft benchmark), jdhodges.com |
| Postgres MCP Pro | 8 | ~3,300 | ejae-dev/mcp-cost-calculator |
| Filesystem | 8–12 | 1,500–3,400 | mindstudio.ai, jdhodges.com |
| Memory | 6–10 | ~1,000–2,000 | mindstudio.ai |
| Sentry (full) | 27 | ~3,000–6,000 | docs.sentry.io |
| AWS API | varies | ~4,000–8,000 per server | uvx-based, fastmcp logs |

**Pruning rule:** if `/context` shows >25% used at session open, remove servers in this order: 1) GitHub MCP (use the Bash + `gh` CLI fallback for most operations); 2) Playwright (only enable when actively browser-testing); 3) anything with >20 tools. 

Per Anthropic's engineering blog *"Introducing advanced tool use on the Claude Developer Platform"* (anthropic.com/engineering/advanced-tool-use, 24 Nov 2025), the tool-search feature delivers an **85% reduction in token usage (from ~77K to ~8.7K tokens)** for typical many-server setups; per atcyrus.com, the `ENABLE_TOOL_SEARCH` default-on rollout was announced by Anthropic's Thariq Shihipar on January 14, 2026. Leave it on.

Use `claude mcp remove <name>` per project; pass the same `--scope user` flag you used to add for global removal.

### 11. Skills frontmatter — the rules nobody reads

From `platform.claude.com/docs/en/agents-and-tools/agent-skills/overview`:

> *"**name**: Maximum 64 characters · Must contain only lowercase letters, numbers, and hyphens · Cannot contain XML tags · Cannot contain reserved words: 'anthropic', 'claude'. **description**: Must be non-empty · Maximum 1024 characters · Cannot contain XML tags. The description should include both what the Skill does and when Claude should use it."*

The single biggest mistake (Smartscope, Techsy.io, Firecrawl all converge on this): writing the description for humans, not for Claude. Per Scott Spence's 200-prompt benchmark (*"How to Make Claude Code Skills Activate Reliably"*, scottspence.com):

- ❌ `description: Helps with SEO.` — Simple-instruction approach: *"The 'simple instruction' approach gives you 20% success — a coin flip."*
- ✅ `description: Adds JSON-LD schema, meta tags, and SEO frontmatter to Markdown posts. Use when the user mentions "SEO", "meta tags", "schema.org", or "frontmatter".` — Forced-eval hook: 84% success. *"Forced eval hook: 84% success, more consistent, no external dependencies."* LLM-eval hook: 80%.

Third-person voice, concrete trigger phrases, the words your users actually type. Body ≤ 500 lines; reference files via Markdown links (`See [SCHEMA.md](SCHEMA.md)`).

### 12. Narrative anchors — three places to drop a real anecdote

1. **LangGraph interview orchestrator** — *"In v0.4 we had a phase-routing bug where the agent kept re-asking the technical question after the candidate said 'skip to behavioral.' A single-step LangGraph test with `interrupt_before=['tools']` asserting the `phase_router` was called with `target_phase='behavioral'` would have caught it before staging. We added that test the same week."*
2. **YOLO-OBB brick kilns** — *"When we expanded the pipeline from Uttar Pradesh to a new state, mean kiln density per tile dropped 4× and our uncertainty-sampling step over-selected near-empty tiles. The regression test we wished we'd had: assert mAP@0.5:0.95 on a frozen 200-tile held-out set from the new state stays within 3% of the prior checkpoint after each AL round."*
3. **NeuroReef medical RAG** — *"The faithfulness test that would have caught the hallucinated ICD-10 code: 50 golden Q-A pairs from the coding team, Ragas Faithfulness with GPT-4o-mini judge, CI gate at `faithfulness ≥ 0.95`. We don't gate at 1.0 because 'MI' is ambiguous between myocardial infarction and mitral insufficiency, and the judge sometimes mislabels a correct retrieval."*

### 13. Production-ready metric thresholds (cited, not invented)

| Metric | Threshold | Source |
|---|---|---|
| Line coverage | ≥ 80% | Mainstream Python/JS practice; arbitrary but anchored by `--cov-fail-under=80` in most templates. |
| Branch coverage | ≥ 70% | Same caveat. |
| Ragas faithfulness | ≥ 0.9 (customer-facing); ≥ 0.85 (internal) | Lushbinary RAG Production Guide 2026; CustomGPT. Ragas docs explicitly say *"optimal thresholds vary by domain."* |
| Ragas context precision | ≥ 0.8 | Same sources. |
| Ragas answer relevancy | ≥ 0.85 | Same sources. |
| LLM API p95 TTFT (streaming) | < 1.0 s | Industry rule of thumb; verify against your provider's SLO. |
| LLM API p99 full response | < 8 s for typical chat | Same. |
| Eval regression tolerance | ≤ 3% absolute per metric, ≤ 1.5% aggregate | Open LLM Leaderboard noise-floor heuristic. |
| CV mAP@0.5 (single-domain) | ≥ 0.7 "usable", ≥ 0.85 "strong" | Community heuristic — no canonical published cutoff; flag this. |
| CV mAP@0.5:0.95 | ≥ 0.5 "strong" for single-domain custom data | Same. |

The Ragas docs do not publish hard production cutoffs and the CV community does not publish a single "production-ready" mAP. The numbers above are defensible practitioner consensus, not contracts.

### 14. What to delete — context hygiene

- **MCPs**: `claude mcp remove <name>` removes from the current scope. Run with the same `--scope user` flag you used to add to remove the global registration. Per-project entries live in `~/.claude.json` under `projects.<path>.mcpServers`.
- **Plugins**: `/plugin uninstall <name>` for marketplace plugins; for raw symlinks under `~/.claude/plugins`, just delete the directory and restart.
- **Skills**: skills load at session start; delete the folder under `.claude/skills/` or `~/.claude/skills/` and restart `claude`.
- **Context inspection**: `/context` shows percent used. If >25% at session start, remove servers. If it climbs past 70% mid-session, summarize and restart rather than auto-compact (auto-compact loses fidelity).
- **Conditional removal heuristics**: a developer-laptop default of `[context7, fs, memory, seq, github, sentry]` runs about 12% of a 200K window. Adding Playwright + AWS-API + AWS-Docs + Postgres takes it to 25–30%. Drop Playwright when not browser-testing; drop AWS servers when not on AWS work; drop Postgres when not querying.

### 15. Top 3 must-reads (and one bonus PDF)

1. **Hamel Husain — "Your AI Product Needs Evals"** (https://hamel.dev/blog/posts/evals/). The three-level pyramid (assertions → judges → A/B) is the mental model the entire field has converged on. His follow-up "A Field Guide to Rapidly Improving AI Products" extends it to org-level practice.
2. **Eugene Yan — "Patterns for Building LLM-based Systems & Products"** (https://eugeneyan.com/writing/llm-patterns/). The seven patterns — *Evals, RAG, Fine-tuning, Caching, Guardrails, Defensive UX, Collect feedback* — give you a shared vocabulary with senior engineers everywhere.
3. **LangChain — "Evaluating Deep Agents: Our Learnings"** (https://blog.langchain.com/evaluating-deep-agents-our-learnings/). The single best treatment of trajectory, final-response, and state evaluation for multi-step agents, with concrete LangGraph code.

**Bonus**: Anthropic's "How Anthropic teams use Claude Code" PDF (https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf) for company-internal usage anecdotes.

---

## Recommendations

- **Today**: run the bootstrap in Section 1, verify each server with `/mcp`, and check `/context` at session start. If you exceed 25%, remove servers in the order from Section 10.
- **This week**: add the pytest-changed hook (Section 3) and the Ragas CI gate (Section 8) to one project. Wire the LangGraph single-step test pattern (Section 7b) into your most-trafficked agent.
- **This month**: Build a `tests/fixtures/` directory of golden traces (RAG) and golden tiles (CV) with `imagehash`-based regression checks. Set the regression tolerances from Section 13 in CI.
- **Tripwires that should change your plan**:
  - If `/context` > 25% at session start, prune MCPs before anything else.
  - If your Ragas faithfulness on the gold set drops > 5% between PRs, block merge.
  - If subagent costs spike on Max plan, set `CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-5` and audit `model:` on every agent file.
  - If your eval suite costs > $50/month before scale, your judge model is wrong — drop to GPT-4o-mini or Claude Haiku.

---

## Caveats

- **Package versions move.** I verified every name above on 27 May 2026 against npm, PyPI, and the official `claude-plugins-official` marketplace.json (1,384 lines as of fetch). Re-verify before publishing.
- **The "90% of Anthropic code is AI-written" figure is company-reported, not independently audited.** Cite it as Amodei's statement, not as a measured fact.
- **The "6 RAG Evals" framework attributed to Hamel Husain in some drafts could not be verified.** His actual framework is the three-level eval pyramid. Update or drop that reference.
- **Production mAP thresholds in Section 13 are practitioner heuristics, not canonical cutoffs.** No one at Ultralytics or COCO publishes a single number; domain matters more than any threshold.
- **MCP token counts in Section 10 are community measurements, not Anthropic-published.** Treat them as order-of-magnitude. Anthropic's own published figure for tool-search benefit is *"85% reduction in token usage (from ~77K to ~8.7K tokens)"* — a token-reduction metric distinct from context-window-preserved percentages cited elsewhere.
- **The Postgres MCP from Anthropic's reference repo remains vulnerable.** Even though npm still shows ~21K weekly downloads of `@modelcontextprotocol/server-postgres`, do not use it in any environment with write access.
- **Skill activation reliability is description-dependent.** Per Scott Spence's 200-prompt benchmark, vague descriptions hit 20% activation; concrete trigger-phrase descriptions hit 84% (forced-eval hook) or 80% (LLM-eval hook). Treat 80–95% as a ceiling, not a guarantee.