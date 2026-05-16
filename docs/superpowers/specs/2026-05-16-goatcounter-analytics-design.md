# Design: GoatCounter analytics for iamabhishek.cloud

**Date:** 2026-05-16
**Status:** Approved

## Goal

Understand visitor numbers (and basic breakdowns) for the personal site
deployed on GitHub Pages at `iamabhishek.cloud`, without server logs,
cookies, consent banners, or bloating the hand-crafted single-file site.

## Decision

Use **GoatCounter** (privacy-friendly, free for personal use, ~3.5 KB
async script). Chosen over Cloudflare Web Analytics and Google Analytics
because:

- The audience is LLM/ML engineers — a crowd that disproportionately runs
  ad blockers and privacy extensions. GA and Cloudflare Insights endpoints
  are commonly blocklisted, which silently undercounts exactly this
  audience. GoatCounter's endpoint is rarely blocked, keeping numbers
  honest.
- Lightest touch: one small async script, no cookies, no `localStorage`,
  no consent banner (GDPR-safe — GoatCounter stores no PII; IP is used
  transiently server-side for country/dedup and not stored).
- No big-corp dependency, which fits the site's craft ethos.

## Account (manual prerequisite — already done)

- GoatCounter site code: `abhishek2602`
- Dashboard: `https://abhishek2602.goatcounter.com`
- Beacon endpoint: `https://abhishek2602.goatcounter.com/count`

## The Change

Single addition to [index.html](../../../index.html), placed just before
the closing `</body>` tag, after the existing inline site `<script>`
block (currently starting at line 1743). No build step, no new files, no
CSP changes (the page has no Content-Security-Policy meta tag).

The base snippet:

```html
<script data-goatcounter="https://abhishek2602.goatcounter.com/count"
        async src="//gc.zgo.at/count.js"></script>
```

### Localhost / non-production filter (approved)

Dev visits from `localhost` / `127.0.0.1` must not pollute the stats.
Implement an **explicit production-domain guard** rather than relying on
count.js internals, so behavior is self-documenting and robust to any
future change in count.js default localhost handling.

Approach: define `window.goatcounter` before loading count.js, using the
documented `path` callback. Returning `null` from the callback tells
GoatCounter not to record the pageview. The beacon is only recorded when
`location.hostname === 'iamabhishek.cloud'`:

```html
<!-- Privacy-friendly analytics (GoatCounter) — production domain only -->
<script>
  window.goatcounter = {
    path: function (p) {
      return location.hostname === 'iamabhishek.cloud' ? p : null;
    }
  };
</script>
<script data-goatcounter="https://abhishek2602.goatcounter.com/count"
        async src="//gc.zgo.at/count.js"></script>
```

(The exact GoatCounter callback API will be confirmed against current
GoatCounter docs during implementation; intent is fixed: count only on
the production domain.)

## Data Flow

1. Visitor loads `iamabhishek.cloud`.
2. `count.js` loads asynchronously (non-blocking).
3. `path` callback runs; on the production domain it returns the path,
   otherwise `null` (skip).
4. On a non-skipped load, one fire-and-forget beacon is sent to
   `https://abhishek2602.goatcounter.com/count` with: path, referrer,
   screen size, and a server-derived country (IP not stored).
5. Stats are viewed at `https://abhishek2602.goatcounter.com`:
   visitors, page views, referrers, browsers, systems, locations over
   time.

## Error Handling

`async` + fire-and-forget. If GoatCounter is down or the script is
blocked, the page is completely unaffected and analytics simply records
nothing. No try/catch or fallback needed.

## Out of Scope (YAGNI)

- Custom events / click tracking (the site is a single static page;
  pageview counts answer the stated question).
- SPA / hash-route tracking (anchor links like `#work` are same-page;
  one pageview per load is sufficient for "visitor numbers").
- Self-hosting GoatCounter (hosted free tier is sufficient).

## Verification

After deploy to GitHub Pages:

1. Visit `https://iamabhishek.cloud` in a normal browser.
2. Confirm the hit appears in `https://abhishek2602.goatcounter.com`
   within seconds.
3. Open the site from `localhost` (or a non-production host) and confirm
   that visit does **not** appear in the dashboard.
4. Confirm the page renders identically with the script blocked
   (e.g. devtools request blocking) — no layout or behavior change.
