# Grille — CRLF and Special Character Patterns Reference

## CRLF Line Endings

Windows text files use CRLF (`\r\n`). `fs_str_replace` fails when `old_str` spans multiple lines in a CRLF file — the CR character causes match failure even when the content looks correct.

### The Pattern: Single-Line Anchors

Always use a unique single-line string as the anchor for `fs_str_replace`:

```
✅ old_str="## Session Wrap-Up"               ← unique single-line section header
✅ old_str="timeout_seconds: 30,"              ← unique single-line expression
✅ old_str="<!-- END -->"                      ← unique end-of-file marker
❌ old_str="fn main() {\n    let x = 1;"      ← spans lines, CRLF breaks it
❌ old_str="## Overview\n\nThis skill"         ← spans lines, CRLF breaks it
```

### When You Must Span Lines

If the target cannot be captured in a single-line anchor:
1. Use `fs_diff` to preview the proposed full-file content
2. Use `fs_write_file` with `overwrite: true` for a full-file rewrite
3. Verify with `fs_stat` — confirm size changed

### Append-Only Documents

For journals and accumulated docs (`02_DevJournal.md`, `SessionSummaries.md`, `03_LessonsLearned.md`, `SecurityDecisions.md`):
- Always use `fs_str_replace` targeting a unique anchor at the very end of the file
- **Never** use `fs_write_file` with `overwrite: true` — this destroys all prior history
- Read the last 500 bytes of the file first to find the unique terminal anchor

## Dollar Signs in Values

Both PowerShell and Docker Compose perform variable interpolation on `$`:

```
❌ Password: P@ss$word123    ← "$word" expanded to empty string
✅ Password: PAssword123Dev  ← safe in all contexts
```

Affected contexts:
- Docker Compose `.env` files and inline environment values
- `ps_run` script strings (PowerShell double-quoted strings)
- Any value interpolated through a shell

Use alphanumeric-only passwords in all demo, dev, and Compose-managed environments.

## Percent Signs in URLs

Passwords injected into PostgreSQL URLs must be percent-encoded. Grille handles this automatically for `secret_ref`-based SQL connections. If constructing a URL manually, `%` must be `%25`, `@` must be `%40`, etc.
