# Cerberus — drive-guard

A Claude Code plugin that blocks Claude from **reading, changing, or deleting Google
Shared drives**, while leaving everything else — your **My Drive**, your **local code**,
and the **web** — fully usable.

Shared drives are how teams expose access to documents they should not hand to an agent
wholesale. `drive-guard` is the guardrail that keeps an agent out of them without crippling
the rest of the workflow.

## What it does

A coding agent can reach a Shared drive through **two doors**, and `drive-guard` shuts both
with a single `PreToolUse` hook:

- **The filesystem.** On macOS a Shared drive mounts as an ordinary folder
  (`~/Library/CloudStorage/GoogleDrive-*/Shared drives/...`). The hook inspects every file
  tool — `Read`, `Edit`, `Write`, `MultiEdit`, `NotebookEdit`, `NotebookRead`, `Glob`,
  `Grep`, `LS` — and every `Bash` command, and denies anything that targets a protected
  path. For `Bash` it parses redirections, `tee`/`dd`, in-place editors (`sed -i`, `perl -i`),
  `cp`/`mv`/`rsync`, `git` mutations, subshells (`$(...)`, backticks), and `eval`/`-c`
  wrappers, not just the bare command name.
- **The Google Drive MCP connector.** The same hook denies every
  `mcp__claude_ai_Google_Drive__*` tool call outright (search, read, create, copy, download,
  metadata, permissions, etc.).

The guard fails **closed**: if it hits an internal error while deciding on a real target, it
denies the call rather than guessing. (The one exception is an unparseable hook event, which
can't identify any target — that is allowed with a warning on stderr so a malformed event
can't brick every tool call.)

## Install (single user)

```
/plugin marketplace add ir272/cerberus-oss
/plugin install drive-guard@cerberus
```

`drive-guard.py` is single-file, standard-library-only Python 3 — no `pip install`. You need
`python3` (or `python`) on `PATH`; the launcher (`run-guard.sh`) probes for one at runtime.

## Enforce org-wide

Installing the plugin is the *packaging*; making it **non-optional** comes from Claude Code
**managed settings**, which you push through the Claude.ai admin console or your MDM. Two keys
do the work:

- **`extraKnownMarketplaces`** — registers `ir272/cerberus-oss` as a trusted marketplace so
  every seat can resolve the plugin without each user adding it by hand.
- **`enabledPlugins`** — force-enables `drive-guard@cerberus` so users cannot turn it off.

Managed settings outrank user and project settings, so once these are pushed the hook runs on
every seat and can't be disabled locally. (The managed-settings file is environment-specific
and is not shipped in this repo — author it for your org.)

## Configuration

The guard reads four environment variables (all optional):

| Variable | Default | Effect |
| --- | --- | --- |
| `DRIVE_GUARD_MODE` | `block` | `block` denies all access (read and write) to protected paths. `readonly` allows reads but denies writes/deletes/renames. The MCP connector is denied in **both** modes. **Note:** the bundled `hooks.json` invokes the guard with an explicit `--mode block`, and the CLI flag wins over the env var — so setting `DRIVE_GUARD_MODE=readonly` has **no effect** unless you edit the hook's args to drop `--mode block` (or change it to `--mode readonly`). |
| `DRIVE_GUARD_PROTECTED` | see below | `PATH`-separated list of protected paths/globs. Overrides the built-in default entirely. Supports `*` and `**`. |
| `DRIVE_GUARD_AUDIT` | _(off)_ | If set to a file path, appends one JSON line per decision (`allow`/`deny`) with timestamp, tool, target, and reason. |
| `DRIVE_GUARD_SHELL` | `auto` | Windows-only hint for how to tokenize `Bash` commands: `posix`/`bash`/`wsl` vs. `windows`/`powershell`/`cmd`. Ignored on macOS/Linux. |

### Default protected path

If `DRIVE_GUARD_PROTECTED` is unset, the guard protects the platform default:

- **macOS:** `~/Library/CloudStorage/GoogleDrive-*/Shared drives` (the glob covers any
  signed-in account).
- **Windows:** `G:\Shared drives` and the Git-Bash/MSYS form `/g/Shared drives`.

**Confirm your team's actual Drive mount path.** Drive for Desktop can mount under a different
account folder or a different drive letter; if yours differs, set `DRIVE_GUARD_PROTECTED`
explicitly (it replaces the default — it does not add to it).

## Limits

Be honest about what a `PreToolUse` hook can and cannot do:

- **Static wildcards and `find` are handled.** The `Bash` parser does more than match the
  literal path text. It expands shell globs against the real filesystem (`glob.glob()`), so a
  wildcard token that resolves into the protected tree is denied. It also inspects `find`'s own
  argv — `-path`/`-ipath`/`-wholename` patterns that name a protected path, and search roots
  at or above a protected directory combined with any filter — and denies those too. So
  shell-glob and `find`-based access are **largely mitigated**, not a wide-open hole.
- **A simple variable still gets caught.** Because the guard also scans the raw command text
  for the protected path, an assignment like `D="<protected path>"; cat "$D/x"` is **denied** —
  the literal path string appears in the command. Naive variable assembly is *not* a bypass.
- **The true residual is fully dynamic obfuscation.** What the hook genuinely cannot see is a
  protected path that never appears literally and is not a static wildcard — e.g. a path
  *decoded from base64 at runtime*, or *assembled from the contents of a file*. There the path
  string simply does not exist in the command text or as a glob the shell can expand ahead of
  time, so text/glob inspection has nothing to match. This is a fundamental limit of inspecting
  command strings, not a bug to be patched away — the backstop for it is **Google-side
  Viewer-only** (below), which is unbypassable regardless of obfuscation.
- **The hook is the convenience layer, not the security boundary.** It runs inside Claude
  Code; anything that doesn't go through Claude Code's tools isn't seen by it.

### Operator warning: the hook is best-effort and fails open

The launcher (`run-guard.sh`) probes for a Python interpreter at runtime. If **no `python3` or
`python` is on `PATH`**, it warns on stderr and **exits 0 — allowing the call** (fail-open on
the launcher, so a missing interpreter never bricks every tool call). The guard's own
fail-closed posture only applies *once the script actually runs*. Treat the plugin as
best-effort.

For guaranteed, **fail-closed** enforcement that needs no Python, pair the plugin with two
controls that don't depend on the hook running:

1. **Claude Code managed-settings `permissions.deny`** on the Shared-drive globs. These are
   enforced by Claude Code itself (no interpreter required) and outrank user/project settings.
   Sketch:

   ```json
   {
     "permissions": {
       "deny": [
         "Read(~/Library/CloudStorage/GoogleDrive-*/Shared drives/**)",
         "Write(~/Library/CloudStorage/GoogleDrive-*/Shared drives/**)",
         "Edit(~/Library/CloudStorage/GoogleDrive-*/Shared drives/**)"
       ]
     }
   }
   ```

2. **Google-side Viewer-only** on the Shared drive — the unbypassable backstop for the dynamic
   residual above (see next section).

### Backstop: Google-side Viewer-only

The unbypassable control is on **Google's side**: set the in-scope accounts to **Viewer** on
the Shared drive. Viewer permission is enforced by Google for every client and platform — the
agent (or anyone using the account) simply cannot write, regardless of how a command is
obfuscated or which tool is used. Use `drive-guard` for fast, friendly, in-session blocking;
use Viewer-only on the Shared drive for true integrity. (This repo does **not** rely on any
macOS OS-level sandbox as a backstop — that is out of scope here.)

## What's inside

```
.
├── README.md                         # this file
├── LICENSE                           # repo license
├── .claude-plugin/marketplace.json   # marketplace "cerberus" -> drive-guard
└── plugins/drive-guard/
    ├── .claude-plugin/plugin.json    # plugin manifest
    ├── hooks/hooks.json              # single PreToolUse hook entry
    ├── scripts/run-guard.sh          # launcher: picks python3/python at runtime
    ├── scripts/drive-guard.py        # the guard (stdlib-only Python 3)
    └── README.md                     # plugin-internal notes
```

## License

See [LICENSE](LICENSE).
