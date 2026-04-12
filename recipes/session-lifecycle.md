# Session Lifecycle

Inject context when sessions start, persist environment variables, back up before compaction, and log on exit.

## Goal

Every new Claude Code session should open with the context it needs: current git branch, active tasks, environment variables. When the session ends, cleanup and logging should happen automatically. This recipe wires up all four lifecycle hooks.

## Hook Configuration

Add this to `.claude/settings.json`:

```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "npm install && echo 'Dependencies installed'",
            "timeout": 120
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
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/precompact.mjs",
            "async": true
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/session-end.mjs"
          }
        ]
      }
    ]
  }
}
```

## Code: Session Start

Save as `.claude/hooks/session-start.mjs`:

```javascript
#!/usr/bin/env node
import { execSync } from "child_process";
import { readFileSync, existsSync, appendFileSync } from "fs";

let context = "=== SESSION CONTEXT ===\n";

// Git info
try {
  const branch = execSync("git branch --show-current", { encoding: "utf-8" }).trim();
  const status = execSync("git status --porcelain", { encoding: "utf-8" }).trim();
  const changes = status ? status.split("\n").length : 0;
  context += `Branch: ${branch}\nUncommitted changes: ${changes}\n`;
} catch {
  context += "Git: not available\n";
}

// Active tasks
const taskFile = ".claude/tasks/current.md";
if (existsSync(taskFile)) {
  const tasks = readFileSync(taskFile, "utf-8").trim();
  context += `\nActive Tasks:\n${tasks}\n`;
}

context += "=== END ===";

// Persist env vars if the env file is available
const envFile = process.env.CLAUDE_ENV_FILE;
if (envFile) {
  appendFileSync(envFile, "export NODE_ENV=development\n");
}

const output = {
  hookSpecificOutput: {
    hookEventName: "SessionStart",
    additionalContext: context,
  },
};

console.log(JSON.stringify(output));
process.exit(0);
```

## Code: Session End Logger

Save as `.claude/hooks/session-end.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, appendFileSync, mkdirSync } from "fs";
import { join } from "path";

const input = JSON.parse(readFileSync(0, "utf-8"));

const logDir = join(".claude", "logs");
mkdirSync(logDir, { recursive: true });

const entry = {
  session_id: input.session_id,
  ended_at: new Date().toISOString(),
  reason: input.reason || "unknown",
};

appendFileSync(
  join(logDir, "session-history.jsonl"),
  JSON.stringify(entry) + "\n"
);

process.exit(0);
```

## Code: PreCompact Backup

Save as `.claude/hooks/precompact.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, writeFileSync, mkdirSync, existsSync, copyFileSync } from "fs";
import { join } from "path";

const input = JSON.parse(readFileSync(0, "utf-8"));
const transcriptPath = input.transcript_path || "";

if (transcriptPath && existsSync(transcriptPath)) {
  const backupDir = join(".claude", "backups");
  mkdirSync(backupDir, { recursive: true });

  const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
  copyFileSync(transcriptPath, join(backupDir, `transcript-${timestamp}.jsonl`));
}

process.exit(0);
```

## SessionStart Matchers

Target different session events with matchers:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [{ "type": "command", "command": "echo 'New session'" }]
      },
      {
        "matcher": "resume",
        "hooks": [{ "type": "command", "command": "echo 'Resumed'" }]
      },
      {
        "matcher": "compact",
        "hooks": [{ "type": "command", "command": "echo 'Post-compaction'" }]
      }
    ]
  }
}
```

Available matchers: `startup`, `resume`, `clear`, `compact`.

## How It Works

Four hooks cover the full session lifecycle.

**Setup** runs only when you pass `--init` to Claude Code. Use it for heavy one-time work like installing dependencies or running database migrations. The `--init-only` flag runs the hook and exits, which makes it CI-friendly.

**SessionStart** fires every time a session begins or resumes. Keep it fast. Load git status, active tasks, and any environment variables Claude needs for the current session.

**PreCompact** fires right before compaction. The `async: true` flag prevents it from slowing down compaction. This is your last chance to capture the full transcript before the summarizer reduces it.

**SessionEnd** fires when the session terminates. Log the session ID, timestamp, and exit reason for analytics or debugging.

These lifecycle hooks are the foundation of the hook system used in [Build This Now](https://www.buildthisnow.com) to manage agent sessions across complex multi-step builds.
