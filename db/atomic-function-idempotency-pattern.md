---
name: atomic-function-idempotency-pattern
description: SELECT-then-INSERT idempotency pattern for event-sourced atomic DB functions — prevents duplicate events on retry without advisory locks
type: pattern
confidence: experimental
captured_during: heart-engine-T4
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Atomic Function Idempotency Pattern

Every state-transition function in an event-sourced system must be safe to call twice with the same intent. Network retries, webhook re-delivery, and admin double-submit all produce duplicate calls. The dedup key is the guard.

## Pattern

```sql
CREATE OR REPLACE FUNCTION example_action(
  p_entity_id UUID,
  p_actor     TEXT,
  p_correlation_id UUID
)
RETURNS TABLE (event_id UUID, entity_id UUID)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_existing_event_id UUID;
  v_existing_entity_id UUID;
  v_new_event_id UUID;
  v_dedup_key TEXT := 'example.actioned:' || p_entity_id;
BEGIN
  -- Step 1: Check for prior execution with same dedup key
  SELECT e.id, (e.payload->>'entity_id')::uuid
    INTO v_existing_event_id, v_existing_entity_id
  FROM events e
  WHERE e.verb = 'example.actioned' AND e.dedup_key = v_dedup_key
  LIMIT 1;

  -- Step 2: Return early if already processed
  IF v_existing_event_id IS NOT NULL THEN
    RETURN QUERY SELECT v_existing_event_id, v_existing_entity_id;
    RETURN;
  END IF;

  -- Step 3: Validation (raise before any writes)
  -- ... validation logic here ...

  -- Step 4: Projection write
  -- INSERT / UPDATE on projection table

  -- Step 5: Event write (dedup_key enforced by UNIQUE constraint)
  INSERT INTO events (correlation_id, actor, verb, object_type, object_id, payload, dedup_key)
  VALUES (p_correlation_id, p_actor, 'example.actioned', 'entity', p_entity_id, '{}', v_dedup_key)
  RETURNING id INTO v_new_event_id;

  RETURN QUERY SELECT v_new_event_id, p_entity_id;
END;
$$;
```

## Dedup key construction

`'<verb>:<stable_identifier>'` — the stable identifier must be invariant across retries:

| Function | Dedup key |
|---|---|
| `project_create` | `'project.created:' \|\| p_slug` |
| `project_advance_phase` | `'project.phase_advanced:' \|\| p_project_id \|\| ':' \|\| p_to_phase \|\| ':' \|\| p_actor` |
| `intake_qualify` | `'intake.qualified:' \|\| p_intake_submission_id` |
| `system_client_seed` | `'system.client_seeded:' \|\| p_slug` |

Include `p_actor` in the dedup key only when the same entity transition can legitimately occur multiple times by different actors (phase advance by different operators at different times).

## Constraint setup

The events table must have:
```sql
CONSTRAINT events_dedup_key_unique UNIQUE (verb, dedup_key)
-- NOT DEFERRABLE (default) — checked immediately, blocks phantom events
```

Do NOT use `DEFERRABLE INITIALLY DEFERRED`: under concurrent execution, TXN A checks dedup→not found, TXN B checks dedup→not found, TXN A inserts→commits. TXN B inserts→unique violation at commit, but TXN A's event_id was already returned to its caller. The deferred constraint allows this gap; immediate enforcement prevents it.

## Return shape

Always `RETURNS TABLE (event_id UUID, entity_id UUID)` — return the event_id so callers can include it in correlation traces. On idempotent return, the event_id is the *original* event_id from the prior call, not a new one.

## Order of operations

1. Dedup check (SELECT only, no writes)
2. Validation (raise before any writes — avoids partial state)
3. Projection write (INSERT or UPDATE)
4. Event INSERT (dedup constraint is final guard)
5. RETURN QUERY

Never validate after writing. A validation failure after a projection write leaves an orphaned row.
