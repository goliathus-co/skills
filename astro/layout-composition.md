---
name: astro/layout-composition
confidence: experimental
supersedes: []
applies_to: [astro, template-astro-content, all astro client projects]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro Layout Composition — BaseLayout + ContentLayout

## Pattern

Two-layer layout system. BaseLayout owns the HTML shell; ContentLayout wraps it for prose content.

```
BaseLayout.astro          — <html>, <head>, meta, OG, <body>
└── ContentLayout.astro   — container widths, prose typography
    └── MDX page / .astro page
```

## BaseLayout.astro

```astro
---
import '../styles/global.css';

interface Props {
  title?: string;
  description?: string;
  ogTitle?: string;
  ogDescription?: string;
  ogImage?: string;
  canonicalUrl?: string;
}

const {
  title = 'Goliathus',
  description = 'Built by Goliathus.',
  ogTitle = title,
  ogDescription = description,
  ogImage,
  canonicalUrl,
} = Astro.props;
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>{title}</title>
    <meta name="description" content={description} />
    <!-- OG, Twitter, canonical -->
    <slot name="head" />
  </head>
  <body class="bg-paper text-ink font-body">
    <slot name="nav" />
    <slot />
    <slot name="footer" />
  </body>
</html>
```

Key points:
- Three named slots: `nav`, default, `footer` — consumers fill what they need
- `global.css` imported once here — not in individual pages
- OG defaults cascade: `ogTitle` falls back to `title`, etc.
- `<slot name="head" />` for per-page additional `<head>` content (schemas, preloads)

## ContentLayout.astro

```astro
---
import BaseLayout from './BaseLayout.astro';

interface Props { /* same as BaseLayout */ }

const props = Astro.props;
---

<BaseLayout {...props}>
  <slot name="nav" slot="nav" />
  <main>
    <div class="content-layout__container">
      <article class="prose">
        <slot />
      </article>
    </div>
  </main>
  <slot name="footer" slot="footer" />
</BaseLayout>
```

Key point: named slots must be re-passed explicitly (`slot="nav" slot="nav"`) — Astro does not automatically forward named slots through layout layers.

## Usage in a page

```astro
---
import BaseLayout from '../layouts/BaseLayout.astro';
import Nav from '../components/layout/Nav.astro';
import Footer from '../components/layout/Footer.astro';
---

<BaseLayout title="Page title">
  <Nav slot="nav" state="populated" logo={{ text: 'Goliathus', href: '/' }} items={[...]} />
  <main>...</main>
  <Footer slot="footer" companyName="Goliathus" />
</BaseLayout>
```

## Usage with ContentLayout (MDX)

```astro
---
import ContentLayout from '../layouts/ContentLayout.astro';
const { frontmatter } = Astro.props;
---

<ContentLayout title={frontmatter.title} description={frontmatter.description}>
  <slot />
</ContentLayout>
```

## Gotcha: global.css import location

Import `global.css` in `BaseLayout.astro` only. Importing in individual pages causes duplicate `<style>` tags in the built HTML. Astro does not deduplicate CSS imports across layouts.

## Prose styles

`ContentLayout` applies prose styles via `:global()` selectors scoped to `.prose`. This avoids Tailwind prose plugin dependency while keeping styles predictable for MDX content.

Typography uses token variables, not raw values:
- `font-family: var(--font-display)` for headings
- `font-family: var(--font-mono)` for code blocks
- `color: var(--ink)` / `var(--ink-soft)` for text hierarchy

## Container widths

- `BaseLayout` — no max-width (full bleed, components control their own containers)
- `ContentLayout` — `max-width: 72ch` centered, good for longform reading
