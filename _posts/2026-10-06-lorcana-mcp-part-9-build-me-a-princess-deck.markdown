---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 9: \"Build Me a Princess Deck\""
modified:
categories: blog
excerpt: "The deck builder only understood ink colors. People don't ask for decks that way — they ask for a Princess deck, a Detective deck, a Stitch deck. And Hyperia City's cards were sitting in the database, invisible to it for another two weeks. v2.5.0 adds themed decks and a preview mode for unreleased sets, and this is how to use both."
tags: [mcp, lorcana, python, claude, ai, tcg, deckbuilding]
comments: true
date: 2026-10-06T10:00:00-04:00
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

`build_deck` has been in `lorcana-mcp` since [Part 3](/blog/lorcana-mcp-part-3-building-an-ai-deck-builder/). You give it an ink pair, a format, and a mode (only cards you own, the ideal deck priced against your collection, or the ideal deck at full market price), and it assembles a legal, curve-balanced 60.

It had a blind spot I hadn't noticed until I tried to talk to it like a player. Nobody says *"build me an Amber/Ruby deck."* They say *"build me a Princess deck."* *"I want something with Detectives."* *"Make me a Stitch deck."* The theme comes first and the ink colors are a detail, sometimes one they don't know yet.

And a second gap showed up the same week: **Hyperia City**. The set has been fully revealed and LorcanaJSON already lists its cards, but it isn't legal until October 23. Until then the deck builder acted as if those cards didn't exist, which is exactly when you most want to brew with them.

[v2.5.0](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.5.0) fixes both.

---

### Themed decks

Every Lorcana character carries *classifications*: Storyborn, Hero, Villain, Princess, Detective, Pirate, Seven Dwarfs, Toy, Super. There are 49 of them in the card data. They were always there; the builder just never looked at them.

Now it does. You can just ask Claude in plain language:

> *"I need a deck based on princesses, Core legal, with Ruby and Amber inks."*

Claude turns that into one call:

```
build_deck(theme="princess", format="core", ink_colors="Ruby,Amber")
```

`theme` accepts either a classification (`"princess"`, `"detective"`, `"seven dwarfs"`, `"villain"`) or a character name (`"mickey mouse"`, `"stitch"`). Capitalization and plurals don't matter, so "Princesses" works fine.

#### Members and payoffs

The first version I sketched just boosted every Princess character. That gets you a pile of Princesses, not a Princess *deck*. What makes a tribal deck work is the handful of cards that *care* about the tribe:

- **Beast - Gracious Prince:** *"Your Princess characters get +1 ¤ and +1 ⛉."*
- **Aurora - Holding Court:** quest with her and your next Princess or Queen costs 1 less.
- **Moana - Of Motunui:** readies your other Princesses when she quests.

Beast isn't a Princess himself, so a pure "boost the tribe" rule skips right past the best card for it. So the builder scores two things separately:

- **Members** carry the classification (or the character name). They get a bonus.
- **Payoffs** have rules text that mentions the theme. They get another bonus, and a card that's both (Aurora is a Princess *and* rewards Princesses) gets both.

The bonus is tuned so a mediocre Princess beats most unrelated filler, but doesn't beat a genuinely strong staple. It's a strong preference, not a straitjacket.

One small detail mattered more than I expected: payoff matching needs a *whole word*. Without that, every card mentioning a "Prince" counts as a Princess payoff. Prince and Princess are both real classifications, and they are not the same deck.

#### Don't know the colors? Don't pass them

Ink colors are optional when you give a theme. Leave them out and the builder takes the format's legal card pool, counts theme cards in every two-ink pair, and picks the best one:

```
build_deck(theme="detective", format="core")
```
```
Inks auto-picked for theme: Sapphire/Steel has the most Detective cards
legal in Core EN (27 distinct).
```

That's the right answer: Judy Hopps and Nick Wilde live in those colors. It's the question a new player can't answer yet and shouldn't have to.

#### It tells you how themed the deck actually is

The stats block now ends with a theme line. For the Ruby/Amber Princess example:

```
- Theme (Princess): 32/60 cards are Princess characters; payoffs:
  Aurora - Holding Court, Cinderella - Gentle and Kind, Moana - Of Motunui
```

And when a theme can't carry a deck, it says so instead of quietly handing you 60 cards of filler with a Princess sticker on it. Musketeer has exactly one card left in Core rotation, so you get:

```
- THIN THEME: only 0 Musketeer cards made the deck — the legal
  Amber/Amethyst pool doesn't have enough. This is already the best pair
  for it — try a non-rotating format like infinity.
```

That honesty is the same principle as the [last post](/blog/lorcana-mcp-part-8-the-snapshot-problem/): a tool that sounds confident should also say when it's out of its depth.

---

### Preview mode: Hyperia City before release

Legality in `lorcana-mcp` comes from duels.ink, and duels.ink doesn't list a set as legal anywhere until it actually releases. So for the two weeks when everyone is theorycrafting with the new cards, the deck builder couldn't use any of them.

The new flag:

```
build_deck(ink_colors="Amber,Amethyst", format="core", include_preview=True)
```

The interesting part was deciding what counts as "upcoming." LorcanaJSON marks both unreleased sets *and* rotated-out sets as not Core-legal, so `allowed: false` alone can't tell Hyperia City apart from sets that rotated out in July. The rotation group can. Every set belongs to one, and a set that isn't legal yet but sits in the newest group (or a newer one) must be upcoming, while a set in an older group has rotated out. So nothing is hardcoded to Hyperia City; Into the Inkdark will be picked up the same way once its cards appear.

Preview cards are clearly marked so you never mistake a brew for a legal list:

```
Preview mode: includes not-yet-legal cards from Hyperia City (112 cards
revealed, releases 2026-10-23). Card pool may be incomplete until release.

| 1 | Mickey Mouse - Best in Town (preview) | Character | 4 |
| 5 | Everyone Knows Juanita (preview)      | Action - Song | 4 |
```

The duels.ink import block stays clean, so you can still paste it straight into a deck builder. Two caveats are printed rather than hidden: the pool is whatever LorcanaJSON has so far, and preview cards usually have no TCGPlayer price yet, so `market` mode totals leave them out and tell you how many it skipped.

---

### They work together

All the options combine, which is where it gets fun:

> *"Build me a Princess deck from cards I own, including Hyperia City."*

```
build_deck(theme="princess", mode="collection",
           collection_csv="…/enriched_collection.csv",
           include_preview=True)
```

Themed, auto-colored, limited to what's in my binder, with the new set included. `rotation_safe=True` still works on top of that, and preview cards count as rotation-safe because an upcoming set is by definition in the newest group.

---

### What it still isn't

`build_deck` is still a heuristic: curve, stat efficiency, keyword value, and now theme. It doesn't understand combos. It also still trusts card data that has known holes; a few songs show up in the API with only their "sing this for free" reminder text and none of their actual effect, and the builder will happily score them blind. Treat what it gives you as a strong first draft for a theme, not a tournament list.

But the first draft now starts from the question a player actually asks.

### Try it

Upgrade and restart your MCP client:

```
pip install -U lorcana-mcp
```

Then just ask for the deck you want: a Princess deck, a Detective deck, a Hyperia City brew.

- [v2.5.0 release](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.5.0) · [PyPI](https://pypi.org/project/lorcana-mcp/) · [MCP Registry](https://registry.modelcontextprotocol.io/?q=lorcana)
- Setup for Claude Desktop: [Add lorcana-mcp to Claude Desktop](/blog/add-lorcana-mcp-to-claude-desktop/)
- Earlier: [Part 8 — The Snapshot Problem](/blog/lorcana-mcp-part-8-the-snapshot-problem/) · [Part 1 — Why I Built It](/blog/lorcana-mcp-part-1-why-i-built-it/)
