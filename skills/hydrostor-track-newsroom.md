---
name: Track the Hydrostor newsroom
description: >-
  Read and monitor Hydrostor's press releases, company insights and featured media coverage through
  the public read-only content API behind hydrostor.ca. No credentials required.
api: openapi/hydrostor-posts-api-openapi.yml
operations: [listCategoryTerms, listPosts, getPost, listAuthors, getSeoHead, getOembed]
---

# Track the Hydrostor newsroom

Hydrostor's newsroom is the richest part of its public surface — 172 published posts on 2026-08-22,
covering funding rounds, project milestones, regulatory filings and press coverage. The human view
is at https://hydrostor.ca/media/ and there is an RSS feed at https://hydrostor.ca/feed/, but only
this API gives you the full archive with filtering, pagination and category joins.

Base URL: `https://hydrostor.ca/wp-json`

## Authentication

None. Anonymous over HTTPS.

## Steps

1. **Get the category vocabulary first** — `listCategoryTerms`
   `GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count`
   Four categories exist, and three of them are real: **Featured Coverage** (id 8, 96 posts),
   **Press Release** (id 9, 54 posts), **Insights** (id 34, 24 posts) and Uncategorized (id 1, 0).
   The distinction matters — Press Release is Hydrostor speaking, Featured Coverage is third-party
   journalism republished under Hydrostor's byline.

2. **List posts, filtered** — `listPosts`
   `GET /wp/v2/posts?categories=9&per_page=20&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,categories,author,featured_media`
   Add `after=2026-01-01T00:00:00` for a date window, or `modified_after=` to catch edits.
   `X-WP-Total` gives the matching count before you page through it.

3. **Poll for new items** — the same call with `after=<last-seen-date>` and `per_page=1` is a cheap
   change check: read `X-WP-Total` and stop if it is `0`. There is no ETag or `Last-Modified` on
   these responses, so conditional requests are not available; poll politely.

4. **Retrieve one post** — `getPost`
   `GET /wp/v2/posts/{id}`
   `content.rendered` is the article body as HTML. `excerpt.rendered` is a usable summary if you
   only need the gist.

5. **Attribute it** — `listAuthors`
   `GET /wp/v2/users?_fields=id,name,slug,link`
   Three public authors. Anonymous callers get the `view` projection only; email and role require
   authentication and return `401 rest_forbidden`.

6. **Get link metadata cheaply** — `getOembed` or `getSeoHead`
   `GET /oembed/1.0/embed?url=<post url>` returns title, author, thumbnail and embed HTML.
   `GET /yoast/v1/get_head?url=<post url>` returns the full head plus a parsed schema.org graph —
   use this when you need the canonical URL, publish date and Open Graph image without fetching the
   article.

## Conventions that will bite you

- **Tags are empty.** The `post_tag` taxonomy is registered and every post returns a `tags` array,
  but there are **zero terms** — categories are the only working classification.
- **`per_page` maxes at 100**; over that is a 400, not a clamp.
- **Send `_fields`.** A full post record is dominated by `content.rendered` and `yoast_head_json`.
- **Dates are site-local**, not UTC: the site reports `gmt_offset: -4` / `America/New_York`. The
  `date` field is local; use `date_gmt` when comparing across sources.
- **Errors**: match on `code`. See `errors/hydrostor-problem-types.yml`.
- **No rate-limit headers, no changelog, no deprecation policy.** See
  `conventions/hydrostor-conventions.yml` and `lifecycle/hydrostor-lifecycle.yml`.
