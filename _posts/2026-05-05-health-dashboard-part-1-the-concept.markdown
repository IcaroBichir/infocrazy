---
layout: post
title: "Health Dashboard, Part 1: The Concept and First Scratch"
modified:
categories: tech
excerpt: "I have Strava and MyFitnessPal talking to Claude. The natural next step was a dashboard that puts everything in one place — not a chat, an actual screen I can glance at."
tags: [health, dashboard, strava, myfitnesspal, python, streamlit, claude, ai]
comments: true
authored_by: ai
date: 2026-05-05T18:00:00-04:00
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

Over the past few weeks I built two MCP servers: [one for Strava](/tech/wiring-claude-to-my-running-shoes-part-1-why-i-built-it/) and [one for MyFitnessPal](/tech/building-my-health-dashboard-part-1-why-myfitnesspal-needed-an-mcp/). Both run locally. Both give Claude read access to data that was previously locked in apps.

The result is that I can have real conversations about my health data. That was the goal, and it works.

But conversations have a problem: you have to start them. Every time I want to see how my week is going, I open Claude and ask. It's fast, but it's still friction. I want something I can just *look at*.

That's the dashboard.

---

### The problem with the apps I already have

Strava has a dashboard. MyFitnessPal has a dashboard. Withings has a dashboard. Each one is a silo — it shows you its own data, in its own format, with no awareness that the other two exist.

What I actually want to know is not "how far did I run this week." It's "how far did I run, how many calories did I burn, how much did I eat, what's my weight doing, and does any of it add up?" None of the existing apps answer that question. They each answer one-third of it.

So I'm building the dashboard myself.

---

### What this first version does

The v0.1 is deliberately narrow. Just Strava. Just activities. Three sections:

**Today** — every workout logged since midnight. Most mornings this is one run or one lifting session. Sometimes it's empty.

**Yesterday** — same thing, one day back. Useful for a quick sanity check: "what did I actually do yesterday?" without having to remember.

**Last 7 Days** — the full list of activities for the rolling week, plus four aggregate numbers at the top: total activities, total runs, total run distance, and total moving time.

Each activity card shows the same set of fields regardless of which section it's in: name, time of day, distance, moving time, and two metrics that are sport-aware. For runs, that's pace and heart rate. For rides, it's speed and heart rate. For everything else — lifting, walks, hikes — it's elevation and heart rate.

That's it. No charts yet. No trends. No nutrition. This is the skeleton.

---

### The stack

The dashboard is a [Streamlit](https://streamlit.io) app that imports `StravaClient` directly from the strava-mcp package. No HTTP layer between them — it's a Python import.

That means the SQLite cache built into strava-mcp applies automatically. When the dashboard loads, it makes one call to fetch the last 7 days of activities. That response is cached locally for an hour. Subsequent page loads or widget interactions read from SQLite, not from the Strava API. Refreshing the screen is instant.

The architecture is intentionally flat:

```
Streamlit app
    └── StravaClient (strava-mcp)
            └── SQLite cache (~/.config/strava-mcp/cache.db)
                    └── Strava REST API (on cache miss)
```

There's a **↺ Refresh** button in the UI that clears Streamlit's in-memory state and re-fetches. If you want to force a hard refresh all the way to the Strava API, you run `strava-mcp cache clear` in the terminal first.

The code lives at `/Users/icaro/icaro_lifestyle/tech/health_dashboard`.

---

### What's missing from this version

A lot.

The activity cards only show distance, time, pace, and heart rate. Strava gives us more: elevation gain, suffer score, PR count, achievement count, gear used, splits, best efforts. None of that is surfaced yet. I want to add elevation as a standard field, and a badge for PRs so a strong effort is visually distinct from a recovery run.

There's no weekly trend. The "Last 7 Days" section shows aggregate totals, but no sense of whether that's more or less than the previous week, or whether it's tracking toward your monthly goal. That context is the part that actually matters.

There's no nutrition. I can see workouts. I cannot see calories, macros, or weight. Half the picture.

---

### Where this is going

**More exercise depth.** Elevation, suffer score, PR flags, and eventually splits and heart rate zone breakdowns. The goal is a card that tells the full story of a workout at a glance — not just that I ran 10K, but that it was a hard effort, set a 5K best split, and cost 850 calories.

**MyFitnessPal integration.** This is the other half. Once nutrition data is on the same screen as workout data, the questions get interesting. Calorie balance. Protein on training days versus rest days. Whether my intake is keeping up with my volume. The MFP MCP server was already built — wiring it into the dashboard turned out to be the thing that revealed the MFP server was silently broken. [Part 2](/tech/health-dashboard-part-2-wiring-in-nutrition/) covers the nutrition UI and the Cloudflare debugging arc that came with it.

**A weekly summary panel.** Not a chart, a brief. Seven days of workouts and nutrition distilled into five numbers and a sentence about the trend. The kind of thing you check on Sunday morning before planning the week ahead.

This first version is a scratch in the direction of all of that. The useful part is that it runs, it loads fast, and it shows me real data when I open it.

The rest is iteration.
