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

**Footgun:** after `json.dumps()`, you must `replace("</script>", "<\\/script>")` on the encoded string. The template JSON lives inside an outer `<script type="__bundler/template">…</script>` block; any raw `</script>` in the JSON string body will end that outer script element early, truncating the template. The browser then throws `Unterminated string in JSON at position …`. Python's `json.loads` won't catch this (it's lenient about non-escaped slashes in input); verify with `node -e "JSON.parse(...)"` which uses the same V8 parser as the browser.

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

## Assets

`assets/` holds images and the shared stylesheet. Images currently include `logo.png`, `app-screenshot.png`, and team/testimonial portraits. Prefer descriptive filenames over generic ones (e.g., `soukas-peter.jpg`, not `headshot-3.jpg`).
