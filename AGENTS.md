# PixelForge

Free, fully client-side image format converter, monetized via Google AdSense. A
deliberate single-purpose spin-out from the AnyConvert multi-tool site — one
keyword-matched domain per tool is the whole SEO strategy here. **This project
must never grow a second tool, a nav to other tools, or a "more tools"
section.** If another tool idea comes up, it gets its own standalone Astro
project under `~/iCloud/website/<name>`, not a page here.

## What it does

Convert images between JPG, PNG, WebP, GIF, BMP, AVIF, SVG, TIFF, and iPhone
HEIC/HEIF — entirely in the browser, no uploads, no accounts. Input formats
your browser can natively display are decoded via the native `<img>` /
`canvas` pipeline; HEIC/HEIF and TIFF (which browsers can't read natively) are
decoded on-device first via small helper libraries, then handed to the same
canvas pipeline. Output is PNG, JPEG, WebP, GIF, or BMP.

Deliberately NOT supported: raw camera formats (CR2/NEF/ARW/DNG) — real
demosaicing isn't feasible client-side; ICO output — rarely requested, skipped
for scope.

## Stack

- **AstroJS** (static output, `output: 'static'` in `astro.config.mjs`) +
  **Tailwind v4** (`@tailwindcss/vite` plugin, imported in
  `src/styles/global.css` via `@import 'tailwindcss'` + a `@theme` block for
  the custom color/font tokens).
- **Fonts**: "Fraunces" (display/headings) + "Inter" (body/UI), loaded via a
  Google Fonts `<link>` in `Layout.astro`'s `<head>` with `display=swap`.
- **libheif-js** (`libheif-js/wasm-bundle`) — decodes HEIC/HEIF client-side.
  NOT `heic2any` (unmaintained, fails on real modern iPhone HEIC files).
  Usage: `new libheif.HeifDecoder().decode(arrayBuffer)` →
  `image.display(imageData, cb)` → draw onto canvas → `canvas.toBlob(...,
  'image/png')`.
- **utif2** — decodes TIFF client-side (`UTIF.decode` → `UTIF.decodeImage` →
  `UTIF.toRGBA8` → `ImageData` → canvas). Pure JS, no WASM.
- **gifenc** — encodes GIF client-side (`canvas.toBlob` cannot produce
  `image/gif` in any browser). Quantize + applyPalette + GIFEncoder, ~5KB, no
  worker needed.
- BMP output has no library — hand-written (uncompressed 24-bit
  BITMAPINFOHEADER, ~50 lines), the format is simple enough not to need one.
- **@astrojs/sitemap** for `sitemap-index.xml` + `robots.txt` pointing at it.
- **Netlify** hosting. Deploy: `netlify deploy --prod --dir=dist` (auth via
  `~/.claude/credentials/netlify.env` → `NETLIFY_AUTH_TOKEN`).

This project intentionally installs ONLY what this one tool needs. It does
NOT include `@ffmpeg/*`, `@xenova/transformers`, `upscaler`, `qr-code-styling`,
`canvas-confetti`, or any other library from the parent AnyConvert
dependency list — those belong to other tools that live in their own
standalone projects.

## Design system (mandatory, do not deviate)

This is a premium redesign away from the generic white/cyan AI-tool look.
Cream/gold, editorial-serif headings — NOT another SaaS-blue clone.

- Background: `#FAF6EC` (warm cream)
- Card/surface: `#FFFFFF`, with a warm `#E8DFC8` border (never gray)
- Headings text: `#2B2013` (warm espresso brown, NOT pure black)
- Body text: `#5C4F3D` (warm brown-gray); muted/secondary text `#8A7E68`
- Accent/CTA/links/active states: `#C9982E` (warm gold), hover `#B8860B`
  (darker gold)
- Fonts: **Fraunces** for ALL headings (font-weight 600-700, slightly
  negative letter-spacing for a tightened premium look); **Inter** for body
  text, labels, buttons, and all interactive UI.
- Generous whitespace, soft shadows (never harsh), `rounded-xl` corners on
  cards/buttons, no gradients, no dark mode toggle (cream-only aesthetic is
  the whole point).

If you touch `src/styles/global.css` or `Layout.astro`, preserve these tokens
exactly. Do not reach for slate/gray/cyan — that's the old AnyConvert look
this project was deliberately split away from.

## Structure

- `src/layouts/Layout.astro` — shared shell: brand-only header (no nav to
  other tools by design), footer (About/Privacy/FAQ + copyright), SEO meta
  tags (title/description/canonical/OG/Twitter), Google Fonts link.
- `src/pages/index.astro` — the ONE tool. Canvas-based image conversion, same
  logic as AnyConvert's `image-converter.astro`, wrapped in the cream/gold
  design system. Contains the `escapeHtml()` helper — any user-controlled
  string (currently: file names) MUST be passed through it before being
  interpolated into `innerHTML`. Status text set via `textContent`, not
  `innerHTML`.
- `src/pages/about.astro`, `privacy.astro`, `faq.astro`, `404.astro` —
  required SEO/trust pages, linked from both the header nav and an in-page
  link row on the homepage (not just the footer), per AdSense eligibility
  requirements.
- `public/robots.txt`, `public/ads.txt` (placeholder — replace with the real
  AdSense publisher line once approved), `public/favicon.svg`.

## Security posture

Same audit finding as AnyConvert applies here: any user-controlled text
(filenames, etc.) rendered via `innerHTML` MUST go through the local
`escapeHtml()` helper first. Status messages and other strings we generate
ourselves are set via `textContent`/`className`, never `innerHTML`, so they
don't need escaping — but never relax that distinction.

## Dev / test

- `npm run dev` — dev server on port **4331** (see `.claude/launch.json`,
  config name `pixelforge-dev`). Chosen to not collide with AnyConvert's 4325.
- **iCloud sync hygiene**: `node_modules` is renamed to `node_modules.nosync`
  with a symlink (`node_modules -> node_modules.nosync`) so iCloud doesn't try
  to sync tens of thousands of tiny dependency files. `tsconfig.json`'s
  `exclude` array must list BOTH `"node_modules"` and `"node_modules.nosync"`
  — `tsc`/`astro check` does not treat `.nosync` as an implicit exclusion
  suffix, and without the explicit second entry `astro check` crashes trying
  to type-check into `node_modules.nosync`.
- File-input-driven tools can't be tested with a real OS file picker in
  headless Playwright/preview tooling. Test by constructing a `File` via
  `new File([blob], name, {type})`, wrapping it in a `DataTransfer`, setting
  `input.files = dt.files`, then dispatching a `change` event — then call
  `document.getElementById('convert-btn').click()` directly rather than a
  coordinate-based click tool, which can get intercepted by the Astro dev
  toolbar overlay.

## Deploy

Netlify site `pixelforge-520` (the plain `pixelforge` name was already taken
by another project in the account, so Netlify auto-suffixed it — actual
production URL is https://pixelforge-520.netlify.app; see `.netlify/state.json`
for the site ID).

```
source ~/.claude/credentials/netlify.env
export NETLIFY_AUTH_TOKEN=$NETLIFY_API_KEY
netlify deploy --prod --dir=dist
```

## Still needed before this earns anything (manual, Shreyas)

1. Buy/point a real keyword-matched domain.
2. Get real traffic (≥10 daily users per Google Analytics) before applying
   for AdSense.
3. Apply for AdSense, then replace the placeholder line in `public/ads.txt`
   with the real `pub-XXXXXXXXXXXXXXXX` line and add the AdSense script +
   Auto Ads.
4. Submit to Google Search Console + Bing Webmaster Tools.
