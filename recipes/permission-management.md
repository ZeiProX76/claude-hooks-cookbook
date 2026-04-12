# Permission Management

Auto-approve safe commands and block dangerous ones without clicking through every permission prompt.

## Goal

Eliminate permission fatigue. Safe operations like reading files and running tests should never ask for approval. Dangerous operations like `rm -rf` and `git push --force main` should never execute. Everything in between gets a second look.

## Hook Configuration

Add this to `.claude/settings.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/permission-check.mjs"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/security-block.mjs"
          }
        ]
      }
    ]
  }
}
```

Two hooks work together. The PermissionRequest hook auto-approves known safe commands. The PreToolUse hook blocks dangerous ones before they can run.

## Code: Auto-Approve Safe Commands

Save as `.claude/hooks/permission-check.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync } from "fs";

const input = JSON.parse(readFileSync(0, "utf-8"));
const command = input.tool_input?.command || "";

const SAFE_PREFIXES = [
  "npm test",
  "npm run lint",
  "npm run build",
  "npm run typecheck",
  "npx tsc",
  "npx eslint",
  "npx prettier",
  "git status",
  "git diff",
  "git log",
  "ls",
  "cat ",
  "echo ",
  "node --version",
];

for (const prefix of SAFE_PREFIXES) {
  if (command.startsWith(prefix)) {
    const output = {
      hookSpecificOutput: {
        hookEventName: "PermissionRequest",
        decision: { behavior: "allow" },
      },
    };
    console.log(JSON.stringify(output));
    process.exit(0);
  }
}

// Fall through to normal permission prompt for everything else
process.exit(0);
```

## Code: Block Dangerous Commands

Save as `.claude/hooks/security-block.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync } from "fs";

const input = JSON.parse(readFileSync(0, "utf-8"));

if (input.tool_name !== "Bash") process.exit(0);

const command = input.tool_input?.command || "";

const BLOCKED = [
  /\brm\s+.*-[a-z]*r[a-z]*f/i,
  /sudo\s+rm/i,
  /chmod\s+777/i,
  /git\s+push\s+--force.*main/i,
  /mkfs\s/i,
  /:\(\)\s*\{\s*:\|:\&\s*\}\s*;/,
];

for (const pattern of BLOCKED) {
  if (pattern.test(command)) {
    console.error("BLOCKED: dangerous command pattern detected");
    process.exit(2);
  }
}

process.exit(0);
```

## How It Works

The permission system has two layers.

**PermissionRequest** fires when Claude Code is about to show you a permission dialog. The hook checks the command against a list of safe prefixes. If it matches, the hook returns an `allow` decision and you never see the prompt. If it does not match, the hook exits cleanly and the normal permission dialog appears.

**PreToolUse** fires before any tool executes. The hook tests the command against regex patterns for destructive operations. If it matches, exit code `2` blocks the tool completely. Claude receives the error message and knows the command was rejected.

This two-layer approach gives you speed on safe commands and protection against dangerous ones. Commands that fall in between still go through the normal permission prompt, so you keep control where it matters.

Production AI build systems like [Build This Now](https://www.buildthisnow.com) use this pattern to let agents run freely while keeping hard guardrails on destructive operations.
