---
title: Test Post — Blog Infrastructure Verified
description: A quick test to confirm the blog sync pipeline works end to end.
date: 2026-04-14
author: Yeti Team
category: Engineering
readingTime: 1 min read
---

This is a test post to verify the blog sync pipeline. If you can read this, the following works:

- GitHub repo fetch with PAT authentication
- Markdown frontmatter parsing
- HTML rendering
- Date-based publish filtering
- Table storage and retrieval via extends_table!

## What Was Built

The blog infrastructure syncs posts from a private GitHub repo into yeti's RocksDB storage. Posts are markdown with YAML frontmatter. The `date` field controls when a post goes live — future-dated posts are invisible to readers.

## Delete Me

This post exists only to verify the pipeline. Delete it after confirming everything works.
