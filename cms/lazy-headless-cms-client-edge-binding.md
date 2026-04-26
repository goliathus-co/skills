---
name: cms/lazy-headless-cms-client-edge-binding
confidence: experimental
supersedes: []
applies_to: [cloudflare, astro, cms, sanity, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Lazy Headless CMS Client — Edge/CF Workers Binding

## Problem

CF Workers injects environment variables at request time, not at module load time.
Instantiating a CMS client at the top level of a module → env vars are still empty → client receives empty/invalid config → runtime error at first request.

CF lessons L20: any SDK client initialized at module top-level on CF Workers will fail.

## Pattern — lazy singleton

```ts
// src/lib/sanity.ts (current implementation: Sanity — may be replaced at engagement)
import { createClient, type SanityClient } from '@sanity/client';

let _client: SanityClient | null = null;

export function getSanityClient(): SanityClient {
  if (!_client) {
    // L2: use || not ?? — empty string is not nullish; ?? doesn't fire for ""
    const projectId = import.meta.env.PUBLIC_SANITY_PROJECT_ID || '';
    const dataset = import.meta.env.PUBLIC_SANITY_DATASET || 'production';

    if (!projectId) {
      throw new Error(
        'PUBLIC_SANITY_PROJECT_ID not set. See ENV.md.',
      );
    }

    _client = createClient({
      projectId,
      dataset,
      apiVersion: '2025-01-01',
      useCdn: import.meta.env.PROD,
    });
  }
  return _client;
}
```

Call `getSanityClient()` inside async functions, route handlers, or Astro frontmatter — never at module scope.

## Why this works

- First call happens inside a request handler, after CF Workers has bound env vars
- Subsequent calls return cached `_client` — no re-initialization overhead
- `_client = null` reset possible for testing (replace module or inject mock)

## CF lessons applied

| Lesson | Application |
|---|---|
| L20 | Lazy singleton — never top-level instantiation |
| L1 | `PUBLIC_` prefix on vars needed at prerender time |
| L2 | `||` not `??` for env var fallbacks |

## Swap contract

This skill is capability-named. Sanity is the current implementation. The pattern applies to any headless CMS client:
- Payload CMS: same pattern, different import
- Contentful: same pattern, different client
- Strapi: same pattern, different fetch URL

The swap cost is confined to `src/lib/<cms>.ts`. No component changes required.

## Guard pattern at page level

```astro
---
import { getSanityClient } from '~/lib/sanity';
import EmptyState from '~/components/states/EmptyState.astro';

let posts = [];
let error = null;
const configured = Boolean(import.meta.env.PUBLIC_SANITY_PROJECT_ID);

if (configured) {
  try {
    posts = await getSanityClient().fetch('*[_type == "post"][0...10]');
  } catch (e) {
    error = e instanceof Error ? e.message : 'Unknown error';
  }
}
---

{!configured && <EmptyState message="CMS not configured. See ENV.md." />}
{error && <ErrorState message={error} />}
{configured && !error && posts.length === 0 && <EmptyState message="No posts yet." />}
{configured && !error && posts.length > 0 && <ContentList items={posts} />}
```

The `configured` guard lets the page render safely in template form (no env vars set) while showing live data in production.
