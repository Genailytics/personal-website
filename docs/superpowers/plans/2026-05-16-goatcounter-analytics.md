# GoatCounter Analytics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add privacy-friendly visitor analytics to `iamabhishek.cloud` via a single async GoatCounter snippet that only counts hits on the production domain.

**Architecture:** One inline config `<script>` (sets `window.goatcounter.path` to a guard callback) followed by the async GoatCounter `count.js` loader, inserted just before `</body>` in the single-file site, after the existing site script. No build step, no new runtime files, no CSP changes (page has no Content-Security-Policy).

**Tech Stack:** Static HTML/JS, GoatCounter hosted (site code `abhishek2602`), GitHub Pages.

**Spec:** [docs/superpowers/specs/2026-05-16-goatcounter-analytics-design.md](../specs/2026-05-16-goatcounter-analytics-design.md)

**Testing note:** This repo is a single static `index.html` with no test runner or build. Adding a browser-test harness for a 4-line analytics snippet would violate YAGNI. The skill's verify-before-done intent is preserved via the explicit, concrete manual verification in Task 2 (expected outputs stated). No automated test is written because there is genuinely nothing to run it with.

---

## File Structure

- Modify: `index.html` — insert analytics snippet between the closing `</script>` of the existing site script (line 1969) and `</body>` (line 1971). This is the only file changed. Responsibility unchanged: still the single self-contained page; the snippet is an isolated, self-contained block at the end of `<body>`.

---

### Task 1: Add the GoatCounter snippet to index.html

**Files:**
- Modify: `index.html` (insert after line 1969 `</script>`, before line 1971 `</body>`)

- [ ] **Step 1: Confirm the insertion point is unchanged**

Run: `sed -n '1969,1972p' index.html`
Expected output exactly:
```
</script>

</body>
</html>
```
If it does not match (file edited since planning), find the final `</script>` before `</body>` and use that as the insertion point instead.

- [ ] **Step 2: Make the edit**

Replace the blank line between `</script>` (line 1969) and `</body>` (line 1971) with the analytics block. The unique anchor for the edit is the literal text:

```
</script>

</body>
```

Replace it with:

```
</script>

<!-- Privacy-friendly analytics (GoatCounter) — counts production domain only -->
<script>
  window.goatcounter = {
    path: function (p) {
      return location.hostname === 'iamabhishek.cloud' ? p : null;
    }
  };
</script>
<script data-goatcounter="https://abhishek2602.goatcounter.com/count"
        async src="//gc.zgo.at/count.js"></script>

</body>
```

Rationale for the API used: GoatCounter's `count.js` reads a global `window.goatcounter` object defined *before* it loads. Its documented `path` setting accepts a callback `function(path)`; returning `null` cancels the pageview request. So the beacon fires only when `location.hostname === 'iamabhishek.cloud'`, and is suppressed on `localhost`, `127.0.0.1`, `*.github.io` previews, or any other host. `count.js` is loaded `async` and fire-and-forget, so it cannot block or break page render.

- [ ] **Step 3: Verify the edit landed correctly**

Run: `sed -n '1969,1982p' index.html`
Expected: shows `</script>`, blank line, the `<!-- Privacy-friendly analytics ... -->` comment, the inline `window.goatcounter` config script, the async `count.js` script with `data-goatcounter="https://abhishek2602.goatcounter.com/count"`, blank line, then `</body>` and `</html>`.

- [ ] **Step 4: Sanity-check the HTML is still well-formed**

Run: `grep -c '</body>' index.html && grep -c '</html>' index.html && grep -c 'gc.zgo.at/count.js' index.html`
Expected: `1` and `1` and `1` (exactly one of each — no duplicate body/html tags, exactly one analytics loader).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add GoatCounter analytics (production-domain only)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Deploy and verify live (manual, human-in-the-loop)

**Files:** none (verification only)

This task confirms real behavior. It requires the change to be live on GitHub Pages and a human with a browser + the GoatCounter dashboard. The implementing agent should present these steps to the user and wait for confirmation rather than claim success unverified.

- [ ] **Step 1: Push so GitHub Pages rebuilds**

```bash
git push origin develop
```
Then merge/deploy per the repo's normal flow so `iamabhishek.cloud` serves the new `index.html`. (Confirm with the user which branch GitHub Pages serves; if Pages serves `main`, open a PR / merge as the user prefers.)

- [ ] **Step 2: Verify a production hit is recorded**

Open `https://iamabhishek.cloud` in a normal browser (analytics/ad-block disabled). Within ~30s, the visit should appear at `https://abhishek2602.goatcounter.com`.
Expected: pageview count for `/` increments by 1.

- [ ] **Step 3: Verify non-production is NOT recorded**

Serve the same file locally (e.g. `python3 -m http.server 8000` in the repo root) and open `http://localhost:8000`. Reload a few times.
Expected: GoatCounter dashboard count does **not** change (the `path` guard returned `null` because hostname ≠ `iamabhishek.cloud`).

- [ ] **Step 4: Verify graceful degradation**

In browser devtools (Network tab → block request URL pattern `gc.zgo.at`), reload `https://iamabhishek.cloud`.
Expected: page renders and behaves identically (rotator, terminal, fade-ins, konami all work); only the analytics beacon is absent. No console errors that break the page.

- [ ] **Step 5: Confirm done**

Report to the user: production hit recorded ✅, localhost suppressed ✅, page unaffected when blocked ✅. Point them to their dashboard: `https://abhishek2602.goatcounter.com`.

---

## Self-Review

**1. Spec coverage:**
- Goal (understand visitor numbers, no server logs/cookies/banner) → Task 1 adds GoatCounter; nature of tool covered by spec decision. ✅
- Decision: GoatCounter, site code `abhishek2602`, endpoint `…/count` → Task 1 Step 2 uses exact endpoint. ✅
- The Change: single snippet before `</body>`, after existing script, no CSP/build changes → Task 1 insertion point + Step 1 verification. ✅
- Localhost / non-production filter via production-domain guard → Task 1 Step 2 `path` callback; verified in Task 2 Step 3. ✅
- Data flow / error handling (async, fire-and-forget) → Task 1 Step 2 rationale + Task 2 Step 4. ✅
- Out of scope (no custom events, no SPA tracking, no self-host) → not implemented, correctly absent. ✅
- Verification checklist (prod hit, localhost suppressed, renders when blocked) → Task 2 Steps 2–4 map 1:1 to spec's Verification section. ✅
No gaps.

**2. Placeholder scan:** No TBD/TODO/"handle edge cases"/"similar to". All code shown literally. ✅

**3. Type consistency:** Single symbol used — `window.goatcounter.path` callback returning `p` or `null`; the same `iamabhishek.cloud` hostname literal appears in plan, spec, and verification consistently. Endpoint string `https://abhishek2602.goatcounter.com/count` consistent across plan and spec. ✅
