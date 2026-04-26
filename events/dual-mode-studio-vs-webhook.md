---
name: dual-mode-studio-vs-webhook
description: One emit() function with two execution paths — direct Supabase INSERT in admin (service_role available), HTTP POST to /api/events/ingest in portal/external callers (no direct DB access)
type: pattern
confidence: experimental
captured_during: heart-engine-T5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Dual Mode: Studio Direct vs Webhook

## The problem

Two contexts need to emit events with identical semantics:

1. **goliathus-admin** — has `SUPABASE_SERVICE_ROLE_KEY`, runs server-side, can INSERT directly to `events`
2. **goliathus-portal** / external callers — no direct DB access; must go through the `/api/events/ingest` webhook

A single `@goliathus/events` module must work in both contexts without the call site needing to know which path runs.

## Mode selection

```typescript
// packages/@goliathus/events/src/emit.ts

export type EmitMode = 'direct' | 'webhook';

export async function emit(
  event: GoliathusEvent,
  context: EventContext,
  options?: {
    mode?: EmitMode;              // default: 'direct' if SUPABASE_SERVICE_ROLE_KEY available
    ingestUrl?: string;           // required for 'webhook' mode
    ingestSecret?: string;        // required for 'webhook' mode
    client?: SupabaseClient;      // optional override for 'direct' mode
  }
): Promise<{ eventId: string }> {
  const mode = options?.mode ?? (process.env.SUPABASE_SERVICE_ROLE_KEY ? 'direct' : 'webhook');

  if (mode === 'direct') {
    return emitDirect(event, context, options?.client);
  } else {
    return emitViaWebhook(event, context, options?.ingestUrl!, options?.ingestSecret!);
  }
}
```

## Direct path (admin)

```typescript
async function emitDirect(
  event: GoliathusEvent,
  context: EventContext,
  client?: SupabaseClient
): Promise<{ eventId: string }> {
  const supabase = client ?? createServiceRoleClient();
  const { data, error } = await supabase.from('events').insert({
    correlation_id: context.correlationId,
    actor: context.actor,
    verb: event.verb,
    object_type: deriveObjectType(event.verb),
    payload: event.payload,
    dedup_key: context.dedupKey,
    occurred_at: context.occurredAt ?? new Date().toISOString(),
  }).select('id').single();

  if (error) throw new Error(`emit direct failed: ${error.message}`);
  return { eventId: data.id };
}
```

## Webhook path (portal / external)

```typescript
async function emitViaWebhook(
  event: GoliathusEvent,
  context: EventContext,
  ingestUrl: string,
  ingestSecret: string
): Promise<{ eventId: string }> {
  const res = await fetch(`${ingestUrl}/api/events/ingest`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-goliathus-ingest-secret': ingestSecret,
    },
    body: JSON.stringify({ verb: event.verb, payload: event.payload, context }),
  });

  if (!res.ok) throw new Error(`emit webhook failed: ${res.status} ${await res.text()}`);
  const json = await res.json();
  return { eventId: json.eventId };
}
```

## Environment variable pattern

| Var | admin | portal |
|---|---|---|
| `SUPABASE_SERVICE_ROLE_KEY` | ✓ present | ✗ absent |
| `GOLIATHUS_INGEST_URL` | optional | ✓ required |
| `GOLIATHUS_INGEST_SECRET` | present (for verification) | ✓ required |

The mode auto-selects based on env var presence. No conditional imports needed at call site.

## Supabase runtime requirement

`@goliathus/supabase-js` uses Node.js `net` and `tls` APIs. Both paths (direct + webhook) require:

```typescript
// In any Next.js API route or server component using this module:
export const runtime = 'nodejs'; // not 'edge'
```

CF Pages / Edge runtime: use webhook mode only (no direct Supabase access from edge).

## Related

- `events/typed-emit-discriminated-union.md` — type system for the event payload
- `webhook/shared-secret-auth-pattern.md` — auth for the webhook mode endpoint
