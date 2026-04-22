# franfabrizio.dev — Tech Blog

## Overview

A personal tech blog documenting homelab experiences, DevOps knowledge, DIY projects, and practical tutorials. Built with Hugo, hosted on Cloudflare Pages.

## Project Structure

```
content/
  posts/<topic>/          # Posts organized by topic
  posts/homelab-101.md    # Some posts use flat structure

layouts/                  # Hugo layouts
themes/hugo-theme-m10c/  # Minimal performance-optimized theme
```

## Post Creation Workflow

### Creating a New Post

1. **Create post** 
   - Run the command `hugo new -k post-bundle <title of post>` to create a new post. This must be run in the posts/ subdirectory to create the post in the right location.
   - This uses the post-bundle I've pre-defined. It will create a folder and an index.md. Topic folders keep related posts together
   - Flat structure for standalone posts (e.g., `homelab-101.md`) but these are legacy posts. New posts should use the command and structure described above.
 
2. **Frontmatter essentials**:
   ```yaml
   ---
   title: "Your Title Here"
   date: 2026-04-14T00:00:00-07:00  # Use local timezone
   draft: false                     # Set to true while writing
   toc: true                        # Show table of contents
   tags: ["tag1", "tag2"]          # Use consistent slug-style tags
   ---
   ```

3. **Write content**:
   - Practical, no-fluff tutorials
   - Include error messages, commands, and expected output
   - Link to related posts when relevant
   - Use code blocks with language identifiers

4. **Preview locally**:
   - Run `hugo` or start dev server to preview
   - Check for broken links and rendering issues

5. **Publish**:
   - Set `draft: false`
   - Commit with clear message: `feat: add <topic> post`
   - Push to trigger Cloudflare Pages build

### Post Naming Conventions

Accept hugo defaults for file and directory naming.

## Content Style Guidelines

- **TIL series**: "Things I Learned" quick tips and gotchas
- **Practical focus**: Show actual commands, errors, and solutions
- **Avoid**: Marketing fluff, overly theoretical deep dives without implementation
- **DO link**: Related posts, external docs, and references

## Tech Stack Notes

- **Hugo extended**: Required for SCSS compilation (`hugo_extended`)
- **Theme**: `hugo-theme-m10c` (minimal, 100/100 Lighthouse)
- **Cloudflare Pages**: Deployed via `wrangler.toml`
- **Analytics**: Google Analytics (G-8LBKY582ZV)
- **Comments**: Disqus integration

## Common Tasks

### Check post count
```bash
find content/posts -name "*.md" | wc -l
```

### List all posts
```bash
find content/posts -name "index.md" -o -name "*.md" | grep -v test | sort
```

### Create post with folder

Use the `hugo new` command outlined above.

## Avoid

- Don't create documentation files (`.md` READMEs) unless explicitly requested
- Don't add emojis to content files
- Don't refactor working code unless asked
- Don't add features beyond scope

## Preferences

- **Git commits**: Keep messages imperative, present tense ("Add post", not "Added post"). AI-initiated commits should indicate that (e.g. [via Claude])
- **Branches**: Use `feat/<description>` or `fix/<description>` naming
- **Reviews**: Focus on clarity and correctness, not style nitpicks
- **Testing**: Verify posts render correctly before committing