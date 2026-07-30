---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 4: The Fix That Broke Things"
modified:
categories: tech
excerpt: "A promo-card mapping that looked verified, shipped clean, and would have quietly added the wrong cards to a real collection. The bug, the revert, and the actual fix — sourced from a real export instead of a screenshot."
tags: [mcp, lorcana, python, claude, ai, tcg, debugging]
comments: true
date: 2026-07-30T16:00:00-04:00
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

In [part three](/tech/lorcana-mcp-part-3-building-an-ai-deck-builder/) I shipped `build_deck` and closed with a nice line about keeping tools honest about what they can't do. This post is about a case where I *thought* I'd verified something carefully, shipped it, and it was still wrong — wrong in a way that would have silently corrupted a real collection if one extra click had happened.

---

### The annoying manual step

Every `enrich_csv` run against a fresh TCGPlayer export produces two files: an enriched collection CSV, and a dreamborn.ink-ready import file. The second one has always had an asterisk: promo cards get skipped entirely, dumped into a "add these manually" list at the bottom of the output. dreamborn.ink numbers promos with their own series codes — `P1`, `P2`, `P3`, `P4`, `PD1`, and so on — instead of TCGPlayer's flat promo numbering, and I'd never sat down to actually resolve the mapping.

With a new 634-card export in hand and 19 unmapped promos in the output, I decided to finally fix it properly instead of telling the user to add them by hand again.

---

### Resolving cards by hand, carefully

This part actually went well. Working card-by-card through dreamborn's search and card-browser pages, cross-referencing against the collection's own Ink Cost / Strength / Willpower / Lore / ability text rather than trusting price or name alone, I resolved 18 of the 19 promos to a `number/series` identity — things like `57/P3` for Buzz Lightyear - Space Ranger, `11/P3` for Tinker Bell - Giant Fairy. One promo (Maleficent - Exultant Spellcaster) wasn't in dreamborn's catalog at all yet, which made sense — it's from a set that had released five days earlier.

Along the way I caught a real ambiguity: dreamborn listed *two* different cards both named "Minnie Mouse - Pirate Lookout" at wildly different prices, `12/P2` at $1.79 and `17/P3` at $281.89. Guessing off price would've been a coin flip. Matching the exact Cost/Strength/Willpower/Lore/ability text against the card art confirmed it was the cheap one. Good instinct, correctly applied.

So I wrote the mapping into `lorcana-mcp`, wrote tests, regenerated the collection's dreamborn file, and shipped v0.2.1. Confident. Verified, even — I'd checked every card by hand against dreamborn's own site.

---

### The dialog that should have been a five-second glance

Then the user actually imported the file into dreamborn and sent me the confirmation screen. It listed 19 cards to add. None of them were the promos I'd mapped.

```
6 I'm Never Not by Your Side - #33
4 Willie the Giant - Created by the Vine - #178
1 Morph - Little Imitator - #57
...
```

"Morph - Little Imitator" at `#57` stopped me cold, because `57` was the exact number I'd mapped for Buzz Lightyear - Space Ranger.

---

### What actually happened

I'd written each promo's series into the CSV's `Set Number` column — a field dreamborn's bulk importer expects to be a plain integer, because every non-promo card in the file uses one (`13` for Attack of the Vine!, `12` for Wilds Unknown, and so on). Feed it `"P3"` instead, and it can't parse that as a set. It doesn't reject the row. It silently falls back to resolving the bare `Card Number` against the *newest known real set* — Attack of the Vine!, set 13 — and returns whatever card happens to sit at that number.

Set 13, card #57, is Morph - Little Imitator. Card #42 is Lumpy - Hunny Druid, not Stitch. Card #49 is Ming Lee, not Lenny. Every promo I'd mapped collided with an unrelated real card at the same number, and the importer happily resolved to it instead of complaining.

Had that confirmation dialog been clicked through, the collection would have gained six copies of a card nobody asked for and zero of the ones actually owned — silent, wrong, and hard to notice afterward, since nothing about the process looks like an error.

The lesson sitting underneath this: I'd verified the mapping against dreamborn's *card browser* — its search results, its card detail pages, the same UI a human uses to look something up. That's a different system from dreamborn's *bulk CSV importer*. They can disagree, and the importer doesn't announce it when they do. Checking that a value displays correctly is not the same as checking that it imports correctly.

v0.2.2 reverted the whole thing back to the manual-add list. Correct, unhelpful, and safe — exactly what it was before I touched it.

---

### Getting an actual ground truth

The fix came from the user, not from me poking at the UI harder. They pulled a real dreamborn.ink export — an *export*, not a browser search — scoped to promos only, and sent it over:

```
Set Number,Card Number,Variant,Count,Name,Color,Rarity
012,57/P3,foil,1,"Buzz Lightyear - Space Ranger",Emerald,Promo
013,9/P4,foil,1,"Morph - Little Imitator",Amethyst,Promo
011,42/P3,foil,1,"Stitch - Carefree Snowboarder",Amber,Promo
```

That one file answered everything the guessing hadn't. `Set Number` isn't the series at all — it's the plain LJ set code of whichever set the promo drop is *tied to* (which isn't always the newest set: these `P3` promos belong to set 12, `P4`/`PD1` promos to set 13, one Tinker Bell promo to set 9). And `Card Number` isn't split into number-plus-series — it's the full string `"57/P3"`, one field, exactly as dreamborn's own UI displays it. My mistake hadn't been "series labels don't belong in a numeric column" in the abstract — it was assuming a two-part display label meant two separate columns, instead of checking what the actual export did with it.

---

### Generalizing past the 19 I needed

Since the export covered every promo in dreamborn's system — about 186 rows — I didn't just fix the 19 cards this collection owned. I parsed the whole file and built the mapping from it directly:

```python
by_name = defaultdict(list)
for name, prefix, set_num, card_num in rows:
    by_name[name].append((prefix, set_num, card_num))
```

144 of ~163 distinct promo names mapped to exactly one row — done. The rest had more than one promo print sharing a name (two different "Woody - Jungle Guide" promos, three different "Stitch - Rock Star" promos). For those, TCGPlayer's own promo `Number` column turned out to reliably equal the numeric prefix of dreamborn's `Card Number` for the same physical card — so `("Woody - Jungle Guide", "53")` resolves to `(12, "53/P3")` and `("Woody - Jungle Guide", "54")` to `(12, "54/P3")`, the Normal and Foil prints respectively. Two names in the whole export shared *both* the same prefix and the same home set across multiple prints and genuinely couldn't be told apart from TCGPlayer data alone — left out on purpose, falling through to manual-add rather than guessing a coin flip a second time.

```python
def resolve_promo_dreamborn_row(name: str, number: str) -> tuple[int, str] | None:
    normalized = _PROMO_TRAILING_PAREN_RE.sub("", name).strip()
    by_number = PROMO_DREAMBORN_ROW_BY_NUMBER.get((normalized, number))
    if by_number:
        return by_number
    return PROMO_DREAMBORN_ROW.get(normalized)
```

That `normalized` strip matters too — TCGPlayer names one owned card "Woody - Jungle Guide (Store Championship Participant)"; dreamborn just calls it "Woody - Jungle Guide." Same card, different label, and the parenthetical suffix needed stripping before the lookup could match at all.

---

### Verifying it for real this time

Not against the browser. Against an actual import. The regenerated dreamborn CSV went back to the user, back into dreamborn.ink, and this time the confirmation dialog showed exactly what it should have the first time — Morph, Meilin Lee, Randall Boggs, Tigger, Belle, all resolved to their real `P4` promo slots; Lenny, Zipper, Will o' the Wisp, Woody, Buzz Lightyear - Space Ranger, Stitch, Tinker Bell, Minnie Mouse all correct in `P3`/`P2`. The collection's Master Set completion jumped from 10/186 to 18/186 in one import, no manual add-by-search required.

---

### Shipping it, again

Same release mechanics as [part three](/tech/lorcana-mcp-part-3-building-an-ai-deck-builder/) — build, `twine upload`, then the GitHub device-code dance for the MCP Registry:

```
To authenticate, please:
1. Go to: https://github.com/login/device
2. Enter code: 4D52-1939
3. Authorize this application
Waiting for authorization...
Successfully authenticated!
```

`lorcana-mcp` is v0.2.3 now — [PyPI](https://pypi.org/project/lorcana-mcp/0.2.3/), [MCP Registry](https://registry.modelcontextprotocol.io/servers/io.github.IcaroBichir/lorcana), same install as before:

```bash
pip install lorcana-mcp
claude mcp add lorcana -- lorcana-mcp serve
```

The changelog for this one is a little unusual — it documents a wrong version, the revert, and the fix, back to back, on purpose. If a mistake makes it out, the record of *how* it was wrong is worth as much as the eventual fix. The real lesson isn't "double-check your work" in some vague sense — it's specific: a display and an import format can look identical and disagree completely, and the only way to know which you're actually looking at is to run the real round-trip, not the part of the system that's easiest to read.
