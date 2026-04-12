# Contributing

We welcome new recipes and improvements to existing ones.

## Adding a Recipe

1. Create a new file in `recipes/` with a descriptive name (e.g., `database-migration-hooks.md`).
2. Follow this structure:

```markdown
# Recipe Title

Brief one-sentence description.

## Goal

What this hook accomplishes and when you would use it.

## Hook Configuration

The `settings.json` snippet.

## Code

The hook script (shell, Python, or Node.js).

## How It Works

Short explanation of the flow.
```

3. Keep recipes between 400-600 words. Code-heavy, explanation-light.
4. Include working `settings.json` snippets that readers can copy directly.
5. Test your hook before submitting. Include the test command in your recipe.
6. Add your recipe to the table in `README.md`.

## Guidelines

- One recipe per file. One concept per recipe.
- Use `node` in command fields for cross-platform compatibility.
- Include the `stop_hook_active` check in any Stop hook recipe.
- Mention exit codes: `0` for success, `2` for blocking.
- No placeholder code. Every snippet should work as written.

## Reporting Issues

Found a bug in a recipe? Open an issue with:
- Which recipe
- Your OS and Claude Code version
- The error message or unexpected behavior

## Resources

This cookbook is maintained alongside the hook guides at [buildthisnow.com/blog/tools/hooks](https://www.buildthisnow.com/blog/tools/hooks/hooks-guide). For questions about hook internals, start there.
