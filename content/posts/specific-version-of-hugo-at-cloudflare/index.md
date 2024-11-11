---
title: "Using a Specific Version of Hugo at Cloudflare"
date: 2024-11-10T23:29:19-06:00
draft: false
toc: false
---

Just a quick post for those who might be struggling to get Cloudflare to use a specific version of Hugo for a site they are hosting there. The [Cloudflare documentation suggests](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/#use-a-specific-or-newer-hugo-version) that you can add a `HUGO_VERSION` environment variable to your site build via your Cloudflare dashboard. This may indeed be the case, but I struggled with it for quite a while, and my environment variable was having no apparent effect.

How I eventually fixed it is to use Cloudflare's Wrangler, their CLI solution for controlling their Workers (which are used to build your Hugo site). Wrangler can read configuration from a `wrangler.toml` file in the root of your repo. I created the following `wrangler.toml`:

```toml
name = "franfabrizio-dev-blog"
pages_build_output_dir = "public"

[vars]
HUGO_VERSION = "0.136.5"
```

Change the name to your blog's name, of course.

This worked immediately, and I prefer having the config in the repo anyhow. Their documentation strangely makes no mention of this approach. Hopefully this saves someone some frustration!