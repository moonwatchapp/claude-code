---
name: moonwatch-setup
description: "Interactive setup wizard for connecting Claude Code to Moonwatch. Configures your personal API key, selects log files, installs or falls back for the `mw` CLI, and verifies SDK setup."
user_invocable: true
---

# Setup Wizard

Read `../moonwatch/references/moonwatch-guide.md`, then follow the `## Setup Wizard` section.

Before using Moonwatch commands, check `command -v mw`. If it is missing, ask once to install globally with `npm install -g @moonwatch/cli`; if that fails or is declined, use `npx -y @moonwatch/cli@latest` as the command prefix.
