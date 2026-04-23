# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Local dev server
hugo server

# Production build (same as CI)
hugo --minify
```

## Architecture

Hugo static site deployed to GitHub Pages via GitHub Actions (`.github/workflows/deploy.yml`). Pushes to `main` trigger automatic build and deploy.

**Theme**: PaperMod is a git submodule at `themes/PaperMod/`. Custom overrides in `layouts/` take precedence over the theme — edit there, not in `themes/`.

**CMS**: Sveltia CMS at `/admin` (`static/admin/`). Uses GitHub backend with OAuth proxy hosted on Render. Saving in CMS commits directly to `main`. Editorial workflow is not supported by Sveltia CMS.

**Content**: Posts go in `content/posts/`, static pages directly under `content/`. Front matter uses TOML format (`+++`). Posts with `draft = true` are excluded from the production build.

**Comments**: Giscus (GitHub Discussions) — config in `hugo.toml` under `[params.giscus]`, rendered via `layouts/partials/comments.html` which syncs theme with the blog's dark/light mode.

**Search**: Fuse.js — requires `home = ["HTML", "JSON"]` in `hugo.toml` outputs and the `content/search.md` page.
