# Claude Code Multi-Agent Plugins

A collection of multi-agent orchestration plugins for Claude Code. These plugins coordinate specialized agents to implement complex, multi-file coding tasks through parallel execution and intelligent planning.

## Overview

| Plugin | Pattern | Planning | MCP Required | Best For |
|--------|---------|----------|--------------|----------|
| [pair-pipeline](#pair-pipeline) | Iterative | Direct | None | Exploratory tasks, no dependencies |
| [pair-swarm](#pair-swarm) | One-shot | Direct | None | Well-defined tasks, no dependencies |
| [repoprompt-pair-pipeline](#repoprompt-pair-pipeline) | Iterative | RepoPrompt | RepoPrompt | Exploratory tasks with RepoPrompt |
| [repoprompt-swarm](#repoprompt-swarm) | One-shot | RepoPrompt | RepoPrompt | Well-defined tasks with RepoPrompt |
| [codex-pair-pipeline](#codex-pair-pipeline) | Iterative | Codex | Codex MCP | Exploratory tasks with Codex |
| [codex-swarm](#codex-swarm) | One-shot | Codex | Codex MCP | Well-defined tasks with Codex |

### Pattern Comparison

**Iterative (Pipeline)** - Human-in-the-loop with checkpoints:
- Discovery phase with user checkpoints
- User can add research, clarify requirements
- Best for: complex features, unfamiliar code, exploratory work

**One-shot (Swarm)** - Fast parallel execution:
- `/plan` creates implementation plan
- `/code` executes plan in parallel
- Best for: well-defined tasks, when you know what you want

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

> **Note:** The `OPENAI_API_KEY` environment variable is no longer supported. You must use `codex login` to configure authentication.

---

## Installation

### Step 1: Add the Marketplace

In Claude Code, add the marketplace from GitHub:

```bash
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins
```

### Step 2: Install Plugins

Install the plugins you want to use:

```bash
/plugin install pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install pair-swarm@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins
```

Or enable them in your `.claude/settings.local.json`:

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
# Install the tuannvm/codex-mcp-server (provides working session support)
claude mcp add codex-cli -- npx -y codex-mcp-server
```

> **Note:** We use [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) instead of the official `codex mcp-server` because the official server has a bug where it doesn't return conversation IDs ([Issue #3712](https://github.com/openai/codex/issues/3712)), breaking multi-turn conversations.

---

## Plugin Details

---

## pair-pipeline

**Multi-agent orchestration without external dependencies.**

Iterative discovery with user checkpoints, direct planning, and parallel execution. Self-contained with no MCP requirements.

### Commands

```bash
# Start new task with discovery loop
/pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens

# Start with research (scouts run in parallel)
/pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices

# Resume with accumulated context
/pair-pipeline:orchestrate command:start-resume | task:Add password reset flow
```

### Flow

```
Discovery Loop (with checkpoints)
         |
         v
    code-scout --> CHECKPOINT --> optional doc-scout --> CHECKPOINT
         |
         v
    planner-start (or planner-start-resume)
         |
         v
    plan-coders (parallel) --> Results
```

### Agents

| Agent | Purpose | Output |
|-------|---------|--------|
| code-scout | Investigate codebase | CODE_CONTEXT |
| doc-scout | Fetch external docs | EXTERNAL_CONTEXT |
| planner-start | Create implementation plan | File lists + instructions |
| planner-start-resume | Continue with accumulated context | File lists + instructions |
| plan-coder | Implement single file | Status (COMPLETE/BLOCKED) |

---

## pair-swarm

**One-shot swarm commands without external dependencies.**

Fast parallel execution without iterative loops. Create a plan, then execute it.

### Commands

```bash
# Create implementation plan
/pair-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js

# Execute the plan
/pair-swarm:code plan:[paste plan from /plan output]
```

### Flow

```
/plan                              /code
  |                                  |
  v                                  v
code-scout + doc-scout        Parse plan
(parallel)                          |
  |                                  v
  v                           plan-coders (parallel)
planner                              |
  |                                  v
  v                            Results Table
IMPLEMENTATION PLAN
```

### Agents

| Agent | Purpose | Output |
|-------|---------|--------|
| code-scout | Investigate codebase | CODE_CONTEXT |
| doc-scout | Fetch external docs | EXTERNAL_CONTEXT |
| planner | Create implementation plan | File lists + instructions |
| plan-coder | Implement single file | Status (COMPLETE/BLOCKED) |

---

## repoprompt-pair-pipeline

**Multi-agent orchestration with RepoPrompt MCP.**

Iterative discovery with user checkpoints. RepoPrompt creates detailed architectural plans with intelligent context management.

### Requirements

- **RepoPrompt MCP** - Download from [repoprompt.com](https://repoprompt.com)

### Commands

```bash
# Start new task with discovery loop
/repoprompt-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens

# Start with research (scouts run in parallel)
/repoprompt-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices

# Resume in same RepoPrompt chat
/repoprompt-pair-pipeline:orchestrate command:start-resume | task:Add password reset flow

# Execute existing plan by chat_id
/repoprompt-pair-pipeline:orchestrate command:fetch | task:parti-url-cleanup-EA6D74
```

### Flow

```
Discovery Loop (with checkpoints)
         |
         v
    code-scout --> CHECKPOINT --> optional doc-scout --> CHECKPOINT
         |
         v
    planner-start (RepoPrompt context_builder)
         |
         v
    chat_id + file_lists
         |
         v
    plan-coders (fetch from RepoPrompt, parallel) --> Results
```

### Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | CODE_CONTEXT |
| doc-scout | Fetch external docs | Research tools | EXTERNAL_CONTEXT |
| planner-start | Create plan via RepoPrompt | context_builder | chat_id + file lists |
| planner-start-resume | Continue existing chat | chat_send | chat_id + file lists |
| planner-fetch | Fetch existing plan | chats | chat_id + file lists |
| plan-coder | Implement file (fetches from RepoPrompt) | Edit, Write, chats | Status |

---

## repoprompt-swarm

**One-shot swarm commands with RepoPrompt planning.**

Fast parallel execution with RepoPrompt's intelligent context handling.

### Requirements

- **RepoPrompt MCP** - Download from [repoprompt.com](https://repoprompt.com)

### Commands

```bash
# Create implementation plan via RepoPrompt
/repoprompt-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js

# Execute the plan
/repoprompt-swarm:code chat_id:[chat_id from /plan output]
```

### Flow

```
/plan                              /code
  |                                  |
  v                                  v
code-scout + doc-scout        Fetch plan from RepoPrompt
(parallel)                          |
  |                                  v
  v                           plan-coders (parallel)
planner (RepoPrompt)                 |
  |                                  v
  v                            Results Table
chat_id + file_lists
```

### Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | CODE_CONTEXT |
| doc-scout | Fetch external docs | Research tools | EXTERNAL_CONTEXT |
| planner | Create plan via RepoPrompt | context_builder | chat_id + file lists |
| plan-coder | Implement file (fetches from RepoPrompt) | Edit, Write, chats | Status |

---

## codex-pair-pipeline

**Multi-agent orchestration with Codex MCP using gpt-5.2 with high reasoning effort.**

Iterative discovery with user checkpoints. Codex creates detailed architectural plans using the Architect system prompt.

### Requirements

- **Codex MCP** - Install with `claude mcp add codex-cli -- npx -y codex-mcp-server`

### Commands

```bash
# Start new task with discovery loop
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens

# Start with research (scouts run in parallel)
/codex-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices

# Continue with new plan (builds on previous context)
/codex-pair-pipeline:orchestrate command:start-resume | task:Add password reset flow
```

### Flow

```
Discovery Loop (with checkpoints)
         |
         v
    code-scout --> CHECKPOINT --> optional doc-scout --> CHECKPOINT
         |
         v
    planner-start (Codex with gpt-5.2 + Architect prompt)
         |
         v
    Full plan + file_lists
         |
         v
    plan-coders (receive plan directly, parallel) --> Results
```

### Architect System Prompt

The planner sends this as `developer-instructions` to Codex:

```
You are a senior software architect specializing in code design and implementation planning. Your role is to:

1. Analyze the requested changes and break them down into clear, actionable steps
2. Create a detailed implementation plan that includes:
   - Files that need to be modified
   - Specific code sections requiring changes
   - New functions, methods, or classes to be added
   - Dependencies or imports to be updated
   - Data structure modifications
   - Interface changes
   - Configuration updates

For each change:
- Describe the exact location in the code where changes are needed
- Explain the logic and reasoning behind each modification
- Provide example signatures, parameters, and return types
- Note any potential side effects or impacts on other parts of the codebase
- Highlight critical architectural decisions that need to be made

You may include short code snippets to illustrate specific patterns, signatures, or structures, but do not implement the full solution.

Focus solely on the technical implementation plan - exclude testing, validation, and deployment considerations unless they directly impact the architecture.
```

### Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | CODE_CONTEXT |
| doc-scout | Fetch external docs | Research tools | EXTERNAL_CONTEXT |
| planner-start | Create plan via Codex | mcp__codex-cli__codex | Full plan + file lists |
| planner-start-resume | Create new plan via Codex | mcp__codex-cli__codex | Full plan + file lists |
| plan-coder | Implement file (receives plan directly) | Edit, Write | Status |

---

## codex-swarm

**One-shot swarm commands with Codex MCP planning using gpt-5.2 with high reasoning effort.**

Fast parallel execution with Codex's architectural planning capabilities.

### Requirements

- **Codex MCP** - Install with `claude mcp add codex-cli -- npx -y codex-mcp-server`

### Commands

```bash
# Create implementation plan via Codex
/codex-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js

# Execute the plan
/codex-swarm:code plan:[paste plan from /plan output]
```

### Flow

```
/plan                              /code
  |                                  |
  v                                  v
code-scout + doc-scout        Parse plan
(parallel)                          |
  |                                  v
  v                           plan-coders (parallel)
planner (Codex gpt-5.2)             |
  |                                  v
  v                            Results Table
Full plan + file_lists
```

### Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | CODE_CONTEXT |
| doc-scout | Fetch external docs | Research tools | EXTERNAL_CONTEXT |
| planner | Create plan via Codex | mcp__codex-cli__codex | Full plan + file lists |
| plan-coder | Implement file (receives plan directly) | Edit, Write | Status |

---

## Choosing the Right Plugin

### Decision Tree

```
Do you need iterative discovery with checkpoints?
  |
  +-- YES --> Do you have an MCP server preference?
  |             |
  |             +-- RepoPrompt --> repoprompt-pair-pipeline
  |             +-- Codex --> codex-pair-pipeline
  |             +-- None/Standalone --> pair-pipeline
  |
  +-- NO --> Do you have an MCP server preference?
              |
              +-- RepoPrompt --> repoprompt-swarm
              +-- Codex --> codex-swarm
              +-- None/Standalone --> pair-swarm
```

### Recommendation by Use Case

| Use Case | Recommended Plugin |
|----------|-------------------|
| Exploring unfamiliar codebase | repoprompt-pair-pipeline or codex-pair-pipeline |
| Quick feature implementation | repoprompt-swarm or codex-swarm |
| No external dependencies | pair-pipeline or pair-swarm |
| Complex architectural changes | codex-pair-pipeline (gpt-5.2) |
| Best context management | repoprompt-pair-pipeline |
| Maximum speed | pair-swarm (no MCP overhead) |

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

Include research when working with:
- External APIs: `research:Stripe API Node.js`
- Unfamiliar libraries: `research:React Query v5 mutations`
- Best practices: `research:JWT refresh token best practices`

### Handling Blocked Status

When a plan-coder returns BLOCKED:
1. Read the error details in the status
2. Fix the underlying issue
3. Re-run with `command:start-resume` (pipeline) or same ID (swarm)

### Parallel Execution

All plugins spawn plan-coders in parallel for maximum speed. Each coder:
1. Fetches its specific instructions
2. Implements the changes
3. Verifies with code quality checks
4. Reports status

---

## Directory Structure

```
.
├── .claude-plugin/
│   └── marketplace.json          # Plugin registry
├── pair-pipeline/                # Standalone iterative pipeline
│   ├── .claude-plugin/plugin.json
│   ├── README.md
│   ├── agents/
│   ├── commands/
│   └── skills/
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
- Run `codex mcp-server` in a separate terminal
- Check for port conflicts (default: varies)
- Ensure Codex CLI is installed: `npm list -g @anthropic/codex`

### Plugin Not Found

1. Verify marketplace is added: `/plugin marketplace list`
2. Check `settings.local.json` has the plugin enabled
3. Restart Claude Code after changes

### Slow Execution

- Codex operations can take 30s-5min depending on complexity
- Set appropriate timeouts in MCP Inspector (600000ms recommended)
- For faster execution, use pair-* plugins (no MCP overhead)

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
