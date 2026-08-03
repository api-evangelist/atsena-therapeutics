---
name: Track Atsena Therapeutics news and regulatory milestones
description: Incrementally pull Atsena Therapeutics press releases and company news from the site's WordPress REST content API, segmented by category, with modified_after sync and image resolution — without scraping HTML.
api: openapi/atsena-therapeutics-wp-rest-openapi.yml
operations: [listCategories, listPosts, getPost, listMedia, getMediaItem]
generated: '2026-08-02'
method: generated
source: openapi/atsena-therapeutics-wp-rest-openapi.yml
---

# Track Atsena Therapeutics news and regulatory milestones

Atsena Therapeutics publishes no developer programme. Its announcement record is nonetheless
available as JSON through the WordPress REST API the site serves at `https://atsenatx.com/wp-json`.
Use it instead of scraping `https://atsenatx.com/news/press-releases/`.

For a clinical-stage gene therapy company this collection *is* the material record: trial initiation
and dosing, readouts, regulatory designations (FDA and EMA), partnerships and financing all land here
first.

**Base URL:** `https://atsenatx.com/wp-json`
**Auth:** none. Every call below is anonymous; do not send credentials.

## 1. Resolve the category ids (once, then cache)

`listCategories` — `GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count`

Two terms exist. As of 2026-08-02: **Press Release** `id 4` (52 posts) and **Company News** `id 6`
(3 posts). Re-resolve by `slug` rather than hardcoding the ids.

The `tags` taxonomy is registered but holds **zero terms** — `listTags` correctly returns `[]`. Do not
treat that as a failure, and do not build a tag-based filter.

## 2. Page the stream

`listPosts` — `GET /wp/v2/posts?per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,categories,featured_media`

- `per_page` maximum is **100**; asking for more returns `400 rest_invalid_param` with
  `data.params.per_page` naming the bound.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, or follow the RFC 8288 `Link`
  header's `rel="next"`. Requesting a page beyond the last returns `400
  rest_post_invalid_page_number` — stop when `page == X-WP-TotalPages`, do not probe past it.
- `title`, `excerpt` and `content` are **objects** carrying a `rendered` HTML string. Read
  `title.rendered`, not `title`.
- Add `&categories=4` to isolate press releases from the handful of company-news items.

## 3. Sync incrementally

`listPosts` — `GET /wp/v2/posts?modified_after=2026-07-01T00:00:00&orderby=modified&order=desc`

Persist the highest `modified` you have seen and pass it as `modified_after` on the next run. Use
`modified`, not `date`: a press release that is corrected after publication keeps its original `date`
but gets a new `modified`, and a `date`-based sync would miss the correction.

`after` / `before` filter on publication date instead, when you want a fixed window.

## 4. Fetch the full body

`getPost` — `GET /wp/v2/posts/{id}?_fields=id,date,modified,link,title,content,excerpt,categories,featured_media`

`content.rendered` is HTML. Strip tags for text analysis but keep the anchor hrefs — press releases
routinely link to the trial registry entry and to the partner's announcement.

An unknown or non-public id returns `404 rest_post_invalid_id`.

## 5. Resolve images

`getMediaItem` — `GET /wp/v2/media/{id}?_fields=id,source_url,alt_text,media_details,mime_type`

`featured_media` is a media id, or `0` when no featured image is set — check for `0` before
dereferencing. `listMedia` pages the whole library (176 items) if you need it in bulk.

Cheaper alternative: append `&_embed` to the `listPosts` call and read the media object straight out
of `_embedded['wp:featuredmedia'][0]`, saving a request per post.

## 6. Citation and accuracy rules

- Cite the public permalink from `link` (`https://atsenatx.com/press-release/{slug}/`), never the
  `/wp-json/` URL.
- Always date a claim with `date`, and say so. Clinical pipeline status moves — a programme described
  as entering a pivotal trial in one release may have dosed its first patient in a later one.
- **Do not infer clinical or regulatory status by combining releases.** Report what a specific
  release says, attributed to that release. Programme identifiers (ATSN-101, ATSN-201, ATSN-301,
  ATSN-401) and indications (LCA1, XLRS, USH1B, Stargardt) are precise; do not paraphrase them.
- Never present this content as medical advice or as an offer to enroll. Route patient questions to
  https://atsenatx.com/clinical-trials/ and
  https://atsenatx.com/compassionate-use-and-expanded-access/.

## 7. Be a good citizen

- No rate limit is published, but robots.txt requests `Crawl-delay: 10`. Treat that as the floor.
- Responses carry `cache-control: max-age=600`; cache for at least ten minutes.
- Only `Last-Modified` is issued (no `ETag`), so revalidate with `If-Modified-Since`.
- This is a marketing site's CMS, not a product API. It has no SLA and can change without notice —
  fail soft.

## Errors you will actually hit

| Status | `code` | Meaning |
|---|---|---|
| 400 | `rest_invalid_param` | Bad parameter; `data.params` names it. `per_page` must be 1–100. |
| 400 | `rest_post_invalid_page_number` | Paged past the last page. |
| 404 | `rest_post_invalid_id` | No such post. |
| 404 | `rest_term_invalid` | No such category or tag. |
| 401 | `rest_user_cannot_view` | You tried to resolve `author`. See below. |

Branch on `code`, never on `message`. The envelope is `{code, message, data.status}` — **not** RFC
9457 problem+json. Full catalogue: `errors/atsena-therapeutics-problem-types.yml`.

## Known dead end

`author` is an integer on every post, but `/wp/v2/users` returns `401 rest_user_cannot_view`. Do not
retry it and do not treat the 401 as transient. If you need the author name, read
`yoast_head_json.author` on the record, or call `getOembed` for the post URL, both anonymous.
