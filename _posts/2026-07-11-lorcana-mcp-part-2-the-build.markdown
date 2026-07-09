---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 2: The Build"
modified:
categories: tech
excerpt: "A CSV enrichment pipeline, a fuzzy card-matching engine, and what happened when I audited two competing open-source Lorcana MCP servers instead of building in a vacuum."
tags: [mcp, lorcana, python, claude, ai, tcg, testing]
comments: true
date: 2026-07-11T09:00:00-04:00
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

In [part one](/tech/lorcana-mcp-part-1-why-i-built-it/) I covered why I built this and why it needed no OAuth at all. This post is about what's actually inside.

The full source is at [github.com/IcaroBichir/lorcana-mcp](https://github.com/IcaroBichir/lorcana-mcp). `pip install lorcana-mcp` gets you running in about a minute. This post is for anyone who wants to see how it works.

---

### Stack and structure

Plain Python, `mcp[cli]`'s `FastMCP` for tool registration, `urllib` for HTTP (no async needed — these are quick lookups, not streaming). Five files, each with one job:

```
lorcana_mcp/
├── api.py          # Card pool fetching, filtering, fuzzy resolution, pricing
├── deck.py         # Pure deck-list parsing and analysis (no network)
├── deckbuilder.py  # Deck synthesis — candidate pools, scoring, allocation
├── cache.py        # 24h file-based JSON cache
└── server.py       # MCP tool definitions (the I/O layer)
```

The split between `api.py`/`deck.py`/`deckbuilder.py` (pure logic, no I/O) and `server.py` (fetches data, formats markdown, talks to Claude) is deliberate. Every pure function is trivially testable without mocking a network call — 258 tests run in under half a second because almost none of them touch a socket.

---

### The enrichment pipeline

The core problem: TCGPlayer's collection export has your quantities and prices, but zero gameplay data. No ink cost, no keywords, no ability text — just names, set names, and card numbers.

`enrich_csv` merges that export against two community card databases — LorcanaJSON for recent sets, lorcana-api.com for anything before it — matched by set name and card number, and writes back ten new columns: Ink, Ink Cost, Card Type, Subtypes, Strength, Willpower, Lore Points, Inkable, Keywords, and Abilities. It also writes a second file, ready to import straight into dreamborn.ink's deck builder.

The two ugliest bugs in this pipeline were both ID-matching mistakes, which is the kind of bug that fails silently instead of loudly:

- **Wrong price-matching column.** TCGPlayer's export has two ID columns — `Product ID` and `TCGplayer Id` — that look interchangeable and aren't. `TCGplayer Id` is a secondary listing ID that doesn't correspond to anything else. `Product ID` is the one that matches tcgcsv.com's live pricing feed and LorcanaJSON's `externalLinks.tcgPlayerId`. Matching on the wrong one doesn't error — it just silently refreshes 0 of 391 rows.
- **A rejected User-Agent.** tcgcsv.com returns a 401 for Python's default `urllib` User-Agent string. Every live price lookup in the tool was quietly failing until I added a real UA header.

Neither of these crashed anything. Both just returned wrong or empty data, which is worse — you don't get an error to chase, you get a plausible-looking answer that's quietly incomplete.

---

### Fuzzy card resolution

Card names in casual conversation almost never match the database exactly. "Goofy musketeer" needs to become "Goofy - Musketeer." "Elsa" needs to return every Elsa printed, ranked by how recent the set is. Typos need to survive.

`resolve_card` tokenizes the query and scores it against each card's name (weighted higher) and subtitle, tolerating missing dashes, missing subtitles, word-order shuffles, and typos via a fuzzy-ratio comparison on longer tokens:

```python
def score_candidates(query: str, set_name: str = "") -> list[tuple[float, dict]]:
    query_tokens = _tokenize(query)
    scored = []
    for card in fetch_lorcana_json():
        score = _card_score(query_tokens, card)
        if score > 0:
            scored.append((score, card))
    return sorted(scored, key=lambda sc: (sc[0], _set_recency(sc[1])), reverse=True)
```

The result is classified three ways: a single confident match, a ranked top-3 when the query is genuinely ambiguous, or "not found." Every other name-based tool in the server — `lookup_card`, `analyze_deck`'s unresolved-card detection, `find_song_synergies`' song lookup — rides on this same scorer, so improving it once improved everything downstream for free.

---

### Testing without a network

Every fetch function (`fetch_lorcana_json`, `fetch_duels_ink`, `fetch_tcgcsv_prices`) is a thin, mockable seam. Tests patch the function directly rather than the HTTP layer underneath it:

```python
with patch("lorcana_mcp.api.fetch_lorcana_json", return_value=_FUZZY_CARDS):
    result = resolve_card("elsa")
```

This mattered more than I expected while building the deck-synthesis feature in part 3 — I found a real perf bug specifically *because* a couple of new tests were slow, which turned out to mean they were silently hitting the real network instead of a mock. More on that next post.

---

### Auditing the competition instead of guessing

Before building anything new, I looked at two other open-source Lorcana MCP servers — [danielenricocahall/lorcana-mcp](https://github.com/danielenricocahall/lorcana-mcp) (Python, Lorcast-backed, Docker-distributed) and [gregario/lorcana-oracle](https://github.com/gregario/lorcana-oracle) (TypeScript, bundled offline card data) — and did a real feature diff instead of assuming I already had the best tool.

A few things they had that I didn't:

- **`character_versions`** — every printing of a character across sets, useful for Shift-strategy decks. Still on my TODO.
- **`browse_franchise`** — cards by Disney franchise, which maps directly onto Wilds Unknown's Toy/Super tribal synergy mechanic. Also still open.
- **`import_deck`/fuzzy deck-list resolution** — turned out I already had this; `analyze_deck` and `what_am_i_missing` were already running every line through the same fuzzy resolver from `resolve_card`. Good to confirm, not a gap.

And a few things neither of them had at all: the TCGPlayer enrichment pipeline itself, dreamborn.ink export, a `what_am_i_missing` completion-cost calculator, and — as of the next post — an actual deck-*building* tool instead of just deck lookup and analysis.

I wrote all of this up as a public `TODO.md` in the repo rather than keeping it in my head. Half the value of comparing against other projects is having a paper trail for what to build next.

---

### Running it

```bash
pip install lorcana-mcp
claude mcp add lorcana -- lorcana-mcp serve
```

Then just talk to it:

> "Enrich my collection at ~/Downloads/export.csv"

No auth step, no config file. Restart Claude after adding it, and the server starts on demand over stdio, same as every other MCP server in this series.

---

Next up: [teaching it to actually build decks](/tech/lorcana-mcp-part-3-building-an-ai-deck-builder/) — and the two real bugs it took to get a heuristic deck builder to reliably hit 60 cards.
