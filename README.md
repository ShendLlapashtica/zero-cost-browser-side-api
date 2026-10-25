# Zero-Cost Browser-Side API Architecture

**How to serve live third-party data on a website at zero marginal
infrastructure cost — by deleting the server from the request path.**

A production architecture write-up with measured numbers: request counts,
payload sizes, rate-limit shapes, circuit-breaker behaviour and the failure
modes nobody writes about. No framework, no dependency, no server.

> I run data integrations where the per-request cost decides whether the
> product exists. This repository documents one pattern that took a live
> integration's marginal cost from cents-per-thousand-requests to **zero**,
> what it cost me in control, and the exact measurements behind both claims.
>
> — [Shend Llapashtica](https://shend.dev) · [github.com/ShendLlapashtica](https://github.com/ShendLlapashtica)

---

## The problem in one paragraph

You resell or re-present data from a third-party catalogue API. The obvious
build is a server in the middle: your API calls theirs, caches the result,
normalises it, hands it to your front end. That server is also the only thing
in the system that costs money per request — a function invocation, a few
cache commands, and egress bytes, on every single call. At a few thousand
requests a day it is a rounding error. At a few hundred thousand it is a line
item that grows with your traffic whether or not your revenue does.

## The pattern

If the upstream API answers browsers directly — `Access-Control-Allow-Origin`
on its responses — then the visitor's own browser can be the client. You ship
**one static JavaScript file** from a CDN. It renders the data, and every data
request after that is made by the visitor, from the visitor's own address, at
the visitor's own expense.

```
Server-in-the-middle                     Browser-direct
─────────────────────                    ──────────────
visitor → your API  (invocation)         visitor → CDN (static file, cached)
        → your cache (commands)          visitor → upstream API  (their cost)
        → egress proxy (bytes)
        → upstream API

cost: per request, forever               cost: zero per request
```

What that removes, per request: **one function invocation, every cache
command, and every egress byte.** Not reduced. Removed — there is no server in
the path to bill for.

## Measured results

From a live production integration. Every figure below was measured against
the deployed build in a real browser, not estimated.

| What | Measurement |
|---|---|
| Static bundle, gzipped | **37,341 bytes**, cached by the browser after first load |
| Requests per page view (grid of 8 records) | **13 total** — 1 to my CDN (the file), 2 data fetches, 8 images straight from the vendor's image CDN |
| Requests that touch my infrastructure | **1**, ever: the script download |
| Latency, data fetch | ~440–570 ms direct, roughly **10× faster** than the same query through my own server path (no proxy hop) |
| Function invocations per data request | **0** |
| Cache/database commands per data request | **0** |
| Egress bytes on my side per data request | **0** |

The comparison figures for the server-in-the-middle path, from the same
system's billing: per-command cache pricing worked out to roughly **1 cent per
1,000 requests**, and residential egress was the larger cost — a per-gigabyte
plan ran $25–36/month before being replaced by flat-rate ISP addresses at
**$4.70/month per address with 50 GB included.** Those are the numbers that go
to zero.

## What it is not

This pattern is not a cache, not a proxy, and not a CDN trick. It is a
**relocation of the client**. That relocation is the entire benefit and the
entire cost:

- **Always live.** There is no cache in the path, so every record on screen
  was fetched during that page view. No staleness, no "last updated", nothing
  to invalidate — a property that is true by construction rather than by
  assertion.
- **No shared state.** No cross-visitor cache, no "sold/removed" markers, no
  price-change history, no new-arrivals feed. Anything that requires knowing
  what the data looked like yesterday needs a server, and stays on one.
- **The upstream sees the visitor, not you.** Their rate limits apply per
  visitor address, which is why there is effectively no shared limit to hit —
  and also why protecting the visitor's address becomes your job.

## Read next

| Document | What it covers |
|---|---|
| [Architecture](docs/01-architecture.md) | Both request paths in full, what moves to the browser, what cannot |
| [Cost model](docs/02-cost-model.md) | Where per-request cost actually comes from, with real billing figures |
| [Pacing and rate limits](docs/03-pacing-and-rate-limits.md) | Why a request *budget* is not enough and request *shape* is what gets you blocked |
| [Circuit breaker and self-healing](docs/04-circuit-breaker-and-self-healing.md) | Failing without hammering, and recovering with nobody watching |
| [When not to use this](docs/05-when-not-to-use-this.md) | The four failure modes, honestly, and the cases where a server wins |

## Who this is for

Developers and technical founders who:

- resell, aggregate or re-present a third-party catalogue or listings API
- are watching serverless function invocations, per-command cache billing or
  residential proxy egress grow with traffic
- need **live** data on a page rather than a nightly sync
- ship embeddable widgets to clients and want them to cost nothing to run
- are deciding between a server-side proxy, an edge function and a
  browser-side client

## Keywords

zero-cost architecture · browser-side API client · client-side data fetching ·
CORS direct fetch · serverless cost reduction · function invocation cost ·
per-command cache billing · residential proxy cost · static CDN widget ·
embeddable widget architecture · rate limit pacing · circuit breaker pattern ·
self-healing front end · live data without cache · third-party API integration
· marginal cost per request · API reselling architecture

---

## About the author

I build and operate third-party data integrations end to end: egress,
proxying, caching tiers, cost control, client-facing APIs and the embeddable
front ends that consume them. The patterns here come from systems in
production with paying clients, where the cost per request is the business
model.

- Site: **[shend.dev](https://shend.dev)**
- GitHub: **[@ShendLlapashtica](https://github.com/ShendLlapashtica)**

Available for work on API integration, cost reduction on metered
infrastructure, and embeddable data widgets. Open an issue or reach me through
the site.

## Licence

[MIT](LICENSE) — the writing and the patterns. Deliberately no vendor names,
no endpoints and no client details appear anywhere in this repository.
