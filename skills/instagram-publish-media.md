---
name: instagram-publish-media
description: Publish a photo, video, reel or carousel to an Instagram professional account using the two-step container flow, respecting the 100-posts-per-24-hours limit.
api: instagram:instagram-publishing-api
operations:
  - getContentPublishingLimit
  - createMediaContainer
  - getContainer
  - publishMedia
generated: '2026-08-29'
method: generated
source: openapi/instagram-publishing-api-openapi.yml
---

# Publish media to Instagram

Publishing is a **two-step, one-way** flow. You stage a container, then you publish it. Once
published there is no unpublish — the only way back is `deleteMedia`, which is itself
irreversible. Treat the publish call as an action that needs human confirmation.

## Before you start

- Base host: `https://graph.instagram.com` (Instagram Login) or `https://graph.facebook.com`
  (Facebook Login). Prefix the path with the version, e.g. `/v26.0/`.
- Permission required: `instagram_business_content_publish` (Instagram Login) or
  `instagram_content_publish` (Facebook Login). Both need App Review for Advanced Access.
- Send the token as `Authorization: Bearer {token}`, not as a query parameter.

## Steps

### 1. Check the quota first — `getContentPublishingLimit`

```
GET /{user_id}/content_publishing_limit
```

An account is capped at **100 API-published posts in a rolling 24 hours**. A carousel counts as
one. Meta's own guidance is that the app should enforce this itself. Read the quota before you
stage anything — a container you cannot publish is wasted work and expires in 24 hours anyway.

### 2. Stage the media — `createMediaContainer`

```
POST /{user_id}/media
```

Body fields (from `CreateContainerRequest`): `image_url`, `video_url`, `caption`, `media_type`,
`is_carousel_item`, `location_id`, `user_tags`.

- The media must be at a **publicly reachable URL** — Meta fetches it, you do not upload bytes.
- Returns `{"id": "<container_id>"}`.
- **This does not publish anything.** Abandoning the container is the safe rollback: an
  unpublished container expires on its own after 24 hours.

### 3. Poll until the container is ready — `getContainer`

```
GET /{container_id}?fields=status_code
```

Video and reel containers process asynchronously. Do not call publish until the container reports
it is finished. Poll with backoff; do not tight-loop — every poll counts against the rate limit.

### 4. Publish — `publishMedia`

```
POST /{user_id}/media_publish?creation_id={container_id}
```

Returns the published media id.

**Stop here if you are an autonomous agent.** This is the point of no return. Get a human to
confirm the caption and the media before this call.

## Errors you will actually hit

Read `error.code`, not the HTTP status — Meta returns HTTP 400 for most failures.

| code | meaning | what to do |
|---|---|---|
| 506 | Duplicate post — identical content published consecutively | change the content; this is not an idempotency guarantee |
| 4 / 17 / 613 | rate limited | back off; read `X-App-Usage` and `X-Business-Use-Case-Usage` |
| 190 (+subcode) | token expired/revoked | re-authorize; check the subcode |
| 10 / 200-299 | permission missing | the permission was never granted or was revoked |
| 1609005 | link could not be scraped | verify the URL resolves publicly |

## Retry rules

There is **no idempotency key** on this API. If a `publishMedia` call times out, do **not** blind
retry — first re-read the account's media (`getUserMedia`) to check whether the post landed.
Retrying `createMediaContainer` is safe: it creates a new container, not a duplicate post.

See `conventions/instagram-conventions.yml`, `errors/instagram-problem-types.yml`,
`rate-limits/instagram-rate-limits.yml`.
