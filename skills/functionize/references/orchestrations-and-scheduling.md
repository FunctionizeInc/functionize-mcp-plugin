# Orchestrations, Scheduling, TDM, Cross-Test Data, and CI/CD

This file covers how to bundle tests into orchestrations, drive them with data, schedule them, share state across tests, and integrate with CI/CD pipelines.

---

## Orchestration configuration

Two **run types**:

- **`Background`** — plain multi-test execution
- **`TDM`** — data-driven iteration where each datasource row produces one execution

### Anatomy of an orchestration config (annotated)

```
name, Status                          ← identifying
orchRunType: "Background" | "TDM"
scheduled: "On Demand" | "Hourly" | "4hour" | "12hour" | "Daily" | "Monthly"
executionType: "Parallel" | "Sequential"
orchestrationType: "single" | "group"
rerun_failed: 0/1                      ← 1 = auto-retry failures
rerun_incomplete: 0/1
testcase: "{TEST_ID_1},{TEST_ID_2}"    ← comma-separated test IDs
runOrder: [{testId, wait}]             ← Sequential only; wait is per-test delay seconds
orch_env_configs_list: [{label, id, defaultSelected}]
browsers: "chrome,firefox"             ← multi-browser multiplies executions

# TDM-only fields
tdm_dataset_id: "{DATASET_ID}"
tdm_mapping_type: "all" | subset
data: [{datasource_id: row_index, uniq_id: "DS001"}, ...]
mapping: [{testcase_id, datasource_id, action_id, attribute, column_name}]

# Misc
capture_screenshot: 0/1
email_alerts: "alice@,bob@"            ← run-summary email on completion
folders, tag_ids                        ← run all tests in these
```

### Create-Orchestration dialog — the full option surface

The UI dialog has tabs: **Details · Notifications · Advanced · Cookies · Headers · Environment Configuration · Integrations.** Beyond the anatomy fields above, the less-obvious options:

- **Details:** Title (required), Projects, Environments, Folders, **Tags / Excluded Tags** (include/exclude by tag), Test Case picker, Schedules (On Demand default), Select Browsers, Orchestration Run Type (**Background** vs **TDM Run**).
- **Advanced:** Run Order (Parallel default / Sequential), Single vs Group, **Group And Env Based Execution**, **Ignore Prior Run Data**, **Run without proxy**, Re-run (Incomplete / Failed / **Visible For Tester**), **Runtime Stream** (pick an execution stream), **Resource Environment**, Email Alerts + Email Note, Environment Management (`URL::new URL`), Proxy Setting (`IP:PORT`), Windows Resolution (W×H), Alerts Delivery, Webhook URL, **Submit Linked Tests to TCM** (push results to the linked test-case-management tool).
- **Cookies / Headers:** inject per-orchestration cookies and request headers (name/value pairs) for the whole run.
- **Notifications / Integrations:** the connector set documented under Notifications and CI/CD below.

### Single vs Group orchestration

- **`single`** — flat list of tests, sequential or parallel.
- **`group`** — tests partitioned into named groups. Groups run in configured order; within each group, parallelism. Use for "Smoke" → "Regression" → "API Validation" checkpoints.

**Groups vs folders are orthogonal.** Folders organize tests in the UI. Groups partition execution within one orchestration. A test can be in folder "Login Flows" and group "Smoke Tests" simultaneously.

### How many run at once — execution licenses

Parallel execution is bounded by the account's available **execution licenses** (concurrent-run capacity). When an orchestration has more tests ready to run than licenses free, the excess does **not** fail — those tests **queue and start automatically** as licenses free up, with no re-trigger needed. A large parallel suite still runs to completion on a smaller license count; it just takes longer.

### Test dependencies — there are NONE

**No pass/fail gating between tests in an orchestration.** Test B does not wait for Test A to pass. The only sequencing primitive is run order — a flat ordered list. If A fails, B still runs. The `wait` field on each runOrder entry is a per-test delay in seconds (e.g., "wait 60 seconds before starting B"), not a dependency.

For true dependency-based execution, gate at the CI/CD level — `GET /api/v4/orchestration/{id}/run`, poll `GET /api/v4/orchestration/{id}/status`, conditionally trigger the next orchestration.

### Conditional execution — not native

"Run B only if A failed" doesn't exist. Workarounds:

- **Groups as checkpoints** — provides structure but doesn't actually gate.
- **`rerun_failed=1`** — orchestration-level auto-retry of failures, not different-test conditional execution.
- **CI/CD gating** — your pipeline runs orchestration, reads result, decides what's next.

### Stop-on-failure — also not supported

The orchestration runs every test regardless of any test's outcome. It does NOT cascade-terminate when one test fails. The only failure-related knobs are **per-test**: `continue_on_error` and `continue_on_verification_failure` (see `capabilities.md` § Run tests, and `diagnostics-and-maintenance.md` for the tolerance-mode concept), and they apply to the step list within a single test, not across tests.

---

## TDM (test data management)

TDM is the binding layer that connects a datasource (CSV / JSON) to test action attributes.

### Use TDM when

- One workflow needs to run against many inputs (50 states, 100 user accounts, 20 SKUs)
- You want automatic row-by-row iteration — one execution per datasource row
- You need write-back: capture step output back into the datasource

### Use a datasource directly (not TDM) when

- You're storing reference data a custom-code step / extension reads ad-hoc
- The data doesn't drive step-by-step iteration

### Concrete example — state-tax orchestration

Datasource `state-tax-rates`:
```
state, tax_rate, expected_total
CA, 0.0725, 107.25
NY, 0.08875, 108.88
TX, 0.0625, 106.25
FL, 0.06, 106.00
```

TDM attachment:
```
Action: "Select State" dropdown (action_id: act_select_state)
  → column "state" → attribute "value" (column_type: read)

Action: "Verify Tax Total" (action_id: act_verify_total)
  → column "expected_total" → attribute "text" (column_type: read)
```

Each datasource row drives one test execution — Row 1 → "CA" (expects "107.25"), Row 2 → "NY" (expects "108.88"), and so on. Don't hand-author per-row TDM references in step fields; the binding supplies each row's values.

### Write-back

Attach a column as `column_type=write` and the step's captured output is written back into that column for the row.

```
Action: "Capture Order ID" (action_id: act_capture_order)
  → column "order_id" → attribute "text" (column_type: write)
```

### Reading the current row from a custom-code step

During a TDM orchestration run, a custom-code step can read the current datasource row's values by column name (e.g. the row's `state` and `expected_total` columns). This is available only while a TDM orchestration is running.

### Gotcha — `fze datasource delete` is GLOBAL

`fze datasource delete` removes the datasource from ALL tests and the TDM listing. Affects every other test using it. To remove a single test's mapping, use `fze test tdm-detach` instead.

---

## Cross-test data sharing

### What works vs doesn't

| Mechanism | Cross-test? |
|---|---|
| A local (single-run) value | ❌ Local to one run; destroyed at end |
| A project variable (read) | ✅ |
| A project variable (write) | ✅ **but only from a custom-code step** — native steps cannot write |
| A value captured from an earlier step | ❌ Within the same test only |
| Orchestration-level variables | ❌ Don't exist |
| TDM write-back | ❌ Not designed for this — write-back updates a row, but the next test reads its own row in sequence |

### Pattern: project variable as shared memory

A cross-test handoff **must** use a **project variable**, not an execution preset: a preset is read-only human-set config, so a test cannot write to it at run time. The pipe is a pre-existing project variable, written from a custom-code step (native capture only ever creates a local). (Which home holds which value: the `functionize` skill → `dsl-and-actions.md`.)

**Test A (setup):** in a custom-code step, capture the newly created record's id from an earlier step's result (for example, parse the digits out of a "User ID: 48291" confirmation) and write it to a project variable named `newUserId`.

**Test B (consumer):** reads the `newUserId` project variable as an input value.

Or in instructions for create-agent tests:
```
Log in as the newly created user (its id is saved in the newUserId project variable).
```

### Orchestration setup for chained tests

**Sequential execution is mandatory.** Parallel means Test B could start before Test A writes the variable.

```
Orchestration "Nightly Regression" — Sequential:
  Test 1: "Setup — Create Test User"    → writes the testUserId project variable
  Test 2: "Edit User Profile"           → reads the testUserId project variable
  Test 3: "Delete User"                 → reads the testUserId project variable
  Test 4: "Verify User Deleted"         → reads the testUserId project variable
```

Each downstream test must act on the handed-off id (the `testUserId` project variable) — never re-locate the record by position or recency ("the first/most-recent user"). This is the data-isolation contract; see `functionize-prompting` anti-pattern `data-isolation`.

**Capture-on-creation is the basis of the handoff:** the upstream test must **capture the record's identifying value (name or system-generated id) at the moment it creates the record** and write it to the shared variable. That captured value is the **only** handle every downstream test uses to **search for, open, and verify** the record — never a re-derived or position-based lookup. (Same rule within a single test; the cross-test case just persists the value in a project variable instead of a local one.)

Pre-create the variable (initial value can be empty):
```
fze project variables add \
  --project-id {PROJECT_ID} --environment-id {ENVIRONMENT_ID} \
  --name newUserId --value ""
```

Project variables are scoped to **(project, environment)**. Both tests must be in the same project and target the same environment.

### Cross-orchestration sharing works automatically

Project variables persist across all runs in the project / environment. Test A in Orchestration 1 sets a project variable named `token` to `"abc"`; Test B in Orchestration 2 reads the `token` project variable and gets `"abc"`.

---

## Native orchestration scheduling

**Discrete cadence, not cron.** `--schedule` accepts only:

| Value | Behavior |
|---|---|
| `"On Demand"` | Manual or API only |
| `"Hourly"` | Every hour |
| `"4hour"` | Every 4 hours |
| `"12hour"` | Every 12 hours |
| `"Daily"` | Once per day |
| `"Monthly"` | Once per month |

**Weekly — schedule the orchestration on a single chosen day of the week** (e.g. every Monday), the native way to run weekly. Still encode the intended day/time in the orchestration title for the read-back, since the `get` response exposes the resolved cadence, not the raw schedule params.

### `--schedule-params`

Free-text string interpreted server-side: `"02:00 UTC"`, `"19:00 IST"`, `"08:00 PST"`. The `get` response doesn't expose the raw params — only the resolved cadence and last-run timestamp, so encode the intended schedule in the orchestration title.

### Calendar limits

Beyond the chosen-day weekly cadence above, there is **no holiday suppression and no "business days only" filter** — a Daily schedule fires every day, weekends and holidays included. For that kind of conditional scheduling, use CI-triggered runs.

### Native scheduling vs CI-triggered — which to use

| Dimension | Native scheduling | CI-triggered (REST API) |
|---|---|---|
| Setup | Two flags on `fze orchestration create` | CI YAML + REST + auth management |
| Cadence | Discrete only | Full cron — any expression |
| Calendar awareness | None | Your CI decides |
| Conditional gating | None | Run only on deploy success, PR merge, main-branch |
| Notifications | Email, SMS, Slack, PagerDuty, MS Teams, Rocket Chat, generic Webhook (set in the UI; `--email-alerts` is the CLI email flag) | Scheduled digests, PR status, a destination with no built-in connector |
| URL / variable overrides per run | Not available | Full per-run injection |
| Chained orchestrations | None | Pipeline stages — gate B on A's outcome |
| Infrastructure | Zero — platform-managed | You own the CI |

**Common pattern:** native scheduling for simple recurring regression with email; CI-triggered for deployment-pipeline integration. They coexist on the same orchestration — an orchestration can be scheduled natively AND triggered ad-hoc from CI.

### Example — daily smoke with email

```
fze orchestration create \
  --title "Production Smoke — Nightly" \
  --project-id {PROJECT_ID} \
  --environment-id {ENVIRONMENT_ID} \
  --run-type Background \
  --execution-type Parallel \
  --capture-screenshot 1 \
  --email-alerts "qa-lead@company.com,oncall@company.com" \
  --testcase "{TEST_ID_1},{TEST_ID_2},{TEST_ID_3},{TEST_ID_4},{TEST_ID_5}" \
  --rerun-failed 1 \
  --schedule Daily --schedule-params "02:00 UTC"
```

---

## CI/CD integration

**Rich native integrations both directions — REST is the fallback, not the default.** Functionize has native inbound CI triggers and a broad set of native outbound notifications including a generic Webhook and Microsoft Teams (full connector lists under Notifications below). Reach for the REST API polling pattern below only for something genuinely off those lists (a bespoke dashboard, a scheduled digest, a destination with no built-in connector).

### Universal pattern

1. `GET /api/v4/orchestration/{ORCH_ID}/run` starts the run (use `POST /api/v4/orchestration/{ORCH_ID}/runwithenvironmentconfig` to pass env config, or `/run/{dataset}` for a TDM dataset)
2. Poll `GET /api/v4/orchestration/{ORCH_ID}/status` (or `/detail`) until it completes
3. Read the passed / failed counts from the response
4. Exit non-zero if any failed

**Auth: OAuth2.** `POST /api/v4/generateToken` mints an access token (validate with `GET /api/v4/validatetoken`); pass it on every call via the **`accesstoken`** header (not `Authorization: Bearer`). The API base is **`/api/v4`** — the full endpoint list + exact response fields are in the platform's API Documentation.

### GitHub Actions

```yaml
name: Functionize Regression
on:
  push:
    branches: [main, staging]
  schedule:
    - cron: '0 6 * * 1-5'

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Functionize orchestration
        id: trigger
        run: |
          # <api-base> and field names below are illustrative — confirm the API base host and exact response fields in your Studio API Documentation
          RESPONSE=$(curl -s \
            "https://<api-base>/api/v4/orchestration/${{ secrets.FZE_ORCH_ID }}/run" \
            -H "accesstoken: ${{ secrets.FZE_API_TOKEN }}" \
            -H "Content-Type: application/json")
          echo "run_id=$(echo $RESPONSE | jq -r '.run_id')" >> $GITHUB_OUTPUT

      - name: Wait for completion
        run: |
          for i in $(seq 1 120); do
            STATUS=$(curl -s \
              "https://<api-base>/api/v4/orchestration/${{ secrets.FZE_ORCH_ID }}/status" \
              -H "accesstoken: ${{ secrets.FZE_API_TOKEN }}")
            STATE=$(echo $STATUS | jq -r '.status')
            if [ "$STATE" = "COMPLETED" ]; then
              PASSED=$(echo $STATUS | jq -r '.passed')
              FAILED=$(echo $STATUS | jq -r '.failure')
              echo "Passed: $PASSED, Failed: $FAILED"
              [ "$FAILED" -gt 0 ] && exit 1 || exit 0
            fi
            sleep 30
          done
          echo "Timed out" && exit 1
```

Same pattern works for Jenkins, GitLab CI, CircleCI — only the YAML syntax differs. (Several have a native inbound-trigger integration — see Notifications — so you may not need the curl at all.)

### Single test run

```bash
# GET /test/{id}/run, or POST /test/{id}/runwithvar to pass variables
curl "https://<api-base>/api/v4/test/{TEST_ID}/run" \
  -H "accesstoken: ${FZE_API_TOKEN}" \
  -H "Content-Type: application/json"
```

### What Functionize provides vs. what you build

| Capability | Native | You build |
|---|---|---|
| Trigger / poll / get results | ✅ REST API | |
| Inbound CI triggers | ✅ Native (see Notifications) | GitLab + others via REST |
| Email / SMS alerts | ✅ Native (Alerts settings; `--email-alerts` for email from the CLI) | |
| Slack / PagerDuty / MS Teams / Rocket Chat alerts | ✅ Native (see Notifications) | |
| Generic outbound webhook | ✅ Native (Webhooks notification + Webhook URL field) | |
| Error tracking / issue sync | ✅ Native (see Notifications) | |
| Daily / weekly digest | ❌ | ✅ Cron + REST API + custom email |
| PR status updates | ❌ | ✅ Your CI script |
| Custom dashboards | ❌ | ✅ Your aggregator |
| Test-case-management sync | ✅ Native (see Notifications) | |
| Defect management | ✅ Native (Jira, FogBugz) | |

---

## Notifications

Functionize has native alerting and outbound-notification integrations — configure them in the project/team **Alerts** settings and on the **Integrations** page (the CLI `--email-alerts` flag is just the email channel for an orchestration, not the whole picture).

**Native alert channels:**
- **Email** — multiple recipients, each independently enabled/disabled and testable; team-level on/off. (`--email-alerts` is the CLI equivalent for an orchestration.)
- **SMS / phone** — add phone numbers (with carrier/provider) to receive text alerts.
- **Slack** — via an incoming-webhook URL + channel name; supports a **separate failure channel**, so failures can route to a different Slack channel than general alerts.
- **PagerDuty** — outbound notification integration.

**Delivery controls:** an *Alerts Delivery* mode (e.g. "errors only" vs. all results), a **Send Alerts For All Projects** toggle, and a **Maximum Alerts per hour** cap.

**Alert types you can enable:** Status (pass/fail), Page-load threshold, and performance thresholds — Performance DomLoaded, Performance Interactive, Performance First Paint — each with its own threshold (e.g. page-load 30s). These DO drive alerts; they are not merely dashboard flags.

**Per-orchestration Notifications (Create/Edit Orchestration → Notifications tab)** — each is a built-in connector with its own config fields: **Jenkins, Slack, FogBugz, Sentry.io, GitLab, Jira, Rocket Chat, Webhooks (generic), and MS Teams.** The Advanced tab also exposes a **Webhook URL**, **Email Alerts**, an **Alerts Delivery** mode, and **Submit Linked Tests to TCM**.

**Genuinely not native — build a relay only for these:** scheduled digests, bespoke dashboards, PR status updates, or a destination with no built-in connector. For those, poll the REST API and POST to your destination. (Microsoft Teams and generic webhooks are NOT in this bucket — they're built in.)

**Other native integrations (Integrations page):**
- **Test Case Management:** TestRail, Xray, Zephyr Squad, Rally, Azure DevOps, Sealights.
- **Defect Management:** Jira, FogBugz.
- **Inbound Triggers (start a run from your pipeline):** GitHub, CircleCI, Heroku, Jenkins, Travis CI, PagerDuty, AWS CodePipeline, Jira, Spinnaker.io, Azure DevOps.

---

## Reporting and flakiness analytics

### Native data sources + windows

| Level | What you get | Max window |
|---|---|---|
| `fze test result-list` (per-test) | Timestamped pass/fail per run, browser, orchestration link | 30 days |
| `fze test reports` (aggregate) | Total pass/fail/warning counts + average time | **7 days** |
| `fze orchestration csv-report` | Per-run pass/fail, per-test step-level reasons, Execution Time, Actions Passed/Failed | Per run |

### What's NOT built in

- No cross-test flakiness dashboard
- No pass-rate threshold filtering server-side
- No time-series trend view

### Practical "where's my flake?" workflow (200-test suite)

1. **Broad signal:** `fze test reports` with 7-day window → account-level pass/fail counts.
2. **Project drill-down:** `fze test list --project-id=N --env=live` → last-run status per test.
3. **Compute pass rates:** for suspicious tests, pull `fze test result-list` and compute `PASSED / (PASSED + FAILED + ERROR)` over 30 days. Rank ascending.
4. **Orchestration flake:** `fze orchestration csv-report` across recent `listofrunids` → per-test per-run pass/fail. CSV has Status, Actions Passed/Failed, Failure Steps, Execution Time.

**Aggregation is client-side.** The raw data is available per-test and per-orchestration; computing trends and thresholds is your job.
