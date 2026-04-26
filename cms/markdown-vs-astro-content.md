---
name: cms/markdown-vs-astro-content
confidence: experimental
supersedes: []
applies_to: [astro, content, mdx, cms, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Markdown vs Astro Content Collections — Decision Rationale

## The question

When should a Goliathus project use Astro content collections (MDX) vs a raw Markdown pipeline vs a headless CMS vs a hybrid?

## Decision matrix

| Signal | Use MDX content collections | Use headless CMS | Use raw Markdown |
|---|---|---|---|
| Client edits content | No (dev-only) | Yes | No (dev-only) |
| Content has structured data (images, tags, references) | Sometimes | Yes | Rarely |
| Content volume | Low–medium | Medium–high | Low |
| Offline / version-controlled content | Yes | No (CMS-managed) | Yes |
| Multi-language content | With i18n plugin | With CMS locale support | Rarely |
| Template default | ✓ | Wired as skeleton | — |

## Goliathus defaults

**`template-astro-content`** ships both:
1. **Astro content collections** (MDX) — for placeholder and documentation pages that live in the repo. Managed by developer. Draft flag prevents production exposure.
2. **Sanity connector skeleton** — wired but unconfigured. Activated at engagement start when client needs editorial CMS access.

This hybrid is intentional: some content (legal pages, privacy, error pages) is dev-managed; other content (blog, case studies, team bios) is client-managed through Sanity.

## Why MDX over a custom Markdown pipeline

- Type-safe frontmatter via Zod schemas in `content.config.ts`
- `getCollection()` API with filter support (draft exclusion, date sorting)
- Component imports inside `.mdx` files — Astro components render without client JS
- Hot-reload in dev
- Works with Astro's static output — no server required for content listing

Custom Markdown pipelines (remark + rehype manual wiring) are harder to maintain across Astro major versions and duplicate what content collections provide.

## Why Sanity over other CMSes

- Sanity Studio deploys to client's domain — no third-party login required
- GROQ query language handles relational content cleanly
- Real-time preview with live data
- Free tier adequate for ESSENCE/PRESENCE engagements

This is a provider decision at engagement start, not a template constraint. The `cms/lazy-headless-cms-client-edge-binding` pattern applies regardless of CMS vendor.

## Switching from MDX to CMS at apply time

Typical apply-time flow:
1. Provision Sanity project (or other CMS)
2. Set `PUBLIC_SANITY_PROJECT_ID` in CF Pages env
3. Create CMS schema matching content types needed
4. Migrate placeholder MDX pages to CMS entries
5. Remove `src/content/pages/` or keep for dev-managed pages

MDX and CMS coexist — no forced migration.
