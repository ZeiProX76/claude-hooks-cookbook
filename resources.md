# Resources

Useful references for working with Claude Code hooks.

## Official Documentation

- [Claude Code Hooks Guide](https://www.buildthisnow.com/blog/tools/hooks/hooks-guide): Complete reference for all 12 hook lifecycle events, exit codes, JSON output format, and configuration options.
- [Anthropic Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code): Official Claude Code documentation from Anthropic.

## Hook Guides by Topic

- [Setup Hooks](https://www.buildthisnow.com/blog/tools/hooks/claude-code-setup-hooks): Deterministic, agentic, and interactive setup patterns.
- [Permission Hooks](https://www.buildthisnow.com/blog/tools/hooks/permission-hook-guide): Three-tier permission delegation with auto-approve, auto-deny, and LLM analysis.
- [Context Recovery](https://www.buildthisnow.com/blog/tools/hooks/context-recovery-hook): Token-based and percentage-based backup triggers using StatusLine.
- [Session Lifecycle](https://www.buildthisnow.com/blog/tools/hooks/session-lifecycle-hooks): SessionStart, Setup, PreCompact, and SessionEnd patterns.
- [Skill Activation](https://www.buildthisnow.com/blog/tools/hooks/skill-activation-hook): Auto-loading skills based on prompt keywords.
- [Stop Hooks](https://www.buildthisnow.com/blog/tools/hooks/stop-hook-task-enforcement): Task enforcement with test gates, build validation, and loop protection.
- [Self-Validating Agents](https://www.buildthisnow.com/blog/tools/hooks/self-validating-agents): Embedding validation hooks directly in agent definitions.
- [Cross-Platform Hooks](https://www.buildthisnow.com/blog/tools/hooks/cross-platform-hooks): Writing hooks that run on Windows, Linux, and macOS.

## Tools

- [Node.js](https://nodejs.org): Required by Claude Code. Use `node` in hook commands for cross-platform compatibility.
- [Just](https://github.com/casey/just): Command runner that pairs well with setup hooks for team workflows.
- [Prettier](https://prettier.io): Code formatter commonly used in PostToolUse hooks.
- [ESLint](https://eslint.org): Linter for JavaScript/TypeScript, used in auto-format and Stop hook recipes.

## Production Example

[Build This Now](https://www.buildthisnow.com) is a SaaS build system that uses Claude Code hooks extensively to coordinate 18 AI agents. The hook patterns in this cookbook are based on real production usage across skill activation, permission management, context recovery, and code validation workflows.
