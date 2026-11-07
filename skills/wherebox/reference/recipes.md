# Recipes

Working code for the common integrations. The TypeScript snippets are type-checked against `@wherebox/sdk`, the Python snippets are checked against the `wherebox` package, and every request example is parsed with the map spec in this skill's CI.

Install the SDKs with `npm install @wherebox/sdk` (TypeScript, Node 20+, browsers and edge runtimes) or `pip install wherebox` (Python 3.9+, no dependencies).

## TypeScript: a signed `<img>` URL from a server route

Create the client on the server with the publishable key and its signing secret. `url()` then adds `apikey`, `exp` and `sig`. The browser only ever receives the finished URL.

<!-- snippet: ts-skill-signed-img -->
```ts
import { createClient, WhereboxError, PostOnlyError } from '@wherebox/sdk';

const wherebox = createClient({
  baseUrl: process.env.WHEREBOX_URL ?? 'https://maps.example.com',
  publishableKey: process.env.WHEREBOX_PUBLISHABLE_KEY,
  signingSecret: process.env.WHEREBOX_SIGNING_SECRET, // server only, never in a client bundle
});

// e.g. app/api/map-url/route.ts (any framework with web-standard Request/Response)
export async function GET(request: Request): Promise<Response> {
  const area = new URL(request.url).searchParams.get('area');
  if (!area) return Response.json({ error: 'area is required' }, { status: 400 });
  try {
    // Round the expiry so repeated views reuse one URL (and one cached image).
    const hour = 3600;
    const expiresIn = hour - (Math.floor(Date.now() / 1000) % hour) + hour;
    const src = await wherebox.url(
      { area, pois: [{ place: 'Brno', label: 'Our office' }], width: 468, height: 280, scale: 2 },
      { format: 'webp', expiresIn },
    );
    return Response.json({ src, width: 468, height: 280 });
  } catch (err) {
    if (err instanceof PostOnlyError) return Response.json({ error: err.message }, { status: 400 });
    if (err instanceof WhereboxError) return Response.json({ error: err.message }, { status: 400 });
    throw err;
  }
}
```

The request it signs:

<!-- example: skill-ts-signed-img -->
```http
GET /v1/map.webp?area=Czechia&poi=place:Brno|label:Our+office&w=468&h=280&scale=2
```

## TypeScript: render bytes on the server and handle names

`render()` sends `POST /v1/map` with the secret key in the `Authorization` header. A 409 carries `candidates` with ready-to-use `ref` values, and a 404 carries `suggestions`.

<!-- snippet: ts-skill-render-names -->
```ts
import { writeFile } from 'node:fs/promises';
import { createClient, WhereboxError } from '@wherebox/sdk';

const wherebox = createClient({
  baseUrl: process.env.WHEREBOX_URL ?? 'https://maps.example.com',
  secretKey: process.env.WHEREBOX_SECRET_KEY, // header only, never in a URL
});

type Outcome =
  | { kind: 'ok'; file: string }
  | { kind: 'choose'; options: Array<{ ref: string; label: string }> }
  | { kind: 'unknown'; suggestions: string[] };

export async function renderArea(area: string, file: string): Promise<Outcome> {
  try {
    const { image } = await wherebox.render({ area, width: 468, height: 280 });
    await writeFile(file, image);
    return { kind: 'ok', file };
  } catch (err) {
    if (!(err instanceof WhereboxError)) throw err;
    if (err.code === 'ambiguous') {
      // Let the user pick, then call renderArea(option.ref, file) and store the ref, not the name.
      return { kind: 'choose', options: (err.candidates ?? []).map((c) => ({ ref: c.ref, label: `${c.name} (${c.context ?? c.adminLevel})` })) };
    }
    if (err.code === 'not-found') return { kind: 'unknown', suggestions: (err.suggestions ?? []).map((s) => s.name) };
    const wait = err.retryAfter;
    if ((err.code === 'busy' || err.code === 'rate-limited') && wait !== undefined) {
      await new Promise((r) => setTimeout(r, wait * 1000));
      return renderArea(area, file); // one retry; real code should cap attempts
    }
    throw err;
  }
}
```

The body it sends for an ambiguous name (this example parses, but the server answers 409):

<!-- example: skill-ambiguous-post nonrender -->
```json
{ "area": "Nová Ves", "width": 468, "height": 280 }
```

Picking a candidate by its `ref`:

<!-- example: skill-ambiguous-ref -->
```json
{ "area": "osm:r442590", "width": 468, "height": 280 }
```

## Python: signed URLs and server-side renders

<!-- snippet: py-skill-client -->
```python
import os

from wherebox import Client, WhereboxError, PostOnlyError

client = Client(
    os.environ.get("WHEREBOX_URL", "https://maps.example.com"),
    secret_key=os.environ.get("WHEREBOX_SECRET_KEY"),            # server-side render()
    publishable_key=os.environ.get("WHEREBOX_PUBLISHABLE_KEY"),  # embeds
    signing_secret=os.environ.get("WHEREBOX_SIGNING_SECRET"),    # never sent to a browser
)

spec = {
    "area": "Czechia",
    "pois": [{"place": "Chřiby", "marker": {"shape": "triangle"}}],
    "neighbors": {"show": "flag+code", "fit": "shrink"},
    "width": 468,
    "height": 280,
}


def img_src(spec):
    try:
        return client.url(spec, format="webp", expires_in=24 * 3600)
    except PostOnlyError as err:
        raise ValueError(f"needs POST: {err.paths}") from err


def render_png(spec, path):
    try:
        result = client.render(spec)
    except WhereboxError as err:
        if err.code == "ambiguous":
            return [(c["ref"], c.get("context")) for c in err.candidates]  # let the user pick
        if err.code == "not-found":
            return [s["name"] for s in err.suggestions]
        raise
    with open(path, "wb") as f:
        f.write(result.image)
    return path
```

The same spec as a body:

<!-- example: skill-python-spec -->
```json
{
  "area": "Czechia",
  "pois": [{ "place": "Chřiby", "marker": { "shape": "triangle" } }],
  "neighbors": { "show": "flag+code", "fit": "shrink" },
  "width": 468,
  "height": 280
}
```

## Without an SDK: server-to-server

Send the secret key in the header and the spec as JSON:

```bash
curl -fsS -X POST "$WHEREBOX_URL/v1/map" \
  -H "Authorization: Bearer $WHEREBOX_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"area":"Czechia","width":468,"height":280}' -o czechia.png
```

<!-- example: skill-curl-body -->
```json
{ "area": "Czechia", "width": 468, "height": 280 }
```

## Signing without an SDK

Port this to the project's language when no SDK fits. The server verifies exactly this algorithm.

<!-- generated:skill-signing:start -->
Take the URL without `apikey` and `sig`, then:

1. Add `apikey=<your pk_ key>`, and `exp=<Unix seconds>` if the URL should expire.
2. Build the **canonical request**: the path, `?`, then every parameter except `sig`, sorted by name and then by value (UTF-16 code unit order), each written as `encodeURIComponent(name)=encodeURIComponent(value)` and joined with `&`. Repeated parameters stay repeated.
3. Compute `HMAC-SHA256` of the canonical request, using the signing secret (hex-decoded to bytes) as the key.
4. Encode the digest as base64url without padding, keep the first 32 characters, and add it as `sig`.

Two details matter. First, sign the *decoded* values. In a query string `+` and `%20` both mean a space, and your canonical request must encode a space as `%20` either way. A literal plus sign (`%2B`) stays `%2B`. Second, the order of parameters in the final URL does not matter, but nothing may be added, removed or changed after signing.

The reference lists the parameters you add: `apikey`, `sig` and `exp`.

Node.js, standard library only (CommonJS; in an ES module use `import { createHmac } from 'node:crypto'`):

```js
const { createHmac } = require('node:crypto');

function canonicalRequest(pathname, searchParams) {
  const pairs = [...searchParams.entries()].filter(([name]) => name !== 'sig');
  pairs.sort(([an, av], [bn, bv]) => (an < bn ? -1 : an > bn ? 1 : av < bv ? -1 : av > bv ? 1 : 0));
  const query = pairs.map(([name, value]) => `${encodeURIComponent(name)}=${encodeURIComponent(value)}`).join('&');
  return `${pathname}?${query}`;
}

/** url: absolute URL; signingSecret: the hex signing secret; expiresAt: optional Date. */
function signUrl(url, publishableKey, signingSecret, expiresAt) {
  const u = new URL(url);
  u.searchParams.delete('sig');
  u.searchParams.set('apikey', publishableKey);
  if (expiresAt) u.searchParams.set('exp', String(Math.floor(expiresAt.getTime() / 1000)));
  const sig = createHmac('sha256', Buffer.from(signingSecret, 'hex'))
    .update(canonicalRequest(u.pathname, u.searchParams))
    .digest('base64url')
    .slice(0, 32);
  u.searchParams.set('sig', sig);
  return u.toString();
}
```

Python, standard library only:

```python
import base64
import hashlib
import hmac
import time
from urllib.parse import parse_qsl, quote, urlsplit, urlunsplit

# Characters that JavaScript's encodeURIComponent leaves alone.
SAFE = "-_.!~*'()"


def _enc(text):
    return quote(text, safe=SAFE)


def _utf16_key(pair):
    # JavaScript compares strings by UTF-16 code units; Python compares code points.
    return (pair[0].encode("utf-16-be"), pair[1].encode("utf-16-be"))


def canonical_request(path, pairs):
    kept = sorted(((k, v) for k, v in pairs if k != "sig"), key=_utf16_key)
    return path + "?" + "&".join(_enc(k) + "=" + _enc(v) for k, v in kept)


def sign_url(url, publishable_key, signing_secret, expires_at=None):
    """url: absolute URL; signing_secret: the hex signing secret; expires_at: optional Unix seconds."""
    parts = urlsplit(url)
    pairs = [(k, v) for k, v in parse_qsl(parts.query, keep_blank_values=True) if k not in ("sig", "apikey")]
    pairs.append(("apikey", publishable_key))
    if expires_at is not None:
        pairs = [(k, v) for k, v in pairs if k != "exp"] + [("exp", str(int(expires_at)))]
    digest = hmac.new(
        bytes.fromhex(signing_secret),
        canonical_request(parts.path, pairs).encode("utf-8"),
        hashlib.sha256,
    ).digest()
    sig = base64.urlsafe_b64encode(digest).rstrip(b"=").decode("ascii")[:32]
    pairs.append(("sig", sig))
    query = "&".join(_enc(k) + "=" + _enc(v) for k, v in pairs)
    return urlunsplit((parts.scheme, parts.netloc, parts.path, query, ""))


# signed = sign_url("https://maps.example.com/v1/map.png?area=Czechia&w=468&h=280",
#                   PUBLISHABLE_KEY, SIGNING_SECRET, expires_at=time.time() + 3600)
```
<!-- generated:skill-signing:end -->

## A route and a flight

Straight segments through a via point, with labels at both ends:

<!-- example: skill-route-straight -->
```http
GET /v1/map.png?from=49.1951,16.6068&to=49.2266,17.6667&via=49.24,16.9&profile=straight&fromlabel=Brno&tolabel=Zlín&w=480&h=280
```

A great-circle flight line, dashed:

<!-- example: skill-route-geodesic -->
```http
GET /v1/map.png?from=50.1,14.26&to=40.64,-73.78&profile=geodesic&fromlabel=Prague&tolabel=New+York&routedash=2,1.5&w=520&h=300
```

Several routes in one map need POST:

<!-- example: skill-two-routes -->
```json
{
  "routes": [
    { "from": { "lat": 49.1951, "lon": 16.6068 }, "to": { "lat": 49.2266, "lon": 17.6667 }, "profile": "straight" },
    { "from": { "lat": 49.1951, "lon": 16.6068 }, "to": { "lat": 48.8, "lon": 16.64 }, "profile": "straight", "color": "#1565c0" }
  ],
  "width": 480,
  "height": 300
}
```
