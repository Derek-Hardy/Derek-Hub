---
name: derek-hub
description: Use this skill when working on Derek's personal Hugo blog in the Derek-Hub repository, including adding posts, changing Hugo/PaperMod configuration, editing content, or validating the local site.
---

# Derek-Hub Blog

## Project Shape

- This repository is Derek's personal blog, built with Hugo and the PaperMod theme.
- Main Hugo config: `config.yml`.
- Theme: `themes/paper-mod`, referenced as `theme: [paper-mod]`.
- Content lives under `content/`.
- Blog posts live under `content/posts/`.
- About page lives at `content/about/_index.md`.
- Generated output lives under `public/`; avoid editing it directly unless explicitly asked.
- Custom project files are minimal:
  - `layouts/shortcodes/details.html` provides a `details` shortcode.
  - `assets/css/common/custom.css` contains `.fixed-code` styling for scrollable expanded code/content.

## Content Conventions

Use TOML front matter for posts:

```toml
+++
date = 'YYYY-MM-DDTHH:MM:SS+08:00'
draft = true
title = 'Post Title'
+++
```

- New posts should usually be created in `content/posts/<slug>.md`.
- Existing posts use `draft = false` for published posts and `draft = true` for unpublished drafts.
- The blog voice is personal and engineering-focused: Python, Go, distributed scheduling, ad tech, productivity, language learning, and career notes.
- Prefer concise explanations with practical examples and Markdown code blocks.

## Common Commands

Create a new post:

```bash
hugo new posts/<new-post-name>.md
```

Run locally:

```bash
hugo server
```

Build:

```bash
hugo --gc --minify
```

## Site Configuration Notes

- `baseURL` is `https://derek-hardy.github.io`.
- Site title is `Derek's Hub`.
- Important PaperMod params enabled in `config.yml`:
  - breadcrumbs
  - reading time
  - previous/next post links
  - table of contents
- Main menu has `Posts` and `About`.
- Home intro presents Derek as a software engineer in Singapore, currently at Goldman Sachs, previously at ByteDance/Pangle.

## Deployment

- GitHub Actions workflow: `.github/workflows/hugo.yaml`.
- Deploys to GitHub Pages on pushes to `master`.
- Workflow uses Hugo Extended `0.139.0`, checks out submodules recursively, builds with `hugo --gc --minify`, then deploys `public/`.

## Editing Guidance

- Match existing Hugo and Markdown style before adding new structure.
- Keep changes scoped to source files such as `content/`, `config.yml`, `layouts/`, and `assets/`.
- Do not manually edit generated `public/` output for normal content changes.
- If adding a new post shell by hand, use the existing TOML front matter style and Singapore timezone offset `+08:00`.
