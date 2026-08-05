---
layout: post
title: "Health Dashboard Part 6: Closing the Loop with Body Composition"
modified:
categories: blog
excerpt: "Part 5 ended with a gap: nutrition, training load, and calorie balance — but no scale data. Withings body composition is now wired into every tab, and the AI coach evaluation in Nutrition × Exercise can finally ask whether the numbers are moving in the right direction."
tags: [health, dashboard, withings, python, streamlit, bodycomposition, nutrition, strava]
comments: true
date: 2026-05-29T11:00:00-04:00
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

The final section of [Part 5](/blog/health-dashboard-part-5-nutrition-exercise-correlation/) listed what was still missing:

> **Weight trend.** MFP measurements (body weight over time) would complete the picture — you'd see nutrition, training load, and what the scale is doing in one place.

That gap is closed. Withings body composition — weight, fat%, muscle mass, fat-free mass, bone mass, hydration — is now present across the dashboard, and most importantly wired into the Nutrition × Exercise tab's AI coach evaluation.

---

### The data source

The Withings scale produces a richer reading than a standard weight scale. A single step-on generates six body composition values: weight, fat ratio, fat mass, fat-free mass, muscle mass, bone mass, and hydration. These come back from the API as typed measurement records — integer values with a unit exponent — merged per day into one clean dict.

The dashboard imports `WithingsClient` directly, the same pattern as `StravaClient` and `MFPClient`. It gets the SQLite cache for free: 24-hour TTL for measurements (scale data doesn't update mid-day), 15-minute TTL for today's activity data.

---

### Where it appears

**Today & Yesterday** — a compact card below calorie balance shows the most recent scale reading: weight, fat%, muscle. No trend here since you're looking at a single day, but you always know when the last reading was taken.

**Last 7 Days** — a full body comp card sits in the weekly summary block, above the per-day breakdown. If there are two or more readings in the 7-day window, each field shows a delta vs the oldest reading:

```
Weight     74.60 kg   (+0.15 kg)
Body Fat   16.6%      (−1.0%)
Muscle     59.38 kg   (+0.87 kg)
Fat-Free   62.22 kg   (+0.55 kg)
```

**Last 30 Days** — same full card, covering the full 30-day window. This is where meaningful trends appear: a full month of training and nutrition data has enough resolution to see if fat% is moving.

---

### The Nutrition × Exercise tab

This is where body composition matters most. The tab already had nutrition averages and calorie balance. What it was missing was the outcome: is the combination of food and training actually doing what you intend?

Body composition now anchors the top of the tab — before the 15-day and 30-day sections — as a standalone snapshot:

```
⚖️ Body Composition

Most recent reading: 2026-05-29. Trend deltas vs earliest reading in window (2026-05-01).

Weight     74.60 kg   (−0.60 kg)
Body Fat   16.6%      (−1.0%)
Muscle     59.38 kg   (+0.87 kg)
Fat-Free   62.22 kg   (+0.67 kg)
Fat Mass   12.38 kg   (−1.50 kg)
Bone Mass   2.82 kg   (−0.01 kg)
Hydration  41.52 kg   (+0.44 kg)
```

That snapshot is followed by the 15-day and 30-day correlation sections, each with their workout-vs-rest-day nutrition comparison and energy balance cards.

---

### The AI coach evaluation

The correlation evaluation was already analyzing prior-day carbs vs next-day pace, protein vs recovery, and calorie balance sustainability. It now has a fifth dimension: body composition.

The evaluation context passed to `claude -p` includes the most recent scale reading and the trend deltas for the period:

```
### Body composition (most recent reading: 2026-05-29)
- weight 74.6 kg | body fat 16.6% | muscle 59.4 kg | fat-free mass 62.2 kg
- Change over period: weight −0.6 kg | body fat −1.0% | muscle +0.9 kg
```

The coach prompt adds a fifth instruction: assess whether the calorie balance and macros are consistent with the body composition trend the scale is showing — call out if the data supports or contradicts the athlete's likely goals.

This closes the loop that the previous version couldn't close. A typical evaluation might now read:

> The 30-day calorie deficit averaging 380 kcal/day is consistent with the 0.6 kg weight loss. More interesting: fat mass dropped 1.5 kg while muscle increased 0.9 kg — a body recomposition pattern, which aligns with the protein intake averaging 142g/day (1.9g/kg bodyweight). The deficit isn't aggressive enough to risk lean mass loss at this protein level. The only gap is the two weeks in early May where protein dropped below 110g/day — those weeks show slower pace and higher HR, but the scale data doesn't show any muscle loss, suggesting the window was short enough not to affect composition.

That kind of analysis — crossing nutrition, training performance, and body composition in the same paragraph — wasn't possible before.

---

### How evaluation generation picks it up

The `generate_evals.py` script runs at dashboard startup via `./start`. It already fetches Strava and MFP data to build the correlation context. It now also fetches Withings measurements for the period and passes them into `build_correlation_context()`:

```python
withings_month = get_withings_measurements(MONTH_START, TODAY)

_generate_corr("15d", "Last 15 Days", all_month, nutrition_month,
               FIFTEEN_START, withings_month, force)
_generate_corr("30d", "Last 30 Days", all_month, nutrition_month,
               MONTH_START,   withings_month, force)
```

The Withings call is cached at 24 hours, so it adds no meaningful latency to startup.

---

### What the dashboard looks like now

Five tabs, three data sources, AI evaluations at two layers:

| Tab | Strava | MFP | Withings |
|---|---|---|---|
| Today & Yesterday | ✓ | ✓ | compact card |
| Last 7 Days | ✓ | ✓ | full card + trend |
| Last 30 Days | ✓ | ✓ | full card + trend |
| 🏃 Running | ✓ | — | — |
| 🔗 Nutrition × Exercise | ✓ | ✓ | snapshot + in AI eval |

The running tab doesn't include body composition because the relationship there is indirect — body comp is an outcome of the nutrition-training combination, not a run-level metric. It belongs in the correlation view.

---

### What's still open

**Workout type differentiation.** The Nutrition × Exercise tab treats all activities the same. A strength session has a different relationship with protein and calories than a 15K run. Separating by sport type would sharpen the correlation analysis.

**Longer windows.** Thirty days is still the ceiling on data fetching. Body composition changes slowly — a 60 or 90-day window would be more useful for spotting genuine trends vs week-to-week noise.

**Withings sleep data.** The MCP server already exposes `get_sleep_summary`. Sleep quality affects both training performance and body composition. It's the next obvious addition to the correlation tab.

The core loop — did what I ate and how I trained this month show up on the scale — is fully answerable now. The dashboard runs locally, requires no cloud infrastructure beyond the original Strava, MFP, and Withings accounts, and generates coaching evaluations via the Claude CLI already running in the terminal.

[Part 4](/blog/health-dashboard-part-4-ai-coach-without-api-key/) covers the evaluation architecture in detail. The Withings MCP that powers the body composition data is described in the [companion post](/blog/withings-mcp-body-composition-in-claude/).
