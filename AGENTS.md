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

## Post Creation

```bash
hugo new -k post-bundle posts/<slug>
```

Frontmatter:
```yaml
---
title: "Your Title Here"
date: 2026-04-14T00:00:00-07:00  # Use local timezone
draft: true                       # Set false when ready
toc: true
tags: ["tag1", "tag2"]
---
```

## Common Tasks

- Preview: `hugo server`
- List posts: `find content/posts -name "index.md" -o -name "*.md" | grep -v test | sort`
- Post count: `find content/posts -name "*.md" | wc -l`

## Style

- Practical, no-fluff tutorials with actual commands and errors
- Link related posts and external docs
- Use code blocks with language identifiers
- No emojis in content

## Commit Style

- Imperative present tense: "Add post" not "Added post"
- AI-initiated commits: include "[via Claude]"
- Branch naming: `feat/<description>`, `fix/<description>`

## Tech Stack

- Hugo extended (required for SCSS)
- Theme: hugo-theme-m10c
- Analytics: Google Analytics (G-8LBKY582ZV)
- Comments: Disqus
