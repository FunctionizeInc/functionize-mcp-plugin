# What Functionize CAN Do — the capability map

The affirmative twin of `limitations.md` — when someone asks "can Functionize do X?", check here before reaching for the boundary list. You create, run, maintain, and scale automated UI and functional tests by describing them in plain English; the platform generates the steps, adapts them when the app changes, and its agents handle generation, diagnosis, and repair.

Everything below is reached the same way — **describe what a user can do and the outcome you want**, and the Functionize agent turns it into executable steps. You never hand-author steps, selectors, or syntax; describe intent and let the platform materialize it.

---

## Create and generate tests

- Turn plain-English instructions into executable steps; a generated test runs across multiple browsers.
- Reuse shared workflows (login, navigation) as named, referenceable components rather than re-describing them each time.
- Test public or privately-hosted apps.
- Broad action surface out of one prompt: UI interactions, navigation across tabs and windows, verification and visual validation, flow control, native dialogs, switching between browser contexts, data and variables, custom logic, API and database steps, file handling, email/SMS, and per-test configuration.

## Run tests

- **Live Execution** (full speed) and **Live Debug** (step-by-step, inspectable) on cloud-managed browsers — no infrastructure to run.
- **Chrome, Firefox, and Edge**, run in parallel or in sequence within an orchestration; batch execution supported.
- Runs are asynchronous — submit and collect results on completion.
- Select an **execution preset** per run (or override project variables at trigger time), and run against a different environment by overriding the target URL — no test edit needed.
- Configurable per run: screen resolution/dimensions, timezone and locale, resource sizing (standard/large/extra-large), and region. A run is stoppable mid-flight, records video, and captures per-step screenshots.
- Per-test tuning: timing model, missing-element timeout, page-load timeout, self-heal sensitivity, and advanced options (unsigned certs, disable web security, memory saver, continue-on-error, continue-on-verification-failure, DNS override, HTTP auth, custom headers/cookies, proxy, video capture).

## Maintain and self-heal

- Tests auto-adapt to UI change; self-heal sensitivity is configurable, and the platform learns from prior successful runs.
- On failure the platform analyzes the run end-to-end (screenshots, steps, timing, network), identifies the root cause, and its agents apply fixes — element targeting, timing, step edits, or a full regeneration.
- Every change creates a **restore point** for instant rollback, with side-by-side screenshot comparison against the last successful run.
- Per-step tuning: execution strategies, optional/suppressed steps, and per-step visual validation with thresholds.

## Orchestrate and schedule

- Combine tests from multiple projects into a suite, run in parallel or sequence, with named groups that carry a per-group browser assignment and halt-on-failure.
- Auto-rerun failed or incomplete tests; target tests by folder or tag; hold multiple environment configurations per orchestration and run the whole suite against a different environment without editing tests.
- **Scheduling:** on-demand, hourly, every 4 hours, every 12 hours, daily, and monthly. For a weekly run, schedule an orchestration on a single chosen day. (Orchestration detail lives in `orchestrations-and-scheduling.md`.)
- **Data-driven orchestrations:** attach datasource rows as datasets and run the suite once per row; mark a default dataset; run all rows or a subset.

## Test data management (TDM)

- Datasources in CSV, JSON, Excel, or Google Sheets (multi-sheet), with full create/read/update/delete including adding and updating rows, truncating, and searching.
- Map datasource columns to step inputs and write test outputs back to a datasource; the write-back either **overwrites** the mapped column value or **appends** to it, with configurable empty-cell handling on read.
- Run a test once per enrolled row, or select specific rows; column values are available as runtime variables.

## Variables and dynamic data

- **Execution-preset** (human-set config resolved per run — the modern home for **secrets** when marked *masked* (encrypted in the keystore, hidden in logs/screenshots/artifacts) and for **per-environment config**; multiple presets per project, typically one per environment, selected at run time), **project** (persist across runs — inter-test data pipes, and the fallback home for secrets where presets aren't available), **local** (single execution), **TDM** (from rows), and **previous-step** values (reference an earlier step's value later). See `dsl-and-actions.md` for which home a value belongs in.
- A built-in random-data generator produces fresh **structural** values per run (strings, numbers, email, phone, date of birth). Values that must **look realistic** — person names, addresses, companies, free text — have **no generator**; they come from a custom-code step or a datasource. See `functionize-prompting` for the phrasings.
- Capture and store values from elements, content, or literals; verify variable values; set cookies and local/session storage; compute values with custom JavaScript; and override project variables at trigger time.

## Integrations and CI/CD

- A full REST API triggers tests and orchestrations and retrieves results, so any CI/CD system drives it (Jenkins, GitHub Actions, GitLab CI, CircleCI, Azure DevOps, Bamboo, TeamCity, or any HTTP client); API-key auth; results returned via callback.
- **Test-case management:** push results to your TCM tool — TestRail, Xray, Zephyr Squad, Azure DevOps Test Plans, Rally, and more (authoritative list + setup in `orchestrations-and-scheduling.md`).
- **Bug tracking:** Jira — push results and failure details with fix-version / test-cycle mapping.
- **Notifications:** email per orchestration, and webhooks to Slack, Teams, or any endpoint.
- **Extensions** run custom execution logic (Node.js, Python, Go, Java) at four hooks — before/after test and before/after step — to manipulate variables, override pass/fail, call external APIs or databases, process files, or transform screenshots; external dependencies are supported and extensions are hosted in-platform. You **describe the extension you need** and the platform runs it — you do not write one here.

## Visual validation

- Compare an element or region against a stored baseline with a configurable match threshold, compare against another step in the same run, or run a full-page visual check.
- Baselines are updatable from any run; the match threshold is tunable for dynamic content.

## Advanced interfaces

- **Context switching:** auto-detect and switch between tabs, windows, popups, and iframes, or switch explicitly.
- **API testing:** any HTTP method, headers, and auth; verify responses, status codes, and resource loading, and carry a response into later steps.
- **Database testing:** run queries mid-test and read results into variables for end-to-end "did it actually land" checks.
- **Email and SMS:** read emails (including verification and 2FA codes) through the built-in reader — always an **`@functionizeapp.com`** address, with a unique address available per run — and receive, read, or send SMS through provisioned numbers.
- **Precise element targeting:** when the AI needs guidance, direct it by visible attributes, text, structure, or proximity.
- **Security and private connectivity:** reach apps behind a firewall through a secure tunnel with no firewall changes; three connectivity models (multi-tenant cloud, static-IP allowlist, secure tunnel); mutual TLS per test; and per-test proxy, HTTP auth, headers, and cookies.

## Projects and environments

- Projects carry independent settings, variables, and environments; project config is overridable per test.
- Multiple environments per project (prod / staging / dev / QA), each with its own URL and variable set; clone an environment from a base; enable or disable environments.
- **User management:** role-based access (Admin / User); invite by email; deactivate a user without deleting their work.
- **Organization:** nested folders for tests, a separate folder structure for components, and tags for categorizing and for targeting groups in an orchestration.

## Reporting and analytics

- Per-step pass/fail with error messages, screenshots, and action logs; full execution history (timestamps, browser, status) with date-range filtering.
- Orchestration-level reports (test-by-test breakdown, duration, status), CSV export, and video recordings.

---

**Cross-browser, stated honestly.** Running several browsers at once is supported — each browser is its own execution. Running the **same logged-in test** across browsers *concurrently* can conflict on the shared user or data it touches, so serialize those or stage per-browser data; a no-login flow (marketing, anonymous e-commerce) runs cleanly in parallel. For what's out of scope, see `limitations.md`.
