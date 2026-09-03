---
layout: post
title: "Teaching Claude My Disney Lorcana Collection, Part 7: \"Can Other People Use This?\""
modified:
categories: blog
excerpt: "The project has been on PyPI and the MCP Registry for months, so I assumed \"let other people use it\" was already done. It wasn't — being a published package says nothing about whether a non-programmer can install it, and nothing at all about ChatGPT. The fix was a one-click bundle and a real install guide, plus accepting one hard line I can't design around."
tags: [mcp, lorcana, python, claude, ai, tcg, distribution]
comments: true
date: 2026-09-03T17:10:00-04:00
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

The question was: *"How do I make this so other people can use it — on Claude or ChatGPT?"*

My first reaction was that this was already handled. `lorcana-mcp` has been on [PyPI](https://pypi.org/project/lorcana-mcp/) since [part one](/blog/lorcana-mcp-part-1-why-i-built-it/), it's in the official MCP Registry, it's listed on mcp.so and Glama. `pip install lorcana-mcp` works for anyone. Done, right?

That's the shortcut this project keeps teaching me not to trust: something that's true in the case I tested and quietly isn't in the case I hadn't. "It's a published package" is not the same as "someone who doesn't live in a terminal can install it," and it says *nothing* about ChatGPT.

---

### The line I can't design around

MCP servers connect over one of two transports.

**stdio** — the client launches the server as a local subprocess and talks to it over standard input/output. This is what `lorcana-mcp serve` does. Every desktop MCP client works this way: Claude Desktop, Claude Code, Cursor, VS Code, Windsurf, Zed.

**streamable HTTP** — the server is a web service at an HTTPS URL, and the client connects to it over the network.

Claude.ai in the browser and ChatGPT — app or web — *only* speak the second one. They have no way to launch a subprocess on your machine; they're not on your machine. They connect to a URL or they connect to nothing.

No amount of packaging bridges that. I could publish ten more releases and submit to every directory that exists, and ChatGPT still could not use this server, because there is no URL to point it at. Supporting those clients isn't a distribution problem. It's a "run a server on the public internet, forever" problem.

---

### Why I'm not standing up a server

To reach ChatGPT and Claude web I'd need to host `lorcana-mcp` somewhere with an always-on HTTPS endpoint. That means a hosting bill that's mine indefinitely, an auth story (both clients expect OAuth), rate limiting so one person's script doesn't run up my costs, and uptime that's my problem at 2am. For a hobby tool that reads public card data, that's a bad trade — I'd be turning a thing I maintain in an afternoon into a small operations job.

And there's a second reason that made the decision easy. The two tools people actually use this for — `enrich_csv` and `audit_csv` — read a CSV off your disk and write two new files back next to it. A server running on someone else's machine has no access to *your* files. A hosted `lorcana-mcp` would be the read-only half of the tool: card lookup, search, deck analysis. The half that makes it worth installing — "here's my actual collection, enrich it" — structurally cannot run anywhere but your own computer.

So "make it work on ChatGPT" was never "the same tool, somewhere else." It was "a lesser version of the tool, that I pay to run." I decided the honest move was to make the local path — the one where the whole tool works — as close to frictionless as I could, and to write down plainly that web and ChatGPT are out.

---

### The friction that was actually there

Here's what installing this looked like before, for someone on Claude Desktop who isn't a developer:

1. Install Python, or install `uv`, from a terminal.
2. Find `claude_desktop_config.json` — a file in a Library folder most people have never opened.
3. Hand-edit JSON without a trailing-comma mistake.
4. Know to fully quit and reopen Claude Desktop, not just close the window.

Every one of those is a place to give up. The README had the config snippet, but a snippet isn't an install.

The fix is a **Claude Desktop bundle** — an [`.mcpb` file](https://github.com/modelcontextprotocol/mcpb) (formerly `.dxt`). It's a single file you double-click. Claude Desktop unpacks it, uses its own bundled copy of `uv` to set up an isolated environment, and starts the server. No terminal, no config file, no Python install. The whole thing is:

```
Settings → Extensions → Install extension… → pick the file → Install
```

The bundle itself is deliberately tiny — a manifest and a one-line launcher:

```python
from lorcana_mcp.server import mcp

if __name__ == "__main__":
    mcp.run()
```

It doesn't vendor the source. Its dependency spec pins `lorcana-mcp==2.1.0`, so on first launch `uv` installs that exact version from PyPI and runs it. The bundle can never drift into being a stale fork of the real package — it's a launcher for it.

Packing it is one command (`mcpb pack`), and I wired that into CI: on a version tag, GitHub Actions validates the manifest, packs the `.mcpb`, and attaches it to the release. It pointedly does **not** touch PyPI or the MCP Registry — [that release path is four careful manual steps](/blog/lorcana-mcp-part-6-a-format-thats-two-weeks-old/) and I didn't want an automation standing in the middle of it. CI builds the artifact; a human still cuts the release.

---

### Writing down where it runs

The other half was a real install guide instead of a README section. [`docs/INSTALL.md`](https://github.com/IcaroBichir/lorcana-mcp/blob/main/docs/INSTALL.md) now has copy-paste config for eight clients — Claude Desktop (bundle and manual), Claude Code, Cursor, VS Code, Windsurf, Cline, Zed, and "any other stdio client" — which all collapse to the same thing:

```bash
uvx lorcana-mcp serve
```

`uvx` fetches and caches the package on first run, so for most clients there's no explicit install step at all — the config *is* the install.

And it has a section titled **"What this server can't do,"** which says in plain words that Claude.ai web and ChatGPT aren't supported, and why: they need a hosted remote server, and the file-based tools couldn't work remotely regardless. Better for someone to read that in thirty seconds than to spend an evening trying to add a URL that doesn't exist.

---

The pattern across seven of these posts is the same each time. "Publish the code and people can use it" sounds finished, and it hides two things: that "use it" splits across two runtimes that share nothing, and that the best part of this particular tool is the part that can't leave your laptop. The work wasn't reaching more clients. It was making it genuinely easy on the clients where the whole thing actually runs, and being upfront about the rest.

- [Claude Desktop bundle + all releases](https://github.com/IcaroBichir/lorcana-mcp/releases/latest)
- [Install guide (every client)](https://github.com/IcaroBichir/lorcana-mcp/blob/main/docs/INSTALL.md)
- [PyPI](https://pypi.org/project/lorcana-mcp/) · [MCP Registry](https://registry.modelcontextprotocol.io/?q=lorcana)
