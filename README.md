# Yeti Blog

Content repository for the [yetirocks.com/www/blog](https://yetirocks.com/www/blog). Posts are markdown files in `posts/` with YAML frontmatter. The website fetches directly from this repo — no build or deploy step needed.

## Adding a Post

1. Create a new `.md` file in `posts/` — the filename becomes the URL slug:
   - `posts/my-great-post.md` → `yetirocks.com/www/blog/my-great-post`
   - Use lowercase, hyphens for spaces, no special characters
   - Keep slugs short and descriptive

2. Add YAML frontmatter at the top of the file (between `---` markers)

3. Write content in standard Markdown below the frontmatter

4. Push to `main` — the post appears on the website within 60 seconds

## File Template

```markdown
---
title: Your Post Title Here
description: A compelling 1-2 sentence description. This appears in search results, social media cards, and the blog index. Keep it under 160 characters.
date: 2026-04-20
author: Your Name
category: Engineering
readingTime: 5 min read
---

Opening paragraph — hook the reader. No heading here, the title is rendered automatically from the frontmatter.

## First Section Heading

Body content. Use short paragraphs. Write for engineers who skim.

## Second Section Heading

More content. Include code examples where relevant.

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
| `title` | Text | Post title. Appears as `<title>`, `<h1>`, and in social cards. Keep under 70 characters. |
| `description` | Text | SEO meta description and social card text. 1-2 sentences, under 160 characters. |
| `date` | `YYYY-MM-DD` | Publication date. Posts are sorted newest-first on the blog index. |
| `author` | Text | Author name as displayed. |
| `category` | One of: `Engineering`, `Company`, `Product` | Shown as a label on blog cards. |
| `readingTime` | Text | e.g., `5 min read`. Estimate based on ~200 words per minute. |

## Categories

- **Engineering** — Technical deep dives, architecture decisions, performance analysis, benchmarks, how-it-works explainers
- **Company** — Company news, team updates, culture, hiring, milestones
- **Product** — Feature announcements, release notes, roadmap updates, use case showcases

## Markdown Features

Standard Markdown is supported:

- **Headings**: `## H2`, `### H3` (don't use `# H1` — the title is the H1)
- **Code blocks**: Triple backticks with language hint (```graphql, ```rust, ```bash, ```typescript)
- **Inline code**: Single backticks for `field names`, `@directives`, `commands`
- **Links**: `[text](url)` — use for references, documentation links
- **Bold/italic**: `**bold**`, `*italic*`
- **Lists**: Bulleted (`-`) and numbered (`1.`)
- **Blockquotes**: `>` for callouts or quotes
- **Images**: `![alt text](url)` — use absolute URLs to hosted images

## Style Guide

- **Tone**: Technical but accessible. Write like you're explaining to a smart colleague, not a professor or a 5-year-old.
- **Length**: 800-1500 words (4-8 min read). Longer is fine for deep dives.
- **Structure**: Start with the problem, show the solution, end with what's next.
- **Code examples**: Include runnable examples where possible. Show the schema, the request, the response.
- **No fluff**: Every sentence should inform or persuade. Cut "In this blog post, we will discuss..."
- **Comparisons**: When comparing to competitors, be factual and specific. Show benchmarks, not opinions.

## Content Ideas (by category)

### Engineering
- Architecture deep dives (how RocksDB sharding works, how the plugin compiler works)
- Performance analysis (benchmark breakdowns, optimization stories)
- How-to tutorials (build X with Yeti in Y minutes)
- Technology explainers (why MessagePack, why Rust, why schema-first)

### Product
- Feature launches (new directives, new capabilities)
- Release notes (what changed, why it matters)
- Use case showcases (real applications built on Yeti)

### Company
- Why we're building this
- Team and culture
- Milestones and roadmap

## Technical Details

- The website fetches from the GitHub raw content API — no authentication needed for public repos
- Posts are cached for 60 seconds on the client side
- Markdown is rendered to HTML via `marked` in the browser
- SEO tags (title, description, Open Graph) are set from frontmatter fields
- The blog index fetches all `.md` filenames from `posts/`, then fetches each file's frontmatter in parallel
