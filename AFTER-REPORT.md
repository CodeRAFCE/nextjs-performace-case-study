# DesignHQ Landing Page: "After" Lighthouse Report

Results after the optimization work. Every number comes from the Lighthouse JSON files in `after/` and the baseline files in this folder (see [BEFORE-REPORT.md](BEFORE-REPORT.md)). Nothing is estimated.

## How it was measured

Same setup as the baseline, so the two reports can be compared directly.

| | Before | After |
|---|---|---|
| URL | `https://designhq-optimized.vercel.app/` | same (full landing page, all sections) |
| Date | 2026-10-22 | 2026-10-24 |
| Lighthouse | 13.5.0, headless Chrome | 13.5.0, headless Chrome |
| Runs | 3 per device, per category | 3 per device, per category |
| Desktop | `--preset=desktop` | same |
| Mobile | default (simulated Moto G Power, slow 4G, 4x CPU) | same |

```bash
npx lighthouse@13.5.0 https://designhq-optimized.vercel.app/ --preset=desktop --only-categories=performance --output=json --output-path=after/performance/desktop-1.json --chrome-flags="--headless=new" --quiet
```

---

## 1. Category scores (median of 3)

| Category | Desktop before | Desktop after | Mobile before | Mobile after |
|---|---|---|---|---|
| **Performance** | 62 | **100** | 54 | **92** |
| Accessibility | 88 | **100** | 88 | **100** |
| Best Practices | 96 | **100** | 96 | **100** |
| SEO | 100 | 100 | 100 | 100 |

All runs: desktop performance 99 / 100 / 100, mobile 92 / 92 / 91. The before runs spread from 50 to 76 on desktop; the after runs are consistent.

Desktop accessibility was 92 in the first after-run (footer links, see section 3). After that fix was deployed, the 3 desktop accessibility runs were repeated: 100 / 100 / 100. The files in `after/accessibility/desktop-*.json` are those re-runs.

## 2. Performance metrics (median of 3)

Google's "good" thresholds: FCP ≤ 1.8 s, LCP ≤ 2.5 s, TBT ≤ 200 ms, CLS ≤ 0.1, Speed Index ≤ 3.4 s.

### Desktop

| Metric | Before | After | Change |
|---|---|---|---|
| First Contentful Paint | 0.4 s | 0.4 s | same |
| Largest Contentful Paint | 2.1 s (1.9–3.5) | **0.6 s** | −71% |
| Total Blocking Time | 0 ms | 0 ms | same |
| **Cumulative Layout Shift** | **0.718** (0.18–0.93) | **0.001** | poor → good |
| Speed Index | 2.6 s | **0.6 s** | −77% |
| Page weight | 6,522 KB | **557 KB** | **−91%** |
| Requests | 64 | 48 | −16 |

### Mobile

| Metric | Before | After | Change |
|---|---|---|---|
| First Contentful Paint | 6.1 s | **1.7 s** | −72% |
| **Largest Contentful Paint** | **15.1 s** | **3.3 s** | −78% |
| Total Blocking Time | 167 ms | **76 ms** | −54% |
| Cumulative Layout Shift | 0.005 | 0.000 | good |
| Speed Index | 10.9 s | **1.8 s** | −83% |
| Page weight | 4,405 KB | **512 KB** | **−88%** |
| Requests | 51 | 40 | −11 |

### Where the bytes went (mobile, run 1, transfer size)

| Resource | Before | After |
|---|---|---|
| Images | 3,263 KB | **95 KB** |
| Fonts | 781 KB | **138 KB** |
| JavaScript | 312 KB | **230 KB** |

### LCP breakdown (mobile, run 1, from the real trace)

| Phase | Before | After |
|---|---|---|
| Time to first byte | 112 ms | 110 ms |
| Resource load delay | **1,185 ms** | **31 ms** |
| Resource load duration | 71 ms | 34 ms |
| Element render delay | **1,044 ms** | **200 ms** |
| **Total (real)** | **2,412 ms** | **375 ms** |

The LCP element is the same in both runs: the first hero marquee image ("Project 1"). It now loads almost immediately and renders about 5x sooner.

### Cross-check with other tools (mobile)

The CLI is the main comparison because the baseline was measured with it. Two other tools agree:

| Mobile | CLI (this report) | Chrome DevTools Lighthouse (3 runs) | PageSpeed Insights |
|---|---|---|---|
| Performance | 92 / 92 / 91 | 92 / 94 / 92 | 97–98 |
| FCP | 1.7 s | 1.7 s | 1.1–1.7 s |
| LCP | 3.3 s | 3.3 / 2.9 / 3.2 s | 2.0–2.2 s |
| TBT | 76 ms | 60 / 70 / 60 ms | 30–40 ms |
| CLS | 0 | 0 | 0–0.002 |
| Accessibility / BP / SEO | 100 / 100 / 100 | 100 / 100 / 100 | 100 / 100 / 100 |

DevTools and the CLI run the same Lighthouse with the same simulated throttling on the same machine, so they match. PageSpeed Insights runs on Google's servers with a different network and CPU, so its numbers are a little better.

---

## 3. What was changed

Each change maps to a finding in the before report.

| Finding (before) | Change | Result |
|---|---|---|
| **Images unresized** (`unoptimized: true`, 1.3 MB portfolio image) | Turned the Next image optimizer on (AVIF/WebP) and set correct `sizes` per image | Images 3,263 → 95 KB (mobile) |
| **LCP image was lazy-loaded and undiscoverable** | First 4 marquee images `eager`, first 2 `fetchPriority="high"` | Load delay 1,185 → 31 ms |
| **Desktop CLS from the marquee** (`width={0} height={0}`) | Real `width`/`height` for every marquee image | CLS 0.718 → 0.001 |
| **Heavy fonts** (TTF/OTF, 781 KB) | `next/font/local`, WOFF2, Latin subset; only the heading font is preloaded so body fonts don't compete with CSS | Fonts 781 → 138 KB; CSS arrives ~430 ms sooner |
| **Render-blocking CSS** | Removed dead CSS and unused files (CSS 98.7 → 78 KB), then inlined it (`experimental.inlineCss`) | FCP 6.1 → 1.7 s (mobile) |
| **Unused / heavy JavaScript** | Fixed barrel-file imports that pulled in `@headlessui/react` (−110 KB), with an ESLint rule to keep it fixed; lazy-loaded the booking modal; split sections into server components with small client parts | First-load JS 557 → ~430 KB (raw) |
| **Long main-thread task** (387 ms of GSAP layout reads on load) | Replaced GSAP ScrollTrigger reveals with CSS transitions and one IntersectionObserver per section; hero heading rendered in JSX instead of rewritten by GSAP | Task gone; TBT 167 → 76 ms (mobile) |
| **Accessibility** (contrast, link names, target size) | Darker text colours, `aria-label` on the logo link, decorative arrows hidden from screen readers, 24 px carousel dots; footer links (desktop only) changed from `text-black/40` (2.84:1) to `text-black/60` (5.74:1) with a 24 px minimum height | 88 → 100 on mobile and desktop |
| **Console errors** (404 prefetches of pages not in this repo) | `prefetch={false}` on links to missing pages (marked with TODO comments) | Best Practices 96 → 100 |

Also removed: source files no route imports, and 109 unused public assets (85 → 64 MB in the repo).

---

## 4. What is still open

### Mobile LCP is 3.3 s (simulated) but 375 ms in the real trace
Lighthouse's mobile mode doesn't slow the page down. It loads the page at full speed, then **estimates** the timing on a slow 4G phone, assuming every request that started before LCP shares one slow connection. About 520 KB starts before LCP (mostly JS and fonts), which is where the 3.3 s comes from. PageSpeed Insights reported 2.0 s LCP and a performance score of 97 for the same deploy.

The biggest remaining lever is JavaScript that isn't needed for first paint: GSAP + Embla are ~70 KB gzipped of ~259 KB first-load JS. See "Possible TODOs for Improvement" in the project README.

---

## 5. Summary

| | Before | After |
|---|---|---|
| Performance (desktop / mobile) | 62 / 54 | **100 / 92** |
| Accessibility (desktop / mobile) | 88 / 88 | **100 / 100** |
| Best Practices | 96 | **100** |
| SEO | 100 | 100 |
| Mobile LCP (simulated) | 15.1 s | **3.3 s** |
| Desktop CLS | 0.718 | **0.001** |
| Page weight (desktop / mobile) | 6.5 MB / 4.4 MB | **0.56 MB / 0.51 MB** |
