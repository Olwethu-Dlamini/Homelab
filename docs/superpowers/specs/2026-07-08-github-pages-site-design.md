# GitHub Pages Teaching Site — Design

**Date:** 2026-07-08
**Repo:** Homelab (`github.com/Olwethu-Dlamini/Homelab`)
**Live URL (once enabled):** https://olwethu-dlamini.github.io/Homelab/

## Goal

Publish a GitHub Pages website that acts as the front door to the homelab
teaching material: a rich summary landing page linking to the full guide and
the hypervisor docs. No build step — plain static files served from `main`.

## Approach

Hand-written `index.html` (chosen over Jekyll/MkDocs for zero build and full
control), styled to match `home-server-guide-v2.html`.

## Scope

**In scope**
- `index.html` — summary landing page (hero + "the journey" overview + card grid).
- `.nojekyll` — serve files as-is, no Jekyll processing.
- Enable Pages: source `main` / root (manual, one-time, via repo Settings).

**Out of scope (YAGNI)**
- Rendering `.md` docs natively in-site (they link to GitHub's rendered views).
- Custom domain (using default `*.github.io`).
- Search / sidebar nav / MkDocs — noted as a future upgrade path.

## How content is served

- `index.html` and `home-server-guide-v2.html` → served by Pages (same domain).
- The four hypervisor `.md` docs → linked to their **GitHub rendered blob
  views** (`.../blob/main/docs/hypervisors/*.md`), which render Markdown with
  full styling and no build.

## Page structure (`index.html`)

- **Hero** — gradient title, one-line pitch, primary button to the guide.
- **The journey** — short teaching overview: foundations → hypervisor → build →
  deploy → harden.
- **Card grid** — links to: Complete Guide (v2), and the 4 hypervisor docs.
- **Footer** — repo link, license note.

Styling reuses the guide's palette (dark `#0f1419`, accent `#5db0ff`, green
`#7ee787`, purple `#bc8cff`, gradient headings), self-contained inline CSS.

## Delivery plan (one topic per commit, push after each)

1. Add this design doc.
2. Add `index.html` + `.nojekyll`.
3. (Manual) Enable Pages in repo Settings → Pages → source `main` / root.

## Success criteria

- Site loads at the Pages URL with the styled landing page.
- Every card links correctly (guide on-domain; docs to GitHub rendered views).
- Look is consistent with the existing guide.
