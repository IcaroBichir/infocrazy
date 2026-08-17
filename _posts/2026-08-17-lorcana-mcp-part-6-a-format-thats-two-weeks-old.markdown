---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 6: A Deck Builder for a Format That's Two Weeks Old"
modified:
categories: blog
excerpt: "Ravensburger shipped a brand-new multiplayer singleton format while this project was already running. Teaching the MCP server to build for it meant fetching data that didn't exist a month ago, finding a scoring bug that quietly dropped the headline card from its own deck, and fixing a UX mistake I only caught because I watched someone else hit it live."
tags: [mcp, lorcana, python, claude, ai, tcg, deckbuilding]
comments: true
date: 2026-08-17T15:50:00-04:00
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

In [part five](/blog/lorcana-mcp-part-5-just-say-a-card/) the story was about what "the MCP does it" actually means for a deck I asked for. This post is about a deck I asked for that the tool *couldn't* build at all — because the format didn't exist yet.

---

### "Do you have the ability to build a coconut deck?"

That was the actual question. My first instinct was that it was a typo or a joke — Lorcana doesn't have a "coconut" ink color, and nothing in this project's reference file mentioned one. I said as much and offered to build whatever format was actually meant instead.

Turns out it wasn't a typo. **Format Coconut** is a real thing: Ravensburger opened a public beta on 2026-07-28 for a new Commander-style multiplayer format — 18 digital-only "Coconut" cards (three per ink), one chosen per deck as an always-on passive ability that never gets played, up to three ink colors instead of two, singleton deckbuilding (one copy of everything except the Coconut's own associated character, which can run up to four), and a race to 25 lore instead of 20. It had been live for about three weeks when I was asked to look into it, and I'd never heard of it — reasonably, since it postdates whatever data trained me.

So the actual ask became: research an entire format from scratch, then teach the MCP server to build decks for it.

---

### Finding an actual data source

Official rules PDFs and press coverage got me the shape of the format, but not the 18 cards' exact ability text — the kind of thing you don't want to transcribe from a screenshot into a tool real people will build decks from. That's what happened once already in this project ([part four](/blog/lorcana-mcp-part-4-the-fix-that-broke-things/) — trusting a browser display instead of a real export, and silently corrupting an import).

LorcanaJSON, the same structured card-data project this whole server already depends on, had already added it:

```
https://lorcanajson.org/files/current/en/formatCoconutCards.json
```

Same schema as the regular card file, missing a few fields that don't apply, and each entry carries an `associatedCardName` — the real, already-printed character card the Coconut lets you run four copies of. That field is what turns "pick a Coconut" into something a deck builder can actually act on instead of just displaying as flavor text.

```python
def fetch_format_coconut_cards() -> list[dict]:
    cached = _cache.get("format_coconut_cards")
    if cached is not None:
        return cached
    data = json.loads(_fetch(
        "https://lorcanajson.org/files/current/en/formatCoconutCards.json"
    ))
    cards = data["cards"]
    _cache.set("format_coconut_cards", cards)
    return cards
```

Same 24-hour cache every other fetch in this server already uses. Beta data, though — LorcanaJSON says so directly in its own docs: this file can change shape or disappear if Ravensburger ends the beta. Worth remembering next time something about it looks wrong.

---

### Wiring it into a builder that assumed 4-of-everything

`build_deck` had one hard assumption baked in from day one: every card in a legal deck can run up to four copies, capped only by what you own in collection mode. Format Coconut breaks that outright — singleton, with exactly one named exception per deck. That meant a new `max_copies` rule threaded through the whole allocation path:

```python
def _max_copies(card: dict) -> int:
    cap = 1 if (fmt == "coconut" and card.get("fullName") != coconut_associated_name) else 4
    if mode == "collection":
        cap = min(cap, owned_counts.get((card.get("fullName") or "").lower(), 0))
    return cap
```

Ink-color legality got looser, not stricter, for this format — the official rules say every released card is fair game, no rotation, no banned list yet. So instead of the usual duels.ink legality lookup, Coconut just short-circuits to true:

```python
if fmt == "coconut":
    return True
```

That part went smoothly. The part that didn't was the scoring.

---

### The bug: the Coconut's own card didn't show up in its own deck

First real test build — Stitch - Rock Star, an Amber Coconut whose ability plays a cheap character for free. I built an Amber/Ruby deck around it and checked the output for how many copies of Stitch it actually included.

Zero.

Stitch - Rock Star costs 6 ink. The existing scoring model rewards stat-efficiency-per-cost, which structurally favors cheap cards — a 1-cost character with decent stats posts a much better ratio than a 6-cost one ever can, bonus or no bonus. I'd added a synergy bonus for the associated character specifically, the same pattern this project already uses for Shift-family payoffs, and it still lost. The bonus nudges a ranking; it doesn't override an allocator that ranks cheap cards higher everywhere else in the pool by design.

The fix borrowed a pattern already sitting in this codebase for a near-identical problem — Duo Shift cards, which need *two* specific named cards to actually function and can't just hope the scoring bonus finds them both. That existing code trims the weakest current pick to force a guaranteed enabler in. I wrote the same trim-to-force-it logic for the Coconut's own character:

```python
def ensure_coconut_associated_card(picks, pool, associated_full_name, max_copies_fn):
    ...
    want = max(0, max_copies_fn(candidate))
    remaining = want
    while remaining > 0:
        weakest_fn = min(
            (fn for fn in order if picks_map[fn][1] > 0),
            key=lambda fn: score_card(picks_map[fn][0]),
            default=None,
        )
        if weakest_fn is None:
            break
        trim = min(remaining, picks_map[weakest_fn][1])
        picks_map[weakest_fn][1] -= trim
        ...
```

Re-ran the same build. Four copies of Stitch - Rock Star, everything else still singleton, still 60 cards. The lesson isn't new to this project, just reapplied: a scoring bonus is a suggestion, not a guarantee, and the two need to stay separate on purpose.

---

### The UX mistake I only found because someone else hit it

Shipped the feature, explained how it worked, offered an example call. The very next message back was someone actually trying it:

```
build_deck(format="coconut")
```

No ink colors — reasonable, since you don't need to know your colors yet if you're still choosing which Coconut to build around. It crashed. `ink_colors` was a required argument with no default, because every other format in this tool has always needed one up front. Format Coconut is the first one where that's not true the moment you're just browsing options.

```
TypeError: build_deck() missing 1 required positional argument: 'ink_colors'
```

I'd built the "list all 18 Coconuts" fallback correctly, then guarded it behind a parameter I'd forgotten to make optional for exactly that path. Made `ink_colors` default to an empty string, and split the validation: browsing mode skips the "you must provide a color" check entirely; building mode (a Coconut actually chosen) still enforces it, same as every format before it.

That fix also opened the door to something better than "list all 18" every time. Pass a color while browsing and the listing filters to just that ink instead of dumping all eighteen:

```
build_deck("Amber", format="coconut")
```

```
**Amber**
- Ariel - Spectacular Singer — Whenever a Princess character of yours
  sings a song, gain lore equal to her ◊. (Focus: Princess + Song lore burst)
- Pocahontas - Peacekeeper — ...
- Stitch - Rock Star — ... (Focus: cheap-character swarm/flood)
```

That "Focus" tag isn't generated from the same synergy data the scoring engine uses — I tried auto-deriving readable phrasing from the mechanical tags first, and it read like a spec sheet ("cards named Moana, cards named Heihei, cards named Pua"). Hand-writing one short phrase per Coconut, decoupled from the scoring code entirely, just reads better. Eighteen cards is few enough that "hand-write it" beats "derive it cleverly" without much of a contest.

---

### Building a real deck surfaced one more gap

Picked Robin Hood - Sneaky Sleuth — Emerald, deals 1 damage to something whenever you play a character named Robin Hood — paired it Emerald/Ruby/Steel, and built. The output looked complete: 60 cards, singleton respected, four copies of the Coconut itself. But the Coconut's trigger doesn't care *which* Robin Hood you play, and the build only had one.

Checking the full card pool for every character literally named "Robin Hood" turned up six more, legal in that same three-color pool, none of which the heuristic had picked. One of them, **Robin Hood - Champion of Sherwood**, Shifts onto *any* character named Robin Hood — not itself specifically, any of them — which means playing it also fires the Coconut's own damage trigger a second time in the same turn, on top of the Shift discount. That's exactly the kind of thing this project's own disclaimer already warns about on every `build_deck` result: it optimizes curve and stat efficiency, not "this card's whole reason for existing is a name match six other cards also happen to share."

Swapping those six in — funded by cutting the least thematic filler the heuristic had chosen — actually dropped the price too, from $38.04 to $32.69, since two of the cuts were an unusually expensive Enchanted-tier pull and a chase common. Same 60 cards, more on-theme, cheaper. Wrote the whole thing up as a deck file the same way every other deck in this project gets tracked, including a rules ambiguity I didn't want to paper over: the Coconut's free starting item, Robin's Bow, is granted "from your collection" — phrasing that doesn't match how this game normally refers to zones, and reads a lot like a Commander-style fetch from outside the deck entirely. Flagged it instead of guessing.

---

### Shipping it

Same four-step release this project always does, documented once and followed exactly since — commit and push to `main`, tag and cut a GitHub release, build and upload to PyPI, then the MCP Registry:

```
Publishing to https://registry.modelcontextprotocol.io...
Error: publish failed: server returned status 401:
{"detail":"Invalid or expired Registry JWT token"}
```

Third time now. The registry's login token expires on its own schedule, unrelated to how often this project actually ships — [part four](/blog/lorcana-mcp-part-4-the-fix-that-broke-things/) hit this once, the 1.0.0 release hit it again, and 2.0.0 makes three. Same fix every time: a GitHub device-code login, a code to type into a browser, a short wait.

```
To authenticate, please:
1. Go to: https://github.com/login/device
2. Enter code: 2BFB-E771
3. Authorize this application
Waiting for authorization...
Successfully authenticated!
```

`lorcana-mcp` is v2.0.0 now — a major bump on purpose, since it changes what a legal deck even looks like for this one new format value, not a patch to existing behavior. 310 tests, up from 278, all still passing, all still network-free.

- [PyPI](https://pypi.org/project/lorcana-mcp/2.0.0/)
- [GitHub release](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.0.0)
- [MCP Registry](https://registry.modelcontextprotocol.io/?q=lorcana)

```bash
pip install lorcana-mcp
claude mcp add lorcana -- lorcana-mcp serve
```

```
build_deck(format="coconut")
```

---

The pattern across six of these posts now is less "I built a clever thing" and more "I kept finding the places where a reasonable-looking shortcut quietly wasn't one" — a promo mapping that displayed right and imported wrong, a heuristic that finds curve but not combos, a scoring bonus that nudges but doesn't guarantee, a required argument that stopped making sense the moment the format itself allowed browsing before building. None of those are exotic bugs. They're all the same shape: something that worked in the case I tested and quietly didn't in the case I hadn't. Format Coconut just gave me four new chances to find one in a single week, because it's new enough that there's no accumulated folklore to lean on yet — official rules, one JSON file, and whatever a real build actually does when you run it.
