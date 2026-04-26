---
name: clerk-middleware-public-routes
description: Clerk middleware catches all /api/ routes by default — M2M and health routes need explicit exemption in isPublicRoute or Clerk returns 401 before the handler runs
type: trap
confidence: experimental
captured_during: heart-engine-T5-M5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Clerk Middleware Public Routes Exemption

## The trap

Clerk middleware's default matcher catches `/(api|trpc)(.*)`. Every route under `/api/` goes through
`auth.protect()` unless it's in `isPublicRoute`. This includes:
- Health check endpoints (`/api/health`) — polled by Uptime Kuma, no Clerk session
- M2M webhook endpoints (`/api/events/ingest`) — called by portal/scripts with shared-secret, no Clerk session

Without exemption, Clerk returns its own `401 Unauthorized` before the route handler runs.
Unit tests DO NOT catch this — unit tests import the handler directly and bypass middleware.

## Fix

```typescript
// src/middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isPublicRoute = createRouteMatcher([
  '/api/health',
  '/api/events/ingest',  // M2M — auth via x-goliathus-ingest-secret, not Clerk session
  '/sign-in(.*)',
  '/sign-up(.*)',
]);

export default clerkMiddleware(async (auth, request) => {
  if (isPublicRoute(request)) return; // pass through without auth check
  await auth.protect();               // require Clerk session for everything else
});

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
};
```

## Which routes to exempt

| Route | Reason |
|---|---|
| `/api/health` | Polled by monitoring (Uptime Kuma); no Clerk session |
| `/api/events/ingest` | M2M webhook from portal/scripts; auth via `x-goliathus-ingest-secret` |
| `/sign-in(.*)`, `/sign-up(.*)` | Clerk auth pages — must be public |

Do NOT exempt admin UI routes, API routes that return user data, or any route that should require login.

## Verification

Deploy to Vercel preview (or test locally with `pnpm dev`):

```bash
# Should return 200 (not 401)
curl -I https://your-preview.vercel.app/api/health

# Should return 401 from your handler (not Clerk's generic 401)
curl -X POST https://your-preview.vercel.app/api/events/ingest \
  -H "Content-Type: application/json" -d '{}'
# Response: {"error":"Unauthorized"} — your handler's 401
# Wrong response: Clerk's HTML error page or Clerk JSON error
```

The difference: your handler's 401 has the shape you defined. Clerk's 401 is a different shape and
often includes `clerkError: true` in the response body.

## Why unit tests don't catch this

Unit tests import the route handler function directly:
```typescript
import { POST } from '@/app/api/events/ingest/route';
const res = await POST(mockRequest);
```
This bypasses Next.js middleware entirely. The middleware runs only in actual HTTP requests.
Always test M2M routes with a live HTTP call against a deployed (or dev server) instance.

## Related

- `webhook/shared-secret-auth-pattern.md` — the full M2M auth implementation
