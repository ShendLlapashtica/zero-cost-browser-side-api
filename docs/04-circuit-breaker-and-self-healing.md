# Failing without hammering, recovering with nobody watching

[← back to README](../README.md)

A browser-side integration has no operator. Nobody is tailing logs when it
breaks, because the logs are in other people's browsers. So the client has to
handle both halves of an outage by itself: **stop asking** while the upstream
is down, and **start again** when it comes back.

## The breaker

When an upstream stops answering, continuing to ask is not merely useless — it
is the single behaviour most likely to turn a brief block into a long one.

```js
const BREAKER_FAILS   = 4;        // consecutive failures before opening
const BREAKER_BASE_MS = 30000;    // first cooldown
const BREAKER_MAX_MS  = 600000;   // ceiling, doubling each time

function breakerGate() {
  if (!openUntil) return;
  const now = Date.now();
  if (now < openUntil) throw new ApiError('upstream not answering; waiting');
  // Re-arm BEFORE probing, so a page full of callers sends one request
  // to find out rather than fifty.
  openUntil = now + cooldown;
}
```

Three details that matter more than the structure:

- **Re-arm before the half-open probe.** Otherwise every pending caller probes
  at once the moment the cooldown expires, and you have replaced a flood with
  a slightly later flood.
- **A `404` is not an outage.** A record that does not exist says nothing about
  the upstream's health. Counting it opens the breaker during normal use.
- **A cancelled request is not a failure.** A caller that navigated away, or a
  timeout the caller chose, must not count — otherwise ordinary page
  navigation trips the breaker.

```js
function noteFail(err) {
  if (err?.status === 404) return;                     // missing ≠ down
  if (err?.status === 429) fails = BREAKER_FAILS;      // believe it immediately
  else fails++;
  if (fails >= BREAKER_FAILS) {
    openUntil = Date.now() + cooldown;
    cooldown  = Math.min(cooldown * 2, BREAKER_MAX_MS);
  }
}
```

### Measured

Against a completely dead upstream, with the page making twelve searches:

| | requests actually sent |
|---|---|
| Before the breaker | **24** (12 searches × 2 attempts), and it would have continued |
| With the breaker | **4**, then silence |
| On a `429` | **1**, then silence |

Everything after the fourth failure is refused inside the browser and costs
nothing — no request leaves, no socket opens.

## Never persist breaker state

Tempting, and wrong. A breaker state written to local storage means a visitor
who arrived during a five-minute outage returns tomorrow to a page that still
refuses to load. Keep it in memory: a reload is always a clean start, and the
cost of that choice is at most one wasted request per page view during an
outage.

The same rule applies to the per-tab request ceiling. Any guard that can
*deny service* must not outlive the page, or you have built a landmine for a
visitor who did nothing wrong.

## Self-healing: "momentarily unavailable" has to be momentary

A failed render that stays failed is indistinguishable from a broken product.
The fix is a retry schedule that widens, is bounded, and only runs when a
human could actually be looking:

```js
const RETRY_STEPS = [20000, 45000, 90000, 180000];  // then every 5 min
const RETRY_TAIL  = 300000;
const RETRY_MAX   = 24;                             // ~2 hours, then stop
```

Rules that make it safe:

- **Only while the tab is visible.** A background tab retrying for hours is
  invisible load with nobody to benefit from it.
- **Stop if the element left the page.** The host page may have replaced it;
  it is not yours to refill any more.
- **Bounded.** Twenty-four attempts is far past any real page view. Unbounded
  retries are a cron job you did not mean to write.
- **Reset the counter on success**, so a later failure starts the schedule
  over rather than being treated as attempt twenty.

The elegant part is the interaction with the breaker: **while the breaker is
open, a retry costs no request at all**, because it is refused locally. The
schedule is therefore free for exactly as long as the upstream is down, and
fires a real request in the first window after it recovers.

### Measured, in a real browser

The upstream was blocked at the network layer, the page loaded, the block was
lifted, and then nothing touched the page:

```
  6s  upstream blocked  → 0 cards, "momentarily unavailable, retrying…", 2 requests
  6s  block lifted. Nothing touches the page from here on.
 11s  0 cards, 2 requests        (breaker open — the retry cost nothing)
 16s  0 cards, 2 requests
 21s  0 cards, 2 requests
 26s  4 cards, 3 requests        ← recovered by itself
```

Twenty seconds, three requests in total, no human involved.

## What you still cannot do from the browser

You cannot alert. Nobody is watching the console in a visitor's browser, so a
breaker that opens in a thousand browsers is silent to you.

The cheap answer, if you also run a server-side path: make your own site use
the same route as the widget, and monitor your own site. Then the alarm you
already have fires on your own traffic when the browser route closes, and the
widget's silence has a sentinel. Without something like that, this
architecture's failure mode is *quiet*, and quiet failures are the expensive
kind.

---

Next: [when not to use this](05-when-not-to-use-this.md)
