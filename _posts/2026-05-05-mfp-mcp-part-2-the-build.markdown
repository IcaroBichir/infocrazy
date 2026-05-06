---
layout: post
title: "Building a MyFitnessPal MCP: Cookies, Scrapers, and Six Tools"
modified:
categories: tech
excerpt: "No public API, a Python 3.14 build failure, and a web scraper doing the work a real API should be doing. Here's how it came together."
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

In [part one](/tech/myfitnesspal-killed-their-api-heres-how-i-got-my-data-back/) I explained why I built this and where it fits in the larger health dashboard I'm working toward. Now let's talk about the implementation — including the part where my Python version refused to cooperate, and the part where the auth model changed under me mid-build.

---

### The core problem: no official API

Strava has a clean OAuth 2.0 API. MyFitnessPal shut theirs down in 2020 with no replacement.

That means there are two paths forward: screen-scrape the web interface yourself, or use a library someone else already built that does the scraping for you. The `myfitnesspal` Python library (by [coddingtonbear](https://github.com/coddingtonbear)) takes the second approach. It logs into the MFP website using your credentials, maintains a session, and parses the diary HTML into structured Python objects.

It's not pretty in principle. In practice, it works, and it covers everything I care about — diary entries, macros, exercise logs, body measurements. That was enough to move forward.

---

### Stack

Five dependencies, each doing one thing:

- **`mcp[cli]`** — Anthropic's Python SDK for building MCP servers
- **`myfitnesspal`** — web scraper / unofficial API client for MFP
- **`lxml>=5.0`** — HTML parser; `myfitnesspal` depends on it
- **`browser-cookie3`** — reads Chrome's encrypted cookie store
- **`click`** — CLI for the `auth` and `serve` commands
- **`requests`** — used during auth to resolve the MFP username

That `lxml` constraint took me longer than it should have.

---

### The Python 3.14 problem

My Homebrew Python is 3.14. The `myfitnesspal` library depends on `lxml`, which has C extensions. `lxml` 4.x doesn't build against Python 3.14 — it uses a private CPython API (`_PyLong_AsByteArray`) that changed signatures in 3.14.

The error looks like this:

```
error: too few arguments to function call, expected 6, have 5
src/lxml/etree.c:269768
```

The fix is not to downgrade Python. The fix is to specify `lxml>=5.0`, which ships with pre-built wheels for Python 3.14:

```
pip install "lxml>=5.0" myfitnesspal mcp[cli]
```

That resolved it immediately. I pinned this in `pyproject.toml` so it doesn't happen to anyone else who installs from the source:

```toml
[project]
dependencies = [
    "mcp[cli]",
    "myfitnesspal",
    "lxml>=5.0",
]
```

---

### Auth: what I planned, what broke, and what I actually shipped

The original plan was simple: pass username and password to the library, get a session, done. The `myfitnesspal` library (v1.14.0) supports exactly this:

```python
client = myfitnesspal.Client(username, password=password, unit_aware=True)
```

That plan fell apart quickly. MFP changed the HTML on their login page, and the library's form parser couldn't find the `authenticity_token` field it needed to submit credentials. Every call failed with an `IndexError` deep in the scraper.

So I switched to the cookie-based approach: instead of logging in with credentials, borrow the session cookies from a browser that's already logged in. [`browser_cookie3`](https://github.com/borisbabic/browser_cookie3) does exactly that — it reads Chrome's encrypted cookie database and returns a `CookieJar` of the cookies for any given domain.

On macOS, Chrome encrypts its cookie database using a key stored in the system Keychain under "Chrome Safe Storage." When `browser_cookie3.chrome()` is called for the first time, macOS shows a dialog asking for permission to access that key. That's the one real friction point: you have to click Allow.

Here's where it got tricky. The library's `Client` constructor only accepts `username` and `password` — there's no `cookiejar=` parameter. The solution is to create the client with `login=False` (skipping credential auth entirely) and inject the cookies directly into the underlying `requests.Session` that the library uses for all its HTTP calls:

```python
cj, username = load_auth()   # cookies from Chrome + username from MFP redirect
client = myfitnesspal.Client(username=username, login=False, unit_aware=True)
client.session.cookies.update(cj)
```

That second line is the key — every subsequent `session.get()` call the library makes will carry those cookies, so MFP sees an authenticated request without the library ever having gone through the login form.

**Getting the username.** The diary URL is `/food/diary/USERNAME` — the library needs the username to build that path. With `login=False`, it never fetches the user's profile. The fix: after loading the cookies from Chrome, make one request to `/food/diary` and follow the redirect. MFP redirects logged-in users to their own diary, so the final URL contains the username:

```python
def _fetch_username(cj: CookieJar) -> str:
    s = requests.Session()
    s.cookies.update(cj)
    r = s.get("https://www.myfitnesspal.com/food/diary", allow_redirects=True)
    if "/food/diary/" in r.url:
        return r.url.split("/food/diary/")[1].split("?")[0].rstrip("/")
    raise RuntimeError("Could not resolve MFP username — are you logged into Chrome?")
```

**Caching both.** The Keychain dialog fires every time `browser_cookie3` reads from Chrome. To avoid this on every Claude Code session start, the auth module caches both the cookies and the resolved username together in `~/.config/mfp-mcp/cookies.json` (mode 0600, JSON not pickle — no arbitrary code execution risk). On subsequent starts, it loads from the file if it's less than 12 hours old, with a corruption fallback that deletes the cache and re-reads from Chrome:

```python
def load_auth() -> tuple[CookieJar, str]:
    if COOKIE_CACHE.exists() and time.time() - COOKIE_CACHE.stat().st_mtime < _COOKIE_TTL:
        try:
            data = json.loads(COOKIE_CACHE.read_text())
            return _list_to_cookiejar(data["cookies"]), data["username"]
        except (json.JSONDecodeError, KeyError, OSError):
            COOKIE_CACHE.unlink(missing_ok=True)
    cj = browser_cookie3.chrome(domain_name="myfitnesspal.com")
    username = _fetch_username(cj)
    # os.open with 0o600 so the file is never world-readable, even briefly
    fd = os.open(COOKIE_CACHE, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    with os.fdopen(fd, "w") as f:
        json.dump({"username": username, "cookies": _cookies_to_list(cj)}, f)
    return cj, username
```

The practical result: run `mfp-mcp auth` after install, click Allow once, and you're done. It prints `Auth OK — logged in as: your_username` on success. The server works silently across every Claude Code session after that until the cache expires.

`unit_aware=True` tells the library to attach unit metadata to measurements (grams, calories, pounds) rather than returning bare floats. The catch: those typed objects — `Energy`, `Mass` — aren't JSON-serializable. A `_to_number()` helper extracts `.value` from each one before returning data from a tool.

---

### The six tools

Same design question as the Strava build: what will I actually ask Claude about?

| Tool | What it returns |
|---|---|
| `get_food_diary` | Full diary for one day: every meal, every food item, macros per entry and per meal |
| `get_food_diary_range` | Daily totals only (no per-entry detail) for up to 30 days |
| `get_exercise_diary` | Exercise and cardio log for one day |
| `get_measurements` | Measurement history (weight, body fat, etc.) going back up to 365 days |
| `get_nutrition_summary` | Aggregated and averaged macros for a date range — totals, daily averages, goals |
| `get_goals` | Current daily nutrition targets from MFP |

A few design decisions worth noting:

**Why `get_food_diary` and `get_food_diary_range` are separate.** A full diary for one day — every food item with nutrition info — is a lot of tokens. For trend questions ("how's my protein been this week?") you don't need that detail, you need daily totals. The range tool returns just those totals, keeping the context window manageable for multi-day queries.

**30-day cap on range queries.** The `myfitnesspal` library makes one HTTP request per day, which means 30 days is already 30 requests. I capped it there to avoid hitting MFP's rate limits or triggering any bot detection. If you need longer windows, call the tool twice.

**Nutrition summary returns both totals and averages.** When Claude is answering a weekly question, it's more useful to say "you averaged 142g of protein per day" than "you consumed 996g over 7 days." The summary tool pre-computes both so Claude doesn't have to.

---

### The FastMCP pattern

Tool registration is the same pattern as the Strava server — `@mcp.tool()` decorator, docstring describes the tool to Claude:

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
    - meals: list of meals, each with name, entries (food items + nutrition), and meal totals
    - daily_totals: summed nutrition across all meals
    - goals: daily nutrition targets
    """
    d = _parse_date(date)
    day = MFPClient().get_date(d.year, d.month, d.day)
    ...
```

Write the docstring for Claude, not for a human developer. Be explicit about argument formats, what the return structure looks like, and what the limits are. That docstring is what Claude reads when it decides whether and how to call the tool.

---

### Connecting to Claude Code

Make sure you're logged into myfitnesspal.com in Chrome, then:

```bash
git clone https://github.com/IcaroBichir/mcp_myfitnesspal.git ~/mcp_myfitnesspal
cd ~/mcp_myfitnesspal && python3 -m venv .venv
.venv/bin/pip install -e .

# Warm the cookie cache — macOS will show a Keychain dialog. Click Allow.
.venv/bin/mfp-mcp auth

claude mcp add -s user myfitnesspal \
  ~/mcp_myfitnesspal/.venv/bin/mfp-mcp -- serve
```

The `auth` command reads cookies from Chrome, resolves your MFP username via the `/food/diary` redirect, and saves both to `~/.config/mfp-mcp/cookies.json`. It prints `Auth OK — logged in as: your_username` on success. Without it, the Keychain dialog appears on the first tool call inside Claude — run it here once, click Allow, and it won't appear again until the 12-hour cache expires.

The `-s user` flag writes to `~/.claude.json` — the file Claude Code actually manages — and makes the server available globally across every session. Don't manually edit `.mcp.json` files; changes there won't be picked up properly across sessions.

The README at [github.com/IcaroBichir/mcp_myfitnesspal](https://github.com/IcaroBichir/mcp_myfitnesspal) also has a single prompt you can paste into Claude Code to automate the whole setup.

---

### What's next

Both MCP servers are running. Claude can now see my workouts (Strava) and my food log (MFP). The next step is the cross-source work — asking questions that require both datasets simultaneously.

That's the health dashboard post I'll write once I've actually used it for a few weeks. No point writing about query results I haven't seen yet.
