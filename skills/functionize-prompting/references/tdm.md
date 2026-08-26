# TDM — Test Data Management

Load this reference when the user wants to write data-driven tests, share data across multiple test runs, or understand how to attach external data to test steps.

---

## 1. What TDM is for

A **TDM datasource** is a table of input values (a CSV / JSON / XLS / Google Sheet) that drives a test with external data instead of hardcoded values. Use it for parametrized input: one login test whose datasource holds 50 rows of `email, password, expected_result` runs once per row — same logic, different data each run.

**Author workflow:**

1. **Create** — upload the file as a datasource.
2. **Attach** — map datasource columns to specific test step fields (the `email` column → the email-input step's value).
3. **Run in TDM mode** — a TDM orchestration iterates the datasource one row per test execution.
4. **(Optional) Write-back** — a step writes a captured runtime value back into the datasource.
5. **Maintain** — replace content, add or truncate rows as needs change.

---

## 2. Data files — formats, encoding, limits

**Accepted formats:** CSV, JSON, XLS, Google Sheets.

CSV is the most common choice — simple to author, easy to diff in source control, no encoding gotchas if you stick with UTF-8.

**Encoding:** UTF-8. No byte-order mark required.

**Quoting:** Standard CSV quoting. Fields containing commas or newlines must be double-quoted. **Leading and trailing whitespace is preserved** — so `" admin@test.com"` with a leading space is treated literally as that string. Strip whitespace before uploading.

**Headers:** The first row is interpreted as column names. These names become the column identifiers used in TDM attachment. Column names are case-sensitive.

Example file:
```csv
email,password,expected_display
alice.wilson@example.com,Pass123!,Alice Wilson
bob.johnson@example.com,Secure456!,Bob Johnson
```

→ Three columns: `email`, `password`, `expected_display`. Two data rows (the header row is not counted).

**Limits:** No documented hard cap. Practical limits are storage and execution time — each row = one full test execution, so 200 rows = 200 sequential test runs. Keep datasources scoped to a single test purpose; larger datasources mean longer orchestration runs.

---

## 3. Attaching to steps

TDM attachment is configured on the step, not written into the test instructions — **there is no TDM token you write.** You attach, per step: which datasource, which column, which step field the value goes into, and whether the operation is `read` (default) or `write`.

Attach through the platform UI or the CLI / API **after the test is generated**. Generate the test first, then attach.

### Prompt-side phrasing

The create agent doesn't need to know TDM details at generation time. Write the prompt in terms of intent and use a placeholder value or a named project variable for now. Attach TDM columns afterward.

```
Perform login at https://app.example.com/login with the username and password from the login variables
Navigate to the Dashboard
Verify the user's display name matches the expected value
```

Generates clean steps. Then attach `email` → the email input step's value, `password` → the password input step's value, `expected_display` → the verification step's expected text. The test continues to read naturally; TDM works underneath.

### Never a prompt instruction — the forbid

**TDM has no prompt token, no DSL construct, and no in-test variable namespace.** A datasource column binds to a step field in the platform UI (or CLI/API) **after** generation — never through a line in the prompt. So a prompt instruction that tries to **bind a datasource column to a step field** — read a column into a field, or write a value back to a column — **cannot materialize**, and it fails **silently**: the same contamination class as the regex / contains-a-digit / inline-credential forbids, where the agent pattern-matches "use a datasource value" onto the closest thing it can express, a prompt line the platform cannot honor.

- ❌ **WRONG — a datasource read or write-back written as a prompt line:** *"read the `email` column from the current datasource row"*, or *"write the `resultId` back to the `result_id` column of the current datasource row."*
- **Consequence:** there is no prompt token or action type for a datasource read/write-back — the binding is made by **attaching the column to the step field in the platform** (§3), after generation, not by a prompt line.
- ✅ **RIGHT — plain intent in the prompt, column attached in the platform:** write the step in plain intent, using a **placeholder value or a named project variable** for now (see *Prompt-side phrasing* above); then **attach** the datasource column to that step field in the platform (§3). For a **runtime value that feeds a write-back**, do the **verify-capture in the prompt** (`fundamentals.md` §4 (a) — a write-only value is still an element value, so the uniform native verify-capture applies; SKILL.md #28) and **attach the column in `write` mode in the platform** (§5). The capture is a prompt step; the **write itself is not** — it is the attachment.

**Not this rule:** a **custom-JavaScript step** may read the current row's columns *in code* at runtime (§8) — that is a JS step accessing runtime context, not a prompt-authored column→field binding, and it is a legitimate fallback when downstream logic genuinely needs the value.

---

## 4. Reading data

**One row per test execution.** All TDM-attached steps in a single test reference the same row — there is no per-step row switching.

**Empty cell:** if the mapped column is empty in the current row, the step gets an empty value. Per-step config lets you pass the empty value through or skip the step — set this on the step when the empty-row case matters.

---

## 5. Writing back to the datasource

Attach a column in `write` mode and the step's value (or another captured attribute) is written into the current row. Written values are durable — they persist across runs and are readable in later runs. **The write-back is that attachment — never a prompt instruction to "write the value back to the column"** (that silently fails; see §3 *Never a prompt instruction — the forbid*).

**A captured runtime value feeding a write-back is write-only — verify-capture it.** When the value written back is *read at runtime* (a record id, a returned reference number) rather than computed, an empty read writes an empty cell **silently**, and the next run consuming that row gets nothing. Verify-capture it in one instruction — `verify the new record id is not empty and capture it as resultId` — then attach `resultId` to the write column. This is the **write-only** case of the uniform native verify-capture (`fundamentals.md` §4 (a)): a datasource-column write is a store, not a field re-entry, so an empty write never fails loudly on its own — which is *why* the verify matters here even though nothing re-enters the value.

**Write modes:**
- `update` — overwrite the existing value in that column for the current row
- `add` — append to the existing value (accumulate across runs)

### Use case for write-back

Record audit data per row — e.g., "which timestamp did this row's test run finish?" Attach a `last_used` write column to the verification step at the end of the test. After each run, that row's `last_used` column gets stamped.

To capture a timestamp for write-back, add a **custom-JavaScript step** — *"run custom JavaScript to compute the current ISO timestamp and store it in a local variable named `now`."* Then the write-back step uses that captured `now` value.

---

## 6. Reusable vs single-use data

### Reusable data

Credentials, URLs, configuration values that work indefinitely:

```csv
email,password,role
admin@example.com,Pass123!,admin
viewer@example.com,View456!,viewer
```

These rows can be used repeatedly. Each orchestration run cycles through them. No special configuration — that's the default behavior.

### Single-use data

Signup emails, coupon codes, serial numbers where the application rejects duplicates:

```csv
email,first_name,last_name
test_a7f3@functionizeapp.com,Alice,Smith
test_b9e1@functionizeapp.com,Bob,Jones
test_c2d4@functionizeapp.com,Carol,Davis
```

**Pattern for single-use data:** pre-generate enough rows to cover your expected run volume. If you need 50 signup tests, create 50 unique rows. After all rows are consumed, the datasource is exhausted (see failure modes below).

### "Fresh row each run, never reuse"

There is no built-in "mark row as consumed" flag. The pattern is:

1. Create a datasource with N unique rows.
2. Run a TDM orchestration that iterates N times.
3. Do not re-run the same orchestration against the same datasource without first replacing or truncating the consumed rows.
4. If you need indefinite unique data, describe a freshly generated random email on the functionizeapp.com domain in the test instructions instead of TDM (see `native-actions.md`). TDM is for finite, curated datasets.

### Hybrid approach — best of both

Use TDM for structured test data and a generated value for uniqueness:

```
Register a new user with a unique email — generate a random email on the functionizeapp.com domain — plus the first name from TDM and the last name from TDM
```

The TDM datasource holds first_name + last_name pairs (curated, reusable). The email is freshly generated each run (unique). You get both readable test data and zero-collision uniqueness.

### Record ownership in TDM rows

A TDM row that drives a **mutating** flow (a row whose test edits/cancels/consumes a record) follows the isolation contract (SKILL.md anti-pattern #20): the row carries a **unique-per-run identifier the test owns**, not a pointer to a shared record selected by position. Two safe shapes:
- **Seed-key column** — the row supplies a unique key (`account_ref`, `coupon_code`) the test uses to *create* its own record, so each row owns distinct data. Single-use; pre-generate enough rows.
- **Dedicated pinned record per row** — each row names a stable identifier of its *own* record (`order_id`), never "the first/most-recent one".

Never drive a mutating flow from a row that selects "whichever record is on top," and never run such a datasource with **parallel** execution against overlapping rows — two rows must never pick each other's data (see the contention failure mode in §11). Use **random, not timestamp**, for the uniqueness suffix; timestamps collide under parallelism. Static read-only inputs (search terms, curated name pairs) have no ownership concern and stay freely reusable (#19 territory).

---

## 7. Iteration mode — row-by-row execution

**Each datasource row produces exactly one complete test execution.** One row = one full test run. A 20-row datasource produces 20 separate test executions.

**Iteration is triggered at the orchestration level** — not at the test level or step level. When you set up the test run, you choose a TDM-mode orchestration that points at the datasource. The orchestration iterates rows sequentially.

Within a single test execution, all TDM-attached steps reference the same row. There is no per-step row switching, and a single execution never consumes multiple rows.

---

## 8. TDM vs test variables

TDM data is **not a token** — it's a step-field mapping (§3–4), not something you reference in the instructions. Inside a custom-JavaScript step the current row's columns are available to the code, so you can copy a TDM value into a captured local value if downstream logic needs it.

**Key distinction:** preset variables are for human-set config that's the same across a run (environment URLs, shared credentials — mark secrets masked), and project variables for inter-test data pipes; TDM is for data that varies per row (different users, different inputs, different expected outputs). They're complementary, not overlapping.

---

## 9. The global delete gotcha

Deleting a datasource is a **GLOBAL, irreversible operation** that removes the datasource entirely AND disconnects it from ALL tests that reference it. If you have ten tests using the same datasource and you delete it, all ten lose their TDM attachments simultaneously.

**To remove TDM from a single test without affecting other tests,** use the per-step or per-test detach operation. This removes the column mappings from that test's steps but leaves the datasource intact and available to other tests.

**Never use "delete datasource" to revert a TDM change to a single test.**

---

## 10. Cross-test data sharing

Use **TDM** for test inputs that vary per row; use **preset variables** for human-set config that's constant across a run (base URLs, API endpoints, shared credentials — mark secrets masked). Multiple tests in one orchestration can attach to the same TDM column and get the same value from the current row.

**To pass a runtime value from test A to test B in the same orchestration,** hand it off in **two steps**: in test A, **verify-capture the value to a local variable** (`fundamentals.md` §4 (a), #28), then **write that local into a pre-created project variable with a custom-JavaScript step** (`specialty-steps.md` §5); test B reads that project variable by name. The value must land in a **local first** and the project variable must already exist — a one-clause *"verify X and capture it as a project variable named Y"* is the silent no-op #30 names, because a native capture only ever creates a local (`fundamentals.md` §4 (b)). Do NOT use TDM write-back for this — rows are assigned at the start of the orchestration, not updated mid-flight.

---

## 11. Failure modes

| Failure | Cause | Fix / prevention |
|---|---|---|
| Column doesn't exist | A step reads `"userEmail"` but datasource headers are `first_name, last_name` | Verify column names match exactly (case-sensitive) between the datasource and the step attachment |
| Datasource exhausted | TDM orchestration configured for more iterations than the datasource has rows | Ensure row count ≥ expected iterations. Add more rows or replace the datasource content |
| Type mismatch | CSV datasources deliver values as text — `"42"`, not `42`. JSON and XLS datasources preserve typed values, but if your application strictly type-checks a numeric input it may reject the string form. | Use a custom-JavaScript step to coerce the text to a number before the field that needs it |
| Parallel runs consuming the same single-use row | Orchestration set to parallel execution with the same row range assigned | Use sequential execution for TDM orchestrations with single-use data, or ensure parallel branches get non-overlapping row ranges |

---

## 12. Worked example — 20 test users, 5 sequential runs of a login test

This example walks through every step from datasource creation to orchestration execution. All identifiers shown are placeholders.

### Step 1 — author the CSV

```csv
email,password,expected_display
alice.wilson@example.com,Pass123!,Alice Wilson
bob.johnson@example.com,Secure456!,Bob Johnson
carol.davis@example.com,Login789!,Carol Davis
dan.martinez@example.com,Test000!,Dan Martinez
eve.thompson@example.com,Pwd111!,Eve Thompson
frank.garcia@example.com,Auth222!,Frank Garcia
grace.lee@example.com,Pass333!,Grace Lee
henry.brown@example.com,Secure444!,Henry Brown
iris.clark@example.com,Login555!,Iris Clark
jack.wright@example.com,Test666!,Jack Wright
kate.hall@example.com,Pwd777!,Kate Hall
liam.allen@example.com,Auth888!,Liam Allen
mia.scott@example.com,Pass999!,Mia Scott
noah.green@example.com,Secure000!,Noah Green
olivia.adams@example.com,Login111!,Olivia Adams
peter.baker@example.com,Test222!,Peter Baker
quinn.nelson@example.com,Pwd333!,Quinn Nelson
rachel.carter@example.com,Auth444!,Rachel Carter
sam.mitchell@example.com,Pass555!,Sam Mitchell
tina.roberts@example.com,Secure666!,Tina Roberts
```

20 rows, 3 columns. Save as `users.csv`.

### Step 2 — upload the file as a datasource

Through the platform UI or the CLI/API, create a new datasource named `login-test-users` of type CSV, pointing at `users.csv`. The platform returns a datasource ID.

### Step 3 — create the login test

The test prompt:

```
Perform login at https://app.example.com/login with the username and password from the login variables
Verify the dashboard loads and displays the expected user name
```

Two paragraphs is all you need. The create agent generates steps for the email input, password input, login button click, and the display-name verification.

### Step 4 — attach the columns

In the platform UI (or via the API), find each generated step and attach:

- The email input step → read `email` column into its entered value
- The password input step → read `password` column into its entered value
- The display-name verification → read `expected_display` column into its expected text

### Step 5 — create a TDM-mode orchestration

Create a new orchestration of type "TDM" (row-by-row iteration), point it at the login test you just created, and select the `login-test-users` datasource. Choose **Sequential** execution for safety with single-use-style data.

### Step 6 — set iteration count to 5

The orchestration starts at row 1 and iterates 5 rows.

### Step 7 — run it

The orchestration executes:
- Run 1: Alice Wilson's credentials
- Run 2: Bob Johnson's credentials
- Run 3: Carol Davis's credentials
- Run 4: Dan Martinez's credentials
- Run 5: Eve Thompson's credentials

Each run logs in with a different user and verifies the correct display name appears on the dashboard.

The remaining 15 rows in the datasource are untouched, available for future runs.

### Step 8 (optional) — add audit write-back

To track which row was used at what time, add a `last_used_at` column to the datasource and attach it as a `write` column on the verification step. Add a custom-JavaScript step earlier in the test to capture the current timestamp into a local variable named `now`, then have the write-back step use that captured value.

After each run, the corresponding row's `last_used_at` is stamped — easy to see which rows have been exercised.

---

## 13. TDM decision matrix

| You want to... | Use this |
|---|---|
| Run the same test with N different inputs | TDM datasource + TDM orchestration |
| Share a base URL across all tests | Project variable |
| Pass a runtime value from test A to test B in an orchestration | two steps — verify-capture to a local, then a custom-JavaScript step writes it into a pre-created project variable (SKILL.md #30, `fundamentals.md` §4 (b), `specialty-steps.md` §5) |
| Generate unique emails per run in a signup flow | an inline "random email on the functionizeapp.com domain" intent |
| Remove TDM from one test only | Per-step or per-test detach (do NOT delete the datasource) |
| Permanently retire a datasource | Delete it — but be aware ALL tests referencing it lose their attachments |
