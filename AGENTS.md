# Agent Guide: Publishing a Blog Post

This is a plain **Jekyll** blog deployed by **GitHub Pages** to <https://tanmaybangale.github.io>. There is no Node, no bundler, no GitHub Actions workflow — pushing to `main` builds and deploys automatically in ~60 seconds.

When the user hands you blog content (rough notes, a draft, a doc, or a topic), follow the steps below.

---

## 1. Create the post file

Create `_posts/YYYY-MM-DD-url-slug.md`:

- **`YYYY-MM-DD`**: Use today's date (in the user's local timezone) unless the user specifies another date. Note: Jekyll skips posts dated in the future unless `future: true` is set, so do not use a future date.
- **`url-slug`**: Lowercase, hyphen-separated, short and specific (e.g. `mcp-gateway-over-legacy-erps`). The live URL will be `/blog/<url-slug>/`.

### Required front matter

```yaml
---
title: "Concrete, specific title (avoid clickbait)"
description: "One or two sentences shown under the title on the home page and in link previews."
tags: [agents, mcp, evals]   # 2–4 lowercase tags; rendered as #tag on the site
featured: false              # set true only if the user asks to feature it
---
```

Do **not** set `layout:` (it defaults to `post` via `_config.yml`). Do **not** include `published: false` unless the user asks for a draft.

---

## 2. Format the body (Kramdown + Rouge + MathJax)

Write the body directly below the closing `---`. Do **not** repeat the `<h1>` title in the body — `_layouts/post.html` already renders `page.title`, the date, read time, and tags at the top.

Supported elements (styled in `assets/style.css`):

- **Section headings**: start at `##` (`<h2>`) and `###` (`<h3>`).
- **Fenced code blocks** with a language tag (`python`, `bash`, `json`, `sql`, `yaml`, `go`, etc.):
  ````markdown
  ```python
  def step(state):
      return llm(state)
  ```
  ````
- **Inline code**: `` `identifier` ``.
- **Math** (MathJax v3 is loaded on post pages): inline `$$p^n$$` or display:
  ```markdown
  $$
  P(\text{success}) = p^{n}
  $$
  ```
  *(Kramdown uses `$$...$$` for both inline and block math; put block math on its own lines surrounded by blank lines.)*
- **Tables**: standard GitHub-flavoured Markdown tables.
- **Blockquotes**: `> callout or takeaway`.
- **Images / diagrams**: save files under `assets/img/<post-slug>/...` and reference them as `![caption]({{ '/assets/img/<post-slug>/file.png' | relative_url }})`.

### House style & confidentiality check

- Tone is **engineering notebook**: direct, concrete, numbers and runnable code over hype.
- The author works at Google (previously AWS). **Never include employer-confidential details, internal tool names, or identifiable customer names** unless the user explicitly confirms they are public. Anonymise client engagements ("a global BPO", "a quick-commerce platform") while keeping the architecture and numbers intact.

---

## 3. Update `preview.html` (only if previewing locally without Jekyll)

`preview.html` is a standalone static mockup of the home page (excluded from the Jekyll build in `_config.yml`) used when Jekyll isn't installed locally.

- On the live site, `index.html` loops over `site.posts` automatically — **you do not need to edit `index.html`**.
- If the user wants to see the new post listed in the local `preview.html`, prepend a `<li>` entry inside `<ul class="posts">` matching the existing structure (date, `.p-title`, `.p-desc`, `.p-meta` with read time `words / 220 + 1` and `#tags`).
- While removing the sample post for the first real publish, also delete `_posts/2026-10-10-react-agent-from-scratch.md` (currently `published: false`) if the user no longer wants it.

---

## 4. Publish (git push)

Once the user approves the post:

```bash
git status
git add _posts/YYYY-MM-DD-url-slug.md assets/img/   # include any images added
git commit -m "post: <short title>"
git push origin main
```

If this directory is not yet a git repo connected to `tanmaybangale/tanmaybangale.github.io`:

```bash
git init -b main
git remote add origin https://github.com/tanmaybangale/tanmaybangale.github.io.git
git add .
git commit -m "Initial blog"
git push -u origin main
```

---

## Key files in this repo

| File | Purpose |
|---|---|
| `_config.yml` | Site title, tagline, credentials, social links, permalink (`/blog/:title/`), exclude list |
| `_layouts/default.html` | Outer shell: grid background, sticky terminal top bar, footer |
| `_layouts/post.html` | Post template: back link, title, date, read-time estimate, tags, prose container, MathJax |
| `index.html` | Home page: intro + automatic list of all published posts in `_posts/` |
| `assets/style.css` | All styles (graph-paper background, typography, Rouge syntax colours) |
| `preview.html` | Local static preview of the home page (not deployed) |
