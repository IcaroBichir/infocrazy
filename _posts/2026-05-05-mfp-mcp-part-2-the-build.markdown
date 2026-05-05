---
layout: post
title: "Building My Health Dashboard, Part 2: How I Built the MFP MCP"
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

In [part one](/tech/building-my-health-dashboard-part-1-why-myfitnesspal-needed-an-mcp/) I explained why I built this and where it fits in the larger health dashboard I'm working toward. Now let's talk about the implementation — including the part where my Python version refused to cooperate.

---

### The core problem: no official API

Strava has a clean OAuth 2.0 API. MyFitnessPal shut theirs down in 2020 with no replacement.

That means there are two paths forward: screen-scrape the web interface yourself, or use a library someone else already built that does the scraping for you. The `myfitnesspal` Python library (by [coddingtonbear](https://github.com/coddingtonbear)) takes the second approach. It logs into the MFP website using your credentials, maintains a session, and parses the diary HTML into structured Python objects.

It's not pretty in principle. In practice, it works, and it covers everything I care about — diary entries, macros, exercise logs, body measurements. That was enough to move forward.

---

### Stack

Four dependencies, each doing one thing:

- **`mcp[cli]`** — Anthropic's Python SDK for building MCP servers
- **`myfitnesspal`** — web scraper / unofficial API client for MFP
- **`python-dotenv`** — load credentials from a `.env` file
- **`lxml>=5.0`** — HTML parser; `myfitnesspal` depends on it

That last constraint took me longer than it should have.

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
pip install "lxml>=5.0" myfitnesspal mcp[cli] python-dotenv
```

That resolved it immediately. I pinned this in `pyproject.toml` so it doesn't happen to anyone else who installs from the source:

```toml
[project]
dependencies = [
    "mcp[cli]",
    "myfitnesspal",
    "python-dotenv",
    "lxml>=5.0",
]
```

---

### Credentials

Unlike Strava, there's no OAuth dance here. You supply your MFP username and password, the library logs in, the session is held in memory for the duration of the server process.

I keep credentials in a `.env` file in the project directory:

```
MFP_USERNAME=your_myfitnesspal_username
MFP_PASSWORD=your_myfitnesspal_password
```

The server loads them at startup via `python-dotenv`. The client itself is lazy-initialized — it doesn't log in until the first tool call, which means startup is fast and you'll see the auth error at call time (not silently at launch).

```python
_client = None

def _get_client():
    global _client
    if _client is None:
        import myfitnesspal
        username = os.environ.get("MFP_USERNAME")
        password = os.environ.get("MFP_PASSWORD")
        if not username or not password:
            raise RuntimeError("MFP_USERNAME and MFP_PASSWORD must be set")
        _client = myfitnesspal.Client(username, password=password, unit_aware=True)
    return _client
```

`unit_aware=True` tells the library to attach unit metadata to measurements — grams, calories, pounds — instead of returning bare floats.

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
mcp = FastMCP("MyFitnessPal")

@mcp.tool()
def get_food_diary(date: str) -> dict:
    """
    Return the full food diary for a single day.

    Args:
        date: Date in YYYY-MM-DD format (e.g. "2026-05-01")

    Returns a dict with keys:
    - date: the queried date
    - meals: list of meals, each with name, entries (food items + nutrition), and meal totals
    - daily_totals: summed nutrition across all meals
    - goals: daily nutrition targets
    """
    d = _parse_date(date)
    client = _get_client()
    day = client.get_date(d.year, d.month, d.day)
    ...
```

Write the docstring for Claude, not for a human developer. Be explicit about argument formats, what the return structure looks like, and what the limits are. That docstring is what Claude reads when it decides whether and how to call the tool.

---

### Connecting to Claude Code

Add to `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "myfitnesspal": {
      "command": "/path/to/mcp_myfitnesspal/.venv/bin/python3",
      "args": ["/path/to/mcp_myfitnesspal/server.py"],
      "env": {
        "MFP_USERNAME": "your_username",
        "MFP_PASSWORD": "your_password"
      }
    }
  }
}
```

Or use a `.env` file in the project directory and leave the `env` block out of the config — `python-dotenv` will pick it up automatically when the server starts.

---

### What's next

Both MCP servers are running. Claude can now see my workouts (Strava) and my food log (MFP). The next step is the cross-source work — asking questions that require both datasets simultaneously.

That's the health dashboard post I'll write once I've actually used it for a few weeks. No point writing about query results I haven't seen yet.
