---
name: smoke-test-pattern
description: How to write idempotent smoke tests for Heart Engine atomic functions — fixed UUIDs, sequential flow, idempotency batch, event count verification
type: pattern
confidence: experimental
captured_during: heart-engine-T6
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Heart Engine Smoke Test Pattern

## Purpose

Verify the entire Heart Engine stack after any DB migration: functions exist, have correct permissions,
write projections correctly, and enforce idempotency. Run against the live remote DB (not a local stub).

## Key design choices

**Fixed test UUIDs** — smoke tests use hard-coded UUIDs so they're idempotent across runs:
```typescript
const SMOKE_CLIENT_SLUG = 'smoke-client-heart-engine-v01';
const SMOKE_INTAKE_ID = '00000000-0000-0000-0000-000000000001';
const SMOKE_PAYMENT_STRIPE_ID = 'pi_smoke_test_heart_engine_v01';
```

First run: creates rows. Second run: hits dedup keys, returns same event_ids, no new rows. Safe to re-run.

**Sequential flow** — functions have FK dependencies; must run in order:
```
system_client_seed → intake_qualify → intake_convert → project_advance_phase → retainer_advance_state
                                ↓
                        intake_submissions row exists (FK for qualify/convert)
                                              ↓
                                     projects row exists (FK for advance_phase)
```

**Idempotency batch** — re-call all 6 functions after initial run. Assert same event_id returned:
```typescript
const secondCallEventId = await callFunction(sameArgs);
assert(secondCallEventId === firstCallEventId, 'Idempotency violated');
```

**Event count** — verify events were written (not just function return value):
```typescript
const { count } = await service.from('events').select('*', { count: 'exact', head: true });
assert(count !== null && count > 0);
```
Note: `{ count: 'exact', head: true }` — `data` is null when using `head: true`; count is on the response object.

## RPC call pattern

```typescript
// Always cast the return value — supabase-js returns unknown[]
const { data, error } = await service.rpc('system_client_seed', {
  p_slug: SMOKE_CLIENT_SLUG,
  p_name: 'Smoke Test Client',
  p_tier: 'ESSENCE',
  p_actor: 'smoke-test',
  p_correlation_id: SMOKE_CORRELATION_ID,
});

if (error) throw new Error(`system_client_seed failed: ${error.message}`);
const eventId = (data as Array<{ event_id: string }>)[0].event_id;
```

The `data` type is `unknown` from supabase-js. Cast to `Array<{ event_id: string }>` — the function returns
`RETURNS TABLE (event_id UUID, ...)` which serialises as an array.

## Direct INSERT for external-source rows

`intake_submissions` and `payments` have initial rows created by external sources (form, Stripe webhook),
not by atomic functions. In smoke tests, create these directly:

```typescript
// Create intake_submissions row directly (simulating portal form POST)
await service.from('intake_submissions').upsert({
  id: SMOKE_INTAKE_ID,
  status: 'submitted',
  // ...
}, { onConflict: 'id', ignoreDuplicates: true });
```

Then call `intake_qualify(SMOKE_INTAKE_ID, ...)` as the atomic function path.

## Test structure template

```typescript
let passed = 0;
let failed = 0;

async function test(name: string, fn: () => Promise<void>) {
  try {
    await fn();
    console.log(`  ✓ ${name}`);
    passed++;
  } catch (err) {
    console.error(`  ✗ ${name}: ${err instanceof Error ? err.message : String(err)}`);
    failed++;
  }
}

// ... tests ...

console.log(`\n${passed}/${passed + failed} PASS`);
if (failed > 0) process.exit(1);
```

## When to run

- After any `supabase db push --linked` (migration applied)
- After any change to atomic function definitions
- After any grant/role change (migrations touching BYPASSRLS, privileges)
- Before tagging a release (e.g. heart-engine-v0.1)

## Related

- `db/atomic-function-idempotency-pattern.md` — how dedup_key idempotency works inside functions
- `db/insert-returning-requires-select.md` — SELECT privilege required for INSERT...RETURNING
- `db/rls-bypass-for-security-definer-role.md` — why BYPASSRLS is required for goliathus_writer
