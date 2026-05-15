---
name: moonwatch
description: "Use for ALL Moonwatch operations: investigating runtime bugs, analyzing logs, placing diagnostic logging, reviewing watchers, setting up monitoring dashboards, or setting up Moonwatch. Triggers automatically when the user mentions errors, log analysis, watcher investigations, monitoring, metrics, dashboards, or asks about issues in their running application. Load this skill BEFORE using the Moonwatch CLI through npx."
user_invocable: true
---

# Moonwatch

Use Moonwatch to get runtime evidence instead of guessing from static code. Moonwatch operations must use the latest `@moonwatch/cli` through `npx -y @moonwatch/cli@latest`; do not ask to install a global CLI binary.

## CLI Usage

Use this command prefix for every Moonwatch CLI command:

```bash
npx -y @moonwatch/cli@latest
```

For example, run `npx -y @moonwatch/cli@latest whoami` to verify auth.

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
4. Query watcher logs with `npx -y @moonwatch/cli@latest logs query <logFileId> "SELECT ... WHERE watcher_id = '<uuid>' ..."`.
5. Record findings with `npx -y @moonwatch/cli@latest watchers update`.
6. Fix the bug.
7. Remove temporary diagnostic logs before marking the watcher resolved.
