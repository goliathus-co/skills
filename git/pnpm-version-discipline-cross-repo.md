---
name: git/pnpm-version-discipline-cross-repo
confidence: experimental
supersedes: []
applies_to: [pnpm, package-manager, cross-repo, ci]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# pnpm Version Discipline — Cross-Repo Baseline

## Rule

All Goliathus repos use **pnpm 10.x**. Pin exact version in `packageManager` field.

```json
{
  "packageManager": "pnpm@10.33.2"
}
```

**Why:** Decided at W1 with first template (template-astro-content). Three repos in and version drift creates migration debt — different lockfile formats, different `onlyBuiltDependencies` behavior, different peer-dep resolution. One standard eliminates the class of problem.

## pnpm v10 requirements every new repo must include

```json
{
  "pnpm": {
    "onlyBuiltDependencies": ["esbuild", "workerd", "sharp"]
  }
}
```

**Why:** pnpm v10 requires explicit approval for native build scripts. Without this, `pnpm install` succeeds but esbuild/workerd/sharp binaries are not built → `pnpm build` fails with confusing binary-not-found errors. `onlyBuiltDependencies` is CI-friendly (no interactive prompt). Add more packages if the project requires other native dependencies.

## CI configuration

In GitHub Actions, do NOT read `packageManager` field — it can fail silently if the field is malformed:

```yaml
- uses: pnpm/action-setup@v4
  with:
    version: 10   # explicit, not inferred from packageManager field
```

**Why:** `pnpm/action-setup` can fall back to 8.x silently when `packageManager` field is absent or malformed. Explicit `version: 10` makes failure loud and obvious.

## Cross-repo coupling

When adding a new Goliathus repo:
1. Set `packageManager: pnpm@<current 10.x>` in package.json
2. Add `pnpm.onlyBuiltDependencies` with native deps for that stack
3. CI workflow: `version: 10` in pnpm/action-setup
4. Verify with `pnpm --version` — must print 10.x

Do not let a repo go live on pnpm 9.x — lockfile format differences cause `pnpm install --frozen-lockfile` failures in CI for any developer on pnpm 10.
