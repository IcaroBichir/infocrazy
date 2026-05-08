---
layout: post
title: "Wiring Claude to My Running Shoes, Part 2: The Build"
modified:
categories: tech
excerpt: "OAuth is always the part nobody wants to explain. Here's the whole thing — plus how I designed the five tools and why I cut the other seven."
tags: [mcp, strava, python, claude, ai, oauth]
comments: true
date: 2026-05-04T09:00:00-04:00
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

In [part one](/tech/i-wired-claude-to-my-running-shoes-here-s-why/) I explained why I built this. Now let's talk about how.

The full source is at [github.com/icarobichir/strava-mcp](https://github.com/icarobichir/strava-mcp). You can install it with `pip install strava-mcp` and be running in about 10 minutes. This post is for anyone who wants to understand what's inside.

---

### Stack and structure

The server is pure Python, no framework beyond the libraries that do their specific jobs:

- **`mcp[cli]`** — Anthropic's Python SDK for building MCP servers
- **`httpx`** — async-friendly HTTP client for Strava API calls
- **`click`** — CLI entry points (`strava-mcp auth` and `strava-mcp serve`)

Four files, each with one job:

```
strava_mcp/
├── __main__.py   # CLI (auth + serve commands)
├── auth.py       # OAuth flow + token storage/refresh
├── client.py     # Thin wrapper around Strava's REST API
└── server.py     # MCP tool definitions
```

That's it. No database, no config files beyond the token JSON, no background processes.

---

### OAuth: the part nobody wants to explain

Strava uses OAuth 2.0 with the authorization code grant. Here's the full flow:

1. Build an authorization URL with your client ID, requested scopes, and a redirect URI
2. Open that URL in the user's browser
3. Strava asks the user to approve access
4. Strava redirects to your redirect URI with an authorization code as a query parameter
5. Exchange that code for access + refresh tokens via a POST request
6. Store the tokens; refresh automatically when they expire

Steps 1 and 5 are straightforward HTTP. Step 3 is Strava's business. The interesting part is step 4 — *how do you catch a browser redirect in a CLI tool?*

The answer is a temporary HTTP server on localhost.

```python
class _CallbackHandler(BaseHTTPRequestHandler):
    auth_code: str | None = None

    def do_GET(self) -> None:
        params = parse_qs(urlparse(self.path).query)
        _CallbackHandler.auth_code = params.get("code", [None])[0]
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.end_headers()
        self.wfile.write(
            b"<h1>Authorization complete!</h1>"
            b"<p>You can close this tab and return to the terminal.</p>"
        )

    def log_message(self, *args) -> None:
        pass  # suppress request logs
```

Spin up `HTTPServer` on port 8765, handle exactly one request, then shut down. The browser redirects to `http://localhost:8765?code=...`, the handler captures the code, done. No web framework needed — Python's standard library has everything.

The redirect URI you register in your Strava app settings is `http://localhost:8765`. This is the domain Strava will redirect to after authorization. It doesn't need to be publicly reachable — it just needs to match what you registered.

**Token storage:** Tokens go to `~/.config/strava-mcp/tokens.json` with `chmod 0o600`. The file stores client ID, client secret, access token, refresh token, and `expires_at` (a Unix timestamp). Strava access tokens expire every six hours, so auto-refresh matters:

```python
def refresh_if_needed() -> dict:
    tokens = load_tokens()
    if tokens["expires_at"] > time.time() + 60:
        return tokens  # still valid

    resp = httpx.post(_TOKEN_URL, data={
        "client_id": tokens["client_id"],
        "client_secret": tokens["client_secret"],
        "refresh_token": tokens["refresh_token"],
        "grant_type": "refresh_token",
    })
    resp.raise_for_status()
    new_data = resp.json()
    _save(tokens["client_id"], tokens["client_secret"], new_data)
    return {**tokens, **new_data}
```

The 60-second buffer means you don't get a mid-conversation token expiry.

---

### Designing the tools: from 12 to 5

I started with a list of everything Strava's API can return. Segments, streams, clubs, routes, kudos, comments, photos, gear, routes, starred segments — the API surface is large.

Then I applied one filter: *"Would I actually ask Claude about this?"*

Not "could this be useful" — specifically, "will I type a question about this in conversation?" That cut the list fast.

**What I kept:**

| Tool | What it does |
|---|---|
| `list_activities` | Summary list with filters (sport, date range, limit) |
| `get_activity` | Full detail: splits, segment efforts, best efforts, laps |
| `get_athlete_stats` | Aggregate totals: recent / YTD / all-time by sport |
| `get_athlete` | Profile: name, location, weight, FTP, measurement preference |
| `get_gear` | Bike or shoe details with total distance logged |

**What I cut:**

- **Segments**: interesting for competitive cyclists, irrelevant to how I actually train
- **Streams** (raw GPS/power/cadence time-series): useful eventually, but Claude doesn't need 3,000 GPS points to answer "how was my heart rate on that long run"
- **Clubs**: I'm not in any
- **Kudos/comments**: social, not training analysis
- **Routes**: I don't use Strava's route planner

The tools I kept cover every question I've actually wanted to ask. The others can be added later.

**One design decision worth noting:** distances come back from Strava in meters, times in seconds, speeds in m/s. I left the conversion to Claude rather than doing it server-side. The system prompt tells Claude to always present values in human-readable units. This keeps the server simple and lets Claude handle the locale — if you prefer miles, just say so.

---

### The MCP server itself

[FastMCP](https://github.com/jlowin/fastmcp) makes tool registration minimal:

```python
mcp = FastMCP(
    "Strava",
    instructions=(
        "Use these tools to retrieve the authenticated user's Strava data. "
        "Distances are in meters, times in seconds, speeds in m/s. "
        "Always convert to human-readable units (km or miles, min/km or min/mile, "
        "HH:MM:SS) when presenting results."
    ),
)

@mcp.tool()
def list_activities(
    limit: int = 30,
    sport_type: Optional[str] = None,
    after: Optional[str] = None,
    before: Optional[str] = None,
) -> list[dict]:
    """
    List the athlete's activities with summary stats.
    ...
    """
    client = StravaClient()
    activities = client.list_activities(
        per_page=min(limit, 200),
        after=_iso_to_ts(after) if after else None,
        before=_iso_to_ts(before) if before else None,
    )
    if sport_type:
        activities = [a for a in activities if a.get("sport_type") == sport_type]
    return [{k: v for k, v in a.items() if k in _SUMMARY_KEYS and v is not None}
            for a in activities[:limit]]
```

The docstring becomes the tool description Claude reads when deciding whether and how to use it. Write it for Claude, not for a human developer — be specific about argument formats and units.

---

### Running it with Claude

**Install:**

```bash
pip install strava-mcp
strava-mcp auth
```

**Add to Claude Code** (`~/.claude/settings.json`):

```json
{
  "mcpServers": {
    "strava": {
      "command": "strava-mcp",
      "args": ["serve"]
    }
  }
}
```

**Add to Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "strava": {
      "command": "strava-mcp",
      "args": ["serve"]
    }
  }
}
```

Restart Claude after adding the config. The server starts on demand via stdio — no port to manage, no background process to babysit.

---

Next up: [what I've actually done with it](/tech/what-i-actually-did-with-my-strava-mcp-server/).
