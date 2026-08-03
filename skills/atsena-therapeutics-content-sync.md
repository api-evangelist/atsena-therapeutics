---
name: Mirror the Atsena Therapeutics site content
description: Do a complete, resumable pull of every anonymously readable record on the Atsena Therapeutics WordPress REST API — posts, pages, media and taxonomy — with correct pagination, incremental sync and the gated-surface stops.
api: openapi/atsena-therapeutics-wp-rest-openapi.yml
operations: [getRouteIndex, listTypes, listTaxonomies, listStatuses, listPosts, listPages, listMedia, listCategories, listTags, searchContent, getOembed]
generated: '2026-08-02'
method: generated
source: openapi/atsena-therapeutics-wp-rest-openapi.yml
---

# Mirror the Atsena Therapeutics site content

A complete pull of everything atsenatx.com exposes without credentials. Useful for building a local
index, a RAG corpus or a change monitor over a clinical-stage company's public record.

**Base URL:** `https://atsenatx.com/wp-json`
**Auth:** none — and none is obtainable. See "Where to stop" below.

## Budget

The entire anonymous corpus, measured 2026-08-02:

| Collection | Operation | Records |
|---|---|---|
| Posts | `listPosts` | 55 |
| Pages | `listPages` | 29 |
| Media | `listMedia` | 176 |
| Categories | `listCategories` | 2 |
| Tags | `listTags` | 0 |
| Search index | `searchContent` | 72 |

At `per_page=100` that is **six requests** for a full mirror. This is a small corpus — do not build a
distributed crawler for it, and do not re-pull it more than once a day.

## 1. Discover before you pull

`getRouteIndex` — `GET /`
`listTypes` — `GET /wp/v2/types`
`listTaxonomies` — `GET /wp/v2/taxonomies`
`listStatuses` — `GET /wp/v2/statuses`

Take the type list from the server rather than assuming. If a custom type is ever added with
`show_in_rest` enabled it will appear in `listTypes` with its `rest_base`, and your mirror should pick
it up without a code change. As of this writing the set is stock WordPress only.

Also read `page_on_front` from the route index — it names the live homepage record (id 5), which
matters because a second published homepage (`home-new`, id 1614) also exists.

## 2. Page every collection correctly

`GET /wp/v2/{rest_base}?per_page=100&page={n}&orderby=id&order=asc`

- `per_page` max is **100**; more returns `400 rest_invalid_param`.
- Loop until `page == X-WP-TotalPages`. Paging past the end returns `400
  rest_post_invalid_page_number`, **not** an empty array — an "until empty" loop will log a spurious
  error on the last iteration.
- Order by `id asc` for a stable mirror. Ordering by `date` while records are being edited can
  duplicate or skip rows across page boundaries.
- Trim with `_fields` to only what you store. Full `content.rendered` on 176 media records is mostly
  wasted bytes.

## 3. Sync incrementally after the first pull

`GET /wp/v2/posts?modified_after={iso8601}&orderby=modified&order=asc&per_page=100`

Works on posts, pages and media. Persist the highest `modified` seen and pass it next run. Use
`modified`, not `date`.

Deletions are invisible this way — a removed record simply stops appearing. If you need
delete-detection, periodically re-pull the id list with `_fields=id,modified` (three cheap calls) and
diff against your store.

## 4. Store the right fields

- `title`, `content`, `excerpt`, `caption`, `guid` are **objects** with a `rendered` string. Store
  `.rendered`.
- `content.rendered` is HTML, not markdown or text.
- `acf` is a free-form object whose shape varies per record and is not schematised by the provider —
  store it opaquely, do not assume keys.
- `yoast_head_json` carries useful metadata the rest of the API does not expose, including
  `author` (a name string), `article_published_time` and `canonical`.
- Keep `link` as the citable public URL.

## 5. Cheap enrichment

`getOembed` — `GET /oembed/1.0/embed?url={public link}`

Returns `{version, provider_name, provider_url, author_name, title, type, html}` for any page or post
URL. This is the supported anonymous route to an author name, given that the users collection is
closed.

Note `/oembed/1.0/proxy` is **not** available — it returns `401 rest_forbidden`.

## Where to stop — gated surfaces

These return `401` to every anonymous caller. They are permanent, not transient: **do not retry, do
not back off and retry, do not treat them as an outage.**

| Path | Status | `code` |
|---|---|---|
| `/wp/v2/users` | 401 | `rest_user_cannot_view` |
| `/wp/v2/settings` | 401 | `rest_forbidden` |
| `/wp/v2/plugins` | 401 | `rest_forbidden` |
| `/wp/v2/templates` | 401 | `rest_forbidden` |
| `/wp/v2/menus` | 401 | `rest_forbidden` |
| `/wp-abilities/v1/abilities` | 401 | `rest_forbidden` |
| `/oembed/1.0/proxy` | 401 | `rest_forbidden` |
| `/wp/v2/comments` | 403 | `rest_comment_disabled` |
| `?context=edit` on any route | 401 | `rest_forbidden_context` |

The only credential this surface understands is a WordPress application password minted inside
wp-admin. There is no self-service signup and no developer programme — a third party cannot obtain
one. Treat the gated set as permanently out of scope.

**Never attempt a write.** Every write route is registered but administrative. This is a live
production site belonging to a clinical-stage company; the anonymous surface is read-only by design
(`OPTIONS /wp/v2/posts` returns `Allow: GET`).

## Politeness

- No published rate limit. robots.txt requests `Crawl-delay: 10` — honour it.
- `cache-control: max-age=600`; cache accordingly.
- Item routes send `Last-Modified` but **no** `ETag` — revalidate with `If-Modified-Since`, not
  `If-None-Match`.
- Cloudflare and WP Engine front the origin and may throttle without warning. Back off on 429/503.

## Content handling

This corpus is a gene therapy company's public communications: press releases, programme pages and
patient-facing material about inherited retinal diseases. If you index it for retrieval, keep the
`link` and `date` on every chunk so answers stay citable and dateable, and never let it be presented
as medical advice or as trial-enrollment guidance.
