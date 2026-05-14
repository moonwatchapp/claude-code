---
name: moonwatch
description: "Use for ALL Moonwatch operations: investigating runtime bugs, analyzing logs, placing diagnostic logging, reviewing watchers, setting up monitoring dashboards, or setting up Moonwatch. Triggers automatically when the user mentions errors, log analysis, watcher investigations, monitoring, metrics, dashboards, or asks about issues in their running application. Load this skill BEFORE using the Moonwatch `mw` CLI."
user_invocable: true
---

# Moonwatch

Use Moonwatch to get runtime evidence instead of guessing from static code. Moonwatch operations use the `mw` CLI from `@moonwatch/cli`.

## CLI Setup

Before the first Moonwatch command in a session:

1. Run `command -v mw`.
2. If missing, ask once whether to install globally with `npm install -g @moonwatch/cli`.
3. After installing, verify with `mw --version` or `mw whoami`.
4. If install fails or the user declines, use `npx -y @moonwatch/cli@latest` as the command prefix for every `mw` command.

Do not block Moonwatch usage just because global install is unavailable.

## Required Context

For the complete Moonwatch workflow, read `references/moonwatch-guide.md`, then follow the section that matches the user's task.

Resolve the log file before querying logs or managing watchers:

- Prefer `MOONWATCH_LOG_FILE_ID` for production.
- Use `MOONWATCH_LOG_FILE_ID_DEV` only when the user explicitly mentions local/dev.
- If no log file is configured, follow `references/moonwatch-guide.md#setup-wizard`.

## Investigation Workflow

1. Create or reuse a watcher for runtime bugs.
2. Place watcher-scoped `logger.debug()` logs around relevant code paths.
3. Ask the user to reproduce the issue when reproduction is needed.
4. Query watcher logs with `mw logs query <logFileId> "SELECT ... WHERE watcher_id = '<uuid>' ..."`.
5. Record findings with `mw watchers update`.
6. Fix the bug.
7. Remove temporary diagnostic logs before marking the watcher resolved.
