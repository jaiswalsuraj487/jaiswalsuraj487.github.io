---
title: "Supercharging Claude Code: Plugins, MCP Servers & Skills"
author: "Suraj Jaiswal"
date: "May 6, 2026"
format: html
categories: ['Agentic AI', 'Developer Tools', 'Claude Code']
---

# Supercharging Claude Code: Plugins, MCP Servers & Skills

Claude Code is Anthropic's official CLI for interacting with Claude directly in your terminal. Out of the box it's powerful — but the real magic comes from extending it with **plugins**, **MCP servers**, and **skills**. This post walks through exactly what I set up and how to replicate it on any machine.

---

## What's the difference?

| Type | What it is | How to use |
|---|---|---|
| **Plugin** | Bundles skills + agents + hooks into one installable package | `/plugin install <name>@claude-plugins-official` |
| **MCP Server** | External tool Claude can call (browser, Drive, DB, APIs) | `claude mcp add <name> -- <command>` |
| **Skill** | A slash command that loads specific workflow instructions | `/skill-name` in the prompt |
| **Agent** | A specialized sub-agent Claude can spawn for a task | `@agent-name` in the prompt |

---

## Prerequisites

```bash
# Install Claude Code
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
```

---

## 1. Install Node.js (required for MCP servers)

Most MCP servers run via `npx`. Check first:

```bash
which node && node --version
```

If not found, install via Homebrew (macOS):

```bash
brew install node
```

---

## 2. Add Context7 MCP — Live Library Docs

[Context7](https://github.com/upstash/context7) by Upstash fetches up-to-date documentation for any library on demand. Instead of Claude guessing from training data, it pulls current docs.

```bash
claude mcp add context7 -- npx -y @upstash/context7-mcp
```

Verify it's connected:

```bash
claude mcp list
```

**Usage:** Just mention the library in your prompt:
```
how do I use useQuery in TanStack Query v5? use context7
```

---

## 3. Install ralph-loop Plugin — Autonomous Dev Loops

`ralph-loop` implements the **Ralph Wiggum technique** — an iterative loop where Claude keeps working on a task until it's done or hits a max iteration limit. Great for "fix all failing tests" type tasks.

Inside Claude Code:
```
/plugin install ralph-loop@claude-plugins-official
/reload-plugins
```

**Usage:**
```
/ralph-loop "fix all failing tests" --max-iterations 10 --completion-promise "ALL TESTS PASS"
/cancel-ralph   ← to stop the loop early
```

**ralph-loop vs /loop skill:**

| | `/loop` | `/ralph-loop` |
|---|---|---|
| Trigger | Time-based (every N minutes) | Completion-based |
| Stops when | You cancel it | Promise met or max iterations hit |
| Context | Fresh each run | Preserves file changes + git history |
| Best for | Polling / monitoring | Autonomous task completion |

---

## 4. Install agent-browser — Browser Automation

`agent-browser` by Vercel Labs is a native Rust CLI for browser automation. It lets Claude navigate websites, fill forms, click buttons, extract data, and take screenshots — all from the terminal.

```bash
# Step 1: Install CLI
npm install -g agent-browser

# Step 2: Download Chrome for Testing
agent-browser install

# Step 3: Register as Claude Code skill (run inside Claude Code)
npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y
```

**Usage — just describe the task naturally:**
```
open linkedin.com and search for AI Engineer jobs in India
take a screenshot of the results
find salary data for Amazon ML Engineer roles in Canada
```

**Note:** The skill installs at the **project level** by default. For global access across all projects, add the `-g` flag:
```bash
npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y -g
```

---

## 5. Install superpowers Plugin — Dev Workflow Skills

The `superpowers` plugin adds a full suite of structured development workflow skills:

```
/plugin install superpowers@claude-plugins-official
/reload-plugins
```

Skills included:

| Skill | When to use |
|---|---|
| `/brainstorming` | Before any feature work — explores intent first |
| `/writing-plans` | Plan multi-step implementation before touching code |
| `/executing-plans` | Execute a written plan with review checkpoints |
| `/systematic-debugging` | Structured bug fixing workflow |
| `/test-driven-development` | TDD before writing implementation code |
| `/dispatching-parallel-agents` | Run 2+ independent tasks in parallel |
| `/verification-before-completion` | Verify before claiming anything is "done" |
| `/finishing-a-development-branch` | Guided merge/PR/cleanup after implementation |
| `/using-git-worktrees` | Isolated feature branches with safety checks |

---

## 6. Install feature-dev Plugin — Feature Development Agents

```
/plugin install feature-dev@claude-plugins-official
/reload-plugins
```

Adds 3 specialized agents:

| Agent | What it does |
|---|---|
| `@code-architect` | Designs architecture and implementation blueprints |
| `@code-explorer` | Deep-dives into existing codebase to map patterns |
| `@code-reviewer` | Reviews for bugs, security, and code quality |

**Usage:**
```
@code-architect design a FastAPI authentication system with JWT
@code-explorer how does the data pipeline work in this repo?
@code-reviewer check my last commit for issues
```

---

## 7. Install code-simplifier Plugin

```
/plugin install code-simplifier@claude-plugins-official
/reload-plugins
```

Adds `@code-simplifier` agent — reviews recently changed code for reuse, quality, and efficiency, then fixes issues found.

---

## Checking what's installed

Inside Claude Code, run `/context` to see everything loaded:

```
/context     ← shows MCP tools, skills, agents, memory, token usage
/mcp         ← manage MCP server connections
/skills      ← browse available skills
/agents      ← browse available agents
/plugin list ← list installed plugins
```

---

## Full setup checklist

```bash
# Terminal (outside Claude Code)
brew install node
npm install -g agent-browser
agent-browser install
claude mcp add context7 -- npx -y @upstash/context7-mcp

# Inside Claude Code prompt
/plugin install ralph-loop@claude-plugins-official
/plugin install superpowers@claude-plugins-official
/plugin install feature-dev@claude-plugins-official
/plugin install code-simplifier@claude-plugins-official
/reload-plugins

# Add agent-browser skill
npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -y -g
```

---

## Replicating on a new machine

Everything except `agent-browser` skill (project-local) is stored in `~/.claude.json` and `~/.claude/` — so it follows your user account. On a new machine:

1. Install Claude Code + Node.js
2. Re-run `claude mcp add` for each MCP server
3. Re-run `/plugin install` for each plugin inside Claude Code
4. Re-run `npx skills add` for agent-browser

> Tip: Keep this checklist bookmarked — it takes under 5 minutes to fully replicate your setup on any machine.

---

## Resources

- [Claude Code Docs](https://code.claude.com/docs)
- [claude-plugins-official (GitHub)](https://github.com/anthropics/claude-plugins-official)
- [Context7 MCP](https://github.com/upstash/context7)
- [agent-browser (Vercel Labs)](https://github.com/vercel-labs/agent-browser)
