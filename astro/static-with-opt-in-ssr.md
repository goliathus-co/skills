---
name: astro/static-with-opt-in-ssr
confidence: experimental
supersedes: []
applies_to: [astro, cloudflare, static, ssr]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Astro 5 — Static output with opt-in SSR per page

## Pattern

Use `output: 'static'` in `astro.config.mjs`. All pages prerender by default.
To opt a single page into SSR (edge function), add to that page:

```ts
export const prerender = false;
```

This is the Astro 5 pattern. Do NOT use the deprecated `output: 'hybrid'` — it was removed in Astro 6 (CF lessons L9).

## Why static-first

ESSENCE/PRESENCE tier sites are content-heavy, traffic-heavy, budget-light.
Static output = CF Pages CDN serves HTML directly, no function invocation cost.
SSR opt-in is reserved for: contact form endpoints, API routes, dynamic personalisation.

## Config

```js
// astro.config.mjs
export default defineConfig({
  output: 'static',
  adapter: cloudflare(),
  // ...
});
```

## SSR opt-in example (API route)

```ts
// src/pages/api/contact.ts
export const prerender = false; // this route runs as CF Worker function

export const POST: APIRoute = async ({ request }) => { ... };
```

## Notes

- With `@astrojs/cloudflare` v12, `output: 'static'` still generates `_worker.js` in dist — this is expected. CF Pages uses it for SSR routes and routing.
- `_routes.json` in dist controls which paths go to the worker vs static CDN.
- Build logs show `mode: "server"` even for static output — this is the adapter's internal mode, not a misconfiguration.

## Trap

Do NOT confuse `output: 'static'` with "no adapter". Static output + CF adapter = CDN-first with SSR fallback. No adapter = local preview only, no CF deployment.
