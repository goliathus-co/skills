---
name: clerk-protect-rewrite-curl-artifact
description: auth.protect() returns protect-rewrite 404 for plain curl — this is a Clerk isPageRequest artifact, not a bug. Browsers get correct 307 redirect. How to diagnose, why it happens, and what NEXT_PUBLIC_CLERK_SIGN_IN_URL actually does.
type: trap
confidence: tested
captured_during: intake-w3-T5.5
correlation: 29c5f4b9-7ed6-4d72-a20a-a078b641855e
---

# Clerk `protect-rewrite` — curl artifact, not a middleware bug

## The trap

```bash
curl -sI https://admin.goliathus.co.uk/inbox
# HTTP/2 404
# x-clerk-auth-reason: protect-rewrite, session-token-and-uat-missing
```

Looks broken. It is not. Browser gets `307 → /sign-in`. The middleware is correct.

**Spent 3 broken middleware commits chasing this.** Don't repeat.

## Why it happens

Clerk's `createProtect` (in `@clerk/nextjs/dist/cjs/server/protect.js`) has a branch for
unauthenticated users:

```javascript
const handleUnauthenticated = () => {
  if (unauthenticatedUrl) {
    return redirect(unauthenticatedUrl);   // explicit override
  }
  if (isPageRequest(request)) {
    return redirectToSignIn();             // browser → redirect to /sign-in
  }
  return notFound();                       // API/curl → 404 rewrite
};
```

`isPageRequest` checks:
- `Sec-Fetch-Dest: document` or `iframe`
- `Accept` includes `text/html`
- App Router internal navigation headers
- Pages Router `__nextjs_data` header

`curl` by default sends **none of these**. Clerk sees an API-like request, calls `notFound()`,
Next.js middleware catches it and issues a rewrite to a fake URL with `x-clerk-auth-reason:
protect-rewrite`. This is intentional — API routes should return 404/401, not redirect to a
login page.

## Correct test command

```bash
# Always test browser-auth routes with browser-like headers
curl -sI https://admin.goliathus.co.uk/inbox \
  -H "Sec-Fetch-Dest: document" \
  -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"

# Expected (correct):
# HTTP/2 307
# location: /sign-in
# x-clerk-auth-reason: session-token-and-uat-missing
```

Without browser headers: `protect-rewrite` (expected for API clients)
With browser headers: `307 → /sign-in` (correct for browsers)

## What NEXT_PUBLIC_CLERK_SIGN_IN_URL actually does

This env var is **NOT** what `auth.protect()` uses to redirect. Common misconception.

`auth.protect()` gets the sign-in URL from `requestState.signInUrl`, which is resolved
by the Clerk backend's `authenticateRequest()` from the Clerk instance configuration
(Dashboard → Instance settings → Paths). No env var needed for redirect to work.

`NEXT_PUBLIC_CLERK_SIGN_IN_URL` is used by:
- Clerk's `<SignIn>` and `<SignUp>` client components (where to mount them)
- `redirectToSignIn()` / `redirectToSignUp()` helper functions
- Clerk's JavaScript SDK for client-side navigation

If you see `protect-rewrite` in a browser (not curl), THEN check:
1. Is the route in `isPublicRoute`? If yes, Clerk doesn't protect it.
2. Is `Sec-Fetch-Dest: document` present in the actual browser request? Use DevTools to check.
3. Is `requestState.signInUrl` configured in Clerk Dashboard?

## The 3 approaches that don't work (avoid)

```typescript
// ❌ clerkMiddleware options signInUrl — does NOT affect auth.protect() redirect
export default clerkMiddleware(handler, { signInUrl: '/sign-in' });

// ❌ auth.protect({ unauthenticatedUrl: '/sign-in' }) — crashes with relative path
// (Clerk's redirect() adapter calls NextResponse.redirect which needs absolute URL)
await auth.protect({ unauthenticatedUrl: '/sign-in' });

// ❌ manual await auth() + NextResponse.redirect — may conflict with Clerk internals
const { userId } = await auth();
if (!userId) return NextResponse.redirect(new URL('/sign-in', request.url));
```

## What to use

```typescript
// ✓ Just auth.protect() — works correctly for browsers without any extra config
export default clerkMiddleware(async (auth, request) => {
  if (!isPublicRoute(request)) {
    await auth.protect();
  }
});
```

Clerk resolves the sign-in URL from its backend instance config. No env var override needed
for the standard goliathus-admin setup.

## Related

- `nextjs/clerk-middleware-public-routes.md` — which routes to exempt from auth.protect()
