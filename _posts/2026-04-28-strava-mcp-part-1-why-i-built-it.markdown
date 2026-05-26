---
layout: post
title: "Wiring Claude to My Running Shoes, Part 1: Why I Built It"
modified:
categories: tech
excerpt: "I wanted to ask Claude 'how did my training go this month?' and get a real answer. That meant building something."
tags: [mcp, strava, python, claude, ai, running]
comments: true
date: 2026-04-28T09:00:00-04:00
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

I train about two hours a day. Running, lifting, sometimes both. Strava has six years of that data — every split, every heart rate spike, every embarrassing slow Sunday. And yet every time I opened Claude to think through my training, I had to paste in a bunch of numbers manually like it was 2010.

That bothered me enough to build something.

---

### The problem: Claude is smart but blind

Claude can analyze training load, spot trends, flag overtraining signals, compare months, build weekly summaries. The analysis capability is genuinely there. What's missing is access — Claude has no idea what you actually did last Tuesday.

The usual workaround is to export a CSV from Strava, paste it in, and ask your question. This works once. It doesn't work when you want an ongoing relationship with your data, or when you want to ask follow-up questions, or when you just want to type "how does this week compare to the same week last year?" and get an answer without a data wrangling session first.

I wanted something closer to a conversation. Not a one-shot query — an actual back-and-forth where I could ask about pace trends, then ask why, then ask what to do about it.

---

### What MCP is (and why it matters here)

The Model Context Protocol is an open standard Anthropic published for giving AI models structured access to external tools and data sources. Think of it like a USB interface — instead of each integration being custom-wired, MCP defines a common plug shape.

When you build an MCP server, you're defining a set of **tools** that Claude can call. Claude decides when to use them, what arguments to pass, and how to incorporate the results into its response. From your side of the conversation, you ask a question and Claude answers it, having fetched the relevant data in the background.

This is different from RAG, which shoves context into the prompt upfront. With tools, Claude pulls data on demand, which matters for interactive conversations where you don't know what you'll need ahead of time.

---

### Why Strava specifically

A few reasons:

**The data is rich.** Strava activities include distance, time, heart rate, GPS, splits, elevation, power (if you have a sensor), segment efforts, best efforts, and more. There's a lot to analyze.

**The API is solid.** Strava's API has been around since 2012. It's well-documented, stable, and has a free tier that's generous enough for personal use (100 requests per 15 minutes, 1,000 per day).

**I already use it.** This sounds obvious, but it matters. I'm not building a demo — I'm building something I'll actually use every day. That changes the design decisions.

---

### Local-first by default

My training data is personal. Heart rate, weight, recovery patterns — I'm not interested in routing any of that through someone else's server to power an analytics product I didn't ask for.

The MCP server runs entirely on my machine. When Claude uses it, requests go from Claude → local server → Strava API → local server → Claude. Nothing is stored in the cloud beyond what Strava already has. The OAuth tokens live in `~/.config/strava-mcp/tokens.json` with `chmod 600`.

This isn't paranoia — it's just a reasonable default for health-adjacent data.

---

### What it looks like when it works

Once the server is running and connected to Claude, the conversation looks like this:

> **Me:** How does my running volume this month compare to last month?

> **Claude:** This month (April) you logged 12 runs for 87.4 km, averaging 7.3 km per run. Last month (March) you had 10 runs for 71.2 km, averaging 7.1 km. Volume is up 23%, roughly consistent with your long run progression — your two longest runs this month (14.2 km and 16.1 km) were both personal bests for the year.

That answer required five API calls to Strava. I asked one question in plain English. That's the gap this server closes.

---

### The series

This is the first of four posts. Here's where we're going:

1. **This post** — why I built it and what MCP is
2. **The build** — OAuth, the code, how it's structured, where to find it
3. **Using it** — real prompts, what Claude found, how I've changed my training because of it
4. **What's next** — limitations, features I want to add, and what the open source release taught me

The code is already live at [github.com/icarobichir/strava-mcp](https://github.com/icarobichir/strava-mcp). If you want to skip ahead, the README will get you running in about 10 minutes.
