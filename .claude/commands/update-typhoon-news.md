---
description: Run the full hourly typhoon-news pipeline (search -> NYT-style rewrite -> build page -> FB post draft -> publish)
---

You are running the hourly update pipeline for the typhoon news site (repo: wangminglun/typhoon-news, working dir: this repo root). Follow these 4 stages in order. Each stage is a distinct subagent per the project's PRD.md.

## Stage 1 — Search subagent
Spawn a `general-purpose` Agent (foreground) with WebSearch access. Task: find the current top 10 most newsworthy typhoon stories right now, prioritizing any storm actively affecting Taiwan; if none, use the most significant active tropical cyclone worldwide. For each item get: headline, source name, real URL from search results (never fabricate), published time, and a factual 2-3 sentence summary. Also get a 3-5 sentence "overall situation" summary to seed the headline story. Run multiple distinct search queries to ensure freshness — do not reuse stale results from a previous run if the storm has moved on, weakened, dissipated, or been replaced by a new one.

## Stage 2 — NYT-style writer subagent
Spawn a `general-purpose` Agent (foreground). Pass it stage 1's raw findings verbatim. Task: rewrite in New York Times editorial style (strong lede, nut graf, precise attribution, measured tone) in Traditional Chinese (繁體中文), since the audience is Taiwanese readers. Require exact output structure: `===HEADLINE_TITLE===`, `===HEADLINE_DEK===`, `===HEADLINE_BODY===` (300-450 words), then `===ITEM_1===` through `===ITEM_10===` each with TITLE/DEK/URL lines. See PRD.md for full rationale.

## Stage 3 — Page builder (integration)
Using stage 2's output, regenerate `index.html` in the repo root:
- Update the `<p class="meta">` "最後更新" timestamp to current time (Asia/Taipei).
- Replace the headline `<h2>`, `.dek`, and `.body` paragraphs with the new HEADLINE_TITLE/DEK/BODY.
- Replace all 10 `<li>` entries in `.news-list` with the new items (title, dek, href = URL), keeping the existing `style.css` and page structure/markup conventions unchanged.
- Also update `data/latest.json` with the same structured data (headline + 10 items + generated_at ISO timestamp).
Keep the design system (masthead, serif NYT-like styling in style.css) intact — do not redesign, only refresh content.

## Stage 4 — Facebook post draft
Rewrite `fb_post.txt`: 2-4 sentence hook based on the new headline, then the page link `https://wangminglun.github.io/typhoon-news/`, a note that it auto-updates hourly, and a few relevant hashtags. This file is for the user to manually copy-paste to Facebook — do NOT attempt to auto-post via any API.

## Publish
Commit all changed files (`index.html`, `style.css` if touched, `data/latest.json`, `fb_post.txt`) with a message like `Hourly update: <headline short summary>` and push to `origin main`. Verify the push succeeded.

If, after a genuine search, there is truly no active or newsworthy typhoon anywhere in the world, replace the headline with a clear "目前無活躍颱风" (no active typhoon) notice and use the 10 slots for the most recent past storm recap / seasonal outlook instead of fabricating an active storm.
