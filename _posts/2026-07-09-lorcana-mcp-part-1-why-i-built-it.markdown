---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 1: Why I Built It"
modified:
categories: tech
excerpt: "Three tabs open just to answer 'can I build this deck': TCGPlayer for what I own, dreamborn.ink for the deck, a wiki for the card text. So I gave Claude the collection instead."
tags: [mcp, lorcana, python, claude, ai, tcg]
comments: true
date: 2026-07-09T09:00:00-04:00
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

I collect Disney Lorcana. A few hundred cards, spread across a dozen sets, with the usual TCG problem: I never actually know what I own relative to what I'd need to build something good. Every time I wanted to answer "can I build a competitive Amber/Sapphire deck with what I have," it turned into a three-tab exercise — TCGPlayer for my export and prices, dreamborn.ink for the deck builder, a wiki for card text and rulings I couldn't remember. Twenty minutes of tab-switching to answer one question.

If you've been reading along, this is the same shape of problem I solved for training data with [strava-mcp](/tech/strava-mcp-part-1-why-i-built-it/) and nutrition with [mfp-mcp](/tech/mfp-mcp-part-1-why-i-built-it/): Claude is genuinely good at this kind of reasoning — curve analysis, format legality, cost math — it just can't see the data. So I built [lorcana-mcp](https://github.com/IcaroBichir/lorcana-mcp) to give it a real card database and my actual collection.

---

### The problem: three sources of truth, none of them talking to each other

A Lorcana collection has three distinct pieces of data, and no single place has all three:

- **What you own** — TCGPlayer's export CSV, if you've kept it current. It has quantities and prices, but no gameplay data — no ink cost, no keywords, no ability text.
- **What a card does** — split across two community APIs (LorcanaJSON for recent sets, lorcana-api.com for older ones), plus a wiki when the APIs are wrong or incomplete.
- **What's legal** — format rotation is its own moving target. Core Constructed drops whole sets in groups, not one at a time, and the "safe" window shifts every time a new set goes live.

None of this is Claude's fault for not knowing — it's that the data was never in one place for anything to read, including me.

---

### Different shape than the health servers

Strava, MyFitnessPal, and Withings all needed an OAuth dance before they were useful — you're authenticating as yourself against a service that holds your personal data. Lorcana doesn't have that problem at all. The card data is entirely public (LorcanaJSON, lorcana-api.com, duels.ink, and tcgcsv.com for pricing are all open, no-auth APIs), and the only "personal" piece is a CSV you already exported yourself.

That means `lorcana-mcp` needs zero configuration. No account, no login, no `auth` command to run first. `pip install`, point it at your export, done. That's a genuinely different kind of MCP server to build — no token refresh logic, no scopes to think about — and it made the whole project feel more like a data-modeling problem than an integration problem.

---

### What it does today

Ten tools, roughly in the order I built them:

| Tool | What it does |
|---|---|
| `enrich_csv` | Turns a bare TCGPlayer export into a real database — ink cost, stats, keywords, ability text — plus a dreamborn.ink-ready import file |
| `lookup_card` / `resolve_card` | Look up a card by name, including fuzzy/informal names ("goofy musketeer," "big pete") |
| `search_cards` | Filter the whole card pool by color, type, rarity, cost, keyword, ability text |
| `find_song_synergies` | Who can sing a given song, and for how cheap |
| `filter_collection` | Which of your cards are legal in Core, Infinity, or Poorcana |
| `audit_csv` | Sanity-check an enriched CSV against live data |
| `analyze_deck` | Curve, color split, and legality for a pasted decklist |
| `what_am_i_missing` | Diff a decklist against your collection and price the gap |
| `build_deck` | Automatically assemble a legal, curve-balanced deck for an ink pair |

That last one — `build_deck` — didn't exist when I started this post series. It showed up mid-project because a conversation about "how should Claude build decks for me" turned into "why isn't this just a tool everyone gets." That's the whole of part 3.

---

### What it looks like when it works

> **Me:** I want a Core Constructed Amber/Sapphire deck. What am I missing and what would it cost?

> **Claude:** *(calls `build_deck` with `mode="ideal"` against your collection CSV)* — Here's a curve-balanced 60-card build. You already own 22 of the 60 slots. To complete it: 4x Merlin's Carpetbag ($2.08), 4x The Queen - Fairest of All ($2.88), 3x Akood et Emuti ($14.94)... **Estimated cost to complete: $31.32**, live TCGPlayer snapshot via tcgcsv.com.

That answer used to be a spreadsheet. Now it's one sentence.

---

### The series

Three posts:

1. **This post** — the problem and why it's shaped differently than the health MCP servers
2. **The build** — the enrichment pipeline, the fuzzy card-matching engine, and what I learned auditing two other open-source Lorcana MCP servers
3. **Building an AI deck builder** — the heuristic scoring engine behind `build_deck`, two real bugs it took to get there, and shipping it to PyPI and the MCP Registry

The code is live at [github.com/IcaroBichir/lorcana-mcp](https://github.com/IcaroBichir/lorcana-mcp), published on PyPI as `lorcana-mcp`, and listed on the [official MCP Registry](https://registry.modelcontextprotocol.io/servers/io.github.IcaroBichir/lorcana). If you collect Lorcana and want to skip ahead: `pip install lorcana-mcp`, then `claude mcp add lorcana -- lorcana-mcp serve`.
