# RepoPrompt Swarm

One-shot swarm commands with RepoPrompt planning. No iterative loops or checkpoints - just parallel agent execution with RepoPrompt's intelligent planning.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins
```

## Commands

### /plan - Create Implementation Plan via RepoPrompt

Spawns code-scout and doc-scout in parallel, then uses RepoPrompt to create an implementation plan.

```bash
/repoprompt-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/repoprompt-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/repoprompt-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> RepoPrompt planner -> chat_id output

**Output:** A chat_id referencing the RepoPrompt plan, ready for `/code`.

### /code - Execute RepoPrompt Plan

Takes a chat_id and spawns plan-coders in parallel to implement all files. Coders fetch their instructions from RepoPrompt.

```bash
/repoprompt-swarm:code chat_id:[chat_id from /plan output]
```

**Flow:** Fetch plan from RepoPrompt -> spawn all plan-coders (parallel) -> collect results

**Output:** Status table showing which files were completed/blocked.

## Architecture Diagram

```
       /plan command                      /code command
             │                                  │
             ▼                                  ▼
     ┌───────────────┐                  ┌───────────────┐
     │  Parse Input  │                  │  Fetch Plan   │
     │  task:        │                  │from RepoPrompt│
     │  research:    │                  │  (chat_id)    │
     └───────┬───────┘                  └───────┬───────┘
             │                                  │
             ▼                                  ▼
     ┌───────┴───────┐                  ┌───────┴───────┐
     │               │                  │               │
     ▼               ▼                  ▼               ▼
┌──────────┐  ┌──────────┐        ┌──────────┐  ┌──────────┐
│code-scout│  │doc-scout │        │plan-coder│  │plan-coder│
└────┬─────┘  └────┬─────┘        │  file1   │  │  file2   │
     │             │              │(fetches  │  │(fetches  │
     ▼             ▼              │ from RP) │  │ from RP) │
 CODE_CONTEXT  EXTERNAL_          └────┬─────┘  └────┬─────┘
     │         CONTEXT                 │             │
     └──────┬──────┘                   ▼             ▼
            │                       COMPLETE      COMPLETE
            ▼                          │             │
     ┌───────────┐                     └──────┬──────┘
     │  planner  │                            │
     │  (uses    │                            ▼
     │ RepoPrompt│                    ┌───────────────┐
     └─────┬─────┘                    │ Results Table │
           │                          └───────────────┘
           ▼
   ┌────────────────┐
   │    CHAT_ID     │
   │  + file lists  │
   └────────────────┘
```

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT |
| planner | Synthesize context, send to RepoPrompt | mcp__RepoPrompt__context_builder | chat_id + file lists |
| plan-coder | Implement single file (fetches from RepoPrompt) | Read, Edit, Write, Glob, Grep, Bash, mcp__RepoPrompt__chats | Status + verified |

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- The plan is stored in RepoPrompt - just pass the chat_id
- If some files are BLOCKED, fix issues and re-run `/code` with the same chat_id
- Each coder fetches its instructions from RepoPrompt independently

**When things go wrong:**
- BLOCKED status includes error details - read them
- Check if RepoPrompt MCP is running
- Try regenerating the plan with more specific task/research

## Comparison with Other Plugins

| Feature | repoprompt-swarm | pair-swarm | repoprompt-pair-pipeline |
|---------|------------------|------------|--------------------------|
| Execution | One-shot | One-shot | Iterative with checkpoints |
| Planning | RepoPrompt MCP | Direct (no MCP) | RepoPrompt MCP |
| User control | Review plan, then execute | Review plan, then execute | Checkpoints during discovery |
| Commands | /plan + /code (separate) | /plan + /code (separate) | /orchestrate (all-in-one) |
| Use case | Well-defined tasks with RepoPrompt | Well-defined tasks, no MCP | Exploratory tasks |

Use **repoprompt-swarm** when you know what you want and want RepoPrompt's intelligent planning.

Use **pair-swarm** when you don't need RepoPrompt (no MCP dependency).

Use **repoprompt-pair-pipeline** when you need iterative discovery with user checkpoints.

## Requirements

- **RepoPrompt MCP** - Required for planning and plan retrieval
- **Claude Code** - Orchestration and execution
