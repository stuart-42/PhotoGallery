# Cloudinary 404 — Diagnosis and Fix

## TL;DR

`t_strip_card` is **not** the problem. The transformation works. The bug is in **every Cloudinary URL on the site**: the public IDs include a folder prefix (`roaming-lens/galleries/<gallery>/`) that does not exist in Cloudinary's actual public IDs. Strip the prefix and every URL resolves.

This affects `index.html` (7 strip cards) **and every gallery HTML file** (`data-public-id` attributes used by `detectAndAssignSpans()`). If the site appears to render images locally, those galleries are likely broken in production too.

## How I diagnosed it

Tested URLs against `https://res.cloudinary.com/deeck6k0n/image/upload/...`:

| URL | Result |
|---|---|
| `t_strip_card/roaming-lens/galleries/alpine-expedition/DSCF0612_sn6gr6` | 404 |
| `t_tile_landscape/roaming-lens/galleries/alpine-expedition/DSCF0612_sn6gr6` | 404 |
| `roaming-lens/galleries/alpine-expedition/DSCF0612_sn6gr6` (no transform) | 404 |
| `sample.jpg` (Cloudinary's built-in demo) | 200, valid JPEG |
| `DSCF0612_sn6gr6` (no prefix, no transform) | 200, 940KB JPEG |
| `t_strip_card/DSCF0612_sn6gr6` (no prefix) | 200, 75KB JPEG |
| `t_tile_landscape/DSCF0717a_xvqp76` (gallery image, no prefix) | 200, 51KB JPEG |

The cloud (`deeck6k0n`) is reachable, the named transformation `t_strip_card` exists and works (the 940KB → 75KB delta proves it's being applied), and the bare public IDs resolve. The folder string in the URL is the only thing that breaks them.

## Why this happens

In Cloudinary, "folders" are dashboard organisation only — they don't become part of the public ID unless **Settings → Upload → "Use folder as public ID prefix"** is on. With dynamic folders mode (the modern default), folder metadata lives separately from the public ID. So an image you see at `roaming-lens/galleries/alpine-expedition/DSCF0612` in the dashboard has the public ID `DSCF0612_sn6gr6` — the path is purely cosmetic.

## What needs to change

### 1. `index.html` — strip card URLs (7 occurrences)

Find:
```
https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/roaming-lens/galleries/<anything>/
```
Replace with:
```
https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/
```

The corrected file is provided as `index.proposed.html` alongside this note.

### 2. All seven gallery HTML files — `data-public-id` attributes

Find (regex, in each `gallery-*.html`):
```
data-public-id="roaming-lens/galleries/[^/]+/
```
Replace with:
```
data-public-id="
```

Counts to expect (matches what's referenced in PROJECT_HANDOFF.md):

| File | Tiles |
|---|---|
| `gallery-alpine-expedition.html` | 35 |
| `gallery-heritage-whimsical.html` | 57 |
| `gallery-sun-tropical.html` | 39 |
| `gallery-winter-wander.html` | 24 |
| `gallery-motion-match.html` | 23 |
| `gallery-nest-nook.html` | 23 |
| `gallery-wild-natural.html` | 41 |
| **Total** | **242** |

The probe-and-detect JS in each gallery already builds URLs from `data-public-id` — once the attribute values are correct, lightbox + tile + probe URLs all fix themselves.

### 3. `PROJECT_HANDOFF.md` — documentation drift

The handoff currently documents public IDs with the folder prefix. After fixing the code, update the doc so future-you doesn't re-introduce the bug.

## Claude Code prompt to apply this

Paste this into Claude Code in VS Code from the project root:

```
In all .html files in this directory, remove the Cloudinary folder prefix from public IDs.

Specifically, find every occurrence of "roaming-lens/galleries/<anything>/" inside Cloudinary URLs and data-public-id attributes, and remove the "roaming-lens/galleries/<anything>/" portion only — leaving the bare public ID.

Two patterns to fix:

1. Background-image URLs in index.html:
   FROM: https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/roaming-lens/galleries/<gallery>/<id>
   TO:   https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/<id>

2. data-public-id attributes in all gallery-*.html files:
   FROM: data-public-id="roaming-lens/galleries/<gallery>/<id>"
   TO:   data-public-id="<id>"

Do NOT change anything else. Do not touch picsum.photos URLs, CDN script tags, or other content.
After the change, expected counts: 7 occurrences fixed in index.html, 242 across the seven gallery-*.html files.
```

## After applying the fix

1. Hard-reload the homepage and one gallery page locally.
2. Verify in DevTools Network tab that the strip card and gallery image requests return 200 (not 404). Sample URL to spot-check: `https://res.cloudinary.com/deeck6k0n/image/upload/t_strip_card/DSCF0612_sn6gr6` — should load a small (~75KB) JPEG.
3. Push to GitHub → Vercel auto-deploys.

## On the upload preset / future uploads

Going forward you have two choices:

- **Keep public IDs bare** (recommended — matches what's in Cloudinary now). Just don't add the path on the HTML side.
- **Re-upload with folder prefix in public ID** by enabling "Use folder as public ID prefix" in Cloudinary Settings → Upload, then re-uploading. This is more work and provides no real benefit; the dashboard folder view stays the same either way.

The migration script for Phase 2 (Supabase) should store the bare public ID — Supabase doesn't care about Cloudinary's folder organisation, and you'll want the same identifier on both sides.
