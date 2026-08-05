---
name: Track Life Biosciences press releases
description: Pull the company's press releases and news posts from the public WordPress content API, page through the full corpus, and detect what is new since a prior run.
api: openapi/life-biosciences-wordpress-content-openapi.yml
operations: [listCategories, listPosts, getPost]
generated: '2026-08-04'
method: generated
---

# Track Life Biosciences press releases

Life Biosciences publishes corporate news — IND clearances, financings, trial milestones — as
WordPress posts. The whole corpus is readable anonymously as JSON. Use this when you need the
company's announcements as structured data rather than scraped HTML.

## Before you start

- Base URL: `https://www.lifebiosciences.com/wp-json`
- **No credential.** Do not send an `Authorization` header; none is issued or expected.
  (`authentication/life-biosciences-authentication.yml`)
- Be polite: the site's `robots.txt` asks for a `Crawl-delay: 10`, and REST responses carry
  `Cache-Control: max-age=600`. Poll no more than once every 10 minutes; there is no published rate
  limit and no rate-limit header to back off against.

## Steps

1. **Find the news category.** Call `listCategories` with `_fields=id,count,name,slug`. The press
   release category is the one with `slug: press-release` (id `3`, 17 posts as of 2026-08-04). Read
   its `count` — that is your expected corpus size.

2. **Page the corpus.** Call `listPosts` with `categories=<id>`, `per_page=100`, `page=1`, and
   `orderby=date&order=desc`. Read `X-WP-Total` and `X-WP-TotalPages` from the response headers and
   loop `page` until you have consumed them. `per_page` above 100 returns HTTP 400
   `rest_invalid_param` — do not retry it, lower the value.

3. **Keep the payload small.** Add `_fields=id,date,modified,slug,link,title,excerpt,categories` when
   you only need a listing. Add `_embed` instead when you want the featured image and terms inlined
   in a single request.

4. **Fetch full text on demand.** Call `getPost` with the `id` to get `content.rendered`. Titles and
   bodies come back as rendered HTML with HTML entities (`&#8212;`) and non-breaking hyphens in
   product names — normalize `ER‑100` (U+2011) to `ER-100` before matching.

5. **Detect what is new.** On subsequent runs, call `listPosts` with `after=<ISO8601 of your last
   run>` for new posts and `modified_after=<same>` for edits to existing ones. Both parameters are
   declared on the operation. Store the highest `modified` you have seen; there is no ETag or
   Last-Modified to rely on and no webhook to subscribe to.

6. **Cheaper alternative.** If you only need notification and not structure, poll the RSS feed at
   `https://www.lifebiosciences.com/feed/` or `https://www.lifebiosciences.com/news/feed/`.

## Errors

The envelope is `{"code": ..., "message": ..., "data": {"status": ...}}` — **not** RFC 9457
problem+json. Handle:

- `400 rest_invalid_param` — a parameter is out of range; read `data.params` for which one.
- `404 rest_post_invalid_id` — the post id does not resolve; refresh your id list.
- `404 rest_no_route` — you built a path that does not exist; re-read the route index at `/wp-json/`.

Full catalogue: `errors/life-biosciences-problem-types.yml`.

## Do not

- Do not attempt writes. The collections return `Allow: GET` and no credential exists.
- Do not call `/wp/v2/users` — it returns `401 rest_user_cannot_view` anonymously by design.
- Do not treat this as a supported product API. Life Biosciences documents nothing about it and can
  change or close it without notice; there is no deprecation policy, SLA or status page.
