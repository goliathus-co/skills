---
name: traps/astro5-cloudflare-adapter-peerdep
confidence: experimental
supersedes: []
applies_to: [astro, cloudflare, tailwind, pnpm]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Trap: @astrojs/cloudflare v12 peer-dep conflict + @astrojs/tailwind deprecation

## Trap 1 — CF adapter peer-dep mismatch

**Symptom:** `pnpm install` warns about peer dependency conflicts with `@astrojs/cloudflare@12.6.12` and Astro 5.x.

**Cause:** `@astrojs/cloudflare@12` declared peer of `astro@^5`. As Astro releases 5.x minor versions, semver ranges can conflict. CF lessons L11.

**Fix:** Add to `.npmrc`:
```
legacy-peer-deps=true
```

This is documented technical debt. The correct resolution is to upgrade the adapter, but v13 is for Workers (not Pages) — see Trap 3.

**Technical debt note:** `.npmrc legacy-peer-deps=true` should be tracked as `quality_debt` row once Heart Engine exists. Severity: `minor`.

## Trap 2 — @astrojs/tailwind v6 does not exist

**Symptom:** Brief specifies `@astrojs/tailwind ^6.0.0` but the package was deprecated at v5. pnpm install fails with "No matching version found."

**Cause:** For Tailwind CSS v4, the `@astrojs/tailwind` integration was replaced by the native Vite plugin approach (`@tailwindcss/vite`). The brief's version table has an error.

**Correct approach for Tailwind 4 + Astro 5:**

```bash
pnpm add tailwindcss @tailwindcss/vite
```

In `astro.config.mjs`:
```js
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  vite: {
    plugins: [tailwindcss()],
  },
});
```

No `integrations: [tailwind()]` call — Tailwind 4 is a Vite plugin, not an Astro integration.

**Token configuration** (T2): Use CSS `@theme` directive in `src/styles/global.css`, not `tailwind.config.ts` extend block. See `design/brand-dna-token-mapping.md`.

## Trap 3 — @astrojs/cloudflare v12 vs v13 deploy target

**Symptom:** Build succeeds but CF Pages deploy outputs "No functions dir found."

**Cause:** v13 generates Workers-compatible output (`dist/client/` + `dist/server/entry.mjs`). CF Pages expects `_worker.js` in dist root, which v12 generates.

**Rule:** Pin `@astrojs/cloudflare` to **exactly `12.6.12`** for CF Pages deployments. CF lessons L10.

```json
{ "@astrojs/cloudflare": "12.6.12" }
```

## Trap 4 — pnpm v10 build script approval

**Symptom:** `pnpm install` completes but esbuild/workerd are not built. Subsequent `pnpm build` fails with binary not found.

**Cause:** pnpm v10 introduced explicit build script approval. esbuild, workerd, and sharp require native binaries built via install scripts.

**Fix:** Add to `package.json`:
```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["esbuild", "workerd", "sharp"]
  }
}
```

This replaces the interactive `pnpm approve-builds` flow and is CI-friendly.
