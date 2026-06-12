# HomelabForge Blog

Source for [blog.homelabforge.dev](https://blog.homelabforge.dev) — *Production DevOps for Homelabs*.

Built with [Hugo](https://gohugo.io/) (extended) and [PaperMod](https://github.com/adityatelange/hugo-PaperMod), deployed on [Cloudflare Pages](https://pages.cloudflare.com/). Migrated from Hashnode in June 2026.

## Stack

| | |
|---|---|
| Engine | Hugo extended `0.163.1` (pinned via `HUGO_VERSION`) |
| Theme | PaperMod (git submodule) + brand overrides in `assets/css/extended/` |
| Hosting | Cloudflare Pages — build `hugo --minify`, output `public/` |
| Diagrams | Mermaid 11 (self-hosted, fingerprinted + SRI) via code-block render hook, loaded only on pages that use it |
| Feed | Full-content RSS at [`/rss.xml`](https://blog.homelabforge.dev/rss.xml) |

## Local development

```bash
git clone --recurse-submodules https://github.com/malpanez/homelabforge-blog.git
cd homelabforge-blog
hugo server -D        # serves drafts at http://localhost:1313
```

## Writing a post

```bash
hugo new content posts/my-new-post/index.md
```

Posts are [page bundles](https://gohugo.io/content-management/page-bundles/): each post lives in its own folder with its images next to `index.md`, referenced relatively. The archetype pre-fills the front matter (`title`, `date`, `slug`, `tags`, `series`, `description`, `cover`, `draft: true`).

Code blocks are highlighted with the `github-dark` style. Mermaid diagrams work out of the box:

````markdown
```mermaid
flowchart LR
    A[Write] --> B[PR] --> C[Preview] --> D[Merge] --> E[Live]
```
````

## Publishing flow

1. Branch + commit + push, open a PR.
2. Cloudflare Pages builds a preview URL for the PR.
3. Review, merge to `main` → production deploy.

To publish a draft, flip `draft: false` and update `date`.

## Layout notes

- `layouts/rss.xml` — custom full-content feed served at `/rss.xml` (Hashnode-compatible path, no redirect).
- `layouts/_markup/render-codeblock-mermaid.html` — renders ` ```mermaid ` fences and injects mermaid.js once per page. The loader lives in the render hook (not `extend_footer`) because PaperMod caches the footer partial across pages.
- `layouts/baseof.html`, `layouts/_partials/templates/opengraph.html` — minimal copies of PaperMod templates with deprecated `.Language.*` calls fixed for a zero-warning build.
- `static/_redirects` — Cloudflare Pages redirects (legacy `/archive` → `/archives/`).

## License

Content © Miguel Alpañez. Code snippets in posts are MIT unless stated otherwise.
