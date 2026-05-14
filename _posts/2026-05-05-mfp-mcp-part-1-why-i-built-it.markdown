---
layout: post
title: "MyFitnessPal Killed Their API. Here's How I Got My Data Back."
modified:
categories: tech
excerpt: "I already wired Claude to my workouts. Nutrition was the obvious next piece — except MyFitnessPal killed their public API in 2020."
tags: [mcp, myfitnesspal, python, claude, ai, health, nutrition]
comments: true
date: 2026-05-05T09:00:00-04:00
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

**Update (May 2026):** After publishing this series, MFP's Cloudflare Bot Fight Mode started blocking all automated HTTP clients — including the `python-myfitnesspal` HTML scraper the original implementation depended on. The approach described in Part 2 no longer works as written. [Part 3](/tech/mfp-mcp-part-3-cloudflare-killed-the-scraper/) tells that story and covers the replacement: MFP's undocumented mobile JSON API, which has no Cloudflare protection.

---

A few weeks ago I finished wiring Claude to Strava. Claude can now see every workout I've logged — distance, pace, heart rate, volume by week, how this month compares to last. That [series is here](/tech/wiring-claude-to-my-running-shoes-part-1-why-i-built-it/) if you want context.

Almost immediately I hit the next problem. I'd ask something like: *"Is my protein intake keeping up with my training volume this month?"* and Claude had no way to answer it. Half the picture was there. The other half — what I eat — was completely dark.

That's the gap this MCP fills.

---

### The data I already have

I've been logging food in MyFitnessPal on and off for years. Not obsessively, but consistently enough that the database is meaningful. Calories, macros, meal timing, days I didn't log at all — it's a real record of how I actually eat, not a curated version.

That data has never been useful outside the MFP app itself. I can see charts in the MFP dashboard. I can scroll my diary. I cannot ask a question about it.

The Strava MCP changed how I think about my workout data. I want the same thing for nutrition.

---

### The question that actually motivated this

The question I kept coming back to: **am I eating enough to support the training I'm doing?**

That sounds simple. It isn't. The answer depends on workout volume (Strava), calorie intake (MFP), macro split (MFP), which days I trained versus rested (both), and what my weight is doing over time (MFP measurements). No single app shows all of that. No dashboard I've found combines them.

Claude can hold all of it in context at once. Claude can reason across datasets. The only thing stopping it was access — it couldn't see either source until I built the MCP servers to give it that access.

Now it can see Strava. Now I'm adding MFP.

---

### Why MyFitnessPal specifically

A few reasons, same logic as the Strava decision.

**The data is there.** I've been using MFP long enough that there's a real dataset to analyze. Switching apps to get better API access would mean starting over. Not worth it.

**It's what I actually use.** The best data source is the one you consistently log into. MFP's barcode scanner and food database are fast enough that I actually do it.

**The alternative is manual.** Without this, my workflow is: export a CSV from MFP, paste it into Claude, ask my question, discard everything when the context window closes, repeat next time. That friction means I don't do it. Removing that friction is the whole point.

---

### The API problem

Here's the part that makes this trickier than the Strava build: **MyFitnessPal deprecated their public API in 2020.** No official way to pull your data programmatically. No OAuth flow, no API keys, no documented endpoints.

Strava's API has been around since 2012 and is well-maintained. MFP quietly pulled theirs and never replaced it.

The `myfitnesspal` Python library exists specifically for this situation. It authenticates against the MFP website using your credentials, maintains a session, and scrapes diary HTML into Python objects. I started there — and it worked, until it didn't.

What I didn't know when I first published this series: MFP runs Cloudflare Bot Fight Mode on their HTML pages. It lets browsers through and blocks everything else. The scraper worked initially because the session cookies held a short-lived Cloudflare clearance token. When that expired, every request started returning 403 with no path forward — not with Python requests, not with a Chrome-impersonating TLS stack, not with a real Chrome binary under automation.

The replacement is cleaner, and ironically less fragile: MFP's undocumented mobile JSON API at `api.myfitnesspal.com`. No Cloudflare protection. Clean JSON responses. A bearer token you can get using your existing session cookies from a single endpoint on the main domain. The whole scraping layer is gone.

[Part 3](/tech/mfp-mcp-part-3-cloudflare-killed-the-scraper/) is entirely about this — why the scraper broke, every bypass I tried, and how we ended up with a better implementation than the one I originally designed.

---

### The bigger picture

This is the first of what will eventually be a proper health dashboard. Here's what I'm building toward:

**Layer 1 — data sources, each with its own MCP server:**
- Strava: workouts, mileage, heart rate, volume ✓
- MyFitnessPal: calories, macros, weight, body composition ← this post
- Withings (eventually): sleep, HRV, continuous weight sync

**Layer 2 — cross-source questions:**
- How does my calorie intake track against my training load week over week?
- Am I in a calorie deficit on rest days and a surplus on long run days?
- What does my protein intake look like on days I lift versus days I run?
- How does my resting heart rate correlate with sleep and recovery?

**Layer 3 — a dashboard:**
A Claude agent that pulls from all sources and generates a weekly health brief. Not a chart — a conversation I can ask follow-up questions in.

That's the destination. Right now I'm building the roads.

---

### Local-first, same as Strava

My nutrition data is more personal than my workout data. Weight trends, eating patterns, bad weeks — this is not something I'm comfortable routing through third-party infrastructure.

The MFP MCP runs entirely on my machine. Auth uses your existing Chrome browser session — `browser_cookie3` reads the cookies Chrome already holds for MFP, so you never hand this tool your password. You do need to set `MFP_USERNAME` in a `.env` file (your MFP profile username, not your email). A bearer token is then fetched from MFP and cached in SQLite for up to 8 days. Nothing is transmitted beyond MFP's own endpoints. Same philosophy as the Strava server: Claude talks to a local process, the local process talks to the data source, nothing else is in the loop.

---

### The series

This is the first of two posts. Here's the structure:

1. **This post** — why I built it and where it fits in the bigger health dashboard
2. **The build** — the approach to a dead API, the Python stack, the six tools, how to connect it to Claude

The code is live at [github.com/IcaroBichir/mcp_myfitnesspal](https://github.com/IcaroBichir/mcp_myfitnesspal). The README has a copy-paste prompt you can send to Claude to set it up automatically, and a manual step-by-step if you prefer doing it yourself.
