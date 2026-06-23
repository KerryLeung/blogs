# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal Jekyll tech blog hosted on GitHub Pages at `https://kerryleung.github.io/blogs/`. It is a fork of the Hux Blog / BY Blog theme. Content is mostly Chinese/English posts on Go, big data, and backend topics. There is no test suite or application code — the "build" is Jekyll site generation plus a Grunt asset pipeline for CSS/JS.

## Commands

Jekyll itself (Ruby) builds the site; the npm toolchain only compiles front-end assets.

- `jekyll serve -w` — local dev server with live reload (default `http://localhost:4000/blogs/`).
- `npm install` — installs the LESS compiler (`less` + `less-plugin-clean-css`). The legacy Grunt devDeps (grunt 0.4.5 / grunt-contrib-less) are kept in `package.json` for history but **do not install on modern Node** — use the `less` path below instead.
- `npm run css` — **the supported CSS build.** Compiles `less/hux-blog.less` → `css/hux-blog.css` and `--clean-css`-minifies → `css/hux-blog.min.css`. **Run after editing any `less/*.less`** — the committed `css/hux-blog.css` / `.min.css` are generated artifacts and the site loads the `.min.css`. (`lessc` emits harmless DEPRECATED warnings about parenthesis-less mixin calls — ignore them.)
- JS (`js/hux-blog.js` → `.min.js`) has no working npm step here; minify manually only if you touch it (the committed `.min.js` is otherwise current).
- `grunt` / `grunt watch` — the original asset pipeline; **non-functional on modern Node**, listed only because older docs reference it.

Deploy: push to the GitHub Pages branch (the user works directly on `main`); GitHub builds Jekyll automatically. The `npm run push`/`boil`/`cafe` scripts reference legacy remotes that may not exist here.

> LESS gotcha: `lessc` 4.x rejects inline `//` comments *inside* a multi-line value (e.g. a `font-family` list). `less/mixins.less` was rewritten to avoid this; keep value lists comment-free or the build breaks.

## Authoring posts

- Posts live in `_posts/` named `YYYY-MM-DD-title.md` (Jekyll requires this date prefix).
- Each post starts with YAML front matter. Copy an existing post's front matter rather than writing from scratch. Key fields:
  - `layout: post`, `title`, `subtitle`, `date`, `author`, `header-img` (e.g. `img/post-bg-desk.png`), `catalog: true` (enables the side-catalog TOC), and a `tags:` list.
- Markdown is **kramdown with GitHub Flavored Markdown** input and **Rouge** highlighting (`_config.yml`). Use GFM fenced code blocks. The catalog is built from `##`/`###` headers, so header IDs matter — this is why kramdown is used instead of redcarpet.
- Tags drive `tags.html`; a tag appears on the homepage only when its count exceeds `featured-condition-size` (currently 1).

## Architecture

- **Layout chain** (`_layouts/`): everything extends `default.html`, which pulls partials from `_includes/` (`head.html`, `nav.html`, `footer.html`). `post.html`, `page.html`, and `keynote.html` are the leaf layouts. `keynote.html` embeds external slide decks.
- **Site config** (`_config.yml`): `baseurl: /blogs` — all internal links use `{{ site.baseurl }}`, so absolute paths must include `/blogs`. Also wires Gitalk (GitHub-issue comments), Google + Baidu Analytics, pagination (10/page), and the sidebar.
- **PWA / offline**: `sw.js` is a service worker (cache-first with cache-busting query strings); `pwa/manifest.json` + `pwa/icons/` provide the installable app shell; `offline.html` is the fallback page. `service-worker: true` in `_config.yml` enables registration.
- **Assets**: edit sources in `less/` and `js/hux-blog.js` only; the `css/` and `*.min.*` outputs are Grunt-generated. Vendored libraries (bootstrap, jquery, etc.) in `css/`/`js/` are not built and should be edited only when upgrading the vendor copy.

## Conventions

- Do not hand-edit generated files (`css/hux-blog.css`, `css/hux-blog.min.css`, `js/hux-blog.min.js`); change the `less/`/`js` source and run `npm run css`. `css/syntax.css` is the one hand-maintained stylesheet (Rouge GitHub light theme) — code-block chrome lives on its `.highlight pre, pre` rule and must stay visually in sync with the LESS code styles.
- `_site/` is the Jekyll build output and is gitignored — never edit or commit it.
- Internal URLs must be prefixed with `{{ site.baseurl }}` (or `/blogs`) because the site is served from a subpath.
