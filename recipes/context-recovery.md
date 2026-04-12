# Context Recovery

Back up session state automatically so compaction never destroys your progress.

## Goal

When Claude Code hits its context window limit, auto-compaction fires and summarizes the session. The summary captures the gist but loses specifics: exact error messages, function signatures, rejected approaches. This recipe creates structured backups at regular intervals so you can restore full detail after compaction.

## Hook Configuration

Add this to `.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "node .claude/hooks/context-monitor.mjs"
  },
  "hooks": {
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/precompact-backup.mjs",
            "async": true
          }
        ]
      }
    ]
  }
}
```

Two pieces work together. The StatusLine monitor tracks token usage every turn and triggers backups at thresholds. The PreCompact hook fires one final backup right before compaction happens.

## Code: StatusLine Monitor

Save as `.claude/hooks/context-monitor.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, writeFileSync, mkdirSync, existsSync } from "fs";
import { join } from "path";

const input = JSON.parse(readFileSync(0, "utf-8"));
const ctx = input.context_window || {};
const windowSize = ctx.context_window_size || 200000;
const pctRemain = ctx.remaining_percentage || 100;

// Account for the 33K autocompact buffer
const BUFFER_TOKENS = 33000;
const bufferPct = (BUFFER_TOKENS / windowSize) * 100;
const freeUntilCompact = Math.max(0, pctRemain - bufferPct);
const tokensUsed = Math.round(windowSize * (1 - pctRemain / 100));

// Load state
const stateDir = join(".claude", "hooks");
const statePath = join(stateDir, "context-state.json");
let state = { lastBackupTokens: 0, lastFree: 100, backupCount: 0 };

if (existsSync(statePath)) {
  try { state = JSON.parse(readFileSync(statePath, "utf-8")); } catch {}
}

// Token-based triggers: first at 50K, then every 10K
const FIRST_BACKUP = 50000;
const INTERVAL = 10000;
let shouldBackup = false;

if (tokensUsed >= FIRST_BACKUP) {
  if (state.lastBackupTokens < FIRST_BACKUP) shouldBackup = true;
  else if (tokensUsed - state.lastBackupTokens >= INTERVAL) shouldBackup = true;
}

// Percentage triggers as safety net
const THRESHOLDS = [30, 15, 5];
for (const t of THRESHOLDS) {
  if (state.lastFree > t && freeUntilCompact <= t) shouldBackup = true;
}

if (shouldBackup) {
  state.lastBackupTokens = tokensUsed;
  state.backupCount += 1;

  const backupDir = join(".claude", "backups");
  mkdirSync(backupDir, { recursive: true });

  const now = new Date();
  const name = `${state.backupCount}-backup-${now.toISOString().replace(/[:.]/g, "-")}.md`;
  const content = `# Session Backup\n\n**Tokens Used:** ${tokensUsed}\n**Free Until Compact:** ${freeUntilCompact.toFixed(1)}%\n**Timestamp:** ${now.toISOString()}\n`;

  writeFileSync(join(backupDir, name), content);
  state.currentBackup = name;
}

state.lastFree = freeUntilCompact;
writeFileSync(statePath, JSON.stringify(state, null, 2));

// Display status
const display = `${Math.round(freeUntilCompact)}% free (${Math.round(tokensUsed/1000)}K/${Math.round(windowSize/1000)}K)`;
const backupLine = state.currentBackup ? `\n-> .claude/backups/${state.currentBackup}` : "";
process.stdout.write(display + backupLine);
```

## Code: PreCompact Backup

Save as `.claude/hooks/precompact-backup.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, writeFileSync, mkdirSync } from "fs";
import { join } from "path";

const input = JSON.parse(readFileSync(0, "utf-8"));
const trigger = input.trigger || "unknown";

const backupDir = join(".claude", "backups");
mkdirSync(backupDir, { recursive: true });

const now = new Date();
const name = `precompact-${trigger}-${now.toISOString().replace(/[:.]/g, "-")}.md`;
const content = `# Pre-Compaction Backup\n\n**Trigger:** ${trigger}\n**Timestamp:** ${now.toISOString()}\n`;

writeFileSync(join(backupDir, name), content);
process.exit(0);
```

## How It Works

The StatusLine hook runs every turn and receives live token data. It tracks two things: total tokens used and the percentage of free context remaining. The 33K autocompact buffer is subtracted from the raw percentage to get an accurate "free until compaction" number.

Backups fire on two rails. Token-based triggers start at 50K tokens used and repeat every 10K after that. Percentage-based triggers at 30%, 15%, and 5% act as a safety net for smaller context windows.

The PreCompact hook catches the moment right before compaction fires. The `async: true` flag keeps it from slowing down the compaction process.

After compaction, run `/clear` to start fresh, then load your latest backup file. This avoids mixing the auto-generated compaction summary with your structured backup.

For the full three-file architecture with transcript parsing, see the [context recovery guide at buildthisnow.com](https://www.buildthisnow.com/blog/tools/hooks/context-recovery-hook).
