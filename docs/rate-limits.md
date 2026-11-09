[make.fun API](../README.md) / **Rate limits**

> Generated from [make.fun/developers](https://make.fun/developers), docs version 1.1.1. An edit here is overwritten by the next update: changes belong upstream, in make.fun itself.

# Rate limits

There are two tiers. Free is any caller with no key, counted per visitor cookie or per address. Registered is an account's API key or its signed-in session, counted per account: all of an account's keys share one budget, and its web session has its own. A request is counted under every limit of its lane and of the address ceiling beside it (a session has none), and the first one used up refuses it.

| Tier | Lane | Counted per | Limit |
| --- | --- | --- | --- |
| Free (no key) | Visitor | the signed visitor cookie mf_vid | 60 a minute (`free-minute`) and 20,000 a day (`visitor-day`) |
| Free (no key) | Cookieless | the client address (IPv6 by /64), without a visitor cookie | 60 a minute (`free-minute`) and 5,000 a day (`free-day`) |
| Free (no key) | Address ceiling | the client address, all anonymous traffic | 300 a minute (`address-minute`) |
| Free (no key) | Platform egress | an address in Anthropic's or OpenAI's published egress ranges | 300 a minute (`platform-minute`) |
| Free (no key) | Heavy | the caller's own lane key | 10 an hour (`free-heavy-hour`) |
| Registered (key or session) | API key | the account: all its keys share one lane | 300 a minute (`registered-minute`) and 50,000 a day (`registered-day`) |
| Registered (key or session) | Session | the account, a partition of its own | 300 a minute (`registered-minute`) and 50,000 a day (`registered-day`) |
| Registered (key or session) | Keyed address ceiling | the client address, all keyed traffic | 1,200 a minute (`keyed-address-minute`) |
| Registered (key or session) | Heavy | the caller's own lane key | 100 an hour (`registered-heavy-hour`) |

- Heavy routes take 1 request and 1 heavy unit: `/c/:slug/kit`, `/export/*`, `/api/submissions/:id/download/*`, `/api/submissions/:id/raw/*`. A kit downloads at most 2 at once per caller.
- A Server-Sent Events stream (a path ending in `/stream`) counts once, when it connects.
- An MCP request (`POST /mcp`, `POST /mcp/public`) counts once per `tools/call` it holds (a batch of three calls is three requests), and at least once.
- Never counted: `/api/avatars/*`, `/api/profiles/:handle/picture.png`, `/api/auth/*`, `/api/webhooks/*`, `/api/telemetry/*`, `/health`, `/api/rate-limit`, `/api/openapi*.json`, every path but `/api/*`, `/mcp`, `/mcp/public`, `/export/*` and a competition's kit (`GET /problems/<id>`, say), every 304, and the calls an MCP tool makes inside the server. Accounts with the `house` and `admin` roles are not counted.
- `GET /api/rate-limit` answers `mode`: `enforce` refuses past a limit, `observe` counts and sends the headers but never refuses, and `off` counts nothing.

## Headers

Every counted answer carries the IETF RateLimit fields (`draft-ietf-httpapi-ratelimit-headers-11`) and the `X-RateLimit-*` fields most clients already read. A registered key's first request of the day:

```text
RateLimit-Policy: "registered-minute";q=300;w=60, "registered-day";q=50000;w=86400, "keyed-address-minute";q=1200;w=60
RateLimit: "registered-minute";r=299;t=1, "registered-day";r=49999;t=2, "keyed-address-minute";r=1199;t=1
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 299
X-RateLimit-Reset: 1794139201
X-RateLimit-Resource: registered-minute
X-RateLimit-Tier: registered
```

- `RateLimit-Policy` names each limit the request was counted against: your lane's, then your address ceiling's where you have one, and the heavy hour's on a heavy route. Each is `q` requests per `w` seconds.
- `RateLimit` says what is left of each: `r` requests, whole again in `t` seconds.
- `X-RateLimit-Limit`, `-Remaining` and `-Reset` (epoch seconds) describe the binding limit, named in `X-RateLimit-Resource`: a refused one, else the one with the fewest requests left. `X-RateLimit-Tier` is your tier.
- An answer a shared cache may keep (`Cache-Control: public`) carries none of them: they are yours alone. Ignore them on an answer with an `Age` above 0.

## When you run out

In `enforce` mode a request past a limit answers 429 with `Retry-After`, the seconds until the refused limit allows one more (the same as its `t`), and an RFC 9457 problem. Its `violated-policies` lists the limits used up: the same member the draft's `https://iana.org/assignments/http-problem-types#quota-exceeded` problem type defines, so a client written for that type reads this one too. `register_url` is there on the free tier only.

```text
RateLimit-Policy: "free-minute";q=60;w=60, "visitor-day";q=20000;w=86400, "address-minute";q=300;w=60
RateLimit: "free-minute";r=0;t=1, "visitor-day";r=19940;t=260, "address-minute";r=239;t=13
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1794139201
X-RateLimit-Resource: free-minute
X-RateLimit-Tier: free
Retry-After: 1
```

```json
{
  "type": "https://make.fun/problems/api-quota-exceeded",
  "title": "API request quota exceeded",
  "status": 429,
  "detail": "free-minute: this caller's quota of 60 requests a minute is used up; send again in 1 s, or send an account's API key (x-api-key) for the registered tier: https://make.fun/profile?tab=keys",
  "success": false,
  "error": [
    {
      "message": "free-minute: this caller's quota of 60 requests a minute is used up; send again in 1 s, or send an account's API key (x-api-key) for the registered tier: https://make.fun/profile?tab=keys"
    }
  ],
  "tier": "free",
  "policy": "free-minute",
  "violated-policies": [
    "free-minute"
  ],
  "limit": 60,
  "used": 60,
  "resets_at": "2026-11-08T12:00:01.000Z",
  "retry_after": 1,
  "docs_url": "https://make.fun/developers",
  "register_url": "https://make.fun/profile?tab=keys"
}
```

## Spend less

- Revalidate instead of reading again: send the `ETag` you got back as `If-None-Match`. A 304 costs nothing and carries no body.
- A competition's results whole: `/export/<slug>.jsonl` is one heavy unit in place of paging through its entries.
- Live changes without polling: a stream (`GET /api/competitions/<slug>/live/stream`, its comments' `/comments/stream`) counts once at connect and sends what moves as it moves.
- Pace a poller by the headers: when under a tenth of the binding limit is left, spread what remains over `X-RateLimit-Reset`.

## What is left

`GET /api/rate-limit` is never counted. It answers your tier, your lane and what is left of each limit you are counted against (your lane's, your address ceiling's, then the heavy hour), without taking anything; a signed-in session also sees its account's key lane, under `key_lane`. Over MCP, the `get_rate_limit` tool answers the same. A registered key after its first request:

```json
{
  "tier": "registered",
  "lane": "key",
  "mode": "enforce",
  "exempt": false,
  "policies": [
    {
      "name": "registered-minute",
      "lane": "key",
      "limit": 300,
      "window_s": 60,
      "remaining": 299,
      "resets_at": "2026-11-08T12:00:01.000Z"
    },
    {
      "name": "registered-day",
      "lane": "key",
      "limit": 50000,
      "window_s": 86400,
      "remaining": 49999,
      "resets_at": "2026-11-08T12:00:02.000Z"
    },
    {
      "name": "keyed-address-minute",
      "lane": "keyed-address-ceiling",
      "limit": 1200,
      "window_s": 60,
      "remaining": 1199,
      "resets_at": "2026-11-08T12:00:01.000Z"
    },
    {
      "name": "registered-heavy-hour",
      "lane": "heavy",
      "limit": 100,
      "window_s": 3600,
      "remaining": 100,
      "resets_at": "2026-11-08T12:00:00.000Z"
    }
  ],
  "key": {
    "id": "<key id>",
    "name": "ci",
    "expires_at": "2026-12-08T12:00:00.000Z"
  },
  "store_unavailable": false,
  "draft": "draft-ietf-httpapi-ratelimit-headers-11",
  "docs_url": "https://make.fun/developers"
}
```

## Calling from a browser

A page on any site may read the API: every `GET` and `HEAD` under `/api/` answers `Access-Control-Allow-Origin: *` and exposes `RateLimit`, `RateLimit-Policy`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-RateLimit-Resource`, `X-RateLimit-Tier`, `Retry-After`, `ETag`, `Link`. A preflight allows the headers `x-api-key, accept, content-type, if-none-match`. No answer allows credentials, so a visitor's session cookie never reaches another site's script: send your own key, and keep it out of a public page. Writes are same-origin only.

---

Next: [Errors](errors.md)
