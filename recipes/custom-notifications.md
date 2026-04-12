# Custom Notifications

Send Slack messages, emails, or webhook alerts from Claude Code hook events.

## Goal

Get notified when important events happen during a Claude Code session: a task finishes, a build fails, compaction is about to fire, or an agent completes its work. This recipe uses async hooks and HTTP hooks to send notifications without slowing Claude down.

## Hook Configuration: Slack via Async Command

Add this to `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/notify-slack.mjs",
            "async": true
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/notify-slack.mjs",
            "async": true
          }
        ]
      }
    ]
  }
}
```

The `async: true` flag runs the notification in the background. Claude never waits for Slack to respond.

## Code: Slack Webhook Notifier

Save as `.claude/hooks/notify-slack.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync } from "fs";
import { request } from "https";

const SLACK_WEBHOOK = process.env.SLACK_WEBHOOK_URL;
if (!SLACK_WEBHOOK) process.exit(0);

const input = JSON.parse(readFileSync(0, "utf-8"));
const event = input.hook_event_name || "Unknown";
const sessionId = input.session_id || "unknown";

const messages = {
  Stop: ":white_check_mark: Claude finished a task",
  PreCompact: ":warning: Context compaction starting",
  SessionEnd: ":door: Session ended",
  PostToolUseFailure: ":x: Tool execution failed",
};

const text = messages[event] || `Hook event: ${event}`;
const payload = JSON.stringify({
  text: `${text}\nSession: \`${sessionId.slice(0, 8)}\``,
});

const url = new URL(SLACK_WEBHOOK);
const req = request({
  hostname: url.hostname,
  path: url.pathname,
  method: "POST",
  headers: { "Content-Type": "application/json" },
});

req.on("error", () => {});
req.write(payload);
req.end();

// Give the request a moment to flush
setTimeout(() => process.exit(0), 1000);
```

Set your webhook URL as an environment variable:

```bash
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/T00/B00/xxxx"
```

## Alternative: HTTP Hook for Remote Logging

Instead of running a local script, point an HTTP hook directly at your endpoint:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://your-server.com/hooks/claude-events",
            "timeout": 10,
            "headers": {
              "Authorization": "Bearer $HOOK_API_KEY"
            },
            "allowedEnvVars": ["HOOK_API_KEY"]
          }
        ]
      }
    ]
  }
}
```

Claude Code POSTs the event JSON to your URL. Your server receives the full event payload and can route it to Slack, email, PagerDuty, or any other service.

HTTP hooks that fail (non-2xx, timeout, connection error) are non-blocking by default. Notifications never interrupt Claude's work.

## Code: Email Notification via Node

Save as `.claude/hooks/notify-email.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync } from "fs";
import { request } from "https";

const API_KEY = process.env.RESEND_API_KEY;
if (!API_KEY) process.exit(0);

const input = JSON.parse(readFileSync(0, "utf-8"));
const event = input.hook_event_name || "Unknown";

const payload = JSON.stringify({
  from: "hooks@yourdomain.com",
  to: "you@yourdomain.com",
  subject: `Claude Code: ${event}`,
  text: `Event: ${event}\nSession: ${input.session_id || "unknown"}\nTime: ${new Date().toISOString()}`,
});

const req = request({
  hostname: "api.resend.com",
  path: "/emails",
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${API_KEY}`,
  },
});

req.on("error", () => {});
req.write(payload);
req.end();

setTimeout(() => process.exit(0), 1000);
```

## How It Works

Async hooks are the key. Setting `async: true` means the hook runs in the background while Claude continues working. This is critical for notifications because network calls can be slow or fail, and neither should block Claude's response.

HTTP hooks provide a second path. Instead of running a local script, Claude Code POSTs the event data directly to a URL you control. The `allowedEnvVars` field restricts which environment variables can be interpolated into headers, preventing accidental secret exposure.

Use async command hooks when you want full control over the notification logic (formatting, routing, retries). Use HTTP hooks when you already have a webhook endpoint and want zero local code.

For a complete reference on async hooks and HTTP hooks, see the [hooks guide at buildthisnow.com](https://www.buildthisnow.com/blog/tools/hooks/hooks-guide).
