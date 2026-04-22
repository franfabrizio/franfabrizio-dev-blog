# AGENTS.md

## Hugo Blog for franfabrizio.dev

Static site built with **Hugo** (version 0.136.5), deployed to **Cloudflare Pages**.

## Development

```bash
hugo server -D          # dev server with drafts
hugo                    # production build -> ./public
```

## Build & Deploy

- Builds output to `public/`
- Cloudflare Pages auto-deploys on push via `wrangler.toml`
- `HUGO_VERSION` env var specified in wrangler.toml

## Structure

- `content/posts/` - blog posts (markdown with bundle images)
- `themes/hugo-theme-m10c/` - git submodule (franfabrizio/hugo-theme-m10c)
- `layouts/shortcodes/` - custom shortcodes (e.g., `{{</* toc */>}}`)

## Key Files

- `config.toml` - Hugo config, theme, menu, analytics
- `wrangler.toml` - Cloudflare Pages build config

## Common Tasks

- Add new post: `hugo new posts/<slug>/index.md`
- Preview: `hugo server`
- CI is Cloudflare Pages - no local test step needed