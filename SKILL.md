---
name: wp-render-optimization
description: >
  A complete method for diagnosing and optimizing WordPress rendering performance
  (PageSpeed / Core Web Vitals) behind a page cache + CDN. Use when the task involves:
  improving PageSpeed/Lighthouse/GTmetrix scores, investigating poor LCP/FCP/CLS/TBT/TTFB,
  configuring caching (WP Rocket, LiteSpeed, Cloudflare), WebP/image delivery, JS delay,
  or any "slow WordPress site" problem. Triggers: "pagespeed", "core web vitals",
  "slow site", "high LCP", "CLS", "optimize rendering", "WordPress cache",
  "Lighthouse score", "GTmetrix".
compatibility: Brand-agnostic. Assumes WordPress + a page cache (WP Rocket or similar) + a CDN (Cloudflare or similar). Tools: curl, PageSpeed Insights API key, SSH access to the server.
metadata:
  version: "1.0.0"
  origin: "Distilled from a real production migration + speed sprint: mobile 52 → 97 in one day, on a 1,000+ article news portal running AdSense. 10 production-earned gotchas catalogued."
---

# WordPress Render Optimization — The Method

This is the material I would hand an apprentice. It is not a list of tips: it is a
mental model + a working loop + a catalog of real-world traps. Generic advice
("install a cache plugin", "use WebP") fails because it treats the site as one thing.
A production WordPress site is a **chain of 7 layers that lie to each other**.

## 1. The mental model: the rendering chain

```
DB → PHP/theme → page cache → origin (nginx) → CDN edge → browser cache → parse/render
```

Golden rules of the chain:

1. **Every layer can undo or hide the work of the previous one.** The page cache saves
   HTML BEFORE another plugin rewrites it. The CDN caches ONE variant and serves it to
   everyone. The browser keeps whatever the CDN sent yesterday.
2. **Optimizing = making sure each layer hands the next one the right bytes, early.**
3. **You never optimize "the site" — you optimize the HTML variant the real visitor
   receives.** Figuring out WHICH variant that is, is half the job.

## 2. The working loop (never skip steps)

```
MEASURE → LOCATE the element/phase → HYPOTHESIS → VERIFY THE BYTES → CHANGE 1 THING → PURGE EVERYTHING → RE-MEASURE
```

### Measuring correctly (where most people fail)

- **Lab ≠ field.** PSI shows two blocks: Lighthouse (lab, right now) and CrUX (field,
  a **28-day rolling window**). If field CLS is 0.28 and lab CLS is 0, the problem is
  HISTORICAL — already fixed, or inherited from a previous era of the site. Don't
  chase ghosts: confirm in the lab before touching anything.
- **Never measure right after a full purge.** With a cold edge, Lighthouse simulates
  inflated LCP/TTI (we watched a fake "15.7s" LCP with a real observed LCP of 2.5s).
  Protocol: purge → warm the HTML + every asset (curl loop) → wait a minute →
  measure 3× → use the MEDIAN. One sample is not data, it's an anecdote.
- **Measure the right variant.** `curl` with a desktop UA, a mobile UA, through the
  CDN, and straight at the origin (`--resolve`). If those four don't return the SAME
  optimized HTML, you found your problem before even opening Lighthouse.

### Locating (the actual skill)

A bad score is not a diagnosis. Open the PSI JSON and answer:
- **Which ELEMENT is the LCP?** (`lcp-breakdown-insight` + `layout-shifts` audits)
- **Which LCP PHASE dominates?** TTFB / load delay / load duration / **render delay**.
  High render delay = blocking CSS/JS; load delay = late discovery (missing preload);
  load duration = heavy image. Each phase has a different medicine.
- **Which element caused the CLS?** The audit names the node AND the cause ("Media
  element lacking an explicit size"). A whole-container shift = something ABOVE it
  (header, font, banner) changed size.

## 3. The order of levers (highest gain ÷ lowest risk first)

1. **TTFB / HTML caching.** Page cache working at the origin (confirm the cache
   plugin's footer comment in the HTML) + HTML cached at the CDN edge with short TTLs
   (10min browser / 1h edge) + **automatic purge on publish** (otherwise new content
   disappears). HTML must NEVER carry a long browser TTL — you cannot purge browsers.
2. **Critical rendering path.** Every synchronous CSS/JS in `<head>` delays first
   paint. Dequeue what isn't used (dashicons for anonymous visitors is the classic),
   defer what must run early but needn't block (CMP/consent), delay-until-interaction
   everything else (analytics, pixels, embeds).
3. **The LCP element.** The hero image needs: `fetchpriority="high"` + eager + out of
   lazy-load + `<link rel="preload">` in the head + **the preload and the `<img>`
   pointing to the SAME file** (same format, same srcset variant — otherwise the
   browser downloads it twice). WebP via `<picture>` in the HTML (see gotcha #1).
4. **Fonts.** Self-hosted, woff2, `font-display: swap` **+ preload** — swap without
   preload swaps the font after first paint and reflows the whole page (CLS ~0.2).
5. **Below-the-fold images.** Lazy-load + `sizes`/`sizeSlug` proportional to the
   DISPLAYED size (a 92px card must not download a 768px variant) + explicit
   width/height on EVERY `<img>` — including the header logo. That's the most common
   and most invisible CLS source.
6. **DOM.** Only touch it if measured >818 elements. A dense editorial homepage at
   ~750 is fine.
7. **Hosting.** The last lever, not the first. TTFB <150ms = hosting is not your problem.

## 4. The trap catalog (all hit in production)

These are the ones NO tutorial tells you:

1. **Free-tier CDNs cache ONE variant per URL** — no device split, no `Accept`
   negotiation. Consequences: (a) the origin MUST serve identical, optimized HTML to
   every UA; (b) WebP via header negotiation does not work at the edge — HTML-level
   `<picture>` is the only cache-safe path.
2. **"Mobile cache" disabled in the page cache = mobile visitors get the RAW page**
   (no minify, no delay, nothing) — and they poison the CDN with it for everyone.
   With a responsive theme: mobile cache ON, separate mobile files OFF.
3. **Optimizers trample each other.** The page cache saves its buffer BEFORE the
   image plugin rewrites `<picture>`. Solution: a bridge on the cache plugin's buffer
   filter (e.g. `rocket_buffer` at priority 5 calling EWWW's page rewriter).
4. **Exclusion lists collide.** `data-no-lazy` (to escape lazy-load) sits in EWWW's
   default picture-webp exclusion list — the hero escaped lazy-load and silently lost
   WebP. Always grep the OTHER plugin's exclusion list.
5. **URL-keyed caches never show up in staging.** Remove-Unused-CSS / critical-CSS
   store CSS PER production URL — staging via IP/alternate host never hits those keys.
   During a migration or theme swap: disable and regenerate BEFORE the cutover.
6. **Inherited zone config bites years later.** A legacy "Cache Everything" rule +
   1-year browser TTL on an old CDN zone = browsers freeze your site. Audit the CDN
   rules, not just WordPress.
7. **Deploys with `rsync --delete` erase server-generated artifacts** (stray `.webp`
   files). Every derived asset must live in the repo.
8. **The config says one thing, the bytes say another.** ALWAYS verify the served
   HTML: delay markers present? `<picture>` around the hero? `defer` on the script?
   One `grep` against the real HTML beats ten admin screens. And mind your own grep:
   a `<picture>` source can be ~700 chars long — short search windows give false
   negatives.
9. **Field metrics have 28 days of inertia.** After a fix, CrUX takes up to a month
   to reflect it. Write down the fix date and stop re-optimizing what's already solved.
10. **Ads are the mobile ceiling.** Eager AdSense costs 20-30 lab points. Delaying it
    until interaction wins them back — but that is a REVENUE decision, not a technical
    one: run it as an experiment (7-day comparison window, revert if revenue drops >5%)
    with a one-command rollback prepared.

## 5. Verification playbook (run after EVERY change)

```bash
# 1. Is the served variant the optimized one? (repeat with a mobile UA and via --resolve at the origin)
curl -s https://SITE/ -A "Mozilla/5.0 (Macintosh)" | grep -c "rocketlazyloadscript"   # delay active?
curl -s https://SITE/ | grep -o '<img fetchpriority[^>]*' | head -1                     # hero eager?

# 2. Cache headers correct?
curl -sI https://SITE/                       # HTML: short max-age (≤600), HIT/MISS at the CDN
curl -sI https://SITE/path/static.css        # static: max-age=31536000, immutable
curl -sI https://SITE/wp-json/wp/v2/posts    # API: DYNAMIC/BYPASS (never cached)

# 3. Purge → warm → measure 3× → median (never a single sample, never a cold edge)
```

Acceptance checklist: LCP ≤2.5s mobile lab · CLS = 0 · TBT <200ms · HTML ≤600s browser
TTL · static 1y immutable · REST/admin bypassed · purge-on-publish tested · identical
page for mobile and desktop · zero double-download of the LCP image.

## 6. Never do this

- Optimize without a saved baseline (keep the PSI JSONs).
- Change two things between measurements.
- Trust staging tests for URL-keyed behavior (RUCSS, edge cache).
- Long browser TTL on HTML (irreversible until it expires).
- Delay a consent/CMP script (that's a legal issue, not a performance one).
- Switch hosting while TTFB is healthy.
- Declare victory on one sample — ad-auction variance is ±20 mobile points.
