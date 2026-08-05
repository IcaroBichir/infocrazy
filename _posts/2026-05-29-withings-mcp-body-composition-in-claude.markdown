---
layout: post
title: "Withings MCP: Body Composition Data in Claude"
modified:
categories: blog
excerpt: "A fourth MCP server, this time for the Withings scale. OAuth2 is clean, the API is stable, and the data — weight, fat%, muscle mass — slots naturally alongside Strava and MFP in any conversation about training."
tags: [health, mcp, withings, python, bodycomposition, claude, anthropic]
comments: true
date: 2026-05-29T10:00:00-04:00
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

The Strava MCP exposes training data. The MFP MCP exposes nutrition. A third question has always been adjacent to both: is the combination actually working? Are calories, macros, and training load moving the body composition numbers in the right direction?

That's a Withings question. The scale measures weight, body fat percentage, muscle mass, fat-free mass, bone mass, and hydration. It's the outcome variable. Adding it as a fourth MCP server means Claude can now look at all three dimensions in the same conversation.

---

### Why Withings was straightforward

The MFP MCP required navigating Cloudflare, browser cookie extraction, a two-step token exchange, and careful routing through the mobile API domain to avoid bot detection. The Strava MCP involved standard OAuth2 with some edge cases around token refresh.

Withings is just OAuth2. Standard authorization code flow, access token, refresh token. The API documentation is public and accurate. The endpoints return clean JSON with a predictable envelope:

```json
{
  "status": 0,
  "body": { ... }
}
```

Status 0 is success. Any other status is an error with a message. There's no Cloudflare, no cookie extraction, no scraping.

The only friction was the local OAuth callback — the authorization flow needs a redirect URI that Withings will call after the user approves access. For a local-first tool, I use a temporary HTTP server on `localhost:8080` to catch the callback during `withings-mcp auth`. The server starts, opens the Withings authorization page in the browser, waits for the callback, captures the code, exchanges it for tokens, and shuts down. After that, the tokens live at `~/.config/withings-mcp/tokens.json` and the CLI never needs OAuth again — access tokens auto-refresh on expiry.

---

### The measurement types

The interesting technical detail is how Withings represents body composition measurements. Every scale reading is a list of `{type, value, unit}` tuples:

```json
{
  "measures": [
    {"type": 1,  "value": 7460,  "unit": -2},
    {"type": 6,  "value": 1660,  "unit": -3},
    {"type": 76, "value": 5938,  "unit": -2}
  ]
}
```

Type 1 is weight, type 6 is fat ratio, type 76 is muscle mass. The actual value is `value × 10^unit` — so 7460 × 10⁻² = 74.60 kg, and 1660 × 10⁻³ = 1.660 (which is fat ratio expressed as 16.60%).

The full type map:

| Type ID | Field |
|---|---|
| 1 | weight_kg |
| 5 | fat_free_mass_kg |
| 6 | fat_ratio_pct |
| 8 | fat_mass_kg |
| 76 | muscle_mass_kg |
| 77 | hydration_kg |
| 88 | bone_mass_kg |

A single scale reading produces multiple measurement groups (one per sensor firing). The client merges all groups for the same date into one record:

```python
by_date: dict[str, dict] = {}
for grp in body.get("measuregrps", []):
    d = datetime.fromtimestamp(grp["date"], tz=timezone.utc).date().isoformat()
    if d not in by_date:
        by_date[d] = {"date": d}
    for m in grp.get("measures", []):
        name = _MEAS_TYPE_NAMES.get(m["type"])
        if name:
            by_date[d][name] = _apply_unit(m["value"], m["unit"])
```

The result is one clean dict per day: `{"date": "2026-05-29", "weight_kg": 74.60, "fat_ratio_pct": 16.60, "muscle_mass_kg": 59.38, ...}`.

---

### The four tools

The server exposes four tools via FastMCP:

**`get_measurements(start, end)`** — body composition readings, newest first. This is the primary tool — weight, fat%, muscle mass, and the rest. Cache TTL is 24 hours since scale data changes slowly.

**`get_activity_summary(start, end)`** — daily activity totals from the Withings activity tracker: steps, distance, calories, HR. Different from Strava — Withings counts passive activity throughout the day, not just workout sessions.

**`get_workouts(start, end)`** — explicit workout sessions recorded through Withings. Includes sport type, duration, HR zones, and calories. The sport type comes back as a numeric category ID that maps to a human-readable string.

**`get_sleep_summary(start, end)`** — nightly sleep breakdown: total sleep, deep/light/REM durations, sleep score, HR and respiratory rate during sleep.

The cache TTL design distinguishes between historical data (immutable, 24h) and today's data (may update through the day, 15 minutes). The measurements endpoint always uses 24h since you're not stepping on the scale mid-afternoon.

---

### What it enables in Claude

With all four servers running, a single Claude session has access to:

- Strava: what workouts happened, at what intensity, burning how many calories
- MFP: what was eaten, with what macro breakdown
- Withings: what the scale says — weight, fat%, muscle mass — and how it's trending

Asking Claude to check how the last 30 days of nutrition and training are reflecting in body composition is now a one-prompt question. The data is already loaded:

> My last 30 days: ran 4x per week, averaged 1,800 kcal/day, protein around 140g. Scale went from 75.2 kg at 17.6% fat to 74.6 kg at 16.6%. Does that look right for the calorie deficit?

That kind of question — anchored in actual numbers, covering all three dimensions — is what having all the data in context enables.

---

### Installation

```bash
cd mcp_withings
python3 -m venv .venv
.venv/bin/pip install -e ".[dev]"
.venv/bin/withings-mcp auth    # needs WITHINGS_CLIENT_ID + CLIENT_SECRET in .env
.venv/bin/withings-mcp serve   # stdio transport for Claude
```

Register it in `~/.claude/.mcp.json`:

```json
{
  "mcpServers": {
    "withings": {
      "command": "/path/to/mcp_withings/.venv/bin/withings-mcp",
      "args": ["serve"]
    }
  }
}
```

After `auth`, the credentials are stored in `tokens.json` — `serve` never needs the `.env` file again.

---

The code is on GitHub at [IcaroBichir/mcp_withings](https://github.com/IcaroBichir/mcp_withings). The next post covers how body composition data was wired into the health dashboard — completing the picture that the [Nutrition × Exercise tab](/blog/health-dashboard-part-5-nutrition-exercise-correlation/) was missing.
