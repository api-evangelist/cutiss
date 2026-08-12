---
name: cutiss-map-organization
description: Build the CUTISS team directory — enumerate the cutiss-teams taxonomy, list the member profiles filed under each unit, and resolve each profile's portrait image.
api: cutiss:cutiss-team-api
operations:
  - listTeams
  - getTeamsItem
  - listTeam
  - getTeamItem
  - listMedia
  - getMediaItem
generated: '2026-08-11'
method: generated
source: openapi/cutiss-team-api-openapi.yml, openapi/cutiss-teams-api-openapi.yml, openapi/cutiss-media-api-openapi.yml
---

# Map the CUTISS organization

CUTISS publishes its people as structured data: a `team_member` custom post type (128 profiles) filed
under a `cutiss-teams` taxonomy (18 terms). Together they describe the organizational structure of a
~50-person company across both language trees.

Base URL: `https://cutiss.swiss/wp-json/wp/v2`. No credentials.

## Steps

1. **Enumerate the organizational units.** Call `listTeams` on `/cutiss-teams` with `per_page=100`. Each
   term returns `id`, `name`, `slug`, `count` and `parent`. A non-zero `parent` means the unit nests —
   build the tree before listing members so sub-teams are not flattened into their parent.
2. **List profiles per unit.** Call `listTeam` on `/team_member` filtered to one term at a time using the
   taxonomy query parameter for `cutiss-teams`, with `per_page=100` and `orderby=title&order=asc`.
   - Cross-check your totals against each term's `count`. A mismatch means the term nests and you need
     the children too.
3. **Resolve portraits in one pass.** Each profile carries `featured_media`, a `MediaItem` id (`0` when
   unset). Collect the non-zero ids and call `listMedia` on `/media` with `include=<comma-separated ids>`
   and `per_page=100` rather than calling `getMediaItem` once per person. Read `source_url` for the file
   and `media_details.sizes` for the variants.
   - Alternative: add `_embed` to step 2 and WordPress inlines the media object, collapsing this step.
4. **Fetch a full profile body only if you need the bio.** `getTeamItem` on `/team_member/{id}` returns
   `content.rendered`.

## Rules

- **Author ids do not resolve.** Profiles carry an `author` field, but `/wp/v2/users` returns
  `401 rest_forbidden`. Do not build anything that depends on resolving it.
- **128 profiles is not 128 people.** English and German records are separate objects with distinct ids.
  Deduplicate on the `/de/` path segment in `link`, or on `slug`, before reporting a headcount — CUTISS
  employs roughly 50 people.
- **This is personal data.** The directory names real individuals with photographs. It is publicly
  published by CUTISS, but handle it accordingly and do not redistribute portraits — media is
  `© CUTISS AG`, not openly licensed.
- **`per_page` is capped at 100**; page with `X-WP-TotalPages`.
- **Read-only.** All write methods return `401 rest_forbidden`.
