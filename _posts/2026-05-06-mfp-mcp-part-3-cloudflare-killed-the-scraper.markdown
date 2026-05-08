---
layout: post
title: "MFP MCP Part 3: Cloudflare Killed the Scraper (We Found a Better API)"
modified:
categories: tech
excerpt: "The dashboard revealed that my published MFP implementation was broken. Every bypass failed. Then we found that api.myfitnesspal.com exists and has no Cloudflare protection at all."
tags: [mcp, myfitnesspal, python, claude, ai, cloudflare, health, nutrition]
comments: true
authored_by: ai
date: 2026-05-06T12:00:00-04:00
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

The original MFP MCP used `python-myfitnesspal` — an unofficial library that scrapes MFP's HTML diary pages. I published two posts about building it. The code worked when I wrote those posts.

Then I wired the MFP server into the health dashboard, pulled up nutrition data for the first time, and got nothing. Empty cards. No error. Just silence.

That silence turned into a two-session debugging arc that ended with the scraper completely replaced by a cleaner API that was there the whole time.

---

### What broke and when

The symptom was deceptive. `mfp-mcp auth` printed `Auth OK — logged in as: icbichir1`. Claude could invoke the MFP tools and they returned results. The results just had empty nutrition data — zeros across every macro, every day.

The library was working. The authentication was working. The requests were going out. MFP was responding — just with 403 pages that the library silently parsed into empty data rather than raising an error.

Once we confirmed that raw `requests` calls to `www.myfitnesspal.com/food/diary/icbichir1` were returning 403, the diagnosis was clear: Cloudflare Bot Fight Mode.

---

### Why this worked before and stopped working

MFP's Cloudflare configuration uses a `cf_clearance` cookie to distinguish browsers that passed a bot challenge from automated clients. The `python-myfitnesspal` library was injecting this cookie from Chrome's session alongside the auth cookies. When `cf_clearance` was fresh, requests succeeded. When it expired — a matter of hours — everything returned 403.

When I first built the server, the cache was warm and the cookies were fresh. When I built the dashboard weeks later and pulled up MFP data for the first time in a while, the clearance had expired. The original implementation had no way to renew it.

---

### Every bypass that failed

The obvious path was: get a fresh `cf_clearance` somehow, inject it, keep going.

**Fresh `cf_clearance` from Chrome + requests**: Still 403. Cloudflare's clearance cookies are bound to the TLS fingerprint from the browser that solved the challenge. Python's `requests` library has a different TLS fingerprint than Chrome, so Cloudflare rejects the combination even with a valid cookie.

**`curl_cffi` Chrome TLS impersonation**: `curl_cffi` can impersonate Chrome's TLS handshake at the C level, matching the JA3/JA4 fingerprint. Still 403. Cloudflare was checking beyond TLS — JavaScript execution context, browser behavior signals, something that `curl_cffi` doesn't replicate.

**Playwright with Chromium**: This is a real browser, just headless. Still 403. Playwright's Chromium is not Chrome — different binary, different TLS session history, detectable.

**Playwright with Chrome (`channel='chrome'`)**: This uses the actual Chrome binary installed on the system, not Playwright's bundled Chromium. Launched headed (`headless=False`) with `--disable-blink-features=AutomationControlled` and a patched `navigator.webdriver`. Still 403 immediately, then the page transitioned to Cloudflare's "Just a moment..." challenge page. That page ran JavaScript for 30 seconds and never resolved — Cloudflare's behavioral fingerprinting was detecting the automation context even with the real Chrome binary.

**AppleScript + Chrome**: The cleanest bypass would be using the user's actual running Chrome session — same process, same session state, no automation detection. macOS AppleScript can execute JavaScript in Chrome tabs, which would let us load the page and extract the HTML. This requires enabling **View > Developer > Allow JavaScript from Apple Events** in Chrome. We considered it and ruled it out — it grants any application on the machine the ability to run JavaScript in any Chrome tab, which is too broad a permission to require as a setup step.

At this point we had tried everything in the browser automation stack. None of it worked. Cloudflare's bot detection is multi-layered and operates at a level that automated browsers can't reliably fake.

<div class="human-aside">
<p>I tried five things. every single one got 403'd. requests with chrome cookies: no. curl_cffi with chrome tls impersonation: no. playwright with chromium: no. playwright with the actual chrome binary, real binary, automation flag patched, headless=false: 403 immediately then a thirty-second challenge spinner that never resolved. at some point you're not debugging a bug, you're debugging a wall. what ended up working was noticing, while looking at failure logs, that a different subdomain was returning 200s. the api was there the whole time. I had published two posts based on an implementation with a time bomb in it. the bomb went off the first time I actually used it in production. that's how this works sometimes.</p>
</div>

---

### Finding the actual API

While debugging the request failures, we noticed something: `api.myfitnesspal.com` was returning 200s.

MFP's main domain (`www.myfitnesspal.com`) is heavily protected. But `api.myfitnesspal.com` — the endpoint the mobile app uses — has no Cloudflare Bot Fight Mode at all. Different subdomain, different Cloudflare configuration.

The next question: does it need its own auth, or can we use the existing session?

There's an endpoint on the main domain that's also Cloudflare-free: `www.myfitnesspal.com/user/auth_token?refresh=true`. Send it a GET request with your existing session cookies (the ones Chrome already holds) and it returns a JSON object with an OAuth bearer token:

```json
{
  "token_type": "Bearer",
  "access_token": "...",
  "expires_in": 864000,
  "refresh_token": "..."
}
```

864000 seconds is 10 days.

The bearer token's payload is base64-encoded and contains the numeric user ID. Decode it:

```python
decoded = base64.b64decode(token + "==", validate=False).decode("utf-8", errors="replace")
# a:mfp-main-js:{user_id}::mfp-js:{timestamp}:{expiry}{signature}
user_id = decoded.split(":")[2]  # "62972721106813"
```

Then use both against `api.myfitnesspal.com`:

```python
api = requests.Session()
api.headers.update({
    "Authorization": f"Bearer {token}",
    "mfp-client-id": "mfp-main-js",
    "mfp-user-id": user_id,
    "Accept": "application/json",
})

r = api.get(f"https://api.myfitnesspal.com/v2/diary?username={username}&date=2026-05-06")
```

Response:

```json
{
  "items": [
    {
      "type": "diary_meal",
      "diary_meal": "Lunch",
      "nutritional_contents": {
        "protein": 52.92,
        "fat": 16.17,
        "carbohydrates": 64.61,
        "energy": {"unit": "calories", "value": 593.14}
      }
    },
    {
      "type": "exercise_entry",
      "exercise": {"description": "Running"},
      "duration": 3600,
      "energy": {"unit": "calories", "value": 580.0}
    }
  ]
}
```

Actual data. No 403. Clean JSON. No HTML to parse.

The measurements endpoint works the same way:

```python
r = api.get(
    f"https://api.myfitnesspal.com/v2/measurements"
    f"?username={username}&type=Weight&from_date=2026-04-01&to_date=2026-05-06"
)
```

---

### What changed in the codebase

The `python-myfitnesspal` library is gone. So is `lxml`. The scraper is gone. What replaced it:

**`api_client.py`** — `MFPApiClient` handles the two-step auth (cookies → bearer token) and wraps the `api.myfitnesspal.com` endpoints. The bearer token is cached in SQLite for 8 days so we don't hit the token endpoint on every instantiation. Diary data caches for 30 minutes; measurements for an hour.

**`cache.py`** — SQLite cache with the same `CacheStore` pattern as strava-mcp. `mfp-mcp cache stats` and `mfp-mcp cache clear` work the same way as `strava-mcp cache stats/clear`.

**`client.py`** — `MFPClient` delegates to `MFPApiClient` and returns dataclass objects (`DayData`, `Meal`, `ExerciseSet`) compatible with the interface `server.py` already expected. The six tools in `server.py` didn't change.

The `pyproject.toml` lost two dependencies (`myfitnesspal`, `lxml`) and the setup instructions got a new required step (`MFP_USERNAME` in `.env`).

---

### What's different about the data

**Meal-level vs. entry-level.** The HTML scraper returned individual food items — every entry in a meal with its own nutrition info. The `/v2/diary` endpoint returns meal-level aggregates. You get the total for Lunch but not the list of what composed it. For the questions I actually ask Claude — "how was my protein today?", "what's my weekly calorie average?" — meal totals are sufficient.

**Goals currently missing.** The original `get_goals` tool read the goal totals from the HTML diary page. The mobile API stores macro goals as percentages, and the endpoint that computes them into calorie/gram targets isn't obvious. `get_goals` currently returns empty. The calorie consumption data is accurate; the targets for comparison aren't there yet.

---

### The real lesson

This whole detour happened because the scraper worked when I published the posts, and then quietly stopped working when conditions changed — an expired Cloudflare cookie, and no mechanism to renew it. The failure mode was silent: the library returned empty data rather than raising an exception.

The replacement is genuinely better. A proper JSON API is more stable than HTML scraping. The response format is explicit. The auth is standard OAuth rather than cookie injection. The bearer token lives for 10 days versus Cloudflare clearance tokens that expire in hours.

I wrote two blog posts based on an implementation that had a time bomb in it. The dashboard was what finally detonated it by actually exercising the full stack against real data.

[Dashboard Part 2](/tech/health-dashboard-part-2-wiring-in-nutrition/) picks up from here — what the nutrition integration looks like on screen, and what it enables once both data sources are actually flowing.
