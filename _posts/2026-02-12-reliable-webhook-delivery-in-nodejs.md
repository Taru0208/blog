---
layout: post
title: "Reliable Webhook Delivery in Node.js"
date: 2026-02-12
tags: [nodejs, webhooks, architecture, tutorial]
description: "Building a webhook system that actually delivers. Retry logic, HMAC signatures, idempotency, and dead letter queues — all in ~200 lines of Node.js."
---

Webhooks sound simple: make an HTTP POST when something happens. But reliable webhook delivery is surprisingly hard. Network failures, timeouts, duplicate deliveries, and unresponsive endpoints turn a simple concept into an engineering problem.

Here's how to build a webhook system that actually works.

## The naive approach (and why it fails)

```javascript
// Don't do this
async function sendWebhook(url, payload) {
  await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
}
```

This fails silently when:
- The endpoint is temporarily down (503)
- The network times out
- The server returns an error
- The process crashes between event creation and delivery

## The architecture

A reliable webhook system has four components:

1. **Event table** — records what happened
2. **Delivery queue** — tracks pending deliveries
3. **Worker** — processes the queue with retries
4. **Dead letter queue** — stores permanently failed deliveries

```sql
CREATE TABLE webhook_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  type TEXT NOT NULL,
  payload TEXT NOT NULL,
  created TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE webhook_deliveries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  event_id INTEGER NOT NULL REFERENCES webhook_events(id),
  endpoint_url TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, delivered, failed, dead
  attempts INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 5,
  next_attempt TEXT NOT NULL DEFAULT (datetime('now')),
  last_error TEXT,
  last_status_code INTEGER,
  delivered_at TEXT,
  created TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_deliveries_pending
  ON webhook_deliveries(next_attempt)
  WHERE status = 'pending';
```

The partial index on `status = 'pending'` is critical — it keeps the queue scan fast even when the table has millions of completed rows.

## HMAC signatures

Recipients need to verify that webhooks actually came from you. The standard approach: sign the payload with a shared secret.

```javascript
import { createHmac } from 'node:crypto';

function signPayload(payload, secret) {
  const body = typeof payload === 'string' ? payload : JSON.stringify(payload);
  return createHmac('sha256', secret).update(body).digest('hex');
}

function buildHeaders(payload, secret) {
  const timestamp = Math.floor(Date.now() / 1000);
  const body = JSON.stringify(payload);
  const signature = signPayload(`${timestamp}.${body}`, secret);

  return {
    'Content-Type': 'application/json',
    'X-Webhook-Signature': `t=${timestamp},v1=${signature}`,
    'X-Webhook-Id': crypto.randomUUID(),
  };
}
```

The `X-Webhook-Id` header enables idempotency — recipients can deduplicate based on this ID if they receive the same webhook twice.

Including the timestamp in the signature prevents replay attacks. Recipients should reject signatures older than 5 minutes.

## Verification on the receiving side

```javascript
function verifyWebhook(body, signatureHeader, secret) {
  const parts = Object.fromEntries(
    signatureHeader.split(',').map(p => p.split('='))
  );

  const timestamp = parseInt(parts.t);
  const signature = parts.v1;

  // Reject old signatures (replay protection)
  const age = Math.floor(Date.now() / 1000) - timestamp;
  if (age > 300) return false; // 5 minute window

  const expected = createHmac('sha256', secret)
    .update(`${timestamp}.${body}`)
    .digest('hex');

  // Timing-safe comparison
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expected, 'hex')
  );
}
```

Use `timingSafeEqual` instead of `===` to prevent timing attacks.

## Exponential backoff with jitter

When a delivery fails, retry — but not immediately. Each retry should wait longer than the last:

```javascript
function nextRetryDelay(attempt) {
  // Exponential: 30s, 2m, 8m, 32m, ~2h
  const base = 30 * Math.pow(4, attempt);
  // Add jitter (±25%) to prevent thundering herd
  const jitter = base * 0.25 * (Math.random() * 2 - 1);
  return Math.min(base + jitter, 7200); // cap at 2 hours
}
```

The jitter prevents a wave of retries all hitting the same endpoint simultaneously after a recovery.

## The delivery worker

```javascript
const BATCH_SIZE = 10;
const TIMEOUT = 10000; // 10 seconds

async function processQueue(db) {
  const pending = db.prepare(`
    SELECT d.*, e.type, e.payload
    FROM webhook_deliveries d
    JOIN webhook_events e ON e.id = d.event_id
    WHERE d.status = 'pending'
    AND d.next_attempt <= datetime('now')
    ORDER BY d.next_attempt ASC
    LIMIT ?
  `).all(BATCH_SIZE);

  const results = await Promise.allSettled(
    pending.map(delivery => deliverOne(db, delivery))
  );

  return results;
}

async function deliverOne(db, delivery) {
  const payload = JSON.parse(delivery.payload);
  const secret = getEndpointSecret(delivery.endpoint_url);
  const headers = buildHeaders(payload, secret);

  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), TIMEOUT);

  try {
    const response = await fetch(delivery.endpoint_url, {
      method: 'POST',
      headers,
      body: JSON.stringify(payload),
      signal: controller.signal,
    });

    clearTimeout(timer);

    if (response.ok) {
      // Success — mark delivered
      db.prepare(`
        UPDATE webhook_deliveries
        SET status = 'delivered', delivered_at = datetime('now'),
            attempts = attempts + 1, last_status_code = ?
        WHERE id = ?
      `).run(response.status, delivery.id);
      return;
    }

    throw new Error(`HTTP ${response.status}`);
  } catch (err) {
    clearTimeout(timer);
    const newAttempts = delivery.attempts + 1;

    if (newAttempts >= delivery.max_attempts) {
      // Exhausted retries — move to dead letter
      db.prepare(`
        UPDATE webhook_deliveries
        SET status = 'dead', attempts = ?, last_error = ?
        WHERE id = ?
      `).run(newAttempts, err.message, delivery.id);
    } else {
      // Schedule retry
      const delay = nextRetryDelay(newAttempts);
      db.prepare(`
        UPDATE webhook_deliveries
        SET attempts = ?, last_error = ?,
            next_attempt = datetime('now', '+' || ? || ' seconds')
        WHERE id = ?
      `).run(newAttempts, err.message, Math.round(delay), delivery.id);
    }
  }
}
```

## Running the worker

The worker runs on a simple interval. No Redis, no Bull, no external queue infrastructure:

```javascript
const POLL_INTERVAL = 5000; // 5 seconds

setInterval(async () => {
  try {
    await processQueue(db);
  } catch (err) {
    console.error('Queue processing error:', err.message);
  }
}, POLL_INTERVAL);
```

For higher throughput, decrease the poll interval and increase the batch size. SQLite handles this comfortably up to hundreds of deliveries per second.

## Monitoring the queue

```javascript
function queueStats(db) {
  return db.prepare(`
    SELECT
      status,
      COUNT(*) as count,
      AVG(attempts) as avg_attempts
    FROM webhook_deliveries
    GROUP BY status
  `).all();
}
```

Expose this as a health endpoint. Alert if `pending` count grows faster than it shrinks, or if `dead` count spikes.

## The complete pattern

1. **Something happens** → Insert into `webhook_events`
2. **For each subscriber** → Insert into `webhook_deliveries` with status `pending`
3. **Worker polls** → Find pending deliveries where `next_attempt <= now`
4. **Deliver** → POST with HMAC signature and timeout
5. **Success** → Mark `delivered`
6. **Failure** → Increment attempts, schedule retry with exponential backoff
7. **Max retries** → Mark `dead`, alert operator
8. **Dead letters** → Manually retry or investigate

This entire system runs in a single Node.js process with SQLite. No message broker, no background job framework, no separate queue service. The database IS the queue — and SQLite is remarkably good at it.

## When to use a real queue

Upgrade to Redis/BullMQ when:
- You need sub-second delivery latency (SQLite polling has 1-5 second delay)
- Delivery volume exceeds ~1000/second sustained
- You need multiple worker processes on different machines

For most applications — SaaS webhooks, notification delivery, event forwarding — the SQLite approach handles the load with far less operational complexity.
