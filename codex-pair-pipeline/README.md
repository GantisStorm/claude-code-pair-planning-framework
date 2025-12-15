# Codex Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks with Codex MCP using [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server). Uses **iterative discovery** where users build context incrementally, then Codex creates detailed architectural plans using the Architect system prompt.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins

# Install the Codex MCP server
claude mcp add codex-cli -- npx -y codex-mcp-server
```

## Commands

### command:start - Discovery Loop

Iterative discovery with checkpoints. Best for complex features.

```bash
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens
/codex-pair-pipeline:orchestrate command:start | task:Fix login button not responding on mobile
/codex-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Codex planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Codex planning -> execution

### command:start-resume - Continue with New Plan

Discovery loop, then create a new plan (builds on previous context).

```bash
/codex-pair-pipeline:orchestrate command:start-resume | task:Add password reset flow
/codex-pair-pipeline:orchestrate command:start-resume | task:Add rate limiting to the new endpoints
/codex-pair-pipeline:orchestrate command:start-resume | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Codex planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Codex planning -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - Codex (using `gpt-5.2` model with high reasoning effort) receives Architect system prompt + CODE_CONTEXT + architectural narrative and creates detailed plan with per-file instructions
4. **Execution** - Coders receive plan directly and implement in parallel, verify with code-quality skill

| Command | Discovery | Planning |
|---------|-----------|----------|
| `command:start` | Checkpoints + optional research | Codex (planner-start) |
| `command:start-resume` | Checkpoints + optional research | Codex (planner-start-resume) |

## Architecture Diagram

```
                              ORCHESTRATOR
                                   │
                                   ▼
                        ┌────────────────────┐
                        │  command:start or  │
                        │  start-resume      │
                        └─────────┬──────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│              ITERATIVE DISCOVERY LOOP                   │
│              (Human-in-the-loop control)                │
│                                                         │
│  ┌─────────────────┐                                    │
│  │   code-scout    │                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│           │ CODE_CONTEXT                                │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │   CHECKPOINT    │                                    │
│  │ (AskUserQuestion│                                    │
│  │ 1. Clarifying Qs│                                    │
│  │ 2. "Add research│                                    │
│  │  or "Complete"  │                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│      ┌────┴────┐                                        │
│      │         │                                        │
│      ▼         │                                        │
│  ┌─────────┐   │                                        │
│  │doc-scout│   │                                        │
│  └────┬────┘   │                                        │
│       │        │                                        │
│       │ EXTERNAL_CONTEXT                                │
│       ▼        │                                        │
│  ┌─────────────────┐                                    │
│  │   CHECKPOINT    │                                    │
│  │ 1. Clarifying Qs│                                    │
│  │ 2. "Add more"   │                                    │
│  │  or "Complete"  │                                    │
│  └───┬────┬────────┘                                    │
│      │    │                                             │
│      │    └───► (loop back)                             │
│      │                                                  │
│      │ context_package                                  │
└──────┼──────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────┐
│  planner-start  │
│       or        │
│ planner-start-  │
│      resume     │
│ (Codex gpt-5.2) │
└────────┬────────┘
         │
         │ FULL PLAN + file_lists
         ▼
┌─────────────────┐
│   plan-coder    │
│    (1/file)     │
│                 │
│ Input:          │
│  target_file    │
│  action         │
│  plan (direct)  │
└────────┬────────┘
         │
         │ status: COMPLETE/BLOCKED
         ▼
┌─────────────────┐
│  ORCHESTRATOR   │
│ Phase 3 Review  │
│                 │
│ Collects results│
│ Reports to user │
└─────────────────┘
```

**Key flows:**
- `command:start` -> discovery -> planner-start -> plan-coder
- `command:start-resume` -> discovery -> planner-start-resume -> plan-coder

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Synthesize prompt, create plan via Codex | mcp__codex-cli__codex | Full plan + file lists |
| planner-start-resume | Synthesize prompt, create new plan via Codex | mcp__codex-cli__codex | Full plan + file lists |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | status + verified |

## Plan Distribution

Plans are passed directly from planners to coders - the orchestrator distributes per-file instructions. This avoids session continuation issues and ensures each coder has exactly the instructions it needs.

The tuannvm/codex-mcp-server is still required for planners to communicate with Codex, but coders don't need MCP access.

## Tips

**Choosing the right command:**
- `command:start` - Explore unfamiliar code, want checkpoints
- `command:start-resume` - Build on previous context with a new plan

**Getting good results:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari" not "Login broken"
- Use `research:` for external APIs

**When things go wrong:**
- BLOCKED status includes error details - read them
- Re-run with `command:start-resume` after fixing blockers
- Incomplete context? Add research at checkpoints

## See Also

For RepoPrompt-based planning, see **repoprompt-pair-pipeline** which uses RepoPrompt's context_builder for planning.

## Requirements

- **Codex MCP** - [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) - Install with `claude mcp add codex-cli -- npx -y codex-mcp-server`
- **Claude Code** - Orchestration, discovery, and execution
