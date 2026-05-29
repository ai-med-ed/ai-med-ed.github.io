# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Static marketing site for Dendro (AI-powered medical education for residency programs and health systems). Plain HTML + CSS, no build step, no JS framework, no package manager, no tests. Deployable as static files.

## Running locally

There is no toolchain. Open files directly or serve the directory:

```
python3 -m http.server 8000
```

## Architecture

### Pages

- `index.html` — the homepage. **This is a self-contained bundler artifact, not a hand-written HTML page.** The real markup lives JSON-encoded on a single line inside `<script type="__bundler/template">`. Assets (logo, screenshots, etc.) are base64-packed into `<script type="__bundler/manifest">`. On load, a small bootloader decodes assets to blob URLs, swaps UUIDs in the template, and replaces the document. Anything you append directly to `<body>` gets discarded at runtime. To edit content, you must modify the JSON-encoded template string — see "Editing index.html" below.
- `about.html`, `media.html`, `contact.html` — normal standalone HTML pages.
- `robots.txt`, `sitemap.xml` — SEO crawler files. Both reference `https://dendroeducation.com` as the canonical host. If the canonical host changes, update both files plus every `<link rel="canonical">`, `og:url`, and JSON-LD `url` field across all four HTML pages.

The homepage uses internal anchors (`#home`, `#how`, `#features`, `#media`, `#testimonial`, `#about`, `#contact`) and a single-page nav. Other pages link back to anchors on `index.html`.

### Editing `index.html`

Don't try to hand-edit the escaped HTML on the template line. Use a short Python script: read the file, `json.loads()` line 181 (1-indexed) to get the real HTML, edit that string, `json.dumps()` back, and write the file. Always round-trip-verify (re-parse and assert the edits are present) before committing. The bundler swallows errors into an overlay — silent breakage is the failure mode.

**Footgun:** after `json.dumps()`, you must `replace("</script>", "<\\/script>")` on the encoded string. The template JSON lives inside an outer `<script type="__bundler/template">…</script>` block; any raw `</script>` in the JSON string body will end that outer script element early, truncating the template. The browser then throws `Unterminated string in JSON at position …`. Python's `json.loads` won't catch this (it's lenient about non-escaped slashes in input); verify with `node -e "JSON.parse(...)"` which uses the same V8 parser as the browser. The decoded template currently contains multiple `<script>` blocks (the runtime nav/scroll/carousel/form JS, plus two JSON-LD blocks in `<head>`), so the escape rule matters more than ever — adding more `<script>` blocks (e.g., analytics) compounds the risk if you forget.

### Other pages: shared styles

`about.html`, `media.html`, and `contact.html` share `assets/styles.css`, which owns the design system (tokens, typography scale, nav, buttons, pills, cards, footer, animations, responsive breakpoints). Page-specific styles live in a `<style>` block inside each page's `<head>` — this is intentional, not technical debt. When adding visual elements:

- Reusable primitives (a new button variant, a token, a card style used on more than one page) → `assets/styles.css`.
- One-page layouts and section-specific styling (hero, feature grid, testimonial block) → that page's inline `<style>`.

Note: `index.html` does **not** link `assets/styles.css` — the bundler embeds its own copy of the design tokens inside its `<style>` block. If you change `assets/styles.css`, the homepage won't pick the change up automatically.

### Design tokens

`:root` in `assets/styles.css` is the single source of truth for color, spacing, radii, type, shadows, and transitions. Never hardcode hex colors, px spacing values, or font families in page styles — always reference `var(--c-green)`, `var(--space-lg)`, `var(--r-xl)`, `var(--font-display)`, etc. This is how the site stays visually consistent across pages without a component framework.

The palette is a dark green theme: backgrounds `--c-bg`/`--c-bg2`/`--c-bg3`/`--c-surface` from darkest to lightest, accent `--c-green` (#a8e063), text `--c-text`/`--c-text-sub`/`--c-text-dim` from brightest to dimmest. Spacing scale is `xs/sm/md/lg/xl/2xl/3xl` (4 / 8 / 16 / 32 / 64 / 96 / 128 px).

### Shared structural elements

Nav and footer markup is duplicated across pages (no templating). When changing nav links, footer links, or the logo treatment, update all `.html` files together. The nav scroll-blur effect is driven by ~6 lines of JS at the bottom of each page that toggles `.scrolled` on `#navWrapper` — keep this in sync if you copy a page.

### Typography utility classes

Use `.t-display`, `.t-headline`, `.t-title`, `.t-body`, `.t-body-lg`, `.t-small`, `.t-label` for size/weight, and `.t-green`/`.t-sub`/`.t-dim` for color. Prefer composing these over redefining font-size/weight inline.

### Animation conventions

Entry animations use `.anim-fade-up` / `.anim-fade-in` with `.anim-delay-1..4` stagger classes. They fire once on page load via CSS keyframes — there's no IntersectionObserver, so don't add `anim-*` classes far down a page expecting them to trigger on scroll.

### Carousels

Two carousels exist on the homepage — `#how` (process steps) and previously `#features` (now a static grid). Both are driven by a generic `initCarousel(rootId)` function in the homepage's inline script. The function null-guards on missing root (`if (!root) return;`), so it's safe to leave call sites if you remove the corresponding DOM, but cleaner to remove the call too. If you ever add a third carousel, follow the same DOM shape: `.carousel-tabs > .carousel-tab[data-i]`, `.carousel-slide-text[data-i]`, `.carousel-image[data-i]`. The IntersectionObserver inside `initCarousel` auto-pauses when scrolled out of view.

### Contact form

The contact form is embedded inside the homepage's `#contact` section (not on `contact.html`, which still exists as a fallback page but is no longer the primary contact path — homepage form is the live one). Nav "Contact" links scroll-to `#contact`.

The form submits to **Web3Forms** (`https://api.web3forms.com/submit`) via fetch with `FormData`. The access key sits in a hidden input — it's public-by-design (ships in the page source), not a secret. A hidden `botcheck` checkbox acts as a honeypot; Web3Forms drops any submission where it's filled. On success → show `#formSuccess`; on error → re-enable the button and show a fallback message pointing to `hossam_zaki@brown.edu`.

If spam becomes a problem, the next step is enabling hCaptcha in the Web3Forms dashboard, not changing the markup.

### SEO

Every page has page-specific `<title>`, `<meta name="description">`, `<link rel="canonical">`, Open Graph tags, Twitter Card tags, and favicon links. The homepage additionally has two `application/ld+json` blocks (Organization + SoftwareApplication structured data) — those JSON-LD scripts live inside the bundler template, so the `</script>` escape rule applies when editing them.

Canonical host is `https://dendroeducation.com`. If you change it, the search-and-replace targets are: every `<link rel="canonical">`, every `og:url`, every `twitter:` URL, the JSON-LD `url` and `logo` fields on the homepage, `robots.txt`, and `sitemap.xml`. Use a single `find . -name '*.html' -o -name 'robots.txt' -o -name 'sitemap.xml' | xargs grep -l dendroeducation` to find them all.

Social card image is currently `assets/logo.png` (512×512 square). Twitter card type is `summary` (small square) — match this to the image. If you swap in a 1200×630 designed card, also change `twitter:card` back to `summary_large_image` and re-add the `og:image:width`/`og:image:height` meta tags.

## Assets

`assets/` holds images and the shared stylesheet. Images currently include `logo.png` (512×512, also used as favicon and social card), `app-screenshot.png`, team portraits (`jay.jpeg`, `soukas-peter.jpg`), and `BVP_2025.mp4` (238MB Brown Venture Prize pitch video). Prefer descriptive filenames over generic ones (e.g., `soukas-peter.jpg`, not `headshot-3.jpg`).

**Heads up on the .mp4:** 238MB is too large for most static hosts' file-size limits (Netlify free is 25MB per file, Vercel is 100MB) and will balloon the repo if pushed to git. Before deploying, either (a) move the video to YouTube/Vimeo and embed (the planned path — see the media-section TODO), or (b) move it to git-lfs / a CDN. Don't push the raw file to GitHub without addressing this — it'll hit the 100MB GitHub file limit on push.
