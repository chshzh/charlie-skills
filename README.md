# .claude

Personal AI configuration, rules, and skills for Claude CLI and Cursor.

**Default path:** `~/.claude/`
Override with the `CLAUDE_CONFIG_DIR` environment variable.

## Contents

| Path | Purpose | Loaded |
|---|---|---|
| `CLAUDE.md` | Global behavioral guidelines | Every session, every project |
| `skills/` | Reusable agent skills | On invocation (`/skill-name`) or auto-invoked by Claude |
| `rules/` | Topic-scoped instructions (optional) | At session start, or when matching file paths enter context |
| `settings.json` | Global permissions and hooks (optional) | At session start; overridden by project-level settings |
| `agents/` | Subagent definitions (optional) | When delegated to or @-mentioned |
| `output-styles/` | Custom output styles (optional) | When selected via `outputStyle` setting |

> Runtime data (`projects/`, `debug/`, `session-env/`, `plans/`, `statsig/`, `todos/`) is written by Claude automatically and is excluded from git via `.gitignore`.

## Skills

Skills live under `skills/` as named directories, each with a `SKILL.md` entry point.
Invoke a skill with `/skill-name` or by asking Claude to use it by name.

See individual `SKILL.md` files for usage details and supported arguments.
