---
name: vercel-deploy-nextjs-clerk-supabase
description: Deploy a Next.js + Clerk + Supabase app to Vercel Pro — project setup, env vars, custom domain, common failure modes
type: pattern
confidence: experimental
captured_during: heart-engine-T-deploy
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Vercel Deploy — Next.js + Clerk + Supabase

## Stack

Next.js 15 App Router + Clerk auth + Supabase (service_role + anon) + shared ingest secret.
Deployment target: Vercel Pro (required for Clerk middleware — see trap below).

## Required env vars

Set in Vercel → Project → Settings → Environment Variables.
All vars needed in **Production** and **Preview** environments.

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<service-role-key>

# Clerk — MUST be from the Clerk instance bound to this domain (see Clerk section below)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_<...>
CLERK_SECRET_KEY=sk_live_<...>
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up

# Goliathus ingest (if this app hosts /api/events/ingest)
GOLIATHUS_INGEST_SECRET=<64-char hex>
```

**Clerk key type:** production domains require `pk_live_` / `sk_live_`. Using `pk_test_` on a
production custom domain causes `MIDDLEWARE_INVOCATION_FAILED` (see traps).

## Clerk per-domain architecture

Each primary domain gets its own Clerk instance. One instance per "application" (admin, portal, client).

| Domain | Clerk app | Key prefix |
|---|---|---|
| `admin.goliathus.co.uk` | goliathus-admin | `pk_live_` |
| `client.goliathus.co.uk` | goliathus-portal | `pk_live_` |

Setting up a new Clerk app:
1. Clerk dashboard → Create application
2. Set "Home URL" to the production domain (e.g. `https://admin.goliathus.co.uk`)
3. Copy publishable key + secret key → Vercel env vars

## Sign-in / sign-up pages

Without these routes, `auth.protect()` redirects to `/sign-in` which 404s.

```
src/app/sign-in/[[...sign-in]]/page.tsx
src/app/sign-up/[[...sign-up]]/page.tsx
```

```tsx
// sign-in/[[...sign-in]]/page.tsx
import { SignIn } from '@clerk/nextjs';
export default function SignInPage() {
  return (
    <main className="min-h-screen flex items-center justify-center">
      <SignIn />
    </main>
  );
}
```

Same pattern for sign-up with `<SignUp />`.

## Custom domain setup

1. Vercel → Project → Settings → Domains → Add domain
2. Vercel shows required DNS record (CNAME to `cname.vercel-dns.com` for subdomains)
3. Add the record in **the authoritative DNS provider** (not Cloudflare if Cloudflare isn't your NS)

**Critical:** check who actually serves DNS for your domain:
```bash
dig yourdomain.co.uk NS +short
# → ns1.namehero.com means DNS is at Namehero, NOT Cloudflare
```

Cloudflare DNS records have no effect unless Cloudflare is the authoritative NS.
Vercel TLS certificate provisions after DNS resolves (~5-15 min after record goes live).

## Supabase requires Node.js runtime

Any Next.js API route that imports `@supabase/supabase-js` must declare:

```typescript
export const runtime = 'nodejs';
```

Without it, Vercel may bundle for Edge runtime which lacks `node:net` / `node:tls`.
This applies to: `/api/events/ingest`, `/api/health`, any server action using Supabase.

See: `nextjs/supabase-requires-nodejs-runtime.md`

## Clerk middleware — exempt M2M routes

The `/api/events/ingest` endpoint is called by services without Clerk sessions.
Without middleware exemption, Clerk's `auth.protect()` fires before route handler → 401.

```typescript
// src/middleware.ts
const isPublicRoute = createRouteMatcher([
  '/api/health',
  '/api/events/ingest',  // M2M — auth via x-goliathus-ingest-secret
  '/sign-in(.*)',
  '/sign-up(.*)',
]);
```

See: `nextjs/clerk-middleware-public-routes.md`

## Deployment Protection on preview URLs

Vercel adds Deployment Protection to preview URLs by default.
External callers (curl, portal) hit a 401 from Vercel's own protection layer before the app.

**Fix:** use the production custom domain (`admin.goliathus.co.uk`) for all integration tests,
not the Vercel preview URL (`goliathus-admin-xyz.vercel.app`).

Or bypass via `x-vercel-protection-bypass` header with the bypass token from Vercel settings —
but only for CI/testing, never expose the bypass token publicly.

## Traps

**TRAP: `MIDDLEWARE_INVOCATION_FAILED`**
Symptom: All routes return 500 on first request; Vercel logs show middleware crash.
Cause 1: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` missing entirely.
Cause 2: Using `pk_test_` key on a production custom domain (wrong Clerk instance).
Fix: check Clerk dashboard that the key's "Home URL" matches the production domain.

**TRAP: DNS record added to wrong provider**
Symptom: `dig admin.yourdomain.co.uk CNAME +short` returns empty after 30 min.
Cause: Record added in Cloudflare but Cloudflare isn't the authoritative NS.
Fix: `dig yourdomain.co.uk NS +short` → find actual NS → add record there.

**TRAP: TLS provisioning delay**
Symptom: domain resolves in dig but browser shows `SSL_ERROR_SYSCALL`.
Cause: Vercel TLS cert not yet provisioned (can take 5-15 min after DNS resolves).
Fix: wait; verify Vercel dashboard shows cert as "Valid" under Domains.

**TRAP: First user has no role**
Symptom: app opens, user signs in, but no content shown.
Cause: app may have role-gated UI; first user has no role assigned in Clerk.
Fix: set up roles in Clerk dashboard → Users → assign role, or make root page public.

## Related

- `nextjs/supabase-requires-nodejs-runtime.md`
- `nextjs/clerk-middleware-public-routes.md`
- `webhook/shared-secret-auth-pattern.md`
