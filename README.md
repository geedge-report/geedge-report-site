# Geedge Report

Jekyll site for Geedge Report, an ongoing project studying the September 2025 Geedge Networks
leak. It hosts *Technical Analysis of the Geedge Networks Firewall Source Code Leak*
(USENIX Security '26) and follow-up work.

The landing page is written for a general audience; the paper write-up post and the
`/findings/` pages carry the technical material.

The paper post's URL is stored once, as `paper.post_url` in `_config.yml`, and referenced
everywhere via `{{ site.paper.post_url }}`. If you re-date or rename that post, update that one
value.

**Note:** `jekyll serve` reads `_config.yml` once at startup. Changes to the nav, title, author
list, or `baseurl` require restarting the server — content and CSS hot-reload fine.

## Run locally

```bash
bundle install && bundle exec jekyll serve --livereload
```

Then open <http://localhost:4000>.

## Structure

| Path | Description |
|---|---|
| `_config.yml` | Site settings, author list, affiliations, nav, paper metadata |
| `index.html` | Landing page: plain-language project overview, short findings, paper CTA |
| `_posts/*-geedge-firewall-source-code-analysis.md` | The paper write-up post: metadata, abstract, full findings, section walkthrough, citation |
| `_findings/*.md` | Technical deep-dives at `/findings/<name>/`, one per paper section; indexed from the paper post's `#walkthrough` |
| `posts.html` | Post index (`/posts/`) |
| `_posts/*.md` | Dated posts, `YYYY-MM-DD-slug.md` |
| `resources.md` | Release index, artifact, publication/access policy, press reporting, related work |
| `team.html` | Authors, generated from `_config.yml` |
| `_layouts/`, `_includes/` | Templates |
| `assets/css/main.css` | All styling (light + dark, no JS, no external assets) |

## Adding a post

Create `_posts/YYYY-MM-DD-slug.md`:

```markdown
---
title: "Post title"
subtitle: "One-line summary shown in listings."
author: "Name"
---

Body in Markdown.
```

The layout is applied automatically. Add `* TOC` followed by `{:toc}` at the top for a
table of contents.

Note: pages that contain Markdown must use a `.md` extension — Jekyll does not run the
Markdown converter on `.html` files. Use `.html` only for pages written entirely in
HTML + Liquid (`index.html`, `posts.html`, `team.html`).

## Adding a findings page

Create `_findings/slug.md` with `title`, `subtitle`, `summary`, `paper_section`, and `order`
in the front matter. It will appear automatically on the home page, the findings index, and in
the prev/next pager, ordered by `order`.

## Before publishing

1. Set `url` and `baseurl` in `_config.yml` for your deployment target.
2. Confirm `contact_email` in `_config.yml` — it is currently a placeholder.
3. Fill in the `site:` field for any author who wants a homepage link.
4. If using a custom domain, add a `CNAME` file at the repo root.

## Deployment

`.github/workflows/pages.yml` builds and deploys on push to `main`. In the repo settings, set
**Pages → Build and deployment → Source** to **GitHub Actions**.

## Content policy

This site publishes only material that appears in the camera-ready paper and in public
reporting. It does not host leaked source code, the TSG build, or deployment instructions —
see the "What we publish, and what we don't" section of `resources.md` (anchor `#access`),
which is what every in-page policy link points to.

## Adding a resource

`resources.md` opens with a release index table. To publish a new resource, add a row there
and give it a section below with a matching `{#anchor}`. The file currently contains one
stubbed placeholder resource (the TSG fingerprint reference), marked with "Placeholder"
callouts — fill it in or delete both the row and its section.
