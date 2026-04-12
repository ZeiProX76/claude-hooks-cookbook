# Auto-Commit Hooks

Format, lint, and validate files automatically every time Claude writes or edits code.

## Goal

Run formatters and linters on every file Claude touches, so code is always clean before it reaches your review. This replaces manual formatting passes and catches lint errors the moment they appear.

## Hook Configuration

Add this to `.claude/settings.json`:

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
          },
          {
            "type": "command",
            "command": "npx eslint --fix \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

Both hooks run in parallel after every Write or Edit operation. The file is formatted and linted before Claude's response appears.

## Adding a Build Gate

To prevent Claude from finishing a task while lint errors or type errors remain, pair the PostToolUse hooks with a Stop hook:

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/lint-gate.mjs"
          }
        ]
      }
    ]
  }
}
```

## Code

Save this as `.claude/hooks/lint-gate.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync } from "fs";
import { execSync } from "child_process";

const input = JSON.parse(readFileSync(0, "utf-8"));

// Prevent infinite loops
if (input.stop_hook_active) {
  process.exit(0);
}

try {
  execSync("npx eslint src/ --max-warnings=0", {
    timeout: 60000,
    stdio: "pipe",
  });
} catch (err) {
  const output = {
    decision: "block",
    reason: "Lint errors remain. Fix them before completing the task.",
  };
  console.log(JSON.stringify(output));
  process.exit(0);
}

process.exit(0);
```

## How It Works

The system operates at two levels.

**Per-file (PostToolUse)**: Every time Claude writes or edits a file, Prettier reformats it and ESLint auto-fixes what it can. This catches most formatting and simple lint issues immediately.

**Per-task (Stop)**: When Claude tries to finish a response, the Stop hook runs ESLint across the entire `src/` directory. If any errors remain, Claude is blocked from stopping and must fix them first.

The `stop_hook_active` check at the top of the script prevents infinite loops. Without it, Claude would block itself forever if it cannot resolve the lint errors.

You can extend this pattern to include type checking (`npx tsc --noEmit`), test runs (`npm test`), or any other validation. Stack multiple checks in the Stop hook to enforce whatever "done" means for your project.

For a complete breakdown of Stop hook patterns including test gates and build validation, see the [hooks guide at buildthisnow.com](https://www.buildthisnow.com/blog/tools/hooks/stop-hook-task-enforcement).
