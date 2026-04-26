---
name: insert-returning-requires-select
description: PostgreSQL requires SELECT privilege on a table when using INSERT...RETURNING — this is distinct from INSERT privilege and catches SECURITY DEFINER function authors by surprise
type: trap
confidence: experimental
captured_during: heart-engine-T6
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# INSERT ... RETURNING Requires SELECT

## The trap

PostgreSQL requires `SELECT` privilege on a table when using `INSERT ... RETURNING column_list`.
To return the row (or specific columns), Postgres must read back what it just wrote — which requires SELECT.

Most `SECURITY DEFINER` atomic functions use `INSERT INTO events (...) RETURNING id`. This works only if
the executing role has SELECT on `events`. Same for `INSERT INTO clients (...) RETURNING id`.

Additionally, atomic functions often do validation SELECTs before writing:
```sql
SELECT current_phase FROM projects WHERE id = p_project_id INTO v_phase;
```
This also requires SELECT on `projects`.

## Symptom

```
ERROR: permission denied for table clients
```
...from inside a `SECURITY DEFINER` function that has `INSERT` privilege on `clients` but not `SELECT`.

## Fix

Grant SELECT on every table the function reads or uses in `RETURNING`:

```sql
GRANT SELECT ON clients            TO goliathus_writer;
GRANT SELECT ON projects           TO goliathus_writer;
GRANT SELECT ON intake_submissions TO goliathus_writer;
GRANT SELECT ON payments           TO goliathus_writer;
GRANT SELECT ON agent_executions   TO goliathus_writer;
GRANT SELECT ON quality_debt       TO goliathus_writer;
GRANT SELECT ON gate_bypassed_log  TO goliathus_writer;
GRANT SELECT ON inbox_raw          TO goliathus_writer;
```

## Why this is safe for a NOLOGIN writer role

`goliathus_writer` is `NOLOGIN` — no external session can authenticate as it. Only `SECURITY DEFINER`
functions execute as this role. Those functions are the access control boundary. Granting SELECT to
`goliathus_writer` on its own projection tables doesn't open any external read path.

External SELECT access is controlled by `service_role` and `anon`/`authenticated` grants (separate layer).

## Checklist for new atomic functions

When writing a new `SECURITY DEFINER` function that touches a table:
- [ ] Does it use `INSERT ... RETURNING`? → owner role needs SELECT on that table
- [ ] Does it do a validation SELECT before writing? → owner role needs SELECT on that table
- [ ] Does it read from events for dedup check? → owner role needs SELECT on events

Run after adding a new function:
```sql
SELECT has_table_privilege('goliathus_writer', '<table>', 'SELECT');
-- Expected: true
```

## Related

- `db/rls-bypass-for-security-definer-role.md` — the other common cause of `permission denied for table` in SECURITY DEFINER functions
- `db/security-definer-function-grants.md` — full grant setup
