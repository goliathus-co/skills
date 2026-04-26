---
name: astro/api-routes-prerender-false
confidence: experimental
supersedes: []
applies_to: [astro, cloudflare, api, forms, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro 5 — API Routes with prerender = false

## Pattern

API routes in an Astro 5 static site opt into SSR by exporting `prerender = false`. This is required for any route that must run on each request (POST handlers, dynamic data, etc.).

```ts
// src/pages/api/contact.ts
import type { APIRoute } from 'astro';

export const prerender = false; // CF lessons L9 — Astro 5 opt-in SSR syntax

export const POST: APIRoute = async ({ request }) => {
  try {
    const data = await request.json();
    if (!data.email || !data.message) {
      return new Response(JSON.stringify({ error: 'Missing fields' }), {
        status: 400,
        headers: { 'Content-Type': 'application/json' },
      });
    }
    // handle valid data
    return new Response(JSON.stringify({ ok: true }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' },
    });
  } catch {
    return new Response(JSON.stringify({ error: 'Bad request' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json' },
    });
  }
};
```

## Rules

- `export const prerender = false` — required. Without it, Astro tries to prerender the route at build time and POST handlers fail.
- Return `new Response(JSON.stringify(...), { headers: { 'Content-Type': 'application/json' } })` — always set Content-Type explicitly.
- Always wrap `request.json()` in try/catch — malformed JSON throws, not returns.
- Validate at the boundary; return 400 for missing/invalid fields before any processing.
- SKELETON comment: mark stubs with `// SKELETON: real implementation via @goliathus/email (W2+)` so apply-time search finds them.

## Health endpoint contract

`/api/health` is a BREAKING-radius contract per CLAUDE.md §11. Path and response shape are fixed:

```ts
// src/pages/api/health.ts
export const prerender = false;
export const GET: APIRoute = () =>
  new Response(
    JSON.stringify({ status: 'ok', checked_at: new Date().toISOString(), version: '0.1.0' }),
    { status: 200, headers: { 'Content-Type': 'application/json' } }
  );
```

Uptime Kuma polls this. Changing the path or shape silently breaks monitoring 15 min later.

## CF adapter behaviour

With `@astrojs/cloudflare` v12 + `output: 'static'`, opting pages into SSR generates `_routes.json` that directs those paths to `_worker.js`. The build log shows `mode: "server"` — this is the adapter's internal mode, not an error. See skill `astro/static-with-opt-in-ssr.md`.

## Contact form skeleton pattern

Frontend form (vanilla `<form>` + inline `<script>`) posts JSON to `/api/contact`. Endpoint logs and returns `{ ok: true }` until `@goliathus/email` is wired (W2+). No DB writes; no email sends; skeleton is testable end-to-end via curl.
