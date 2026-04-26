---
name: design/brand-dna-token-mapping
confidence: experimental
supersedes: []
applies_to: [design, tokens, tailwind, astro, css]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Brand DNA Token Mapping — Goliathus Token System

## Three-tier strict hierarchy

```
Tier 1 — Primitives (raw values, named by scale)
  e.g. --color-sand-100: #f5f0e6
       --color-copper-600: #7d2e1f

Tier 2 — Semantic (named by role, references Tier 1)
  e.g. --paper: var(--color-sand-100)
       --rust: var(--color-copper-600)

Tier 3 — Tailwind @theme (utility class generation, references Tier 2)
  e.g. @theme inline { --color-paper: var(--paper); }
  → .bg-paper { background-color: var(--paper); }
```

Components consume **Tier 2** semantic tokens only. Never `--color-sand-100` in a component.

## Token naming rule

Semantic token names describe **role/intent**, not value or scale:
- `--paper` ✓ (role: page canvas)
- `--color-sand-100` ✗ in component (scale name — belongs in primitives only)
- `--bg-sand-light` ✗ (too specific — not reusable)

## Tailwind 4 integration — @theme inline

Use `@theme inline` (not plain `@theme`) to pass CSS variable references through to utility classes. Without `inline`, Tailwind converts colors to OKLCH at build time.

```css
@theme inline {
  --color-paper: var(--paper);   /* → .bg-paper { background-color: var(--paper); } */
  --color-rust: var(--rust);     /* → .text-rust { color: var(--rust); } */
}
```

**Critical trap:** Do NOT define `--font-display` in `:root` AND `@theme inline { --font-display: var(--font-display); }`. This creates a circular reference. `@theme inline` writes to `:root` — it would reference itself.

**Fix:** Put font stacks directly in `@theme inline` (no var() reference for fonts):
```css
@theme inline {
  --font-display: 'Fraunces', Georgia, serif;  /* NOT var(--font-display) */
}
```

The font stack is still available as `var(--font-display)` everywhere because `@theme inline` writes it to `:root`.

## Goliathus Brand DNA exact values

```css
/* Tier 1 — Primitives */
--color-sand-50: #fbf7ed;     /* paper raised */
--color-sand-100: #f5f0e6;    /* paper / canvas */
--color-sand-300: #d6cdb8;    /* rule / borders */
--color-ink-400: #8a8278;     /* ink muted */
--color-ink-600: #4a4440;     /* ink soft */
--color-ink-900: #1a1614;     /* ink / primary text */
--color-ink-950: #0f0d0b;     /* graphite / inverse */
--color-copper-600: #7d2e1f;  /* rust / signature accent */

/* Tier 2 — Semantic */
--paper: var(--color-sand-100);
--paper-raised: var(--color-sand-50);
--ink: var(--color-ink-900);
--ink-soft: var(--color-ink-600);
--ink-muted: var(--color-ink-400);
--rule: var(--color-sand-300);
--graphite: var(--color-ink-950);
--rust: var(--color-copper-600);
```

## What generates from the system

Given the above + `@theme inline`:
- `bg-paper`, `text-paper` — page canvas
- `text-ink`, `bg-ink` — primary text
- `text-ink-soft`, `text-ink-muted` — secondary text, captions
- `border-rule`, `border-rule/50` (Tailwind opacity modifier) — hairline borders
- `text-rust`, `bg-rust` — signature accent (ration strictly per Brand DNA)
- `bg-graphite`, `text-graphite` — inverse sections

## Anti-patterns

❌ `style="color: #7d2e1f"` — hardcoded hex in component
❌ `text-[#7d2e1f]` — arbitrary value bypasses token system
❌ `var(--color-copper-600)` in component — Tier 1 leaking to component
❌ `theme.extend.colors` in tailwind.config.ts — Tailwind v3 pattern, incompatible with v4
