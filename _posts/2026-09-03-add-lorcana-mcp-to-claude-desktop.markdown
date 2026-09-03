---
layout: post
title: "Add Your Disney Lorcana Collection to Claude Desktop in One Click"
modified:
categories: blog
excerpt: "The short version: download one file, double-click it, and Claude can read your Lorcana collection, look up any card, check deck legality, and price out what a deck costs to finish. Here's the setup for Claude Desktop and every other MCP client, plus a walkthrough of enriching a collection export."
tags: [mcp, lorcana, claude, tcg, how-to]
comments: true
date: 2026-09-03T18:30:00-04:00
---

![Disney Lorcana Trading Card Game logo](/images/lorcana-tcg-logo.jpg)

[`lorcana-mcp`](https://github.com/IcaroBichir/lorcana-mcp) plugs Disney Lorcana card data — and your own TCGPlayer collection export — straight into Claude. Once it's connected you can just ask: *"enrich my collection,"* *"what's that card, big pete,"* *"is this deck legal,"* *"what do I need to finish this deck and what would it cost."* [Full background here](/blog/lorcana-mcp-part-1-why-i-built-it/).

![Two Disney Lorcana cards from the Wilds Unknown set — You've Got a Friend in Me and Alien - True Believer](/images/lorcana-wilds-unknown-cards.jpg)

Setup takes about a minute.

### Claude Desktop — one click

1. Download **`lorcana-mcp.mcpb`** from the [latest release](https://github.com/IcaroBichir/lorcana-mcp/releases/latest).
2. Double-click it (or **Settings → Extensions → Install extension…**).
3. Click **Install**, enable it, restart Claude Desktop if asked.

No terminal, no Python, no config file — the bundle brings its own runtime.

### Claude Code, Cursor, VS Code, Windsurf, Cline, Zed

One line, using [`uv`](https://docs.astral.sh/uv/):

```bash
claude mcp add lorcana -- uvx lorcana-mcp serve
```

For the other clients it's the same command (`uvx lorcana-mcp serve`) in that client's MCP config. Exact copy-paste snippets for each are in **[docs/INSTALL.md](https://github.com/IcaroBichir/lorcana-mcp/blob/main/docs/INSTALL.md)**.

### Enrich your collection

This is the one that needs your own data. Export your collection from TCGPlayer — **My Account → My Collection → Export** — which gives you a bare CSV: card names, set names, quantities, and not much else.

Hand it to Claude in Claude Desktop:

> Enrich my collection at /Users/you/Downloads/Lorcana_090326.csv

Use the full absolute path — `~` and relative paths don't resolve from inside the server. Claude runs the `enrich_csv` tool and writes two new files next to your export:

- **`enriched_Lorcana_090326.csv`** — your collection plus ten columns it didn't have: ink, cost, card type, subtypes, strength, willpower, lore, inkable, keywords, and full ability text, along with format legality.
- **`dreamborn_Lorcana_090326.csv`** — formatted to import straight into [dreamborn.ink](https://dreamborn.ink).

![Enchanted-rarity Disney Lorcana cards — Diablo, Mufasa, and Elsa - Spirit of Winter — fanned over a dollar-sign background](/images/lorcana-enchanted-cards.jpg)

From there it's a normal conversation, except Claude now knows exactly what you own:

> Which of my cards are legal in Core Constructed?

> What am I missing to build this deck, and what would it cost to finish? *(paste a decklist)*

> Build me an Amber/Steel deck using only cards I own.

Re-exported your collection later? Point Claude at the previous `enriched_*.csv` as a cache and it only looks up the new cards. Add *"and refresh prices"* to pull current market values while it's at it.

---

*Runs locally — nothing hosted, no account, no API key. It works in Claude Desktop and other desktop MCP clients, but **not** Claude.ai in the browser or ChatGPT, which only connect to hosted servers ([why](https://github.com/IcaroBichir/lorcana-mcp/blob/main/docs/INSTALL.md#what-this-server-cant-do)). Card imagery ©Disney.*
