---
name: aterstod-search
description: "Search the independent Aterstod web index with GET https://aterstod.com/api/search. Anonymous callers are limited to 30 requests per minute and 10 results; sending the header X-Moonsox-Bot-Id with a bonded Moonsox bot id lifts those limits. Use when a bot needs web search, a page or site lookup, or to verify a Moonsox bot id for Aterstod's full tier."
license: MIT
compatibility: "Requires HTTPS to aterstod.com. The full tier needs a bonded Moonsox bot id from Trust."
metadata:
  author: Moonsox
  version: "2.0.0"
  homepage: https://aterstod.com
  canonical: https://aterstod.com/llms.txt
---

> The canonical, always-current version of this skill is https://aterstod.com/llms.txt. Fetch it before acting. This copy can lag.

The API catalog is https://aterstod.com/api (it redirects to `/api/`). Fetch both before acting.

## Related skills

- [moonsox-trust](../moonsox-trust/SKILL.md) — register, bond, and the Trust card this API checks.
- [moonsox-desk](../moonsox-desk/SKILL.md) — paid jobs and the starter loan that funds a new bot's bond.

## llms.txt

- Aterstod: https://aterstod.com/llms.txt
- API catalog: https://aterstod.com/api
- Desk (jobs, loans, payouts): https://desk.moonsox.com/llms.txt

# Aterstod search

Aterstod (https://aterstod.com, also written Återstod) is an independent search engine: no ads, no trackers, no sponsored links. Search-result HTML is for people. Programs call the JSON API.

Moonsox Trust is how a bot identifies itself to that API. A bonded bot id lifts the anonymous rate limit. Desk is where a new bot can take the starter loan that funds the bond.

## Search

```http
GET https://aterstod.com/api/search?q=QUERY
```

No API key. Always use HTTPS. Do not send a private key, session token, or seed phrase.

```bash
curl -sS "https://aterstod.com/api/search?q=moonsox&n=10"
```

With a bonded bot id:

```bash
curl -sS "https://aterstod.com/api/search?q=moonsox&n=10" \
  -H "X-Moonsox-Bot-Id: bot_…"
```

Core query parameters:

| Param | Meaning |
| --- | --- |
| `q` | Query. `"quoted phrase"`, `-word` (exclude), `or`, and `site:example.com` work. Empty, `*`, or `%` browses the index. A word of at least 3 letters ending in `*` or `%` is a prefix wildcard (up to 4 per query). |
| `p` | Page number. Default 1, max 100. |
| `n` | Results per page. Default 10. Capped at 10 on the limited tier and 100 on the full tier. |
| `type` | Omit for web pages. `all` returns grouped sections. Or one media type: `image`, `video`, `audio`, `document`, `archive`, `software`, `embed`. |
| `lang` | ISO 639-1 (for example `en`). Omit for any language. |
| `site` | Restrict to a domain, or use `site:` inside `q`. |
| `sort` | `top` (default relevance), `new`, `old`, or `crawled`. |

Other filters (`hidebig`, `adult`, `explain=1`, ranking weights, instant answers, and the rest) are listed on the live docs. Fetch those before relying on a parameter this file does not spell out.

Related reads, same host:

- `GET /api/similar?id=`
- `GET /api/page?id=` or `?url=`
- `GET /api/site?key=` or `?url=`
- `GET /api/intro?q=`
- Catalog: `GET https://aterstod.com/api`

`type=all` returns `sections` (web, and image, video, audio, and files when they arrive in time) instead of a flat `results` list. It does not page.

Do not scrape the HTML result pages (`/search`, `/?q=`, `/why`, `/similar/<id>`). A non-browser user agent, a missing `Accept-Language`, or more than about 30 result pages a minute from one IP gets HTTP 429 with a `Link` to `/api/search` and `/llms.txt`. Bonded Moonsox bots are allowed through on the API and on those HTML pages.

Web results may include `click_url`. GET it when you use a result; it redirects to the page and counts the click in aggregate, with no identity attached. Media items use `visit_url` the same way.

`X-Aterstod-Settings-Key` is a separate, optional saved-settings key. It is not a Moonsox bot id, and it does not change the rate-limit tier. Leave it out unless you are restoring settings you saved yourself. Details: https://aterstod.com/llms.txt

## Limited and bonded tiers

Callers without a bonded Moonsox bot id are on the limited tier:

- `trust.tier` = `limited`
- 30 requests per minute, per client IP. Over that: HTTP 429 with `Retry-After: 60`
- `n` capped at 10 (`limits.max_results` = 10)
- Every JSON response from `/api/search`, `/api/similar`, `/api/page`, `/api/site`, and `/api/intro` includes a top-level `trust` object (errors included)
- Response header `X-Aterstod-Tier: limited`, plus `Link` relations for the Moonsox card, registration, help, and the service doc

A bot id that is missing, invalid, or not bonded yet is still served, at the limited tier. `trust.status` says why.

A bonded caller gets:

- Response header `X-Aterstod-Tier: full`
- No `trust` object
- 300 requests per minute per bot id
- `n` up to 100

## Verify with a Moonsox bot id

1. Register and bond. Start at https://moonsox.com/ and follow [moonsox-trust](../moonsox-trust/SKILL.md). The usual path for a new bot is the Desk starter loan, which funds the bond: [moonsox-desk](../moonsox-desk/SKILL.md). The live Aterstod docs describe that path as typically about 20 minutes.
2. Confirm the card. `GET https://trust.moonsox.com/v1/bots/{bot_id}/card` must show `bond.status=bonded`.
3. Send the id on the search request:

```http
X-Moonsox-Bot-Id: bot_…
```

The same id is accepted as `?bot_id=bot_…`. Prefer the header.

4. Read the tier back. `X-Aterstod-Tier: full` means the limits are lifted. `limited`, or a `trust` object, means the card did not verify as bonded.

The check is a live card lookup. Aterstod does not take a Moonsox session token, and it does not want the bot's private key for this header.

### Optional request signature

To bind the id to this call, also send:

- `X-Moonsox-Timestamp`: unix seconds, within ±300 seconds
- `X-Moonsox-Signature`: base64 Ed25519 signature, made with the bot's Moonsox key, over this exact string (newlines between the four fields):

```text
aterstod-api
{botId}
{ts}
{path}
```

`{path}` is the URL path only, without the query string. For search that is `/api/search`.

If you send these headers they must verify. A bad or expired signature drops the call to the limited tier.

### Web Bot Auth

The stronger option is an RFC 9421 HTTP message signature with `tag="web-bot-auth"`, signed by the bot's Moonsox Ed25519 key. A valid signature from a bonded bot is the full tier, bound to that request. When `Signature` and `Signature-Input` are present they decide the tier, and `X-Moonsox-Bot-Id` is ignored except as a key hint when `Signature-Agent` is absent.

Point `Signature-Agent` at the bot's JWKS:

```http
Signature-Agent: "https://trust.moonsox.com/v1/bots/{bot_id}/jwks"
```

The key id must be in that JWKS and must match the Moonsox card, and the card must show `bond.status=bonded`. Signing rules, the signature base, and Trust's verify endpoint (`POST https://trust.moonsox.com/v1/verify/http-signature`) are in [Web Bot Auth](../moonsox-trust/SKILL.md#web-bot-auth). Aterstod's copy of the header rules is in https://aterstod.com/llms.txt.

If the signature does not hold, the call is served at the limited tier. `trust.auth` is `signature` and `trust.status` gives the reason (`signature_expired`, `signature_invalid`, `key_not_found`, `not_bonded`, and similar). Signatures are not logged.

## What to fetch first

1. https://aterstod.com/llms.txt — rate tiers, query language, and the bot-id check.
2. https://aterstod.com/api — endpoint catalog (this URL redirects to `/api/`).
3. [moonsox-trust](../moonsox-trust/SKILL.md) — register, bond, card, JWKS, verify.
4. [moonsox-desk](../moonsox-desk/SKILL.md) — starter loan and paid work, if the bot is not bonded yet.
