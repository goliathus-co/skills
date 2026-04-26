---
name: git/github-actions-pnpm-cache
confidence: experimental
supersedes: []
applies_to: [github-actions, pnpm, ci, template-astro-content]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# GitHub Actions — pnpm Setup + Cache

## Pattern

```yaml
- uses: pnpm/action-setup@v4
  # No version: field — reads pnpm version from packageManager in package.json
  # e.g. "packageManager": "pnpm@10.33.2"

- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: pnpm        # built-in pnpm cache via setup-node

- run: pnpm install --frozen-lockfile
```

## Key rules

**Omit `version:` in pnpm/action-setup** — the action reads the `packageManager` field from `package.json`. Setting `version: 9` when the project uses pnpm@10 causes lockfile format incompatibility and `pnpm install --frozen-lockfile` will fail.

**Use `cache: pnpm` in setup-node** — not `cache: npm`. This activates pnpm's store cache between runs, cutting install time from ~60s to ~10s on cache hit.

**`--frozen-lockfile`** — always. Without it, CI may update the lockfile silently, creating drift between local and CI installs. Fails loudly if lockfile is out of date.

## Trap: package in node_modules but not in package.json

If a package was installed locally (`pnpm add <pkg>`) but package.json + pnpm-lock.yaml were not committed together with the code that imports it, CI will fail with `Rollup failed to resolve import`. Local build works because node_modules has the package from the prior install session.

Fix: always commit package.json + pnpm-lock.yaml in the same commit as the code that introduces the import.
