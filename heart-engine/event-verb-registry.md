---
name: event-verb-registry
description: How the Heart Engine verb registry works — where to add verbs, what the type system enforces, and what "active" vs "pending" verbs mean
type: pattern
confidence: experimental
captured_during: heart-engine-T5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Heart Engine Verb Registry

## What it is

The verb registry is the TypeScript discriminated union in `packages/@goliathus/events/src/types.ts`.
Every event verb must appear in this union before `emit()` will accept it. Adding a verb = adding a union
member with its payload type. No separate registration step.

## Active verbs at v0.1

```typescript
export type GoliathusEvent =
  | { verb: 'agent.execution_started';    payload: AgentExecutionStartedPayload }
  | { verb: 'agent.execution_completed';  payload: AgentExecutionCompletedPayload }
  | { verb: 'intake.submitted';           payload: IntakeSubmittedPayload }
  | { verb: 'intake.qualified';           payload: IntakeQualifiedPayload }
  | { verb: 'intake.declined';            payload: IntakeDeclinedPayload }
  | { verb: 'intake.converted';           payload: IntakeConvertedPayload }
  | { verb: 'project.phase_advanced';     payload: ProjectPhaseAdvancedPayload }
  | { verb: 'retainer.state_advanced';    payload: RetainerStateAdvancedPayload }
  | { verb: 'payment.created';            payload: PaymentCreatedPayload }
  | { verb: 'payment.revenue_recognized'; payload: PaymentRevenueRecognizedPayload }
  | { verb: 'gate.bypassed';              payload: GateBypassed Payload }
  | { verb: 'quality_debt.recorded';      payload: QualityDebtRecordedPayload }
  | { verb: 'system.client_seeded';       payload: SystemClientSeededPayload };
```

## Verb naming convention

`<domain>.<past_tense_event>` — always past tense (event already happened).

| Good | Bad |
|---|---|
| `intake.qualified` | `intake.qualify` |
| `project.phase_advanced` | `project.advance_phase` |
| `payment.revenue_recognized` | `payment.recognize_revenue` |

The function name is imperative (`intake_qualify`); the event name is past tense (`intake.qualified`).

## What's not in the registry (markdown-only)

Some events exist in `docs/agent-execution-log.md` but don't have registered verbs:
- `sentinel_verdict_issued` — SENTINEL operational verb; no corresponding emit() call
- `agent_protocol_grounding_decided` — session meta; not a system state change

These stay in markdown. Adding them to the registry requires:
1. Defining the payload type
2. Deciding whether they have atomic function equivalents or are emit()-only
3. SENTINEL review (schema change)

## Adding a new verb

1. Add payload type interface in `types.ts`
2. Add union member to `GoliathusEvent`
3. If the verb has an atomic function: add the function migration (SCHEMA blast radius + SENTINEL review)
4. If emit()-only: call `emit()` directly at the appropriate point in application code
5. Update this skill file with the new verb

## Events table verb column

The `verb` column in `events` is `text` with no DB-level enum constraint. The constraint is entirely
TypeScript at the emit() call site. This is intentional — adding a verb doesn't require a DB migration,
only a TypeScript change. The tradeoff: DB allows any string in `verb`; application only emits valid strings.

If a future requirement needs DB-level verb validation: add a CHECK constraint or enum type in a migration.
