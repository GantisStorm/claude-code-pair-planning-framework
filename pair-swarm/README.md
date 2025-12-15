# Pair Swarm

One-shot swarm commands for planning and coding. No iterative loops or checkpoints - just parallel agent execution.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install pair-swarm@claude-code-repoprompt-codex-plugins
```

## Commands

### /plan - Create Implementation Plan

Spawns code-scout and doc-scout in parallel, then creates an implementation plan.

```bash
/pair-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/pair-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/pair-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> planner -> plan output

**Output:** A structured implementation plan with per-file instructions, ready for `/code`.

### /code - Execute Implementation Plan

Takes a plan and spawns plan-coders in parallel to implement all files.

```bash
/pair-swarm:code plan:[paste plan from /plan output]
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
     │             │              └────┬─────┘  └────┬─────┘
     ▼             ▼                   │             │
 CODE_CONTEXT  EXTERNAL_               ▼             ▼
     │         CONTEXT              COMPLETE      COMPLETE
     └──────┬──────┘                   │             │
            │                          └──────┬──────┘
            ▼                                 │
     ┌───────────┐                            ▼
     │  planner  │                    ┌───────────────┐
     └─────┬─────┘                    │ Results Table │
           │                          └───────────────┘
           ▼
   ┌────────────────┐
   │ IMPLEMENTATION │
   │     PLAN       │
   │ files_to_edit  │
   │files_to_create │
   │ per-file instr │
   └────────────────┘
```

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT |
| planner | Create implementation plan | Read, Glob, Grep, Bash | File lists + instructions |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | Status + verified |

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- Review the plan before executing - make adjustments if needed
- If some files are BLOCKED, fix issues and re-run `/code` with the same plan

**When things go wrong:**
- BLOCKED status includes error details - read them
- Check if the plan had ambiguous instructions
- Try regenerating the plan with more specific task/research

## Comparison with pair-pipeline

| Feature | pair-swarm | pair-pipeline |
|---------|------------|---------------|
| Execution | One-shot | Iterative with checkpoints |
| User control | Review plan, then execute | Checkpoints during discovery |
| Use case | Well-defined tasks | Exploratory tasks |
| Commands | /plan + /code (separate) | /orchestrate (all-in-one) |

Use **pair-swarm** when you know what you want and just need fast parallel execution.

Use **pair-pipeline** when you need iterative discovery with user checkpoints.

## Requirements

- **Claude Code** - Orchestration and execution
