---
layout: post
title: "Health Dashboard Part 4: An AI Coach Without an API Key"
modified:
categories: tech
excerpt: "The Running tab ends with a cached coaching analysis. Generating it doesn't require ANTHROPIC_API_KEY in the dashboard env. It uses claude -p — the CLI already running in the terminal."
tags: [health, dashboard, ai, claude, python, streamlit, running, anthropic]
comments: true
date: 2026-05-26T14:00:00-04:00
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

The [Running tab](/tech/health-dashboard-part-3-running-tab/) built in Part 3 ends with a coaching evaluation box for each time window — a few paragraphs analyzing pace trends, training load, recovery quality, and what to do next. The box renders immediately on page load. The content is specific to the actual numbers, not generic advice.

None of it requires `ANTHROPIC_API_KEY` set in the dashboard environment.

---

### The constraint

The dashboard is a local Streamlit app. It runs on my machine, imports Python clients directly, reads from local SQLite caches. No cloud infrastructure, no environment credentials committed anywhere.

The obvious way to add AI text generation to a Streamlit app is:

```python
import anthropic
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY
response = client.messages.create(...)
```

That requires a standalone API key. For a personal local tool, setting up a separate billing account and managing an API key just to generate cached coaching notes felt like unnecessary overhead. The Anthropic model I'm already using is Claude Code — the CLI running in my terminal — and that auth lives somewhere completely different.

So the question became: can I reach Claude without a standalone API key?

---

### Claude Code is a CLI

Claude Code ships as the `claude` command. Most of the time it runs interactively. But it also has a non-interactive print mode:

```bash
claude -p "analyze this training data and give me coaching feedback"
```

`-p` runs the prompt, streams the response to stdout, and exits. No conversation state, no interactive session. Prompt in, text out.

---

### The architecture

The evaluation flow has three distinct layers that never overlap:

**Generation** (runs in the Claude Code session, either at startup or on demand):

```
StravaClient → fetch 30 days of runs
compute_run_metrics() → aggregate stats
build_evaluation_context() → format everything as a readable prompt
subprocess.run(["claude", "-p", prompt]) → evaluation text
DashboardCache.set("run_eval:7d:2026-05-26", text, ttl=8h)
```

**Dashboard** (read-only, no LLM calls):

```
DashboardCache.get("run_eval:7d:2026-05-26") → render markdown
```

**On cache miss** (first run or after a forced invalidation):

```
Dashboard renders "Request evaluation" button
→ saves context to ~/.config/health-dashboard/pending_evals/7d.txt
→ prompts user to ask Claude Code to generate
```

The dashboard code has no awareness of how evaluation text was produced. It calls `.get()`. If there's a value, it renders it. If not, it shows a button. Generation is entirely external.

---

### The cache design

Three evaluation periods, three daily cache keys:

```
run_eval:7d:2026-05-26
run_eval:15d:2026-05-26
run_eval:30d:2026-05-26
```

The date suffix means each evaluation is naturally scoped per day — Monday's key won't be served on Tuesday. TTL is 8 hours, which covers a full day of dashboard use without forcing a regeneration mid-morning if you open it twice.

The cache lives in `~/.config/health-dashboard/cache.db`, a SQLite file with one table:

```sql
CREATE TABLE cache (
    key TEXT PRIMARY KEY,
    data TEXT,
    expires_at REAL
)
```

Same structure as the Strava and MFP caches. Writing an evaluation:

```python
def write_evaluation_to_cache(period: str, text: str) -> None:
    _cache.set(f"run_eval:{period}:{date.today().isoformat()}", text, ttl=28800)
```

Reading it in the dashboard is a `.get()` call. The interface is intentionally thin — the cache doesn't know what generated the text, and doesn't care.

---

### The start script integration

`generate_evals.py` wraps the full generation flow:

```python
def _generate(period, runs, metrics, force):
    if not force and evaluation_is_cached(period):
        print(f"  {period}  ✓  cached")
        return

    ctx = build_evaluation_context(period, runs, metrics, TODAY)
    full_prompt = f"{COACHING_SYSTEM}\n\n{ctx}"

    result = subprocess.run(
        ["claude", "-p", full_prompt],
        capture_output=True, text=True, timeout=180
    )
    write_evaluation_to_cache(period, result.stdout.strip())
```

The `start` script calls this before launching Streamlit:

```bash
ALL_CACHED=$(.venv/bin/python -c "
from claude_eval import evaluation_is_cached
print('1' if all(evaluation_is_cached(p) for p in ['7d','15d','30d']) else '0')
")

if [ "$ALL_CACHED" = "1" ]; then
  echo "  Evals   ✓  (all cached)"
else
  .venv/bin/python generate_evals.py || true
fi
```

If all three are cached, it's a one-line check and startup is instant. If any are missing, generation runs — three Claude calls that add about 30 seconds. The `|| true` means a failure doesn't block the launch.

The full startup output now looks like:

```
  Strava  ✓
  MFP     ✓  (cache expires in 340m)
Running evaluations:
   7d  ✓  cached
  15d  generating… ✓
  30d  ✓  cached

Dashboard starting...
  → http://localhost:8501
```

Pass `--no-eval` to skip the check entirely — useful when you want the dashboard open fast and you're not looking at the Running tab. Any other flags forward straight to Streamlit.

---

### The MFP data question

While building this, I wanted to confirm that the 7-day nutrition data was actually showing different values for different days. The project notes had a long-standing warning: `/v2/diary` always returned today's diary regardless of the `date` parameter.

Calling it directly for three different dates:

| Date | Calories | Protein | Carbs | Fat |
|---|---|---|---|---|
| May 26 | 1,185 kcal | 97.6g | 137.5g | 34.1g |
| May 24 | 1,246 kcal | 86.3g | 123.5g | 50.2g |
| May 22 | 2,085 kcal | 166.9g | 215.1g | 69.4g |

Clearly distinct values. The bug is gone — either it was fixed in the API at some point, or it was a different issue than what we observed earlier. Either way, the 7-day nutrition data in the dashboard is real historical data per date. The warning has been removed from the docs.

---

### The division of responsibility

The coaching evaluation is the most "AI" feature the dashboard has, and it works without the dashboard itself being an AI application. The dashboard is a display layer. The analysis happens in the Claude Code session that's already running — either automatically at startup or when you explicitly ask for it.

For a personal local tool, this ends up feeling like the right split. The dashboard should be fast, local, and focused on rendering. The analysis can be done deliberately, on demand, by the tool that's already there.

Running `./start` is now the full morning ritual: auth checked, evaluations confirmed fresh or generated, browser ready with a coaching read of the last 30 days already waiting.
