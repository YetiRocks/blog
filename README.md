# Yeti Blog

Blog posts for [yetirocks.com](https://yetirocks.com). Posts are markdown files in `posts/` with YAML frontmatter.

## Adding a Post

Create a `.md` file in `posts/`:

```markdown
---
title: Your Post Title
description: A short description for SEO and social cards.
date: 2026-04-20
author: Your Name
category: Engineering
readingTime: 5 min read
---

Your content here. Supports full Markdown including code blocks,
links, images, and headings.
```

Push to `main`. The yeti-www blog picks up new posts automatically — no rebuild or redeploy needed.

## Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| title | yes | Post title (used in title tag and OG tags) |
| description | yes | Short description for SEO meta and social cards |
| date | yes | Publication date (YYYY-MM-DD) |
| author | yes | Author name |
| category | yes | One of: Engineering, Company, Product |
| readingTime | yes | e.g., "5 min read" |
