---
layout: post
title: "Turning API Wrappers into MCP Servers on Apify"
date: 2026-02-26
tags: [python, mcp, apify, fastmcp, ai]
description: "I had 10 Apify Actors wrapping public APIs. I turned 6 of them into MCP servers that AI agents can call directly. Here's the architecture, the gotchas, and what I'd do differently."
---

I had a collection of Apify Actors — Python wrappers around public government APIs like SEC EDGAR, PubMed, Eurostat, and OpenFDA. They worked fine as batch data extractors. But with MCP (Model Context Protocol) gaining traction, I wanted AI agents to query these APIs directly.

The result: 6 MCP servers running on Apify's Standby mode, each reusing the API client code from its parent Actor. Here's what I learned.

## The Architecture

Each MCP server is a minimal Python project:

```
openfda-mcp-server/
├── Dockerfile
├── requirements.txt
├── __main__.py
├── src/
│   ├── __init__.py
│   ├── __main__.py    # ← Important: Apify runs `python -m src`
│   ├── api.py         # Reused from the data Actor
│   └── main.py        # FastMCP server + tool definitions
└── tests/
    └── test_api.py
```

The key insight: the `api.py` file is nearly identical to the one in the data Actor. A MCP server is just a different interface to the same API client.

## FastMCP + Apify Standby

[FastMCP](https://github.com/jlowin/fastmcp) makes the server side trivial:

```python
from fastmcp import FastMCP
import uvicorn

server = FastMCP(
    "OpenFDA Safety Data MCP Server",
    instructions="Access FDA safety data — drug adverse events, recalls, device incidents.",
)

@server.tool()
async def search_drug_events(
    drug_name: Annotated[str | None, "Drug name (e.g. 'aspirin')"] = None,
    max_results: Annotated[int, "Max results (1-500)"] = 25,
) -> str:
    # ... query the API, return JSON string
```

Apify's [Standby mode](https://docs.apify.com/platform/actors/running/standby) keeps the server alive between requests — it's essentially a managed container that spins up on first request and stays warm for `idleTimeoutSecs`. The entry point:

```python
async def main():
    async with Actor:
        if os.environ.get("APIFY_META_ORIGIN") != "STANDBY":
            Actor.log.error("This Actor must run in Standby mode.")
            return

        port = int(os.environ.get("ACTOR_STANDBY_PORT", "8080"))
        app = server.http_app(transport="streamable-http", path="/mcp")

        config = uvicorn.Config(app, host="0.0.0.0", port=port, log_level="info")
        await uvicorn.Server(config).serve()
```

The MCP endpoint ends up at `https://your-username--actor-name.apify.actor/mcp`.

## Gotcha #1: `src/__main__.py`

Apify's Python Docker image runs `python -m src`, not `python __main__.py`. If you only have a root-level `__main__.py`, you'll get:

```
RuntimeWarning: coroutine 'main' was never awaited
```

The fix: create `src/__main__.py` with a relative import:

```python
import asyncio
from .main import main
asyncio.run(main())
```

The root `__main__.py` (with `from src.main import main`) is for local development. The one inside `src/` is what Apify actually executes.

## Gotcha #2: FastMCP 3.x Breaking Changes

FastMCP 3.x removed the `description` parameter from the constructor. Use `instructions` instead:

```python
# FastMCP 2.x (broken)
server = FastMCP("Name", description="...")

# FastMCP 3.x (correct)
server = FastMCP("Name", instructions="...")
```

Also, `http_app()` requires explicit transport:

```python
app = server.http_app(transport="streamable-http", path="/mcp")
```

## Gotcha #3: Standby Activation via API

For a while I thought Standby could only be enabled through the Apify Console UI — which requires Playwright automation and dealing with Cloudflare challenges. Turns out it's a simple API call:

```bash
curl -X PUT "https://api.apify.com/v2/acts/{actorId}?token={token}" \
  -H "Content-Type: application/json" \
  -d '{
    "actorStandby": {
      "isEnabled": true,
      "desiredRequestsPerActorRun": 100,
      "maxRequestsPerActorRun": 200,
      "idleTimeoutSecs": 300,
      "build": "latest",
      "memoryMbytes": 256
    }
  }'
```

This isn't documented prominently, but the `actorStandby` field on the Actor update endpoint works perfectly. The entire deploy workflow — create Actor, upload source, build, enable Standby, set public visibility — can be done purely through the API.

## Gotcha #4: Memory Sizing

MCP servers are lightweight HTTP processes. They don't need the 1024MB default — 256MB is plenty for a Python FastMCP + uvicorn + httpx stack. On Apify's free tier (8192MB total), this means you can run more servers simultaneously.

## The Deploy Script

I ended up with a single deploy script that handles all 16 actors (10 data + 6 MCP):

```python
ACTORS = {
    "EbqhiLZ8PPw3Hkdts": "kr-real-estate-transaction",
    "l1mum5gMOXWnZIw9k": "openfda-mcp-server",
    # ... 14 more
}
```

One complication: some actors use version "1.0", others "0.0" (depending on how they were created). The script tries "1.0" first, falls back to "0.0":

```python
for version in ("1.0", "0.0"):
    url = f"{BASE}/acts/{actor_id}/versions/{version}?token={TOKEN}"
    data, err = api_request(url, "PUT", payload)
    if err and "not found" in err.lower() and version == "1.0":
        continue
    break
```

## What I'd Do Differently

**Start with MCP, not batch Actors.** The batch Actor pattern (paginate, push_data, dataset) is useful for ETL, but the market for API wrappers on Apify is dominated by social media scrapers with 100K+ users. Government API wrappers are niche — maybe 10-100 users ceiling. MCP servers for AI agents might have better growth potential since the market is new.

**Use a shared API client package.** Right now each MCP server has its own copy of `api.py`. A monorepo with a shared client library would reduce duplication. But for 6 servers, copy-paste is honestly fine.

**Test the MCP tools directly.** My tests cover the API client layer, not the FastMCP tool wrappers. Since the tools are thin (parse params → call API → format JSON), this works in practice. But for more complex tools with business logic, you'd want to test the tool functions themselves.

## The Numbers

| Metric | Value |
|--------|-------|
| MCP servers | 6 |
| Total tests | 374 |
| Standby memory per server | 256 MB |
| API key required | No (all free public APIs) |
| Time to build one server | ~30 minutes |
| Deploy method | Pure API (no browser automation) |

All six servers are live on the [Apify Store](https://apify.com/aligned_kettledrum) — Crossref, SEC EDGAR, PubMed, Eurostat, US Census, and OpenFDA. Each connects to any MCP-compatible client (Claude Desktop, Cursor, etc.) via the Standby URL.
