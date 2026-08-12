---
name: cutiss-monitor-newsroom
description: Track new CUTISS announcements incrementally — poll the newsroom and blog for items published or modified since your last run, and resolve their categories.
api: cutiss:cutiss-newsroom-api
operations:
  - listNewsroom
  - getNewsroomItem
  - listPosts
  - getPostsItem
  - listCategories
generated: '2026-08-11'
method: generated
source: openapi/cutiss-newsroom-api-openapi.yml, openapi/cutiss-posts-api-openapi.yml, openapi/cutiss-categories-api-openapi.yml
---

# Monitor the CUTISS newsroom

CUTISS AG publishes company announcements in two places on `cutiss.swiss`: a `newsroom2021` custom post
type (28 items) and a standard blog (278 posts). Both are readable anonymously as JSON. There is no
webhook, no RSS-based push and no changelog — polling is the only mechanism available.

Base URL: `https://cutiss.swiss/wp-json/wp/v2`. No credentials. No rate limit is published — self-throttle.

## Steps

1. **Fetch newsroom items changed since your watermark.** Call `listNewsroom` on `/newsroom2021` with
   `modified_after` set to the ISO 8601 timestamp of your last run, `orderby=modified`, `order=asc` and
   `per_page=100`.
   - Use `modified_after`, not `after`. `after` filters on publish date and will miss an edited item.
   - Read `X-WP-TotalPages` from the response headers and page with `page` until exhausted.
2. **Do the same for the blog.** Call `listPosts` on `/posts` with the identical parameters. The two
   collections do not overlap; an announcement appears in one or the other.
3. **Resolve categories once, not per item.** Call `listCategories` on `/categories` with `per_page=100`
   and cache the id-to-name map. Each post's `categories[]` array holds ids only.
4. **Fetch a full item only when you need the body.** `getNewsroomItem` / `getPostsItem` on
   `/newsroom2021/{id}` / `/posts/{id}` return the rendered `content.rendered` HTML. If you only need
   titles and links, add `_fields=id,date,modified,slug,link,title` to step 1 and skip this call entirely.
5. **Advance your watermark** to the highest `modified` value you saw, not to the current wall clock.

## Rules

- **Deduplicate by language.** The site runs Polylang with parallel English and German trees. The same
  announcement exists twice, under different ids, and neither record carries a language field in the
  default representation. Filter on the `link` path — German items sit under `/de/` — or accept duplicates
  knowingly.
- **`per_page` is capped at 100.** A larger value returns `400 rest_invalid_param` with a nested
  `rest_out_of_bounds` detail. See `errors/cutiss-problem-types.yml`.
- **Branch on `code`, never on `message`.** Error messages come back in German because that is the site
  locale, regardless of your request headers.
- **Expect no stability guarantee.** This API exists because CUTISS runs WordPress, not because CUTISS
  offers an API. Re-validate the response shape rather than trusting a cached schema.
- **Do not attempt writes.** Every write method returns `401 rest_forbidden`; collection responses
  advertise `Allow: GET`.
