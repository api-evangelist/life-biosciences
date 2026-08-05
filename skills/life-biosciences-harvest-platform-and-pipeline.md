---
name: Harvest the Life Biosciences platform and pipeline pages
description: Resolve and retrieve the company's science pages — platform, pipeline, ER-100, publications, leadership — as structured JSON via search and the pages collection, instead of scraping the site.
api: openapi/life-biosciences-wordpress-content-openapi.yml
operations: [search, listPages, getPage, listTypes, listMedia]
generated: '2026-08-04'
method: generated
---

# Harvest the Life Biosciences platform and pipeline pages

The company's science content — the epigenetic restoration platform, the ER-100 program, the MASH
preclinical work, publications, leadership and the scientific advisory board — lives in WordPress
pages, all readable anonymously as JSON.

## Before you start

- Base URL: `https://www.lifebiosciences.com/wp-json`
- No credential. See `authentication/life-biosciences-authentication.yml`.
- **Know the shape of the corpus first.** Call `listTypes`. The site registers only stock WordPress
  types (`post`, `page`, `attachment`, …) — there is **no** custom post type for pipeline programs,
  publications or clinical trials. That means program-level facts are unstructured HTML inside page
  bodies; you must parse them out yourself, and you should say so when you cite them.

## Steps

1. **Locate by term.** Call `search` with `search=ER-100` (or `pipeline`, `epigenetic
   restoration`, `MASH`). You get `{id, title, url, type, subtype}` and a HAL `_links.self` pointing
   at the full record. `subtype` tells you whether to follow up on `pages` or `posts`.

2. **Or enumerate directly.** Call `listPages` with `per_page=100` and
   `_fields=id,slug,link,title,parent,menu_order`. Observed slugs worth pinning:
   `our-platform`, `targeting-the-biology-of-aging`, `publications`, `pipeline`,
   `optic-neuropathies-er-100`, `about-life-biosciences`, `leadership`,
   `scientific-advisory-board`, `board-of-directors`, `co-founders`, `news`, `press-releases`,
   `in-the-media`, `contact`, `join-us`, `terms-of-use`, `privacy-policy`.
   You can also fetch one directly with `listPages?slug=<slug>` rather than by id — slugs are stable
   across content edits, ids are not guaranteed to be.

3. **Retrieve the body.** Call `getPage` with the id and read `content.rendered`. Add `_embed` to
   inline the featured media and terms in the same request.

4. **Walk the hierarchy.** Pages nest via `parent` (for example `press-releases` and `in-the-media`
   sit under `news`). Build the tree from `parent` + `menu_order` to reproduce the site's
   information architecture.

5. **Collect figures and PDFs.** Call `listMedia` with `parent=<page id>` (or `search=`) and read
   `source_url`, `mime_type` and `alt_text`. Publications are commonly linked out to journals rather
   than hosted, so expect external links inside `content.rendered`.

6. **Two draft pages exist.** `test7` (Board of Directors) and `test8` (Co-Founders) are duplicate
   working copies of the canonical `/about-us/` pages and are publicly listed. Prefer the
   `/about-us/...` slugs and ignore the `test*` ones.

## Errors

WordPress envelope `{code, message, data.status}` — not RFC 9457. `400 rest_invalid_param` on an
out-of-range `per_page`, `404 rest_post_invalid_id` on a stale id, `404 rest_no_route` on a bad
path. See `errors/life-biosciences-problem-types.yml`.

## Do not

- Do not infer trial status, dosing or regulatory state from page HTML without quoting it — this is
  clinical-stage biopharma content and the page body is the only authority.
- Do not attempt writes or privileged reads; the surface is `Allow: GET` and `/wp/v2/users` is 401.
