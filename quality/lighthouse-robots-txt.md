---
name: lighthouse-robots-txt
description: Static sites (Astro/CF Pages) need an explicit public/robots.txt — missing file causes Lighthouse SEO audit failure, dropping SEO score from 100 to ~91. One-line fix.
type: trap
confidence: tested
captured_during: intake-w3-T6
correlation: 29c5f4b9-7ed6-4d72-a20a-a078b641855e
---

# Lighthouse SEO — robots.txt required for static sites

## The trap

Lighthouse SEO audit includes `robots-txt` check. Static site generators (Astro, plain HTML)
do NOT auto-generate `robots.txt`. Missing file = audit fails = SEO score drops ~9 points.

Observed: intake-form (Astro + CF Pages) scored SEO=91 desktop without robots.txt, SEO=100 after.

The `robots-txt` audit is weighted ~10% of the SEO category score in Lighthouse.

## Fix

Create `public/robots.txt` (Astro, static HTML) or `static/robots.txt` (other frameworks):

```
User-agent: *
Allow: /
```

This is the correct default for public-facing sites. Crawlers allowed everywhere.

## When to block crawlers instead

For staging / preview environments that should not be indexed:

```
User-agent: *
Disallow: /
```

For CF Pages preview URLs (`*.pages.dev` subdomain), consider blocking:
```
# In public/robots.txt — only served from brief.goliathus.co.uk, not from pages.dev
User-agent: *
Allow: /
```

Note: Cloudflare Pages custom domain and `*.pages.dev` URL serve the same `robots.txt`.
If you want preview deploys to be noindex, use a `<meta name="robots" content="noindex">`
in the HTML head instead (conditional on `window.location.hostname`), not in robots.txt.

## Verification

```bash
# Confirm file is live
curl -s https://your-domain.com/robots.txt

# Quick Lighthouse SEO check without full run
npx lighthouse https://your-domain.com/ \
  --only-categories=seo \
  --preset=desktop \
  --output=json --output-path=/tmp/seo.json \
  --chrome-flags="--headless=new --no-sandbox" \
  --quiet && \
cat /tmp/seo.json | python3 -c "
import sys,json; d=json.load(sys.stdin)
a=d['audits']['robots-txt']
print(f'robots-txt: score={a[\"score\"]} — {a.get(\"displayValue\",\"\")}')
print(f'SEO total: {round(d[\"categories\"][\"seo\"][\"score\"]*100)}')
"
```

## Applies to

- All Astro repos deployed to CF Pages (intake-form, template-astro-content, template-astro-minimal, client repos)
- Any static site without a framework that auto-generates robots.txt (Next.js does auto-generate one)
- Next.js on Vercel auto-generates `robots.txt` from `app/robots.ts` — no manual file needed there

## Related

- Next.js App Router auto-generates robots.txt from `app/robots.ts` — no manual step needed for goliathus-admin/portal
- For Astro: `public/robots.txt` is the correct location; it copies to `dist/robots.txt` at build
