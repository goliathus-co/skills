---
name: external-source-vs-state-transition-grant-pattern
description: PC6's single-writer discipline applies to state transitions, not external-source initial row creation — intake forms and Stripe webhooks need direct service_role INSERT on their first row
type: pattern
confidence: experimental
captured_during: heart-engine-T6
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# External Source vs State Transition Grant Pattern

## The distinction

**PC6** (Master Plan): all state transitions go through atomic SECURITY DEFINER functions. No direct writes to
projection tables from application code.

This applies to **state transitions** — e.g. qualifying an intake, advancing a project phase, recognising
revenue. These operations change the projection's current state and must emit an event for auditability.

It does NOT apply to **external-source initial row creation** — the first INSERT into a projection table
by an authenticated external party (intake form, Stripe webhook). These initial rows:

1. Come from sources that don't have atomic functions for their entry verb (e.g. `intake.submitted`,
   `payment.created` — verbs that fire when external data arrives, before any Goliathus logic runs)
2. Exist as the raw record that atomic functions subsequently act on (qualify, decline, convert)
3. Are created by `service_role` acting on behalf of authenticated external calls

## Which projections need service_role INSERT

| Projection | Initial creation source | Needs service_role INSERT? |
|---|---|---|
| `intake_submissions` | Intake form POST handler (`goliathus-portal`) | Yes |
| `payments` | Stripe webhook handler (payment.succeeded) | Yes |
| `clients` | `system_client_seed()` atomic function | No |
| `projects` | `intake_convert()` atomic function | No |
| `inbox_raw` | (future — raw webhook inbox) | TBD |
| `agent_executions` | Agent framework | No |
| `quality_debt` | Via `quality_debt.recorded` emit | No |
| `gate_bypassed_log` | `log_gate_bypass()` atomic function | No |

## Migration pattern

```sql
-- After M3 revoking all from service_role, selectively re-grant INSERT
-- only on projections with legitimate external-source initial creation:
GRANT INSERT ON intake_submissions TO service_role;
GRANT INSERT ON payments TO service_role;

-- All other state transitions on these tables still go through atomic functions:
-- intake_qualify(), intake_decline(), intake_convert() — not direct UPDATE
-- recognize_revenue() — not direct UPDATE
```

## Why this is correct

The distinction is:
- **Created by external source, no atomic function equivalent** → service_role INSERT allowed for first row
- **State changed by Goliathus logic** → atomic function required (PC6)

An intake form creates the first `intake_submissions` row. After that, `intake_qualify()` transitions it.
`intake_qualify()` does a SELECT on the row (already exists), then UPDATE + event INSERT. The service_role INSERT
path and the atomic function path don't conflict — they operate at different lifecycle stages.

## Decision point: when to add an atomic function instead

If the initial creation needs validation (duplicate check, schema enforcement), add an `intake_submit()` or
`payment_receive()` atomic function instead of direct service_role INSERT. This gives you:
- Dedup semantics on the creation event
- Consistent event trail from the first moment

For v0.1, direct service_role INSERT is acceptable because the portal's form handler and Stripe's webhook
both have their own auth/validation layers before writing to Supabase.

## Related

- `db/rls-bypass-for-security-definer-role.md` — BYPASSRLS for the atomic function owner
- `db/security-definer-function-grants.md` — full grant setup for SECURITY DEFINER functions
