---
name: Browse the Hydrostor A-CAES project portfolio
description: >-
  Retrieve Hydrostor's Advanced Compressed Air Energy Storage projects — Goderich, Willow Rock,
  Silver City, Quinte, Wellington and Copper Valley — using the public read-only content API behind
  hydrostor.ca. No credentials required.
api: openapi/hydrostor-projects-api-openapi.yml
operations: [search, listProjects, getProject, getMediaItem, getSeoHead]
---

# Browse the Hydrostor A-CAES project portfolio

Hydrostor publishes no developer program. The only machine-readable surface is the WordPress REST
content API behind its marketing site, and the one genuinely company-specific resource on it is the
`project` custom post type — the A-CAES facilities Hydrostor has built or is developing.

Base URL: `https://hydrostor.ca/wp-json`

## Authentication

None. Every operation below is anonymous over HTTPS. Do not send an `Authorization` header — there
is no public credential to send, and the write methods that would use one are staff-only and answer
`401 rest_cannot_create`.

## Steps

1. **List the portfolio** — `listProjects`
   `GET /wp/v2/project?per_page=100&_fields=id,slug,title,excerpt,link,date,modified`
   All records fit in one page — there were 6 published projects on 2026-08-22. `title` and
   `excerpt` are objects shaped `{"rendered": "..."}`, not strings; read `.rendered` and unescape
   HTML entities (the API returns `world&#8217;s`, not `world's`).

2. **Retrieve one project in full** — `getProject`
   `GET /wp/v2/project/{id}`
   The full record carries the rendered page body in `content.rendered`, which is where the
   capacity, duration, location and status of the facility actually live — there are no structured
   fields for them. Add `_embed` to inline the featured image and author in the same round trip.

3. **Resolve imagery** — `getMediaItem`
   `GET /wp/v2/media/{featured_media}`
   Read `source_url` for the original and `media_details.sizes` for the generated variants. Note
   that `featured_media` is `0` on some projects (observed on Goderich, id 10825), meaning no
   featured image is attached — check before you dereference.

4. **Get structured metadata without parsing HTML** — `getSeoHead`
   `GET /yoast/v1/get_head?url=https://hydrostor.ca/project/{slug}/`
   Returns the rendered head plus a parsed schema.org `@graph` for the project page. This is the
   cheapest structured view of a project and avoids scraping `content.rendered`.

5. **Find a project by name instead of id** — `search`
   `GET /wp/v2/search?search=goderich&per_page=5`
   Returns `{id, title, url, type, subtype}`. Filter on `subtype == "project"` — the same query also
   matches newsroom posts about the project, which is usually useful rather than noise.

## Conventions that will bite you

- **The project taxonomies are empty.** `project_category` and `project_tag` are registered and
  readable, and both are accepted as filter parameters, but they carried **zero terms** on
  2026-08-22 and every project record returns `[]` for both. You cannot filter projects by region,
  market or status through the API — parse `content.rendered` or use the human page.
- **Pagination**: `per_page` is capped at 100. Exceeding it returns HTTP 400 `rest_invalid_param`
  with a `data.details.<param>` block, not a clamped result. Read `X-WP-Total` and
  `X-WP-TotalPages` for the record count.
- **Always send `_fields`**: an unfiltered project record carries the entire rendered page body.
- **Ids are not typed**: post, page, media and project ids share one integer sequence. Carry the
  resource type with the id or you will fetch the wrong object. `search` is the only endpoint that
  returns both.
- **Errors**: match on `code`, never on `message`. `404 rest_post_invalid_id` means "not readable",
  which includes unpublished — it does not prove the record does not exist. Full catalog in
  `errors/hydrostor-problem-types.yml`.
- **No rate-limit signal**: no `RateLimit-*` or `Retry-After` header is ever returned. Self-throttle.
- **No idempotency, no versioning commitment, no status page, no changelog.** Treat this surface as
  unmanaged and cache what you read; see `lifecycle/hydrostor-lifecycle.yml`.
