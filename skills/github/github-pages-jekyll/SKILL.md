---
name: github-pages-jekyll
description: Deploy a Jekyll markdown blog from a GitHub repo's /docs folder — Pages API, workflow, collections, pitfalls.
version: 1.0.0
tags: [GitHub Pages, Jekyll, Blog, Markdown, Static Site]
---

# GitHub Pages Jekyll Deployment

Deploy a blog/site from markdown files in a repo's `/docs` folder using GitHub Pages with Jekyll collections. Covers API enablement, workflow setup, Jekyll config, and every pitfall encountered.

## Trigger

When the user wants to host markdown content from a repo as a static site on GitHub Pages.

## Quick Start

Three files, one API call:

1. `docs/_config.yml` — Jekyll config with collections
2. `docs/index.md` — homepage with frontmatter
3. `.github/workflows/pages.yml` — build + deploy action
4. `POST /repos/{owner}/{repo}/pages` — enable Pages

## 1. Enable Pages via API

```bash
# Enable Pages from /docs on main branch
curl -X POST -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/OWNER/REPO/pages" \
  -d '{"source":{"branch":"main","path":"/docs"}}'

# Set build type to "workflow" (required for GitHub Actions)
curl -X PUT -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/OWNER/REPO/pages" \
  -d '{"build_type":"workflow"}'
```

## 2. Jekyll Config (`docs/_config.yml`)

```yaml
title: My Blog
description: Description here

collections:
  blog:
    output: true
    permalink: /blog/:name
```

Collections go in `docs/_blog/` (underscore prefix). Jekyll processes collection files, outputting them at the `permalink` path. The collection name in config matches the directory without underscore prefix.

## 3. Homepage (`docs/index.md`)

Must have Jekyll frontmatter:
```markdown
---
title: My Blog
layout: default
---

Content here...
```

## 4. Blog Posts (`docs/_blog/*.md`)

Every file MUST have frontmatter (`---` block at top). Jekyll silently skips `.md` files without it.

```markdown
---
title: Post Title
layout: default
---

Content...
```

## 5. GitHub Actions Workflow (`.github/workflows/pages.yml`)

```yaml
name: Pages
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/jekyll-build-pages@v1
        with:
          source: docs
      - uses: actions/upload-pages-artifact@v3
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/deploy-pages@v4
```

## Pitfalls

### Spaces in file/directory names → 404
GitHub Pages cannot serve files from directories with spaces. Use clean slugs: `open-qna.md` not `Open QNA.md`. Rename or copy to flat directory with URL-safe names.

### Jekyll ignores files without frontmatter
Every `.md` file in a collection must start with `---\n...\n---`. Without it, Jekyll silently skips the file — no error, just 404.

### Workflow path filter blocks self-trigger
```yaml
paths: [docs/**]
```
This prevents the workflow from triggering when only `.github/` files change. Remove the filter or push a `docs/` file to trigger the first build.

### Legacy vs Workflow build types
"legacy" build type copies files raw — subdirectories with spaces/encoding break. Always use "workflow" with GitHub Actions.

### Pages deployment takes ~45s
After push, the workflow runs `jekyll-build-pages` (~20s) + `deploy-pages` (~20s). Wait 45-60s before testing URLs.

### Build not triggering on new commits
Check workflow runs: `GET /repos/{owner}/{repo}/actions/runs`. If none appear, the workflow trigger is wrong (branch mismatch, path filter, etc.).

## Skip List
- Custom Jekyll themes (minima/Cayman) — add when raw markdown looks too plain
- RSS, sitemap, search — add when traffic warrants
- Obsidian wikilink resolution — Jekyll doesn't support `[[links]]`, use relative markdown links
- Custom layouts (header, footer, nav) — add when the blog needs branding
