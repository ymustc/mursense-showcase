# MurSense Showcase

Live product overview + demo video for **MurSense** (知几), a
conversational AI agent for DAS bridge structural health monitoring.

**→ https://mursense.miao-yu.com** (mirror: https://ymustc.github.io/mursense-showcase/)

**Live product demo** (the real product, not just the video): **https://mursense-demo.miao-yu.com**

This is a static GitHub Pages site, not a code repo — it exists
separately from [mursense-public](https://github.com/ymustc/mursense-public)
(the actual source code) because ~80MB of demo video doesn't belong in
a git history someone might clone to read code.

## Contents

| File | What it is |
|---|---|
| `index.html` | Language picker landing page |
| `en.html` / `zh.html` / `fr.html` | Full product overview page (English / 中文 / Français) — problem/solution, architecture, trust model, performance numbers, roadmap |
| `final_en.mp4` / `final_zh.mp4` / `final_fr.mp4` | ~6 minute demo video per language, embedded in the corresponding page |
| `final_*.srt` | Subtitle files for each video |

## Updating

The site is dual-hosted: Cloudflare Pages serves the primary domain
(mursense.miao-yu.com) and GitHub Pages serves the mirror directly
from `main`. See `DEPLOY.md` for how a change reaches both, and for
the deployment pitfalls already hit. The `en/zh/fr.html` files
here are synced from `product-page/` in the private `mursense-upgrade`
working repo; there's no build step, just a manual copy when that
source changes.

## Source code

The actual product — frontend, API, agent, algorithm core — lives at
[github.com/ymustc/mursense-public](https://github.com/ymustc/mursense-public).

---

*Solo-built by Miao Yu. Copyright © 2026 Miao Yu.*
