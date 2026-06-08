# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

One-page B2B landing for Matías Laporta ("Asesoría Consultiva" — growth & sales infrastructure), a fully static Astro site deployed to GitHub Pages at `matiaslaporta.com`. All copy is in Spanish.

## Commands

```bash
npm install          # deps (one-time / after dep changes)
npm run dev          # dev server at http://localhost:4321/
npm run build        # static export to ./dist
npm run preview      # serve the production build locally
```

No linter, no tests — verify visually in the dev server.

**Deploy is automatic:** every push to `main` runs `.github/workflows/deploy.yml` (withastro/action → actions/deploy-pages). There is no manual deploy step. Watch a run with `gh run watch <id> --repo MatiasLaporta/matiaslaporta-web --exit-status`.

## Stack constraints

- **Astro 5**, `output: 'static'` — must stay statically exportable (no SSR / endpoints / runtime APIs).
- **Tailwind v3** via `@astrojs/tailwind` v5 (NOT v4). Palette keys live in `tailwind.config.mjs` and are used everywhere (`text-primary`, `bg-secondary`, …); renaming/removing a key breaks every component.
- **`lightningcss` is a REQUIRED devDependency** — `astro.config.mjs` sets `vite.build.cssMinify: 'lightningcss'`. It only runs during `build` (not `dev`), so a missing dep looks fine locally but **fails CI**. Keep it in `package.json`.
- **No client framework.** Interactivity is vanilla JS in plain `<script>` tags inside `.astro` components (Astro bundles and inlines them via Vite). Use `is:inline` only to load a raw CDN script.

## Brand palette (Tailwind keys)

| Key | Hex | Name / use |
|-----|-----|------|
| `primary` | `#1a2b56` | Azul — dark-blue section bgs, dark cards on light sections, dark text on light sections |
| `secondary` | `#f3cd74` | Amarillo — main brand accent (yellow CTAs, badges, headline highlights, glows) |
| `tertiary` | `#2f5972` | Celeste — muted slate-teal (used sparingly) |
| `background` | `#f5fbf9` | Blanco — off-white (NOT pure white; prefer `bg-background` over `bg-white`) |
| `dark` | `#1c2143` | Azul Marino — darkest navy (section bgs, footer text) |
| `accent` | `#5dcad1` | Cian — bright cyan; the `.btn-cyan` color and "destacar" pops |

The animated particle network draws nodes in hard-coded `#00E5FF` (brighter than `accent`) — that cyan is not a palette token.

## High-level architecture

Two pages, both wrapping `src/layouts/Layout.astro` and setting their own `<title>`/`description` (these OVERRIDE the defaults in `Layout.astro` — change page copy in the page, not the layout):

- **`src/pages/index.astro`** — the main landing. Assembles the component stack in this order:

  ```
  Header → Hero → PainPoints → About → Comparison → Services → TestimonialMarquee → CTA → Footer
  ```

- **`src/pages/tarjeta.astro`** — the `/tarjeta` digital business card (see its own section below). It does NOT use the component stack; it builds everything inline.

### Section backgrounds

When you change a section's background, flip every text/border color inside it — a leftover light `text-*` on a light bg makes copy invisible. Audit before finishing.

- **Dark (`bg-dark`):** Hero, Services, CTA — all three carry the particle network. Cards inside Services/CTA sit on `bg-primary`.
- **Light (`bg-background`):** PainPoints (but its 3 cards are `bg-primary` dark with white text), About, Comparison, TestimonialMarquee, Footer.
- **Header:** sticky; starts `bg-background`, JS toggles `.scrolled` (→ light grey + shadow) once `scrollY > 8`.

### Shared interactive mechanisms (`Layout.astro` + `global.css`)

1. **`.reveal`** — starts `opacity:0; translateY(40px)`; an `IntersectionObserver` in `Layout.astro` adds `.visible`. Put it on a section's top-level containers, not every child.
2. **`.hover-glow`** — lifts 5px + soft **yellow** glow on hover (cards, CTAs).
3. **`@keyframes marqueeLarge` + `.marquee-track`** — TestimonialMarquee duplicates its array (`[...t, ...t]`) in a `flex w-max` track animating `translateX(0)→-50%`. Duration is **90s desktop / 120s mobile**, set via the `.marquee-track` class (+ a `max-width:767px` override) — do NOT put the duration back inline, and do NOT deduplicate the array (the copy is what makes the loop seamless).

All respect `prefers-reduced-motion`.

### Particle network (4 copies, duplicated on purpose)

Hero, Services, CTA and the `/tarjeta` page each have their own `<canvas>` (`#network-canvas`, `#services-canvas`, `#cta-canvas`, `#tarjeta-canvas`) and a near-identical self-contained vanilla-JS `<script>` that draws ~70 `#00E5FF` nodes via `requestAnimationFrame`, linking nearby nodes and nodes near the cursor. Pattern per section: canvas is `absolute inset-0 z-0`, the section/`<main>` is `relative overflow-hidden`, and content sits in a sibling with `relative z-10`. Mouse coords come from `canvas.parentElement`, not `window`. The scripts are copy-pasted (one per component/page) intentionally — Astro module-scopes each `<script>`, so identical variable names don't collide. (Perf note: that's a separate rAF loop per visible page; if a page ever lags, pause off-screen canvases.) **If you change the node behavior, change every copy.**

### Buttons (`global.css` `@layer components`)

- `.btn-primary` — yellow (`bg-secondary`) + yellow `shadow-glow`.
- `.btn-cyan` — cyan (`bg-accent`) + cyan glow; for CTAs that should pop cyan.
- `.btn-ghost` — outlined light button.

Every CTA button (Header desktop + mobile, Hero, the 3 Services cards, CTA section) links to the Google Calendar booking URL **`https://calendar.app.google/2yt17PjZkAg5CcDE9`** with `target="_blank"`. This URL is hard-coded in several files — update all of them if it changes.

### Header

`Header.astro` has one `<script>` doing two things: the scroll color toggle (`.scrolled`) and the mobile hamburger menu (`#menu-toggle` toggles `#mobile-menu`, swaps open/close icons, updates `aria-expanded`). Desktop nav + CTA are `hidden md:*`; the hamburger + dropdown are `md:hidden`, and the dropdown holds the nav links plus the "Agendar Reunión" button.

### Comparison: dual layout

`Comparison.astro` renders the same `rows` array twice: a 3-column grid table (`hidden md:block`) for desktop, and a `md:hidden` stack that shows ALL of "Lo Tradicional" first, then ALL of "Nuestra Infraestructura" (so mobile doesn't interleave them). **Edit both layouts when changing rows.**

### Testimonials

`TestimonialMarquee.astro` renders quotes via `<Fragment set:html={t.quote} />` so quotes can embed `<strong>` for bold phrases. Each testimonial has an `avatar` field → `/logos/<company>.png`, shown as a 56px circle to the left of the name.

### Digital business card (`/tarjeta`)

`src/pages/tarjeta.astro` is a standalone `bg-dark` page — a Linktree-style card, NOT part of the landing's component stack. Structure: a centered `max-w-md` (`md:max-w-[30rem]`) column over its own particle canvas, holding `perfil.png` (circular, yellow ring) → `nombre.png` (the yellow ML name logo — used here instead of `Logo.png` so it reads on the dark bg) → tagline → buttons → footer.

- Data arrays drive the buttons, each carrying an inline SVG path rendered via `set:html` (`fill`/`stroke` chosen by an optional `fill` flag — filled brand glyphs vs. stroked line icons):
  - **`compact`** — small round `bg-primary` icon buttons in one row: `tel:` and `mailto:`, plus the icon-only **"Guardar Contacto"** `<button id="save-contact">` (the vCard trigger lives here, NOT in `links`).
  - **`links`** — large `.link-card` buttons, in order: *Escríbeme por WhatsApp* (`wa.me` with a pre-filled `text=`, flagged `highlight` → gets `.link-card-hl`), *Mi LinkedIn*, *LinkedIn de la Agencia*, *Ver Soluciones Comerciales* (→ `https://matiaslaporta.com/?ref=nfc`, the NFC tracking param), *Agendar Reunión*.
  - **`shareTargets`** — drives the **Share dropdown** in the footer (`#share-toggle` button + `#share-panel`): brand-colored round icons for Copiar (clipboard), WhatsApp, LinkedIn, Facebook, X, Email. `shareMessage` (with `*WhatsApp bold*` asterisks) is used for WhatsApp; `shareMessagePlain` (no asterisks) for X/Email; `shareTitle` is the email subject.
- **`.link-card`** + **`.link-card-hl`** are page-scoped `<style>` (NOT global.css): `.link-card` is transparent bg + `border-secondary` with a yellow glow + lift on `:hover`/`:focus-visible`/`:active`; `.link-card-hl` keeps a permanent translucent-yellow bg (intensifies on hover) — used only for the WhatsApp button. Distinct from the global `.hover-glow`.
- **GA4 tracking:** every `links` button (via a `track` field) and the share button carry an inline `onclick` firing `gtag('event','click_boton_tarjeta',{nombre_boton})`, guarded by `if(typeof gtag !== 'undefined')`. `gtag` only exists if GTM (see Deployment) loads a GA4 config tag.
- **Different booking URL:** this page uses `https://calendar.app.google/wK6AkvtZDWwxCNMy5` — NOT the landing's `2yt17PjZkAg5CcDE9`. They are independent; don't sync them.
- **"Guardar Contacto"** builds a vCard 3.0 string client-side (name, phone, email, web only) and downloads it as a Blob (`Matias-Laporta.vcf`) — no library, no file on disk.
- Lives at `matiaslaporta.com/tarjeta` (same Pages deploy as the landing — there is no separate subdomain).

### Section skeleton & accent pattern

New sections mirror: `<section id> → .container-tight → .reveal(badge + h2 + subtitle) → .reveal(grid/cards)`. `.container-tight` caps `max-w-6xl` with responsive padding. Badges use the pill style `inline-flex … rounded-full border border-secondary/40 bg-secondary/10 … text-secondary` with a leading glow dot.

Accent words use the inline "tech glow": `text-secondary [text-shadow:_0_0_20px_rgba(243,205,116,0.35)]` (use `0.45` at hero scale). The rgba MUST match `secondary` yellow. Do NOT use a `bg-secondary` highlight or `.gradient-text` (both were rejected). On LIGHT sections, headline accents instead use `text-primary underline decoration-secondary` — bright accent colors fail contrast on the off-white bg.

## Assets (`public/`, served from root)

- `Logo.png` — ML wordmark (Header + Footer); reference as `/Logo.png` (exact case — GitHub Pages is case-sensitive). Avoid spaces in asset filenames.
- `nombre.png` — yellow "ML" name logo used on the dark `/tarjeta` page (where `Logo.png` would not read).
- `perfil.png` — profile photo (used in both About and the `/tarjeta` header).
- `logos/*.png` — testimonial company avatars: `tvn`, `coach-confidencial`, `unab`, `somos-propiedades`, `walmart`, `ferreteria-santos`, `fia`, `vaco`.
- `favicon.svg`, `robots.txt`, `CNAME` (`matiaslaporta.com`). `robots.txt` references `sitemap-index.xml` but no sitemap integration is installed — if you add one, install `@astrojs/sitemap`.

## Deployment

- Repo **`MatiasLaporta/matiaslaporta-web`** (public). Pages source = GitHub Actions; custom domain `matiaslaporta.com`; HTTPS enforced.
- The owner has TWO GitHub accounts — the correct one is **MatiasLaporta** (not MatiasDigitals). `gh api user --jq .login` is the source of truth; `gh auth status` account labels have been misleading.
- Apex custom domain needs base path `/` — do NOT set `base` in `astro.config.mjs`. `public/CNAME` persists the domain across Actions deploys.
- **Google Tag Manager** (`GTM-WPWD39TW`) is in `Layout.astro` — the `is:inline` loader high in `<head>` + the `<noscript>` iframe right after `<body>` — so it covers BOTH pages. GTM only fires on the live site (not localhost) and only after the container is published; `gtag`/GA4 events depend on a GA4 config tag existing inside GTM.

## Environment

- Project root `d:\ML` (Windows / PowerShell). PowerShell here-strings for `git commit -m` are fragile here — prefer single-line `-m` messages.
- Astro pinned `^5.1.0`; ignore the "Astro 6.x available" dev notice unless explicitly migrating.
- The owner is non-technical — when giving terminal steps, state exactly WHERE to run them (e.g. "the PowerShell terminal in VSCode").
