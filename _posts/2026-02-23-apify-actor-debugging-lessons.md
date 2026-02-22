---
layout: post
title: "Three Debugging Lessons from Building Apify Actors"
date: 2026-02-23
tags: [python, apify, debugging, logging]
description: "Python's standard logging is invisible in Apify cloud. The Actor context manager silently swallows exceptions. And that 'totalUsers' metric doesn't mean what you think."
---

I built a batch of Apify Actors — Python wrappers around public APIs like SEC EDGAR, Crossref, and Eurostat. The code was straightforward. The debugging wasn't.

Here are three things that cost me hours and aren't in the docs.

## 1. Your logs are invisible

Python's standard logging module doesn't work in Apify cloud:

```python
import logging
logger = logging.getLogger(__name__)

# This produces zero output in Apify's log console
logger.info("Processing 100 items...")
logger.error("API returned 500!")
```

The Apify SDK configures its own logging pipeline. Standard Python loggers write to a handler that goes nowhere. Your Actor runs, hits an error, exits cleanly — and you see nothing.

The fix:

```python
from apify import Actor

# In main.py — use Actor.log
Actor.log.info("Processing 100 items...")
Actor.log.error("API returned 500!")

# In library modules (api.py etc.) — use print()
# Apify captures stdout
print("[api] Request failed: timeout")
```

`Actor.log` writes to Apify's structured log. `print()` writes to stdout, which Apify also captures. Either works; `logging.getLogger()` doesn't.

## 2. `async with Actor` eats your exceptions

The standard Actor pattern looks like this:

```python
async def main() -> None:
    async with Actor:
        actor_input = await Actor.get_input() or {}
        # ... do work ...
```

If an unhandled exception occurs inside that `async with` block, the Actor context manager catches it and exits with code 0. No traceback. No error in the log (especially if you're using the invisible `logging` module from lesson 1). The run shows "SUCCEEDED" with 0 dataset items.

This is what happened to me: an `httpx` timeout propagated up, `async with Actor` caught it, and the Actor exited cleanly. The only signal was an empty dataset — which I assumed was an input configuration issue, not a crash.

The fix is explicit exception handling:

```python
async def main() -> None:
    async with Actor:
        actor_input = await Actor.get_input() or {}
        try:
            results = await fetch_data(actor_input)
            for item in results:
                await Actor.push_data([item])
            Actor.log.info(f"Done. {len(results)} results.")
        except Exception as e:
            Actor.log.error(f"Failed: {type(e).__name__}: {e}")
            raise  # Re-raise so the run status shows FAILED
```

The `raise` is important. Without it, the Actor exits with code 0 and Apify's auto-test thinks everything is fine.

## 3. Metrics lie (at first)

Apify's API returns a `totalUsers` field for each Actor. After publishing, I saw `totalUsers: 2` on every Actor — including unpublished ones. For a moment, this looked like organic discovery.

It's not. The `2` is a system baseline — likely counting the account owner and possibly an internal Apify reference. Every Actor starts at 2 regardless of actual usage.

To check real usage, look at `totalRuns` and subtract your own test runs. Or check individual run logs for runs you didn't trigger. At day 2 post-publish, the honest number for all my Actors was: 0 external users.

This matters because premature optimization based on vanity metrics wastes time. "Actor X has 2 users, Actor Y has 2 users" tells you nothing. Wait for differentiated numbers before making decisions about which Actors to improve.

## Summary

| Problem | Symptom | Fix |
|---------|---------|-----|
| `logging.getLogger` invisible | No log output in Apify console | Use `Actor.log` or `print()` |
| Silent exception swallowing | SUCCEEDED with 0 items | Add `try/except` with `raise` |
| Baseline metric inflation | All Actors show 2 users | Ignore until numbers differentiate |

The common thread: Apify's platform has its own conventions that diverge from standard Python practices. The docs cover the happy path. The debugging path, you learn the hard way.
