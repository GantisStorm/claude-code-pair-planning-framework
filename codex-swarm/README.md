# Codex Swarm

One-shot swarm commands with Codex MCP planning using [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server). No iterative loops or checkpoints - just parallel agent execution with Codex's intelligent planning using `gpt-5.2` with high reasoning effort.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins

# Install the Codex MCP server
claude mcp add codex-cli -- npx -y codex-mcp-server
```

## Commands

### /plan - Create Implementation Plan via Codex

Spawns code-scout and doc-scout in parallel, then uses Codex with the Architect system prompt to create an implementation plan.

```bash
/codex-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/codex-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/codex-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> Codex planner (gpt-5.2) -> full plan output

**Output:** A complete implementation plan with per-file instructions, ready for `/code`.

### /code - Execute Implementation Plan

Takes the plan from `/plan` and spawns plan-coders in parallel to implement all files. Each coder receives its instructions directly.

```bash
/codex-swarm:code plan:[paste plan from /plan output]
```

**Flow:** Parse plan -> spawn all plan-coders (parallel) -> collect results

**Output:** Status table showing which files were completed/blocked.

## Architecture Diagram

```
       /plan command                      /code command
             │                                  │
             ▼                                  ▼
     ┌───────────────┐                  ┌───────────────┐
     │  Parse Input  │                  │  Parse Plan   │
     │  task:        │                  │ files_to_edit │
     │  research:    │                  │files_to_create│
     └───────┬───────┘                  └───────┬───────┘
             │                                  │
             ▼                                  ▼
     ┌───────┴───────┐                  ┌───────┴───────┐
     │               │                  │               │
     ▼               ▼                  ▼               ▼
┌──────────┐  ┌──────────┐        ┌──────────┐  ┌──────────┐
│code-scout│  │doc-scout │        │plan-coder│  │plan-coder│
└────┬─────┘  └────┬─────┘        │  file1   │  │  file2   │
     │             │              │(receives │  │(receives │
     ▼             ▼              │ plan     │  │ plan     │
 CODE_CONTEXT  EXTERNAL_          │ directly)│  │ directly)│
     │         CONTEXT            └────┬─────┘  └────┬─────┘
     └──────┬──────┘                   │             │
            │                          ▼             ▼
            ▼                       COMPLETE      COMPLETE
     ┌───────────┐                     │             │
     │  planner  │                     └──────┬──────┘
     │  (Codex   │                            │
     │  gpt-5.2) │                            ▼
     └─────┬─────┘                    ┌───────────────┐
           │                          │ Results Table │
           ▼                          └───────────────┘
   ┌────────────────┐
   │   FULL PLAN    │
   │  + file lists  │
   └────────────────┘
```

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT |
| planner | Synthesize context, send to Codex | mcp__codex-cli__codex | Full plan + file lists |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | Status + verified |

## Plan Distribution

Plans are passed directly from `/plan` to `/code` - the orchestrator distributes per-file instructions to coders. This avoids session continuation issues and ensures each coder has exactly the instructions it needs.

The tuannvm/codex-mcp-server is still required for the planner to communicate with Codex, but coders don't need MCP access.

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- Copy the Implementation Plan section from `/plan` output
- If some files are BLOCKED, fix issues and re-run `/plan` with a modified task
- Each coder receives its instructions directly - no MCP needed

**When things go wrong:**
- BLOCKED status includes error details - read them
- Check if Codex MCP server is installed: `claude mcp list` should show `codex-cli`
- Try regenerating the plan with more specific task/research

## Comparison with Other Plugins

| Feature | codex-swarm | repoprompt-swarm | codex-pair-pipeline |
|---------|-------------|------------------|---------------------|
| Execution | One-shot | One-shot | Iterative with checkpoints |
| Planning | Codex MCP (gpt-5.2) | RepoPrompt MCP | Codex MCP (gpt-5.2) |
| User control | Review plan, then execute | Review plan, then execute | Checkpoints during discovery |
| Commands | /plan + /code (separate) | /plan + /code (separate) | /orchestrate (all-in-one) |
| Use case | Well-defined tasks with Codex | Well-defined tasks with RepoPrompt | Exploratory tasks |

Use **codex-swarm** when you know what you want and want Codex's gpt-5.2 architectural planning with high reasoning effort.

Use **repoprompt-swarm** when you prefer RepoPrompt's context handling.

Use **codex-pair-pipeline** when you need iterative discovery with user checkpoints.

## Requirements

- **Codex MCP** - [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) - Install with `claude mcp add codex-cli -- npx -y codex-mcp-server`
- **Claude Code** - Orchestration and execution
