# Tanmay Bangale — tech blog

Plain Jekyll, deployed by GitHub Pages to <https://tanmaybangale.github.io>: no Node, no Actions, no build step.

> **Using an AI agent to publish a post?** Point it at [`AGENTS.md`](AGENTS.md) — it has the full step-by-step workflow, front-matter schema, formatting rules, and confidentiality checks.

## Publish a new post
Add `_posts/YYYY-MM-DD-my-slug.md`:

```md
---
title: "Post title"
description: "One-line summary shown on the landing page."
tags: [llm, agents]
featured: false
---
Markdown here. Code blocks, $$math$$, tables all work.
```

Commit and push to `main`. It's live in about a minute and appears on the landing page automatically.

## Personalise
Edit `_config.yml` (tagline, credentials, links).

## Go live (one time)
1. Create a public repo named **`tanmaybangale.github.io`**.
2. Push this folder to `main`.
3. Repo → Settings → Pages → Source: *Deploy from a branch* → `main` / root.
