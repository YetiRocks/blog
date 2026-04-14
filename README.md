# Yeti Blog

Content repository for [yetirocks.com/www/blog](https://yetirocks.com/www/blog). The website syncs from this repo automatically — no build or deploy step needed.

## Repository Structure

```
posts/
  my-post-slug/
    index.md          — Post content (Markdown with YAML frontmatter)
    hero_image.png    — Hero image (optional, displayed below title)
    diagram.png       — Additional images referenced in markdown
```

Each post is a **folder**. The folder name is the URL slug.

## Adding a Post

1. Create a folder in `posts/` — the name becomes the URL slug:
   - `posts/my-great-post/` → `yetirocks.com/www/blog/my-great-post`
   - Use lowercase, hyphens for spaces, no special characters

2. Create `index.md` inside with YAML frontmatter + content

3. Optionally add `hero_image.png` for a banner image

4. Push to `main` — hit `/www/api/blogsync` to trigger sync

## File Template

```markdown
---
title: Your Post Title Here
description: A compelling 1-2 sentence description. Appears in search results and social cards. Under 160 characters.
date: 2026-04-20
author: Your Name
category: Engineering
readingTime: 5 min read
---

Opening paragraph — hook the reader. No heading needed, the title renders automatically.

## First Section

Body content. Short paragraphs. Write for engineers who skim.

## Second Section

Include code examples:

```graphql
type Example @table @export {
  id: ID! @primaryKey
  name: String! @indexed
}
`` `

## Conclusion

End with a call to action or forward-looking statement.
```

## Frontmatter Fields

All fields are required.

| Field | Format | Description |
|-------|--------|-------------|
| `title` | Text | Post title. Used in `<title>`, `<h1>`, and social cards. Under 70 characters. |
| `description` | Text | SEO meta description and social card text. Under 160 characters. |
| `date` | `YYYY-MM-DD` | Publication date. Posts sorted newest-first. |
| `author` | Text | Author name as displayed. |
| `category` | `Engineering` / `Company` / `Product` | Label on blog cards. |
| `readingTime` | Text | e.g., `5 min read`. ~200 words per minute. |

## Hero Image

The hero image appears below the title and description, above the post content. It's displayed at a **3:1 aspect ratio** (width is 3x height), cropped to fill.

| Property | Requirement |
|----------|-------------|
| **Filename** | Must be `hero_image.png` (exact name) |
| **Minimum size** | 1200 x 400 px (3:1 ratio) |
| **Recommended size** | 1800 x 600 px (sharp on retina displays) |
| **Format** | PNG or JPG (name must end in `.png`) |
| **Aspect ratio** | 3:1 — wider images are cropped from center, taller images are cropped top/bottom |
| **Content** | Avoid text in the image (it won't be readable on mobile). Abstract visuals, code screenshots, architecture diagrams, or photos work best. |
| **Optional** | If no `hero_image.png` is present, the post renders without a hero image. |

## Inline Images

Reference images in the same folder using standard Markdown:

```markdown
![Architecture diagram](architecture.png)
```

Images are automatically resolved to GitHub raw content URLs. Use descriptive alt text — it becomes a caption below the image.

## Categories

- **Engineering** — Technical deep dives, architecture, benchmarks, how-it-works
- **Company** — News, team updates, culture, hiring, milestones
- **Product** — Feature launches, release notes, use case showcases

## Style Guide

- **Tone**: Technical but accessible. Smart colleague, not professor or 5-year-old.
- **Length**: 800-1500 words (4-8 min read). Longer for deep dives.
- **Structure**: Problem → solution → what's next.
- **Code**: Include runnable examples. Show schema, request, response.
- **No fluff**: Every sentence informs or persuades. Cut "In this post, we will discuss..."
- **Comparisons**: Factual and specific. Benchmarks, not opinions.

## How It Works

1. The yeti-www `blog_sync` resource fetches the directory listing from this repo via GitHub API
2. For each post folder, it fetches `index.md` and parses the YAML frontmatter
3. Markdown is converted to HTML server-side (no client-side rendering)
4. Relative image paths (e.g., `architecture.png`) are resolved to absolute GitHub raw URLs
5. The rendered post is stored in a local `BlogPost` table (RocksDB)
6. The frontend reads from the local API — fast, no cross-origin requests
7. Hero images are checked via HEAD request; if missing, the field is null
