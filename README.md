# Claude Code Base Rules

This repository contains base rules, conventions, and standards for Claude Code configuration. It is an **abstract project** — it has no runnable code. Its purpose is to be incorporated into real projects, providing a consistent foundation for Claude's behavior across all environments.

Inspired by [aws-qdeveloper-base-rules](https://github.com/PauloFerreira25/aws-qdeveloper-base-rules).

## Structure

Rules are organized in the `.ia/` directory:

```
.ia/
  index.md          ← start here: lists all rules and when to apply them
  01-29             ← base rules, applicable to all projects
  30+               ← reserved for project-specific rules (added per project)
  999-update.md     ← instructions for updating rules from this repository
```

## How to use

There are three ways to incorporate these rules into a project:

1. **ZIP Download** — download the repository as ZIP and extract the `.ia/` directory into your project root
2. **Manual selection** — copy only the rule files relevant to your project from the raw GitHub URLs
3. **Automated update** — use `999-update.md` to let Claude pull and sync the latest rules automatically

## Integration with Claude Code

In your project's `.claude/CLAUDE.md`, add a reference to the index:

```markdown
# Project Memory

The `.ia/` directory is the base for all memory, rules, standards, and models for this project.

Before acting on any task, consult `.ia/index.md` to check if there are relevant rules or conventions to follow.
```

Claude will consult the index at the start of relevant tasks and read only the rule files that apply — keeping token usage minimal.

## Adding project-specific rules

Rules numbered 30+ are reserved for project-specific conventions. Add them directly in your project's `.ia/` directory and register them in `index.md`.
