# viktr

Zola site. Notes on wire formats, networking, erasure coding, and light clients.

**Live:** https://victorchukwuemeka.github.io/viktr

## Daily use

```bash
./dev      # local preview at localhost:1111, drafts included
./build    # production build into public/
```

Write a new post:

```bash
./new-post my-title
```

Front matter:

```toml
+++
title = "My Post"
date = 2026-09-30
draft = true

[taxonomies]
tags = ["rust", "networking"]
+++
```

Delete `draft = true` when it's ready, then:

```bash
git commit -am "publish post" && git push
```

GitHub Actions builds and deploys. Live in about 30 seconds.

## IMPORTANT: why ./dev exists

This site is served from a **subpath** (`/viktr`), not a domain root. So all
internal links go through Zola's `get_url()`, which reads `base_url` from
`zola.toml`.

That means `base_url` has to be correct for whatever you're doing:

| Command | base_url used | Links point to |
|---|---|---|
| `./build`, or a plain `zola build` | `https://victorchukwuemeka.github.io/viktr` | live site |
| `./dev` | `http://localhost:1111` | your machine |

If you run plain `zola serve` without `--base-url`, it generates **production**
URLs, so localhost will load CSS and pages from the deployed site and you won't
see your local edits. That's the failure mode `./dev` exists to prevent.

Do not hardcode `/custom.css` or `/blog/` in templates. Use
`{{ get_url(path='/blog/') }}`. Root-relative paths break under a subpath.

## Layout

```
content/blog/*.md   your posts (Markdown)
content/about.md    about page
templates/          HTML templates - edit rarely
sass/custom.scss    all styling
zola.toml           site config, base_url lives here
.github/workflows/  auto-deploy to GitHub Pages
```

## Deploying

Push to `main`. GitHub Actions runs `./build` and publishes to Pages.

- Build command: `zola build`
- Output: `public/`
- Zola version: `0.23.6` (pinned in `.github/workflows/deploy.yml`)

## Getting a real domain later

1. Buy it (a `.dev` is ~$12/yr at Cloudflare Registrar, no markup)
2. Add it to the repo as a CNAME, and in GitHub: Settings → Pages → Custom domain
3. Change `base_url` in `zola.toml` to the bare domain
4. Commit

## Zola 0.23 gotchas

Hit all of these while setting this up. Most tutorials online are for older versions.

- Config file is `zola.toml`, not `config.toml`
- `lang` → `default_language`
- `author` is a plain string, not a table
- `taxonomies = [{ name = "tags", feed = true }]`, and front matter needs a
  `[taxonomies]` section
- Tera v2: `{% set %}` works, but no `.field` chaining on a function call —
  `get_section(...).pages` errors, needs an intermediate variable
- `slice` filter is gone — use `posts[:10]`
- `date` filter needs a real date, not a string: `{{ now | date(...) }}` not
  `{{ "now" | date(...) }}`
- Sass compiles `sass/custom.scss` → `public/custom.css`, not `style.css`
- A `{% block %}` can only appear once per template, so you can't reuse one for
  both `og:title` and `twitter:title`
- `taiki-e/install-zola-action` no longer exists; the workflow downloads the
  binary from the release instead
- `zola new` is gone in 0.23 (only init/build/serve/check remain). `./new-post`
  writes the front matter instead
- Blog posts render with `templates/page.html`, NOT `single.html`
- Section indexes render with `templates/section.html`, NOT a per-section name
- `page.summary` in front matter is silently ignored; use `description`
- `table_of_contents()` does not exist; `page.toc` is JSON, not HTML
- `{% macro %}` must be declared before use, or Tera errors on the whole build
- Any `markdown.highlighting.theme` gets inlined as an HTML style attribute that
  beats your stylesheet
