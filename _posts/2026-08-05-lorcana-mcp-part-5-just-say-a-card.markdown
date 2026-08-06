---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 5: Just Say a Card"
modified:
categories: blog
excerpt: "I asked for a Ruby/Steel deck built around one newly-pulled card. What actually happened wasn't \"the MCP built it\" — build_deck handed over a rough first draft that got thrown out entirely once real synergy-reading started. The gap between those two is the actual story."
tags: [mcp, lorcana, python, claude, ai, tcg, deckbuilding]
comments: true
date: 2026-08-05T15:30:00-04:00
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

In [part four](/blog/lorcana-mcp-part-4-the-fix-that-broke-things/) I closed out a promo-mapping bug. This post isn't about a bug — it's about what a normal Tuesday actually looks like now, a year and change into this project, when I want a new deck.

---

### The old three tabs, one more time

Part one opened with the problem this whole project was built to solve: three browser tabs open just to answer "can I build this deck" — TCGPlayer for what I own, a deck site for the build, a wiki for card text nobody could remember correctly. Earlier this week I actually tried to go back to that habit for a minute, wanting to browse other people's decks on dreamborn.ink and lorcana.gg for inspiration instead of building from scratch.

It didn't work. dreamborn's deck browser is a client-rendered app sitting behind Cloudflare — nothing in the page you can actually read without executing its JavaScript. lorcana.gg blocks Claude's crawler by name in its `robots.txt`, full stop, no technical workaround worth attempting even if one existed. I gave up on browsing and just... asked for a deck instead.

That turned out to be the better move anyway.

---

### The actual ask

I'd just pulled **Dash Parr & Violet Parr - Super Siblings (Enchanted)** and wanted to build around it:

> "Can you build a deck for me based on Ruby & Steel for me to play and see if I like it. Use all the knowledge you have and search to see what could be a nice synergy, based in the cards I do have already."

One sentence. A color pair, a card I was excited about, and "use what I own." No CSV to open, no tab to switch to.

---

### What "the MCP does it" actually means

Here's the part worth being honest about, in keeping with this series' running theme of not overselling a tool. `build_deck` — the actual MCP function — is a **heuristic curve builder**. Ink pair in, legal ~60-card list out, scored on stat efficiency and keyword value. I ran it first, like I always do now, as a baseline:

It filled the curve with cards like Audrey Ramirez, Mata, and Peter Pan — fine bodies, wrong deck. It missed both Shift chains sitting in the collection entirely. One card at 5–6 cost, seven cards at 7+. I'd documented, back in part three, that `build_deck` says so itself in every result: *"it does not detect multi-card combos or synergy packages."* This was that disclaimer showing up in practice, not just in the fine print.

So the tool wasn't the thing that built the deck. What actually happened next was Claude reading — not searching, reading — the full text of every Ruby and Steel card I own:

```python
agg = defaultdict(lambda: {'qty': 0, 'ink': None, 'cost': None, ...})
for row in rows:
    if row['Ink'] not in ('Ruby', 'Steel', 'Ruby-Steel'):
        continue
    ...
```

173 unique cards, 585 total copies, dumped into one sorted table by cost. That's what actually surfaces synergy a heuristic can't see — not a smarter algorithm, just brute-force reading everything and cross-referencing ability text against the project's own documented combo library and keyword rules.

---

### What was hiding in there

**Dash Parr & Violet Parr - Super Siblings** has Combo Shift 6 — it can shift onto a Dash Parr, a Violet Parr, or *both at once*, and its ability draws a card for every card sitting underneath it whenever it quests or challenges. Shift onto both bases in one play and it draws 2 cards a turn while gaining 2 lore, Evasive, Resist +1 — a real engine, not a vanilla finisher.

The quieter one took actually reading a card I'd have skimmed past: **Mrs. Incredible - Helen Parr**, a forgettable 1/1/2 for 2, turned out to be a legal Shift base — by shared first name — for **Mrs. Incredible - Determined Rescuer**, a 7-cost Shift 5 body three cost tiers up. Nothing about the cheap version hints at the expensive one; you only find it by knowing every card sharing that exact name string exists somewhere in the pool, which meant checking the collection dump, not the deck builder's output.

Neither of those exists in `build_deck`'s scoring model. Stat-efficiency-per-cost has no way to represent "this 2-drop is secretly a discount on a 7-drop I haven't drawn yet."

---

### What "just say a card" actually costs

Rotation came up too, the way it always does now — I asked which sets were in bounds before touching a single card, because that's a standing rule in this project's own reference file, not something I re-derive per conversation. The answer ("Sets 9 through 13") shaped which of the two Wilds Unknown/Attack of the Vine! printings of each Parr character were even legal to reach for.

End to end: one sentence, one clarifying question back, one heuristic pass thrown out, one pass of actually reading 173 cards, two real combo lines found, a 60-card list verified twice over to land on exactly 60 — and it landed in a file next to the thirteen other decks already tracked in this project, ready to import to duels.ink.

That's genuinely faster than three tabs. It's just not because a tool "did it." It's because the tool handed off a rough draft, and reading — the unglamorous kind, every card, every line of text — did the rest.

---

If you collect Lorcana and you're running this MCP, don't just ask it to build a deck. Ask it to build a deck, then ask it what it actually did versus what a fancier scoring function would've let it skip. That question is where the interesting decks come from.
