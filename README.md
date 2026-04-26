# HAR-Troubleshooting

## Executive summary (plain English)
Your HAR capture confirms your diagnosis: there are no critical HTTP failures, but the page is spending a lot of time waiting on **marketing/analytics tags** (Clarity + GTM/HubSpot-related requests), and that creates contention with your own app bundles. The recording started on **April 26, 2026 at 03:42:43 UTC**.

In this capture:
- `onContentLoad` is ~735 ms, but `onLoad` is ~4610 ms (much later because background/tag activity keeps running).
- `e.clarity.ms` is the noisiest third-party endpoint by request count (83 requests).
- GTM, Clarity script load, HubSpot banner, and HubSpot pixel all show meaningful queueing/blocked time.
- Your first-party app bundles (`index-BalvUbkt.js`, `vendor-DlOK_Mxb.js`) repeatedly show high blocked/queueing delays, which is consistent with connection/main-thread competition.

---

## What the HAR says (focused findings)

### 1) Third-party tag pressure is real
From this HAR file:

| Resource group | Requests | Avg total time | P95 total time | Avg blocked | Max blocked |
|---|---:|---:|---:|---:|---:|
| `scripts.clarity.ms/.../clarity.js` | 7 | 486 ms | 724 ms | 60 ms | 136 ms |
| `e.clarity.ms/collect` | 83 | 405 ms | 1231 ms | 28 ms | 470 ms |
| `gtm.js?id=GTM-T2V6LFPF` | 7 | 450 ms | 724 ms | 69 ms | 314 ms |
| `js.hs-banner.com/.../banner.js` | 7 | 364 ms | 557 ms | 87 ms | 211 ms |
| `js.hsadspixel.net/pixels.js` | 7 | 272 ms | 477 ms | 78 ms | 162 ms |

### 2) Core assets are delayed before download
Your core bundles are not just "large" — they are often **waiting** before transfer starts:

| First-party asset | Requests | Avg total time | Avg blocked | Max blocked |
|---|---:|---:|---:|---:|
| `assets/index-BalvUbkt.js` | 7 | 335 ms | 232 ms | 441 ms |
| `assets/vendor-DlOK_Mxb.js` | 7 | 400 ms | 238 ms | 466 ms |

This blocked-time pattern is exactly what you'd expect when many tags compete for network sockets and main-thread execution early in page lifecycle.

---

## Recommended resolution path (novice-friendly, GTM-first)
Because GTM is your “house” for attribution, do this in phases so you preserve business tracking while reducing performance cost.

## Phase 1 — Tag inventory and business owner signoff (1–2 days)
1. In GTM, export current container JSON (Admin → Export Container).
2. Build a simple sheet with columns:
   - Tag name
   - Vendor (GA4, Clarity, HubSpot, Meta, etc.)
   - Trigger(s)
   - Fires on pages
   - Last validated date
   - Business owner
   - Keep / Remove / Needs legal check
3. Remove dead/duplicate tags first (lowest risk, fastest win).
4. Consolidate duplicate pageview/event tags where possible.

**Goal:** no unknown tags and no ownerless tags.

## Phase 2 — Delay non-essential tags (high impact)
Use a trigger model in GTM that prioritizes UX first:

### Keep immediate (Page View / Consent Initialization)
- Critical consent mode logic.
- Absolutely required analytics baseline (usually GA4 page_view only).

### Delay to `Window Loaded` or custom timer (3–5 seconds)
- Clarity base tag.
- HubSpot banner/pixel and non-essential ad pixels.
- Any remarketing/social pixels not needed for immediate conversion logic.

### Fire on interaction (best for low-priority tools)
- Heatmaps/session replay on scroll/click/engagement.
- Low-priority marketing experiments.

**Practical GTM setup:**
- Create a Custom Event trigger like `perf_ready`.
- In site code, push `dataLayer.push({event: 'perf_ready'})` after initial rendering or 3s timeout.
- Move non-essential tags from All Pages to `perf_ready`.

## Phase 3 — Gate by consent + region (compliance + performance)
1. Ensure consent defaults to denied where required until user action.
2. In GTM, map each vendor tag to required consent states.
3. Avoid loading Clarity/HubSpot/Meta before consent where policy requires.

This cuts early requests and legal risk at the same time.

## Phase 4 — Cloudflare optimization layer (safe wins)
Since you’re on Cloudflare DNS + Pages, enable/verify:
1. **HTTP/3 + TLS 1.3** enabled.
2. **Brotli compression** enabled.
3. **Early Hints** enabled (if available on your plan).
4. Cache static JS/CSS aggressively via `Cache-Control` headers from build output.
5. Avoid loading GTM proxy scripts from extra hostnames unless necessary; keep host fan-out low.

## Phase 5 — Protect your core bundles
1. Keep first-party `index`/`vendor` bundles as high priority in HTML order.
2. Add `preload` for your most critical JS/CSS chunks only (do not over-preload).
3. Reduce bundle size through code-splitting where possible.
4. Defer non-critical inline scripts until after first render.

---

## Suggested target state (what “good” looks like)
- Core first-party JS blocked time reduced by at least 40%.
- Clarity collect calls reduced on initial page lifecycle.
- Non-essential tags move from Page View to delayed/interaction triggers.
- `onLoad` gap shrinks meaningfully without breaking attribution.

---

## Validation plan after changes
After each GTM publish (or Cloudflare change):
1. Record a new HAR on homepage + 1 key conversion page.
2. Compare these metrics against this baseline:
   - Count of requests to `e.clarity.ms/collect`
   - Avg/Max blocked time for `index-BalvUbkt.js` and `vendor-DlOK_Mxb.js`
   - `onLoad` timing
3. Run one analytics QA checklist:
   - GA4 page_view received
   - Key conversion events still fire
   - HubSpot lead capture still works
   - No duplicate events

If metrics improve and tracking still passes QA, publish to production.

---

## Quick “start tomorrow” checklist
- [ ] Export GTM container and inventory all tags.
- [ ] Pause/remove dead tags.
- [ ] Move Clarity + HubSpot non-essential tags to delayed trigger.
- [ ] Re-test with HAR and compare blocked time on `index`/`vendor` assets.
- [ ] Verify GA4 + conversion events are still correct.

If you want, next step can be a **copy/paste GTM implementation playbook** (exact trigger names, tag sequencing, and rollback plan) written for non-developers.
