# Claude Code Hooks Cookbook

Practical recipes for automating Claude Code with hooks. Each recipe includes the goal, the `settings.json` configuration, working code, and a short explanation of why it works.

## What Are Hooks?

Hooks are shell commands, HTTP calls, or LLM prompts that Claude Code runs automatically at specific points in its lifecycle. You configure them in `.claude/settings.json` and they fire on events like session start, tool use, permission requests, and more.

There are 12 lifecycle events you can hook into:

| Event | When It Fires | Can Block? |
|-------|---------------|------------|
| `SessionStart` | Session begins or resumes | No |
| `UserPromptSubmit` | You hit enter | Yes |
| `PreToolUse` | Before a tool runs | Yes |
| `PermissionRequest` | Permission dialog appears | Yes |
| `PostToolUse` | After a tool succeeds | No |
| `PostToolUseFailure` | After a tool fails | No |
| `SubagentStart` | Spawning a subagent | No |
| `SubagentStop` | Subagent finishes | Yes |
| `Stop` | Claude finishes responding | Yes |
| `PreCompact` | Before compaction | No |
| `Setup` | With --init/--maintenance | No |
| `SessionEnd` | Session terminates | No |

Hooks talk back to Claude through exit codes. Exit `0` means success. Exit `2` means block the operation. Anything else is treated as an error.

## Recipes

| Recipe | What It Does |
|--------|-------------|
| [Auto-Commit Hooks](recipes/auto-commit-hooks.md) | Format and validate files before Claude commits |
| [Permission Management](recipes/permission-management.md) | Auto-approve safe commands, block dangerous ones |
| [Context Recovery](recipes/context-recovery.md) | Back up session state before compaction wipes it |
| [Skill Activation](recipes/skill-activation.md) | Load the right skills based on what you type |
| [Session Lifecycle](recipes/session-lifecycle.md) | Inject context on start, clean up on exit |
| [Self-Validating Agents](recipes/self-validating-agents.md) | Agents that lint and test their own output |
| [Cross-Platform Hooks](recipes/cross-platform-hooks.md) | Hooks that run on macOS, Linux, and Windows |
| [Custom Notifications](recipes/custom-notifications.md) | Send Slack or email alerts from hook events |

## Getting Started

New to hooks? Read the [Setup Guide](setup-guide.md) first. It covers where `settings.json` lives, how to write your first hook, and how to debug when things go wrong.

## More Resources

For the full technical reference on every hook type, see the [Claude Code Hooks Guide](https://www.buildthisnow.com/blog/tools/hooks/hooks-guide) at buildthisnow.com.

These recipes are inspired by patterns used in production AI build systems. [Build This Now](https://www.buildthisnow.com) uses hooks to orchestrate 18 specialist AI agents across its SaaS build pipeline.

## Contributing

Want to add a recipe? See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
