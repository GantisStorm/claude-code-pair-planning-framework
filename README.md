# The Pair Planning Framework

A **pluggable multi-agent planning framework** for Claude Code. Swap planning engines (Claude, RepoPrompt, Codex, Gemini) while keeping the same discovery agents and execution patterns.

> **The Core Insight:** All planning engines follow the same meta-pattern: **Discovery → Planning → Execution**. The framework standardizes discovery (code-scout, doc-scout) and execution (plan-coder) while making the planning layer pluggable. This means you can switch between Claude's speed, RepoPrompt's file intelligence, Codex's reasoning depth, or Gemini's stateless simplicity - all using identical workflows.

## Core Concept: Pluggable Planning

```
┌───────────────────────────────────────────────────────────────────────┐
│                     PAIR PLANNING FRAMEWORK                           │
│                                                                       │
│   DISCOVERY (same for all)          PLANNING (choose one)             │
│   ────────────────────────          ─────────────────────             │
│                                                                       │
│   ┌─────────────┐                   ┌─────────────────────────────┐   │
│   │ code-scout  │                   │     Claude    (pair-*)      │   │
│   │ (codebase)  │                   ├─────────────────────────────┤   │
│   └──────┬──────┘    CONTEXT        │   RepoPrompt  (repoprompt-*)│   │
│          │          ─────────►      ├─────────────────────────────┤   │
│   ┌──────┴──────┐                   │     Codex     (codex-*)     │   │
│   │  doc-scout  │                   ├─────────────────────────────┤   │
│   │ (external)  │                   │     Gemini    (gemini-*)    │   │
│   └─────────────┘                   └──────────────┬──────────────┘   │
│                                                    │                  │
│                                                    │ PLAN             │
│                                                    ▼                  │
│                                                                       │
│   EXECUTION (same for all)                                            │
│   ────────────────────────                                            │
│                                                                       │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│   │ plan-coder  │  │ plan-coder  │  │ plan-coder  │  (parallel)       │
│   │   file 1    │  │   file 2    │  │   file 3    │                   │
│   └─────────────┘  └─────────────┘  └─────────────┘                   │
└───────────────────────────────────────────────────────────────────────┘
```

**The framework provides:**
- **Shared agents**: code-scout, doc-scout, plan-coder (same implementation across plugins)
- **Pluggable planners**: Swap between Claude, RepoPrompt (rp-cli), Codex CLI, or Gemini CLI
- **Two execution patterns**: Pipeline (iterative) or Swarm (one-shot)

---

## Execution Patterns

The framework offers **two execution patterns** that work with any planning engine.

### Pipeline Pattern (Iterative, Human-in-the-Loop)

```
┌────────────────────────────────────────────────────────────────────────┐
│                       ITERATIVE DISCOVERY LOOP                         │
│                                                                        │
│  code-scout ───► CHECKPOINT ───► doc-scout ───► CHECKPOINT ───► ...    │
│  (background)         │         (background)         │                 │
│       │          User decides         │         User decides           │
│       ▼         "Add research"        ▼        "Complete"              │
│  TaskOutput           │          TaskOutput          │                 │
│       │               │               │              │                 │
│       └───────────────┴───────────────┴──────────────┘                 │
│                               │                                        │
│                       context_package                                  │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
                                ▼
                     ┌────────────────────┐
                     │      PLANNER       │
                     │    (pluggable)     │
                     └─────────┬──────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │ plan-coder │  │ plan-coder │  │ plan-coder │  (background)
        │   file1    │  │   file2    │  │   file3    │
        └──────┬─────┘  └──────┬─────┘  └──────┬─────┘
               │               │               │
               └───────────────┼───────────────┘
                               ▼
                          TaskOutput
                        (collect all)
```

**Characteristics:**
- Single `/orchestrate` command handles entire workflow
- User controls discovery via checkpoints (AskUserQuestion)
- Incrementally add research until context is complete
- Best for: exploratory tasks, unfamiliar codebases, complex features

**Plugins:** `pair-pipeline`, `repoprompt-pair-pipeline`, `codex-pair-pipeline`, `gemini-pair-pipeline`

---

### Swarm Pattern (One-Shot, Fast Execution)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            /plan COMMAND                                │
│                                                                         │
│        task: "Add logout button" | research: "Session handling"         │
│                                    │                                    │
│                    ┌───────────────┴───────────────┐                    │
│                    ▼                               ▼                    │
│              ┌───────────┐                   ┌───────────┐              │
│              │code-scout │                   │ doc-scout │              │
│              │(background│                   │(background│              │
│              └─────┬─────┘                   └─────┬─────┘              │
│                    │           (parallel)          │                    │
│                    └───────────────┬───────────────┘                    │
│                                    ▼                                    │
│                               TaskOutput                                │
│                            (collect both)                               │
│                                    │                                    │
│                                    ▼                                    │
│                          ┌─────────────────┐                            │
│                          │     PLANNER     │                            │
│                          │   (pluggable)   │                            │
│                          └────────┬────────┘                            │
│                                   │                                     │
│                                   ▼                                     │
│                        IMPLEMENTATION PLAN                              │
│                        (files + instructions)                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            /code COMMAND                                │
│                                                                         │
│                   plan: [...] or chat_id: [...]                         │
│                                    │                                    │
│                              Parse input                                │
│                                    │                                    │
│                    ┌───────────────┼───────────────┐                    │
│                    ▼               ▼               ▼                    │
│              ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│              │plan-coder│   │plan-coder│   │plan-coder│                 │
│              │  file1   │   │  file2   │   │  file3   │                 │
│              │(backgrnd)│   │(backgrnd)│   │(backgrnd)│                 │
│              └────┬─────┘   └────┬─────┘   └────┬─────┘                 │
│                   │    (parallel)│              │                       │
│                   └──────────────┼──────────────┘                       │
│                                  ▼                                      │
│                             TaskOutput                                  │
│                           (collect all)                                 │
│                                  │                                      │
│                                  ▼                                      │
│                           RESULTS TABLE                                 │
│                    (file | status | summary)                            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Two separate commands: `/plan` then `/code`
- No checkpoints - review the plan, then execute
- Scouts always run in parallel
- Best for: well-defined tasks, fast execution

**Plugins:** `pair-swarm`, `repoprompt-swarm`, `codex-swarm`, `gemini-swarm`

---

## Planning Engines

### How Plans Flow: Direct vs MCP Fetch

The key architectural difference is **how plans move from planner to coders**.

#### Direct Plan Distribution (pair-*, codex-*, gemini-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│             FULL PLAN TEXT                  │  │
│  └───────────┘         │    (returned directly to orchestrator)      │  │
│                        └─────────────────────┬───────────────────────┘  │
│                                              │                          │
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

**Used by:** pair-*, codex-*, gemini-*

---

#### CLI Plan Storage & Fetch (repoprompt-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│              RepoPrompt (rp-cli)            │  │
│  │           │         │  ┌───────────────────────────────────────┐  │  │
│  │   calls   │         │  │     PLAN STORED IN CHAT SESSION       │  │  │
│  │ rp-cli    │         │  │         chat_id: abc123               │  │  │
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
│  │  via rp-cli     │                 │  via rp-cli     │                │
│  │  chats log      │                 │  chats log      │                │
│  └────────┬────────┘                 └────────┬────────┘                │
│           │                                   │                         │
│           ▼                                   ▼                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        RepoPrompt (rp-cli)                        │  │
│  │           (each coder fetches from same chat_id)                  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Planner stores plan in RepoPrompt via `rp-cli builder`, returns `chat_id`
2. Orchestrator only receives `chat_id` + file lists
3. Each plan-coder **fetches** its instructions via `rp-cli chats log`
4. Plan persists for review/resumption

**Used by:** repoprompt-*

---

### Planning Engine Comparison

| Aspect | pair-* (Claude) | codex-* (Codex) | repoprompt-* (RepoPrompt) | gemini-* (Gemini) |
|--------|-----------------|-----------------|---------------------------|-------------------|
| **Plan storage** | Orchestrator memory | Orchestrator memory | RepoPrompt (rp-cli) | Orchestrator memory |
| **Plan delivery** | In prompt | In prompt | Via `chat_id` fetch | In prompt |
| **External CLI** | None | codex | rp-cli | gemini |
| **Planning model** | Claude | gpt-5.2 (via CLI) | RepoPrompt builder | gemini-3-flash-preview (via CLI) |
| **Session continuation** | Accumulated context | Session ID | `chat_id` | Session index |

---

### The Meta Framework: What's Shared vs What's Different

All four planning engines implement the **same meta framework**. This is what makes them interchangeable:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        THE PAIR PLANNING FRAMEWORK                          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    SHARED ACROSS ALL ENGINES                        │   │
│   │                                                                     │   │
│   │  • code-scout agent (identical implementation)                      │   │
│   │  • doc-scout agent (identical implementation)                       │   │
│   │  • plan-coder agent (identical implementation*)                     │   │
│   │  • Orchestrator patterns (Pipeline or Swarm)                        │   │
│   │  • Input format: task: | research: | mode:                          │   │
│   │  • Output format: files_to_edit, files_to_create, per-file instrs   │   │
│   │                                                                     │   │
│   │  *repoprompt plan-coder adds rp-cli fetch capability                │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    DIFFERENT PER ENGINE                             │   │
│   │                                                                     │   │
│   │  • Planner implementation (how context → plan)                      │   │
│   │  • Plan format (Narrative vs XML)                                   │   │
│   │  • Plan delivery (direct vs CLI fetch)                              │   │
│   │  • External dependencies (none vs CLI)                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---
## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins
```

### Step 2: Install Plugins

```bash
# Standalone (no dependencies)
/plugin install pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install pair-swarm@claude-code-repoprompt-codex-plugins

# RepoPrompt planning (requires rp-cli)
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins

# Codex planning (requires CLI)
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins

# Gemini planning (requires CLI)
/plugin install gemini-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install gemini-swarm@claude-code-repoprompt-codex-plugins
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
    "codex-swarm@claude-code-repoprompt-codex-plugins": true,
    "gemini-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "gemini-swarm@claude-code-repoprompt-codex-plugins": true
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

# Gemini planning
/gemini-pair-pipeline:orchestrate command:start | task:Add user authentication

# With initial research
/pair-pipeline:orchestrate command:start | task:Add OAuth2 | research:Google OAuth2 best practices

# Continue session with new task
/pair-pipeline:orchestrate command:continue | task:Add password reset
```

### Swarm Plugins (One-Shot)

```bash
# Step 1: Create plan
/pair-swarm:plan task:Add logout button | research:Session handling best practices
/repoprompt-swarm:plan task:Add logout button | research:Session handling
/codex-swarm:plan task:Add logout button | research:Session handling
/gemini-swarm:plan task:Add logout button | research:Session handling

# Step 2: Execute plan
/pair-swarm:code plan:[paste plan from above]
/repoprompt-swarm:code chat_id:[chat_id from /plan]
/codex-swarm:code plan:[paste plan from above]
/gemini-swarm:code plan:[paste plan from above]
```

---

## Requirements

### Base Requirements (All Plugins)

- **Claude Code** - The CLI for orchestration and execution
- **Node.js 18+** - For running Claude Code

### External Requirements by Planning Engine

| Planning Engine | Requirement | Setup |
|-----------------|-------------|-------|
| pair-* (Claude) | None | No additional setup |
| repoprompt-* | rp-cli | See below |
| codex-* | Codex CLI | See below |
| gemini-* | Gemini CLI | See below |

#### rp-cli Setup

```bash
# Install rp-cli via RepoPrompt
# RepoPrompt Settings → MCP Server → "Install CLI to PATH"

# Verify
rp-cli --version
```

#### Codex CLI Setup

```bash
# Install Codex CLI
npm install -g @openai/codex
# or: brew install codex

# Authenticate
codex login

# Verify
codex --version
```

#### Gemini CLI Setup

```bash
# Install Gemini CLI
npm install -g @google/gemini-cli
# or: brew install gemini-cli (macOS)

# Verify
gemini --version
```

---

## Directory Structure

```
.
├── .claude-plugin/
│   └── marketplace.json          # Plugin registry
├── pair-pipeline/                # Claude planning + Pipeline
│   ├── agents/                   # code-scout, doc-scout, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality
├── pair-swarm/                   # Claude planning + Swarm
│   ├── agents/                   # code-scout, doc-scout, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality
├── repoprompt-pair-pipeline/     # RepoPrompt planning + Pipeline
│   ├── agents/                   # scouts, plan-coder (rp-cli), planners, planner-context
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, rp-cli
├── repoprompt-swarm/             # RepoPrompt planning + Swarm
│   ├── agents/                   # scouts, plan-coder (rp-cli), planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, rp-cli
├── codex-pair-pipeline/          # Codex planning + Pipeline
│   ├── agents/                   # scouts, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, codex-cli
├── codex-swarm/                  # Codex planning + Swarm
│   ├── agents/                   # scouts, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, codex-cli
├── gemini-pair-pipeline/         # Gemini planning + Pipeline
│   ├── agents/                   # scouts, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, gemini-cli
├── gemini-swarm/                 # Gemini planning + Swarm
│   ├── agents/                   # scouts, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, gemini-cli
└── README.md
```

---
