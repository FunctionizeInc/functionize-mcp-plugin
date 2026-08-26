# Specialty Steps — API, Database, Files, Email, Custom JavaScript

Load this reference when the user is writing prompts that need to step outside the basic UI-interaction surface — calling APIs, querying databases, uploading or downloading files, working with email-driven flows (verification codes, 2FA), or asking for custom JavaScript logic.

Everything here is written as **intent** — you describe what you want and the platform generates the right step. You never write a value token, and you never name the underlying step type.

---

## 1. API call prompts

Describe a **pure HTTP API invocation** — no DOM element interaction, no page rendering — and you get a native API-call step; reference a button, field, or page element instead and you get UI steps.

### What phrasing triggers an API call vs UI

**Triggers an API call:**
- "Call the `POST /api/orders` endpoint..."
- "Send a `GET` request to `https://api.example.com/users`..."
- "Hit the `/auth/token` API with..."
- "Make an API request to..."
- "Invoke the `/api/products` endpoint with..."

**Triggers UI steps (click, input, navigate):**
- "Click the Submit button"
- "Fill in the order form..."
- "Navigate to the Reports page"

The distinction: **HTTP method + endpoint path** → an API call. **Page element or navigation** → a UI step.

### Full API call with auth, body, and response capture in one line

```
Call POST https://api.example.com/orders with headers Authorization: Bearer using the token from the masked preset variable named `api_token` and Content-Type: application/json, and a body that sets customerId to the `customerId` variable and items to `[{"sku": "TSHIRT-001", "qty": 1}]`; verify the response's orderId is not empty and set it as a local variable named `orderId`
```

A response field you capture and then verify, hand off, or substitute into a URL path or query is a runtime read: guard its shape **in the same instruction** (verify it is not empty, then capture) — an empty value dropped into a path silently hits the collection endpoint rather than failing loudly. The when-to-guard rule and the operator palette live in `fundamentals.md` §4 (a). The request-shape variants below omit the guard only to keep the focus on request shape.

### Query parameters

```
Send a GET request to https://api.example.com/search with query parameters q = the searchTerm variable and limit = 20, and capture the first result's id and set it as a local variable named firstResultId
```

### REST vs GraphQL — same step, different body

Both produce a native API call. The body shape differs:

**REST:**
```
Call POST https://api.example.com/orders with body {"product": "SKU-123", "quantity": 1} and capture the response orderId and set it as a local variable named orderId
```

**GraphQL:**
```
Call POST https://api.example.com/graphql with body {"query": "mutation { createOrder(input: {product: \"SKU-123\", quantity: 1}) { orderId } }"} and capture the response data.createOrder.orderId and set it as a local variable named orderId
```

No special keyword for GraphQL — including `query` or `mutation` in the body is enough.

### Mid-test API call patterns

**Setup pattern (API first, then UI)** — the API call seeds backend state before the UI exercises it:

```
Call POST https://api.example.com/seed with body {"scenario": "checkout_flow", "products": ["SKU-001", "SKU-002"]} and capture the response sessionId and set it as a local variable named sessionId
Navigate to https://app.example.com/checkout, passing the sessionId variable as the session query parameter
Verify the cart shows 2 items
Click the Checkout button
```

**Verification pattern (UI first, then API check):**

```
Navigate to https://app.example.com/admin
Create a new user with a random 8-character string as the first name, last name "FZE-" followed by a random 6-character string, and a random email on the functionizeapp.com domain, and verify the userId from the confirmation page is not empty and set it as a local variable named newUserId
Call GET on the https://api.example.com/users endpoint for the user id in the newUserId variable, and verify the response status is 200 and the response body has active: true
```

"Verify the response status" and "verify the response body has" both translate into assertions on the API response.

### Error handling — retry on 5xx

```
Call POST https://api.example.com/orders with body {"product": "SKU-123"} and retry up to 3 times with a 2-second interval if the response status is 5xx
```

For more sophisticated error handling (branching on 404 to skip steps, handling 409 conflicts with a different endpoint), you need an Extension — a custom cloud function with full error-handling logic.

### When to prefer a native API call over custom JavaScript for HTTP

**Always use a native API call for outbound HTTP work.** Custom-JavaScript HTTP via `fetch()` is CSP/CORS-blocked and will silently fail on most production sites; the native API call is not.

### Postman collections and cURL

A **Postman collection or cURL command** runs via an **Extension** — but **the create agent does not generate one from a prompt**, and you cannot reference a collection inline in prompt text; it is added in the UI. For prompt-driven API work inside a generated test, describe the call directly as a native API call. When a test genuinely needs a Postman collection or cURL command, add the built-in cURL / Postman-collection Extension action in the UI (see the `functionize` skill's `advanced-features.md` § Extensions).

---

## 2. Database test prompts

Describe a **database query** — `SELECT`, `INSERT`, `UPDATE`, `DELETE` — and you get a database-query step.

### What phrasing triggers a database query

- "Run a database query to..."
- "Execute a SQL SELECT..."
- "Insert a row into the database..."
- "Use the DB Explorer to..."

### Connection management

The database connection is configured in the platform, not in the prompt. Reference it by name — and the name must match what's configured:

```
Query the "staging-db" database: SELECT id FROM orders WHERE customer_email = the email variable
```

### Query execution with verification

```
Run a database query to find the order by the captured customer email (the email variable) and verify the status column equals COMPLETE
```

Note: querying by a value like an email is safe only when that value uniquely identifies the row your test owns (a freshly seeded or captured key). If the key is shared and could match multiple rows, query by the captured primary id instead — selecting "the order for this email" when several exist is the isolation-contract trap (SKILL.md #20).

More explicit SQL variant:
```
Query the "staging-db" database: SELECT id, status, total FROM orders WHERE id = the orderId variable, and verify the status column equals COMPLETE
```

You can describe the query in natural language and let the agent construct the SQL, or paste the SQL verbatim.

### Assertions against query results

| Assertion type | Prompt phrasing |
|---|---|
| Single-row value | "...and verify the status column equals COMPLETE" |
| Count | "...verify the cnt column is 0" |
| Empty result | "...verify the result set is empty" |
| Multi-row uniformity | "...verify every row's status column equals SHIPPED" |

### Data seeding and cleanup

**Pre-test seeding:**
```
Run a database query to INSERT INTO users (name, email, role) VALUES (a random 8-character string, a random email on the functionizeapp.com domain, 'customer') and capture the generated id and set it as a local variable named seededUserId
Navigate to the https://app.example.com/admin/users/ page for the user id in the seededUserId variable
Verify the user profile page loads with role "customer"
```

**Post-test cleanup:**
```
Run a database query to DELETE FROM users WHERE id = the seededUserId variable
Verify the deletion was successful by running a SELECT on the same id and confirming the result set is empty
```

Clean up seeded rows to prevent data pollution across runs in shared environments.

### When to prefer a native API call over a database query

| Use a database query | Use a native API call |
|---|---|
| Direct DB state verification ("did this record actually land?") | Application-layer verification via REST endpoint |
| Seeding data that has no API endpoint | Seeding through the application's public API |
| Debugging data corruption / raw DB state | Testing API contract, auth, rate limiting |
| No API exists for the operation | The API is the system under test |

**Rule of thumb:** if there's an API wrapping the same operation, prefer the API call — it tests the app as real clients interact with it. Use a database query to bypass the app layer for seeding, cleanup, or raw state verification.

### Cross-step UI → DB comparison

```
Create a new order with product "SKU-001" and capture the confirmation order ID from the page and set it as a local variable named orderId
Run a database query to SELECT id, status FROM orders WHERE id = the orderId variable and verify the id column equals the orderId variable and the status column equals CONFIRMED
```

The double-check (query by the captured id AND verify the id column equals it) guards against queries returning wrong rows due to SQL issues.

---

## 3. File uploads, downloads, and verification

### An upload needs a real, provisioned file — never a bare "upload a file"

**An upload step needs a real file to attach — name one; never write a bare "upload a file."** With nothing named, a bare *"open the File Upload page and upload a file"* line has no real file to use, so it falls back to whatever sits on the runner — and the step can still **generate and run green** without your test file ever being uploaded, so the "upload" exercises nothing.

So **before writing an upload instruction, confirm a real file source exists and name it** — one of the three sources below. When the source is a datasource fixture, it is a **project-scoped resource: resolve it before you reference it (SKILL.md #21)** — if the target project has no such fixture, that is a *missing fixture*; STOP and tell the user to add one (or attach the file at create time, or have the test download/generate it first). Never emit a bare "upload a file" with no real source named.

Because a no-source upload still passes generation, **confirm it after generation too, not just at author time:** open the generated upload step and check its source is the file you provided — the attached file, your named fixture, or the downloaded file — **not** an absolute path on the runner's own filesystem. **A file upload is often realized as a plain text entry into the file input field, not a dedicated upload step** — so look at that step's value. An absolute runner-filesystem path in the file input is the signature to catch — a step whose value is a stray system file on the runner rather than the test file you provisioned.

### Upload — three real sources

**1. A file attached at create time — the most direct for a one-off file.** Hand Functionize the actual file when the test is created: the attach control in Studio, or the file-upload option on `fze test create` (check `fze test create --help` for the exact flag). The file binds to the upload step (no datasource needed) and is replayed at runtime. Reference it in the instruction as *the attached file*:
```
Upload the attached file to the import form, then verify the chosen file name is shown
```
For a file that must be **reused across tests or survive re-authoring**, prefer a datasource fixture (#2). (Attaching is done in Studio or via the CLI; the agent-session tools take text only, so when you're driving a session, attach via Studio/CLI first, then reference *the attached file*.)

**2. A file pre-loaded as a Functionize datasource fixture** — for data-driven or reusable files:
```
Upload the CSV file from the datasource to the import form
```
The file comes from your project's datasource storage (see `tdm.md` for how to upload datasources). Note: a datasource column that holds only a *URL* to a file is **not** a stored file — consuming that needs an Extension to fetch it, so it is not a native upload source.

**3. A file produced earlier in the same run (download → re-upload):**
```
Click the Export button and download the CSV
Upload the downloaded CSV file to the Import form
```
The download step captures the file; the next upload step references it.

All three produce an upload step; only where the file comes from differs.

**When the standard upload can't reach the input.** If the file input is hidden, custom-styled, or JavaScript-driven so the normal upload can't attach to it, Functionize has a **JavaScript-mediated upload variant** — phrase it as *"upload the file using JavaScript"* and the agent falls back to it. The file-source rules above are unchanged; only the attach mechanism differs.

### When you don't have a file yet — the provisioning ladder

The three sources above assume a file already exists somewhere. When the test plan needs an upload but no file is in hand, work this ladder before writing the step — and **never** emit a bare "upload a file" to paper over the gap:

1. **Prefer a source that needs no external file at all.** If the flow can **download then re-upload** (source #3), or the project already holds a suitable **datasource fixture** (source #2), use it — there is nothing to transfer.
2. **Otherwise ask the user to provide the file, and route it to Functionize by whatever path this environment supports.** If you can transfer/attach a file to Functionize from where you are running, do that. If you can't — e.g. you are driving the text-only agent-session tools, which take text only and cannot attach a file — the user attaches it themselves (the Studio attach control, or the file-upload option on `fze test create`, source #1), and you reference it in the prompt as *the attached file*.
3. **If the user has no suitable file, offer to create one — only if you can generate files in this environment.** Ask the **file type and parameters** (format, approximate size, row/column layout, sample content), generate it, then route it via the same attach path as step 2 (or hand it to the user to attach). A generated file still has to reach Functionize by some path — creating it does not bypass step 2's transfer question.
4. **If neither a transfer path nor file-creation is available from here, the upload is unsupported without a manual step.** Say so plainly: the user adds the file on the Functionize platform (Studio attach control / CLI / a datasource) and pastes the prompt themselves. STOP rather than ship a placeholder upload — a no-source upload still **generates and runs green** (the bare-upload trap at the top of this section).

### Download — capture filename and verify metadata

```
Click the Export button
Wait for the CSV download to complete and capture the filename and set it as a local variable named downloadedFile
```

With native verification:
```
Click the Export button
Wait for the download to complete and verify the downloaded file has a .csv extension and is larger than 0 bytes
```

### What's natively verifiable vs Extension-required

**Native (the download step handles these):**

| Check | Phrasing |
|---|---|
| File existence | "...verify the download completed" |
| Extension | "...verify the file has a .csv extension" |
| Non-zero size | "...verify the file is larger than 0 bytes" |
| Filename pattern | "...verify the filename contains 'report'" |

**Requires an Extension (custom server-side code):**

| Need | Why an Extension |
|---|---|
| Verify CSV has 15 rows | Must parse file contents |
| Check column "status" contains only "COMPLETE" | Must read and analyze structured data |
| Compare downloaded CSV row count against a DB query result | Cross-source comparison |
| Verify JSON structure matches a schema | Parsing + validation |
| Extract a specific cell value from row 3, column 5 | Must parse the file programmatically |

**The boundary:** the native download verifies **file metadata** (name, extension, size, existence). Anything that requires **reading inside the file** needs an Extension.

### Capturing data from a downloaded file — needs an Extension

There is no native "extract cell B3 from the downloaded CSV" — it needs an Extension, referenced by name in the prompt:

```
Click Export and wait for the report to download
Run the "extract-csv-cell" extension to read the value from the downloaded file's column "orderId" row 1, verify it is not empty, and set it as a local variable named extractedOrderId
Verify the extractedOrderId variable appears on the confirmation page
```

### Generating unique filenames for parallel-safe upload

Static filenames collide across parallel runs. Use a generated value:

```
Enter a random 8-character string followed by "-import.csv" into the filename field
Click the Upload button
```

Or combine with a meaningful prefix:
```
Set the filename to "order-import-" followed by a random 6-character string and ".csv"
Attach the CSV file from the datasource
Click Submit
```

### Summary — file capability matrix

| Capability | Native | Extension required |
|---|---|---|
| Upload a file attached at create time (Studio / CLI) | ✅ | |
| Upload a static file from datasource | ✅ | |
| Click to trigger a download | ✅ | |
| Verify filename, extension, size | ✅ | |
| Re-upload a just-downloaded file | ✅ | |
| Unique filenames with random strings | ✅ | |
| Parse CSV contents (row count, column values) | | ✅ |
| Verify structured data inside a file | | ✅ |
| Extract a value from a file for later use | | ✅ |
| Compare file contents against DB query results | | ✅ |
| Validate JSON/XML/XLSX internal structure | | ✅ |

---

## 4. Email- and SMS-driven test flows

### Phrasing for 2FA login flow

```
Log in at https://app.example.com/login with the email from the preset variable named test_email and the password from the masked preset variable named test_password
Open the email reader in a new tab and wait for the 2FA email
Verify the verification-code element is not empty and capture it as verificationCode
Switch back to the application tab
Enter the verificationCode variable and submit
Verify the dashboard loads
```

The email reader waits for the matching email but does **not** automatically capture the code — add an explicit step to capture it, then enter that value into the app's code field. **Verify-capture it** (as the example does): *verify the verification-code element is not empty and capture it as [name]* — the same native shape used for any element value (`fundamentals.md` §4 (a)); the reader exposes the code as a discrete, targetable element. **Fall back to the custom-JavaScript parse only when a particular email embeds the code mid-sentence in free-text prose** with no distinct element (the *Extracting structured data from email body* table below).

**Never phrase this as *"read the code from the email and enter it."*** That ambiguous form can produce a one-time literal in place of a live read (anti-pattern #27, the snapshot trap). Always **verify-capture the code into a named local variable, then enter the stored variable**, exactly as the example above does.

Reference the captured variable in the enter-code step, not a re-typed value (anti-pattern #17).

### The `@functionizeapp.com` constraint

The built-in email reader can only receive emails sent to **`@functionizeapp.com`** addresses. If you write:

```
Register with email myuser@gmail.com, then open the email reader and verify the confirmation email arrives
```

A non-`@functionizeapp.com` address won't be caught up front — the email-reader step simply times out at runtime.

**Always use** a `@functionizeapp.com` address with a random local-part for any test that needs to receive a confirmation, verification, or 2FA email — describe it as *"a random email on the functionizeapp.com domain"*. It's readable in test reports and uniquely scoped per run.

```
Register with a random email on the functionizeapp.com domain
```

If you need to test email delivery to a real Gmail / Outlook / corporate inbox (because the *production* email flow uses those domains), that's outside the native reader's capability. Reading such an inbox isn't a built-in feature — it would require a custom Extension that calls the provider's REST API (Gmail / MS Graph) or IMAP, which you'd build and confirm for your environment before relying on it.

### Unique email addresses per run

```
Register a new account with a random email on the functionizeapp.com domain, a random 8-character username, and the password from the masked preset variable named signup_password
Open the email reader in a new tab and wait for the confirmation email
Click the confirmation link in the email
```

A generated random email is unique every run — no collision risk.

### Verifying email delivery — subject and timing

```
Register a new account with a random email on the functionizeapp.com domain and the password from the masked preset variable named signup_password
Open the email reader in a new tab and wait up to 60 seconds for a welcome email with subject containing "Welcome to Acme"
Verify the email body contains the registered username
```

The email reader supports timeout and subject filtering natively. For strict delivery SLA checking:

```
Open the email reader and verify a welcome email arrives within 30 seconds with subject "Welcome to Acme"
```

### Extracting structured data from email body

**Native — the reader exposes these as discrete, targetable elements** (verify-capture them, `fundamentals.md` §4 (a)):

| Data | How |
|---|---|
| Verification code, *if a discrete element* | **verify-capture** it — *verify the verification-code element is not empty and capture it as [name]* (the uniform shape, `fundamentals.md` §4 (a)); if the code sits inside a sentence of prose, use the custom-JavaScript row below |
| Click a link | `Click the confirmation link in the email body` |
| Subject line check | `...with subject containing "Welcome"` |
| Body text presence | `...verify the email body contains "Your order is confirmed"` |

**Custom JavaScript — when the value can't be a discrete-element verify** (embedded in free text and must be parsed out, or several fields at once):

| Need | Why |
|---|---|
| A code or id **embedded in free-text prose** (e.g. `#ORD-\d{8}` inside a sentence) | not a discrete element — custom JavaScript parses it out of the body text |
| Extract multiple fields (order ID + amount + date) | a single JS block: parse + capture into multiple named values |

**Requires an Extension:**

| Need | Why |
|---|---|
| Parse and validate an email attachment (PDF, CSV) | Attachment binary handling needs server-side code |
| Cross-reference email body values against a database | Multi-step orchestration |
| Validate JSON structure embedded in an email | Reliable parsing + schema validation |

### Multi-tab pattern — precise phrasing

```
Log in at https://app.example.com/login with credentials from the preset variables
Open the email reader in a new tab
In the email reader tab, wait for the 2FA email and verify the verification-code element is not empty and capture it as verificationCode
Switch back to the application tab
Enter the verificationCode variable and click Verify
Verify the dashboard shows "Welcome back"
```

Key phrases:
- **"Open … in a new tab"** switches context to a new tab.
- **"In the email reader tab, …"** scopes subsequent steps to that tab (optional but clarifies intent).
- **"Switch back to the application tab"** switches context back.

### SMS / text-message OTP

For an OTP / 2FA flow delivered by **text message**, Functionize has a native **SMS reader** that mirrors the email reader — prefer it over any hand-rolled approach. Phrase it like the email flow:

```
Log in at https://app.example.com/login with credentials from the preset variables
Open the SMS reader in a new tab and wait for the OTP text message
Verify the verification-code element is not empty and capture it as otpCode
Switch back to the application tab
Enter the otpCode variable and submit
Verify the dashboard loads
```

This opens the Functionize SMS reader in a new tab, with the same tab handling as email. Like the email reader, it does **not** auto-capture the code — add the explicit **verify-capture** step and name it; the SMS reader mirrors the email reader and exposes the code as a discrete element, so verify-capture it the same way (*verify the verification-code element is not empty and capture it as [name]*). Fall back to the custom-JavaScript parse only when a message embeds the code mid-sentence in free-text prose (`fundamentals.md` §4 (a)). Put a **smart wait before the receive step** (wait *for* the message, not a fixed sleep): if the text hasn't arrived yet the reader shows zero messages and the step fails. Reference the captured variable in the enter-code step, not a re-typed value (anti-pattern #17).

**The SMS prerequisite — different from email.** Email works out of the box for any `@functionizeapp.com` address. SMS does **not** have an equivalent zero-setup convention: it requires the **account to have a provisioned phone number configured first**. That is a one-time account-level setup (the account admin, or Functionize support) — until a number exists, the SMS reader has nothing to read from. Treat the provisioned number as a project-scoped prerequisite and **resolve it before you reference it (SKILL.md #21)**: confirm the account has one before promising an SMS-OTP test; if it has none, STOP and surface that rather than shipping a test that can't receive a code.

Sending a text — to drive an inbound-SMS flow — is the sibling send-SMS step (*"Send an SMS with the confirmation code"*), and carries the same provisioned-number prerequisite.

### iframe / embedded content (payment widgets, third-party forms)

Third-party widgets — **embedded forms, rich-text editors, payment fields (Stripe, Braintree, PayPal, Adyen), some consent/chat widgets** — render inside an **iframe**: a separate document that must be entered before its fields can be touched. For most of these, **you generally do not need to name the frame or write a switch step** — describe the target by its visible purpose and the frame is entered for you (**cross-origin payment widgets are the exception — see below**):
```
Enter the release notes in the rich-text editor
Complete the embedded signup form with the email from the preset variable named test_user_email
```
A rich-text editor inside a nested or cross-origin iframe alongside sibling frames is typically handled from intent too — *"enter the text into the rich-text editor"* with no mention of frames is usually enough.

**When naming the frame still helps:** the frame's **visible purpose** ("the payment iframe", "the Stripe card frame") is a low-cost disambiguation cue — worth adding when several frames sit close together.

**Cross-origin payment widgets (Stripe, Braintree, Adyen, PayPal…) are a special case.** Such a widget can carry deliberate security or anti-automation controls, so unlike an ordinary embedded field its card inputs aren't guaranteed to be reachable out of the box — supported, but its protections may need evaluating first. So **name the frame by its visible purpose as the default** here, not only when frames crowd together; if the card field still can't be filled, treat it as something to look at rather than forcing it. Keep it intent-level — name the frame by purpose, never by an `iframe#id` / index / CSS selector, and don't hand-write the switch.

### Common email flow patterns

**Signup + confirmation email + first login:**
```
Register a new account at https://app.example.com/signup with a random 8-character username, a random email on the functionizeapp.com domain, and the password from the masked preset variable named signup_password
Open the email reader in a new tab and wait for the confirmation email
Click the confirmation link in the email
In the opened page, verify "Account confirmed" appears
Log in with the same email and password
Verify the welcome dashboard loads
```

**Admin-triggered invitation + SLA check + link capture:**
```
Log in at https://app.example.com/admin
Navigate to Users → Invite User
Enter a random email on the functionizeapp.com domain (prefix it "qa-invite-") and click Send Invitation
Open the email reader in a new tab and wait up to 45 seconds for an invitation email with subject containing "You're invited"
Verify the email body contains "Accept Invitation"
Verify the invitation link is not empty and capture it as inviteLink
```

**Password reset:**
```
Navigate to https://app.example.com/login
Click "Forgot Password"
Enter the email from the preset variable named test_email and submit
Open the email reader in a new tab
Wait for the password reset email and click the reset link
In the opened page, enter a new password — "NewPass_" followed by a random 4-character string and "!" — and confirm
Log in with the email from the preset variable named test_email and the new password
Verify the dashboard loads
```

### Email capability summary

| Capability | Native | Custom JavaScript | Extension |
|---|---|---|---|
| Open email reader, wait for email | ✅ | | |
| Filter by subject line | ✅ | | |
| Extract verification code | ✅ | | |
| Click a link in email body | ✅ | | |
| Verify body text contains string | ✅ | | |
| Regex-extract structured data | | ✅ | |
| Parse & validate email attachments | | | ✅ |
| Cross-reference email content with DB | | | ✅ |

---

## 5. When to ask for custom JavaScript

A custom-JavaScript step runs **inside the browser's window context**. Use it when no other step type can express the operation. The rest of the time, prefer native steps.

### What ONLY custom JavaScript can express — five legitimate use cases

**1. Timestamp math — relative dates for form fields:**
```
Run custom JavaScript to compute yesterday's date in YYYY-MM-DD format and set it as a local variable named yesterday
```
The built-in generated values don't compute dates relative to today. **But reach for custom JavaScript here only as a fallback** — for simply *entering* a relative date in a date field, express it as **intent** and let the agent resolve it: *"set the delivery date to 30 days from today using the date picker"*. You don't compute the date yourself. Use a custom-JavaScript-computed value when you need the **exact** date in a specific format to *also verify* later (one source of truth for both the input and the assertion), or when a stubborn custom widget won't take the intent. Cross-month navigation works on its own — a computed date is needed only when you must verify the exact value later.

**2. Hash / HMAC / token generation:**
```
Run custom JavaScript to compute an HMAC-SHA256 signature of the requestBody variable using the secret from the masked preset variable named api_secret and capture the hex result and set it as a local variable named signature
```
Cryptographic operations need JavaScript and the WebCrypto API.

**3. String-building with conditional logic:**
```
Run custom JavaScript to build a full name by joining the firstName variable, a space, and the lastName variable, applying title case and trimming whitespace, and capture the result and set it as a local variable named fullName
```
Trivial concatenation could be inline, but conditional formatting pushes it into custom JavaScript.

**4. Structured-data parsing from page text:**
```
Run custom JavaScript to parse the JSON response displayed in the <pre> element and extract the field data.items[0].trackingNumber and set it as a local variable named trackingNumber
```
Native verification checks "does text X appear?" — it cannot parse JSON, apply JSONPath, or extract nested fields.

**5. Writing to project variables at runtime:**
```
Run custom JavaScript to write the generatedToken variable into the project variable named last_generated_token
```
Only custom JavaScript can write to a project variable during a test run. Native capture steps only write to local (captured) values — so persisting a runtime value is always two steps: verify-capture it to a **local** first, then this custom-JS step writes that local into the **pre-created** project variable (`fundamentals.md` §4 (b), SKILL.md #30).

### Phrasing that triggers custom JavaScript

- "Run custom JavaScript to..."
- "Execute a custom code step to..."
- "Write a JavaScript snippet that..."

### Phrasing that might WRONGLY route to custom JavaScript

| What you might write | Result | What to write instead |
|---|---|---|
| "Calculate the cart total" | custom JavaScript generated unnecessarily | "Verify the cart total field shows the correct sum" — verify checks the displayed value |
| "Extract the order number from the confirmation page" | custom JavaScript | "Capture the order number from the confirmation message and set it as a local variable named orderId" — native capture on a verify step |
| "Generate a random email" | custom JavaScript | "a random email on the functionizeapp.com domain" — a generated value |
| "Compare two strings" | custom JavaScript | "Verify the page title equals the `expectedTitle` variable" (set earlier) — native verify |

**Rule:** if the operation is **read a value from the page**, **check equality**, or **generate a random string/number/email**, a native step can do it. Reach for custom JavaScript only when there's real computation, transformation, or conditional logic.

### What custom JavaScript CANNOT do

| Limitation | Detail |
|---|---|
| No reliable outbound HTTP | `fetch()` / `XMLHttpRequest` is subject to CSP and CORS — will fail on most production sites. Use a native API call. |
| No filesystem access | Cannot read/write local files. File work needs Extensions. |
| No `require()` / module imports | Only standard browser Web APIs. No npm packages. |
| No cross-tab access | Runs in current tab's window context only. |
| No persistent browser storage write | `localStorage` may be cleared between runs. Use project variables. |

**The sandbox:** browser window context, standard Web APIs only.

### Contrasting prompts: legitimate vs over-engineered

**Legitimate — timestamp math:**
```
Run custom JavaScript to compute the date 30 days from today in MM/DD/YYYY format and set it as a local variable named futureDate
Enter the futureDate variable into the expiration date field
```
(Right when the field takes **typed** text, or you must reuse/verify the **exact** value. For a calendar-**widget** date field, prefer intent — *"set the date to 30 days from today using the date picker"* — and the picker is navigated for you, §5 use case 1.)

**Legitimate — HMAC signing:**
```
Run custom JavaScript to compute an HMAC-SHA256 signature of the payload variable using the key from the masked preset variable named signing_key and capture the hex string and set it as a local variable named signature
Call POST https://api.example.com/webhook with headers X-Signature: the signature variable and body the payload variable
```

**Legitimate — a realistic-looking name (there is no native name generator):**
```
Run a custom-code step that picks a random first name from an inline list of realistic first names — such as Alice, Brian, Clara, David, Elena — and store it in a local variable named firstName
Enter the firstName variable into the First Name field
```
No built-in produces a person's name, so a name that must look real — a field that validates it, or a downstream system that expects real-looking data — comes from an inline list in a custom-code step, captured to a variable and referenced. When realism doesn't matter, a plain random string per field is simpler; the full two-path guidance is in `native-actions.md`.

**Over-engineered — extracting page text:**
```
Run custom JavaScript to read the order confirmation number from the page and set it as a local variable named orderId
```
Should just be: "Capture the order confirmation number from the page and set it as a local variable named orderId".

**Over-engineered — random email:**
```
Run custom JavaScript to generate a random email address and set it as a local variable named email
```
Should just be: "Register with a random email on the functionizeapp.com domain".

**Legitimate — conditional DOM check that influences later steps:**
```
Run custom JavaScript to check if the element with class "error-banner" is visible on the page. Capture "true" if visible, otherwise "false" and set it as a local variable named hasError.
```
Later steps can conditionally branch using the hasError variable.

### Quick reference

| Need | Use |
|---|---|
| Timestamp math, date arithmetic | custom JavaScript |
| Hash, HMAC, encryption | custom JavaScript |
| String concatenation with formatting | custom JavaScript (or inline if trivial) |
| DOM inspection / conditional logic | custom JavaScript |
| Writing to project variables at runtime | custom JavaScript |
| Generate random values | a generated value (describe it in English) |
| Capture page text | Native capture on verify/input |
| String equality check | Native verify |
| HTTP calls outside the page | a native API call — NOT custom JavaScript |
| File I/O, npm modules, Node.js | Extension — NOT custom JavaScript |

**How a custom-JS step reports pass/fail** — its return contract, and the callback for async work — is platform mechanics the agent writes, not prompt craft you author: the detail lives in the `functionize` skill's `advanced-features.md` § Custom JavaScript steps.
