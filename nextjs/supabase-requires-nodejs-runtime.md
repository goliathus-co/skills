---
name: supabase-requires-nodejs-runtime
description: "@supabase/supabase-js uses Node.js net/tls APIs and is not edge-compatible — Next.js API routes calling Supabase must declare runtime = 'nodejs'"
type: trap
confidence: experimental
captured_during: heart-engine-T5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Supabase Requires Node.js Runtime in Next.js

## The trap

`@supabase/supabase-js` uses `node:net`, `node:tls`, and `node:http` internals. These APIs are not
available in the Edge runtime. If a Next.js API route uses `@supabase/supabase-js` and is deployed
to Vercel without explicitly declaring `runtime = 'nodejs'`, Vercel may attempt to bundle it for
the Edge runtime and fail — either at build time or at invocation.

Brief T5 originally specified `runtime = 'edge'` for `/api/events/ingest`. This was incorrect.
The route uses `createClient()` from `@supabase/supabase-js`, requiring Node.js runtime.

## Fix

```typescript
// Any Next.js App Router route that imports from @supabase/supabase-js:
export const runtime = 'nodejs';
```

Place this at the top of the route file, alongside any `dynamic` or `revalidate` exports.

## Where this applies

| File | Supabase import? | runtime declaration |
|---|---|---|
| `src/app/api/events/ingest/route.ts` | Yes | `export const runtime = 'nodejs'` |
| `src/app/api/health/route.ts` | Yes (checks DB) | `export const runtime = 'nodejs'` |
| Server components using Supabase | Yes | Implicit (RSC runs in Node) — no explicit declaration needed |
| `@goliathus/events` emit (direct mode) | Yes | Ensure calling route declares `nodejs` |

## Edge runtime — when it applies

Use `runtime = 'edge'` only for routes that:
- Do NOT import `@supabase/supabase-js` directly
- Do NOT use any Node.js-specific API

For `@goliathus/events` in webhook mode (emit via `fetch` to `/api/events/ingest`), edge runtime is
acceptable on the caller side because the fetch call doesn't use Supabase directly.

## Verification

If you see this error in Vercel logs or build output:
```
Error: Cannot find module 'node:net'
```
or:
```
DynamicServerError: Dynamic server usage: ...
```
...the runtime is wrong. Add `export const runtime = 'nodejs'`.

Locally with `pnpm dev`: Next.js uses Node.js by default so the error won't appear locally.
Always test with `pnpm build` to catch module compatibility issues before deploy.

## Brief amendment

Brief T5.5 originally specified `runtime = 'edge'`. Corrected by SENTINEL T5 Q1 acceptance.
`runtime = 'nodejs'` is now the Goliathus standard for any route touching Supabase.

## Related

- `webhook/shared-secret-auth-pattern.md` — the ingest route implementation
