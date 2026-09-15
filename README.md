# hd-workflow

`hd` is a small workflow helper for **Herdr + Git worktree** development.

It treats a development context as:

```text
Git checkout / worktree
+ Herdr workspace
+ reusable Herdr layout
```

Current version: **0.4.0**

## Install

Requirements:

- `git`
- `herdr`
- `jq`
- `python3`

```bash
mkdir -p ~/.local/bin
cp hd ~/.local/bin/hd
chmod +x ~/.local/bin/hd
```

Make sure `~/.local/bin` is in `PATH`.

## Commands

```bash
hd [--layout NAME]
hd new <branch> [--base REF] [--layout NAME]
hd open <branch> [--layout NAME]
hd list
hd rm [branch|path]
hd layout save [name]
hd layout init [name]
hd doctor
```

### Mental model

```text
hd
→ ensure/focus the current checkout in Herdr

hd new <branch>
→ create a Git worktree + Herdr workspace

hd open <branch>
→ open an existing Git worktree in Herdr
```

`hd rm` removes a linked worktree but **never deletes the Git branch**.

## Layouts

Layout lookup order:

```text
1. <primary-repo>/.herdr/layouts/<name>.json
2. ~/.config/herdr/layouts/<name>.json
```

Save the current Herdr tab as a global layout:

```bash
hd layout save dev
```

Create a repo-local copy from the global layout:

```bash
hd layout init dev
```

This creates:

```text
<primary-repo>/.herdr/layouts/dev.json
```

and adds this rule to the repo-root `.gitignore` when needed:

```gitignore
.herdr/
```

The repo-local layout is intentionally machine-local and is not meant to be committed.

### Layout JSON format

A layout is a recursive tree with two node types:

```text
pane
split
```

The root can be either one pane or a split containing more panes/splits.

#### Pane node

Common fields:

| Field | Required | Meaning |
| --- | --- | --- |
| `type` | yes | Must be `"pane"`. |
| `label` | no | Human-readable pane label. |
| `cwd` | no | Working directory used when Herdr creates the pane. `hd` rewrites pane `cwd` to the active repo/worktree root when applying a layout. |
| `command` | no | Command to launch, represented as an **argv array**. |
| `env` | no | Environment variables for the launched process as a JSON object. |
| `pane_id` | no | Runtime Herdr pane id. It may appear in exported data, but it is session-specific and should not be hand-authored. `hd layout save` removes it. |

Minimal pane:

```json
{
  "type": "pane",
  "label": "shell"
}
```

Run Codex automatically:

```json
{
  "type": "pane",
  "label": "agent",
  "command": ["codex"]
}
```

Run keifu automatically:

```json
{
  "type": "pane",
  "label": "git",
  "command": ["keifu"]
}
```

`command` is an argv array, not a shell command string. Arguments are separate elements:

```json
{
  "type": "pane",
  "label": "tests",
  "command": ["gradle", "test"]
}
```

If shell syntax is intentionally required, invoke a shell explicitly:

```json
{
  "type": "pane",
  "label": "tests",
  "command": ["sh", "-c", "just test && echo done"]
}
```

Environment variables:

```json
{
  "type": "pane",
  "label": "agent",
  "command": ["codex"],
  "env": {
    "HD_ROLE": "agent"
  }
}
```

#### Split node

Common fields:

| Field | Required | Meaning |
| --- | --- | --- |
| `type` | yes | Must be `"split"`. |
| `direction` | yes | `"right"` or `"down"`. |
| `ratio` | yes | Split ratio for the first child, for example `0.65`. |
| `first` | yes | First child node (`pane` or another `split`). |
| `second` | yes | Second child node (`pane` or another `split`). |

Example:

```json
{
  "type": "split",
  "direction": "right",
  "ratio": 0.65,
  "first": {
    "type": "pane",
    "label": "agent",
    "command": ["codex"]
  },
  "second": {
    "type": "split",
    "direction": "down",
    "ratio": 0.7,
    "first": {
      "type": "pane",
      "label": "git",
      "command": ["keifu"]
    },
    "second": {
      "type": "pane",
      "label": "shell"
    }
  }
}
```

Visually this represents roughly:

```text
┌────────────────────────────┬───────────────────┐
│                            │                   │
│          codex             │      keifu        │
│                            │                   │
│                            ├───────────────────┤
│                            │      shell        │
└────────────────────────────┴───────────────────┘
```

When `hd` applies the layout, every pane's `cwd` is rewritten to the active checkout/worktree root. The layout therefore describes **structure and startup behavior**, not a hard-coded project path.

Herdr's declarative layout restore can recreate structure, labels, cwd, env, and optional argv commands. It does not restore live PTYs, scrollback, or already-running processes.

## Worktree behavior

```bash
hd new feature/login --base main
```

- branch missing → create branch + worktree
- branch exists but has no worktree → create worktree from existing branch
- branch already has a worktree → refuse and suggest `hd open`

```bash
hd open feature/login
```

- existing worktree → open/focus it in Herdr
- no worktree → refuse and suggest `hd new`

## Closing a Herdr workspace

CLI:

```bash
herdr workspace close <workspace_id>
```

The default TUI key is:

```text
prefix + Shift+D
```

This closes **Herdr state only**. It does not remove the Git worktree checkout.

If a primary workspace still has linked-worktree workspaces open, Herdr requires explicit group intent:

```bash
herdr workspace close <workspace_id> --group
```

To actually delete a linked Git checkout, use `hd rm` or Herdr's `worktree remove` command instead.

## Herdr layout API

`hd` uses `layout.export` to save layouts and `layout.apply` to bootstrap panes.

For `layout.apply`, `hd` targets the existing tab with **`tab_id` only**; it does not send `workspace_id` at the same time.

## Suggested Herdr config

```toml
[worktrees]
directory = "~/Projects/.worktrees"
```
