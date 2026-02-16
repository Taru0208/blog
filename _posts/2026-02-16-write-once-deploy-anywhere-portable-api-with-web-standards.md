---
layout: post
title: "Write Once, Deploy Anywhere: Building a Portable API with Web Standards"
date: 2026-02-16
tags: [nodejs, cloudflare-workers, api, web-standards, architecture]
description: "Build an API that runs on Node.js and Cloudflare Workers with zero code changes. The secret: use Request and Response instead of Express."
---

Most API tutorials start with Express. But Express ties you to Node.js. If you later want to deploy on Cloudflare Workers, Deno, or Bun — you're rewriting everything.

There's a better approach: build your API around the [Web-standard `Request` and `Response`](https://developer.mozilla.org/en-US/docs/Web/API/Request) APIs. These work everywhere. You write your logic once, then swap out a 5-line entry point to deploy on any runtime.

I used this pattern to build a utility API with 100+ endpoints that runs identically on Node.js (local dev) and Cloudflare Workers (production, free tier) with zero code duplication.

## The core idea

Instead of `(req, res) => { res.json({}) }`, write handlers that take a `Request` and return a `Response`:

```javascript
async function handleRequest(request) {
  const url = new URL(request.url);
  const path = url.pathname;

  if (path === '/hello') {
    return new Response(
      JSON.stringify({ message: 'hello' }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  }

  return new Response(
    JSON.stringify({ error: 'Not found' }),
    { status: 404, headers: { 'Content-Type': 'application/json' } }
  );
}
```

This function knows nothing about Node.js, Cloudflare, or any specific runtime. It's pure Web API.

## A lightweight router

Hardcoding paths gets messy fast. Here's a minimal router that stays framework-free:

```javascript
const routes = new Map();

function get(path, handler) { routes.set(`GET:${path}`, handler); }
function post(path, handler) { routes.set(`POST:${path}`, handler); }

// Register routes
get('/health', async () => {
  return { status: 'ok', uptime: process.uptime?.() ?? 0 };
});

post('/text/slugify', async (body) => {
  const slug = body.text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '');
  return { slug };
});

post('/hash', async (body) => {
  const encoder = new TextEncoder();
  const data = encoder.encode(body.text);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return { hash: hashArray.map(b => b.toString(16).padStart(2, '0')).join('') };
});
```

Notice: handlers take a parsed body and return a plain object. The router handles serialization:

```javascript
export async function handleRequest(request) {
  const url = new URL(request.url);
  const method = request.method;
  const path = url.pathname.replace(/\/$/, '') || '/';

  // CORS preflight
  if (method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: corsHeaders() });
  }

  const handler = routes.get(`${method}:${path}`);
  if (!handler) {
    return jsonResponse({ error: 'Not found', path }, 404);
  }

  try {
    let body = {};
    if (method === 'POST') {
      const text = await request.text();
      if (!text) return jsonResponse({ error: 'Body required' }, 400);
      body = JSON.parse(text);
    }
    const result = await handler(body, request);
    return jsonResponse(result);
  } catch (e) {
    return jsonResponse({ error: e.message }, 400);
  }
}

function jsonResponse(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { 'Content-Type': 'application/json', ...corsHeaders() },
  });
}

function corsHeaders() {
  return {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type',
  };
}
```

That's the entire framework. About 40 lines. No dependencies.

## The deployment adapters

Here's where portability pays off. Each runtime needs a thin adapter — and nothing else.

**Cloudflare Workers** (`worker.js`):

```javascript
import { handleRequest } from './router.js';

export default {
  async fetch(request) {
    return handleRequest(request);
  },
};
```

Three lines. Workers already speak `Request`/`Response` natively, so there's nothing to translate.

**Node.js** (`server.js`):

```javascript
import { createServer } from 'node:http';
import { handleRequest } from './router.js';

const PORT = process.env.PORT || 3000;

const server = createServer(async (req, res) => {
  const url = `http://localhost:${PORT}${req.url}`;
  const body = await new Promise((resolve) => {
    const chunks = [];
    req.on('data', c => chunks.push(c));
    req.on('end', () => resolve(Buffer.concat(chunks).toString()));
  });

  const request = new Request(url, {
    method: req.method,
    headers: req.headers,
    body: ['POST', 'PUT', 'PATCH'].includes(req.method) ? body : undefined,
  });

  const response = await handleRequest(request);
  const responseBody = await response.text();

  res.writeHead(response.status, Object.fromEntries(response.headers.entries()));
  res.end(responseBody);
});

server.listen(PORT, () => console.log(`Running on http://localhost:${PORT}`));
```

Node.js needs a bit more plumbing to convert its native `IncomingMessage` to a Web `Request`. But this is boilerplate you write once and never touch again.

**Bun/Deno** would be even simpler — both support `Request`/`Response` natively, similar to Workers.

## Testing without a runtime

Since handlers are pure functions (input → output), testing is straightforward:

```javascript
import { handleRequest } from './router.js';
import { describe, it } from 'node:test';
import assert from 'node:assert';

describe('API', () => {
  it('slugifies text', async () => {
    const req = new Request('http://test/text/slugify', {
      method: 'POST',
      body: JSON.stringify({ text: 'Hello World!' }),
    });
    const res = await handleRequest(req);
    const data = await res.json();
    assert.strictEqual(data.slug, 'hello-world');
  });

  it('hashes text', async () => {
    const req = new Request('http://test/hash', {
      method: 'POST',
      body: JSON.stringify({ text: 'hello' }),
    });
    const res = await handleRequest(req);
    const data = await res.json();
    assert.strictEqual(data.hash.length, 64); // SHA-256 hex
  });

  it('returns 404 for unknown routes', async () => {
    const req = new Request('http://test/nope');
    const res = await handleRequest(req);
    assert.strictEqual(res.status, 404);
  });
});
```

No mocking, no test servers, no supertest. Just call the function with a `Request`, check the `Response`. The tests run the same code that runs in production.

## Deploying to Cloudflare Workers

Cloudflare Workers give you a free tier of 100,000 requests/day. For a utility API, that's more than enough.

**Setup:**

```bash
# Install wrangler CLI
npm install -g wrangler

# Login
wrangler login
```

**`wrangler.toml`:**

```toml
name = "my-api"
main = "src/worker.js"
compatibility_date = "2024-01-01"
```

**Deploy:**

```bash
wrangler deploy
```

That's it. Your API is live at `my-api.<your-subdomain>.workers.dev` with HTTPS, global CDN, and zero infrastructure management.

## When to use this pattern

**Good fit:**
- Utility/stateless APIs (text processing, data transformation, validation)
- APIs you want to prototype locally and deploy to edge
- Projects where you might switch runtimes later
- Anything that doesn't need runtime-specific features (file system, native modules)

**Not great for:**
- Apps that need Express middleware ecosystem
- Heavy file I/O or streaming
- Apps tightly coupled to Node.js APIs (`child_process`, `fs`, `net`)

## The tradeoffs

You lose the Express middleware ecosystem. No `passport`, no `multer`, no `express-session`. For a utility API, you don't need them. For a full web app, you probably do.

You gain portability. The same test suite validates your code regardless of where it runs. You can develop on Node.js and deploy to Workers, or switch to Bun tomorrow, without touching business logic.

The Web `Request`/`Response` API is the closest thing we have to a universal HTTP interface across JavaScript runtimes. Building on it means your code ages well.

## Summary

1. Write handlers that take `Request`, return `Response`
2. Build a thin router with `Map` — no framework needed
3. Create runtime-specific entry points (3 lines for Workers, ~20 for Node.js)
4. Test by calling `handleRequest()` directly
5. Deploy anywhere

The pattern scales to 100+ endpoints without getting complicated. The framework you don't import is the framework you don't debug.
