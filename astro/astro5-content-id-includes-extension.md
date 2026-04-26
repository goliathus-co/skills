---
name: astro/astro5-content-id-includes-extension
confidence: experimental
supersedes: []
applies_to: [astro, content, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro 5 — content entry.id includes file extension

## Trap

`getCollection()` entries in Astro 5 have `entry.id` that **includes the file extension** (e.g. `example.mdx`). Astro 4 stripped the extension automatically (e.g. `example`).

Using `params: { slug: page.id }` in `getStaticPaths` generates URLs like `/pages/example.mdx/` instead of `/pages/example/`.

## Fix

Strip the extension in `getStaticPaths`:

```ts
return pages.map((page) => ({
  params: { slug: page.id.replace(/\.(md|mdx)$/, '') },
  props: { page },
}));
```

## Why this changed

Astro 5 content collections use a glob loader that preserves the full file path as the id. The Astro 4 behaviour (auto-strip) was removed as part of the content layer rewrite.

## Applies to

Every `getStaticPaths` call over an Astro 5 content collection that maps `entry.id` to a URL segment.
