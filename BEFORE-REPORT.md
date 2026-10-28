# DesignHQ Landing Page: "Before" Lighthouse Report

Baseline measured **before any optimization**. Every number below comes from the six
(per category) Lighthouse JSON files in this folder. Nothing is estimated.

## How it was measured

| | |
|---|---|
| URL | `https://designhq-optimized.vercel.app/` (full landing page, production deployment) |
| Date | 2026-10-22 |
| Lighthouse | 13.5.0, headless Chrome 155 |
| Runs | 3 per device, per category |
| Desktop | `--preset=desktop` |
| Mobile | Lighthouse default (simulated Moto G Power, slow 4G: 150 ms RTT, ~1.6 Mbps, 4x CPU slowdown) |

```bash
npx lighthouse https://designhq-optimized.vercel.app/ --preset=desktop --only-categories=performance --output=json --output-path=desktop-1.json --chrome-flags="--headless=new" --quiet
```

The same command was run with `accessibility`, `best-practices` and `seo`, and without `--preset=desktop` for mobile.

> The "after" runs must use the same command, the same page content (all sections) and 3 runs, so the comparison is fair.

---

## 1. Category scores

| Category | Desktop (runs 1/2/3) | Desktop median | Mobile (runs 1/2/3) | Mobile median |
|---|---|---|---|---|
| **Performance** | 50 / 76 / 62 | **62** | 51 / 56 / 54 | **54** |
| Accessibility | 88 / 88 / 88 | 88 | 88 / 88 / 88 | 88 |
| Best Practices | 96 / 96 / 96 | 96 | 96 / 96 / 96 | 96 |
| SEO | 100 / 100 / 100 | 100 | 100 / 100 / 100 | 100 |

Performance is the weak category and the focus of the optimization work. Accessibility has some room to improve.

## 2. Performance metrics

Thresholds are Google's "good" values: FCP ≤ 1.8 s, LCP ≤ 2.5 s, TBT ≤ 200 ms, CLS ≤ 0.1, Speed Index ≤ 3.4 s.

### Desktop

| Metric | Run 1 | Run 2 | Run 3 | Status |
|---|---|---|---|---|
| First Contentful Paint | 0.4 s | 0.4 s | 0.4 s | good |
| Largest Contentful Paint | 3.5 s | 2.1 s | 1.9 s | poor to good (varies) |
| Total Blocking Time | 0 ms | 0 ms | 0 ms | good |
| **Cumulative Layout Shift** | **0.927** | **0.182** | **0.718** | **poor (all runs)** |
| Speed Index | 2.7 s | 2.6 s | 2.5 s | good |
| Page weight | 6,522 KB | 6,523 KB | 6,522 KB | heavy |
| Requests | 64 | 64 | 64 | |

### Mobile

| Metric | Run 1 | Run 2 | Run 3 | Status |
|---|---|---|---|---|
| First Contentful Paint | 6.1 s | 5.0 s | 6.1 s | poor |
| **Largest Contentful Paint** | **14.9 s** | **15.1 s** | **15.1 s** | **poor (all runs)** |
| Total Blocking Time | 250 ms | 100 ms | 170 ms | borderline |
| Cumulative Layout Shift | 0.005 | 0.015 | 0.004 | good |
| Speed Index | 10.9 s | 9.4 s | 11.1 s | poor |
| Page weight | 4,405 KB | 4,405 KB | 4,405 KB | heavy |
| Requests | 51 | 51 | 51 | |

Mobile numbers use Lighthouse's simulated throttling, so read them as a relative benchmark. The mobile LCP *phase breakdown* is from the real trace and is much shorter than 15 s.

---

## 3. What the data says

### Finding 1: Desktop CLS is caused by the project marquee
- The layout-shift audit attributes about **0.92 of the 0.93** (run 1) to the `animate-marquee` containers (scores 0.513 and 0.409).
- CLS varies a lot between runs (0.93 / 0.18 / 0.72), which suggests it depends on image load timing.
- Hypothesis (to verify): the marquee images in `ProjectMarquee.tsx` use `width={0} height={0}` with `w-auto`. The browser cannot reserve space until each image loads, so the strip reflows.
- Mobile CLS is negligible (≤ 0.015).

### Finding 2: The LCP element is a lazy-loaded marquee image
- LCP element on mobile and desktop: a `<img loading="lazy" width="0" height="0">` from the hero marquee ("Project 1", "Project 2" or "Project 4", depending on the run).
- Lighthouse's LCP-discovery check fails all three tests on mobile: not `fetchpriority=high`, not discoverable in the initial HTML, and uses `loading=lazy`.
- Mobile LCP breakdown (run 1): TTFB 112 ms, **resource load delay 1,185 ms**, load duration 71 ms, **element render delay 1,044 ms**. The wait before the image even starts loading, plus the render delay, dominate.
- Desktop run 1 breakdown: TTFB 112 ms, load delay 457 ms, load duration 345 ms, render delay 167 ms.

### Finding 3: Images are served unresized
- `next.config.ts` has `images: { unoptimized: true }`, so Next's image optimizer is off and raw files are served.
- Lighthouse estimates **2,640 KiB savings on mobile and up to 4,581 KiB on desktop**.
- Largest transfers (mobile run 1): `Portfolio.webp` 1,360 KB (2297×1716 px, displayed at 395×290), `Portfolio-2.webp` 373 KB, `Portfolio-1.webp` 327 KB, `why-us/3.webp` 254 KB, `sticker-img.webp` 190 KB.

### Finding 4: Fonts are heavy
- `OldschoolGroteskVar.ttf` is 364 KB and `OldschoolGrotesk-NormalRegular.otf` is 111 KB (about 475 KB combined).
- `font-display` insight passes.

### Finding 5: Smaller items
- One render-blocking CSS file (17 KB).
- About 86 KB of unused JavaScript (about 80 ms on desktop, about 300 ms on mobile).
- TBT is 0 ms on desktop, so JavaScript execution is **not** the main bottleneck on desktop.

---

## 4. Accessibility (88)

Failing audits (same on all runs):

| Audit | Detail |
|---|---|
| `color-contrast` | `text-sm text-black/30` spans measure about 2.0–2.1:1; 4.5:1 is required (14 elements on desktop, 3 on mobile) |
| `link-name` | 3 links without accessible text: the logo link (`href="/"`) and two icon links (`href="#"`) |
| `target-size` | Small footer links (e.g. `/portfolio`, `/services/branding`) are about 19 px tall; 24 px minimum |

## 5. Best Practices (96)

One failing audit, `errors-in-console`: five `404` responses for `_rsc` prefetches of `/portfolio`, `/blogs`, `/pricing`, `/careers` and `/contact`. These pages are linked from the landing page but are not part of this project, so this is a side effect of migrating only the home page. Resolving them (removing or stubbing the links) would likely restore a 100.

## 6. SEO (100)

No failing audits.

---

## 7. Fix plan (order matters; measure after each step)

| # | Change | Targets |
|---|---|---|
| 1 | Remove `unoptimized: true`, tune `sizes` | image weight, LCP |
| 2 | Fonts: subset / `next/font`, preload what is needed | page weight, FCP |
| 3 | Marquee: real `width`/`height`, eager + `priority` for the first visible image | desktop CLS, LCP |
| 4 | Loading screen in `HomeClient.tsx`: measure its effect, then decide | LCP/FCP (to verify) |
| 5 | Unused JS / render-blocking CSS | TBT, FCP |
| 6 | Accessibility: contrast, link names, target size | Accessibility 88 → 95+ |
| 7 | Best Practices: the 404 prefetches | 96 → 100 |

## 8. After results (fill in per step)

| Step | Perf desktop | Perf mobile | LCP (d / m) | CLS (d) | Notes |
|---|---|---|---|---|---|
| Before | 62 | 54 | 1.9–3.5 s / ~15 s | 0.18–0.93 | this report |
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |

Save each step's JSON files to `lighthouse-report/after/step-N/` using the same folder layout.
