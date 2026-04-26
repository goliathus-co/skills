---
name: typed-emit-discriminated-union
description: Design pattern for a typed event emit module using TypeScript discriminated unions — each verb has its own payload type, preventing mismatched payloads at compile time
type: pattern
confidence: experimental
captured_during: heart-engine-T5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Typed Emit with Discriminated Union

## The pattern

Define one union type per event verb. The `emit()` function narrows on `verb` and enforces the matching payload
shape at the call site. No `Record<string, unknown>` in application code.

## Type definition

```typescript
// packages/@goliathus/events/src/types.ts

export type GoliathusEvent =
  | {
      verb: 'intake.submitted';
      payload: {
        source: string;
        email: string;
        submitted_at: string; // ISO-8601
      };
    }
  | {
      verb: 'intake.qualified';
      payload: {
        intake_submission_id: string;
        actor: string;
      };
    }
  | {
      verb: 'project.phase_advanced';
      payload: {
        project_id: string;
        from_phase: string;
        to_phase: string;
        actor: string;
      };
    }
  // ... one union member per registered verb

export type EventVerb = GoliathusEvent['verb'];

export type EventContext = {
  correlationId: string;
  actor: string;
  dedupKey?: string;
  occurredAt?: string; // ISO-8601; defaults to now() if omitted
};
```

## Emit function signature

```typescript
// packages/@goliathus/events/src/emit.ts

export async function emit(
  event: GoliathusEvent,
  context: EventContext,
  options?: { client?: SupabaseClient }
): Promise<{ eventId: string }> {
  // narrows to correct payload via event.verb
  const { verb, payload } = event;
  // insert to events table or call /api/events/ingest
}
```

## Call site

```typescript
import { emit } from '@goliathus/events';

await emit(
  {
    verb: 'intake.qualified',
    payload: {
      intake_submission_id: submissionId,
      actor: 'admin-triage',
    },
  },
  {
    correlationId: correlationId,
    actor: 'admin-triage',
    dedupKey: `intake.qualified:${submissionId}`,
  }
);
// TypeScript error if payload doesn't match intake.qualified shape
// TypeScript error if verb is not in GoliathusEvent union
```

## Adding a new verb

1. Add union member to `GoliathusEvent` in `types.ts`
2. TypeScript immediately fails at any `emit()` call that passes the new verb without correct payload
3. No other registration required — the type IS the registry at compile time

## Tradeoff vs runtime schema validation

Compile-time only. If you need runtime validation (API boundary), add Zod schema alongside the TypeScript type:

```typescript
import { z } from 'zod';

export const intakeQualifiedPayloadSchema = z.object({
  intake_submission_id: z.string().uuid(),
  actor: z.string().min(1),
});
```

Use Zod at the `/api/events/ingest` webhook boundary where the payload arrives as `unknown` JSON.
Use TypeScript discriminated union within the application where the type is already known.

## Related

- `events/dual-mode-studio-vs-webhook.md` — how emit() works in admin (direct Supabase) vs portal (via webhook)
- `webhook/shared-secret-auth-pattern.md` — auth for /api/events/ingest
