---
name: grille-sql
description: "Use when querying, modifying, or inspecting SQL databases via Grille. WHEN: 'sql query', 'select from', 'insert into', 'run migration', 'check database', 'describe table', 'list tables', 'sql transaction'. DO NOT USE WHEN: 'docker exec psql' (prefer sql_query directly); 'read a file' (use grille-filesystem)."
---

## Overview

This skill covers Grille's SQL module — structured database access with connection-level RBAC, transaction support, and row-safety limits. Apply it whenever Claude needs to read from or write to databases, inspect schemas, or run multi-step mutations within a transaction.

Grille supports PostgreSQL and SQLite connections configured by name in `grille.toml`. Active connections: `nyxis` (SQLite), `grille-demo` (Postgres, AKV-backed password).

## Tools

- `sql_query` — Execute a SELECT statement. Returns rows up to the configured `max_rows` limit (100–500 depending on connection). Read-only path — mutation statements are rejected.
- `sql_execute` — Execute INSERT, UPDATE, or DELETE. UPDATE and DELETE trigger a **Grille Safety Check** automatically (see below). Rejected on `read_only` connections.
- `sql_begin` — Open a transaction session. Returns a `session_id` for subsequent calls.
- `sql_commit` — Commit and close an open transaction session.
- `sql_rollback` — Roll back and close an open transaction session.
- `sql_list_tables` — List all tables with row counts.
- `sql_describe` — Describe a table's schema: columns, types, nullable, defaults, primary key.

## Grille Safety Check — UPDATE and DELETE

When `sql_execute` receives an UPDATE or DELETE, Grille automatically runs a
`SELECT COUNT(*)` with the same WHERE conditions before executing. The result is
returned as a **Grille Safety Check** block. The write does not execute until the
user explicitly authorizes it.

**Flow:**

1. Call `sql_execute` with the UPDATE or DELETE statement (no `authorized` parameter)
2. Grille returns the Safety Check block showing the affected row count
3. Present the count to the user: *"This will modify/delete N rows. Do you want to allow this?"*
4. If the user authorizes: call `sql_execute` again with the identical statement and `authorized=true`
5. If the user declines: do not proceed

**Zero-row writes are blocked outright** — Grille will not execute a write that affects
no rows. This is always a sign of a wrong WHERE clause, not a valid operation.

**`authorized=true` without a prior Safety Check is rejected by Grille.** Do not pass
`authorized=true` on the first call — it will be refused.

**Example Safety Check flow:**
```
# First call — triggers Safety Check
grille:sql_execute
  connection="rekn"
  statement="UPDATE orders SET status = 'archived' WHERE created_at < '2024-01-01'"

# Grille returns:
# ━━━ Grille Safety Check ━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Operation:     UPDATE
# Table:         orders
# Connection:    rekn
# Rows affected: 4,847
#
# This operation will modify 4,847 rows and cannot be undone.
# Authorization is required before Grille will execute this statement.
#
# To authorize: call sql_execute again with authorized=true
# To cancel:    do not proceed
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# → Present to user: "This will modify 4,847 rows. Do you want to allow this?"
# → User says yes:

# Second call — authorized
grille:sql_execute
  connection="rekn"
  statement="UPDATE orders SET status = 'archived' WHERE created_at < '2024-01-01'"
  authorized=true
```

## Patterns

**Read-only inspection:**
```
grille:sql_query
  connection="nyxis"
  query="SELECT id, name, created_at FROM items ORDER BY created_at DESC LIMIT 20"
```

**Schema discovery before writing:**
```
grille:sql_list_tables connection="nyxis"
grille:sql_describe connection="nyxis" table="items"
→ Then construct queries against real column names
```

**Multi-step mutation with transaction:**
```
1. grille:sql_begin connection="rekn"
   → returns session_id = "abc-123"

2. grille:sql_execute
     connection="rekn"
     session="abc-123"
     statement="UPDATE orders SET status = 'shipped' WHERE id = 42"
   → Grille Safety Check returned (1 row)
   → User authorizes

3. grille:sql_execute
     connection="rekn"
     session="abc-123"
     statement="UPDATE orders SET status = 'shipped' WHERE id = 42"
     authorized=true

4. grille:sql_execute
     connection="rekn"
     session="abc-123"
     statement="INSERT INTO shipment_log (order_id, ts) VALUES (42, NOW())"
   → INSERT has no Safety Check — executes immediately

5. grille:sql_query
     connection="rekn"
     session="abc-123"
     query="SELECT status FROM orders WHERE id = 42"
   → Verify before committing

6a. grille:sql_commit session="abc-123"
  — OR —
6b. grille:sql_rollback session="abc-123"   ← if verification failed
```

## Constraints

- **Do not use `sql_execute` for SELECT statements.** `sql_execute` is for mutations only. Use `sql_query` for reads. The tools enforce this.

⛔ **`sql_execute` rejects SELECT. STOP — use `sql_query` for reads.**

- **Always show the Safety Check result to the user before passing `authorized=true`.** The Safety Check exists for the user's benefit — presenting the row count and asking for consent is required, not optional. Never pass `authorized=true` without user confirmation.

⛔ **Never pass `authorized=true` without showing the user the row count and receiving explicit consent.**

- **Do not run multi-step mutations without a transaction.** Without `sql_begin`/`sql_commit`, each `sql_execute` autocommits. A failure midway leaves the database partially mutated. Always wrap multi-step writes in a transaction.

⛔ **Multi-step mutations without a transaction leave the DB partially mutated on failure. STOP — open a transaction with `sql_begin` first.**

- **Always call `sql_rollback` if any step in a transaction fails.** Leaving an open transaction session consumes a database connection and may block other operations.
- **Do not attempt writes on `read_only` connections.** Connections marked `read_only = true` reject `sql_execute` at the Grille Security layer before reaching the database.
- **New SQL connections require a full Claude Desktop restart.** `grille_reload_config` does not register new connection pool entries.
- **Row results are capped at `max_rows`.** Default 100–500 per connection. Add `WHERE` and `LIMIT` clauses for large tables.

## Examples

**Example 1: Find recent errors in application logs**
```
grille:sql_describe connection="nyxis" table="app_logs"
→ Confirm column names

grille:sql_query
  connection="nyxis"
  query="SELECT ts, level, message FROM app_logs WHERE level = 'ERROR' ORDER BY ts DESC LIMIT 50"
```

**Example 2: Authorized bulk update with Safety Check**
```
# Step 1 — trigger Safety Check
grille:sql_execute
  connection="rekn"
  statement="UPDATE users SET display_name = email WHERE display_name IS NULL"

# Grille returns: "Rows affected: 142 — authorization required"
# Present to user: "This will update 142 rows. Authorize?"
# User: "Yes"

# Step 2 — authorized execution
grille:sql_execute
  connection="rekn"
  statement="UPDATE users SET display_name = email WHERE display_name IS NULL"
  authorized=true
```


## Overview

This skill covers Grille's SQL module — structured database access with connection-level RBAC, transaction support, and row-safety limits. Apply it whenever Claude needs to read from or write to databases, inspect schemas, or run multi-step mutations within a transaction.

Grille supports PostgreSQL and SQLite connections configured by name in `grille.toml`. Active connections: `nyxis` (SQLite), `grille-demo` (Postgres, AKV-backed password).

## Tools

- `sql_query` — Execute a SELECT statement. Returns rows up to the configured `max_rows` limit (100–500 depending on connection). Read-only path — mutation statements are rejected.
- `sql_execute` — Execute INSERT, UPDATE, or DELETE. Returns affected row count. Rejected on `read_only` connections.
- `sql_begin` — Open a transaction session. Returns a `session_id` for subsequent calls.
- `sql_commit` — Commit and close an open transaction session.
- `sql_rollback` — Roll back and close an open transaction session.
- `sql_list_tables` — List all tables with row counts.
- `sql_describe` — Describe a table's schema: columns, types, nullable, defaults, primary key.

## Patterns

**Read-only inspection:**
```
grille:sql_query
  connection="nyxis"
  query="SELECT id, name, created_at FROM items ORDER BY created_at DESC LIMIT 20"
```

**Schema discovery before writing:**
```
grille:sql_list_tables connection="nyxis"
grille:sql_describe connection="nyxis" table="items"
→ Then construct queries against real column names
```

**Multi-step mutation with transaction:**
```
1. grille:sql_begin connection="rekn"
   → returns session_id = "abc-123"

2. grille:sql_execute
     connection="rekn"
     session_id="abc-123"
     query="UPDATE orders SET status = 'shipped' WHERE id = 42"

3. grille:sql_execute
     connection="rekn"
     session_id="abc-123"
     query="INSERT INTO shipment_log (order_id, ts) VALUES (42, NOW())"

4. grille:sql_query
     connection="rekn"
     session_id="abc-123"
     query="SELECT status FROM orders WHERE id = 42"
   → Verify before committing

5a. grille:sql_commit session_id="abc-123"
  — OR —
5b. grille:sql_rollback session_id="abc-123"   ← if verification failed
```

## Constraints

- **Do not use `sql_execute` for SELECT statements.** `sql_execute` is for mutations only. Use `sql_query` for reads. The tools enforce this — `sql_query` rejects mutation statements and vice versa.

⛔ **`sql_execute` rejects SELECT. STOP — use `sql_query` for reads.**
- **Do not run multi-step mutations without a transaction.** Without `sql_begin`/`sql_commit`, each `sql_execute` autocommits. A failure midway leaves the database in a partially-mutated state. Always wrap multi-step writes in a transaction.

⛔ **Multi-step mutations without a transaction leave the DB partially mutated on failure. STOP — open a transaction with `sql_begin` first.**
- **Always call `sql_rollback` if any step in a transaction fails.** Leaving an open transaction session consumes a database connection and may block other operations. A try/catch mental model: on any error, roll back before reporting the failure.
- **Do not attempt writes on `read_only` connections.** Connections marked `read_only = true` in `grille.toml` reject `sql_execute` at the Grille layer before reaching the database. Check `sql_list_tables` output for the connection type.
- **Do not add a new SQL connection in `grille.toml` and expect `grille_reload_config` to register it.** New SQL connections require a full Claude Desktop restart. `grille_reload_config` hot-swaps existing config values but does not register new connection pool entries.
- **Row results are capped at `max_rows`.** Default is 100–500 per connection. Add `WHERE` and `LIMIT` clauses to stay within the cap for large tables. Results exceeding the cap are truncated — you will not receive an error, but you may receive incomplete data.

## Examples

**Example 1: Find recent errors in application logs**
```
grille:sql_describe connection="nyxis" table="app_logs"
→ Confirm column names

grille:sql_query
  connection="nyxis"
  query="SELECT ts, level, message FROM app_logs WHERE level = 'ERROR' ORDER BY ts DESC LIMIT 50"
```

**Example 2: Safely backfill a column**
```
grille:sql_begin connection="rekn"
→ session_id = "tx-001"

grille:sql_execute
  connection="rekn"
  session_id="tx-001"
  query="UPDATE users SET display_name = email WHERE display_name IS NULL"

grille:sql_query
  connection="rekn"
  session_id="tx-001"
  query="SELECT count(*) FROM users WHERE display_name IS NULL"
→ Expect 0 rows — if not 0, rollback

grille:sql_commit session_id="tx-001"
```
