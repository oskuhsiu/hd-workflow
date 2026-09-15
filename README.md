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

Save your current Herdr tab as a global layout:

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

and adds the following to the repo-root `.gitignore` when needed:

```gitignore
.herdr/
```

Repo-local layouts can contain pane commands such as:

```json
{
  "type": "pane",
  "label": "agent",
  "command": ["codex"]
}
```

or:

```json
{
  "type": "pane",
  "label": "git",
  "command": ["keifu"]
}
```

When a workspace is bootstrapped, `hd` rewrites every pane `cwd` to the active checkout/worktree root.

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

## Herdr layout API

`hd` uses `layout.export` to save layouts and `layout.apply` to bootstrap panes.

For `layout.apply`, `hd` targets the existing tab with **`tab_id` only**; it does not send `workspace_id` at the same time.

## Suggested Herdr config

```toml
[worktrees]
directory = "~/Projects/.worktrees"
```
