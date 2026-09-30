# Blog

Zola site. Writes about cross-chain infrastructure, Rust, and Solidity.

## Daily use

```bash
zola serve              # live preview at http://localhost:1111
zola build              # output to public/
zola serve --drafts     # include posts marked draft = true
```

Write a new post:

```bash
zola new content/blog/my-post.md
```

It opens an editor with this front matter:

```toml
+++
title = "My Post"
date = 2026-09-30
draft = true

[taxonomies]
tags = ["rust", "bridges"]
+++
```

Delete `draft = true` when it's ready to publish.

## Layout

```
content/blog/*.md   your posts (Markdown)
content/about.md    about page
templates/          HTML templates - edit rarely
sass/custom.scss    all styling
zola.toml           site config, edit base_url here
```

`templates/base.html` is the page shell. Everything else fills into it.

## Deploying to Cloudflare Pages

1. Push this repo to GitHub
2. Cloudflare dashboard -> Workers & Pages -> Create -> Pages -> Connect to Git
3. Select the repo
4. Build settings:
   - Build command: `zola build`
   - Output directory: `public`
   - Environment variable: `ZOLA_VERSION` = `0.23.6`
5. Deploy

Cloudflare gives you `https://<project-name>.pages.dev` free, with HTTPS and
free Web Analytics.

Once you buy a real domain, add it in Cloudflare -> Pages -> your project ->
Custom domains, then update `base_url` in `zola.toml`.

## Gotchas

- Zola 0.23 renamed `config.toml` to `zola.toml`
- Sass compiles `sass/custom.scss` to `public/custom.css` (not `style.css`)
- Templates use Tera v2: `{% set %}`, and no `.field` chaining on function calls
- Inside Tera, dates are real dates: use `{{ now | date(...) }}`, not `{{ "now" | date(...) }}`
