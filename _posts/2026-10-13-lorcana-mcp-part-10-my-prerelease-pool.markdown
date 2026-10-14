---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 10: \"Here's My Prerelease Pool\""
modified:
categories: blog
excerpt: "The Hyperia City prerelease is this Friday: six packs, 45 minutes, build a 40-card deck. v2.6.0 adds build_sealed_deck. Send Claude a list or a photo of what you opened and get a deck back."
tags: [mcp, lorcana, python, claude, ai, tcg, deckbuilding, sealed]
comments: true
date: 2026-10-13T12:00:00-04:00
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

The **Hyperia City prerelease** is this Friday, October 16. It's Sealed: you open six booster packs at the store (about 72 cards), and you have roughly 45 minutes to turn them into a deck.

Every deck tool in `lorcana-mcp` assumed Constructed rules up to now: 60 cards, at most 4 copies of a card, two inks, format legality. None of that applies on Friday. So [v2.6.0](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.6.0) adds one new tool built for it: `build_sealed_deck`.

---

### The rules it follows

- **40-card minimum.**
- **No ink limit.** You can play as many colors as you want.
- **No copy limit.** If you open five copies of a card, you can play all five.
- **Everything you don't play is your sideboard**, and you can swap cards in between games.

There's no legality check, because the set doesn't need to be released yet. The tool reads straight from Hyperia City's card list.

---

### How to use it

Open your packs, then hand Claude the pool. There are two ways.

#### 1. Type or paste a list

One card per line, with a quantity if you have more than one copy. Names don't need to be exact:

> *"Here's my prerelease pool, build me a deck:*
> *2x Napoleon - Patient Watchdog*
> *mickey best in town*
> *Stacey Powerline Superfan*
> *..."*

Collector numbers work too. Every card has one printed at the bottom (`123/204 · EN · 14`), and it's often faster to type than the name:

```
2x 23/204
155
#57
```

#### 2. Send a photo

Spread the cards out on the table, take a picture, and send it with *"build my sealed deck from this."* Claude reads each card and passes them to the tool, one line per card. The tool's description tells Claude to use the collector numbers whenever they're readable, because they hold up better than names under glare or when cards overlap.

Then Claude tells you which cards it couldn't read, so you can confirm them instead of trusting a guess.

---

### What you get back

**First, which colors to play.** It scores all 15 two-ink pairs by how deep your pool is in each one:

```
| Pair           | Playables | Strength |
|----------------|-----------|----------|
| Ruby/Steel     | 32        | 100.8    |
| Ruby/Sapphire  | 28        | 97.3     |
| Amethyst/Ruby  | 30        | 97.2     |
```

**Then the deck,** built around the best pair (or the inks you ask for), sorted by cost, with uninkable cards marked.

**A splash only if you need one.** Six packs give each pair only about 24 cards on average, so the best pair often can't reach 40 alone. When that happens, the tool adds the one extra ink whose best cards fill the gap:

```
## Sealed deck — Ruby/Steel + Amber splash
Ruby/Steel has only 32 playable cards in your pool, so Amber fills the
rest. Ink is colorless in Lorcana — a splash costs synergy, not consistency.
```

That last line is the reason Sealed is more forgiving than it sounds. Ink has no color in Lorcana: any inkable card pays for any card. A third color doesn't make your deck harder to play the way it would in other card games. It only means fewer cards that work together.

**Plus the extras:** curve and inkable counts, a warning if you're running too many uninkables, your strongest sideboard cards for swapping between games, and a plain list you can read off while you sleeve.

---

### It doesn't guess

Hyperia City has three different Merlins. If you type just "merlin," the tool won't pick one for you:

```
### Ambiguous — not included
- 1x "merlin" → Merlin - Bauble Expert (#39);
  Merlin - Ink Drop Tinkerer (#52); Merlin - Profoundly Curious (#57)
```

The numbers next to each one let you answer in one word. Cards it can't match at all get listed separately, so nothing disappears from your pool without you knowing.

---

### What it still isn't

Like `build_deck`, it's a heuristic: cost curve, stats, keywords, and Shift chains. Two gaps matter for Friday:

- It doesn't understand combos.
- It doesn't value Hyperia City's new **ink drop** mechanic yet. That's next on the list.

Treat the deck as a fast first draft, so your 45 minutes go to tuning it instead of sorting cards into piles.

### Try it

```
pip install -U lorcana-mcp
```

Restart your MCP client and bring your pool to Claude on Friday.

- [v2.6.0 release](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.6.0) · [PyPI](https://pypi.org/project/lorcana-mcp/) · [MCP Registry](https://registry.modelcontextprotocol.io/?q=lorcana)
- Setup for Claude Desktop: [Add lorcana-mcp to Claude Desktop](/blog/add-lorcana-mcp-to-claude-desktop/)
- Earlier: [Part 9 — "Build Me a Princess Deck"](/blog/lorcana-mcp-part-9-build-me-a-princess-deck/) · [Part 1 — Why I Built It](/blog/lorcana-mcp-part-1-why-i-built-it/)
