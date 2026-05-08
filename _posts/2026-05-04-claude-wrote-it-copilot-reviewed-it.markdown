---
layout: post
title: "Claude Wrote It, Copilot Reviewed It"
modified:
categories: tech
excerpt: "I used Claude Code to build a Python MCP server from scratch, then let GitHub Copilot review the PR. What happened next was the most useful code review I've had in a while."
tags: [claude, copilot, ai, python, code-review, github]
comments: true
date: 2026-05-04T10:00:00-04:00
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

I recently built a small Python MCP server to give Claude access to my Strava training data. Claude Code wrote most of it. When I opened the PR, GitHub Copilot reviewed it automatically and left 7 comments.

What followed was one of the more interesting workflows I've stumbled into — two AI tools playing opposite roles in the same development loop, and each doing its part genuinely well.

---

### How the build went

I used Claude Code in the terminal to scaffold the entire project: OAuth flow, Strava API client, MCP tool definitions, CLI entry points, packaging, and README. The whole thing — about 550 lines across 8 files — came together in a single session.

The pace was fast. I'd describe what I wanted, Claude would produce it, I'd read it, push back on specific decisions, and we'd iterate. For the OAuth callback handler, for example, I wanted the local HTTP server approach (spin up `localhost:8765`, catch the redirect, shut down). Claude implemented it cleanly using only Python's standard library. I didn't have to write a line of HTTP boilerplate.

The experience of building with Claude Code feels different from autocomplete-style tools. It's closer to pairing with someone who can hold the full context of what you're building and make coherent decisions across files. The output isn't perfect — it's shaped by what you tell it — but the throughput is high and the structural quality is generally solid.

When I was done, I pushed the branch and opened a PR.

---

### What Copilot found

GitHub Copilot's PR review ran automatically and left 7 inline comments. I wasn't expecting much — automated reviews often surface obvious style issues and miss the things that actually matter. That's not what happened here.

The 7 comments broke down like this:

**Security (1)**

> *"The OAuth flow does not send or verify a `state` parameter. Without CSRF protection, any process that can hit the localhost callback can inject an authorization code from a different Strava account and bind this installation to the wrong athlete."*

This one is real. The OAuth `state` parameter is a nonce that protects the callback from being hijacked. The attack surface on localhost is small, but the fix is five lines and the principle matters. Claude built a working OAuth flow but didn't add the state check.

**Correctness (2)**

> *"This `.env` lookup is anchored to the installed package location, not the user's current project directory. After `pip install strava-mcp`, a `.env` placed next to the user's app or shell working directory will never be read."*

This was the most embarrassing one in retrospect. The code used `Path(__file__).parent.parent / ".env"` — which works fine when running from the source checkout, because `__file__` points into the project directory. After `pip install`, `__file__` points into site-packages. I had documented the `.env` workflow in the README; that workflow silently didn't work for anyone who installed via pip.

> *"`limit` is documented as 1–200, but the implementation never validates that. Passing `0` or a negative value produces invalid `per_page` values for Strava, and negative slicing returns the wrong subset instead of rejecting the input."*

Straightforward input validation miss. One line fix.

**Reliability (2)**

> *"These token-exchange requests do not set an HTTP timeout. If Strava's token endpoint stalls or the network hangs, both `auth` and automatic refresh can block indefinitely."*

The `StravaClient._get()` method already had `timeout=15`. The `httpx.post()` calls in `auth.py` — used for the initial token exchange and for refresh — didn't. A hung refresh would block every MCP tool call.

**Documentation (2)**

The README documented the `strava-mcp auth` command but didn't explain that `STRAVA_CLIENT_ID` and `STRAVA_CLIENT_SECRET` needed to be set first. And the setup section and contributing section cited different Strava rate limits (100 vs 200 requests per 15 minutes).

Both of these were things I had actually fixed in a subsequent commit — but the PR still had the old version. Copilot caught them anyway.

---

### The loop

Once I had the review comments, I went back to Claude Code. I described each issue, we looked at the code together, and Claude produced the fixes. The whole remediation pass took about 20 minutes:

- `auth.py`: `Path(__file__)` → `Path.cwd()` for the `.env` lookup
- `auth.py`: added `secrets.token_urlsafe(16)` state generation and callback verification
- `auth.py`: added `timeout=15` to both `httpx.post()` calls
- `server.py`: added `limit = max(1, min(limit, 200))` before the API call
- `README.md`: added credential setup instructions, normalized rate limit

Then I pushed the fixes, and replied to each of Copilot's comments directly on the PR with a note on what was changed and in which commit.

---

### What's actually interesting here

Claude and Copilot are doing structurally different things, and that difference is what makes the loop productive.

**Claude Code** is building. It holds context across the whole project, makes architectural decisions, writes coherent multi-file implementations, and responds to intent. It's an active collaborator. But it's also optimizing for "does this work and does it make sense" — not "what are all the ways this could fail in production."

**Copilot's PR review** is auditing. It reads the diff with the posture of a skeptical reviewer: what's missing, what's inconsistent, what breaks under edge cases. It doesn't know what I wanted — only what I shipped. That's actually the right posture for code review.

The `.env` path bug is a good example of why this matters. Claude and I both knew the intent — load `.env` from "the project root." We both understood `Path(__file__).parent.parent` as "two levels up from the module file." That works during development, so neither of us caught the pip install breakage. Copilot read the code without our context and asked: *where does `__file__` actually point after install?* That's the question we should have asked.

I've had human reviewers catch similar things — bugs that were invisible to the author because the author knew the intent. The value of review is precisely that the reviewer doesn't share your assumptions. Copilot brought that.

---

### The workflow going forward

This isn't a post arguing that AI-written code is good enough, or that AI review replaces human review. The strava-mcp server is a small, low-stakes personal project, and the code still benefits from human eyes before anything important runs on it.

But as a workflow for side projects and prototypes, the loop is genuinely useful:

1. **Build fast with Claude Code** — get to working code quickly
2. **Open a PR and let Copilot review** — get an adversarial read without the overhead of asking a human
3. **Fix with Claude** — address the issues with the same tool that built it
4. **Reply to every comment** — close the loop so the review is actually useful as a record

The human role in this loop is: decide what to build, read the outputs critically, and judge which feedback actually matters. That's not a small role. But the mechanical parts — scaffolding, boilerplate, reviewing for obvious correctness issues — are largely handled.

For a solo engineer building side projects in the gaps between other work, that's a meaningful shift.

---

The PR is at [github.com/IcaroBichir/mcp_strava/pull/1](https://github.com/IcaroBichir/mcp_strava/pull/1) if you want to see the review comments and replies in context. The strava-mcp server itself is the subject of [a separate series](/tech/wiring-claude-to-my-running-shoes-part-1-why-i-built-it/) starting from why I built it.
