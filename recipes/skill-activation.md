# Skill Activation

Automatically load the right skills based on what you type, so Claude always has the context it needs.

## Goal

Instead of manually invoking skills or hoping Claude remembers them from CLAUDE.md, this hook intercepts every prompt and appends skill recommendations before Claude reads it. Type "help me implement a feature" and the relevant skills land in context automatically.

## Hook Configuration

Add this to `.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/skill-activation.mjs"
          }
        ]
      }
    ]
  }
}
```

The hook fires on every prompt you submit, before Claude processes it.

## Skill Rules File

Create `.claude/skills/skill-rules.json` to define your triggers:

```json
{
  "skills": {
    "git-commits": {
      "priority": "high",
      "keywords": ["commit", "git push", "push my code", "save changes"],
      "intentPatterns": ["(create|make).*?commit"]
    },
    "testing": {
      "priority": "high",
      "keywords": ["test", "coverage", "jest", "vitest"],
      "intentPatterns": ["(write|add|create).*?tests?"]
    },
    "database": {
      "priority": "critical",
      "keywords": ["migration", "schema", "database", "postgres", "supabase"],
      "intentPatterns": ["(create|modify|update).*?(table|schema|migration)"]
    },
    "deployment": {
      "priority": "medium",
      "keywords": ["deploy", "vercel", "production", "hosting"],
      "intentPatterns": ["(deploy|ship|push).*?(production|staging|live)"]
    }
  }
}
```

## Code

Save as `.claude/hooks/skill-activation.mjs`:

```javascript
#!/usr/bin/env node
import { readFileSync, existsSync } from "fs";
import { join } from "path";

const input = JSON.parse(readFileSync(0, "utf-8"));
const prompt = (input.prompt || input.message || "").toLowerCase();

if (!prompt || prompt.length < 3) process.exit(0);

const rulesPath = join(".claude", "skills", "skill-rules.json");
if (!existsSync(rulesPath)) process.exit(0);

const rules = JSON.parse(readFileSync(rulesPath, "utf-8"));
const matches = { critical: [], high: [], medium: [], low: [] };

for (const [skill, config] of Object.entries(rules.skills)) {
  let matched = false;

  // Keyword matching
  for (const kw of config.keywords || []) {
    if (prompt.includes(kw.toLowerCase())) { matched = true; break; }
  }

  // Intent pattern matching
  if (!matched) {
    for (const pattern of config.intentPatterns || []) {
      if (new RegExp(pattern, "i").test(prompt)) { matched = true; break; }
    }
  }

  if (matched) {
    const priority = config.priority || "medium";
    matches[priority]?.push(skill);
  }
}

const allMatches = [
  ...matches.critical,
  ...matches.high,
  ...matches.medium,
  ...matches.low,
];

if (allMatches.length === 0) process.exit(0);

// Build the skill recommendation block
let block = "\n\nSKILL ACTIVATION CHECK\n";

if (matches.critical.length > 0) {
  block += "\nCRITICAL SKILLS (REQUIRED):\n";
  matches.critical.forEach(s => { block += `  -> ${s}\n`; });
}
if (matches.high.length > 0) {
  block += "\nRECOMMENDED SKILLS:\n";
  matches.high.forEach(s => { block += `  -> ${s}\n`; });
}
if (matches.medium.length > 0 || matches.low.length > 0) {
  block += "\nOPTIONAL SKILLS:\n";
  [...matches.medium, ...matches.low].forEach(s => { block += `  -> ${s}\n`; });
}

block += "\nACTION: Use Skill tool BEFORE responding\n";

const output = {
  hookSpecificOutput: {
    hookEventName: "UserPromptSubmit",
    additionalContext: block,
  },
};

console.log(JSON.stringify(output));
process.exit(0);
```

## How It Works

When you hit enter, the hook reads your prompt and checks it against `skill-rules.json`. Two matching strategies run in parallel.

**Keywords** are literal substring checks. If your prompt contains "commit" anywhere, the `git-commits` skill fires.

**Intent patterns** use regex to catch natural phrasing. The pattern `(create|make).*?commit` picks up "create a commit" and "make a quick commit" without needing every variation in the keyword list.

Matched skills are grouped by priority (critical, high, medium, low) and appended to your prompt as a structured block. Claude sees both your original message and the skill recommendations in a single input.

The hook finishes in milliseconds. No perceptible delay.

To add a new skill, edit `skill-rules.json` with keywords that match how you actually phrase things. [Build This Now](https://www.buildthisnow.com) uses this pattern to route 21 skill categories through a single UserPromptSubmit hook.
