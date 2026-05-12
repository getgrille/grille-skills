---
name: grille-git
description: "Use when performing git operations on a repository via Grille. WHEN: 'git status', 'what changed', 'commit history', 'git log', 'show diff', 'what files changed', 'commit these changes', 'switch branch', 'create branch', 'stash changes', 'fetch from remote', 'git_status', 'git_log', 'git_diff', 'git_branches', 'git_show', 'git_stash_list', 'git_commit', 'git_checkout', 'git_stash', 'git_stash_pop', 'git_fetch'. DO NOT USE WHEN: 'push to remote' (git push not supported — SSH agent OS limitation; use process_run with git.exe manually); 'clone a repository' (out of scope); 'merge or rebase' (interactive, not supported)."
---

## Overview

This skill covers Grille's Git module — 11 tools for structured git operations via libgit2 (no subprocess). Compared to `process_run` + `git.exe`: 1.8× more token-efficient on average, up to 20-25× on diffs. No subprocess spawn overhead. Structured data Claude can reason over rather than text to parse.

All `path` parameters must be within `fs_allowed_paths`. Write tools require `git_write_enabled = true` in role config. `git push` is not implemented — SSH agent is inaccessible from the Grille subprocess context on Windows (OS limitation).

## Tools

**Read-only:**
- `git_status` — Working tree and index state per file. Staged vs unstaged. ~3× more token-efficient than raw `git status`.
- `git_log` — Commit history: hash, author, date, message. Filter by branch, author, file path, `since`. Max 100 commits.
- `git_diff` — Diff summary (`summary_only=true`, default) or full patch (`summary_only=false`). Modes: working tree vs index, staged vs HEAD, two commits. Filter with `file_path`. Always start with summary.
- `git_branches` — Branch list with last commit info and ahead/behind counts vs upstream. `include_remote=true` for remote-tracking branches.
- `git_show` — Single commit detail: author, date, message, file-level diff stats (or full patch). Defaults to HEAD.
- `git_stash_list` — All stash entries with index, message, and timestamp.

**Write (require `git_write_enabled = true`):**
- `git_commit` — Stage files and commit. `files=["."]` stages all. Returns new commit hash.
- `git_checkout` — Switch branch or create new (`create_branch=true`).
- `git_stash` — Stash working tree changes. `include_untracked=true` for new files.
- `git_stash_pop` — Apply and remove a stash entry. Default `index=0`.
- `git_fetch` — Fetch from remote without merging. Returns structured ref diff: updated branches (with commit count), new refs, pruned refs. Parameters: `remote` (default `"origin"`), `prune` (default false).

## Patterns

**Check state before starting work:**
```
grille:git_status path="C:\\dev\\myproject"
grille:git_branches path="C:\\dev\\myproject"
```

**Review recent history:**
```
grille:git_log path="C:\\dev\\myproject" max_commits=10
grille:git_log path="C:\\dev\\myproject" file_path="src/tools/net.rs"
```

**Inspect a commit:**
```
grille:git_show path="C:\\dev\\myproject" commit="c5ee1cd"
grille:git_show path="C:\\dev\\myproject" commit="HEAD" summary_only=false
```

**Review changes (always start with summary):**
```
grille:git_diff path="C:\\dev\\myproject"
grille:git_diff path="C:\\dev\\myproject" staged=true
grille:git_diff path="C:\\dev\\myproject" file_path="src/main.rs" summary_only=false
grille:git_diff path="C:\\dev\\myproject" commit_a="v0.4.1" commit_b="HEAD"
```

**Commit:**
```
grille:git_status path="C:\\dev\\myproject"
grille:git_commit path="C:\\dev\\myproject" message="feat(net): add net_connections" files=["."]
grille:git_commit path="C:\\dev\\myproject" message="fix: port parsing" files=["src/tools/net.rs"]
```

**Branch:**
```
grille:git_checkout path="C:\\dev\\myproject" target="main"
grille:git_checkout path="C:\\dev\\myproject" target="feature/networking" create_branch=true
```

**Fetch:**
```
grille:git_fetch path="C:\\dev\\myproject"
grille:git_fetch path="C:\\dev\\myproject" prune=true
grille:git_fetch path="C:\\dev\\myproject" remote="upstream"
```

**Stash:**
```
grille:git_stash path="C:\\dev\\myproject" message="WIP"
grille:git_stash_list path="C:\\dev\\myproject"
grille:git_stash_pop path="C:\\dev\\myproject"
```

## Token efficiency vs process_run

| Operation | process_run | git module |
|-----------|------------|------------|
| `git status` (10 files) | ~180 tokens | ~60 tokens |
| `git log -10` | ~250 tokens | ~80 tokens |
| `git diff` summary | ~600 tokens | ~150 tokens |
| `git show` summary | ~500 tokens | ~40 tokens |

**Key rule:** `summary_only=true` is the default on `git_diff` and `git_show`. Request `summary_only=false` only when you need to read actual code. A full diff on an active project can be 2000+ tokens.

## ⛔ Common mistakes

- **`summary_only=false` by default** — always start with summary; drill into specific files only when needed.
- **Trying to push** — not supported. Use `process_run` with `git.exe` (if allowed) or push manually in a terminal.
- **Forgetting `git_write_enabled = true`** — write tools return a clear config error if missing.
- **Using `process_run` + `git.exe` for log/diff/status** — the git module is 3-12× more token-efficient. Prefer it.
