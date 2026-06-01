# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

One-page B2B landing for Matías Laporta ("Arquitecto de Adquisición y Crecimiento"), built as a fully static site for GitHub Pages deployment (target: `matiaslaporta.com`).

## Commands

```bash
npm install          # install dependencies (one-time / after dep changes)
npm run dev          # local dev server at http://localhost:4321/
npm run build        # static export to ./dist (deployable to GitHub Pages)
npm run preview      # serve the production build locally for verification
```

No linter, no test runner. Verify visually with the dev server.

## Stack constraints

- **Astro 5** with `output: 'static'` in `astro.config.mjs`. The site MUST stay statically exportable — do not introduce server endpoints, SSR adapters, or runtime APIs.
- **Tailwind v3** via `@astrojs/tailwind` v5 integration (NOT Tailwind v4). The brand palette lives in `tailwind.config.mjs` and is consumed everywhere as `text-primary`, `bg-secondary`, etc. Renaming or removing keys will break every component.
- **HTML compression + `lightningcss`** CSS minification are enabled in production builds.
- **No client-side framework** (no React/Vue/Svelte). Interactivity is vanilla JS inside `<script>` tags inside `.astro` components. Astro bundles non-`is:inline` scripts via Vite automatically — leave them as plain `<script>` unless a CDN must be loaded raw (then use `is:inline`).

## Brand palette (Tailwind keys)

| Key | Hex | Typical use |
|-----|-----|------|
| `primary` | `#1a2b56` | "Azul" — dark/blue section backgrounds, dark text on light sections |
| `secondary` | `#f3cd74` | "Amarillo" — warm yellow, primary brand accent (CTAs, badges, headline highlights) |
| `tertiary` | `#2f5972` | "Celeste" — muted slate-teal (header bar, secondary surfaces) |
| `background` | `#f5fbf9` | "Blanco" — off-white light surfaces (NOT pure white; use `bg-background`, not `bg-white`) |
| `dark` | `#1c2143` | "Azul Marino" — layered dark surfaces (cards on `bg-primary`) |
| `accent` | `#5dcad1` | "Cian" — bright cyan reserved for "destacar" pop highlights |

## High-level architecture

`src/pages/index.astro` is the ONLY page. It imports every component from `src/components/` and assembles them through `src/layouts/Layout.astro` in this strict order:

```
Header → Hero → PainPoints → About → Comparison → Services → TestimonialMarquee → CTA → Footer
```

### Section alternation (load-bearing)

Sections alternate dark/light to create visual rhythm. Preserve this when editing:

- **Dark** (`bg-primary` / `bg-dark`): Hero, Services, CTA, Footer
- **Light** (`bg-background` / `bg-white`): PainPoints, About, Comparison, TestimonialMarquee

When you change a section's background, also flip every text/border color inside it (light sections use `text-primary` + `text-gray-600`; dark sections use `text-white` + `text-gray-300` + `text-secondary` accents). A leftover `text-white` on a light section makes copy invisible — audit before considering an edit done.

### Three cross-cutting interactive mechanisms (defined in `Layout.astro` + `global.css`)

1. **`.reveal` scroll-in animation.** Any element with class `reveal` starts at `opacity:0; translateY(40px)` and gets `.visible` added by an `IntersectionObserver` initialized in `Layout.astro`. Apply `reveal` to top-level containers of new sections, not to every child.
2. **`.hover-glow` card effect.** CSS-only class that lifts the element 5px and adds a cyan box-shadow on hover. Used on stat cards, service cards, pain-point cards, primary CTAs.
3. **`@keyframes marqueeLarge`** (defined in `global.css`). Used exclusively by `TestimonialMarquee.astro`, which duplicates the testimonial array (`[...testimonials, ...testimonials]`) inside a `flex w-max` track animating `translateX(0)` → `translateX(-50%)` over 120s. The duplication is what makes the loop seamless — do not deduplicate.

All three respect `prefers-reduced-motion: reduce` via the media query at the bottom of `global.css`.

### Hero particle network

`Hero.astro` contains a `<canvas id="network-canvas">` and a self-contained vanilla JS `<script>` that draws ~70 cyan nodes, animates them with `requestAnimationFrame`, and connects nearby nodes (and nodes near the cursor) with translucent cyan lines. The canvas is positioned `absolute inset-0 z-0` inside the Hero `relative overflow-hidden bg-primary` section; the text content sits in a sibling `container-tight relative z-10`. Mouse coordinates come from `canvas.parentElement` (the section), not `window`. Keep this scoped to the Hero — do not lift the script into `Layout.astro`.

### Component layout convention

Every section follows the same skeleton — when adding a new section, mirror it:

```
<section id="..." class="<bg> py-24 lg:py-32">
  <div class="container-tight">
    <div class="reveal text-center">
      <span class="... rounded-full ... uppercase tracking-wider">BADGE</span>
      <h2 class="font-display ...">Title with <span class="text-secondary">accent</span></h2>
      <p class="...">Subtitle</p>
    </div>
    <!-- grid / table / cards, also wrapped in a .reveal container -->
  </div>
</section>
```

The `.container-tight` helper (defined in `global.css`) caps width at `max-w-6xl` with responsive horizontal padding.

### Typography accent pattern

Words highlighted in the accent color use the inline class `text-secondary [text-shadow:_0_0_20px_rgba(243,205,116,0.35)]` (or `0.45` opacity for hero-scale text). This is the canonical "tech glow" treatment. The glow rgba must match `secondary` (yellow `243,205,116`). Do NOT swap it for `bg-secondary` highlight or the `.gradient-text` class — those were rejected in earlier iterations.

## Deployment context

Site target is `matiaslaporta.com` via GitHub Pages. `site: 'https://matiaslaporta.com'` is set in `astro.config.mjs` (drives sitemap / canonical URLs). `public/` contains `favicon.svg` and `robots.txt`; the latter references `sitemap-index.xml` even though no sitemap integration is installed yet — if you add one, install `@astrojs/sitemap` and register it.

## Working environment notes

- Project root: `d:\ML` (Windows, PowerShell). When invoking dev commands prefer PowerShell, but Bash works for file ops.
- `package.json` pins Astro `^5.1.0`; the dev console may show "New version of Astro available: 6.x" — ignore unless explicitly migrating.
