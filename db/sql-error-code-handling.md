---
name: sql-error-code-handling
description: SQLSTATE error code discipline for stored procedures — RAISE with standard codes so callers can distinguish validation errors from constraint violations from not-found
type: pattern
confidence: experimental
captured_during: heart-engine-T4
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# SQL Error Code Handling

Stored procedures must raise errors with explicit SQLSTATE codes. Without them, all errors look like `ERROR 00000` to callers — indistinguishable at the application layer.

## Standard codes used in atomic functions

Current implementation uses PostgreSQL-specific SQLSTATE codes. SQLSTATE is ISO SQL standard; the class meanings are cross-provider, but specific codes within each class vary by implementation.

| SQLSTATE | Class | Meaning | When to use |
|---|---|---|---|
| `22023` | Data Exception | Invalid parameter value | Input fails domain validation (invalid tier, unknown state, bad format) |
| `23503` | Integrity Constraint | Foreign key violation | Referenced row doesn't exist |
| `23505` | Integrity Constraint | Unique violation | Duplicate insert on unique column |
| `P0002` | PL/pgSQL | No data found | Expected row not found (e.g. SELECT INTO returns nothing) |
| `42501` | Insufficient Privilege | Permission denied | Role lacks required privilege |

## Pattern

```sql
-- Validation errors: use 22023 (invalid_parameter_value)
IF p_tier NOT IN ('essence', 'presence', 'platform', 'institution', 'legacy') THEN
  RAISE EXCEPTION 'invalid tier: %', p_tier USING ERRCODE = '22023';
END IF;

-- Not found: use P0002 or check via explicit SELECT
IF v_entity_id IS NULL THEN
  RAISE EXCEPTION 'entity % not found', p_entity_id USING ERRCODE = 'P0002';
END IF;

-- State machine violations: use 22023 with descriptive message
IF p_to_phase NOT IN (v_valid_next) THEN
  RAISE EXCEPTION 'invalid phase transition: % -> %', v_current_phase, p_to_phase
    USING ERRCODE = '22023';
END IF;
```

## Application-layer handling (TypeScript)

```typescript
try {
  const { data } = await supabase.rpc('project_create', params);
} catch (error) {
  if (error.code === '22023') {
    // Validation error — safe to show caller, rethrow as 400
  } else if (error.code === '23503') {
    // FK violation — referenced entity missing, 404 or 422
  } else {
    // Unexpected — log fully, return 500
  }
}
```

## Message format

`'<what was invalid>: <actual value>'` — include the actual value so logs are actionable without re-querying.

Prefer: `RAISE EXCEPTION 'invalid tier_eligibility: %', p_tier_eligibility USING ERRCODE = '22023'`
Avoid: `RAISE EXCEPTION 'invalid input'` — opaque, forces re-read of code to diagnose

## Validation order

1. Enum/range checks on input parameters
2. Foreign key existence checks (SELECT before INSERT to get clear error vs constraint message)
3. State precondition checks (entity must be in expected state)
4. Business rule checks (score threshold, amount > 0, etc.)

Raise at first failure — do not accumulate errors. Stored procedures are synchronous; callers retry after fixing the first error.
