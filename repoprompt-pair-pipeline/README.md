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

Discovery loop, then continue in the same RepoPrompt chat. Preserves file selection context. Runs context optimization before planning.

```bash
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add password reset flow
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add rate limiting to the new endpoints
/repoprompt-pair-pipeline:orchestrate command:continue | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> **planner-context** -> RepoPrompt planning -> execution (continues existing RepoPrompt chat)

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> **planner-context** -> RepoPrompt planning -> execution

### command:fetch - Execute Existing Plan

Execute an existing RepoPrompt plan by chat ID.

```bash
/repoprompt-pair-pipeline:orchestrate command:fetch | task:parti-url-cleanup-EA6D74
```

**Flow:** planner-fetch -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool with `run_in_background: true`, then retrieves results via `TaskOutput`:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - RepoPrompt receives XML architectural instructions and creates detailed plan
4. **Execution** - Coders run in parallel as background tasks, results collected via TaskOutput

**Background execution pattern:**
- Agents spawn with `run_in_background: true`
- Orchestrator uses `TaskOutput task_id: [agent-id]` to retrieve results
- Enables true parallel execution of scouts and coders

| Command | Discovery | Planning |
|---------|-----------|----------|
| `command:start` | Checkpoints + optional research | RepoPrompt (planner-start) |
| `command:continue` | Checkpoints + optional research | planner-context -> RepoPrompt (planner-continue) |
| `command:fetch` | None | Fetches existing plan |

## Architecture Diagram

```
                                    ORCHESTRATOR
                                         │
           ┌─────────────────────────────┼─────────────────────────────┐
           │                             │                             │
           ▼                             ▼                             ▼
┌────────────────────┐        ┌────────────────────┐        ┌────────────────────┐
│   command:start    │        │  command:continue  │        │   command:fetch    │
└─────────┬──────────┘        └─────────┬──────────┘        └─────────┬──────────┘
          │                             │                             │
          ▼                             ▼                             │
┌─────────────────────────────────────────────────────────────────────┐    │
│                    ITERATIVE DISCOVERY LOOP                         │    │
│                    (Human-in-the-loop control)                      │    │
│                                                                     │    │
│  ┌─────────────────┐                                                │    │
│  │   code-scout    │  ◄── run_in_background: true                   │    │
│  └────────┬────────┘                                                │    │
│  ┌────────┴────────┐                                                │    │
│  │   TaskOutput    │  ◄── collect results                           │    │
│  └────────┬────────┘                                                │    │
│           │ CODE_CONTEXT                                            │    │
│           ▼                                                         │    │
│  ┌─────────────────┐                                                │    │
│  │   CHECKPOINT    │◄────────────────────────┐                      │    │
│  │ 1. Clarifying Qs│                         │                      │    │
│  │ 2. "Add research│                         │                      │    │
│  │  or "Complete"  │                         │                      │    │
│  └────────┬────────┘                         │                      │    │
│           │                                  │                      │    │
│      ┌────┴────┐                             │                      │    │
│      ▼         ▼                             │                      │    │
│  ┌─────────┐  "Complete"                     │                      │    │
│  │doc-scout│      │  ◄── run_in_background   │                      │    │
│  └────┬────┘      │                     (loop back)                 │    │
│  ┌────┴────┐      │                          │                      │    │
│  │TaskOutput      │                          │                      │    │
│  └────┬────┘      │                          │                      │    │
│       │ EXTERNAL  │                          │                      │    │
│       │ _CONTEXT  │                          │                      │    │
│       ▼           │                          │                      │    │
│  ┌─────────────────┐                         │                      │    │
│  │   CHECKPOINT    │─────────────────────────┘                      │    │
│  └───┬─────────────┘                                                │    │
│      │ "Complete"                                                   │    │
│      │                                                              │    │
│      │ context_package                                              │    │
└──────┼──────────────────────────────────────────────────────────────┘    │
       │                                                                   │
       ├─────────────────────────────────┐                                 │
       │                                 │                                 │
       ▼ (command:start)                 ▼ (command:continue)              │
┌─────────────────┐             ┌─────────────────┐                        │
│  planner-start  │             │ planner-context │                        │
│                 │             │                 │                        │
│ Creates new     │             │ Evaluates       │                        │
│ RepoPrompt chat │             │ workspace       │                        │
│ via rp-cli      │             │ selection,      │                        │
│ builder         │             │ adjusts files   │                        │
└────────┬────────┘             └────────┬────────┘                        │
         │                               │                                 │
         │                               ▼                                 │
         │                      ┌─────────────────┐                        │
         │                      │planner-continue │                        │
         │                      │                 │                        │
         │                      │ Continues chat  │                        │
         │                      │ via chat_send   │                        │
         │                      └────────┬────────┘                        │
         │                               │                                 │
         │ chat_id + file_lists          │ chat_id + file_lists            │
         └───────────────┬───────────────┘                                 │
                         │                                                 │
                         │                                      ┌──────────┴─────────┐
                         │                                      │   planner-fetch    │
                         │                                      │                    │
                         │                                      │ Fetches existing   │
                         │                                      │ plan from          │
                         │                                      │ RepoPrompt         │
                         │                                      └──────────┬─────────┘
                         │                                                 │
                         │                                                 │ chat_id + file_lists
                         └─────────────────────────┬───────────────────────┘
                                                   │
                                                   ▼
                        ┌───────────────────────────────────────────────────────┐
                        │              PARALLEL BACKGROUND CODERS               │
                        │                                                       │
                        │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │
                        │  │ plan-coder  │ │ plan-coder  │ │ plan-coder  │     │
                        │  │   file 1    │ │   file 2    │ │   file 3    │     │
                        │  │run_in_back- │ │run_in_back- │ │run_in_back- │     │
                        │  │ground: true │ │ground: true │ │ground: true │     │
                        │  │             │ │             │ │             │     │
                        │  │ Fetches plan│ │ Fetches plan│ │ Fetches plan│     │
                        │  │ via rp-cli  │ │ via rp-cli  │ │ via rp-cli  │     │
                        │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘     │
                        │         │               │               │             │
                        │         └───────────────┼───────────────┘             │
                        │                         ▼                             │
                        │               ┌─────────────────┐                     │
                        │               │   TaskOutput    │                     │
                        │               │ (collect all)   │                     │
                        │               └─────────────────┘                     │
                        └───────────────────────┬───────────────────────────────┘
                                                │
                                                │ status: COMPLETE/BLOCKED
                                                ▼
                                    ┌───────────────────────────┐
                                    │  ORCHESTRATOR Review      │
                                    │                           │
                                    │  Collects results,        │
                                    │  reports to user          │
                                    └───────────────────────────┘
```

**Key flows:**
- `command:start` -> discovery -> planner-start -> plan-coder
- `command:continue` -> discovery -> planner-context -> planner-continue -> plan-coder
- `command:fetch` -> planner-fetch -> plan-coder

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Synthesize prompt, create plan via RepoPrompt | Bash (rp-cli builder) | chat_id + file lists |
| planner-context | Evaluate and optimize workspace selection | Bash (rp-cli select, context, etc.) | selection_summary + ready_for_planning |
| planner-continue | Synthesize prompt for new task in existing chat | Bash (rp-cli chat) | chat_id + file lists |
| planner-fetch | Fetch existing plan (NO synthesis) | Bash (rp-cli chats) | chat_id + file lists |
| plan-coder | Implement single file (RepoPrompt mode) | Read, Edit, Write, Glob, Grep, Bash (rp-cli) | status + verified |

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
- Check if RepoPrompt is running and rp-cli is installed (`rp-cli --version`)

## Comparison with Other Plugins

| Feature | repoprompt-pair-pipeline | codex-pair-pipeline | repoprompt-swarm |
|---------|--------------------------|---------------------|------------------|
| Execution | Iterative with checkpoints | Iterative with checkpoints | One-shot |
| Planning | RepoPrompt (rp-cli) | Codex CLI (gpt-5.2) | RepoPrompt (rp-cli) |
| User control | Checkpoints during discovery | Checkpoints during discovery | Review plan, then execute |
| Commands | /orchestrate (all-in-one) | /orchestrate (all-in-one) | /plan + /code (separate) |
| Use case | Exploratory tasks with RepoPrompt | Exploratory tasks with Codex | Well-defined tasks |
| Session | chat_id based continuation | Direct plan passing | chat_id based |

Use **repoprompt-pair-pipeline** when you need iterative discovery with RepoPrompt's context management.

Use **codex-pair-pipeline** when you prefer Codex's gpt-5.2 with high reasoning effort.

Use **repoprompt-swarm** when you know what you want and just need fast parallel execution.

## See Also

For a simpler pipeline without RepoPrompt dependency, see **pair-pipeline** which uses full discovery with checkpoints but direct planning (no external CLI required).

## Requirements

- **RepoPrompt** - Required (running in background)
- **rp-cli** - Install via RepoPrompt Settings → MCP Server → "Install CLI to PATH"
- **Claude Code** - Orchestration, discovery, and execution
