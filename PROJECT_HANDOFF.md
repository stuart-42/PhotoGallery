# Roaming Lens — Project Handoff

## Overview
Photography website for Stuart (content creator / commercial photographer). Sells image licences, has public galleries, private client galleries, and an admin dashboard. **Static HTML + CSS + vanilla JS only** — no build system, no framework, no backend yet.

**Live hosting:** Vercel (auto-deploy from GitHub push)  
**Image hosting:** Cloudinary (cloud name: `deeck6k0n`)  
**Phase 2 planned:** Supabase (auth + database), EmailJS (enquiry forms)

---

## Site Map

| File | Purpose |
|------|---------|
| `index.html` | Homepage — hero, 7-card gallery strip, story teaser, brands band |
| `gallery-alpine-expedition.html` | Public gallery — 35 images |
| `gallery-heritage-whimsical.html` | Public gallery — 57 images |
| `gallery-sun-tropical.html` | Public gallery — 39 images |
| `gallery-winter-wander.html` | Public gallery — 24 images |
| `gallery-motion-match.html` | Public gallery — 23 images |
| `gallery-nest-nook.html` | Public gallery — 23 images |
| `gallery-wild-natural.html` | Public gallery — 41 images |
| `story.html` | About page (has its own `<style>` block — not in styles.css) |
| `private.html` | Client login page (magic link — currently mocked) |
| `client-gallery.html` | Private client portal (favourites, comments, selection bar) |
| `enquiry.html` | Image licensing enquiry form (single image or batch) |
| `admin_dashboard.html` | Admin — overview, sales, gallery performance, clients, enquiries |
| `styles.css` | Global design system (CSS custom properties + grid) |

---

## Cloudinary Setup

**Cloud name:** `deeck6k0n`  
**Base URL:** `https://res.cloudinary.com/deeck6k0n/image/upload`

### Named Transformations (created without t_ prefix — Cloudinary prepends it automatically)

| Name in Cloudinary | URL token | Use |
|--------------------|-----------|-----|
| `hero` | `t_hero` | Homepage hero / gallery hero banners (~2400px) |
| `tile_landscape` | `t_tile_landscape` | Gallery grid — landscape images |
| `tile_portrait` | `t_tile_portrait` | Gallery grid — portrait images |
| `lightbox_public` | `t_lightbox_public` | Public gallery lightbox (watermarked, ~1200px) |
| `lightbox_private` | `t_lightbox_private` | Private client lightbox (clean, ~1200px) |
| `strip_card` | `t_strip_card` | Homepage gallery strip card thumbnails |

> ⚠️ **Known issue:** `t_strip_card` URL returning 404. Needs investigation — either the transformation isn't saved correctly in Cloudinary, or the public ID path format is wrong. Check Cloudinary dashboard → Transformations and verify `strip_card` is listed.

### Folder Structure in Cloudinary
```
roaming-lens/
  galleries/
    alpine-expedition/     (35 images)
    heritage-whimsical/    (57 images)
    sun-tropical/          (39 images)
    winter-wander/         (24 images)
    motion-match/          (23 images)
    nest-nook/             (23 images)
    wild-natural/          (41 images)
```

### Upload Preset
- EXIF/metadata stripped on upload (`fl_force_strip`)
- Signing mode: confirm in Cloudinary dashboard → Upload Presets

### Correct URL format
```
https://res.cloudinary.com/deeck6k0n/image/upload/t_tile_landscape/roaming-lens/galleries/alpine-expedition/DSCF0612_sn6gr6
```

---

## Gallery Architecture (how images load)

Each gallery HTML file has tiles like:
```html
<div class="tile"
     data-public-id="roaming-lens/galleries/FOLDER/PUBLIC_ID"
     data-title="Title text"
     data-cap="Caption text."></div>
```

On page load, `detectAndAssignSpans()` (in each gallery's `<script>` block):
1. Loads a tiny probe image (`w_20,q_1/PUBLIC_ID`) to detect portrait vs landscape
2. Assigns the correct CSS span class (`span-3` through `span-8`)
3. Sets `background-image` using `t_tile_portrait` or `t_tile_landscape`
4. Sets `data-img` to the `t_lightbox_public` URL for the lightbox

**This means:** tiles only need `data-public-id` set — the JS handles everything else.

### Grid System (styles.css)
12-column CSS grid. Span classes and their aspect ratios:
- `span-3` → 3:4 (portrait)
- `span-4` → 4:3
- `span-5` → 5:4 (portrait)
- `span-6` → 3:2
- `span-7` → 7:4
- `span-8` → 2:1
- Collapses to 6-column at ≤900px

### Load More
- Initial: 12 tiles shown
- Batch: 9 per "Load more" click
- Matches future Supabase pagination (`LIMIT 12 OFFSET 0`, `LIMIT 9 OFFSET N`)

---

## Homepage Strip Cards (index.html)

The 7 `.gcard` strip cards need these specific Cloudinary images with `t_strip_card`:

| Gallery | Public ID |
|---------|-----------|
| Winter & Wander | `roaming-lens/galleries/winter-wander/Lappland_Winter_4_miatid` |
| Alpine & Expedition | `roaming-lens/galleries/alpine-expedition/DSCF0612_sn6gr6` |
| Heritage & Whimsical | `roaming-lens/galleries/heritage-whimsical/DSCF9265_zcxzng` |
| Motion & Match | `roaming-lens/galleries/motion-match/DSCF2110a_otjqvi` |
| Nest & Nook | `roaming-lens/galleries/nest-nook/DSCF7246_cubnm2` |
| Sun & Tropical | `roaming-lens/galleries/sun-tropical/DSCF7457_gf8kce` |
| Wild & Natural | `roaming-lens/galleries/wild-natural/DSCF5202_gee7nu` |

> ⚠️ **Blocked by:** `t_strip_card` 404 issue above — fix the transformation first.

---

## Design System (styles.css)

### Colours
```css
--sand, --cream, --cream-deep, --clay, --olive, --ink
--clay-aa   /* 5.2:1 contrast — use for text */
--clay-deep /* 6.8:1 contrast — use for dark text */
```
> `--clay` alone fails WCAG AA on cream — always use `--clay-aa` or `--clay-deep` for text.

### Typography
```css
--serif: Cormorant Garamond  /* display headings */
--sans:  Inter               /* body */
--hand:  Caveat              /* accent script */
```

### Spacing scale: `--s1` (8px) through `--s8` (64px)

---

## Key Per-Page Patterns (copy to every new page)

### Image protection
```js
document.addEventListener('contextmenu', e => {
  if (e.target.classList.contains('tile') || e.target.tagName === 'IMG' || e.target.id === 'photo') {
    e.preventDefault();
  }
});
document.addEventListener('dragstart', e => e.preventDefault());
```

### Mobile nav controller
Each page has an IIFE toggling `aria-expanded` on `.nav-toggle`. CSS uses `aria-expanded="true"` as the open/close trigger — do not add a separate class.

---

## Security Notes

- **Admin auth:** `sessionStorage` key `wl-admin-auth` — mock only, not production safe
- **Client auth:** Plain text password check in `private.html` JS — mock only
- **Image protection:** Right-click/drag blocked client-side only (easily bypassed)
- **Real security:** Requires Phase 2 Supabase auth + signed S3/Cloudinary URLs before any real client data goes live

---

## Pending Tasks

### Immediate
- [ ] **Fix `t_strip_card` 404** — check Cloudinary dashboard, verify transformation exists and is named exactly `strip_card` (no t_ prefix in the dashboard name). Then update `index.html` strip card `<img>` src attributes with the 7 URLs above.
- [ ] **Update `private.html`** — redirect to `client-gallery.html` on successful mock login
- [ ] **Add admin logout button** to `admin_dashboard.html`
- [ ] **Sweep for "Wandering Light"** — replace all remaining instances with "Roaming Lens"

### Next
- [ ] Push all gallery changes to GitHub → triggers Vercel deploy
- [ ] Build Prompt 2: Admin drag-and-drop image ordering (SortableJS + JSON export)
- [ ] Create `client-petitbateau.html` — static private gallery (security-through-obscurity, no login required for initial client review)
- [ ] Wire enquiry form to EmailJS (no backend needed)

### Phase 2 (Supabase)
- [ ] Real auth for admin and clients
- [ ] Database schema: galleries, images (with `is_hero`, `sort_order`), clients, enquiries, enquiry_images
- [ ] Signed URL delivery for licensed images
- [ ] Migration script: Cloudinary metadata → Supabase (Claude Code can write this — all images already in Cloudinary with correct public IDs)

---

## Git → Vercel Workflow

```bash
git add .
git commit -m "your message"
git push origin main
```
Vercel auto-deploys on push. No build step — static files served directly.

---

## Cloudinary Credit Efficiency

- **Fully lazy** — transforms only fire when an image is actually requested
- **Orientation-aware** — portrait images only ever trigger `t_tile_portrait`, landscape only `t_tile_landscape` (no wasted credits on wrong-ratio transforms)
- **Probe image** — JS loads `w_20,q_1` thumbnail (minimal credits) to detect orientation before requesting full tile
- **Caching** — Cloudinary CDN caches transformed images; repeat views cost nothing
- Estimated usage: ~480 credits/month vs 1,000/month free tier limit

---

## Claude Code Prompts (for VS Code)

### Fix strip cards in index.html
```
Update the featured gallery strip cards in index.html. Each .gcard element has an <img> tag —
replace its src with the correct Cloudinary t_strip_card URL.

Pattern: https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/roaming-lens/galleries/FOLDER/PUBLIC_ID

Cards and images:
1. Winter & Wander        → winter-wander/Lappland_Winter_4_miatid
2. Alpine & Expedition    → alpine-expedition/DSCF0612_sn6gr6
3. Heritage & Whimsical   → heritage-whimsical/DSCF9265_zcxzng
4. Motion & Match         → motion-match/DSCF2110a_otjqvi
5. Nest & Nook            → nest-nook/DSCF7246_cubnm2
6. Sun & Tropical         → sun-tropical/DSCF7457_gf8kce
7. Wild & Natural         → wild-natural/DSCF5202_gee7nu

Only change src attributes on strip card images. Do not change any other content.
```

### Add admin logout
```
Add a logout button to admin_dashboard.html. It should clear the sessionStorage key
'wl-admin-auth' and redirect to index.html. Place it in the top-right of the admin
header bar, styled consistently with the existing dashboard. Use the existing
--clay-aa / --cream colour tokens.
```

### Fix private.html redirect
```
In private.html, find the mock magic link / login form submission handler. After
the mock "success" state is shown, add a redirect to client-gallery.html after
a 1.5 second delay. Keep the existing success message visible during the delay.
```

### Sweep Wandering Light → Roaming Lens
```
Search all .html files and styles.css for any remaining references to
"Wandering Light" or "wandering-light" or "wanderinglight" and replace with
"Roaming Lens" / "roaming-lens" / "roaminglens" as appropriate. Do not change
picsum.photos URLs or external CDN links.
```
