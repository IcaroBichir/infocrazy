---
layout: post
title: "Add Your Disney Lorcana Collection to Claude Desktop in One Click"
modified:
categories: blog
excerpt: "The short version: download one file, double-click it, and Claude can read your Lorcana collection, look up any card, check deck legality, and price out what a deck costs to finish. Here's the setup for Claude Desktop and every other MCP client."
tags: [mcp, lorcana, claude, tcg, how-to]
comments: true
date: 2026-09-03T18:30:00-04:00
---

[`lorcana-mcp`](https://github.com/IcaroBichir/lorcana-mcp) plugs Disney Lorcana card data — and your own TCGPlayer collection export — straight into Claude. Once it's connected you can just ask: *"enrich my collection,"* *"what's that card, big pete,"* *"is this deck legal,"* *"what do I need to finish this deck and what would it cost."* [Full background here](/blog/lorcana-mcp-part-1-why-i-built-it/).

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

### Try it

Export your collection from TCGPlayer (**My Account → My Collection → Export**), then tell Claude:

> "Enrich my collection at /Users/you/Downloads/export.csv"

Use an absolute path. You get back a fully enriched CSV plus a file ready to import into [dreamborn.ink](https://dreamborn.ink).

---

*Runs locally — nothing hosted, no account, no API key. It works in Claude Desktop and other desktop MCP clients, but **not** Claude.ai in the browser or ChatGPT, which only connect to hosted servers ([why](https://github.com/IcaroBichir/lorcana-mcp/blob/main/docs/INSTALL.md#what-this-server-cant-do)).*
