# Architecture: moving the client into the browser

[← back to README](../README.md)

Two paths deliver the same records to the same page. The difference is which
machine makes the request, and that decides who pays.

## The server-in-the-middle path

```
┌─────────┐    ┌──────────────┐    ┌───────────┐    ┌──────────────┐
│ visitor │ →  │ your API     │ →  │ cache     │ →  │ egress proxy │ → upstream
│ browser │ ←  │ (invocation) │ ←  │ (commands)│ ←  │ (bytes)      │ ←
└─────────┘    └──────────────┘    └───────────┘    └──────────────┘
```

Every arrow on that diagram is metered by somebody. A single page view of a
grid plus two detail lookups is three invocations, a handful of cache reads
and writes, and the full payload crossing your egress twice — once inbound
from upstream, once outbound to the visitor.

What you get for it is real and worth naming: a normalised response shape, a
shared cache across all visitors, your own rate limits and keys, the upstream
host hidden behind yours, and the ability to know what the data looked like
yesterday.

## The browser-direct path

```
┌─────────┐    ┌─────────────────┐
│ visitor │ →  │ CDN: static .js │   (once, then browser-cached)
│ browser │    └─────────────────┘
│         │ →  upstream API      (visitor's address, visitor's bytes)
│         │ →  upstream image CDN
└─────────┘
```

Your infrastructure appears exactly once, serving a static file. After that
the visitor talks to the upstream directly and your side is idle.

The preconditions are narrow and you must verify each one rather than assume
it:

1. **The upstream answers cross-origin requests.** Specifically it returns
   `Access-Control-Allow-Origin` on the endpoints you need. Verify with an
   explicit `Origin` header, not from a terminal with no origin at all — those
   are different requests and they can get different answers.
2. **The endpoints you need are reachable without a secret.** A browser cannot
   hold a credential. Anything in the file is public, so an API that needs a
   key cannot be called this way — there is no "hiding" a key in a bundle.
3. **Simple requests only.** A browser cannot set `Origin`, `Referer` or
   `User-Agent`; they are forbidden header names. Adding an `Accept` header
   turns a simple request into a preflighted one, which doubles the request
   count for nothing. Send no headers at all.
4. **The response shape is stable enough to read raw.** There is no server to
   normalise it. See [when not to use this](05-when-not-to-use-this.md).

## What moves to the browser, and what cannot

Everything a server adds on top of raw upstream data splits cleanly into two
piles.

**Reproducible in the browser** — compile the tables into the bundle and do it
locally:

- translation and localisation of upstream field values
- currency conversion and display formatting
- unit conversion and derived figures
- category/classification mapping
- query construction, including whatever filter syntax the upstream uses
- resolving a human name to the upstream's own internal identifier, with a
  dictionary first and a live facet lookup as the fallback

That last one is worth a note: the fallback query costs nothing here, because
the visitor's browser pays for it. Decisions that were expensive on a server
become free, which changes what designs are reasonable.

**Not reproducible** — requires knowing history or seeing all visitors:

- "removed" / "sold" / "no longer listed" markers
- change history of any kind
- a feed of what is new since last time
- any cache shared between visitors
- aggregate analytics across visitors
- anything rate-limited, keyed or billed per client

The honest architecture is therefore usually **both**: the browser path serves
browse, search and detail views at zero cost; the server path keeps the
features that need state, and gets a fraction of the traffic.

## Shipping it

The build is deliberately dull: bundle the module twice, commit the output,
let the CDN serve it.

```
src/widget.js  ──bundle──▶  public/widget.js    (IIFE, sets a global)
                            public/widget.mjs   (ESM, for import)
```

Two outputs because a plain `<script src>` tag cannot load an ES module —
`export` is a syntax error there — and the whole point of the simple path is
the script tag:

```html
<script src="https://cdn.example.com/widget.js"></script>
<div data-records data-filter="value" data-count="12"></div>
```

```js
import { createClient } from 'https://cdn.example.com/widget.mjs';
const api = createClient({ lang: 'de' });
const { total, results, live } = await api.cards({ filter: 'value', count: 24 });
```

**Commit the build output.** It feels wrong until you remember that the file
being on the CDN *is* the product; there is no server to build it on demand.

### One pitfall worth stealing

If you hand-port the upstream's query builder into the browser copy, that copy
will silently drift from the server's, and the symptom arrives weeks later as
a `400` in somebody else's browser. Pin it: have the build read the server's
function **as source text**, compile it in isolation, and diff the generated
query strings across a spread of inputs. Fail the build on any difference.

Read it as text rather than importing it — importing a serverless handler
drags in the HTTP client, the proxy ladder and the cache, so the check would
break on an unrelated dependency and tell you nothing about the query string.

---

Next: [the cost model](02-cost-model.md) · [pacing](03-pacing-and-rate-limits.md)
