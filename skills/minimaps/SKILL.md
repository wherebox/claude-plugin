---
name: minimaps
description: Add static map images to an app with the Minimaps API, such as locator maps (an area highlighted and its surroundings faded), labeled points, route or flight lines, and neighboring-country labels. Covers building <img> GET URLs or POST JSON map specs, signing URLs with publishable keys, using the @minimaps/sdk (TypeScript) or minimaps (Python) SDKs, and handling ambiguous or unknown place names. Use when the user wants a locator map, a "where is X" map, a static map image, or a route map picture, or is working with the Minimaps API, its map spec, its API keys or its SDKs.
license: MIT
---

# Minimaps

Minimaps is an HTTP API that returns one PNG, JPEG or WebP image per map. A map is described once, as a **MapSpec**. You can send it as GET query parameters (`/v1/map.png?area=Czechia&w=468&h=280`) or as a JSON body to `POST /v1/map`. Both forms describe the same spec.

Load these on demand:
- [reference/parameters.md](reference/parameters.md): every GET parameter and POI option, with values, defaults and examples, plus POST-only fields and limits.
- [reference/errors.md](reference/errors.md): every error code, its extra fields and a sample body.
- [reference/http.md](reference/http.md): endpoints, the auth parameters (`apikey=`, `sig=`, `exp=`) and the headers.
- [reference/recipes.md](reference/recipes.md): SDK code (TypeScript, Python), signing without an SDK, and handling ambiguous names.

## 1. Look at the project first

Before writing code, check these:
- **Base URL.** Look for an existing setting (for example `MINIMAPS_URL`). It is either the hosted service or a self-hosted server. Never invent a production host. If none is configured, ask the user.
- **Keys.** The hosted service uses keys; a self-hosted server with `MINIMAPS_AUTH=none` (the default) needs none. Look for environment variables holding an `sk_` secret key, or a `pk_` publishable key with its signing secret. Read keys from the environment. Never hard-code them.
- **SDK.** If `@minimaps/sdk` is in `package.json`, or `minimaps` is in the Python dependencies, use it. The SDK builds URLs from the spec, signs them, and turns errors into one class with a `code`. Without an SDK, build URLs with the platform's URL/query encoder, never by string concatenation.
- **Where the request runs.** It is either server code (it can hold secrets) or browser/mobile code (anything in it is public).

## 2. GET URL or POST body

| Situation | Use |
|---|---|
| `<img src>`, Markdown, email, anything a browser fetches | A GET URL. On the hosted service it must be **signed with a publishable key** (section 3). |
| Server-side bytes (PDF, upload to storage, attachments) | `POST /v1/map` with the JSON spec and `Authorization: Bearer sk_…`. |
| A durable image URL to store in a database | `POST /v1/map?response=json`. It returns `{url, key, meta}`, and `url` (`/v1/images/{key}`) is immutable. If it later returns 404 `not-found`, POST again. |
| The spec uses a POST-only field | POST. The SDKs' `url()` raises `PostOnlyError` instead of silently dropping fields. |

POST-only fields:

<!-- generated:skill-post-only:start -->
- `shapes`
- `routes.1+`
- `font.family (custom fonts)`
- `area (GeoJSON)`
- `pois.*.marker.icon (long data URIs work but POST is recommended)`
<!-- generated:skill-post-only:end -->

A minimal locator map as a GET request:

<!-- example: skill-locator -->
```http
GET /v1/map.png?area=Czechia&poi=place:Chřiby|marker:triangle&w=468&h=280
```

The same map as a POST body:

<!-- example: skill-locator-post -->
```json
{
  "area": "Czechia",
  "pois": [{ "place": "Chřiby", "marker": { "shape": "triangle" } }],
  "width": 468,
  "height": 280
}
```

## 3. Keys and signed URLs (hard rules)

- **Never put an `sk_` secret key in a URL**, in HTML, in a client bundle, in a mobile app or in a log. The server rejects a secret key in a URL with 403 `forbidden`. A key that leaked like that must be rotated. Secret keys go only in the `Authorization: Bearer sk_…` header, from server code.
- **Embeds use a publishable key and a signature.** The URL carries `apikey=pk_…` and `sig=…`. The signature is an HMAC that only the signing secret can produce, and it covers the path and every parameter. Compute it **on the server**: with the SDK's `url()` (when the client has the publishable key and signing secret), with `signUrl`/`sign_url`, or with the algorithm in [recipes](reference/recipes.md#signing-without-an-sdk). **The signing secret never reaches a browser.** A browser gets finished URLs from your backend.
- **Sign last.** Changing anything after signing invalidates the URL, including the parameters and the path (`.png` and `.webp` are different paths). Sign decoded values: a space is signed as `%20`, even when the URL shows `+`.
- **Expiry.** `exp=` (Unix seconds, signed) limits how long a leaked URL works. To keep URLs cacheable, round it, for example to the next hour or day. A fresh `exp` on every page view defeats browser and CDN caches and spends quota.
- A self-hosted server without keys ignores `apikey=`, `sig=` and `exp=`, so the same code works against both servers.

## 4. Names, `place:` lookups and their errors

- `area=` takes a name (`Zlínský kraj`) or a stable id: `iso:CZ-72`, `osm:r51684`, `wikidata:Q213`. Store ids, not names, because a name can become ambiguous later.
- Points and route ends take `lat,lon` (used as is, with no lookup) or `place:Name` (a lookup, which also becomes the default label): `poi=place:Brno`, `from=place:Brno`.
- **409 `ambiguous`**: several areas share the name. The body has `candidates[]`, each with a `ref` (for example `osm:r439539`), an `adminLevel` and a `context` ("Česko › Liberecký kraj › …"). Show the contexts to the user, or choose one in code. Then repeat the request with `area=<ref>` and store that ref. You can also narrow the search up front with `level=` and `country=`. Never retry the same name unchanged.
- **404 `not-found`**: no match. The body's `suggestions[]` holds close matches. Offer them to the user; don't guess.
- Look names up ahead of time (for example in a form's autocomplete) with `GET /v1/areas/search?q=` and `GET /v1/places/search?q=`.

<!-- example: skill-area-by-ref -->
```http
GET /v1/map.png?area=osm:r442590&w=360&h=260
```

<!-- example: skill-narrowed nonrender -->
```http
GET /v1/map.png?area=Zlín&level=8&country=CZ&w=360&h=260
```

## 5. Errors and retries

Every error is `application/problem+json`. Branch on the code after `#` in `type` (the SDKs expose it as `err.code`), never on `detail` text.

<!-- generated:skill-errors:start -->
| Status | Code | Title | Extra fields |
|---|---|---|---|
| 400 | `invalid-spec` | Invalid map spec | `issues` |
| 400 | `bad-request` | Bad request | — |
| 404 | `not-found` | Not found | `suggestions` |
| 409 | `ambiguous` | Ambiguous name | `candidates` |
| 401 | `unauthorized` | Unauthorized | — |
| 403 | `forbidden` | Forbidden | — |
| 403 | `plan-limit` | Not included in your plan | — |
| 429 | `rate-limited` | Too many requests | — |
| 429 | `quota-exceeded` | Monthly quota exceeded | — |
| 422 | `no-route` | No route found | — |
| 503 | `busy` | Busy | — |
| 502 | `upstream-failed` | Upstream failure | `detail` |
| 504 | `upstream-timeout` | Upstream timeout | — |
<!-- generated:skill-errors:end -->

- **Retry** `busy` and `rate-limited` after `Retry-After` seconds, and retry 502/504 with backoff and jitter. Give up after a few attempts.
- **Don't retry** other 4xx codes, because the answer will not change. `quota-exceeded` lasts until next month, though already-cached maps keep working. `invalid-spec` lists each bad field in `issues[]`, and unknown parameters are rejected (with a "did you mean") rather than ignored.
- Log `requestId` from the body (also sent as the `X-Request-Id` header) with every error.

## 6. Features worth knowing

Parameter names by topic. The full table is in [reference/parameters.md](reference/parameters.md).

<!-- generated:skill-param-index:start -->
- Area and framing: `area` `level` `country` `bounds` `padding` `projection` `globeview`
- Points: `poi`
- Route: `from` `to` `via` `profile` `polyline` `polylineprecision` `routeprovider` `routecolor` `routewidth` `routecasing` `routedash` `routeopacity` `fromlabel` `tolabel` `frommarker` `tomarker` `showfrom` `showto`
- Image: `w` `h` `scale` `format` `quality` `attribution`
- Basemap: `preset` `style` `key`
- Highlight: `fade` `fadecolor` `outline` `outlinewidth` `tint` `tintopacity` `subdivisions` `desaturate` `blur` `subarea` `subareacolor` `subareaopacity` `subareaoutlinewidth` `worldview`
- Neighboring countries: `neighbors` `neighborfit` `neighborwhich` `neighborsize` `neighborcolor` `neighborupper`
- Labels and typography: `lang` `font` `fontweight`
- Title band: `title` `titlepos` `titlealign` `titlesize` `titleweight` `titlecolor` `titlebg`
- Inside `poi=` (`lat,lon|key:value|…` or `place:Name|…`): `label:` `pos:` `labelsize:` `weight:` `labelcolor:` `leader:` `marker:` `color:` `size:` `stroke:` `strokewidth:` `icon:` `anchor:`
<!-- generated:skill-param-index:end -->

- **Defaults are good.** Highlight colors, fade and outline follow the basemap preset (`preset=`). Set only what the user asked for.
- **Routes.** `profile=straight` and `profile=geodesic` (flights) are drawn locally and always work. Routed profiles (`car`, `bike`, `foot`) need a routing engine on the server or an `X-Routing-Key` header. Without one they return 400 `bad-request`, and on plans without routing they return 403 `plan-limit`. The default profile is `geodesic`, so a route without `profile=` always renders; ask for `profile=car` (or `bike`/`foot`) only when the user wants roads and routing is available. If the app already has a line, pass it as `polyline=`. GET carries one route; use POST for more.
- **Neighbors.** `neighbors=flag+code` (or `name`, `flag+name`) labels the bordering countries. With the default `neighborfit=extend`, **the image can grow** past `w`×`h` so labels fit; the real size is in `X-Image-Width`/`X-Image-Height`. When the layout needs an exact size, use `neighborfit=shrink` or `neighborfit=drop`.
- **Typography.** Labels use local names by default (Deutschland, Polska); `lang=en` switches the language. Bundled fonts are set with `font=` and `fontweight=`. Arabic, Hebrew, Devanagari, Thai, Georgian and Armenian work. CJK does not yet, so it needs a custom font (POST). `title=` draws a band *inside* the requested size.
- **Retina.** Keep `w`/`h` at the CSS size and add `scale=2`. In HTML, set `width`, `height` and a meaningful `alt` on the `<img>`.

<!-- example: skill-route-neighbors -->
```http
GET /v1/map.png?area=Czechia&from=place:Brno&to=place:Zlín&profile=straight&neighbors=flag+code&neighborfit=shrink&w=468&h=280
```

<!-- example: skill-title-lang -->
```http
GET /v1/map.webp?area=Czechia&poi=place:Brno|label:Our+office&lang=en&title=Where+we+are&titlealign=center&w=468&h=300&scale=2
```

## 7. Limits

<!-- generated:skill-limits:start -->
| JSON field | GET parameter | Limit | Default |
|---|---|---|---|
| `width` | `w` | 64–2048 | `600` |
| `height` | `h` | 64–2048 | `400` |
| `scale` | `scale` | 1–3 | `1` |
| `quality` | `quality` | 1–100 | `90` |
| `padding` | `padding` | 0–0.45 | `0.04` |
| `pois` | `poi` | at most 50 items | `[]` |
| `pois[].label` | `label:` in `poi` | at most 160 characters | — |
| `routes` | one route in GET | at most 10 items | `[]` |
| `routes[].via` | `via` | at most 20 items | `[]` |
| `shapes` | POST only | at most 50 items | `[]` |
| `title.text` | `title` | at most 200 characters | — |
| `neighbors.max` | POST only | 1–30 | `12` |
<!-- generated:skill-limits:end -->

On the hosted service, plans also cap the output pixels (`w`×`h`×`scale`²), monthly renders and requests per minute. Exceeding them gives `plan-limit`, `quota-exceeded` or `rate-limited`. Cache hits are free, so **keep URLs stable**. The same spec gives the same URL, the same `ETag` and a cache hit. To force a new URL for the same map, add a parameter starting with `_` (for example `_v=2`), which the server ignores.

## 8. URL pitfalls

- Encode `#` in colors as `%23` (`fadecolor=%23dedede`). A literal `+` in a query string means a space.
- `|` separates POI options, so a label containing `|` needs POST. A `\n` (backslash-n) in a label starts a new line.
- Repeat `poi=` (and `via=`) once per point; don't comma-join them.
- Prefer the `X-Provider-Key` and `X-Routing-Key` headers over putting third-party keys in URLs.
