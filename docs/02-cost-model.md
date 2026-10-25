# The cost model: where per-request cost actually comes from

[← back to README](../README.md)

Most "reduce your cloud bill" writing optimises the wrong line. On a metered
third-party integration the cost is rarely compute — it is the **per-request
fixed charges** that scale exactly with traffic.

## The three meters

### 1. Function invocations

Every request to your API is one invocation, billed per invocation plus
duration. Platform free tiers hide this until they don't. The important
property is that it scales linearly with traffic and cannot be optimised below
one invocation per request, because that is the price of having a server.

### 2. Per-command cache or database billing

This is the one people get wrong, and it is worth being precise about.

Serverless key-value services commonly bill **per command**, not per byte and
not per connection. A single logical operation is rarely one command. A
request that:

- reads a cached payload,
- checks a rate-limit counter,
- increments that counter,
- writes a usage record,
- and records that it wrote one,

is **five or more commands**, and a careless implementation does far worse.
One system measured here was spending roughly **16 commands per request just
to record that it had spent some** — bookkeeping about bookkeeping.

The real figure, from a monthly bill rather than a half-day sample: about
**1 cent per 1,000 requests** at roughly 150,000 commands a day, or around
**€8/month** at that volume. Modest, but it is a floor that only goes up.

Two lessons that generalise:

- **Quote the monthly bill, not a count sampled across part of a day.** A
  half-day count doubled is not a projection, and billing periods have
  commands you did not think about.
- **Multi-get collapses, pipelines usually do not.** Batching five reads into
  one `MGET` is one command. Five commands in a pipeline is still five
  commands; a pipeline saves round-trips, not billing units.

### 3. Egress, and the proxy you did not want

If the upstream blocks datacenter addresses — and many public catalogue APIs
do — your server cannot reach it from a normal cloud host at all. You end up
buying residential or ISP addresses, and the pricing model matters enormously:

| Model | Shape | Measured |
|---|---|---|
| Per-gigabyte residential | pay per byte, forever | **$25–36/month** at modest volume |
| Flat-rate ISP addresses | fixed per address, bandwidth included | **$4.70/month per address, 50 GB included** |

Switching models cut that cost by roughly 85% at the same traffic, and the
marginal cost per request went to zero until the included bandwidth runs out
— about 10 GB/month of actual usage against a 50 GB allowance, in the system
measured here.

A detail worth copying: **ask for less payload.** The facet/aggregation tree
on a search response was **88% of the bytes** and 8.5× the response size on a
24-row query, for a field the listing view never opened. Turning it off was a
one-parameter change and it moved the egress bill more than any caching work.

## What the browser path removes

| Meter | Server-in-the-middle | Browser-direct |
|---|---|---|
| Function invocations | 1+ per request | **0** |
| Cache/DB commands | 2–16 per request | **0** |
| Egress bytes (yours) | full payload, twice | **0** |
| Proxy addresses needed | yes, if upstream blocks datacenters | **none** — visitors are residential by definition |
| Your cost per request | small, and linear in traffic | **zero** |

The one non-zero item left is the static file itself: **37,341 bytes
gzipped**, once per visitor, then browser-cached. At 100,000 unique visitors
that is about 3.7 GB of CDN bandwidth — inside the included allowance of any
paid plan and most free ones.

## The economics are not free, they are traded

You are not getting something for nothing; you are paying in a different
currency:

- **You lose the shared cache.** 1,000 visitors viewing the same record make
  1,000 upstream requests instead of one plus 999 cache hits. You do not pay
  for those, but the upstream absorbs them, and that is a relationship you
  should not abuse — see [pacing](03-pacing-and-rate-limits.md).
- **You lose the hidden upstream.** The host appears in the bundle and in
  every visitor's developer tools. If who you fetch from is commercially
  sensitive, this pattern publishes it. That is a business decision and it
  belongs to whoever owns the business, not to the implementation.
- **You lose normalisation.** No server means no place to absorb an upstream
  field rename.

## Checklist before you move something to the browser

1. Measure the monthly bill of the current path, per meter, before changing
   anything. A half-day sample is not a measurement.
2. Confirm the upstream answers cross-origin requests **with an explicit
   `Origin` header** set to the site that will be making the calls.
3. Confirm the endpoints need no credential.
4. Decide, explicitly and in writing, whether exposing the upstream host is
   acceptable for this integration.
5. Keep the features that need shared state on the server, and expect to run
   both paths.

---

Next: [pacing and rate limits](03-pacing-and-rate-limits.md) ·
[circuit breaker](04-circuit-breaker-and-self-healing.md)
