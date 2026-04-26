---
name: astro/content-collections-with-draft-flag
confidence: experimental
supersedes: []
applies_to: [astro, content, mdx, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro Content Collections with Draft Flag

## Config location — Astro 5 change

Astro 5 uses `src/content.config.ts` (at the `src/` root), NOT `src/content/config.ts` (inside the content directory).

Using the old path generates a deprecation warning at build time:
```
Auto-generating collections for folders in "src/content/" that are not defined as collections.
This is deprecated, so you should define these collections yourself in "src/content.config.ts".
```

## Pattern

```ts
// src/content.config.ts
import { defineCollection, z } from 'astro:content';

const pages = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    description: z.string(),
    publishedAt: z.date().optional(),
    draft: z.boolean().default(false),
  }),
});

export const collections = { pages };
```

## Draft flag behaviour

`draft: true` in frontmatter marks content as excluded from production.

By convention (not enforcement), underscore-prefixed MDX files (`_example.mdx`) are both:
- Excluded from Astro routing (underscore prefix convention)
- Marked `draft: true` in frontmatter

Both serve as redundant guards — underscore prefix is build-time routing exclusion; `draft` is content-layer semantic.

```mdx
---
title: "Example page"
description: "Placeholder content. Replace via brief on apply."
draft: true
---

# Example heading
```

## Filtering drafts in production

When building content lists, always filter by `draft`:

```ts
import { getCollection } from 'astro:content';

const allPages = await getCollection('pages', ({ data }) => {
  return import.meta.env.PROD ? !data.draft : true;
});
```

In development: drafts visible. In production: drafts excluded.

## MDX components in content

Import Astro components inside MDX using the `~/` alias:

```mdx
import ContentBlock from '~/components/content/ContentBlock.astro';

<ContentBlock state="populated" title="Section title">
  Content here.
</ContentBlock>
```

Astro MDX supports `.astro` component imports — no client: directive needed for static components.

## Template placeholders vs client drafts

Template example pages (e.g. `example.mdx`) should use `draft: false`. They are published examples that prove the pipeline works end-to-end, not work-in-progress client content. Setting `draft: true` on a template example causes the PROD build to emit zero pages, making build verification ("HTML contains expected content") impossible.

Rule: **template examples ship as published (`draft: false`); client drafts use `draft: true` and are filtered from PROD by the collection filter.**

## Path alias

`~/` resolves to `src/` via `tsconfig.json` paths:
```json
{ "paths": { "~/*": ["src/*"] } }
```
