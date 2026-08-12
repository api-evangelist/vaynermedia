---
name: vaynermedia-search-content
description: >-
  Search everything VaynerMedia publishes — posts, pages, case studies, media — from one
  endpoint, then fetch the full record from whichever collection the hit came from. The
  cheapest entry point into the agency's public content when you do not know which content
  type holds the answer.
api: vaynermedia:wordpress-content-api
base_url: https://vaynermedia.com/wp-json
auth: none
operations:
  - list_search
  - list_types
  - list_posts
  - get_posts
  - list_pages
  - get_pages
  - list_case_study
generated: '2026-08-12'
method: generated
source: openapi/vaynermedia-wordpress-content-openapi.json
---

# Search VaynerMedia content

`GET /wp/v2/search` is the only operation on this API that spans content types. It returned
**342** indexed records on 2026-08-12 across posts, pages, case studies and media.

## Steps

1. **Discover what exists.** `list_types` — `GET /wp/v2/types` — returns every registered post
   type with its `rest_base`. Do this once per session rather than guessing collection names.
   It is how you learn that this site carries `case_study` and `salient_g_sections` on top of
   WordPress core's types.

2. **Search.** `list_search` — `GET /wp/v2/search?search=<terms>&per_page=100`

   Narrow with `type` (`post`, `term`, `post-format`) and `subtype` (`post`, `page`,
   `case_study`, `attachment`) when you already know the shape you want.

   Each hit is deliberately thin: `id`, `title`, `url`, `type`, `subtype`. It is an index,
   not a content response.

3. **Fetch the real record.** `subtype` names the collection. Map it and call the item
   operation:

   | `subtype` | operation | path |
   |---|---|---|
   | `post` | `get_posts` | `GET /wp/v2/posts/{id}` |
   | `page` | `get_pages` | `GET /wp/v2/pages/{id}` |
   | `case_study` | `get_case_study` | `GET /wp/v2/case_study/{id}` |
   | `attachment` | `get_media` | `GET /wp/v2/media/{id}` |

   Add `_embed` on the item fetch to pull author, featured media and terms in the same round
   trip.

4. **Batch instead of looping.** When you have several ids from the same subtype, use the
   collection with `include=<comma-separated ids>` — one request instead of N.

5. **Trim the payload.** `_fields=id,title,link,date,excerpt` cuts a `list_posts` response
   by an order of magnitude. The full post object carries rendered HTML and a large
   `yoast_head_json` block you almost never need.

## Rules

- **Page with `page` + `per_page`, bounded 1-100.** Read `X-WP-Total` and `X-WP-TotalPages`
  from the response headers, or follow the `Link: rel="next"` header. Both are CORS-exposed.
- **`search` matches title, content and excerpt.** Use `search_columns` to restrict it.
  Combine with `after` / `before` / `modified_after` / `modified_before` for date windows,
  and `orderby=relevance` when searching (the default is `date`).
- **Anonymous only.** Do not authenticate. `templates`, `menu-items` and `settings` return
  401 for anonymous callers and are outside this skill; `users` returns an HTML 403 from the
  edge.
- **Errors are WordPress-shaped, not RFC 9457** — `{code, message, data.status}`. Branch on
  `code`. See `errors/vaynermedia-problem-types.yml`.
- **No idempotency keys, no rate-limit headers, no request id.** Everything here is a safe
  GET, so retry freely, but you get no runtime budget signal and nothing to quote in a
  support ticket. Responses are edge-cached for 600 seconds.
- **This is a marketing site's content API, not a product API.** It is the right tool for
  "what has VaynerMedia published about X" and the wrong tool for anything transactional —
  there is nothing transactional here.
