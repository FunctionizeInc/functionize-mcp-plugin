# Onboarding a New App — Project Setup, Environments, ACS, Variables, Browsers, Folder Structure, First-Test Gotchas

When a user brings a new application into Functionize for the first time, walk them through these decisions in order.

---

## Project-level settings — defaults and when to override

| Setting | Default | Public SaaS | Internal corp app |
|---|---|---|---|
| Timing Model | 5 (slider 0–10) | leave | leave |
| Page Load Timeout | 30s | leave | bump to 60s if on VPN / on-prem |
| Missing Element Timeout | 15s (project default) | leave | **leave — never lower** |
| Accept unsigned certificates (**Certificate Errors**, General tab) | off | leave | on for self-signed certs |
| Disable proxy (run without the **Proxy Setting**, General tab) | off | leave | try on only if app rejects proxy headers — re-enable if no effect |
| Self Heal | 5 (slider 0–10) | leave | leave |
| Continue on verification failure | off | leave | leave (unless soft-fail mode genuinely wanted) |
| Resource environment | standard | leave | leave |

**Critical guardrail:** never lower `missing_element_timeout` to "speed up" tests. Reducing it just makes failures arrive faster — it fixes nothing.

---

## Environments

**One env is fine** when the app has a single URL and all testing targets the same instance.

**Multiple envs** are needed when the team deploys dev / staging / prod separately:

```
Environment  | URL                                  | Purpose
dev          | https://dashboard-dev.internal.co    | Dev builds — flaky, changes often
staging      | https://dashboard-staging.internal.co | Pre-release validation — stable, mirrors prod
live         | https://dashboard.example.com        | Production smoke tests
```

Each env gets its own URL, its own variable scope (different credentials), potentially different timing expectations. Common pattern: smoke tests run hourly against live; full regression runs nightly against staging.

---

## ACS / QuickConnect — when needed and how

Three trigger questions to ask the user:

1. **"Is the app accessible from the public internet?"** URLs starting with `localhost`, `192.168.*`, `10.*`, `172.16-31.*`, or `*.internal` almost certainly need ACS.
2. **"Can you whitelist Functionize IPs in your firewall?"** If yes, whitelisting may work instead of ACS.
3. **"Does anyone on your team have admin access to run a small connector app on a machine inside the network?"** That's the QuickConnect self-service flow.

**Confirm reachability before provisioning.** Don't set up QuickConnect on the strength of a single reachability check — confirm the URL genuinely can't be reached on the standard execution path first. A quick cross-check: see whether another test in the *same project* already runs against that exact URL. If one does, the URL is reachable — proceed on the standard path and do **not** set up QuickConnect. Only provision when reachability is genuinely confirmed absent (no sibling test reaches it, and the trigger questions below point to a private network).

### QuickConnect setup flow

1. Confirm the URL genuinely can't be reached on the standard path (per the note above) → offer QuickConnect.
2. User confirms → Functionize provisions the connector credentials and the per-platform downloads (macOS / Windows / Linux).
3. User installs the connector on a machine inside the network and launches it with the provided credentials.
4. Functionize detects the device connection automatically.
5. Spawn test creation bound to that connector.
6. Tests run through the encrypted connector — no inbound ports, no firewall changes.

**For production**, deploy a dedicated Linux ACS server rather than QuickConnect. QuickConnect is for trial / PoC.

**ACS server requirements (for the dedicated-server path).** Check these gating requirements with the customer's infra team up front:
- **Connectivity:** the connector needs **outbound HTTPS on 443** and must **not** sit behind a proxy that does traffic inspection / decryption (TLS MITM).
- **OS:** **Linux is recommended**; Windows, container/Kubernetes, and macOS are also supported.
- **Hardware:** modest — a small server-class host is sufficient; a faster clock helps more than extra cores.
- **Scope — which traffic crosses the tunnel.** Bind ACS at **Test, Project, or Orchestration** scope; only the test traffic you scope crosses the tunnel.
- **Provisioning and day-2:** the connector is installed with the credentials Functionize provisions, then kept updated and monitored for connection health. Exact egress domains/IPs, firewall rules, install mechanics, and data-residency specifics live in the ACS KB — confirm them with the customer's infra team.

**macOS gotcha:** users may see a security popup on first launch → System Settings → Privacy & Security → "Open Anyway."

---

## Variables — organization pattern for a new project

Reference each variable by its *name*; the platform substitutes the value at run time.

```
PRESET VARIABLES (human-set config, resolved per run from the selected preset; typically one preset per environment):

  Secrets (mark MASKED — encrypted in the keystore, hidden in logs/screenshots/artifacts):
    test_admin_password   = "***"
    test_user_password    = "***"
    test_card_number      = "4111111111111111"    (a designated test card)
    test_card_expiry      = "12/28"
    test_card_cvv         = "123"

  Non-secret identifiers & per-environment config (non-masked — a different value in each preset where it varies):
    test_admin_email      = "admin@functionizeapp.com"
    test_user_email       = "testuser@functionizeapp.com"
    app_base_url          = "https://staging.example.com"
    shipping_address      = "123 Test Street"
    shipping_city         = "Testville"
    shipping_zip          = "90210"

PROJECT VARIABLES (persist across runs):
  Inter-test data pipes (a value one test writes and a later test reads — presets are read-only, so a pipe cannot be a preset):
    shared_order_id       — create it empty; Test A writes it, Test B reads it
  (Also the fallback home for the secrets above where presets aren't available, marked secret.)

TEST-LOCAL VARIABLES (scoped to a single test run):
    order_id              — captured from the confirmation page
    generated_username    — created during registration
    report_name           — e.g. a "Q1-Report-" prefix plus a random 6-character suffix
```

### Anti-patterns to avoid

- Hardcoding credentials in instructions
- Duplicate variables per test for the same value
- Using project or preset variables for ephemeral data
- Putting an inter-test data pipe in a preset — presets are read-only human-set config; a value a test writes at run time belongs in a project variable
- Inventing random-data functions that don't exist — the supported generator set lives in `functionize-prompting`; for a name, use a per-field random string or a custom-code step, never a made-up name-generator function

### Critical: email gotcha

Test accounts MUST use `@functionizeapp.com` addresses — the built-in email reader **cannot read Gmail / Outlook / corporate Exchange**. Other-domain inboxes need a custom Extension integration you'd build and maintain — confirm feasibility before promising it (see `limitations.md` § "Supported — don't mistake these for gaps", the email entry).

---

## Browsers / runtime

**Default: Chrome on current stable runtime.**

Don't rush to create separate Firefox / Edge tests upfront. Create one test in Chrome, get it stable, then run the *same* test on another browser via batch run. Only investigate if it actually fails cross-browser. Browser-specific test suites make sense only when:

- The app has browser-specific rendering or compliance requires it.

**Runtime updates:** only when a specific fix in a newer version addresses your failure, or Functionize support recommends it. Never as a speculative fix.

---

## Folder / tag / orchestration structure — good-practice example

For a SaaS dashboard:

```
FOLDERS:
├── Auth            — Login, SSO, password reset       [smoke, auth]
├── Dashboard       — Widgets, filters, export          [smoke, critical-path]
├── Reports         — Create, schedule, delete reports  [critical-path, reports]
├── User Management — CRUD users, permissions           [admin]
├── Settings        — Profile, billing                  [settings, billing]
└── Integrations    — API keys, webhooks                [api]

TAGS:
  smoke         — 5–8 tests, run every commit
  critical-path — 15–20 tests, run daily
  regression    — all tests, run nightly

ORCHESTRATIONS:
  "Dashboard — Smoke"           → every commit, parallel, Chrome, tag=smoke
  "Dashboard — Daily Critical"  → daily 6AM, sequential, Chrome+Firefox, tag=critical-path
  "Dashboard — Full Regression" → nightly 2AM, parallel, Chrome, tag=regression
  "Dashboard — Edge Smoke"      → Monday 8AM (single chosen day), parallel, Edge, tag=smoke
```

**Rule:** orchestrations for anything you run on a schedule or as a batch. Individual runs for ad-hoc debugging and fix validation.

---

## First 1–5 tests — gotchas to warn the user about

1. **The "recorded ≠ stable" gap.** A recorded test can pass once then fail after a layout shift — recording is a starting point; **create-agent tests are easier to maintain** — they regenerate from the prompt instead of needing per-step surgery. Expect to tune the first run.

2. **Forgetting login.** Every authenticated test needs explicit login instructions (or a Component that handles login) — otherwise it hits a login redirect and fails on the first check.

3. **Rigid verifications.** "Verify exactly 10 results" breaks the moment data changes. Favor existence checks ("verify results are displayed") over exact counts, and prefer a contains/substring match over exact-equals. The full verification-phrasing craft — and its anti-patterns — lives in `functionize-prompting`.

4. **No wait for async content.** Express the async expectation as **intent** — *"wait for the success message to appear, then verify it"* / *"after the record page loads, verify…"* — **not** a fixed `wait N seconds` sleep (see `functionize-prompting` "Waits and synchronization").

5. **Creating all tests at once.** **Ship one test to passing first** — the simplest critical-path workflow (probably login); get it stable, then add the next. This is a *debugging-workload* recommendation, not a capacity limit — Functionize runs large suites concurrently (bounded by your license and infrastructure).
