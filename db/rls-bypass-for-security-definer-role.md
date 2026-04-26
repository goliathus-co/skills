---
name: rls-bypass-for-security-definer-role
description: When SECURITY DEFINER functions own RLS-enabled projection tables, the function-owner role must have BYPASSRLS or every write is denied — even with correct table grants
type: pattern
confidence: experimental
captured_during: heart-engine-T6
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# RLS Bypass for SECURITY DEFINER Role

## The problem

PostgreSQL RLS (Row Level Security) applies to the executing role. When `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
is set and no policies exist for a role, that role is denied all row access — regardless of table-level grants.

`SECURITY DEFINER` functions execute as their owner role. If the owner role has `rolbypassrls=false` (the default),
every write and read inside the function hits RLS denial. This happens even when the owner has explicit
`GRANT INSERT, UPDATE, SELECT ON table TO owner_role`.

Symptom: `ERROR: permission denied for table <projection>` from inside a SECURITY DEFINER function.
Misleading because `GRANT ALL ON <table> TO owner_role` does not fix it — the issue is RLS, not table grants.

## The fix

```sql
ALTER ROLE goliathus_writer BYPASSRLS;
```

## When this is safe

Safe only when the role is:
1. **NOLOGIN** — no external session can authenticate as this role
2. **Owned by a controlled access path** — only SECURITY DEFINER functions execute as it; no direct DB connection

The functions ARE the access control boundary. BYPASSRLS for the function-owner role does not weaken the security
model when no one can connect as that role. The path is: caller → EXECUTE on function → function executes as
goliathus_writer (BYPASSRLS) → writes projection → returns.

If the role ever gains LOGIN capability, BYPASSRLS must be revoked and RLS policies written for each table.

## Verification

```sql
-- Check BYPASSRLS attribute
SELECT rolname, rolbypassrls, rolcanlogin FROM pg_roles WHERE rolname = 'goliathus_writer';
-- Expected: rolbypassrls = true, rolcanlogin = false

-- Check RLS is enabled on projection
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class WHERE relnamespace = 'public'::regnamespace AND relkind = 'r';
-- Expected: relrowsecurity = true for each projection table

-- Verify function can actually write (run smoke test)
SELECT * FROM system_client_seed('test-slug', 'Test Client', 'ESSENCE', 'test-actor', gen_random_uuid());
```

## Diagnosis sequence

When SECURITY DEFINER function fails with `permission denied for table X`:

1. Check `rolbypassrls` for the function owner → if false, this is the issue
2. Check `relrowsecurity` on the table → if true with no policies for the role, BYPASSRLS is required
3. Check table grants (`SELECT relacl FROM pg_class WHERE relname = 'X'`) → confirm grants exist
4. If grants present but writes still fail after BYPASSRLS: check `INSERT ... RETURNING` requires SELECT
   (see companion skill: `db/insert-returning-requires-select.md`)

## Related

- `db/security-definer-function-grants.md` — full SECURITY DEFINER setup sequence
- `db/external-source-vs-state-transition-grant-pattern.md` — when service_role needs INSERT vs atomic function
