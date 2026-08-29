---
name: instagram-embed-content
description: Embed an Instagram post, reel or video on a third-party page using the oEmbed endpoint — the only Instagram API surface callable with no credential at all.
api: instagram:instagram-oembed-api
operations:
  - instagram_oembed
generated: '2026-08-29'
method: generated
source: https://developers.facebook.com/docs/instagram-platform/oembed
---

# Embed Instagram content

This is the outlier in the whole Instagram surface: no OAuth, no App Review, no permission, no
rate-limit negotiation. Since 15 May 2026 the oEmbed endpoint answers anonymously — verified live
on 29 August 2026 with a plain unauthenticated GET returning HTTP 200.

## Call it

```
GET https://graph.facebook.com/v26.0/instagram_oembed?url={permalink}
```

Parameters:

| param | meaning |
|---|---|
| `url` | **required** — the Instagram permalink (post, reel or video) |
| `maxwidth` | maximum render width |
| `fields` | restrict the returned fields |
| `omitscript` | `true` to leave out the `<script>` tag so you can load `embed.js` yourself |
| `hidecaption` | `true` to suppress the caption |

Response is conformant **oEmbed 1.0**:

```json
{
  "version": "1.0",
  "provider_name": "Instagram",
  "provider_url": "https://www.instagram.com/",
  "type": "rich",
  "width": 658,
  "html": "<blockquote class=\"instagram-media\" ...>"
}
```

Because it is real oEmbed, any consumer that already speaks the standard — a CMS, a static site
generator, a chat client — renders Instagram with no bespoke connector.

## Render it

Inject the returned `html` and load the loader script once per page:

```html
<script async src="https://www.instagram.com/embed.js"></script>
```

Pass `omitscript=true` on the oEmbed call if you are managing that script tag yourself, otherwise
you will load it once per embed.

**Caveat worth knowing:** `https://www.instagram.com/embed.js` is **unpinned**. It always serves
the current build, there is no versioned URL and no published integrity hash, so subresource
integrity is not available and you cannot roll back.

## When it fails

Private accounts and deleted posts are not embeddable — the call returns a Meta error envelope
(`error.code`), not an oEmbed document. Check for the `error` key before reading `html`.

See `components/instagram-components.yml` and `conformance/instagram-conformance.yml`.
