---
layout: post
title: "Wiring Claude to My Running Shoes, Part 4: What's Next"
modified:
categories: tech
excerpt: "The server works. Now here's the honest list of what it still can't do, where I want to take it, and what building in public actually felt like."
tags: [mcp, strava, python, claude, ai, open-source]
comments: true
authored_by: ai
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

This is the last post in the series. [Part one](/tech/i-wired-claude-to-my-running-shoes-here-s-why/) was the why. [Part two](/tech/building-the-strava-mcp-server-oauth-tools-and-the-code/) was the build. [Part three](/tech/what-i-actually-did-with-my-strava-data-once-claude-could-read-it/) was what I learned from using it.

This one is the honest retrospective. What the v0.1 server can't do, what I'd add to it, and what I'd approach differently if I started today.

---

### Current limitations worth knowing

**No streaming data.** Strava's `/activities/{id}/streams` endpoint returns raw time-series: GPS coordinates, heart rate per second, power per second, cadence per second. The current server doesn't expose this. For most questions, you don't need it — but if you want Claude to analyze heart rate drift across a long run, or look at power distribution on a ride, you need the stream data. It's on the roadmap.

**Rate limits matter at scale.** Strava's API allows 100 requests per 15 minutes and 1,000 per day (for personal apps). For individual use, this is fine. But an analysis session where Claude is answering several questions about different activities can burn through 20-30 requests quickly. No caching exists yet — every call hits the API fresh. If you're doing a deep-dive analysis session, you'll want to be aware of this.

**No segment data.** If you run the same route regularly, Strava's segment efforts let you track performance on specific sections over time. This would be genuinely useful for answering "am I getting faster on this climb?" — but segments aren't in v0.1. The API supports it; I just haven't built it.

**No webhook support.** Currently there's no way for Claude to know when you've logged a new activity. You have to ask — and if you ask before Strava has synced, you get stale data. A webhook listener that receives Strava's push notifications when activities are created would fix this, but it requires a publicly reachable endpoint, which breaks the local-first model unless you get creative (ngrok, a small VPS, etc.).

---

### What I'd add next

In rough priority order:

**1. Response caching**

A simple in-memory or file-based cache with a short TTL (5-10 minutes) would reduce redundant API calls during a single analysis session significantly. Most questions in a conversation are about the same window of data — fetching it once and reusing it is an easy win.

**2. Streams endpoint**

Raw time-series data for heart rate, cadence, and power opens a different quality of analysis. Questions like "where in this run did my heart rate exceed 85% of max?" or "show me my power distribution across 30-second intervals on this ride" require stream data. This is the next meaningful capability improvement.

**3. Starred segments + segment efforts**

For runners and cyclists who use routes consistently, segment history is where "am I getting fitter?" actually lives. `get_starred_segments` and `get_segment_efforts` are both supported by the API. The main design question is how to surface them usefully — probably as a tool that takes a segment ID and returns all your historical efforts on it.

**4. Richer activity filtering**

`list_activities` currently filters by sport type and date range. I'd want to add filtering by gear, by distance range, and by effort level (suffer score or heart rate zone). These would make analysis sessions more targeted without requiring Claude to fetch 200 activities and filter manually.

---

### What I'd approach differently

**I'd write a test suite from day one.** The server has no automated tests. For something this simple, it's fine — but once streams and caching and segment logic get added, the complexity will warrant proper coverage. The auth flow in particular (the local HTTP server, token refresh logic) would benefit from tests.

**I'd think harder about the tool boundaries.** Right now `get_activity` returns the full Strava activity object, which is large and includes fields Claude rarely needs. I applied a key filter to `list_activities` (the `_SUMMARY_KEYS` set), but not to `get_activity`. In practice this means a single activity fetch can push a lot of tokens. A smarter approach would be to return a summary by default and offer a `verbose` flag for when you need the full detail.

**I'd document the tool descriptions more carefully.** The docstrings on each tool become the descriptions Claude reads when deciding whether and how to use them. I wrote them quickly and they work, but there's room to be more precise about edge cases — what happens if no activities match the filter, what the valid values for `sport_type` are, what date format `after` and `before` expect.

---

### Building in public

I published the repository on GitHub before the blog series was written. That order matters — the code is what it is, and the writing is honest about its current state rather than polished to match an idealized version.

If you've built something on top of this, opened an issue, or just installed it — thank you. Knowing that someone other than me is using a side project is more motivating than any metric.

<div class="human-aside">
<p>I published the code before writing any of the blog posts. on purpose. because I know myself — if I wrote the posts first I would have polished them indefinitely and published nothing. so I reversed it: push the repo, then explain what it does. accountability through doing things in the wrong order. someone starred it before I'd written a single word about it. I still don't know how they found it. if that's you: hi.</p>
</div>

The repo is at [github.com/icarobichir/strava-mcp](https://github.com/icarobichir/strava-mcp). PRs are open. If you want to take a crack at streams or segment support, open an issue first so we can align on approach.

---

### What comes after Strava

The pattern — local MCP server, personal data, read-only access, Claude as the interface — generalizes. Strava was first because it's the data source I interact with most daily.

The next candidates: health records (lab results, trends over time), and maybe MyFitnessPal or a similar nutrition log. The questions I want to ask aren't fundamentally different — "how is X trending?", "what does Y correlate with?", "what am I not noticing?" — but the data sources are.

Local-first, single-athlete, honest about limitations. That's the pattern I'll keep building to.
