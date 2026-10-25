# Pacing: a request budget is not enough, shape is what gets you blocked

[← back to README](../README.md)

Once the browser is the client, there is nothing upstream of your code that
throttles it. No key, no quota, no server of yours in the way. A loop in the
page — yours, or one introduced by whoever maintains the page later — will
hammer the upstream from a single address as fast as the browser allows.

The consequence is not your bill. **It is the visitor's address getting
blocked**, and a blocked visitor sees an empty page and tells nobody. You will
not find out from logs you do not have.

## A rolling budget alone does not work

The first version enforced a rolling-minute budget: 72 requests per minute,
shared by every client on the page, enforced inside the fetch path where no
page misconfiguration could reach it. Correct, and insufficient. Measured
behaviour:

```
all 72 requests left within 3 milliseconds,
then the page sat idle for the remaining 59.997 seconds
```

72 requests in 3 ms from one address is precisely the pattern that gets an
address cut off. The budget was never exceeded. The *shape* was the problem.

## Minimum gap, widening by itself

Add a second rule: a request also waits for a minimum interval since the
**previous request started**, and that interval widens as the minute fills.

```js
// [requests already used this minute, minimum ms since the previous start]
const GAP_LADDER = [[12, 150], [36, 500], [Infinity, 1200]];

function nextSlotDelay(now) {
  stamps = stamps.filter(t => now - t < WINDOW_MS);
  const budget = stamps.length < RATE_PER_MIN ? 0 : stamps[0] + WINDOW_MS - now;
  const gap    = lastStart + gapFor(stamps.length) - now;
  return Math.max(budget, gap, 0);
}
```

The ladder is the whole trick. A real page makes one to three requests, so it
never leaves the first rung and the pacing is invisible. A loop climbs the
ladder within seconds and is throttled to a crawl without ever being told
"no".

Measured after the change, same burst of 20 concurrent calls:

```
starts (ms): 1 164 328 490 649 811 971 1133 1293 1454 1616 1776 2276 2788 …
gaps   (ms):   163 164 162 159 162 160 162 160 161 162 160  500  512  514 …
minimum gap: 159 ms          total span: 5,825 ms
```

Twelve requests at ~160 ms, then the ladder steps to 500 ms. Nothing failed,
nothing errored, and no burst left the browser.

## Serialise admission, or the gap leaks

This is the bug that will catch you. Two callers asking at the same tick both
read the same "no wait needed" and both go immediately — a page that fires a
search and two detail lookups on load is exactly that case. Admission has to
be serialised through one chain:

```js
let queue = Promise.resolve();

function admit(signal) {
  const turn = queue.then(() => waitForSlot(signal));
  queue = turn.then(() => {}, () => {});   // a refusal must not wedge the chain
  return turn;
}
```

Note the `catch` on the stored promise. If a rejected admission is left in the
chain, every later request inherits the rejection and the widget dies quietly.

## Retries are the path around every guard

The retry is the exact moment a well-behaved client becomes a hammer, and in
the first version it was the one path that bypassed both the budget and the
pacer: it retried inside an already-granted slot, 400 ms apart.

Two fixes, both of which generalise to any client you write:

1. **Every attempt goes through admission**, not just the first. A retry is a
   real request; treat it as one.
2. **Back off in seconds, not milliseconds**, with jitter so a page's parallel
   calls do not re-collide: `1000 * (attempt + 1) + random(0..400)`.

## Obey `429` instead of arguing with it

A `429` is the upstream naming the visitor's address. It is the clearest
warning you will ever get, and the correct response is to stop:

```js
const after = Number(res.headers?.get?.('retry-after'));
if (Number.isFinite(after) && after > 0) err.retryAfterMs = Math.min(after * 1000, 30000);
```

Honour `Retry-After`, clamp it so a hostile or broken value cannot park the
page for an hour, and treat the `429` itself as sufficient grounds to open the
[circuit breaker](04-circuit-breaker-and-self-healing.md) immediately rather
than spending three more failures proving what the upstream already said.

## A per-tab ceiling, in memory only

A rolling budget limits rate but not total. A tab left open with a loop in it
can still make 100,000 requests in a day inside a 72-per-minute budget. So
cap the lifetime of a tab:

```js
const SESSION_CAP = 1500;   // a human browsing uses a few dozen
```

**Keep it in memory and nowhere else.** Persisting it to storage means a
visitor can return tomorrow to a page that refuses to load because of
something that happened yesterday — worse than the problem it solves. Every
reload starts clean.

Measured with the cap lowered to 5 for the test: nine calls produced exactly
five requests and then a clear error. The real value of 1,500 is unreachable
by a human and reachable by a loop in about a minute.

## Do not fetch for a page nobody is looking at

Two free wins that also reduce load:

- **Viewport gating.** A widget below the fold waits until it is near the
  viewport before fetching. Measured: a page with two grids made **one**
  request on load, and the second only when the visitor scrolled down.
- **Visibility gating.** A hidden tab fetches nothing; it waits for
  `visibilitychange`. A background tab that fetches nothing cannot get its
  visitor blocked.

Both are a few lines, both are invisible when the content is on screen, and
together they cut request volume on a long page by more than any cache would
have.

## Clamp every option, trust none

The options surface is where a plausible-looking value does real damage, and
on this architecture there is no upstream of you to catch it:

```js
const clamp = (v, lo, hi, dflt) => {
  const n = Number(v);
  return Number.isFinite(n) ? Math.min(hi, Math.max(lo, n)) : dflt;
};

const timeout = clamp(options.timeout, 5000, 30000, 20000);
const retries = clamp(options.retries, 0, 2, 1);
```

`retries: 50` turns one dead search into fifty requests. `timeout: 1` makes
every call fail and retry forever. Pull the value into range and proceed —
nothing should refuse to work over a bad setting, it should just refuse to do
the damaging version of it.

And if there is a switch in the file that can cost money, **make it a
constant, not an option.** A page attribute means the page — or an assistant
editing that page months later — can turn your spending on by typing one word,
with no way for you to see it happen.

---

Next: [circuit breaker and self-healing](04-circuit-breaker-and-self-healing.md)
