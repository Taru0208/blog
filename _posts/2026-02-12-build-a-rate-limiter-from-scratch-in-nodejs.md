---
layout: post
title: "Build a Rate Limiter from Scratch in Node.js"
date: 2026-02-12
tags: [nodejs, api, security, tutorial]
description: "Three rate limiting algorithms implemented step by step. Fixed window, sliding window, and token bucket — with standard HTTP headers and middleware integration."
---

Every API needs rate limiting. Without it, a single client can hammer your server into the ground — whether by accident or on purpose. Most developers reach for a library, but rate limiters are small enough to build yourself and understand completely.

Let's implement three algorithms, compare their tradeoffs, and wire them into a real API.

## Fixed window

The simplest approach. Count requests in a time window (say, 60 seconds). Reset when the window expires.

```javascript
function fixedWindow({ windowMs = 60_000, max = 100 } = {}) {
  const counters = new Map();

  return function check(key) {
    const now = Date.now();
    const entry = counters.get(key);

    if (!entry || now >= entry.resetAt) {
      counters.set(key, { count: 1, resetAt: now + windowMs });
      return { allowed: true, remaining: max - 1, resetAt: now + windowMs };
    }

    if (entry.count >= max) {
      return { allowed: false, remaining: 0, resetAt: entry.resetAt };
    }

    entry.count++;
    return { allowed: true, remaining: max - entry.count, resetAt: entry.resetAt };
  };
}
```

**The problem:** a burst at the boundary. If a client sends 100 requests at 0:59 and another 100 at 1:01, they've made 200 requests in 2 seconds — but each falls in a different window.

**Use when:** you need something simple and the burst problem is acceptable.

## Sliding window log

Track the timestamp of every request. Count how many fall within the window.

```javascript
function slidingWindowLog({ windowMs = 60_000, max = 100 } = {}) {
  const logs = new Map();

  return function check(key) {
    const now = Date.now();
    const cutoff = now - windowMs;

    let entries = logs.get(key) || [];
    // Remove expired entries
    entries = entries.filter(ts => ts > cutoff);

    if (entries.length >= max) {
      logs.set(key, entries);
      return { allowed: false, remaining: 0, resetAt: entries[0] + windowMs };
    }

    entries.push(now);
    logs.set(key, entries);
    return { allowed: true, remaining: max - entries.length, resetAt: now + windowMs };
  };
}
```

**The problem:** memory. Storing every timestamp means O(max) memory per client. At 1000 requests/minute across 10,000 clients, that's 10 million timestamps.

**Use when:** you need precise counting and have a low request limit.

## Sliding window counter

The practical middle ground. Interpolate between two fixed windows to approximate a sliding window — without storing individual timestamps.

```javascript
function slidingWindowCounter({ windowMs = 60_000, max = 100 } = {}) {
  const windows = new Map();

  return function check(key) {
    const now = Date.now();
    const currentWindow = Math.floor(now / windowMs);
    const windowStart = currentWindow * windowMs;
    const elapsed = (now - windowStart) / windowMs; // 0.0 to 1.0

    const entry = windows.get(key) || { prev: 0, curr: 0, window: currentWindow };

    // Advance window if needed
    if (entry.window < currentWindow) {
      entry.prev = entry.window === currentWindow - 1 ? entry.curr : 0;
      entry.curr = 0;
      entry.window = currentWindow;
    }

    // Weighted count: full current + proportional previous
    const weight = 1 - elapsed;
    const count = Math.floor(entry.prev * weight) + entry.curr;

    if (count >= max) {
      windows.set(key, entry);
      return { allowed: false, remaining: 0, resetAt: windowStart + windowMs };
    }

    entry.curr++;
    windows.set(key, entry);
    const newCount = Math.floor(entry.prev * weight) + entry.curr;
    return { allowed: true, remaining: max - newCount, resetAt: windowStart + windowMs };
  };
}
```

This stores only two numbers per client regardless of request volume. The interpolation isn't perfectly accurate, but it's close enough for rate limiting — and it eliminates the boundary burst problem.

**Use when:** you want sliding window accuracy with fixed window memory cost. This is what most production APIs use.

## Token bucket

A different mental model: each client has a bucket of tokens. Requests consume tokens. Tokens refill at a steady rate.

```javascript
function tokenBucket({ capacity = 100, refillRate = 10, refillMs = 1000 } = {}) {
  const buckets = new Map();

  return function check(key) {
    const now = Date.now();
    let bucket = buckets.get(key);

    if (!bucket) {
      bucket = { tokens: capacity - 1, lastRefill: now };
      buckets.set(key, bucket);
      return { allowed: true, remaining: bucket.tokens };
    }

    // Refill tokens based on elapsed time
    const elapsed = now - bucket.lastRefill;
    const refill = Math.floor(elapsed / refillMs) * refillRate;
    bucket.tokens = Math.min(capacity, bucket.tokens + refill);
    bucket.lastRefill = now;

    if (bucket.tokens < 1) {
      return { allowed: false, remaining: 0 };
    }

    bucket.tokens--;
    return { allowed: true, remaining: bucket.tokens };
  };
}
```

Token bucket allows bursts up to `capacity`, then enforces a steady rate of `refillRate` per interval. This is useful when you want to allow short bursts while enforcing a long-term average.

**Use when:** you want to allow controlled bursts. Good for user-facing APIs where occasional spikes are normal.

## Standard HTTP headers

Whichever algorithm you choose, respond with the standard rate limit headers:

```javascript
function setRateLimitHeaders(res, result, max) {
  res.setHeader('X-RateLimit-Limit', String(max));
  res.setHeader('X-RateLimit-Remaining', String(result.remaining));
  if (result.resetAt) {
    res.setHeader('X-RateLimit-Reset', String(Math.ceil(result.resetAt / 1000)));
  }
  if (!result.allowed) {
    res.setHeader('Retry-After', String(
      Math.ceil((result.resetAt - Date.now()) / 1000)
    ));
  }
}
```

These headers tell clients:
- **X-RateLimit-Limit** — the maximum requests allowed
- **X-RateLimit-Remaining** — requests left in the current window
- **X-RateLimit-Reset** — Unix timestamp when the window resets
- **Retry-After** — seconds until the client should retry (only on 429)

## Middleware integration

Wrapping any of these into Express or Node.js http middleware:

```javascript
function rateLimitMiddleware({ algorithm, max, keyFn }) {
  const check = algorithm;

  return (req, res, next) => {
    const key = keyFn ? keyFn(req) : req.ip;
    const result = check(key);

    setRateLimitHeaders(res, result, max);

    if (!result.allowed) {
      res.writeHead(429, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        error: 'Too many requests',
        retryAfter: Math.ceil((result.resetAt - Date.now()) / 1000),
      }));
      return;
    }

    next();
  };
}

// Usage
const limiter = slidingWindowCounter({ windowMs: 60_000, max: 60 });
app.use(rateLimitMiddleware({
  algorithm: limiter,
  max: 60,
  keyFn: (req) => req.headers['x-api-key'] || req.ip,
}));
```

## Memory cleanup

All in-memory implementations leak memory if you don't clean up expired entries:

```javascript
function withCleanup(store, windowMs) {
  const cleanup = setInterval(() => {
    const now = Date.now();
    for (const [key, entry] of store) {
      const lastActivity = entry.resetAt || entry.lastRefill || 0;
      if (now - lastActivity > windowMs * 2) {
        store.delete(key);
      }
    }
  }, windowMs * 2);

  // Don't prevent process exit
  if (cleanup.unref) cleanup.unref();
}
```

Run cleanup at 2x the window interval. More frequent is wasteful; less frequent means stale entries pile up.

## Comparison

| Algorithm | Memory | Accuracy | Burst handling |
|-----------|--------|----------|----------------|
| Fixed window | O(1) per client | Low (boundary burst) | Poor |
| Sliding log | O(max) per client | Exact | Good |
| Sliding counter | O(1) per client | ~99.7% accurate | Good |
| Token bucket | O(1) per client | N/A (different model) | Controlled bursts |

For most APIs: **sliding window counter** is the right default. Low memory, good accuracy, no boundary problem.

For APIs that need burst tolerance: **token bucket**.

For quick prototypes: **fixed window** — just know its limitations.

## When to use Redis instead

In-memory rate limiting works for single-process servers. When you scale to multiple processes or machines, you need a shared store. Redis is the standard choice — `INCR` with `EXPIRE` gives you a fixed window in two commands.

```
MULTI
INCR rate:${key}
EXPIRE rate:${key} 60
EXEC
```

But don't reach for Redis by default. Most applications run on a single server, and in-memory rate limiting is simpler, faster, and has zero operational overhead.
