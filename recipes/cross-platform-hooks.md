# Cross-Platform Hooks

Write hooks that run on macOS, Linux, and Windows without platform-specific wrappers.

## Goal

A hook written on macOS breaks on Windows the moment it references `bash` or `/tmp`. This recipe eliminates platform-specific code by using `node` as the universal runner and following three portable coding rules.

## Hook Configuration

Point every hook at `node` instead of `bash`, `cmd`, or `powershell`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/formatter.mjs"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/session-start.mjs"
          }
        ]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "node .claude/hooks/statusline.mjs"
  }
}
```

Node.js is a hard requirement of Claude Code on every platform, so `node` is always available on `$PATH`. No wrappers needed.

## The Three Rules

### 1. Use `os.homedir()` for Home Directories

```javascript
import { homedir } from "os";
import { join } from "path";

// Resolves to C:\Users\you on Windows, /home/you on Linux
const settingsPath = join(homedir(), ".claude", "settings.json");
```

Never reference `$HOME`, `%USERPROFILE%`, or `$env:USERPROFILE` directly.

### 2. Use `os.tmpdir()` for Temp Files

```javascript
import { tmpdir } from "os";
import { join } from "path";

// Resolves to the correct temp directory on any OS
const cacheFile = join(tmpdir(), "hook-cache.json");
```

Never hardcode `/tmp` or `$env:TEMP`.

### 3. Use `path.join()` for All Paths

```javascript
import { join } from "path";

// Uses the right separator (/ or \) automatically
const logFile = join(".claude", "hooks", "logs", "hook.log");
```

Never concatenate paths with `/` or `\\` by hand.

## Complete Example: Cross-Platform File Logger

Save as `.claude/hooks/file-logger.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, appendFileSync, mkdirSync } from "fs";
import { join } from "path";

const logDir = join(".claude", "hooks", "logs");
mkdirSync(logDir, { recursive: true });

try {
  const input = JSON.parse(readFileSync(0, "utf-8"));
  const toolName = input.tool_name;
  const filePath = input.tool_input?.file_path || "unknown";
  const timestamp = new Date().toISOString();

  appendFileSync(
    join(logDir, "file-changes.log"),
    `${timestamp} | ${toolName} | ${filePath}\n`
  );
} catch {
  // Silent fail so the hook never blocks Claude
}

process.exit(0);
```

Register it:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/file-logger.mjs"
          }
        ]
      }
    ]
  }
}
```

Same behavior on Windows 11, Arch Linux, and macOS. Zero wrappers.

## Cross-Platform Permissions

List both Windows and Unix command names in your permissions block:

```json
{
  "permissions": {
    "allow": [
      "Bash(where:*)",
      "Bash(which:*)",
      "Bash(ps:*)",
      "Bash(tasklist:*)",
      "Bash(node:*)"
    ]
  }
}
```

Whichever command does not exist on the current OS stays silent. Listing both costs nothing.

## Debugging

If a hook runs on one platform but fails on another:

1. Search `.mjs` files for `/` or `\\` in path strings. Replace with `path.join()`.
2. Look for `process.env.HOME` or `process.env.TEMP`. Replace with `os.homedir()` or `os.tmpdir()`.
3. Check `settings.json` for `bash`, `cmd`, or `powershell` in command fields. Replace with `node`.

Test any hook manually:

```bash
echo '{"tool_name":"Write","tool_input":{"file_path":"test.js"}}' | node .claude/hooks/your-hook.mjs
echo $?
```

A `0` exit code on all platforms means the hook is portable.

For more cross-platform patterns and a full release checklist, see the [cross-platform hooks guide at buildthisnow.com](https://www.buildthisnow.com/blog/tools/hooks/cross-platform-hooks).
