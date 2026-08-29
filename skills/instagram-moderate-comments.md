---
name: instagram-moderate-comments
description: Read, reply to, hide and delete comments on Instagram media, preferring reversible moderation actions over destructive ones.
api: instagram:instagram-comments-api
operations:
  - getMediaComments
  - getComment
  - getCommentReplies
  - createComment
  - createCommentReply
  - hideComment
  - deleteComment
generated: '2026-08-29'
method: generated
source: openapi/instagram-comments-api-openapi.yml
---

# Moderate Instagram comments

## The one rule that matters

**Hide, do not delete.** `hideComment` is the only fully reversible write on this platform — the
same operation with `hide=false` restores the comment. `deleteComment` has no undo. An agent
moderating autonomously should hide and escalate; deletion is a human decision.

## Before you start

- Base host `https://graph.instagram.com/v26.0` (Instagram Login) or
  `https://graph.facebook.com/v26.0` (Facebook Login).
- Permission: `instagram_business_manage_comments` (Instagram Login) or
  `instagram_manage_comments` (Facebook Login).

## Reading

### List comments on a post — `getMediaComments`

```
GET /{media_id}/comments?fields=id,text,username,timestamp,like_count,hidden,parent_id
```

`fields` is effectively mandatory — a bare request returns almost nothing. Paginate with
`paging.cursors.after`; follow `paging.next` verbatim rather than rebuilding it.

### One comment — `getComment`

```
GET /{comment_id}?fields=id,text,username,timestamp,like_count,hidden,parent_id
```

### Replies to a comment — `getCommentReplies`

```
GET /{comment_id}/replies
```

`parent_id` on a `Comment` is the only explicit foreign key in the whole Instagram data model — use
it to reassemble threads.

## Writing

### Reply publicly — `createCommentReply`

```
POST /{comment_id}/replies?message=...
```

### Comment on a post — `createComment`

```
POST /{media_id}/comments?message=...
```

Reversible via `deleteComment`.

### Hide or unhide — `hideComment`

```
POST /{comment_id}?hide=true
POST /{comment_id}?hide=false
```

Returns `{"success": true}`. This is your default moderation action.

### Delete — `deleteComment`

```
DELETE /{comment_id}
```

**Irreversible.** No restore, no window, no trash. Require human confirmation.

## Staying current

Subscribe to the `comments` webhook field rather than polling — see
`asyncapi/instagram-webhooks.yml`. Verify `X-Hub-Signature-256` against the raw body before you
act on any payload. Delivery is at-least-once and unordered, and there is no idempotency key on the
REST side, so make your own handler idempotent on `(entry.id, entry.time, field)`.

## Errors

`error.code` is the discriminator, not the HTTP status. `10` and `200-299` mean the permission is
missing; `4`/`17`/`613` mean back off; `190` means re-authorize. See
`errors/instagram-problem-types.yml`.
