# RepoPrompt Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks with RepoPrompt. Uses **iterative discovery** where users build context incrementally, then RepoPrompt creates detailed architectural plans.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
```

## Commands

### command:start - Discovery Loop

Iterative discovery with checkpoints. Best for complex features.

```bash
/repoprompt-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens
/repoprompt-pair-pipeline:orchestrate command:start | task:Fix login button not responding on mobile
/repoprompt-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> RepoPrompt planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> RepoPrompt planning -> execution

### command:continue - Continue Previous Plan

Discovery loop, then continue in the same RepoPrompt chat. Preserves file selection context.

```bash
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add password reset flow
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add rate limiting to the new endpoints
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> RepoPrompt planning -> execution (continues existing RepoPrompt chat)

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> RepoPrompt planning -> execution

### command:fetch - Execute Existing Plan

Execute an existing RepoPrompt plan by chat ID.

```bash
/repoprompt-pair-pipeline:orchestrate command:fetch | task:parti-url-cleanup-EA6D74
```

**Flow:** planner-fetch -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - RepoPrompt synthesizes an architectural narrative prompt and creates detailed plan
4. **Execution** - Coders implement in parallel, verify with code-quality skill

| Command | Discovery | Planning |
|---------|-----------|----------|
| `command:start` | Checkpoints + optional research | RepoPrompt (planner-start) |
| `command:continue` | Checkpoints + optional research | RepoPrompt (planner-continue) |
| `command:fetch` | None | Fetches existing plan |

## Architecture Diagram

```
                              ORCHESTRATOR
                                   │
           ┌───────────────────────┴───────────────────────┐
           │                                               │
           ▼                                               ▼
┌────────────────────┐                          ┌────────────────────┐
│  command:start or  │                          │   command:fetch    │
│  continue          │                          │                    │
└─────────┬──────────┘                          └─────────┬──────────┘
          │                                               │
          ▼                                               │ chat_id
┌─────────────────────────────────────────────────────────┐    │
│              ITERATIVE DISCOVERY LOOP                   │    │
│              (Human-in-the-loop control)                │    │
│                                                         │    │
│  ┌─────────────────┐                                    │    │
│  │   code-scout    │                                    │    │
│  └────────┬────────┘                                    │    │
│           │                                             │    │
│           │ CODE_CONTEXT                                │    │
│           ▼                                             │    │
│  ┌─────────────────┐                                    │    │
│  │   CHECKPOINT    │                                    │    │
│  │ (AskUserQuestion│                                    │    │
│  │ 1. Clarifying Qs│                                    │    │
│  │ 2. "Add research│                                    │    │
│  │  or "Complete"  │                                    │    │
│  └────────┬────────┘                                    │    │
│           │                                             │    │
│      ┌────┴────┐                                        │    │
│      │         │                                        │    │
│      ▼         │                                        │    │
│  ┌─────────┐   │                                        │    │
│  │doc-scout│   │                                        │    │
│  └────┬────┘   │                                        │    │
│       │        │                                        │    │
│       │ EXTERNAL_CONTEXT                                │    │
│       ▼        │                                        │    │
│  ┌─────────────────┐                                    │    │
│  │   CHECKPOINT    │                                    │    │
│  │ 1. Clarifying Qs│                                    │    │
│  │ 2. "Add more"   │                                    │    │
│  │  or "Complete"  │                                    │    │
│  └───┬────┬────────┘                                    │    │
│      │    │                                             │    │
│      │    └───► (loop back)                             │    │
│      │                                                  │    │
│      │ context_package                                  │    │
└──────┼──────────────────────────────────────────────────┘    │
       │                                                       │
       ▼                                                       ▼
┌─────────────────┐                                 ┌─────────────────┐
│  planner-start  │                                 │  planner-fetch  │
│       or        │                                 │                 │
│ planner-continue│                                 │ (fetches from   │
│ (RepoPrompt)    │                                 │  RepoPrompt)    │
│                 │                                 └────────┬────────┘
└────────┬────────┘                                          │
         │                                                   │
         │ chat_id + file_lists                              │ chat_id + file_lists
         ▼                                                   ▼
┌─────────────────┐                                 ┌─────────────────┐
│   plan-coder    │                                 │   plan-coder    │
│    (1/file)     │                                 │    (1/file)     │
│                 │                                 │                 │
│ Input:          │                                 │ Input:          │
│  chat_id        │                                 │  chat_id        │
│  target_file    │                                 │  target_file    │
│                 │                                 │                 │
│ Fetches plan    │                                 │ Fetches plan    │
│ from RepoPrompt │                                 │ from RepoPrompt │
└────────┬────────┘                                 └────────┬────────┘
         │                                                   │
         │ status: COMPLETE/BLOCKED                          │
         └─────────────────────┬─────────────────────────────┘
                               │
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
- `command:continue` -> discovery -> planner-continue -> plan-coder
- `command:fetch` -> planner-fetch -> plan-coder

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Synthesize prompt, create plan via RepoPrompt | context_builder | chat_id + file lists |
| planner-continue | Synthesize prompt for new task in existing chat | chat_send | chat_id + file lists |
| planner-fetch | Fetch existing plan (NO synthesis) | chats | chat_id + file lists |
| plan-coder | Implement single file (RepoPrompt mode) | Read, Edit, Write, Glob, Grep, Bash, chats | status + verified |

## Tips

**Choosing the right command:**
- `command:start` - Explore unfamiliar code, want checkpoints
- `command:continue` - Continue in same RepoPrompt chat
- `command:fetch` - Re-execute existing plan

**Getting good results:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari" not "Login broken"
- Use `research:` for external APIs

**When things go wrong:**
- BLOCKED status includes error details - read them
- Re-run with `command:continue` after fixing blockers
- Incomplete context? Add research at checkpoints

## See Also

For a simpler pipeline without RepoPrompt dependency, see **pair-pipeline** which uses full discovery with checkpoints but fast planning (no MCP required).

## Requirements

- **RepoPrompt MCP** - Required for all planning modes
- **Claude Code** - Orchestration, discovery, and execution
