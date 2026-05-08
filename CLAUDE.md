# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Wandering Light" — a prototype/mockup for a travel-and-family photography site with public galleries, brand-client private galleries, and an admin dashboard. **Static HTML + CSS + vanilla JS only** — no build system, no package manager, no framework, no backend. All forms (e.g. the "magic link" client login on `private.html`) are mocked client-side.

## Running locally

There are no build/test/lint commands. Open the `.html` files directly in a browser, or serve the directory statically (e.g. `python -m http.server` from the repo root) so that the relative `styles.css` link resolves correctly.

External resources are loaded from CDNs at runtime: Google Fonts (Cormorant Garamond, Inter, Caveat), `picsum.photos` for all placeholder imagery, and `cdn.jsdelivr.net/npm/chart.js@4.5.1` (admin dashboard only). The site requires internet access to render properly.

## Architecture

**Pages and how they relate:**
- `index.html` — homepage (hero, featured-galleries strip, story teaser, brands band).
- `gallery.html` — public gallery template: justified grid + lightbox with per-image licensing tiers.
- `private.html` — mock client-login (top half) + an in-page *preview* of what the unlocked client gallery looks like (bottom half, with selection counter and comment UI). Both halves live in one file.
- `story.html` — long-form "about" page. Contains a substantial `<style>` block of *page-specific* CSS in its `<head>`; the global `styles.css` does not own those styles.
- `admin_dashboard.html` — self-contained business dashboard with Chart.js. All CSS and JS are inlined in this file; it does **not** use `styles.css` or share its design tokens.

**Design system (`styles.css`):**
- CSS custom properties drive the entire warm/natural palette: `--sand`, `--cream`, `--cream-deep`, `--clay`, `--olive`, `--ink`, plus an 8px-based `--s1`…`--s8` spacing scale.
- Contrast-paired tokens matter: `--clay` is decorative-only (fails WCAG AA on cream); use `--clay-aa` (5.2:1) or `--clay-deep` (6.8:1) for text. Don't introduce new text colors without checking the same way.
- Type stack: `--serif` for display headings, `--sans` (Inter) for body, `--hand` (Caveat) for accent script.
- Global a11y baked in: `:focus-visible` outline using `--clay-aa`, and a `prefers-reduced-motion` block that kills animations/transitions globally.

**Gallery grid pattern** (used in `gallery.html` and the preview in `private.html`): a 12-column CSS grid `.justified` with `.tile` children that take `.span-3` through `.span-8`. Each span class also sets an `aspect-ratio`, so the layout shape is determined entirely by the span class on each tile. On screens ≤900px, all spans collapse to a 6-column reflow defined in `styles.css:517-519`.

**Lightbox pattern** (`gallery.html`): each `.tile` carries `data-img` (full-res URL), `data-title`, and `data-cap`. Clicking populates `#photo`/`#title`/`#cap` and adds `.open` to `#lightbox`. Closes on `Escape`, backdrop click, or `#close`.

**Image protection:** every page-level `<script>` independently re-installs `contextmenu` and `dragstart` listeners that block right-click/drag on imagery. This is duplicated per-page intentionally (no shared JS file). When adding a new HTML page, copy the same block — and make sure the `e.target` checks cover whatever element types display images on that page (`.tile`, `.img`, `.bg`, `.visual`, `IMG`, etc., depending on context).

**Mobile nav controller:** same per-page duplication. Each page's `<script>` block contains a small IIFE that toggles `aria-expanded` on `.nav-toggle` (and updates its `aria-label`), with handlers for click, link-click-to-close, Escape, and outside click. The CSS uses `.nav-toggle[aria-expanded="true"] + .nav` to drive the open/closed state — `aria-expanded` is the single source of truth, so the script does *not* add a separate "open" class. This keeps screen-reader state correct.

## Cloudinary image hosting — CRITICAL RULES

Cloud name: `deeck6k0n`. Images are served from `https://res.cloudinary.com/deeck6k0n/image/upload/`.

**`data-public-id` values must NEVER include a folder path prefix.**
Cloudinary folders (`roaming-lens/galleries/<gallery>/`) are dashboard display metadata only — they are NOT part of the public ID used in URLs. The working URL pattern is:

```
https://res.cloudinary.com/deeck6k0n/image/upload/<transform>/<publicI