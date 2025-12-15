# The Pair Planning Framework

A **pluggable multi-agent planning framework** for Claude Code. Swap planning engines (Claude, RepoPrompt, Codex) while keeping the same discovery agents and execution patterns.

## Core Concept: Pluggable Planning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PAIR PLANNING FRAMEWORK                             │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────────┐  │
│  │ code-scout  │    │  doc-scout  │    │      PLUGGABLE PLANNER          │  │
│  │ (codebase)  │    │ (external)  │    │                                 │  │
│  └──────┬──────┘    └──────┬──────┘    │  ┌─────────┐  ┌─────────────┐   │  │
│         │                  │           │  │ Claude  │  │ RepoPrompt  │   │  │
│         │   CONTEXT        │           │  │ (pair-*)│  │(repoprompt-)│   │  │
│         └────────┬─────────┘           │  └────┬────┘  └──────┬──────┘   │  │
│                  │                     │       │              │          │  │
│                  │                     │  ┌────┴──────────────┴────┐     │  │
│                  └────────────────────►│  │        Codex          │     │  │
│                                        │  │      (codex-*)        │     │  │
│                                        │  └───────────────────────┘     │  │
│                                        └───────────────┬────────────────┘  │
│                                                        │                   │
│                                                        │ PLAN              │
│                                                        ▼                   │
│                                        ┌───────────────────────────────┐   │
│                                        │         plan-coder            │   │
│                                        │   (parallel file execution)   │   │
│                                        └───────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**The framework provides:**
- **Fixed agents**: code-scout, doc-scout, plan-coder (same across all plugins)
- **Pluggable planners**: Swap between Claude, RepoPrompt MCP, or Codex MCP
- **Two execution patterns**: Pipeline (iterative) or Swarm (one-shot)

---

## Table of Contents

- [Why Pluggable Planning?](#why-pluggable-planning)
- [Execution Patterns](#execution-patterns)
- [Planning Engines](#planning-engines)
- [Agent Architecture](#agent-architecture)
- [Plugin Matrix](#plugin-matrix)
- [Installation](#installation)
- [Usage](#usage)
- [Choosing the Right Plugin](#choosing-the-right-plugin)
- [Requirements](#requirements)
- [Troubleshooting](#troubleshooting)

---

## Why Pluggable Planning?

Different planning engines excel at different tasks:

| Planning Engine | Strengths | Best For |
|-----------------|-----------|----------|
| **Claude** (pair-*) | Fast, no dependencies | Quick tasks, standalone operation |
| **RepoPrompt** (repoprompt-*) | Intelligent context management, file selection | Complex refactors, architectural changes |
| **Codex** (codex-*) | High reasoning (gpt-5.2), detailed architectural plans | Difficult problems, unfamiliar codebases |

The Pair Planning Framework lets you **use the same workflow** regardless of which planner you choose. The discovery agents (code-scout, doc-scout) and execution agents (plan-coder) remain identical - only the planning engine changes.

---

## Execution Patterns

The framework offers **two execution patterns** that work with any planning engine.

### Pipeline Pattern (Iterative, Human-in-the-Loop)

```
┌────────────────────────────────────────────────────────────────────────┐
│                       ITERATIVE DISCOVERY LOOP                         │
│                                                                        │
│  code-scout ───► CHECKPOINT ───► doc-scout ───► CHECKPOINT ───► ...    │
│       │               │               │               │                │
│       │          User decides    User decides    User decides          │
│       │         "Add research"   "Add more"      "Complete"            │
│       │               │               │               │                │
│       └───────────────┴───────────────┴───────────────┘                │
│                               │                                        │
│                       context_package                                  │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
                                ▼
                     ┌────────────────────┐
                     │      PLANNER       │
                     │  (pluggable MCP)   │
                     └─────────┬──────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │ plan-coder │  │ plan-coder │  │ plan-coder │  (parallel)
        │   file1    │  │   file2    │  │   file3    │
        └────────────┘  └────────────┘  └────────────┘
```

**Characteristics:**
- Single `/orchestrate` command handles entire workflow
- User controls discovery via checkpoints (AskUserQuestion)
- Incrementally add research until context is complete
- Best for: exploratory tasks, unfamiliar codebases, complex features

**Plugins:** `pair-pipeline`, `repoprompt-pair-pipeline`, `codex-pair-pipeline`

---

### Swarm Pattern (One-Shot, Fast Execution)

```
┌──────────────────────────────────────────────────┐
│                  /plan COMMAND                   │
│                                                  │
│          ┌─────────────┬─────────────┐           │
│          ▼             ▼             │           │
│    ┌───────────┐ ┌───────────┐       │           │
│    │code-scout │ │ doc-scout │  (parallel)       │
│    └─────┬─────┘ └─────┬─────┘       │           │
│          │             │             │           │
│          └──────┬──────┘             │           │
│                 ▼                    │           │
│          ┌───────────┐               │           │
│          │  PLANNER  │               │           │
│          │(pluggable)│               │           │
│          └─────┬─────┘               │           │
│                │                     │           │
│                ▼                     │           │
│       IMPLEMENTATION PLAN            │           │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│                  /code COMMAND                   │
│                                                  │
│              Parse plan from input               │
│                       │                          │
│       ┌───────────────┼───────────────┐          │
│       ▼               ▼               ▼          │
│ ┌────────────┐ ┌────────────┐ ┌────────────┐     │
│ │ plan-coder │ │ plan-coder │ │ plan-coder │     │
│ │   file1    │ │   file2    │ │   file3    │     │
│ └────────────┘ └────────────┘ └────────────┘     │
│                   (parallel)                     │
│                                                  │
│                 RESULTS TABLE                    │
└──────────────────────────────────────────────────┘
```

**Characteristics:**
- Two separate commands: `/plan` then `/code`
- No checkpoints - review the plan, then execute
- Scouts always run in parallel
- Best for: well-defined tasks, fast execution

**Plugins:** `pair-swarm`, `repoprompt-swarm`, `codex-swarm`

---

## Planning Engines

### How Plans Flow: Direct vs MCP Fetch

The key architectural difference is **how plans move from planner to coders**.

#### Direct Plan Distribution (pair-*, codex-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│             FULL PLAN TEXT                  │  │
│  └───────────┘         │    (returned directly to orchestrator)      │  │
│                        └──────────────────────┬──────────────────────┘  │
│                                               │                         │
│                 ┌────────────────────────────┼────────────────────┐     │
│                 │                            │                    │     │
│                 ▼                            ▼                    ▼     │
│       ┌──────────────┐            ┌──────────────┐      ┌──────────────┐│
│       │  plan-coder  │            │  plan-coder  │      │  plan-coder  ││
│       │              │            │              │      │              ││
│       │target: file1 │            │target: file2 │      │target: file3 ││
│       │plan: [instr1]│            │plan: [instr2]│      │plan: [instr3]││
│       │              │            │              │      │              ││
│       │(plan passed  │            │(plan passed  │      │(plan passed  ││
│       │ IN PROMPT)   │            │ IN PROMPT)   │      │ IN PROMPT)   ││
│       └──────────────┘            └──────────────┘      └──────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Planner returns the full implementation plan to orchestrator
2. Orchestrator parses and extracts per-file instructions
3. Plan-coders receive instructions **embedded in their prompt**
4. No MCP dependency for coders

**Used by:** pair-*, codex-*

---

#### MCP Plan Storage & Fetch (repoprompt-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│              RepoPrompt MCP                 │  │
│  │           │         │  ┌───────────────────────────────────────┐  │  │
│  │   calls   │         │  │     PLAN STORED IN CHAT SESSION       │  │  │
│  │ context_  │         │  │         chat_id: abc123               │  │  │
│  │  builder  │         │  └───────────────────────────────────────┘  │  │
│  └───────────┘         └──────────────────────┬──────────────────────┘  │
│        │                                      │                         │
│        │ returns: chat_id + file_lists        │                         │
│        ▼                                      │                         │
│  ┌─────────────────┐                          │                         │
│  │  Orchestrator   │                          │                         │
│  │   only knows    │                          │                         │
│  │   chat_id &     │                          │                         │
│  │   file list     │                          │                         │
│  └────────┬────────┘                          │                         │
│           │                                   │                         │
│           ▼                                   ▼                         │
│  ┌─────────────────┐                 ┌─────────────────┐                │
│  │   plan-coder    │                 │   plan-coder    │                │
│  │                 │                 │                 │                │
│  │ chat_id: abc123 │                 │ chat_id: abc123 │                │
│  │ target: file1   │                 │ target: file2   │                │
│  │                 │                 │                 │                │
│  │  FETCHES plan   │                 │  FETCHES plan   │                │
│  │  via mcp__Repo  │                 │  via mcp__Repo  │                │
│  │  Prompt__chats  │                 │  Prompt__chats  │                │
│  └────────┬────────┘                 └────────┬────────┘                │
│           │                                   │                         │
│           ▼                                   ▼                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        RepoPrompt MCP                             │  │
│  │           (each coder fetches from same chat_id)                  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Planner stores plan in RepoPrompt, returns `chat_id`
2. Orchestrator only receives `chat_id` + file lists
3. Each plan-coder **fetches** its instructions from RepoPrompt MCP
4. Plan persists for review/resumption

**Used by:** repoprompt-*

---

### Planning Engine Comparison

| Aspect | pair-* (Claude) | codex-* (Codex) | repoprompt-* (RepoPrompt) |
|--------|-----------------|-----------------|---------------------------|
| **Plan storage** | Orchestrator memory | Orchestrator memory | RepoPrompt MCP |
| **Plan delivery** | In prompt | In prompt | Via `chat_id` fetch |
| **Coders need MCP?** | No | No | Yes |
| **Planning model** | Claude | gpt-5.2 | RepoPrompt context_builder |
| **Resume mechanism** | Accumulated context | Limited | `chat_id` continuation |

---

## Agent Architecture

All plugins share the **same four agent types**. Only the planner implementation differs.

| Agent | Role | Input | Output |
|-------|------|-------|--------|
| **code-scout** | Investigate codebase patterns, conventions | `task:` + `mode:` | `CODE_CONTEXT` |
| **doc-scout** | Fetch external documentation, APIs | `query:` | `EXTERNAL_CONTEXT` |
| **planner** | Synthesize context into implementation plan | Context package | Plan + file lists |
| **plan-coder** | Implement changes for ONE file | File + instructions | `COMPLETE` or `BLOCKED` |

### Agent Tools by Planning Engine

| Agent | pair-* | repoprompt-* | codex-* |
|-------|--------|--------------|---------|
| code-scout | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash |
| doc-scout | Research tools | Research tools | Research tools |
| planner | Read, Glob, Grep, Bash | `context_builder` / `chat_send` | `mcp__codex-cli__codex` |
| plan-coder | Read, Edit, Write, Bash | + `mcp__RepoPrompt__chats` | Read, Edit, Write, Bash |

---

## Plugin Matrix

| Plugin | Pattern | Planning Engine | MCP Required | Plan Flow |
|--------|---------|-----------------|--------------|-----------|
| **pair-pipeline** | Pipeline | Claude Code | None | Direct |
| **pair-swarm** | Swarm | Claude Code | None | Direct |
| **repoprompt-pair-pipeline** | Pipeline | RepoPrompt | RepoPrompt | MCP Fetch |
| **repoprompt-swarm** | Swarm | RepoPrompt | RepoPrompt | MCP Fetch |
| **codex-pair-pipeline** | Pipeline | Codex gpt-5.2 | Codex | Direct |
| **codex-swarm** | Swarm | Codex gpt-5.2 | Codex | Direct |

---

## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins
```

### Step 2: Install Plugins

```bash
# Standalone (no MCP required)
/plugin install pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install pair-swarm@claude-code-repoprompt-codex-plugins

# RepoPrompt planning
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins

# Codex planning
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins
```

Or enable in `.claude/settings.local.json`:

```json
{
  "enabledPlugins": {
    "pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "pair-swarm@claude-code-repoprompt-codex-plugins": true,
    "repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "repoprompt-swarm@claude-code-repoprompt-codex-plugins": true,
    "codex-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "codex-swarm@claude-code-repoprompt-codex-plugins": true
  }
}
```

---

## Usage

### Pipeline Plugins (Iterative Discovery)

```bash
# Claude planning (standalone)
/pair-pipeline:orchestrate command:start | task:Add user authentication

# RepoPrompt planning
/repoprompt-pair-pipeline:orchestrate command:start | task:Add user authentication

# Codex planning
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication

# With initial research
/pair-pipeline:orchestrate command:start | task:Add OAuth2 | research:Google OAuth2 best practices

# Resume with accumulated context
/pair-pipeline:orchestrate command:start-resume | task:Add password reset
```

### Swarm Plugins (One-Shot)

```bash
# Step 1: Create plan
/pair-swarm:plan task:Add logout button | research:Session handling best practices
/repoprompt-swarm:plan task:Add logout button | research:Session handling
/codex-swarm:plan task:Add logout button | research:Session handling

# Step 2: Execute plan
/pair-swarm:code plan:[paste plan from above]
/repoprompt-swarm:code chat_id:[chat_id from /plan]
/codex-swarm:code plan:[paste plan from above]
```

---

## Choosing the Right Plugin

### Decision Tree

```
Do you need iterative discovery with checkpoints?
│
├─► YES ─► Which planning engine?
│          │
│          ├─► RepoPrompt (best context) ─► repoprompt-pair-pipeline
│          ├─► Codex (gpt-5.2 reasoning) ─► codex-pair-pipeline
│          └─► None (standalone) ─────────► pair-pipeline
│
└─► NO ──► Which planning engine?
           │
           ├─► RepoPrompt ─► repoprompt-swarm
           ├─► Codex ──────► codex-swarm
           └─► None ───────► pair-swarm
```

### Quick Reference

| Use Case | Recommended |
|----------|-------------|
| Exploring unfamiliar codebase | `repoprompt-pair-pipeline` or `codex-pair-pipeline` |
| Complex architectural changes | `codex-pair-pipeline` (gpt-5.2 reasoning) |
| Best context management | `repoprompt-pair-pipeline` |
| Well-defined task, fast execution | `pair-swarm` (no MCP overhead) |
| No external dependencies | `pair-pipeline` or `pair-swarm` |
| Already using RepoPrompt | `repoprompt-*` variants |

### Tradeoffs

| Consideration | Pipeline | Swarm |
|---------------|----------|-------|
| User control | High (checkpoints) | Low (review plan only) |
| Speed | Slower (iterative) | Faster (one-shot) |
| Context quality | Higher (refined) | Fixed (initial gather) |
| Workflow | Single command | Two commands |

| Consideration | pair-* | repoprompt-* | codex-* |
|---------------|--------|--------------|---------|
| Dependencies | None | RepoPrompt app | Codex CLI + API key |
| Planning quality | Good | Better | Best (gpt-5.2) |
| Speed | Fastest | Medium | Slowest (30s-5min) |
| Resume capability | Context-based | chat_id-based | Limited |

---

## Requirements

### Base Requirements (All Plugins)

- **Claude Code** - The CLI for orchestration and execution
- **Node.js 18+** - For running Claude Code

### MCP Requirements by Planning Engine

| Planning Engine | MCP Server | Setup |
|-----------------|------------|-------|
| pair-* (Claude) | None | No additional setup |
| repoprompt-* | RepoPrompt MCP | [RepoPrompt App](https://repoprompt.com) |
| codex-* | [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) | See below |

#### Codex MCP Setup

```bash
# Prerequisites
npm i -g @openai/codex  # or: brew install codex
codex login --api-key "your-openai-api-key"

# Install MCP server
claude mcp add codex-cli -- npx -y codex-mcp-server
```

> **Note:** We use [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) instead of the official server due to a bug with conversation IDs ([Issue #3712](https://github.com/openai/codex/issues/3712)).

---

## Tips for Best Results

### Writing Good Task Descriptions

**Good:**
- "Add logout button that clears session storage and redirects to /login"
- "Fix login button not responding on mobile Safari - touch events not firing"
- "Add JWT authentication with refresh token rotation"

**Bad:**
- "Add logout"
- "Login broken"
- "Add auth"

### Using Research Effectively

Include `research:` when working with:
- External APIs: `research:Stripe API Node.js`
- Unfamiliar libraries: `research:React Query v5 mutations`
- Best practices: `research:JWT refresh token best practices`

### Handling BLOCKED Status

When a plan-coder returns BLOCKED:
1. Read the error details in the status
2. Fix the underlying issue
3. Re-run with `command:start-resume` (pipeline) or same plan (swarm)

---

## Directory Structure

```
.
├── .claude-plugin/
│   └── marketplace.json          # Plugin registry
├── pair-pipeline/                # Claude planning + Pipeline
├── pair-swarm/                   # Claude planning + Swarm
├── repoprompt-pair-pipeline/     # RepoPrompt planning + Pipeline
├── repoprompt-swarm/             # RepoPrompt planning + Swarm
├── codex-pair-pipeline/          # Codex planning + Pipeline
├── codex-swarm/                  # Codex planning + Swarm
└── README.md
```

Each plugin contains:
- `agents/` - Agent definitions (code-scout, doc-scout, planner, plan-coder)
- `commands/` - Slash commands (orchestrate, plan, code)
- `skills/` - Shared skills (code-quality)

---

## Troubleshooting

### MCP Connection Issues

**RepoPrompt:**
- Ensure RepoPrompt app is running
- Check that your project is open in RepoPrompt
- Verify MCP server status in RepoPrompt settings

**Codex:**
- Verify installation: `claude mcp list` should show `codex-cli`
- Check auth: `codex login --api-key "your-key"`
- Increase timeout for complex tasks (600000ms recommended)

### Plugin Not Found

1. Verify marketplace: `/plugin marketplace list`
2. Check `settings.local.json` has the plugin enabled
3. Restart Claude Code after changes

### Plan-Coder Failures

Common causes:
- Insufficient context from scouts
- Ambiguous plan instructions
- Pre-existing code issues

Solutions:
- Add more research at checkpoints (pipeline)
- Regenerate plan with more specific task/research (swarm)
- Fix issues and use `command:start-resume`

---

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with Claude Code
5. Submit a pull request

---

## License

MIT License - See LICENSE file for details.
