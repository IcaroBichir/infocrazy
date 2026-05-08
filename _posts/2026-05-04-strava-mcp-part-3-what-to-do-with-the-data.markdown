---
layout: post
title: "Wiring Claude to My Running Shoes, Part 3: Using the Data"
modified:
categories: tech
excerpt: "I've been using the Strava MCP server for a few weeks. Here are the prompts that worked, the things Claude found that surprised me, and how it's changed what I track."
tags: [mcp, strava, python, claude, ai, running, training]
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

Once the server was running and Claude could see my Strava data, my first instinct was to ask the most obvious thing: *"Show me my last 10 runs."* Claude returned a formatted table with date, distance, pace, and heart rate. Interesting, but not the point.

The point is what comes after. The follow-up questions. The things you'd only think to ask once you've seen the numbers. That's where this kind of tool earns its place.

This post is about the prompts that actually unlocked something useful — and a few things Claude found that I didn't expect.

---

### Prompts that work well

**Volume and trends:**

> "How does my running volume this month compare to the same month last year? Show totals and weekly averages."

> "Over the last 90 days, is my long run distance trending up, down, or flat? Show me a week-by-week breakdown."

These are the questions I used to answer manually by scrolling through Strava. Now I just ask.

**Pace analysis:**

> "Look at my last 20 runs. Is my pace improving at the same effort level, or am I just running faster by working harder?"

This one is harder to answer from raw numbers. Claude cross-references pace with average heart rate across sessions and gives you a read on whether the efficiency is actually improving. Not a formal VO2 estimate — just pattern recognition on the data you have.

**Gear tracking:**

> "Which shoes have the most kilometers on them? Are any approaching the 800 km replacement point?"

> "I have two pairs of running shoes. How am I alternating between them? Do I have a clear preference?"

Strava tracks gear per activity if you set it up. I hadn't thought much about this data until Claude surfaced it.

**Comparing sport types:**

> "On weeks where I run more than 40 km, how does my strength training volume change compared to weeks under 40 km?"

This revealed that I tend to drop lifting sessions when run volume is high — not intentionally, just a pattern. Worth knowing.

---

### The thing Claude found that surprised me

I asked Claude to look at my last six months and tell me something I probably hadn't noticed.

It flagged that my Saturday long runs average 40 seconds per km slower than my Tuesday runs at similar distances — not just slower in absolute terms, but disproportionate relative to the distance difference. Its hypothesis: I'm not recovering well from Friday's strength session.

I hadn't thought about it that way. I just thought Saturdays were "recovery pace" runs. Turns out they might be "accumulated fatigue" runs, which is different. I've since moved my heavy leg day to Thursday, and the Saturday data has shifted. Still early, but something to watch.

<div class="human-aside">
<p>I actually moved my leg day. thursday instead of friday. because an AI looked at six months of my running logs and noticed my saturdays were consistently slower than the distance should explain. I've had coaching advice from actual humans that was less actionable than this. what gets me is: I think I already knew. somewhere I knew. I just needed something with no ego, no agenda, no "well it depends" — just something that would look at the numbers and say "this looks like accumulated fatigue, not recovery." I moved my leg day.</p>
</div>

I want to be clear: Claude isn't giving me a medical opinion or a certified coaching plan. It's pattern-matching on six months of activity logs. But pattern-matching with good context is genuinely useful, especially when you have enough data.

---

### Prompt patterns that help

A few things I've learned about how to ask:

**Ask for tables.** Claude formats training data well as tables, and tables are easier to scan than prose summaries. Explicitly ask: "show this as a table with columns for date, distance, pace, and heart rate."

**Set a time window.** Open-ended questions like "how is my training going?" generate vague answers. "Over the last 30 days, how is my running going?" gets you something specific.

**Ask for one thing at a time, then follow up.** I get better answers when I start with a specific question and drill down, rather than asking for a full training report upfront. Claude can go deep on one thread when that's what you ask for.

**Name what you want to know, not what you want to see.** Instead of "show me my heart rate data," try "is my resting heart rate trending down over the last 6 weeks, which would suggest improving aerobic fitness?" You get an answer, not a dump.

---

### What it can't do

Claude has read-only access. It can analyze, summarize, and compare — it cannot log activities, edit records, or write anything back to Strava. This is intentional. The MCP server uses `activity:read_all` and `profile:read_all` scopes only.

It also has no real-time data. The Strava API returns what's been synced. If you just got back from a run, you need to open Strava and let it sync before Claude can see it.

And Claude is not a coach. It has no knowledge of your injury history, your goals, your schedule, or your physiology beyond what's in the Strava data. It's a sharp analyst working with incomplete information. Treat the outputs accordingly.

---

### How it's changed what I track

The more useful Claude is with a given data type, the more I want that data to be clean. A few things I've started doing as a result:

- **Logging gear per activity consistently.** I was sporadic about this before. Now the gear data is actually useful.
- **Adding a short description to long runs.** Claude can read activity names and descriptions. "14 km easy, felt tired" is context that affects analysis.
- **Not skipping heart rate.** I sometimes ran without my monitor when the battery was low. Now I bother to charge it.

None of this is revolutionary. But having a tool that actually uses the data is good motivation to collect it properly.

---

Next: [what I'd add, what I'd change, and where the project goes from here](/tech/strava-mcp-improvements-and-next-steps/).
