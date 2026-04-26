---
name: astro/component-state-matrix
confidence: experimental
supersedes: []
applies_to: [astro, template-astro-content, all astro client projects]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro Component State Matrix

## Rule

Every Astro component that can receive data has four mandatory states: `empty | loading | error | populated`.

States are toggled via a `state` prop. In demos/storybook equivalents, pass the prop directly. In production, the parent determines which state to render based on data availability.

## Pattern (worked example — Hero component)

```astro
---
import EmptyState from '../states/EmptyState.astro';
import LoadingState from '../states/LoadingState.astro';
import ErrorState from '../states/ErrorState.astro';

interface Props {
  state?: 'empty' | 'loading' | 'error' | 'populated';
  headline?: string;
  // ... other props
}

const { state = 'populated', headline = '[Hero headline]' } = Astro.props;
---

{state === 'empty' && (
  <section class="hero">
    <EmptyState message="[Hero headline]" />
  </section>
)}

{state === 'loading' && (
  <section class="hero" aria-busy="true">
    <LoadingState lines={4} />
  </section>
)}

{state === 'error' && (
  <section class="hero">
    <ErrorState message="Couldn't load this section. Try again." />
  </section>
)}

{state === 'populated' && (
  <section class="hero">
    <h1>{headline}</h1>
    <!-- full content -->
  </section>
)}
```

## State primitives (src/components/states/)

Three shared primitives — compose these, don't redefine inline:

- `EmptyState.astro` — dashed border card, message + optional CTA link
- `LoadingState.astro` — shimmer skeleton lines, aria-label for screen readers
- `ErrorState.astro` — rust left-border accent, message + optional retry link

## State matrix by component type

| Component type | Empty | Loading | Error | Populated |
|---|---|---|---|---|
| Data-driven (ContentBlock, ContentList) | EmptyState | LoadingState | ErrorState | full content |
| Navigation (Nav) | logo only | skeleton items | error indicator | full links |
| Forms (ContactForm) | blank fields | submitting skeleton | validation errors | success message |
| Layout (Section) | EmptyState | LoadingState skeleton | ErrorState | full slot content |
| Async-free (Footer) | N/A | N/A | N/A | populated only |

## Reductions allowed

Footer is explicitly async-free (copyright year, static links). Document the reduction in the component's interface comment:

```typescript
// States: populated only — no async data.
```

## Token rule

All state rendering must use token classes only:
- Background: `bg-paper`, `bg-paper-raised`
- Text: `text-ink`, `text-ink-soft`, `text-ink-muted`
- Border/accent: `border-rule`, `text-rust`

No raw hex in component files. Hex belongs in `src/styles/tokens.css` only.

## Vocabulary rule

Placeholder messages must pass brand voice check:
- Allowed: `"Nothing here yet."`, `"Loading..."`, `"Couldn't load this. Try again."`
- Banned: `"Oops something went wrong!"`, `"Loading magic..."`, `"Welcome to your amazing journey."`

## Demo page pattern

`src/pages/_components-demo.astro` renders every component in every state. Underscore prefix prevents production routing in Astro 5. This page is the visual review surface for SENTINEL T3 gate.

```astro
<Hero state="empty" />
<Hero state="loading" />
<Hero state="error" />
<Hero state="populated" headline="Built for work that holds." ... />
```

## Checklist

- [ ] All four states declared (or reduction documented)
- [ ] State primitives used — not redefined inline
- [ ] `aria-busy="true"` on loading state containers
- [ ] `role="alert"` on error state containers
- [ ] `role="status"` on empty state containers
- [ ] Vocabulary check passed on all messages
- [ ] Component renders on `_components-demo.astro` in all states
