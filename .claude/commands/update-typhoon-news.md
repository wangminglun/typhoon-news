---
description: Run the full hourly typhoon-news pipeline (search -> NYT-style rewrite -> build page -> FB post draft -> publish)
---

You are running the hourly update pipeline for the typhoon news site (repo: wangminglun/typhoon-news, working dir: this repo root). Follow these 4 stages in order. Each stage is a distinct subagent per the project's PRD.md.

## Stage 1 — Search subagent
Spawn a `general-purpose` Agent (foreground) with WebSearch access. Task: find the current top 10 most newsworthy typhoon stories right now, prioritizing any storm actively affecting Taiwan; if none, use the most significant active tropical cyclone worldwide. For each item get: headline, source name, real URL from search results (never fabricate), published time, and a factual 2-3 sentence summary. Also get a 3-5 sentence "overall situation" summary to seed the headline story. Run multiple distinct search queries to ensure freshness — do not reuse stale results from a previous run if the storm has moved on, weakened, dissipated, or been replaced by a new one.

Also have this subagent determine and report:
- **Alert level**: whether Taiwan's CWA currently has an active 海上颱風警報 (sea warning), 陸上颱風警報 (land warning), or neither ("none") in effect right now.
- **Taichung (台中) impact**: a factual 1-2 sentence note on whether any active system directly affects Taichung right now (wind/rain/school-work suspension), or a clear "目前無直接影響" if not.

## Stage 2 — NYT-style writer subagent
Spawn a `general-purpose` Agent (foreground). Pass it stage 1's raw findings verbatim. Task: rewrite in New York Times editorial style (strong lede, nut graf, precise attribution, measured tone) in Traditional Chinese (繁體中文), since the audience is Taiwanese readers. Require exact output structure: `===HEADLINE_TITLE===`, `===HEADLINE_DEK===`, `===HEADLINE_BODY===` (300-450 words), then `===ITEM_1===` through `===ITEM_10===` each with TITLE/DEK/URL lines. See PRD.md for full rationale.

## Stage 3 — Page builder (integration)
Using stage 2's output, regenerate `index.html` in the repo root:
- Update the `<p class="meta">` "最後更新" timestamp to current time (Asia/Taipei).
- Replace the headline `<h2>`, `.dek`, and `.body` paragraphs with the new HEADLINE_TITLE/DEK/BODY.
- Replace all 10 `<li>` entries in `.news-list` with the new items (title, dek, href = URL), keeping the existing `style.css` and page structure/markup conventions unchanged.
- Update the `.alert-bar` div (right after `<main>`, before the headline article): set its class to `alert-bar alert-none`, `alert-bar alert-sea`, or `alert-bar alert-land` based on stage 1's alert level, and update the `.alert-badge` text (目前無警報／海上颱風警報／陸上颱風警報) and the description text next to it accordingly.
- Update the `.region-watch` div (right after the headline article, before `.quick-links`): update its text with stage 1's Taichung impact note. Keep the `<span class="region-label">台中專區</span>` label.
- Leave the `.quick-links` div's three links unchanged (they are static CWA/reference links, not per-run content): 颱風路徑潛勢圖 (https://www.cwa.gov.tw/V8/C/P/Typhoon/PTA.html), 即時風速觀測 (https://www.cwa.gov.tw/V8/C/W/WindSpeed/WindSpeed_All.html), 颱風警報現況 (https://www.cwa.gov.tw/V8/C/P/Typhoon/TY_WARN.html).
- Also update `data/latest.json` with the same structured data: `generated_at`, `alert_level` ("none"/"sea"/"land"), `alert_text`, `region_watch` ({region: "台中", text: ...}), `quick_links` (keep the same 3 static entries), `headline`, and `items`.
Keep the design system (masthead, serif NYT-like styling in style.css) intact — do not redesign, only refresh content.

## Stage 4 — Facebook post draft
Rewrite `fb_post.txt`: 2-4 sentence hook based on the new headline, then the page link `https://wangminglun.github.io/typhoon-news/`, a note that it auto-updates hourly, and a few relevant hashtags. This file is for the user to manually copy-paste to Facebook — do NOT attempt to auto-post via any API.

## Publish
Commit all changed files (`index.html`, `style.css` if touched, `data/latest.json`, `fb_post.txt`) with a message like `Hourly update: <headline short summary>` and push to `origin main`. Verify the push succeeded.

If, after a genuine search, there is truly no active or newsworthy typhoon anywhere in the world, replace the headline with a clear "目前無活躍颱风" (no active typhoon) notice and use the 10 slots for the most recent past storm recap / seasonal outlook instead of fabricating an active storm.
