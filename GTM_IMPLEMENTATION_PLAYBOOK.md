# GTM Implementation Playbook (Copy/Paste, Non-Developer Friendly)

This is the exact next-step playbook to implement the HAR recommendations without breaking attribution.

Use this when your setup is:
- GTM is the main tag hub.
- Cloudflare Pages hosts the site.
- You need to reduce load pressure from Clarity + HubSpot + social/search tags.

---

## 0) Safety first: create a rollback point (5 minutes)

1. Open GTM → **Admin** → **Export Container**.
2. Name the export: `pre-perf-tuning-YYYY-MM-DD.json`.
3. In **Versions**, create a new version note:
   - `Baseline before delayed-tag rollout`

If anything goes wrong, you can publish this previous version instantly.

---

## 1) Create folders (organization)

In GTM, create folders exactly as named:
- `00 - Core`
- `10 - Performance Delayed`
- `20 - Interaction Only`
- `90 - Paused / Review`

Move tags into these folders so anyone can audit quickly.

---

## 2) Create variables (copy names exactly)

### Variable A: `{{JS - Perf Delay Ms}}`
- Type: **Constant**
- Value: `3000`

### Variable B: `{{JS - Is Consent Granted}}`
- Type: **Custom JavaScript**
- Paste:

```javascript
function () {
  try {
    var c = google_tag_manager['{{Container ID}}'].consent;
    // Returns true when at least analytics storage is granted.
    return c && c.getConsentState && c.getConsentState('analytics_storage') === 'granted';
  } catch (e) {
    return false;
  }
}
```

> If this variable is too advanced for your GTM account, skip it and rely on built-in Consent Settings per tag.

---

## 3) Create triggers (copy names exactly)

### Trigger 1: `TR - Core - All Pages`
- Type: **Page View**
- Fires on: **All Page Views**
- Use for only critical tags (consent + minimal analytics baseline).

### Trigger 2: `TR - Perf - Window Loaded`
- Type: **Window Loaded**
- Fires on: **All Pages**

### Trigger 3: `TR - Perf - 3s Timer`
- Type: **Timer**
- Interval: `{{JS - Perf Delay Ms}}`
- Limit: `1`
- Enable when: `Page URL matches RegEx .*`

### Trigger 4: `TR - Interaction - First Scroll`
- Type: **Scroll Depth**
- Vertical Scroll Depths: `25`
- Fire this trigger on: **All Pages**

### Trigger 5: `TR - Interaction - Click`
- Type: **Click - All Elements**
- Fires on: **Some Clicks**
- Condition: `Click Element matches CSS selector body *`

### Trigger 6: `TR - Perf - Custom Event - perf_ready`
- Type: **Custom Event**
- Event name: `perf_ready`
- Fires on: **All Custom Events**

---

## 4) Tag firing map (what fires when)

Use this exact policy unless a vendor has strict requirements.

## Fire immediately (`TR - Core - All Pages`)
- Consent Mode / CMP integration tag.
- GA4 config tag (minimum baseline only).

## Fire delayed (`TR - Perf - Window Loaded` OR `TR - Perf - 3s Timer`)
- Microsoft Clarity base tag.
- HubSpot banner.js / analytics helper scripts.
- Meta/LinkedIn/other remarketing pageview tags.

## Fire on interaction (`TR - Interaction - First Scroll` OR `TR - Interaction - Click`)
- Heatmap/session replay extras.
- Non-essential experiments.
- Low-priority ad platform pixels.

### Important rule
Do **not** leave non-essential tags on `All Pages` page-view triggers.

---

## 5) Tag sequencing template (copy/paste policy)

Open each non-core tag → **Advanced Settings**:

1. **Tag firing options**: `Once per page`.
2. **Tag sequencing**:
   - Fire a setup tag before this tag: select your consent/setup tag if required.
   - Do not fire if setup fails: enabled.
3. **Consent Settings**:
   - Require additional consent where appropriate (analytics/ad storage).

This prevents early unauthorized firing and duplicate runs.

---

## 6) Optional site-side custom event (`perf_ready`)

If your site team can add one snippet, ask them to add this near app bootstrap:

```html
<script>
  window.dataLayer = window.dataLayer || [];
  window.addEventListener('load', function () {
    setTimeout(function () {
      window.dataLayer.push({ event: 'perf_ready' });
    }, 3000);
  });
</script>
```

Then switch delayed tags to `TR - Perf - Custom Event - perf_ready`.

---

## 7) Publish plan (low risk)

### Publish A (safe)
- Move only Clarity + HubSpot non-essential tags to delayed triggers.
- Publish version name: `perf-step-a-clarity-hubspot-delay`.

### Publish B
- Move social/search remarketing pageview tags to delayed/interaction triggers.
- Publish version name: `perf-step-b-remarketing-delay`.

### Publish C
- Apply interaction-only policy to lowest-value tags.
- Publish version name: `perf-step-c-interaction-only`.

Staggering changes makes it easy to isolate issues.

---

## 8) QA checklist after each publish

Use GTM Preview + browser network + GA4 realtime:

1. Page loads correctly (no visual breakage).
2. GA4 page_view still appears.
3. Lead/contact conversion events still appear.
4. No obvious duplicate events.
5. Clarity/hubspot requests start later than before.
6. HAR shows reduced early blocked time on core bundles.

---

## 9) Rollback plan (copy/paste)

If you see broken tracking:
1. GTM → **Versions**.
2. Select previous stable version: `Baseline before delayed-tag rollout`.
3. Click **Publish**.
4. Confirm GA4 and conversion events recover.

Rollback target time: under 5 minutes.

---

## 10) Owner handoff table (fill this in)

| Tag | Owner | Trigger now | Trigger target | Consent required | Status |
|---|---|---|---|---|---|
| GA4 Config | Analytics Lead | All Pages | Core - All Pages | analytics_storage | Keep |
| Clarity | Marketing Ops | All Pages | Perf - 3s Timer | analytics_storage | Move |
| HubSpot Banner | Marketing Ops | All Pages | Window Loaded | analytics_storage | Move |
| Meta Pixel | Paid Media | All Pages | Interaction/Delayed | ad_storage | Move |

---

## 11) Success criteria (2-week window)

- Fewer early requests to `e.clarity.ms/collect` during initial load.
- Reduced blocked/queueing time for `index-BalvUbkt.js` and `vendor-DlOK_Mxb.js`.
- Attribution still valid (GA4 + lead events intact).
- No consent violations observed.

If you want next, I can produce a **vendor-by-vendor decision matrix** (Keep / Delay / Interaction / Remove) based on your exact GTM container export.
