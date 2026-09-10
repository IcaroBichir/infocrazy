---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 8: The Snapshot Problem"
modified:
categories: blog
excerpt: "It started as \"here are some tournament results, add them.\" It turned into a release and a half, because every part of the tool that hands back a confident answer from data I bundled at build time was quietly out of date — and none of them said so. A mislabel that survived three passes, six different Milo Thatches, decklists that didn't add up to 60, and a card database one failed fetch from useless."
tags: [mcp, lorcana, python, claude, ai, tcg, metagame]
comments: true
date: 2026-09-10T17:30:00-04:00
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

The ask was small: *"Here are the Top 8 decklists from the Asia Championship. Update the MCP and our notes."*

I have a tool for exactly this. `get_meta` returns a hand-maintained snapshot of the Core Constructed metagame — every two-ink pair's tier, rough share, playstyle, and a summary of recent tournament results. I added it a few releases back specifically so "what's strong right now" didn't depend on me remembering. So this was supposed to be a five-minute edit to one Python file.

It was two releases (`v2.2.0` and `v2.3.0`), a new tool, and a bundled fallback database. Because once I started actually cross-checking the tournament data against the real decklists, everything I'd bundled at build time turned out to be a little bit wrong, or a little bit stale, or a little bit missing — and none of it said so.

---

### The mislabel that survived three passes

The Asia Championship went to Michele Carretta on Amethyst/Emerald — a discard-matters control deck I didn't have an entry for at all. Fine, new archetype, add a row. The runner-up and six of the Top 8 all ran the same Amber/Amethyst midrange shell.

Which is when I noticed my snapshot said that shell won the North American Championship two weeks earlier as **Amber/Sapphire**.

I'd written that. It's in three different places — the tier list, the tournament summary, an old decklist file. And it's wrong. Dillon LeDuc's winning 60 has *zero Sapphire cards in it*. Rafiki, both Luisa Madrigals, Cheshire Cat - Inexplicable, Isis Vanderchill, Demona, Tigger, Sven, Dumbo — every one of those is Amethyst. There is no Sapphire card anywhere in the list. It's Amber/Amethyst, and it was Amber/Amethyst when I first wrote it up.

The mislabel survived because at no point had I put the winning decklist next to the label and read down it. I'd been working from archetype names and screenshots of card grids. "Amber/Sapphire" *sounds* like a ramp-y midrange deck; the mistake was self-consistent enough that nothing flagged it. It took a fourth event running the same shell before I bothered to check the cards, and the cards said something different than the label had for a month.

The fix in the file is one line. The lesson isn't. A tier list is a summary of decklists; if nobody ever checks it *against* the decklists, it can drift into confidently describing a deck that doesn't exist.

---

### Which Milo Thatch?

The Asia champion's discard deck runs a card called **Milo Thatch**. My first instinct was `lookup_card "Milo Thatch"`, which returned a card, which had abilities that roughly fit, so I moved on.

Then a second deck ran a Milo Thatch, and the stats I could half-read off the screenshot didn't match the card I'd looked up. So I checked properly, and there are **six** different cards named Milo Thatch across the sets. Three are legal in the current English Core rotation; three are only legal in Infinity or the Japanese rotation. The one the decks were actually running — *Getting His Hands Dirty*, a Set 12 body — is not the one `lookup_card` had handed me.

`lookup_card` isn't broken. Its documented behavior is: on a name that matches multiple printings, silently return the most recent one. For "give me the stats for Elsa - Spirit of Winter" that's the right call — the printings are gameplay-identical, you just want the numbers. But "which printing is legal in Core" and "which printing is cheapest to buy" and "did this card ever get an Enchanted" are real questions, and the answer to all of them is *a table*, not one row.

So `v2.3.0` adds `list_printings`:

```
list_printings("Milo Thatch", fmt="core")

| Card                                | Set           | Printings          | Cost | S/W/L  | Legal in                | Core EN? | Cheapest |
|-------------------------------------|---------------|--------------------|------|--------|-------------------------|----------|----------|
| Milo Thatch - Courageous Explorer   | Wilds Unknown | #108 Common        | 3    | 4/3/1  | Core EN, Infinity, ...  | ✓        | $0.04    |
| Milo Thatch - Getting His Hands Dirty | Wilds Unknown | #82 Super Rare, #230 Enchanted | 7 | 5/5/3 | Core EN, Infinity, ...  | ✓        | $4.03    |
| Milo Thatch - King of Atlantis      | Into the Inklands | #80 Legendary  | 7    | 4/4/3  | Infinity, Core JA       | ✗        | $0.50    |
| Milo Thatch - Undaunted Scholar     | Archazia's Island | #145 Rare      | 2    | 2/2/1  | Infinity, Core JA       | ✗        | $0.12    |
| ...                                                                                                                            |
```

Given a full name it shows that one card's base plus its Enchanted and Epic printings; given a bare character name it shows every card that shares it. Pass a format and it adds the ✓/✗ column and sorts the legal ones first. Legality comes from duels.ink, prices from a daily TCGPlayer mirror; both are best-effort and just say "—" if the fetch fails.

This is the tool I'd have wanted an hour into the session instead of three.

---

### The decklists weren't actually shipped

I'd been keeping the full tournament decklists — NAC, the Asia Championship, Japan's Kobe event, a regional qualifier — as loose Markdown files in a folder on my laptop. Which meant they weren't *in* anything. Nobody who installs `lorcana-mcp` gets them. I get them, on this machine, until I reformat.

So they moved into the package. `get_meta(event="nac")` — or `"asia"`, `"kobe"`, `"fl-ccq"` — now returns that event's full standings and every decklist I have for it. Plain `get_meta()` tells you which events are available.

Moving them in meant I could finally write a test I should have written the first day: *every bundled 60-card list either sums to 60, or is explicitly flagged as not.* It failed immediately. Five of them were off — a second-place list at 59, another at 61, a couple more the community sources had already flagged as incomplete. These were transcription errors inherited from card-grid screenshots, and they'd been sitting in my "confirmed" files for weeks because nothing counted the cards. Now something does, and a list that doesn't add up can't ship without a ⚠ on it saying so.

---

### The snapshot problem, stated plainly

`get_meta` is bundled data. I refresh it when I cut a release. Which means someone running a version from a month ago is getting month-old tournament results *and has no way to know that*. The tool answers "what's strong right now" with total confidence regardless of how old "now" is.

MCP has no mechanism for a server to tell a client it's out of date, and neither PyPI nor the registry notifies anyone. So `v2.2.0` does the small version: about once a day the server asks PyPI for the latest release, and if you're behind, it prints one line to stderr on startup and appends a note to `get_meta`'s own output. It never blocks, never errors, caches the check, and turns off entirely with `LORCANA_MCP_NO_UPDATE_CHECK=1`. It's not a great solution. It's the honest one — the data is a snapshot, so the tool should at least admit when the snapshot is old.

And thinking about "stale bundled data" led straight to the worse version of the same problem: the *entire card database* is one HTTP fetch from [LorcanaJSON](https://lorcanajson.org). If that host is down, `lookup_card`, `search_cards`, `build_deck` — all of it — just fails. So `v2.3.0` bundles a compressed copy of the full card set (~1.1 MB) in the package. It's still live-first: when online you get today's data. When the fetch fails, the server falls back to the bundled copy, prints one warning that the data might be missing the newest set, and keeps working. A refresh script re-pulls it each release.

---

The session started as data entry and turned into four changes, and they're all the same change. A tier list that had never been checked against its own decklists. A lookup that returned one row where the honest answer was a table. Decklists that lived nowhere and had never been counted. A database with a single point of failure and no fallback. Every one of them handed back something that sounded certain and was quietly incomplete. The fixes don't make the tool smarter — they make it say *when it might be wrong*, which for a tool built on a bundled snapshot is most of the job.

- [v2.3.0](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.3.0) · [v2.2.0](https://github.com/IcaroBichir/lorcana-mcp/releases/tag/v2.2.0)
- [PyPI](https://pypi.org/project/lorcana-mcp/) · [MCP Registry](https://registry.modelcontextprotocol.io/?q=lorcana)
- Earlier: [Part 7 — "Can Other People Use This?"](/blog/lorcana-mcp-part-7-can-other-people-use-this/) · [Part 1 — Why I Built It](/blog/lorcana-mcp-part-1-why-i-built-it/)
