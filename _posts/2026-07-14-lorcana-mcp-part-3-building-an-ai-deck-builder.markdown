---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 3: Building an AI Deck Builder"
modified:
categories: tech
excerpt: "A heuristic deck-building engine, two bugs that made 60-card decks structurally impossible to reach, and a perf bug that turned a sub-second tool call into a 13-second one. Then: shipping it."
tags: [mcp, lorcana, python, claude, ai, tcg, algorithms]
comments: true
date: 2026-07-14T09:00:00-04:00
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

In [part two](/tech/lorcana-mcp-part-2-the-build/) I covered the enrichment pipeline and the fuzzy card-matching engine. This post is about the feature that didn't exist a week ago: `build_deck`, a tool that actually assembles a legal decklist instead of just looking one up.

---

### From a conversation habit to a real tool

Before this existed, "build me a deck" was something I did by hand in every conversation: ask which of three modes the person wanted — build only from what they own, build the ideal deck and price the gap, or ignore ownership entirely and build the theoretical best — then manually assemble it with `search_cards` and cross-reference `what_am_i_missing`.

That workflow was solid enough that I'd documented it as a standing instruction. The moment that stopped being good enough was realizing it only worked *in this conversation* — anyone else running the published package didn't get any of it. So it became a real tool: `build_deck(ink_colors, mode, format, collection_csv, rotation_safe)`, three modes built in instead of asked about every time.

---

### What it actually does

Given an ink pair and a format, `build_deck` fetches the live card pool, filters it down to what's legal, and assembles a curve-balanced ~60-card deck:

- **`collection`** — only cards you own, capped at your actual quantity. If you can't reach 60, it says so honestly instead of padding with cards that don't belong.
- **`ideal`** — the best deck regardless of ownership, then (if you hand it a collection CSV) a diff of what you already have versus what you'd need to buy, priced live off tcgcsv.com.
- **`market`** — the best deck, fully priced, ownership ignored entirely.

It's explicitly not a synergy engine — it optimizes ink curve and per-card stat efficiency, not multi-card combos. Every result it returns says so, by name, pointing at specific combos it can't detect (the Merlin/Mim bounce loop, the Steelsong package). A tool that quietly overclaims is worse than one that's upfront about its ceiling.

---

### The scoring and allocation engine

Each candidate card gets scored by type. Characters weigh stat efficiency (strength + willpower + lore, with lore weighted double since it's the actual win condition) plus a keyword bonus — Ward and Evasive score highest, matching this project's own documented rules on which keywords carry the most defensive/lore-securing value. Actions and songs get scored off ability-text pattern matches (`"banish"`, `"draw a card"`, `"deal"`) since there's no structured "this is removal" field to key off of.

The curve targets themselves come from scaling this repo's own documented ink-curve guideline — 1–2 cost: 8–12 cards, 3–4: 12–16, 5–6: 8–12, 7+: 2–6 — proportionally to whatever deck size is being built, using a largest-remainder apportionment so the buckets always sum to exactly the target instead of drifting off by rounding error:

```python
def curve_targets(total: int = 60) -> dict[str, int]:
    base_sum = sum(_CURVE_MIDPOINTS.values())
    raw = {b: v * total / base_sum for b, v in _CURVE_MIDPOINTS.items()}
    floored = {b: int(v) for b, v in raw.items()}
    remainder = total - sum(floored.values())
    for b in sorted(raw, key=lambda b: raw[b] - floored[b], reverse=True)[:remainder]:
        floored[b] += 1
    return floored
```

At `total=60` that resolves to `{"1-2": 16, "3-4": 22, "5-6": 16, "7+": 6}` — same bell-curve shape, exact sum, every time.

---

### Bug one: ink filtering needs a different rule for building than for searching

The existing card search already filtered by color — but it uses *any-match* semantics on purpose, so a Ruby/Emerald dual-ink card surfaces when you search either color. That's correct for search. It's wrong for deck assembly: a Ruby/Emerald card has no business in a Ruby/Amber deck. Deck building needed *subset* semantics — every color on the card has to fit inside the chosen pair — which meant writing a second, stricter filter rather than reusing the search one:

```python
def _color_subset_ok(card: dict, allowed: set[str]) -> bool:
    card_colors = {c.lower() for c in _card_colors(card)}
    return bool(card_colors) and card_colors <= allowed
```

Small function, but it's the difference between a legal deck and one that fails at the table.

---

### Bug two: the type caps summed to less than a deck

I capped card-type composition to this project's own documented guideline ranges — at most 24 Characters, 16 Actions, 8 Items, 4 Locations. Add those up: 52. A legal Core Constructed deck is 60 cards minimum. I'd built a hard ceiling that made a full deck *structurally impossible* the moment those caps actually bound, which showed up immediately in testing: a pool made entirely of Characters capped out at 24 cards and stopped, 36 short.

The fix was making the type caps apply only during the first allocation pass — the one that shapes composition when the pool is diverse enough to allow it — and letting an uncapped backfill pass make up the rest by score alone. Composition guidance stays a preference, not a wall:

```python
if grand_total < total:
    for card in ranked:
        if grand_total >= total:
            break
        try_take(card, min(4, total - grand_total), enforce_type_cap=False)
```

---

### Bug three: a 13-second tool call that should've taken under one

This one I only found because two brand-new tests were suspiciously slow. `build_deck`'s stats section originally ran its own output back through the existing deck-analysis tool, which re-resolves every card by fuzzy name matching against the *entire* live card pool — a real network fetch plus a fuzzy-match pass, for cards whose exact identity I already had sitting in a variable three lines earlier. Redundant by construction.

Swapping that for a function that reads the stats straight off the already-resolved cards cut a synthetic full deck build from about 13 seconds to well under one, on a warm cache. The lesson wasn't really about performance — it was that reaching for an existing tool isn't always the same as reusing it correctly. Sometimes the existing tool solves a different problem than the one you actually have.

---

### Verifying it for real

Before calling it done, I ran all three modes against my actual collection CSV — a real 60-card Amber/Sapphire build entirely from owned cards, an ideal build priced at $31.32 to complete, a full market build, a six-color Infinity build, and a `rotation_safe=True` run that correctly restricted itself to the current safe rotation group without me hardcoding which sets that meant. That last part matters beyond convenience — the safe group shifts every time a new set goes live, so the tool computes it from live rotation metadata instead of a number I'd have to remember to update.

258 tests passing, up from 205 before this feature.

---

### Shipping it

Once it worked, shipping it was the boring-in-a-good-way part: `git push`, build the wheel, `twine upload` to PyPI. The one genuinely interesting step was the MCP Registry — publishing there means proving you control the GitHub identity your server claims to be, via a device-code flow:

```
To authenticate, please:
1. Go to: https://github.com/login/device
2. Enter code: 6146-EF27
3. Authorize this application
Waiting for authorization...
Successfully authenticated!
```

Approve that in a browser, and `mcp-publisher publish` pushes the update live.

`lorcana-mcp` is v0.2.0 now, listed on the [MCP Registry](https://registry.modelcontextprotocol.io/servers/io.github.IcaroBichir/lorcana) and installable from PyPI:

```bash
pip install lorcana-mcp
claude mcp add lorcana -- lorcana-mcp serve
```

If you collect Lorcana, point it at your TCGPlayer export and ask it to build you something. If you don't, the more general lesson traveled across all three of these posts: give the model real data instead of retyping it every conversation, and keep the tool honest about what it can't do.
