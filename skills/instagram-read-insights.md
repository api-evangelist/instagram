---
name: instagram-read-insights
description: Pull account-level and media-level Instagram metrics, including the story metrics that are only reachable before a story expires.
api: instagram:instagram-insights-api
operations:
  - getUserInsights
  - getMediaInsights
  - getUserMedia
  - getUserStories
generated: '2026-08-29'
method: generated
source: openapi/instagram-insights-api-openapi.yml
---

# Read Instagram insights

Read-only skill. Nothing here mutates anything, so there is no reversibility concern — but there
is a **hard data-loss trap**, described at the end.

## Before you start

- Permission: `instagram_manage_insights`.
- Base host `https://graph.instagram.com/v26.0` or `https://graph.facebook.com/v26.0`.

## Account metrics — `getUserInsights`

```
GET /{user_id}/insights?metric=...&period=...&since=...&until=...
```

Insights are the one edge that is **time-ranged rather than cursor-paginated** — `since` and
`until` are UNIX timestamps and `period` selects the aggregation. Response shape is
`{"data":[{"name":..., "period":..., "values":[{"value":..., "end_time":...}]}]}`.

## Media metrics — `getMediaInsights`

```
GET /{media_id}/insights?metric=...
```

Available metrics differ by `media_type` — a reel, a feed image and a story do not expose the same
set. Read `media_type` from `getUserMedia` first and branch.

## Walking the account

1. `getUserMedia` — `GET /{user_id}/media?fields=id,media_type,timestamp,permalink` and paginate.
2. For each media id, `getMediaInsights`.
3. Respect the rate limit: the Instagram Platform BUC budget is
   `4800 * impressions` calls per rolling 24 hours, and a per-media insights walk burns it fast.
   Watch `X-Business-Use-Case-Usage.estimated_time_to_regain_access` — it is the only back-off hint
   Meta gives.

## The story trap

Stories live for 24 hours and then their insights are **gone**. `getUserStories` only returns
currently-active stories, and there is no historical stories edge. A polling integration that runs
less often than daily will silently lose story data forever.

The supported answer is the `story_insights` webhook field, which delivers a story's first-24-hours
metrics before it expires — see `asyncapi/instagram-webhooks.yml`. If you need story analytics, you
need webhooks; polling is not a substitute.

## Errors

`4`/`17`/`613` and the `80xxx` block are all rate limiting. `10` and `200-299` mean
`instagram_manage_insights` was not granted. See `errors/instagram-problem-types.yml`.
