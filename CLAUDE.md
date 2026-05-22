# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
npm run dev       # Start dev server at localhost:4321
npm run build     # Build production site to ./dist/
npm run preview   # Preview the production build locally
npm run astro -- check  # TypeScript/Astro type checking
```

## Architecture

This is an [Astro](https://astro.build) blog site deployed to GitHub Pages at `mtn-man.github.io`. Pushes to `master` trigger the deploy workflow in `.github/workflows/deploy.yml`.

**Content** lives in `src/content/blog/` as `.md` or `.mdx` files. The collection schema is defined in `src/content.config.ts` — every post requires `title`, `description`, and `pubDate` frontmatter; `updatedDate` and `heroImage` are optional.

**Routing** is file-based under `src/pages/`: `index.astro` (home/post list), `about.astro`, `blog/[...slug].astro` (individual posts), and `rss.xml.js` (RSS feed).

**Site-wide constants** (`SITE_TITLE`, `SITE_DESCRIPTION`) are exported from `src/consts.ts` and imported wherever needed.

**Integrations**: `@astrojs/mdx` (MDX support), `@astrojs/sitemap` (auto-generated sitemap), `@astrojs/rss` (RSS feed). The local Atkinson font is loaded via Astro's font API and exposed as the CSS variable `--font-atkinson`.
