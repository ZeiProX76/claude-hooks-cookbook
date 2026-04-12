# Setup Guide

Everything you need to start using Claude Code hooks.

## Where Settings Live

Claude Code reads hook configuration from `settings.json` at three levels:

| Location | Scope | Priority |
|----------|-------|----------|
| `.claude/settings.json` | Project (shared with team) | High |
| `.claude/settings.local.json` | Project (personal, gitignored) | Medium |
| `~/.claude/settings.json` | All projects on your machine | Low |

Project settings override user settings. Use `.claude/settings.local.json` for anything personal (API keys, local paths) and `.claude/settings.json` for hooks the whole team should run.

## Your First Hook

Create `.claude/settings.json` in your project root:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

This auto-formats every file Claude writes. No approval needed. No extra steps.

## Hook Structure

Every hook entry has the same shape:

```json
{
  "matcher": "ToolName",
  "hooks": [
    {
      "type": "command",
      "command": "your-script-here",
      "timeout": 30
    }
  ]
}
```

**matcher**: Which tool or event triggers the hook. Uses regex. `"Write|Edit"` matches both Write and Edit. Leave it empty or omit it to match everything.

**type**: One of `command`, `http`, `prompt`, or `agent`.

**timeout**: Seconds before the hook gets killed. Defaults to 30.

## Hook Types

**Command**: Runs a shell command. The workhorse.

```json
{ "type": "command", "command": "python validate.py" }
```

**HTTP**: POSTs event data to a URL. Good for remote validation or logging.

```json
{ "type": "http", "url": "http://localhost:8080/hook", "timeout": 30 }
```

**Prompt**: Sends a question to the LLM and expects a JSON yes/no answer.

```json
{ "type": "prompt", "prompt": "Are all tasks complete? $ARGUMENTS" }
```

**Agent**: Spawns a subagent that can read files before deciding.

```json
{ "type": "agent", "prompt": "Verify test coverage for changed files" }
```

## Exit Codes

Exit codes are how your hook communicates with Claude Code:

| Code | Meaning |
|------|---------|
| `0` | Success. stdout is processed for JSON output. |
| `2` | Block. The operation is stopped. stderr is sent to Claude. |
| Other | Error. stderr shown to user, execution continues. |

Exit code `2` is the one that blocks things. Use it in `PreToolUse` to prevent a tool from running, or in `Stop` to force Claude to keep working.

## Async Hooks

Add `"async": true` to run a hook in the background without blocking Claude:

```json
{
  "type": "command",
  "command": "node backup-script.js",
  "async": true
}
```

Best for logging, backups, and notifications. Not suitable for security blocking or permission decisions.

## Environment Variables

These are available in every hook:

| Variable | Description |
|----------|-------------|
| `CLAUDE_PROJECT_DIR` | Project root directory |
| `CLAUDE_ENV_FILE` | Write env vars here (SessionStart, Setup only) |
| `CLAUDE_TOOL_INPUT_FILE_PATH` | File path from the tool input (PostToolUse) |

## Matcher Syntax

| Pattern | Matches |
|---------|---------|
| `""` or omitted | All tools |
| `"Bash"` | Only Bash (exact, case-sensitive) |
| `"Write\|Edit"` | Write OR Edit |
| `"mcp__memory__.*"` | All memory MCP tools |

No spaces around the `|`. Matchers are case-sensitive.

## Debugging

**Hook not firing?**
1. Check matcher syntax. Case matters.
2. Verify the settings file is in the right location.
3. Test manually: `echo '{"session_id":"test"}' | python your-hook.py`

**Command failing?**
1. Add logging: `command 2>&1 | tee ~/.claude/hook-debug.log`
2. Run Claude with `--debug` for verbose output.

**Infinite loop on Stop hooks?**
Always check the `stop_hook_active` flag before doing anything:

```python
if input_data.get('stop_hook_active', False):
    sys.exit(0)
```

## Disabling Hooks

Turn off all hooks at once:

```json
{
  "disableAllHooks": true
}
```

Use this when debugging or when a hook is misbehaving.

## Next Steps

Pick a recipe from the [cookbook](README.md) and try it out. Start with one hook that solves your biggest friction point, then add more as needed.

For the complete hook reference with all 12 lifecycle events, visit the [hooks guide at buildthisnow.com](https://www.buildthisnow.com/blog/tools/hooks/hooks-guide).
