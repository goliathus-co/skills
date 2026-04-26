---
name: cloudflare/cloudflare-pages-deploy
confidence: experimental
supersedes: []
applies_to: [cloudflare, astro, ci, github-actions, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Cloudflare Pages — Deploy via Wrangler from CI

## Pattern

Use explicit `wrangler pages deploy` from CI. Do NOT use CF Pages dashboard "Connect to GitHub" auto-deploy.

```yaml
# .github/workflows/deploy.yml
- name: Deploy via wrangler
  uses: cloudflare/wrangler-action@v3
  with:
    apiToken: ${{ secrets.CF_API_TOKEN }}
    accountId: ${{ secrets.CF_ACCOUNT_ID }}
    command: pages deploy dist --project-name=<project-slug>
```

## Why wrangler deploy, not auto-deploy (CF lessons L48)

CF Pages auto-deploy (dashboard → connect GitHub) fails on Astro SSR builds because:
- It runs its own build process that doesn't respect pnpm lockfile
- It doesn't support custom env var injection at build time
- SSR routes (`prerender = false`) require `_worker.js` which the auto-build mangles

Explicit `wrangler pages deploy dist` targets the local `dist/` directory after a clean `pnpm build` in CI.

## Secrets required

GitHub repo secrets:
- `CF_API_TOKEN` — token with `Account → Cloudflare Pages → Edit` + `Account → Account Settings → Read`
- `CF_ACCOUNT_ID` — Cloudflare account ID (visible in CF dashboard right rail)

Store both in Bitwarden: `Goliathus / <project> / CF_API_TOKEN`.

## CF Pages project setup

Create the project once before first deploy:

```bash
pnpm exec wrangler pages project create <project-slug> --production-branch main
```

If the project doesn't exist, `wrangler pages deploy` silently succeeds on some versions but fails on others. Create explicitly.

## wrangler.toml config for CF Pages

```toml
name = "<project-slug>"
compatibility_date = "2025-09-01"
compatibility_flags = ["nodejs_compat"]
pages_build_output_dir = "dist"
```

`pages_build_output_dir` is required for `nodejs_compat` to apply (CF lessons L5).
`name` should match `--project-name` in the deploy command.

## Preview URL and SEO

CF Pages adds `x-robots-tag: noindex` to all `*.pages.dev` preview deployments. This is platform behaviour — Lighthouse SEO score will be low (~58) on preview URLs. Measure SEO only on production domain with custom CNAME.

## Apply-time instruction

When applying this template to a client project:
1. Create CF Pages project in **client's** CF account (not Goliathus account)
2. Replace `--project-name=template-astro-content-preview` with client's project slug
3. Set CF_API_TOKEN + CF_ACCOUNT_ID as GitHub secrets in client's repo
4. Do NOT enable CF Pages dashboard GitHub auto-deploy
