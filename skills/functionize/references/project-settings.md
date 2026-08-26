# Project Settings — the configuration surface

Project Settings (gear → Project Settings) controls project- and environment-level behavior. Tabs: **General · Timeouts · Advanced · Integrations · Headers · Cookies · Alert · Auth · Internal.** Many settings are **per-environment** — the General/Timeouts tabs note *"Changes only apply to the selected environment,"* so set them for each environment (Live, QA, etc.) you run against. When a test's behavior seems off in a way the prompt can't explain (waits, element matching, self-heal, visual tolerance), check here before editing the test.

**Variables are not here.** Runtime variables live in their own areas, separate from this gear → Project Settings dialog: **Project → Execution Presets** (the modern home — where you mark a variable **masked**) and **Project → Variables** (project-scoped variables and secrets). That's the credentials model the whole skill depends on; this dialog is for behavior/reliability settings only.

---

## Reliability & matching (the high-impact knobs)

| Setting | Where | Default | What it does |
|---|---|---|---|
| **Timing Model** | Advanced | slider 0–10, default **5** (per-step DSL field is 1–10) | Aggressive (0) ↔ Conservative (10). Controls execution **pacing** — how briskly the test drives the UI between steps — **NOT** how long the test waits for an element or page to settle (that's the timeouts below). Move it toward Conservative only when the test acts faster than a janky UI can keep up *between* steps; changing it to fix a *wait* problem just slows the whole test. |
| **Self Heal** | Advanced | slider 0–10, default **5** | Aggressive/lenient (0) ↔ Conservative/strict (10) — same axis as `diagnostics-and-maintenance.md`. How readily the platform re-identifies a moved/changed element instead of failing. Conservative heals less (fewer false heals, more genuine fails); aggressive heals more (resilient, but can mask a real change). See `diagnostics-and-maintenance.md`. |
| **Element match requirement** | Advanced | default **92%** | Match-strictness threshold for identifying an element. Tighten to reduce wrong-element matches; loosen if legitimate elements are being missed. |
| **Full page visual check** | Advanced | default **92%** | Match tolerance for full-page visual comparisons. |
| **Missing Element Timeout** | Timeouts | **15s** | How long the test waits for an expected element before failing the step — the lever behind "wait for X to appear" reliability. |
| **Page Load Timeout** | Timeouts | **30s** | Hard cap on page load before the step fails. |
| **Continue On Error** | Advanced | off | Keep running subsequent steps after a step fails (instead of halting). Use with care — it can produce green-looking runs over real failures. |
| **Open Shadow DOM** | Advanced | off | Let selection traverse open shadow trees (needed for some web-component apps). |
| **Filter Hidden Elements** / **Fail on Invisible** | Advanced | off | Ignore hidden elements during selection / fail when a target is present but invisible. |
| **Disable Iframe Sandbox Overwrite** | Advanced | off | Stop the platform from rewriting iframe sandbox attributes (relevant to some payment/3rd-party iframes). |

## Timeouts tab (per environment)

DOM Loaded Threshold (10s) · First Paint Threshold (10s) · Page Load Alert Threshold (30s) · Page Performance Threshold (10s) · **Page Load Timeout (30s)** · **Missing Element Timeout (15s)**. The threshold values feed the performance **Alert types** (below); the two *Timeout* values are hard execution caps.

## General tab (per environment)

Environment selector (Live/QA/…) · **Certificate Errors** (ignore cert errors) · **Environment Management** (a URL-swap mapping so one test runs across environments) · **Host** · **Proxy Setting** · **Region**.

## Auth tab

- **MTLS client key** + **MTLS client certificate (PEM)** — mutual-TLS for apps that require client certs.
- **Additional HTTP Authentication** — basic/HTTP auth per URL (enter the URL without protocol). Credentials still belong in named masked variables (a masked preset variable, or a secret variable where presets aren't available), never inline.

## Alert tab (notifications)

Team Name · **Alerts Delivery** (e.g. "errors only" vs all results) · **Send Alerts For All Projects** · **Email Alerts** (multiple recipients, each with Test + Enable) · **SMS Alerts** (phone number + carrier/provider) · thresholds (Default Alert Threshold, Page-load 30s, **Maximum Alerts per hour** 5, performance thresholds) · **Set Alert Types**: Status, Pageload Threshold, Performance DomLoaded, Performance Interactive, Performance FirstPaint. (Outbound connectors — Slack/PagerDuty/etc. — and per-orchestration notifications: see `orchestrations-and-scheduling.md`.)

## Advanced tab — other options

**Logging / diagnostics:** Enable Logging · Detailed Logging · Capture network logs · Capture console logs · Disable Diagnostic mode · Show Extra Data · Force System Screenshot.

**Environment / browser:** Language · Default browser · Browser Orientation · Window Width/Height · Timezone · **Maximum Concurrent (Parallel) tests allowed** · DNS Override (Domain IP) · Default schedule time (e.g. 08:00 AM EDT).

**Behavior toggles (leave at default unless you know you need them):** Visual Completion · Disable Ajax Cataloging · Disable Notifications · Disable iOS CORS · Update From Run Time · **Disable Web Security** · **Memory Saver Mode On** (lowers the run's browser memory footprint) · Block third party cookies · Skip IE Clear Cookies · Bypass Previous Run Data · **Ignore Prior Run Data** (ignore data carried over from prior runs) · **Ignore variable syntax** (register custom variable-token syntaxes the parser should leave alone).

**Headers / Cookies tabs:** project-level request headers and cookies injected into every test (name/value pairs), the project-wide counterpart to the per-orchestration Cookies/Headers tabs.

---

## When this matters for authoring

- A test that flakes on slow pages → raise the **Missing Element Timeout** / **Page Load Timeout** (these govern how long it waits for elements/pages), before adding explicit waits to the prompt. (The **Timing Model** paces actions — it does *not* extend element/page waits, so it won't fix a slow-to-load flake.)
- Wrong-element matches → tighten **Element match requirement**; missed elements → loosen it.
- Web-component app where selection can't see elements → enable **Open Shadow DOM**.
- "Verify X is not present" passing spuriously → check **Filter Hidden Elements** / **Fail on Invisible**.
- Self-heal masking a real regression (or healing too little) → tune the **Self Heal** setting (`diagnostics-and-maintenance.md`).
- These are project/environment-wide — changing them affects every test in the project, so prefer a prompt-level fix when the issue is one test.
