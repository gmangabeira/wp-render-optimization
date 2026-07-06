# How to Improve Your WordPress Speed Using Claude Code and Fable 5

**A Claude Code skill that turns one brutal day of production WordPress optimization into a repeatable method.**

Mobile PageSpeed **52 → 97 in one day**, on a real news portal with 1,000+ articles, AdSense, a page cache, and a free-tier CDN. Desktop went to 100. CLS to zero. This repo is the distilled method — written as a skill an AI coding agent (or a human apprentice) can actually follow.

## Why this exists

Every WordPress speed tutorial gives you the same tips: "install a cache plugin, use WebP, lazy-load images." We had all of that — and mobile still scored 52.

The real problem is never one tip. A production WordPress site is a **chain of 7 layers that lie to each other**:

```
DB → PHP/theme → page cache → origin → CDN edge → browser cache → parse/render
```

The page cache saves HTML *before* the image plugin rewrites it. The free CDN caches *one* variant per URL and serves the unoptimized mobile page to everyone. The browser keeps HTML the CDN sent a year ago. The config screen says "optimization ON" while the bytes say otherwise.

This skill teaches the debugging discipline, not the tips:

- **The 7-layer mental model** — where each metric is actually born
- **The working loop** — measure → locate the element/phase → verify the bytes → change ONE thing → purge → re-measure
- **The order of levers** — highest gain ÷ lowest risk first (hosting is last, not first)
- **10 production-earned gotchas** — the ones no tutorial mentions (mobile cache poisoning the CDN, exclusion lists colliding, URL-keyed caches invisible in staging, `rsync --delete` eating your WebP files…)
- **A verification playbook** — curl commands and an acceptance checklist to run after every change

## Install (Claude Code)

Drop the skill into any project:

```bash
mkdir -p .claude/skills/wp-render-optimization
curl -o .claude/skills/wp-render-optimization/SKILL.md \
  https://raw.githubusercontent.com/gmangabeira/wp-render-optimization/main/SKILL.md
```

Then ask Claude Code things like:

> "My WordPress LCP is 6s on mobile, diagnose it"
> "Audit my site's cache headers"
> "Why does PageSpeed mobile score way lower than desktop?"

Claude will pick up the skill automatically and follow the method — including the parts that save you from lying to yourself (never measure on a cold edge, never trust a single sample, always read the served bytes instead of the admin screen).

No Claude Code? The skill file is plain markdown — it works just as well as a checklist for humans.

## The case study behind it

One Sunday: full hosting migration (shared LiteSpeed → Docker on a VPS) + speed sprint, done end-to-end by Claude Code (Fable 5) with a human making the calls. Highlights of what the *method* caught that tips never would:

| Villain | Symptom | Layer |
|---|---|---|
| Legacy CDN rule: "Cache Everything" + 1-year browser TTL | Visitors frozen on old HTML | CDN / browser |
| Page cache mobile setting OFF | Mobile got the raw page — and poisoned the CDN for everyone | Page cache → CDN |
| `data-no-lazy` in the image plugin's exclusion list | Hero image silently lost WebP | Plugin collision |
| Preload pointing to a different file than the `<img>` picked | LCP image downloaded twice | Render path |
| Logo `<img>` without width/height | Whole page shifted → CLS 0.216 | Theme |
| Remove-Unused-CSS keyed to production URLs | Perfect staging, broken production homepage | Page cache |

Final numbers, PageSpeed Insights (median of 3, warm edge): **mobile 95-97, desktop 98-100, CLS 0, TBT 0-30ms.**

## What's in the repo

- [`SKILL.md`](SKILL.md) — the full skill: mental model, loop, levers, trap catalog, verification playbook

## License

MIT — use it, fork it, teach your own apprentice.

---

*Built by [Gabriel Mangabeira](https://mangabeira.net) with Claude Code (Fable 5). The method was extracted from a real migration transcript — every gotcha in the catalog was hit, diagnosed, and fixed in production the same day.*
