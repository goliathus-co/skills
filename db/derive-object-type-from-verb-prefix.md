---
name: derive-object-type-from-verb-prefix
description: Auto-derive object_type from dotted verb namespace when caller doesn't supply it explicitly — removes a foot-gun in event-sourced systems where NOT NULL constraints enforce type presence
type: pattern
confidence: experimental
captured_during: heart-engine-V4-closure
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Derive object_type from Verb Prefix

## The problem

Event tables often carry an `object_type` column — the entity class the event refers to
(`project`, `intake_submission`, `payment`, etc.). When callers emit events without
specifying `object_type` explicitly, a NOT NULL constraint fires:

```
ERROR: null value in column "object_type" of relation "events" violates not-null constraint
```

Discovered at V4 live curl: `emit()` direct path passed `object_type: null`
when `context.objectType` was absent. Test returned HTTP 422 instead of 200.

## The fix

The dotted verb namespace already encodes the domain:

```
intake.submitted    → intake  → intake_submission
project.created     → project → project
payment.received    → payment → payment
agent.execution_*   → agent   → agent_execution
retainer.*          → retainer → project (retainers are sub-entities of projects)
system.*            → system  → system
gate.*              → gate    → gate_bypassed_log
quality_debt.*      → quality_debt → quality_debt
```

Derive from prefix, fall back to the prefix itself if no explicit mapping:

```typescript
function deriveObjectType(verb: string): string {
  const prefix = verb.split('.')[0] ?? verb;
  const mapping: Record<string, string> = {
    intake:       'intake_submission',
    project:      'project',
    payment:      'payment',
    agent:        'agent_execution',
    retainer:     'project',
    system:       'system',
    gate:         'gate_bypassed_log',
    quality_debt: 'quality_debt',
  };
  return mapping[prefix] ?? prefix;
}

// In the INSERT:
object_type: context.objectType ?? deriveObjectType(verb),
```

The fallback `?? prefix` means unknown future verbs don't break — they use the domain
prefix directly, which is always at least semantically correct.

## Where this applies

Any event-sourced system with:
- `object_type` column on events table (NOT NULL)
- Dotted verb namespacing (`<domain>.<event>` format)
- Callers who may not always supply `object_type` explicitly

## When NOT to use

If `object_type` is also an enum with a strict known set — the fallback `?? prefix` would
produce unmapped enum values. In that case, emit an error instead of silently deriving.
Only use the silent derivation when `object_type` is a free-text column.

## Mapping maintenance

The mapping lives alongside the verb registry. When adding a new verb domain, add a
mapping entry at the same time. Convention: if the domain is the same word as the
object_type, it's safe to rely on the `?? prefix` fallback rather than adding an
explicit entry.

## Related

- `heart-engine/event-verb-registry.md` — verb registry structure
- `events/typed-emit-discriminated-union.md` — how caller-side payloads are typed per verb
