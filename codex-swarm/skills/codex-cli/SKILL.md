---
name: codex-cli
description: Reference for Codex CLI usage patterns. Consult before calling codex via Bash.
---

# Codex CLI Reference

Quick reference for Codex CLI commands.

## Basic Command

```bash
codex exec "[prompt]" -m gpt-5.2 --reasoning-effort high --sandbox read-only --ask-for-approval never 2>&1
```

## Common Flags

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive execution mode |
| `-m gpt-5.2` | Model selection |
| `--reasoning-effort high` | Reasoning effort (minimal, low, medium, high, xhigh) |
| `--sandbox read-only` | Prevent file modifications |
| `--sandbox workspace-write` | Allow writes within workspace |
| `--ask-for-approval never` | Non-interactive mode |
| `--search` | Enable web search |
| `-C DIR` | Set working directory |

## Session Continuation

```bash
# Resume a previous session
codex resume [session-id] "[prompt]" --sandbox read-only --ask-for-approval never 2>&1
```

Note: Session IDs are returned in Codex output.

## Bash Execution Notes

- Use `dangerouslyDisableSandbox: true` for Bash calls
- Always append `2>&1` to capture all output
- Use timeout of 300000ms (5 min) or longer for complex tasks

## Safety Requirements

**NEVER use these flags:**
- `--dangerously-bypass-approvals-and-sandbox` - FORBIDDEN
- `--sandbox danger-full-access` - FORBIDDEN

## Troubleshooting

**Timeout**: Complex analysis may take 5-10 minutes. Use longer timeout values.

**Authentication**: Codex CLI must be authenticated via `codex login`.

**Flag errors**: Run `codex --help` to verify correct flag usage.

## More Information

- CLI reference: `codex --help`
- Official docs: https://github.com/openai/codex
