# Self-Validating Agents

Agents that automatically lint, format, and test their own output before handing it back to you.

## Goal

Instead of reviewing agent output and discovering lint errors, missing exports, or failing tests after the fact, embed validation directly into the agent definition. The agent checks its own work at two levels: per-file (PostToolUse) and per-task (Stop).

## Hook Configuration

Hooks go in the agent's frontmatter, not in `settings.json`. This scopes the validation to just this agent.

```yaml
# .claude/agents/frontend-builder.md
---
name: frontend-builder
description: Build React components with automatic quality checks
model: sonnet
hooks:
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: 'npx eslint --fix "$CLAUDE_TOOL_INPUT_FILE_PATH" && npx prettier --write "$CLAUDE_TOOL_INPUT_FILE_PATH"'
  Stop:
    - hooks:
        - type: command
          command: "node .claude/hooks/validate-output.mjs"
---
You are a frontend builder agent. Create React components following
the project's established patterns. Every file you write is automatically
linted and formatted by your embedded hooks.
```

## Code: Output Validator (Stop Hook)

Save as `.claude/hooks/validate-output.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, existsSync } from "fs";
import { execSync } from "child_process";

const input = JSON.parse(readFileSync(0, "utf-8"));

// Prevent infinite loops
if (input.stop_hook_active) {
  process.exit(0);
}

const errors = [];

// Check that required files exist
const required = ["src/components/index.ts"];
for (const file of required) {
  if (!existsSync(file)) {
    errors.push(`Missing: ${file}`);
  }
}

// Check for barrel exports
if (existsSync("src/components/index.ts")) {
  const content = readFileSync("src/components/index.ts", "utf-8");
  if (!content.includes("export")) {
    errors.push("index.ts has no exports. Add barrel exports.");
  }
}

// Run the test suite
try {
  execSync("npm test -- --passWithNoTests", {
    timeout: 60000,
    stdio: "pipe",
  });
} catch {
  errors.push("Test suite is failing.");
}

if (errors.length > 0) {
  const output = {
    decision: "block",
    reason: "Output validation failed:\n- " + errors.join("\n- "),
  };
  console.log(JSON.stringify(output));
}

process.exit(0);
```

## Read-Only Reviewer Agent

For critical code paths, add a second agent that can only read files:

```yaml
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: Review agent output without modifying files
model: haiku
disallowedTools: Write, Edit, NotebookEdit
---
You are a read-only code reviewer. Your job:

1. Read all files the builder created or modified
2. Verify exports, type safety, and error handling
3. Run the test suite with Bash
4. Report issues as a list. Do NOT fix anything.
```

Pair the reviewer with the builder using task dependencies:

```
TaskCreate(subject="Build auth module", description="...")
TaskCreate(subject="Review auth module", description="Run code-reviewer on src/auth/")
TaskUpdate(taskId="2", addBlockedBy=["1"])
```

## How It Works

Self-validating agents operate at three tiers.

**Micro (PostToolUse)**: Every file write triggers ESLint and Prettier. Catches syntax errors, formatting issues, and simple lint violations immediately. The hooks are scoped to the agent definition, so different agents can have different validators.

**Macro (Stop)**: When the agent tries to finish, the Stop hook checks structural requirements. Are the expected files present? Do barrel exports exist? Does the test suite pass? If any check fails, the agent is blocked from completing and must fix the issues first.

**Team (Reviewer agent)**: A separate agent with read-only access reviews the builder's output. The `disallowedTools` field enforces this at the tool level. The reviewer reports issues without modifying anything.

Most quality problems get caught at the micro and macro tiers. The reviewer handles integration and architecture issues that per-file checks cannot see.

This three-tier validation pattern is used in [Build This Now](https://www.buildthisnow.com) to ensure every feature built by its 18 AI agents passes quality gates before reaching the user.
