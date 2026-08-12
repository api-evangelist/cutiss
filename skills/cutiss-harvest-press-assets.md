---
name: cutiss-harvest-press-assets
description: Find CUTISS corporate content by keyword and download the press and media assets attached to it, resolving search hits through to media source URLs.
api: cutiss:cutiss-search-api
operations:
  - listSearch
  - listPages
  - getPagesItem
  - listMedia
  - getMediaItem
generated: '2026-08-11'
method: generated
source: openapi/cutiss-search-api-openapi.yml, openapi/cutiss-pages-api-openapi.yml, openapi/cutiss-media-api-openapi.yml
---

# Harvest CUTISS content and press assets

`cutiss.swiss` exposes full-text search across all 514 publicly readable objects and an 842-item media
library. This skill goes from a keyword — `denovoSkin`, `denovoCast`, `Orphan Drug`, `Wyss Zurich` — to
the corporate pages that discuss it and the image and document assets attached to them.

Base URL: `https://cutiss.swiss/wp-json/wp/v2`. No credentials.

## Steps

1. **Search.** Call `listSearch` on `/search` with `search=<keyword>` and `per_page=100`. Each hit returns
   `id`, `title`, `url`, `type` and `subtype` — and nothing else. Search results are a projection, not a
   full object.
2. **Resolve each hit against its real collection.** `subtype` names the entity: `page` → `/pages/{id}`,
   `post` → `/posts/{id}`, `newsroom2021` → `/newsroom2021/{id}`, `team_member` → `/team_member/{id}`.
   Group the ids by subtype and batch-fetch with `include=` on the collection endpoint instead of issuing
   one request per hit.
3. **Pull the corporate pages directly when you want the canonical narrative.** `listPages` on `/pages`
   with `per_page=100` returns all 34 pages; the substantive ones are `about-us`, `technology`,
   `clinical-problem-scars` (Clinical Development), `investors`, `media`, `awards` and `career`.
   `getPagesItem` returns `content.rendered`.
4. **Find the attached assets.** Two paths, and you usually want both:
   - **Featured image:** the resolved object's `featured_media` id → `listMedia` with `include=<ids>`.
   - **Everything attached to an object:** `listMedia` on `/media` filtered by the `parent` of the object
     id, which returns every file uploaded against that page or post.
5. **Download from `source_url`.** Use `media_details.sizes` to pick a variant rather than the full-size
   original when a thumbnail will do, and read `mime_type` to separate images from PDFs and other
   documents.

## Rules

- **Assets are copyrighted.** Media on `cutiss.swiss` is `© CUTISS AG` and carries no open licence. Use
  the `/media/` page terms for press use; do not assume redistribution rights.
- **Search covers both languages.** German equivalents of the same page appear as separate hits under
  `/de/`. Filter the `url` field if you want one language.
- **`per_page` is capped at 100** — a larger value returns `400 rest_invalid_param`.
- **Branch on the error `code`.** Messages are German-localized.
- **A 404 `rest_post_invalid_id`** means the id was not resolvable in that collection — check you used the
  right `subtype` mapping in step 2 rather than retrying.
- **Read-only.** Uploads and edits return `401 rest_forbidden`.
