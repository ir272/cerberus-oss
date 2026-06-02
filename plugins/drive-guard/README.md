# drive-guard

A Claude Code plugin that blocks access to Google **Shared drives** while leaving
everything else (My Drive, local code, the web) fully usable. It is the plugin
packaging of [Cerberus](../../README.md).

## What it locks

- **The filesystem** — Shared drives mount as a normal folder. A `PreToolUse` hook
  inspects every file action and terminal command and denies anything aimed at a
  Shared-drive path.
- **The Google Drive MCP connector** — the same hook denies every
  `mcp__claude_ai_Google_Drive__*` tool call.

## What's inside

```
drive-guard/
├── .claude-plugin/plugin.json   # manifest
├── hooks/hooks.json             # PreToolUse hook wiring (matcher + command)
└── scripts/drive-guard.py       # the guard (single-file, stdlib-only Python 3)
```

`scripts/drive-guard.py` is a synced copy of the canonical
`deliverables/claude-code/drive-guard.py`. Run `python3 tools/sync-plugin.py` from
the repo root after changing the canonical script; `--check` fails on drift.

## Install (single user)

```
/plugin marketplace add ir272/cerberus
/plugin install drive-guard@cerberus
```

## Deploy org-wide (enforced)

A plugin is the *packaging*; enforcement still comes from **managed settings**.
Push `deliverables/claude-code/packaging/plugin/managed-settings.plugin.json` via
the Claude.ai admin console (or MDM). It registers the marketplace
(`extraKnownMarketplaces`) and force-enables the plugin (`enabledPlugins`) so users
cannot disable it.

## Scope of this layer

This plugin delivers the **hook layer only** (filesystem + MCP connector). The
`permissions.deny` rules and the macOS Seatbelt `sandbox` backstop cannot live in a
plugin — they must be added to managed settings separately. See the repo
[`CLAUDE.md`](../../CLAUDE.md) "Deployment (org-wide)" section.

## Configuration

The guard reads the same environment variables as the standalone script:
`DRIVE_GUARD_MODE`, `DRIVE_GUARD_PROTECTED`, `DRIVE_GUARD_AUDIT`, `DRIVE_GUARD_SHELL`.
