---
layout: post
title: "Building a MyFitnessPal MCP: Cookies, a Bearer Token, and Six Tools"
modified: 2026-05-06
categories: blog
excerpt: "No public API, Cloudflare blocking the HTML scraper, and an undocumented mobile JSON API hiding in plain sight. Here's what the build actually looks like."
tags: [mcp, myfitnesspal, python, claude, ai, health, nutrition]
comments: true
date: 2026-05-05T10:00:00-04:00
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

**Note:** This post was substantially updated in May 2026. The original version described an HTML scraping approach using `python-myfitnesspal`. After publishing, Cloudflare Bot Fight Mode on MFP's pages made that approach fail entirely. The implementation described here — using MFP's undocumented mobile JSON API — is what actually ships in the current version. [Part 3](/blog/mfp-mcp-part-3-cloudflare-killed-the-scraper/) tells the full story of that transition.

---

In [part one](/blog/mfp-mcp-part-1-why-i-built-it/) I explained why I built this and where it fits in the larger health dashboard I'm working toward. Now let's talk about the implementation — including the part where my original approach stopped working entirely, and what replaced it.

---

### The core problem: no official API

Strava has a clean OAuth 2.0 API. MyFitnessPal shut theirs down in 2020 with no replacement.

That means there are two paths forward: find an undocumented API and use it directly, or scrape the web interface. The `myfitnesspal` Python library does the latter — it logs into the MFP website, maintains a session, and parses the diary HTML into Python objects. I started there. It worked until Cloudflare made it stop working permanently. More on that in Part 3.

The current implementation doesn't scrape anything. It uses `api.myfitnesspal.com` — the same JSON API the MFP mobile app calls. No Cloudflare protection. Clean structured data.

---

### Stack

Four dependencies:

- **`mcp[cli]`** — Anthropic's Python SDK for building MCP servers
- **`browser-cookie3`** — reads Chrome's encrypted cookie store for auth
- **`requests`** — HTTP client for both the token endpoint and the API
- **`click`** — CLI for the `auth`, `serve`, and `cache` commands

No HTML parser, no scraping library, no `lxml`. The original build needed all of those. The new one doesn't.

---

### Auth: how it actually works

MFP's HTML pages are protected by Cloudflare Bot Fight Mode — automated HTTP clients get 403s regardless of what headers or cookies you send. But two MFP endpoints are Cloudflare-free:

**Step 1 — get a bearer token.** The endpoint `www.myfitnesspal.com/user/auth_token?refresh=true` returns a JSON object with an OAuth bearer token, given valid session cookies. These cookies are the ones Chrome already holds from when you log into MFP normally. `browser_cookie3` reads them from Chrome's encrypted on-disk database:

```python
import browser_cookie3

cj = browser_cookie3.chrome(domain_name="myfitnesspal.com")
# cj is now a CookieJar containing everything Chrome has for myfitnesspal.com
```

On macOS, Chrome encrypts its cookies using a key in the system Keychain under "Chrome Safe Storage." The first time `browser_cookie3` reads from it, macOS shows a dialog asking permission. You click Allow once. After that it's silent.

The token request itself:

```python
s = requests.Session()
s.cookies.update(cj)
s.headers.update({"Accept": "application/json", "X-Requested-With": "XMLHttpRequest"})

r = s.get("https://www.myfitnesspal.com/user/auth_token?refresh=true")
data = r.json()
# data["access_token"] — the bearer token, valid for 10 days
# data["expires_in"]  — seconds until expiry
```

**Step 2 — extract the user ID.** The bearer token is base64-encoded and contains the numeric user ID in its payload:

```python
import base64

decoded = base64.b64decode(token + "==", validate=False).decode("utf-8", errors="replace")
# Format: a:mfp-main-js:{user_id}::mfp-js:{timestamp}:{expiry}{signature}
user_id = decoded.split(":")[2]
```

**Step 3 — query the mobile API.** `api.myfitnesspal.com` is the endpoint the MFP mobile app uses. It requires the bearer token and user ID as headers:

```python
api = requests.Session()
api.headers.update({
    "Authorization": f"Bearer {token}",
    "mfp-client-id": "mfp-main-js",
    "mfp-user-id": user_id,
    "Accept": "application/json",
})

r = api.get(f"https://api.myfitnesspal.com/v2/diary?username={username}&date=2026-05-06")
items = r.json()["items"]
# Clean JSON: diary_meal, exercise_entry, steps_aggregate items
```

**Caching both layers.** Chrome's Keychain dialog fires every time `browser_cookie3` reads from it. The cookie cache (`~/.config/mfp-mcp/cookies.json`, mode 0600, 12-hour TTL) avoids this on every session start. The bearer token (10-day expiry from MFP) is cached in SQLite so we don't need to request a fresh one on every client instantiation. The practical result: run `mfp-mcp auth` once after installing, and data flows silently for days.

The username comes from `MFP_USERNAME` in a `.env` file — this is required. The `mfp-mcp auth` command validates all of this and prints `Auth OK — logged in as: your_username` on success.

---

### The Python 3.14 detail (now moot)

The original post had a long section about `lxml` not building on Python 3.14. That was a real headache at the time. Since the current implementation uses no HTML parser at all, the problem doesn't exist anymore. If you're installing from the current source, you won't hit it.

---

### The SQLite cache

The Strava MCP uses SQLite to cache API responses. The MFP MCP now does the same. Data cached in `~/.config/mfp-mcp/cache.db`:

| What | TTL |
|---|---|
| Bearer token | 8 days (token lives 10; refresh before expiry) |
| Diary data per date | 30 minutes |
| Measurements | 1 hour |

The `mfp-mcp cache stats` command shows how many entries are live versus expired. `mfp-mcp cache clear` wipes everything and forces fresh fetches on the next call.

The TTL on diary data is short because it changes during the day — you're logging meals as you eat them. Measurements change less frequently, so an hour is fine there.

---

### The six tools

| Tool | What it returns |
|---|---|
| `get_food_diary` | Full diary for one day: meals with their nutrition totals |
| `get_food_diary_range` | Daily totals only (no per-entry detail) for up to 30 days |
| `get_exercise_diary` | Exercise log for one day, grouped into a Cardio set |
| `get_measurements` | Measurement history (weight, body fat, etc.) up to 365 days back |
| `get_nutrition_summary` | Aggregated and averaged macros for a date range |
| `get_goals` | Current daily nutrition targets (returns empty — see below) |

**A note on `get_food_diary`.** The original version of this tool returned per-entry detail — every individual food item with its own nutrition info. The mobile API's `/v2/diary` endpoint returns meal-level aggregates, not individual entries. You get the meal total for Lunch but not the breakdown of what made it up. For Claude's most common use cases — "how was my protein today?", "what's my calorie average this week?" — this is sufficient. For drilling into a specific meal, you'd use the MFP app directly.

**A note on `get_goals`.** The mobile API stores macro goals as percentages rather than gram targets, and I haven't found a clean endpoint that returns the computed calorie goal. This tool returns an empty dict for now. The question it was designed to answer — "am I hitting my goals?" — is still answerable by looking at totals in context, it just requires Claude to reason about it rather than having explicit targets to compare against.

**Why range and summary are separate.** Multi-day queries make one API call per day. A 30-day nutrition summary is 30 requests. The range tool caps at 30 days. If you need longer windows, call it twice with adjacent date ranges.

---

### The FastMCP pattern

Tool registration is the same as the Strava server — `@mcp.tool()` decorator, docstring as the tool description:

```python
mcp = FastMCP(
    "MyFitnessPal",
    instructions=(
        "Use these tools to retrieve the authenticated user's MyFitnessPal data. "
        "Nutrition values are in grams, calories in kcal. "
        "Always present macros clearly with labels: calories, protein, carbs, fat."
    ),
)

@mcp.tool()
def get_food_diary(date: str) -> dict:
    """
    Return the full food diary for a single day.

    Args:
        date: Date in YYYY-MM-DD format (e.g. "2026-05-01")

    Returns a dict with keys:
    - date: the queried date (ISO 8601)
    - meals: list of meals with name and nutrition totals
    - daily_totals: summed nutrition across all meals
    - goals: daily nutrition targets (currently empty)
    """
    d = _parse_date(date)
    day = MFPClient().get_date(d.year, d.month, d.day)
    ...
```

Write the docstring for Claude, not for a human developer. Be explicit about argument formats, return structure, and limits. That docstring is what Claude reads when it decides whether and how to call the tool.

---

### Connecting to Claude Code

Make sure you're logged into myfitnesspal.com in Chrome, then:

```bash
git clone https://github.com/IcaroBichir/mcp_myfitnesspal.git ~/mcp_myfitnesspal
cd ~/mcp_myfitnesspal && python3 -m venv .venv
.venv/bin/pip install -e .

# Add your MFP username (NOT your email — your profile username)
echo "MFP_USERNAME=your_username" > .env

# Warm the cookie + token cache
.venv/bin/mfp-mcp auth

claude mcp add -s user myfitnesspal \
  ~/mcp_myfitnesspal/.venv/bin/mfp-mcp -- serve
```

`auth` reads your Chrome cookies, fetches a bearer token, caches both, and prints `Auth OK — logged in as: your_username`. After that, `mfp-mcp serve` starts the MCP server and data flows silently. Run `mfp-mcp cache stats` to confirm the cache is warm.

---

### What's next

Both MCP servers are running. Claude can see workouts and food. The next step is the cross-source work — questions that require both datasets simultaneously — which the health dashboard makes concrete. That story is in [Dashboard Part 2](/blog/health-dashboard-part-2-wiring-in-nutrition/).

If you want the full technical story of why the original scraper stopped working and every bypass that failed before the API pivot, that's [Part 3](/blog/mfp-mcp-part-3-cloudflare-killed-the-scraper/).
