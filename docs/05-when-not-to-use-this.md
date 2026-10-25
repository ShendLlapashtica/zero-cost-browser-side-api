# When not to use this, and how to survive the one real dependency

[← back to README](../README.md)

The pattern has one genuine dependency and three limits. Only the dependency
is dangerous, and it is the one people skip when they write about this.

## The dependency: cross-origin access is a gift, not a contract

Nothing obliges an upstream to keep answering browsers. The system measured
here lost that access for about three and a half weeks — the upstream began
refusing any request carrying an `Origin` header, then resumed — with no
notice either way.

When it closes, **every browser integration you shipped dies at the same
moment**, and the failure is silent: the errors are in other people's
browsers. This is the risk you are actually taking on, and it is worth
engineering for rather than hoping about.

### Make the single point of failure a ladder

Treat the direct fetch as the first rung, not the only one. Each rung below it
must be chosen so that it cannot generate a bill, or the fallback becomes a
worse problem than the outage:

| Rung | Mechanism | Cost shape |
|---|---|---|
| 1 | Browser → upstream directly | zero |
| 2 | Browser → your own thin relay at the edge, which adds the CORS header server-side | zero inside an edge platform's free allowance, which typically **stops** rather than bills on overrun |
| 3 | Browser → a static snapshot on your CDN, written at build time | zero, but not live |
| 4 | Browser → your metered API | the only rung that costs per request |

Two properties make this work. First, **a relay does not send an `Origin`
header**, because it is not a browser — so an `Origin`-based refusal does not
apply to it. That means rung 2 survives exactly the failure that kills rung 1.
Second, pick a platform whose free tier *fails closed*: one that refuses
requests when the daily allowance is gone rather than charging for the
overage. Then the fallback is structurally incapable of producing a bill, and
you can leave it armed permanently instead of reaching for a switch during an
incident.

Before relying on rung 2, verify one thing by measurement: **many public
catalogue APIs refuse datacenter addresses.** An edge relay runs in a
datacenter. If the upstream blocks that address range, rung 2 is dead on
arrival and you need rung 3. These are two independent blocks — one by header,
one by address — and they fail independently.

### Give the silence a sentinel

You cannot alert from a visitor's browser. If you also operate a server-side
path, route your *own* site through the same mechanism as the widget and
monitor your own site. Then the alarm you already have fires on your own
traffic the moment the browser route closes. Without that, this
architecture's failure mode is quiet, and quiet failures are the expensive
kind.

## Limit 1: no shared state, by construction

No cross-visitor cache, no "removed"/"sold" markers, no change history, no
"new since last visit" feed. Anything requiring knowledge of yesterday needs a
server. For a browse-and-view surface this costs nothing; if your product is
built on change detection, this pattern cannot carry it.

## Limit 2: raw upstream shapes, with no shock absorber

There is no server in the path to normalise a response, so an upstream field
rename reaches the screen directly, weeks later, as blank content in somebody
else's browser.

Three real examples from one integration, each found only by testing against
live data:

- a price lived at `advertisement.price`; the obvious-looking `category.price`
  did not exist at all, and the neighbouring `category.originPrice` was a
  different figure entirely
- an image URL was `path` on one endpoint and `location` on another
- a presented row kept the raw payload under a sub-key, so a helper reading
  the top level returned `null` for every image

The mitigation is not cleverness, it is discipline: **assert on live payloads,
prefer denylists to whitelists when passing fields through**, and never trust
a field name you have not seen in a real response. A whitelist silently
deletes any shape you did not anticipate, which is the same bug class as the
rename.

## Limit 3: a share of visitors block third-party requests

Some browsers and networks block cross-origin requests to hosts they do not
recognise. Those visitors see nothing, you cannot measure it, and you cannot
fix it from your side. Whether that share matters is an audience question, not
a technical one — measure it for your own traffic before deciding.

## Cases where the server-in-the-middle wins outright

- **The upstream needs a credential.** A browser cannot hold a secret; there
  is no hiding a key in a bundle.
- **The upstream host is commercially sensitive.** This pattern publishes it
  to every visitor's developer tools.
- **You need a shared cache.** A thousand visitors on one popular record is
  one upstream request plus 999 cache hits on a server, and 1,000 upstream
  requests here.
- **You are contractually responsible for the data's accuracy**, and need a
  single place to validate and correct it.
- **You bill per request.** You cannot meter what never reaches you.

## The honest summary

You are trading **a bill you control** for **a dependency you do not**. With a
fallback ladder and a sentinel on the silence, that trade is usually worth it.
Without them it is a cost saving that works right up until the morning it
doesn't.

---

[← back to README](../README.md)
