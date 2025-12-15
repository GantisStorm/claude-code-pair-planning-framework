# Claude Code Multi-Agent Plugins

A collection of multi-agent orchestration plugins for Claude Code. These plugins coordinate specialized agents to implement complex, multi-file coding tasks through parallel execution and intelligent planning.

## Table of Contents

- [Philosophy: Two Meta-Frameworks](#philosophy-two-meta-frameworks)
- [Common Agent Architecture](#common-agent-architecture)
- [Communication Patterns: The Key Difference](#communication-patterns-the-key-difference)
- [Plugin Matrix](#plugin-matrix)
- [Requirements](#requirements)
- [Installation](#installation)
- [Plugin Details](#plugin-details)
- [Choosing the Right Plugin](#choosing-the-right-plugin)
- [Tips for Best Results](#tips-for-best-results)
- [Troubleshooting](#troubleshooting)

---

## Philosophy: Two Meta-Frameworks

All six plugins are built on **two fundamental execution patterns**. Understanding these patterns is key to choosing the right plugin for your task.

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
                     │   (creates plan)   │
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
- User controls the discovery loop via checkpoints (AskUserQuestion)
- Can add research incrementally until context is complete
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
│          └─────┬─────┘               │           │
│                │                     │           │
│                ▼                     │           │
│       IMPLEMENTATION PLAN            │           │
│        (output to user)              │           │
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
- Scouts always run in parallel (both required)
- Best for: well-defined tasks, when you know what you want

**Plugins:** `pair-swarm`, `repoprompt-swarm`, `codex-swarm`

---

## Common Agent Architecture

All six plugins share the **same four agent types** with identical responsibilities. The only difference is HOW they communicate plans.

| Agent | Role | Input | Output |
|-------|------|-------|--------|
| **code-scout** | Investigate codebase for patterns, integration points, conventions | `task:` + `mode:` | `CODE_CONTEXT` |
| **doc-scout** | Fetch external documentation, APIs, best practices | `query:` | `EXTERNAL_CONTEXT` |
| **planner** | Synthesize context into implementation plan | Context package | Plan + file lists |
| **plan-coder** | Implement changes for ONE file, verify with code-quality | File + instructions | `COMPLETE` or `BLOCKED` |

### Agent Tools by Plugin Family

| Agent | pair-* | repoprompt-* | codex-* |
|-------|--------|--------------|---------|
| code-scout | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash |
| doc-scout | Research tools | Research tools | Research tools |
| planner | Read, Glob, Grep, Bash | `context_builder` / `chat_send` | `mcp__codex-cli__codex` |
| plan-coder | Read, Edit, Write, Bash | Read, Edit, Write, Bash, `chats` | Read, Edit, Write, Bash |

---

## Communication Patterns: The Key Difference

The **critical architectural difference** between plugin families is **how the implementation plan flows from planner to coders**.

### Pattern A: Direct Plan Distribution (pair-*, codex-*)

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
1. Planner creates the full implementation plan and returns it to the orchestrator
2. Orchestrator parses the plan, extracts per-file instructions
3. Orchestrator spawns plan-coders with instructions **embedded in their prompt**
4. Plan-coders receive everything they need - no external fetch required

**Used by:**
- **pair-pipeline / pair-swarm**: Planner is Claude Code itself, creates plan directly
- **codex-pipeline / codex-swarm**: Planner calls Codex MCP (gpt-5.2), receives full plan back

**Advantages:**
- No MCP dependency for coders
- Plan is immutable once created
- Simpler failure handling

---

### Pattern B: MCP Plan Storage & Fetch (repoprompt-*)

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
1. Planner calls `context_builder` or `chat_send` to create plan in RepoPrompt
2. RepoPrompt stores the plan in a chat session, returns `chat_id`
3. Orchestrator receives only `chat_id` + file lists (NOT the full plan)
4. Orchestrator spawns plan-coders with `chat_id` reference
5. Each plan-coder calls `mcp__RepoPrompt__chats` to fetch its instructions

**Used by:**
- **repoprompt-pair-pipeline / repoprompt-swarm**: All planning via RepoPrompt MCP

**Advantages:**
- RepoPrompt's intelligent context management
- Plan persists in RepoPrompt for review/iteration
- Can resume with `chat_id` reference

---

### Comparison Summary

| Aspect | pair-* (Direct) | codex-* (Direct) | repoprompt-* (MCP Fetch) |
|--------|-----------------|------------------|--------------------------|
| **Plan storage** | Orchestrator memory | Orchestrator memory | RepoPrompt MCP |
| **Plan passed to coders** | In prompt | In prompt | Via `chat_id` reference |
| **Coders fetch plan?** | No | No | Yes (from MCP) |
| **Coder MCP dependency** | None | None | Required |
| **Resume mechanism** | Accumulated context | Session ID (limited) | `chat_id` continuation |

---

## Plugin Matrix

| Plugin | Pattern | Plan Flow | MCP Required | Planning Engine |
|--------|---------|-----------|--------------|-----------------|
| **pair-pipeline** | Pipeline | Direct | None | Claude Code |
| **pair-swarm** | Swarm | Direct | None | Claude Code |
| **repoprompt-pair-pipeline** | Pipeline | MCP Fetch | RepoPrompt | RepoPrompt context_builder |
| **repoprompt-swarm** | Swarm | MCP Fetch | RepoPrompt | RepoPrompt context_builder |
| **codex-pair-pipeline** | Pipeline | Direct | Codex | Codex gpt-5.2 (high reasoning) |
| **codex-swarm** | Swarm | Direct | Codex | Codex gpt-5.2 (high reasoning) |

---

## Requirements

### Base Requirements (All Plugins)

- **Claude Code** - The CLI tool for orchestration and execution
- **Node.js 18+** - For running Claude Code

### MCP Server Requirements

| Plugin Family | MCP Server | Installation |
|---------------|------------|--------------|
| pair-* | None | No additional setup |
| repoprompt-* | RepoPrompt MCP | [RepoPrompt App](https://repoprompt.com) |
| codex-* | [Codex MCP Server](https://github.com/tuannvm/codex-mcp-server) | See below |

#### Codex MCP Server Setup (for codex-* plugins)

**Prerequisites:**
1. Install OpenAI Codex CLI v0.50.0+: `npm i -g @openai/codex` or `brew install codex`
2. Configure API key: `codex login --api-key "your-openai-api-key"`

**Install MCP Server:**
```bash
claude mcp add codex-cli -- npx -y codex-mcp-server
```

> **Note:** We use [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) instead of the official `codex mcp-server` because the official server has a bug where it doesn't return conversation IDs ([Issue #3712](https://github.com/openai/codex/issues/3712)), breaking multi-turn conversations.

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

# RepoPrompt-based
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins

# Codex-based
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

### Step 3: Start MCP Servers (if needed)

**For RepoPrompt plugins:**
1. Download and install [RepoPrompt](https://repoprompt.com)
2. Open your project in RepoPrompt
3. The MCP server starts automatically

**For Codex plugins:**
```bash
claude mcp add codex-cli -- npx -y codex-mcp-server
```

---

## Plugin Details

### pair-pipeline

**Standalone iterative pipeline with direct planning.**

No external dependencies. Claude Code handles discovery, planning, and execution.

```bash
# Start discovery loop
/pair-pipeline:orchestrate command:start | task:Add user authentication

# With initial research
/pair-pipeline:orchestrate command:start | task:Add OAuth2 | research:Google OAuth2 best practices

# Resume with accumulated context
/pair-pipeline:orchestrate command:start-resume | task:Add password reset
```

**Communication:** Planner returns full plan → Orchestrator distributes to coders in prompt

---

### pair-swarm

**Standalone one-shot swarm.**

Fast parallel execution without checkpoints.

```bash
# Create plan
/pair-swarm:plan task:Add logout button | research:Session handling best practices

# Execute plan
/pair-swarm:code plan:[paste plan from above]
```

**Communication:** Planner returns full plan → User copies to /code → Orchestrator distributes to coders

---

### repoprompt-pair-pipeline

**Iterative pipeline with RepoPrompt planning.**

RepoPrompt's context_builder creates detailed architectural plans.

```bash
# Start discovery loop
/repoprompt-pair-pipeline:orchestrate command:start | task:Add user authentication

# Resume in same RepoPrompt chat
/repoprompt-pair-pipeline:orchestrate command:start-resume | task:Add password reset

# Execute existing plan by chat_id
/repoprompt-pair-pipeline:orchestrate command:fetch | task:my-chat-id-ABC123
```

**Communication:** Planner stores in RepoPrompt → Returns `chat_id` → Coders fetch via `mcp__RepoPrompt__chats`

---

### repoprompt-swarm

**One-shot swarm with RepoPrompt planning.**

```bash
# Create plan via RepoPrompt
/repoprompt-swarm:plan task:Add logout button | research:Session handling

# Execute by chat_id
/repoprompt-swarm:code chat_id:[chat_id from /plan]
```

**Communication:** Planner stores in RepoPrompt → Returns `chat_id` → /code passes to coders → Coders fetch via MCP

---

### codex-pair-pipeline

**Iterative pipeline with Codex gpt-5.2 planning.**

Uses Codex with high reasoning effort and the Architect system prompt.

```bash
# Start discovery loop
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication

# Continue with new plan
/codex-pair-pipeline:orchestrate command:start-resume | task:Add rate limiting
```

**Communication:** Planner calls Codex MCP → Receives full plan → Orchestrator distributes to coders in prompt

**Architect System Prompt:**
```
You are a senior software architect specializing in code design and implementation planning...
[Detailed instructions for creating implementation plans with file locations, signatures, etc.]
```

---

### codex-swarm

**One-shot swarm with Codex gpt-5.2 planning.**

```bash
# Create plan via Codex
/codex-swarm:plan task:Add logout button | research:Session handling

# Execute plan
/codex-swarm:code plan:[paste full plan from /plan]
```

**Communication:** Planner calls Codex MCP → Returns full plan to user → User pastes to /code → Orchestrator distributes to coders

---

## Choosing the Right Plugin

### Decision Tree

```
Do you need iterative discovery with checkpoints?
│
├─► YES ─► Do you have an MCP preference?
│          │
│          ├─► RepoPrompt (best context) ─► repoprompt-pair-pipeline
│          ├─► Codex (gpt-5.2 reasoning) ─► codex-pair-pipeline
│          └─► None (standalone) ─────────► pair-pipeline
│
└─► NO ──► Do you have an MCP preference?
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

### Pattern vs. MCP Tradeoffs

| Consideration | Pipeline | Swarm |
|---------------|----------|-------|
| User control | High (checkpoints) | Low (review plan only) |
| Speed | Slower (iterative) | Faster (one-shot) |
| Context quality | Higher (refined) | Fixed (initial gather) |
| Workflow | Single command | Two commands |

| Consideration | pair-* | repoprompt-* | codex-* |
|---------------|--------|--------------|---------|
| Dependencies | None | RepoPrompt app | Codex CLI + API key |
| Planning quality | Good | Better (context_builder) | Best (gpt-5.2) |
| Speed | Fastest | Medium | Slowest (30s-5min) |
| Resume capability | Context-based | chat_id-based | Limited |

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
├── pair-pipeline/                # Standalone iterative pipeline
│   ├── agents/
│   │   ├── code-scout.md
│   │   ├── doc-scout.md
│   │   ├── planner-start.md
│   │   ├── planner-start-resume.md
│   │   └── plan-coder.md
│   ├── commands/
│   │   └── orchestrate.md
│   └── skills/
│       └── code-quality/
├── pair-swarm/                   # Standalone one-shot swarm
├── repoprompt-pair-pipeline/     # RepoPrompt iterative pipeline
├── repoprompt-swarm/             # RepoPrompt one-shot swarm
├── codex-pair-pipeline/          # Codex iterative pipeline
├── codex-swarm/                  # Codex one-shot swarm
└── README.md
```

---

## Troubleshooting

### MCP Connection Issues

**RepoPrompt:**
- Ensure RepoPrompt app is running
- Check that your project is open in RepoPrompt
- Verify MCP server status in RepoPrompt settings

**Codex:**
- Verify installation: `claude mcp list` should show `codex-cli`
- Check Codex CLI auth: `codex login --api-key "your-key"`
- Increase timeout for complex tasks (600000ms recommended)

### Plugin Not Found

1. Verify marketplace: `/plugin marketplace list`
2. Check `settings.local.json` has the plugin enabled
3. Restart Claude Code after changes

### Slow Execution

- Codex operations can take 30s-5min depending on complexity
- For faster execution, use `pair-*` plugins (no MCP overhead)
- Reduce task complexity by breaking into smaller pieces

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
