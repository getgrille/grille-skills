---
name: grille-filesystem
description: "Use when reading, writing, editing, searching, or managing files on Windows via Grille. WHEN: 'read file', 'write file', 'edit file', 'search codebase', 'find file', 'search in file', 'grep file', 'list directory', 'copy file', 'delete file', 'diff files', 'append to doc'. DO NOT USE WHEN: running executables (use grille-process), querying databases (use grille-sql), reading registry (use grille-registry)."
---

## Overview

This skill covers all Grille filesystem operations. Apply it whenever Claude needs to read project files, write or modify configuration, perform surgical edits, search codebases, compare file versions, or manage the directory tree. The filesystem module is the most-used module in Grille — nearly every development task touches it.

All filesystem operations are RBAC-gated. Every path is canonicalized and validated against the allowlist in `grille.toml` before any OS call.

## Tools

- `fs_read_file` — Read file contents. Supports UTF-8 and UTF-16 LE/BE. Paginate large files with `offset_bytes` and `max_bytes`. Response header includes `CRLF` in the encoding field when the file uses Windows line endings (`\r\n`) — this tells Claude upfront so it can use `fs_str_replace` with confidence.
- `fs_write_file` — Write a file atomically (temp-then-rename). Default refuses to overwrite; set `overwrite: true` to replace.
- `fs_str_replace` — Surgically replace a unique string in a file. Enforces uniqueness — fails if the string appears zero or more than once. The hero tool for targeted edits. Two modes: (1) string-match: `old_str` + `new_str`; (2) line-range: `start_line` + `end_line` + `new_str` — bypasses string matching entirely, use for large files or when `old_str` would be too long. `old_str` is optional in line-range mode. `dry_run: true` returns a unified diff without writing — use to verify complex replacements before committing. CRLF line endings are handled transparently in both modes.
- `fs_append_file` — Append content to the end of a file. Creates the file if it does not exist.
- `fs_list_directory` — List directory contents recursively. Returns size, type, hidden attribute.
- `fs_find` — Search files by name pattern, content substring, extension, or modification date. Has a 20-second wall-clock timeout for deep walks.
- `fs_copy_file` — Copy a file atomically within allowed paths.
- `fs_stat` — Get metadata: size, timestamps, file type, Windows attributes.
- `fs_diff` — Compare two files, or a file against proposed content, before writing.
- `fs_create_directory` — Create a directory and any missing parents.
- `fs_delete_file` — Soft-delete a file to `%TEMP%\grille-trash\`. Recoverable. Never permanent.
- `fs_delete_directory` — Soft-delete a directory tree to trash. Denied on drive roots and workspace root paths.
- `fs_search` — Search within a single file for a literal substring or regex pattern. Returns matching lines with line numbers and configurable context. Much faster and more token-efficient than paginating `fs_read_file` for large files — returns only the relevant lines. Supports `start_line`/`end_line` scope limiting. **Regex alternation: use `|` not `\|`** — `\|` matches a literal pipe character, not alternation. Example: `pattern="foo|bar"` with `regex: true`.

## Patterns

**Surgical edit — preferred pattern for modifying existing files:**
```
1. fs_read_file to confirm current content and locate the target string
2. fs_str_replace with old_str = unique surrounding context + the target, new_str = replacement
3. fs_stat to verify the file was written (non-zero size, updated mtime)
```

**Dry-run before a complex edit:**
```
grille:fs_str_replace
  path="C:\\dev\\grille\\src\\tools\\ssh.rs"
  old_str="fn build_remote_command("
  new_str="fn build_remote_command_v2("
  dry_run=true
→ Returns unified diff without writing — inspect before committing
```

**Line-range replacement (large files or when old_str would be too long):**
```
1. fs_read_file to identify the line numbers to replace
2. fs_str_replace with start_line=N, end_line=M, new_str="replacement content"
   (old_str is not required in this mode)
3. fs_stat to verify mtime updated
```

**Safe full-file rewrite — only when surgical edit is not feasible:**
```
1. fs_diff to preview the proposed content against the current file
2. fs_write_file with overwrite: true
3. fs_stat to verify
```

**Append to append-only documents (journal, lessons, status):**
```
1. fs_str_replace targeting a unique anchor at the very end of the file
   (e.g. the last line or section header) — NOT fs_append_file with overwrite
   This preserves all prior content and avoids silent data loss.
```

**Search before assuming a path:**
```
fs_find to locate files by name before constructing paths from memory.
fs_list_directory to enumerate a directory before reading individual files.
fs_search to find a specific string, function, or pattern in a known file
  without reading the whole file — especially useful for large source files.
```

**Always verify writes:**
After every fs_write_file, fs_str_replace, or fs_append_file call, confirm with
fs_stat before claiming success. Check that size is non-zero and mtime reflects the
current time.

## Constraints

- **Do not use `fs_write_file` with `overwrite: true` on append-only documents** (`02_DevJournal.md`, `SessionSummaries.md`, `03_LessonsLearned.md`, `SecurityDecisions.md`). An accidental overwrite permanently destroys all prior content — this has happened and caused significant data loss.

⛔ **`fs_write_file overwrite: true` on append-only docs DESTROYS ALL HISTORY. STOP — use `fs_str_replace` with a unique end-of-file anchor.**
- **CRLF files are handled transparently** by `fs_str_replace` — both string-match and line-range modes normalize line endings internally. The `fs_read_file` header will show `Encoding: utf-8 CRLF` when a file uses Windows line endings. You do not need to do anything special — just use `fs_str_replace` normally. The one remaining hazard: multi-line `old_str` spanning a CRLF boundary can still fail if Claude's `old_str` contains a bare `\n` where the file has `\r\n` — use a **single-line anchor** as `old_str` to avoid this entirely.
- **Do not claim a write succeeded without running `fs_stat`.** The tool call returning success is not sufficient — verify the file exists and has non-zero size on disk.
- **Do not write to Windows paths using `create_file` or `str_replace` (the generic Anthropic sandbox tools).** Those tools write to a separate sandbox filesystem that mimics Windows paths but is not real disk. Always use `grille:fs_write_file` and `grille:fs_str_replace` for writes to `C:\`.

⛔ **Writing to `C:\` via `create_file` or `str_replace` is SILENT DATA LOSS — those are sandbox tools. STOP — use `grille:fs_write_file` or `grille:fs_str_replace`, then verify with `grille:fs_stat`.**
- **`fs_delete_file` and `fs_delete_directory` are soft-delete only** — files go to `%TEMP%\grille-trash\`. Files can be manually recovered. There is no permanent delete in Grille v1.
- **`fs_find` has a 20-second timeout** on deep walks. For very large trees, scope the search path narrowly or use a name/extension filter to reduce scan depth.

## Examples

**Example 1: Edit a specific function in a source file**
```
Goal: Change the timeout value in src/tools/fs.rs from 30 to 60

1. grille:fs_read_file path="C:\dev\grille\src\tools\fs.rs"
   → Locate the line: timeout_seconds: 30,
   → Confirm it appears exactly once

2. grille:fs_str_replace
     path="C:\dev\grille\src\tools\fs.rs"
     old_str="timeout_seconds: 30,"
     new_str="timeout_seconds: 60,"

3. grille:fs_stat path="C:\dev\grille\src\tools\fs.rs"
   → Confirm mtime updated
```

**Example 2: Append a new entry to the dev journal**
```
Goal: Add a session summary entry to 02_DevJournal.md

1. grille:fs_read_file path="C:\dev\grille-docs\02_DevJournal.md"
   → Read the last 30 lines to find the unique end anchor

2. grille:fs_str_replace
     path="C:\dev\grille-docs\02_DevJournal.md"
     old_str="<!-- END -->"   ← unique anchor at end of file
     new_str="## Session 27 — 2026-05-05\n\n[new content here]\n\n<!-- END -->"

3. grille:fs_stat path="C:\dev\grille-docs\02_DevJournal.md"
   → Confirm size increased and mtime updated
```
