---
name: vaynermedia-harvest-case-studies
description: >-
  Pull VaynerMedia's published client case studies — the agency's own custom post type — with
  their full rendered bodies, categories, tags and hero imagery, from the public WordPress
  content API. Use when you need the agency's own account of what it did for a brand.
api: vaynermedia:wordpress-content-api
base_url: https://vaynermedia.com/wp-json
auth: none
operations:
  - list_case_study
  - get_case_study
  - list_categories
  - list_tags
  - list_media
generated: '2026-08-12'
method: generated
source: openapi/vaynermedia-wordpress-content-openapi.json
---

# Harvest VaynerMedia case studies

`case_study` is a custom post type VaynerMedia registered on its own site, so it is the one
collection on this API that carries agency-specific content rather than WordPress boilerplate.
There were **4** published case studies at last check (2026-08-12).

## Steps

1. **List the case studies.** `list_case_study` — `GET /wp/v2/case_study?per_page=100&_embed`

   `per_page` is capped at 100 and there are only a handful of records, so one request is
   enough. Read `X-WP-Total` to confirm the count rather than assuming the page was complete.

   Adding `_embed` inlines the featured image, author and terms under `_embedded`, which
   saves you the follow-up reads in steps 3 and 4.

2. **Read the body.** Each record carries `title.rendered`, `content.rendered` and
   `excerpt.rendered`. These are **HTML**, not markdown or plain text — strip or render them
   deliberately. `link` is the canonical public URL; cite that, not the API URL.

3. **Resolve the hero image.** `featured_media` is a media id. With `_embed` it is already in
   `_embedded['wp:featuredmedia'][0].source_url`. Without it, call `list_media` filtered by
   `include=<id>`, or `GET /wp/v2/media/{id}`.

4. **Resolve taxonomy.** `categories` and `tags` are arrays of term ids. With `_embed` they
   arrive under `_embedded['wp:term']`. Otherwise call `list_categories` and `list_tags` with
   `include=<comma-separated ids>` — one call each, not one per term.

5. **Fetch a single case study by id** with `get_case_study` — `GET /wp/v2/case_study/{id}` —
   when you already hold the id and only need one record.

## Rules

- **No credentials.** Every operation here is anonymous. Do not attempt to authenticate; the
  write side of these routes is not yours.
- **Do not walk the author edge.** `author` is an integer id, but `GET /wp/v2/users` returns
  HTTP 403 from the WP Engine edge with an HTML body, not a JSON error. Use `_embed` if you
  need author names.
- **Errors are WordPress-shaped, not RFC 9457.** Branch on `code`, never on `message`:
  `rest_no_route` (404, wrong path or method), `rest_invalid_param` (400, read
  `data.params`), `rest_post_invalid_id` (404, the id does not resolve). See
  `errors/vaynermedia-problem-types.yml`.
- **`per_page` above 100 is a 400**, not a clamp. `rest_invalid_param` with
  `data.details.per_page.code = rest_out_of_bounds`.
- **There is no rate-limit signal.** No `RateLimit-*`, no `Retry-After`, no documented 429.
  Responses are edge-cached for 600s. Pace yourself — the site's own `robots.txt` asks for a
  10-second crawl delay — and treat any non-JSON body as an edge response rather than an API
  error.
- **Ids are not type-scoped.** `posts`, `pages`, `case_study` and `popups` all draw ids from
  the same sequence. Always carry the collection alongside the id.
