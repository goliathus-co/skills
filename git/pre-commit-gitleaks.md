---
name: git/pre-commit-gitleaks
confidence: experimental
supersedes: []
applies_to: [git, security, pre-commit, husky]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Pre-commit: gitleaks hard-fail + lint-staged

## Pattern

```bash
# .husky/pre-commit
#!/usr/bin/env sh

if ! command -v gitleaks >/dev/null 2>&1; then
  echo "ERROR: gitleaks not installed. See README.md installation step."
  echo "Pre-commit hook cannot run without it. Install with: brew install gitleaks (or equivalent)."
  exit 1
fi

pnpm lint-staged
gitleaks protect --staged --no-banner --redact || exit 1
```

## Why hard-fail (not silent skip)

Silent skip = invariant lost. If the hook continues when gitleaks is absent, new developers can accidentally commit secrets during their first session — before they've read the README. Hard-fail forces resolution before first commit. CF lessons: CLAUDE.md §3.5.

## Installation reminder (postinstall)

In `package.json`:
```json
{
  "scripts": {
    "postinstall": "command -v gitleaks >/dev/null 2>&1 || echo 'WARN: gitleaks missing — pre-commit hook will hard-fail until installed (see README)'"
  }
}
```

This surfaces the warning at `pnpm install` time, before the developer hits the hook. Not a fix — just early signal.

## Setup

```bash
brew install gitleaks  # macOS
# or: https://github.com/gitleaks/gitleaks/releases (Linux/Windows)
```

## lint-staged config

```json
{
  "lint-staged": {
    "*.{ts,tsx,astro,js,json}": ["prettier --write"],
    "*.{ts,tsx,astro}": ["eslint --fix"]
  }
}
```

## pnpm v10 note

`"prepare": "husky"` in scripts runs automatically on `pnpm install`. Create `.husky/pre-commit` manually — husky v9 does not auto-create hook files.

## gitleaks config

Stub `.gitleaks.toml` at repo root catches: Stripe live keys, AWS access keys, Clerk secrets, generic high-entropy assignments. Allowlist: `example.com`, `REPLACE_ME`, `placeholder`.
