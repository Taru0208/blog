---
layout: post
title: "What They Don't Tell You About Web Scraping"
date: 2026-02-21
tags: [python, web-scraping, api, networking]
description: "Most scraping tutorials show you BeautifulSoup on a static page. Real-world scraping is about geo-blocking, encoding, rate limits, and knowing when to use the API instead."
---

Most web scraping tutorials look like this: install BeautifulSoup, fetch a URL, parse some HTML, done. Real-world scraping is nothing like that. The code is the easy part. Everything else is harder.

Here's what I've learned from building scrapers for Korean and international websites.

## 1. Geo-blocking is the first wall

Before writing a single line of parsing code, check if the website is even accessible from your server's location.

Many websites block requests from foreign IP addresses. This is especially common with:

- E-commerce sites (inventory and pricing are region-specific)
- Real estate portals (local data, local users)
- Banking and financial services
- Government portals (some, not all)

A quick diagnostic:

```bash
# Check from your server
curl -s -m 10 -o /dev/null -w "HTTP %{http_code}" "https://target-site.com"
```

If you get `403`, `000` (timeout), or a redirect to a geo-block page, you have two options:

1. **Use a proxy in the target country.** Residential proxies cost $5-15/GB. Datacenter proxies are cheaper but more easily detected.
2. **Find the API instead.** Many websites that block scraping still have public APIs that work globally. Check browser DevTools Network tab — filter by XHR/Fetch requests.

Example: a major Korean real estate portal blocks all non-Korean IPs on its web pages, but the national government's public data portal (`data.go.kr`) provides the same real estate transaction data via open APIs accessible worldwide.

**The lesson:** don't scrape what you can API-call.

## 2. The website you see isn't the website you scrape

Modern websites are React/Vue/Angular SPAs. When you `curl` the URL, you get a nearly empty HTML shell with a JavaScript bundle. The actual content loads after JavaScript execution.

Three levels of rendering:

| Level | What you need | Speed | Cost |
|-------|--------------|-------|------|
| Static HTML | `requests` + BeautifulSoup | Fast | Lowest |
| Server-rendered | Same as above | Fast | Lowest |
| Client-rendered (SPA) | Headless browser (Playwright, Puppeteer) | Slow | Higher |

Before reaching for Playwright, check if the site has internal APIs. Open DevTools, go to the Network tab, interact with the page, and look at the XHR/Fetch requests. Most SPAs call clean JSON APIs internally. If you can call those APIs directly, you skip browser rendering entirely.

```python
# Instead of rendering the whole page...
# browser.goto("https://site.com/listings")

# ...call the API directly
import urllib.request
import json

url = "https://site.com/api/listings?region=seoul&page=1"
req = urllib.request.Request(url)
req.add_header("Referer", "https://site.com/")
req.add_header("User-Agent", "Mozilla/5.0 ...")

with urllib.request.urlopen(req) as resp:
    data = json.loads(resp.read().decode("utf-8"))
    # Clean, structured data — no parsing needed
```

## 3. Encoding will betray you

If you're scraping sites in non-Latin scripts (Korean, Japanese, Chinese, Arabic), encoding errors are almost guaranteed at some point.

The `requests` library guesses encoding from HTTP headers. If the server doesn't declare encoding (or declares it wrong), you get garbled text.

```python
import requests

resp = requests.get("https://korean-site.com")
# resp.text might be garbled if encoding detection fails

# Fix: use apparent_encoding
resp.encoding = resp.apparent_encoding
# Now resp.text should be correct

# Alternative: work with bytes + explicit decode
text = resp.content.decode("utf-8")
```

With BeautifulSoup:

```python
from bs4 import BeautifulSoup

# Option 1: let BS4 detect encoding (usually works)
soup = BeautifulSoup(resp.content, "html.parser")

# Option 2: specify encoding explicitly
soup = BeautifulSoup(resp.content, "html.parser", from_encoding="utf-8")
```

**Pro tip:** always save raw bytes to disk first, then parse. If parsing fails, you still have the original data to debug.

## 4. Rate limiting requires strategy, not speed

Getting blocked by rate limiting is the most common scraping failure. Most tutorials don't mention it because their examples fetch 3 pages.

At scale (hundreds or thousands of pages), you need:

```python
import time

class RateLimitedScraper:
    def __init__(self, delay=1.0):
        self.delay = delay
        self._count = 0

    def fetch(self, url):
        if self._count > 0:
            time.sleep(self.delay)
        self._count += 1

        # ... actual request logic ...
```

But fixed delays are crude. Better patterns:

- **Exponential backoff on errors:** if you get a 429 or 503, wait 2s, then 4s, then 8s.
- **Jitter:** add random variation to delays so your pattern doesn't look mechanical.
- **Concurrent but limited:** use a semaphore to allow N parallel requests (but not more).
- **Respect `Retry-After` headers:** some servers tell you exactly how long to wait.

```python
import time
import random

def backoff_fetch(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            resp = fetch(url)
            if resp.status == 429:
                wait = (2 ** attempt) + random.uniform(0, 1)
                time.sleep(wait)
                continue
            return resp
        except Exception:
            time.sleep(2 ** attempt)
    raise Exception(f"Failed after {max_retries} retries: {url}")
```

## 5. Headers matter more than you think

Many APIs check these headers and reject requests that look automated:

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
    "Referer": "https://target-site.com/",
    "Accept": "application/json",
    "Accept-Language": "ko-KR,ko;q=0.9",
    "Sec-Fetch-Dest": "empty",
    "Sec-Fetch-Mode": "cors",
    "Sec-Fetch-Site": "same-origin",
}
```

The `Referer` header is particularly important for internal API calls — the server checks that the request appears to come from its own website.

`Sec-Fetch-*` headers are newer and increasingly checked. They tell the server what initiated the request (a script, a link click, etc.).

## 6. Structure your data early

Don't dump raw HTML into files and "parse later." Define your data structure upfront:

```python
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class Listing:
    id: str
    title: str
    price: str
    area_sqm: float
    floor: str
    region: str
    trade_type: str  # sale, lease, rent
    confirmed_date: str
    tags: str = ""
    description: str = ""
    lat: Optional[float] = None
    lng: Optional[float] = None

# Parsing immediately produces structured data
listing = Listing(
    id="12345",
    title="Apartment 101",
    price="500,000",
    area_sqm=84.5,
    floor="12/25",
    region="Seoul",
    trade_type="sale",
    confirmed_date="2026-02-21",
)

# Easy to serialize
import json
json.dumps(asdict(listing), ensure_ascii=False)
```

This catches data quality issues early. If a field is missing, you know immediately — not three weeks later when you try to analyze the data.

## 7. Know when to stop

The hardest part of scraping isn't technical. It's knowing when the approach doesn't work.

Signs you should stop scraping and find an alternative:

- The site actively fights you (CAPTCHAs, IP bans, JavaScript challenges)
- You need residential proxies just to access the site ($13+/GB)
- The data changes faster than you can scrape it
- An official API exists that provides the same data
- The site's ToS explicitly prohibits scraping (legal risk)

**The best scraper is one you don't have to build.** Public APIs, open data portals, and data marketplace subscriptions are often cheaper and more reliable than maintaining a scraper that breaks every time the target site updates its HTML.

---

Web scraping is a useful skill, but it's also a last resort. Before reaching for BeautifulSoup, check if there's an API, a data portal, or an RSS feed. Your future self will thank you.
