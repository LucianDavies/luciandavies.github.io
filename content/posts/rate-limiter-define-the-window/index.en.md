---
title: "Fixed Window vs Sliding Window vs Token Bucket"
subtitle: "\"100 requests per minute\" is three different rules — a side-by-side guide"
date: 2026-09-20T00:00:00+01:00
lastmod: 2026-09-20T00:00:00+01:00
draft: false
author: "Tonderai Khatai"
authorLink: "https://www.linkedin.com/in/tldkhatai/"
description: "A teaching comparison of the three most common rate limiting algorithms. One traffic pattern, three different outcomes: 200, 100 and 103 requests accepted. Includes a runnable simulation, a comparison table, and a guide to choosing between them."
license: ""
images: []

tags: ["system-design", "rate-limiting", "architecture", "typescript", "algorithms"]
categories: ["system-design"]

featuredImage: "/images/posts/rate-limiter-define-the-window.png"
featuredImagePreview: "/images/posts/rate-limiter-define-the-window.png"

hiddenFromHomePage: false
hiddenFromSearch: false
---

Say a limiter allows 100 requests per minute. At 11:59:59 a user sends 100. At 12:00:01 they send 100 more. That's 200 requests in 2 seconds, and one common implementation lets all of them through.

<!--more-->

That isn't necessarily a bug. It's the point where "100 per minute" stops being one rule and becomes three. This post compares the three usual algorithms side by side, using that single traffic pattern as the test, and ends with a runnable simulation so you can check the numbers yourself. The scenario comes from a graphic by Puneet Patwari on LinkedIn; the code and explanation are my own.

## Bottom Line

| | **Fixed window** | **Sliding window log** | **Token bucket** |
|---|---|---|---|
| **The rule it enforces** | 100 per *calendar* minute | 100 in *any* rolling 60s | Burst of up to 100, refilling at 100/min |
| **Result on our traffic** | **200** accepted | **100** accepted | **~103** accepted |
| **Worst-case burst** | 2× the limit across a boundary | Exactly the limit | Capacity, plus refill during the burst |
| **State per client** | 1 counter | Up to `limit` timestamps | 2 numbers (tokens, last refill time) |
| **Cost per request** | O(1) | O(1) amortised, O(n) worst-case eviction | O(1) |
| **Distributed (Redis)** | `INCR` + `EXPIRE` | Sorted set: `ZADD` / `ZREMRANGEBYSCORE` / `ZCARD` | Small Lua script (read, refill, write atomically) |
| **Best when** | The limit *is* a per-period quota | The limit must hold over every interval | Bursts are fine, sustained load isn't |

If you only remember one line: **define what the limit protects, then pick the algorithm that enforces that, not the one that's easiest to build.**

## The Test: One Traffic Pattern

Every algorithm below faces the identical input:

```text
11:59:59   →  100 requests
12:00:01   →  100 requests
```

A minute-long limit of 100 means different things depending on where you put the edges of the minute, so watch what happens at 12:00:00.

## Algorithm 1: Fixed Window

**The idea:** chop time into calendar minutes and keep one counter per minute. When a new minute starts, the counter starts at zero.

```typescript
function fixedWindowAllow(counts: Map<number, number>, now: number, limit = 100): boolean {
  const window = Math.floor(now / 60_000); // which calendar minute are we in?
  const used = counts.get(window) ?? 0;
  if (used >= limit) return false;
  counts.set(window, used + 1);
  return true;
}
```

**Walk-through:**

1. The 11:59:59 burst lands in window `11:59`. Counter goes 0 → 100. All accepted.
2. At 12:00:00 the window key changes to `12:00`. Counter is 0 again.
3. The 12:00:01 burst lands in the new window. Counter goes 0 → 100. All accepted.

**Result: 200 accepted.** No request was wrongly handled. Each minute, taken on its own, stayed within its limit.

**Strengths:** the cheapest possible limiter. One integer per client, trivially atomic in Redis, easy to explain to users ("you get 100 calls per minute, resetting on the minute").

**Weakness:** the boundary. A client who times their traffic around it can get up to **2× the limit** in a span far shorter than a minute.

## Algorithm 2: Sliding Window Log

**The idea:** remember the timestamp of every accepted request. To decide on a new one, forget anything older than 60 seconds and count what remains.

```typescript
function slidingLogAllow(log: number[], now: number, limit = 100): boolean {
  while (log.length && log[0] <= now - 60_000) log.shift(); // evict expired
  if (log.length >= limit) return false;
  log.push(now);
  return true;
}
```

**Walk-through:**

1. The 11:59:59 burst adds 100 timestamps. The log holds 100 entries.
2. At 12:00:01, a new request arrives. The oldest entries are only 2 seconds old, so nothing is evicted. The log still holds 100.
3. `100 >= 100`, so the request is rejected. Every request in the second burst is rejected the same way.

**Result: 100 accepted.** This is the strictest reading of the sentence: in *any* 60-second span, never more than 100.

**Strengths:** exact. No boundary trick works against it.

**Weakness:** memory. It stores up to `limit` timestamps per client. At 100 per minute that's trivial. At 10,000 per minute across a million clients it's a real bill, and every check touches that data.

**The cheaper cousin, the sliding window counter:** keep two fixed-window counters (current and previous) and estimate the rolling count as `current + previous × (fraction of the previous window still in range)`. It's O(1) space and smooths out the boundary, at the cost of being an approximation that assumes traffic was spread evenly within the previous window.

## Algorithm 3: Token Bucket

**The idea:** no windows at all. A bucket holds up to `capacity` tokens and refills continuously at a fixed rate. Each request spends one token; no token, no entry.

```typescript
type Bucket = { tokens: number; last: number };

function tokenBucketAllow(b: Bucket, now: number, capacity = 100, perMin = 100): boolean {
  const refill = ((now - b.last) / 60_000) * perMin;
  b.tokens = Math.min(capacity, b.tokens + refill);
  b.last = now;
  if (b.tokens < 1) return false;
  b.tokens -= 1;
  return true;
}
```

**Walk-through** (bucket starts full: capacity 100, refill 100 per minute ≈ 1.67 tokens per second):

1. The 11:59:59 burst spends 100 tokens. The bucket is at 0.
2. Two seconds pass. Refill is `2 × 1.67 = 3.33` tokens.
3. The 12:00:01 burst gets those 3 tokens: **3 requests pass**, the other 97 are rejected.

**Result: ~103 accepted.**

**Strengths:** it separates two things the other algorithms fuse together: *how big a burst you tolerate* (capacity) and *how fast you sustain* (refill rate). Cheap to store and cheap to compute.

**Weakness:** "100 per minute" no longer describes the configuration completely. You now choose two numbers, and a capacity of 100 with a refill of 100/min is a different policy from capacity 10 with refill 100/min, even though both are "100 per minute".

## Try It Yourself

This is the whole experiment in one file. Save it as `sim.mjs` and run `node sim.mjs`:

```javascript
const fixed = () => { const c = new Map(); return (now, limit = 100) => { const w = Math.floor(now / 60_000); const u = c.get(w) ?? 0; if (u >= limit) return false; c.set(w, u + 1); return true; }; };
const sliding = () => { const log = []; return (now, limit = 100) => { while (log.length && log[0] <= now - 60_000) log.shift(); if (log.length >= limit) return false; log.push(now); return true; }; };
const bucket = () => { const b = { tokens: 100, last: 0 }; return (now, capacity = 100, perMin = 100) => { b.tokens = Math.min(capacity, b.tokens + ((now - b.last) / 60_000) * perMin); b.last = now; if (b.tokens < 1) return false; b.tokens -= 1; return true; }; };

const T = 12 * 60 * 60_000; // 12:00:00 in ms since midnight
const traffic = [...Array(100).fill(T - 1_000), ...Array(100).fill(T + 1_000)];

for (const [name, make] of Object.entries({ fixed, sliding, bucket })) {
  const allow = make();
  if (name === 'bucket') allow(T - 61_000); // let the bucket start full
  let ok = 0;
  for (const t of traffic) if (allow(t)) ok++;
  console.log(name, ok);
}
```

Output:

```text
fixed 200
sliding 100
bucket 103
```

Things worth changing to see what happens: move the second burst to `T + 59_000` (fixed window still allows it, sliding log still blocks it), or set the bucket's capacity to 10 and watch the first burst get clipped.

## Common Pitfalls

- **Assuming the sentence is the spec.** "100 per minute" fits all three algorithms. Write down which behaviour you need at the boundary.
- **Checking then incrementing in two steps.** In a distributed setup, a read followed by a write lets concurrent requests all see room and all pass. Use an atomic operation (`INCR`, or a Lua script for sliding log and token bucket).
- **Trusting client clocks.** Timestamps must come from the server (or from Redis itself), never from a header.
- **One global limit for everyone.** Decide the key deliberately: per user, per API key, per IP. Per-IP alone punishes people behind shared NATs and does nothing against a botnet.
- **Silent rejection.** Return `429` with a `Retry-After` header. A client can't back off sensibly if it doesn't know for how long.

## Choosing Between Them

Ask what the limit is *for*:

| The limit exists to… | Reach for | Because |
|---|---|---|
| Enforce a billing or plan quota per period | Fixed window | The reset at the boundary *is* the contract, and it's the cheapest |
| Protect a backend that breaks under spikes in any interval | Sliding window (log if small, counter if large) | It holds over every 60-second span, not only aligned ones |
| Allow short bursts but cap sustained load | Token bucket | Burst size and steady rate are separate, tunable knobs |
| Guard a login or password-reset endpoint | Sliding window, with a low limit | Boundary bursts matter most when the endpoint is an attack target |

None of these is "the correct" limiter. The mistake isn't picking fixed window. It's picking it without noticing that it lets 200 through, and finding that out during an incident.

## Attribution

The scenario and comparison graphic are Puneet Patwari's; the code, tables and write-up here are my own. His system design material is at [puneetpatwari.in](https://puneetpatwari.in).
