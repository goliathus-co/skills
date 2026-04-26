---
name: security-definer-function-grants
description: Grant discipline for SECURITY DEFINER functions with a designated writer role — owner, executor, and search_path setup to prevent privilege escalation
type: pattern
confidence: experimental
captured_during: heart-engine-T4
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Security Definer Function Grants

`SECURITY DEFINER` functions execute with the privileges of the function owner, not the caller. This requires explicit setup of: (1) a minimal-privilege owner role, (2) search_path to prevent hijack, (3) EXECUTE granted only to callers who should invoke.

## Setup sequence

### 1. Create owner role (done once)

```sql
CREATE ROLE goliathus_writer NOLOGIN;
GRANT USAGE ON SCHEMA public TO goliathus_writer;
GRANT SELECT, INSERT ON TABLE events TO goliathus_writer;
-- Per projection: INSERT + UPDATE only (SELECT for dedup checks)
GRANT INSERT, UPDATE ON TABLE <projection> TO goliathus_writer;
```

### 2. Prerequisite: migration user must be a member of owner role

Before `ALTER FUNCTION ... OWNER TO goliathus_writer` will succeed:

```sql
-- The role running migrations (postgres on Supabase) must be a member
GRANT goliathus_writer TO postgres;
-- Owner role must have CREATE on the schema to own objects in it
GRANT CREATE ON SCHEMA public TO goliathus_writer;
```

Without `GRANT goliathus_writer TO postgres`: `ERROR: must be able to SET ROLE "goliathus_writer"`
Without `GRANT CREATE ON SCHEMA public TO goliathus_writer`: `ERROR: permission denied for schema public`

### 3. Function definition

```sql
CREATE OR REPLACE FUNCTION my_function(...)
RETURNS ...
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public   -- prevents search_path hijack (see below)
AS $$ ... $$;
```

### 4. Grant ownership and EXECUTE

```sql
ALTER FUNCTION my_function(...) OWNER TO goliathus_writer;
GRANT EXECUTE ON FUNCTION my_function(...) TO service_role;
-- Do NOT grant to anon or authenticated unless explicitly needed
```

## Why search_path matters

Without `SET search_path = public`, an attacker who can create objects in a schema higher up the search path can shadow `pg_catalog` functions (e.g. `length()`, `now()`) with malicious versions that execute inside a SECURITY DEFINER context at elevated privilege. Always set it explicitly.

## Verification

```sql
SELECT proname, prosecdef, proconfig
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace
  AND prosecdef = true;
-- Expected: every row has proconfig = ['search_path=public']
-- Any NULL proconfig on a SECURITY DEFINER function = fix before shipping
```

```sql
SELECT p.proname, r.rolname AS owner
FROM pg_proc p JOIN pg_roles r ON r.oid = p.proowner
WHERE p.pronamespace = 'public'::regnamespace AND p.prosecdef = true;
-- Expected: all rows show goliathus_writer (or designated writer role)
```

## Writer role attributes

```sql
SELECT rolsuper, rolbypassrls, rolcanlogin FROM pg_roles WHERE rolname = 'goliathus_writer';
-- Expected: false, false, false
-- rolsuper must be false — a superuser owner negates SECURITY DEFINER isolation
```

## What service_role gets

| Resource | service_role grant |
|---|---|
| Projection tables | SELECT only (REVOKE ALL first, then re-grant SELECT) |
| Events table | Not restricted below ALL by default; consider tightening |
| Atomic functions | EXECUTE only |

`service_role` is not a Postgres superuser (`rolsuper=false`) despite having `rolbypassrls=true`. REVOKE works on it. Supabase default privileges (`ALTER DEFAULT PRIVILEGES ... GRANT ALL ON TABLES TO service_role`) run at project init — this is why new tables get DELETE/TRUNCATE/TRIGGER automatically. To tighten: `REVOKE ALL ON <table> FROM service_role; GRANT SELECT ON <table> TO service_role;`
