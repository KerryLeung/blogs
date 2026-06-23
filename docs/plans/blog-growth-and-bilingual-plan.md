# Blog Growth & Bilingual (EN/ZH) Implementation Plan

> Living checklist — tick `- [ ]` → `- [x]` as steps land.
> Derived from the deep-research synthesis (Tier 1–3 + bilingual architecture).

## Context

This repo is a personal Jekyll tech blog (Hux theme) on GitHub Pages
(`kerryleung.github.io/blogs`). Goal: document learning & AI-assisted-coding
practice, build personal technical influence, and attract more readers. The
priority feature is **hand-written English/Chinese bilingual** content with a
language toggle at both per-post and site level, English as default.

Confidence tags: **[H]** high, **[V]** verify before relying.

## Progress — 2026-06-21

- ✅ **v2 redesign** (cream/terracotta) committed & pushed (`8ddb088`).
- ✅ **This plan** committed (`44b6731`).
- ✅ **Bilingual EN/ZH** shipped via the **plugin-free** path (works on native Pages,
  no Actions): per-post EN⇆ZH toggle + English translation of `golang-fallthrough`
  (`34cc03a`). Verified by build + HTTP round-trip + Playwright snapshot.
- ✅ **SEO trio** (`jekyll-seo-tag` + `jekyll-sitemap` + `jekyll-feed`) + `gems:`→`plugins:`
  + origin-only `url` fix (`d831af4`, completed by `337a2f5`). Verified: clean
  `/sitemap.xml`, `/feed.xml`, single `<title>`, og/twitter/JSON-LD, correct `/blogs` URLs.
- ⏳ **Blocked on your accounts:** Giscus comments, privacy analytics (see Phase 1).
- ⏳ **Deferred:** full polyglot+Actions bilingual upgrade (Phase 2b–2c); site-level
  language switch; retiring the `about.html` hash toggle.

---

## Phase 0 — Commit the v2 redesign  *(done)*

- [x] Review `git status` / `git diff` for the v2 change set
- [x] Commit sources + regenerated CSS + layouts/includes (keep `_config.dev.yml` untracked)
- [x] Resolve push auth (SSH alias `github.com-personal`, account `KerryLeung`) and push

## Phase 1 — Native-safe quick wins  *[H]*

- [x] **SEO meta**: `jekyll-seo-tag`; `{% seo %}` in `head.html`; `author` + `lang` in `_config.yml`
- [x] **Sitemap**: `jekyll-sitemap` (`/sitemap.xml`)
- [x] **Feed**: `jekyll-feed`; `{% feed_meta %}`; removed hand-rolled `feed.xml`
- [x] Migrate deprecated `gems:` → `plugins:`; fix `url` to origin-only
- [ ] **Comments Gitalk → Giscus** — *needs you:* enable GitHub Discussions on the repo,
      install the Giscus app, copy `repo-id` + `category-id` from giscus.app. Then I wire
      the include (and remove the plaintext `clientSecret` Gitalk leaves in `_config.yml`).
- [ ] **Privacy analytics** — *needs you:* pick Cloudflare Web Analytics **or** GoatCounter,
      create the site, hand me the token/snippet. Then I add a gated include and retire GA/Baidu.

## Phase 2 — Bilingual EN/ZH

**Done (plugin-free, native Pages):**
- [x] Per-post EN⇆ZH toggle via shared `ref` + `lang` front matter; graceful no-op when no translation
- [x] English translation of one existing post (`golang-fallthrough`), distinct indexable URL
- [x] Toggle styling (mono pill)

**Deferred (the full SEO-complete upgrade):**
- [ ] Move deploy to GitHub Actions; add `Gemfile` + `jekyll-polyglot`
- [ ] `/zh/`-prefixed indexable URLs; hreflang alternates; per-language canonical
- [ ] Site-level language switch (localStorage), English default
- [ ] Retire the `about.html` hash-based `#en`/`#zh` toggle in favor of real URLs

## Phase 3 — Distribution, search & polish  *(ongoing)*

- [ ] Canonical cross-posting to dev.to / Hashnode (`rel=canonical` home)
- [ ] On-site search (Pagefind)
- [ ] Related posts / series / reading-time; richer tag pages
- [ ] Performance: image optimization, Core Web Vitals
- [ ] Optional: newsletter (Buttondown)

---

## Verification log

- Local build: `jekyll serve --config _config.yml,_config.dev.yml` → `:4000/blogs/`
- Playwright snapshots captured: bilingual toggle (EN), post head SEO DOM check
- Production: GitHub Actions/Pages builds latest commit; `/sitemap.xml` + `/feed.xml` live after deploy

## Notes / risks
- `url` must be **origin-only** (`https://kerryleung.github.io`); `baseurl: /blogs`. The
  `absolute_url` filter = `url + baseurl + page.url`, so putting `/blogs` in `url` double-prefixes.
- `_config.dev.yml` is a local-only build shim (untracked); production uses `_config.yml`.
