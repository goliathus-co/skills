---
name: shared-secret-auth-pattern
description: M2M webhook authentication using a shared secret header — simpler than JWT for internal service-to-service calls where both sides are Goliathus-controlled
type: pattern
confidence: experimental
captured_during: heart-engine-T5
correlation: 5ae80183-6578-4446-b88b-1986466f6c19
---

# Shared Secret Auth Pattern (M2M Webhook)

## When to use

Machine-to-machine calls between Goliathus services where:
- Both caller and receiver are Goliathus-controlled (not external clients)
- The caller is a server-side process (no browser exposure)
- The secret can be stored in env vars on both sides

Not for: external webhook sources (Stripe, Clerk) — those use their own signature verification.

## Server implementation (Next.js App Router)

```typescript
// src/app/api/events/ingest/route.ts

import { NextRequest, NextResponse } from 'next/server';

const INGEST_SECRET = process.env.GOLIATHUS_INGEST_SECRET ?? '';

export const runtime = 'nodejs'; // supabase-js requires Node APIs

export async function POST(req: NextRequest): Promise<NextResponse> {
  // Auth check — first, before any body parsing
  const secret = req.headers.get('x-goliathus-ingest-secret');
  if (!secret || secret !== INGEST_SECRET) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Parse body
  let body: unknown;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'Invalid JSON' }, { status: 400 });
  }

  // Validate shape (Zod or manual)
  const parsed = ingestPayloadSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ error: 'Bad request', details: parsed.error.flatten() }, { status: 400 });
  }

  // Emit event
  // ...
  return NextResponse.json({ eventId: '...' }, { status: 200 });
}
```

## Clerk middleware exemption (critical)

The `/api/events/ingest` route is called by Clerk-less services (portal server, external scripts).
Clerk middleware wraps all `/api/` routes by default. Without exemption, Clerk's `auth.protect()` fires
before the route handler and returns 401 (wrong 401 — Clerk's, not the secret check).

```typescript
// src/middleware.ts
const isPublicRoute = createRouteMatcher([
  '/api/health',
  '/api/events/ingest',  // M2M — auth via x-goliathus-ingest-secret, not Clerk
  '/sign-in(.*)',
  '/sign-up(.*)',
]);
```

Without this exemption: unit tests pass (unit tests bypass middleware), but live calls from portal fail.
This is the hardest type of bug to catch — unit test green, integration test red.

## Secret generation

```bash
# Generate a 32-byte hex secret (64 chars)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Store in:
- `GOLIATHUS_INGEST_SECRET` on the receiving server (Vercel env var)
- Same key on any calling service (portal, scripts)
- Bitwarden for backup

## Caller implementation

```typescript
const res = await fetch(`${process.env.ADMIN_URL}/api/events/ingest`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-goliathus-ingest-secret': process.env.GOLIATHUS_INGEST_SECRET!,
  },
  body: JSON.stringify(payload),
});

if (!res.ok) throw new Error(`ingest failed: ${res.status}`);
```

## Test cases

```bash
# 200 — valid request
curl -X POST "$URL/api/events/ingest" \
  -H "Content-Type: application/json" \
  -H "x-goliathus-ingest-secret: $SECRET" \
  -d '{"verb":"intake.submitted","payload":{...},"context":{"correlationId":"...","actor":"portal"}}'

# 401 — missing secret
curl -X POST "$URL/api/events/ingest" \
  -H "Content-Type: application/json" \
  -d '{"verb":"intake.submitted","payload":{},"context":{}}'

# 400 — invalid payload shape
curl -X POST "$URL/api/events/ingest" \
  -H "Content-Type: application/json" \
  -H "x-goliathus-ingest-secret: $SECRET" \
  -d '{"not_a_valid":"shape"}'
```

## Related

- `events/dual-mode-studio-vs-webhook.md` — how the caller side is structured in @goliathus/events
- `nextjs/clerk-middleware-public-routes.md` — exempting M2M routes from Clerk middleware
