---
name: Assemble an Atsena Therapeutics pipeline and platform brief
description: Pull the four clinical programme pages, the pipeline, and the two AAV platform-technology pages from the site's WordPress REST content API into one sourced brief, with staleness and citation rules for clinical content.
api: openapi/atsena-therapeutics-wp-rest-openapi.yml
operations: [getRouteIndex, listTypes, listPages, getPage, listPosts, searchContent, getMediaItem]
generated: '2026-08-02'
method: generated
source: openapi/atsena-therapeutics-wp-rest-openapi.yml
---

# Assemble an Atsena Therapeutics pipeline and platform brief

Everything a brief needs about Atsena's science and pipeline is in the **pages** collection, arranged
in a two-level hierarchy you can walk with a single `parent` filter. Do not scrape the site.

**Base URL:** `https://atsenatx.com/wp-json`
**Auth:** none. Every call below is anonymous.

## 1. Confirm the surface (once)

`getRouteIndex` — `GET /` and `listTypes` — `GET /wp/v2/types`

Confirms the namespaces and the REST-exposed content types. Expect exactly the stock WordPress set:
`post`, `page`, `attachment` and the block-editor internals. **There are no custom content types**, so
there is no programmes or people endpoint — the programme content lives in `page` records.

## 2. Walk the page hierarchy

`listPages` — `GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent,menu_order`

29 pages, two levels deep. The parent ids you need (verified 2026-08-02, re-resolve by slug rather
than hardcoding):

| Parent | id | Children |
|---|---|---|
| `/programs/` | 20 | `pipeline`, `xlrs`, `lca1`, `ush1b`, `stgd` |
| `/our-approach/` (Technology) | 570 | `scientific-approach`, `laterally-spreading-aav`, `dual-vector-technology` |
| `/about/` | 9 | `overview`, `team`, `bod`, `founders`, `sab`, `investors`, `partners` |
| `/news/` | 26 | `press-releases`, `presentations-and-publications` |

So: `GET /wp/v2/pages?parent=20` gives the whole clinical pipeline, and `GET /wp/v2/pages?parent=570`
gives the platform technology. `menu_order` reflects the site's intended reading order — sort by it,
not alphabetically.

Two homepage records exist and both are `status: publish` — `home` (id 5) and `home-new` (id 1614),
each with `menu_order: -1`. The route index settles it: `page_on_front` is **5**, so `home` is the
live front page and `home-new` is a staged replacement that is publicly readable but not yet
promoted. Read `page_on_front` from `getRouteIndex` rather than guessing, and do not treat
`home-new` as current content.

## 3. Pull the bodies

`getPage` — `GET /wp/v2/pages/{id}?_fields=id,slug,link,title,content,excerpt,modified`

`content.rendered` is HTML. Preserve tables and list structure — the pipeline page carries programme
stage in a layout table, and flattening it to prose loses the stage-per-programme mapping.

## 4. Cross-check against the newsroom

`listPosts` — `GET /wp/v2/posts?per_page=20&orderby=date&order=desc&_fields=id,date,link,title,excerpt`

**Do this every time.** Static pages go stale between site updates; press releases do not. If a
programme page and a recent release disagree on trial stage, the release is newer — say so and cite
both with their dates. Compare each page's `modified` against the newest post `date`.

`searchContent` — `GET /wp/v2/search?search=ATSN-201&per_page=20` finds every page and post
mentioning a programme in one call (72 items indexed), which is the fastest way to gather all
references to a single asset.

## 5. Figures and diagrams

`getMediaItem` — `GET /wp/v2/media/{id}?_fields=id,source_url,alt_text,media_details`

The mechanism-of-action and pipeline diagrams are in the media library (176 items). Use `alt_text`
as the caption where present; do not invent one.

## Citation and accuracy rules

- Cite the public `link`, never the `/wp-json/` URL.
- Stamp every claim with the page's `modified` or the post's `date`.
- Keep asset and indication naming exact: **ATSN-201** / X-linked retinoschisis (XLRS), **ATSN-101** /
  LCA1 (Leber congenital amaurosis 1), **ATSN-301** / Usher syndrome 1B, **ATSN-401** / Stargardt
  disease. Do not substitute a generic description for a programme code.
- Distinguish the two platforms carefully: the **laterally spreading AAV.SPR capsid** (reaches retinal
  cells without a subretinal bleb) and the **dual-vector technology** (splits oversized genes across
  two vectors). They are separate claims and are not interchangeable.
- Trial stage is a regulated statement. Report the stage a named source asserts, with its date. Do not
  compute or advance a stage yourself.
- This is company marketing copy, not peer-reviewed literature. Attribute it to the company. For
  independent evidence, follow out to the trial registry and publications the pages link to.
- Never present any of it as medical advice.

## Known gaps — do not paper over them

- **The people roster is not on the API.** A `team` custom post type exists on the site (it is listed
  in the Yoast sitemap index at `/team-sitemap.xml`) but it is **not** registered for REST and does
  not appear in `listTypes`. The `/about/team/`, `/about/bod/`, `/about/founders/` and `/about/sab/`
  pages return only their own page bodies, which may render the roster as HTML rather than as
  structured records. There is no JSON roster to fetch — say the data is unavailable rather than
  approximating it.
- `author` on any record cannot be resolved: `/wp/v2/users` returns `401 rest_user_cannot_view`.
- There is no publications endpoint. `/news/presentations-and-publications/` is a single page; the
  publication list is inside its `content.rendered` HTML.
