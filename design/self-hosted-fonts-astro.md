---
name: design/self-hosted-fonts-astro
confidence: experimental
supersedes: []
applies_to: [design, fonts, astro, performance, gdpr]
captured_during: template-astro-content-w1
captured_at: 2026-04-26
---

# Self-Hosted Fonts in Astro + Tailwind 4

## Why self-host

- Google Fonts sends IP to Google servers — GDPR concern
- CDN dependency introduces availability risk
- Self-hosted = no third-party connection on page load
- Matches Goliathus "independence from providers" principle (Foundations Inventory §1.4)

## File strategy

- Format: woff2 only (modern browser coverage ≥ 96%)
- Subset: Latin only (Goliathus default; extend at apply-time for client locales)
- Location: `public/fonts/` — served as static assets at `/fonts/`
- Naming: `<family>-<subset>-<weight>-<style>.woff2` (fontsource convention)

## Getting font files

Use @fontsource packages as source, then remove them:

```bash
pnpm add --save-dev @fontsource-variable/fraunces @fontsource/manrope @fontsource/jetbrains-mono

mkdir -p public/fonts

# Fraunces — full variable font (all axes: wght, opsz, SOFT, WONK)
cp node_modules/@fontsource-variable/fraunces/files/fraunces-latin-full-normal.woff2 public/fonts/

# Manrope — individual static weights
for weight in 300 400 500 600; do
  cp node_modules/@fontsource/manrope/files/manrope-latin-${weight}-normal.woff2 public/fonts/
done

# JetBrains Mono — technical/code weights
for weight in 400 500; do
  cp node_modules/@fontsource/jetbrains-mono/files/jetbrains-mono-latin-${weight}-normal.woff2 public/fonts/
done

pnpm remove @fontsource-variable/fraunces @fontsource/manrope @fontsource/jetbrains-mono
```

Commit the woff2 files to git. They are not large binaries — each Latin subset woff2 is 30-80KB.

## @font-face declarations

Place in `src/styles/global.css` inside `@layer base {}`:

```css
@layer base {
  /* Fraunces variable — wght axis 100-900, opsz axis enabled in browser */
  @font-face {
    font-family: 'Fraunces';
    src: url('/fonts/fraunces-latin-full-normal.woff2') format('woff2');
    font-weight: 100 900;
    font-style: normal;
    font-display: swap;
  }

  /* Manrope static — declare each weight separately */
  @font-face {
    font-family: 'Manrope';
    src: url('/fonts/manrope-latin-300-normal.woff2') format('woff2');
    font-weight: 300;
    font-style: normal;
    font-display: swap;
  }
  /* repeat for 400, 500, 600 */
}
```

## Tailwind 4 integration

Font families are registered via `@theme inline` in `global.css`:

```css
@theme inline {
  --font-display: 'Fraunces', Georgia, serif;
  --font-body: 'Manrope', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', ui-monospace, monospace;
}
```

This makes `--font-display` available as a CSS variable in `:root` AND generates `font-display`, `font-body`, `font-mono` Tailwind utility classes.

## font-display: swap

Use `swap` (not `optional` or `block`):
- `swap` = immediately renders with fallback, then swaps when font loads
- `optional` = skips swap if font takes > 100ms (too aggressive for display fonts)
- `block` = invisible text during load — FOIT, bad UX

For the Goliathus editorial aesthetic, FOUT (flash of unstyled text) is acceptable. The fallback fonts (Georgia, system-ui) are similar enough in weight and rendering.

## Subsetting

Latin subset covers: A-Z, a-z, 0-9, basic punctuation, European Latin characters (ä, ö, é, ñ etc.)

For clients with other language requirements:
- Latin Extended: add `fraunces-latin-ext-*` files
- Cyrillic: check @fontsource for availability
- Document in client's brief what subset was applied

## Build size

7 font files for Goliathus default: ~280KB total on disk. In CF Pages transfer, woff2 is served with proper caching headers (immutable, 1 year). First visit: ~280KB extra. Subsequent: 0 (cached).

This is acceptable for the Goliathus ESSENCE/PRESENCE tier — design integrity justifies the font load.
