---
description: Ingest a user-written blog draft — refine it with the right writing skill, translate it (en<->zh), auto-tag it, publish a bilingual post pair to _posts/, and commit+push.
argument-hint: <path-to-draft.md | "raw text"> [date:YYYY-MM-DD]
---

# /ingest-article

Take one blog draft the user wrote, distill a private note from it, refine it into the right public form, translate it, tag it, publish it as a bilingual post pair, and ship it. The craft lives in the project skills (`atomic-note`, `distinction-essay`); the publishing mechanics and genre routing live here.

Two things happen every run: (1) an `atomic-note` distillation is **always** saved privately, and (2) the **published** form is **auto-detected** (essay / feature-summary / commentary). You decide the genre from the draft — do not ask the user which skill to use; ask only when genuinely torn between two genres or when the copyright guard trips (Step 3).

`$ARGUMENTS` is the draft: either a path to a markdown/text file, or the raw text pasted inline. An optional `date:YYYY-MM-DD` overrides the publish date (default: today). A trailing phrase like `as a atomic-note` / `as an essay` is an **optional hint** for the published form only — it never cancels the private note in Step 2, and a clear genre/copyright signal in the draft overrides a mismatched hint (say so in one line if you override).

## Step 1 — Read the draft and detect language

- If `$ARGUMENTS` is a path, read it. Otherwise treat the text as the draft.
- Detect the **original language**: `zh` if the body is predominantly Chinese, else `en`. Record it as `ORIG_LANG`; the translation target is the other one (`TARGET_LANG`).

## Step 2 — Always distill a private atomic note

Regardless of what gets published, first run the `atomic-note` skill on the draft to capture its single durable distinction, and save the note to `.claude/knowledge/notes/` (create the dir if needed). Filename `YYYY-MM-DD-<slug>.md`.

This note is a **private** thinking artifact: it is **never committed or pushed** (see Guardrails) and never published as-is unless the user explicitly asks. It's cheap to capture now, expensive to rebuild later, and seeds future essays — link it to sibling notes via `[[name]]`. Always produce it, even when the published form is something else.

## Step 3 — Auto-detect the publish genre, then refine

Decide the public form from the draft's shape. **Detect, don't ask** — proceed with the detected genre and state which in one line. Ask the user (with the candidates) only when two genres are equally plausible, or when the copyright guard below trips.

- **Commentary** *(translation / reproduction guard — check this first)* — if the draft is mostly a **translation or reproduction of someone else's article** (a source URL plus a large block mirroring it), do **not** republish it wholesale; that is a copyright matter. Publish the user's **own** commentary: their take + a short attributed summary, sparse quotes, a prominent canonical link, and **all** original authors credited. Strip scraped cruft (webinar CTAs, ads, "register now"). Only reproduce in full if the user asserts they hold republish rights — confirm that first.
- **Feature-summary** — if the draft **reports a tool / feature / model / release**. Publish a faithful summary (summary → background → use-cases → what-changed → personal take → source). The author's own analysis/forward-look is the valuable part — keep it.
- **distinction-essay** — if the draft is an **original argument** with a thesis or a sharp distinction. Refine via the `distinction-essay` skill.

**Before publishing any genre that cites an external source, fact-check every factual claim against the real source with `WebFetch`** (load it via ToolSearch). Apply the chosen skill's quality bar where it fits: epigraph carries the thesis, one bold load-bearing line per section, diagnosis→prescription, every abstract claim grounded in a scene or number, a committed metaphor family for compounding costs, an explicit non-alarmist beat. Keep the author's voice and real numbers; **never invent facts, scenes, citations, or tags**; correct the draft against the source where they disagree and note the correction.

## Step 4 — Translate

Produce a faithful translation of the refined post into `TARGET_LANG`. Translate prose and headings; **do not** translate code blocks, identifiers, URLs, or the front-matter keys. Keep the two versions structurally parallel (same headings, same code) so the language toggle lands the reader in the same place.

## Step 5 — Identify and add tags

- Read the existing tag vocabulary first: `grep -rh "^    - " _posts/*.md | sort -u` (or scan `tags:` blocks). **Reuse an existing tag** rather than coining a near-duplicate.
- Pick 1–4 tags. Follow the repo convention noted in `tags.html`: Chinese for industry/role/company tags, English for foreign products and technical terms (e.g. `golang`, `clickhouse`).
- Both language versions of the post carry the **same** `tags:` list (this is how one tag page lists both).

## Step 6 — Create the bilingual post pair

Pick a kebab-case `SLUG` from the title and a `DATE` (the override or today). Write two files in `_posts/`:

- Original: `_posts/<DATE>-<SLUG>.md` with `lang: <ORIG_LANG>`
- Translation: `_posts/<DATE>-<SLUG>-<TARGET_LANG>.md` with `lang: <TARGET_LANG>`

Both share the **same** `ref: <SLUG>` — this is what wires the plugin-free language toggle (`_layouts/post.html:45`, `site.posts | where: "ref"`). Front matter for each (copy this shape; `title`/`subtitle` in that file's language):

```yaml
---
layout:     post
title:      "<title in this file's language>"
subtitle:   "<subtitle>"
date:       <DATE>
author:     KL
header-img: img/post-bg-desk.png
catalog: true
tags:
    - <tag1>
    - <tag2>
lang: <zh|en>
ref: <SLUG>
---
```

The `date:` field and the filename date prefix must match. `catalog: true` enables the side TOC, which is built from `##`/`###` headers — so keep real markdown headers.

## Step 7 — Verify (the index is automatic)

There is **no index file to edit**: `tags.html` and `_layouts/home.html` render from Jekyll's built-in `site.tags`, so a correct `tags:` block is all that's needed for "click a tag → see all articles". Verify the build and wiring:

- Run `jekyll build` (or `jekyll serve` if already running) and confirm no errors.
- Confirm both files appear under `_site/`, the chosen tag(s) appear on `_site/tags/index.html`, and each post's rendered page shows the EN/ZH language-switch link (proves `ref` paired correctly).

Report build pass/fail with the actual output. Do not claim success without this check.

## Step 8 — Commit and push

Show the user the two new filenames, the chosen tags, and the proposed commit message first. Then (per repo conventions):

- `unset GITHUB_TOKEN` before any `gh`/git remote operation (an invalid token forces 401).
- Commit on `main` and push via the personal SSH remote (alias `github.com-personal`).
- **No `Co-Authored-By` trailer** — user preference, enforced.
- Commit message: a one-line summary like `Post: <title> (bilingual zh/en) [tags: ...]`.

GitHub Pages rebuilds Jekyll on push; nothing else to deploy.

## Guardrails

- **The private note is never committed.** Stage only the two `_posts/` files (Step 8). `.claude/knowledge/notes/` stays local.
- **Copyright:** never republish someone else's article wholesale without confirmed rights. Translation-heavy drafts default to the **Commentary** genre (Step 3) with attribution + canonical link.
- **Fact-check before publishing:** any claim attributed to an external source must be verified against that source with `WebFetch`; credit all authors; correct the draft where it disagrees with the source.
- One draft = one post pair per run. If the draft contains several distinct arguments, say so and ask whether to split — do not silently merge or drop.
- Never invent scenes, numbers, citations, or tags that aren't grounded in the draft or the existing tag set.
- If you cannot detect the language or genre, or the build fails, stop and report — do not push a broken or half-translated pair.
