# Native Functionize capabilities — the full tool surface to prefer over manual workarounds

Load this when you want to know **whether Functionize already does a thing natively** before reaching for a hand-written custom-code step or a manual workaround. It is the menu behind the "Prefer the platform's native capability" rule in `SKILL.md`.

**You write English intent — never syntax.** Describe what you want ("upload the attached file", "wait for the OTP text", "generate a random email on the functionizeapp.com domain", "drag the card into the Done column") and the platform produces the step. This catalog exists so you know a native capability **exists** and phrase your intent to use it instead of hand-rolling it; the action names below are for **recognition**, not something you type into a prompt.

Almost the entire *interaction* library — clicks, waits, verifications, navigation, flow — is **generatable from natural language**. What is **not**: the two actions called out under "Editor-only", and any data value that must **look realistic** (a person's name, an address, a company, a line of prose). Only *structural* data (strings, numbers, email, phone, date of birth) generates natively; realistic values take a random string, a custom-code step, or a datasource (see "Generated values").

## Create-agent boundary — what you CANNOT get from a prompt

Two native capabilities are **platform/editor-only** — the create agent will not produce them from prompt text; a user adds them in the test editor if needed:

| Capability | What it is |
|---|---|
| File-viewer tool | Opens the Functionize file-viewer (new tab) for manual file operations. |
| Mid-run configuration | Changes proxy / mutual-TLS / other test settings mid-run — a power-user action. |

Everything else below is generated from natural-language intent. The one broad exception is **realistic data values** — under "Generated values", only structural data generates natively; realistic values take a random string, a custom-code step, or a datasource.

## UI interaction — the agent picks these from intent

You almost never think about these individually; describe the user action and the agent selects the right one:

| You want to… | Phrase it as |
|---|---|
| Click / double-click / right-click | "Click / Double-click / Right-click the …" |
| Hover to reveal a menu | "Hover over the … menu" |
| Type into a field | "Enter … in the … field" |
| Press a key / submit after typing | "Press Enter" — prefer this over clicking a Search/Submit button after typing (see `fundamentals.md` "Submitting after typing"); "Press the Down arrow" |
| Choose from a dropdown / radio / checkbox | "Select … from the … dropdown", "Choose the … radio", "Check the … box" (see `fundamentals.md` "Selecting from lists") |
| Select text on the page | "Select the text in the … " |
| Drag and drop | "Drag the … onto the …" (native — do **not** hand-roll a custom-code mouse sequence) |
| Sign on a signature pad | "Sign the document on the signature canvas" (native e-signature) |
| Scroll / resize the window | "Scroll to the …", "Resize the window to …" |

## Navigation & waits

| Capability | Phrase it as | Notes |
|---|---|---|
| Navigate / refresh | "Navigate to …", "Refresh the page" | Prefer a stable deep-link over on-site search (SKILL.md #15). |
| Browser history back / forward | "Click the browser **Back** button and verify the previous page", "Click the browser **Forward** button" | history navigation — distinct from re-navigating to a URL |
| Smart wait (condition-bounded) | "Wait **for** the results to load", "Wait **up to** 30s for the banner to clear" | A condition-bounded wait, not a fixed sleep — see `fundamentals.md`, "Waits and synchronization". Never write a bare "wait 30 seconds." |

## Verification

| Capability | Phrase it as | Where |
|---|---|---|
| Assert element / text / property | "Verify the … shows …" | `fundamentals.md` verification section |
| Assert something is **absent** | "Verify the error message is **not** on the page" | native negative verification |
| Assert a captured value | "Verify the order id on the confirmation page equals the `orderId` variable" (set at checkout) | against the named variable you set earlier — never a loose step-reference, never a literal you can't know at authoring time |
| Visual full-page / region baseline | "Visually verify the … page / region" | `templates.md` Template 8 — region-scope it away from volatile data (SKILL.md #16, #20) |

Place verifications at outcome moments only, and verify **durable state, never transient toasts** (SKILL.md style rules).

## Flow control & variables

| Capability | Phrase it as | Notes |
|---|---|---|
| Conditional | "If you see X, do Y" — its own line; no "otherwise" (a two-way is two separate lines) | a standalone conditional that runs only when the condition holds; for vague or variant UI, describe it as a "handle it" in English instead (`fundamentals.md` §3) |
| Loop over items / iterations | "For each row in the results, …" | native looping; for data-driven iteration over a datasource see `tdm.md` |
| Repeat until a terminal state | "Delete each row until the empty-state message appears", "Mark each notification read until the unread badge disappears" | a condition-bounded consume loop — pair with a verify on the terminal state; differs from a fixed-set "for each" |
| Capture a value for later | "Capture the order id and set it as a local variable named `orderId`; later, enter the `orderId` variable when you open the order" | **capture once, reuse the capture** (SKILL.md #17). A native capture only ever creates a **local** variable — reuse it later *in the same test* by the name you gave. Handing a captured value to **another** test is not a one-line `set` into project scope; it is the two-step project-variable write (SKILL.md #30 / `fundamentals.md` §4 (b)). A mis-named reuse fails **quietly** — the step only *warns* — so verify downstream that the field actually got the value |
| Use a stored variable | "Navigate to the base URL from the preset variable named `app_base_url`" | **name the variable in words** — confirm it exists before referencing (SKILL.md #21). A stored value is referenced, never inlined |
| Use a masked preset variable for a secret | "Enter the password from the masked preset variable named `admin_password`" | a human-set secret → a **masked** preset value (encrypted, hidden in logs/screenshots/artifacts); reference by name, never inline the value. Full scope model in `fundamentals.md` §4 |

## Context switching (tabs, iframes, windows)

"Open … in a new tab" / "Switch back to the … tab" switches context natively. **Iframe-embedded fields (rich-text editors, embedded forms) are handled for you** — describe the field by its visible purpose; you rarely need to name the frame. **Cross-origin payment widgets are the exception** — name the frame by purpose (see `specialty-steps.md` §4, iframe subsection).

## Dialogs

Native handling for JavaScript alert / confirm / prompt dialogs. Phrase as "Accept the confirmation dialog" / "Dismiss the alert" / "Enter … in the browser prompt." No custom code needed.

The **browser's own print dialog** is the same kind of JS-blocking dialog and is handled natively too. However it opens — `window.print()`, the Ctrl/Cmd+P shortcut, or a page button that opens it — phrase the intent: "Accept the print dialog", "Dismiss the print dialog", "Click Print and dismiss the print dialog." Accept or dismiss it as a dialog; do **not** stub or override `window.print()`, and do not treat it as an unsupported, OS-level surface outside the DOM.

**Disambiguate by what actually happens on screen, not the button's label.** A "Print" or "Print PDF" control that instead **generates a document** — triggering a download, or opening a generated PDF in a viewer tab — is the file path, not this dialog: handle it as a downloaded file (`specialty-steps.md` §3). An **in-page print-preview overlay** — rendered in the page, not the browser's own dialog — is ordinary page UI: click and verify it like any other element.

## Browser / session state

| Capability | Phrase it as | Caution |
|---|---|---|
| Set a cookie | "Set the session cookie from the masked preset variable named `session_token`" | seeds auth state and skips a slow login. A session token **is a secret** — name the masked variable, never inline the value (SKILL.md *Credentials & secrets*). |
| Set local / session storage | "Set the local-storage key `auth_state` from the masked preset variable named `session_token`" | same secret-handling caution |

Seeding state by cookie/storage is an optimization; when the test's *purpose* is to prove login works, exercise the real login instead.

## Generated values — native for structural data, custom-code or a datasource for the rest

**A generated value beats a hardcoded one** — a literal date ages and breaks across locales, and a static "random" value collides across parallel runs. So when a field needs a fresh or unique value, **describe the value you want in plain English** and the platform generates it; you never write a generator token, only the shape, length, format, and any constraint as intent.

**What generates natively is *structural* — a value defined only by its shape, length, or format**, even a realistic-looking one: a random string, a number, an email, a phone number, a date of birth. **A value drawn from an open-ended real-world vocabulary has no native generator** — a person's name, a street address, a company, a line of prose. A native "random company name" (or city, job title, address, sentence, …) has no built-in to satisfy it. Those values take one of the paths under "No native generator" below.

**If a generated value must appear in more than one step or field (e.g. a password and its confirm-password field), capture it into a local variable first — random data regenerates fresh on every reference.** Generate an email, then verify it on a confirmation screen: capture it after the first step and reference that variable in every later step (the `capture-before-reuse` anti-pattern — full rule in `fundamentals.md` §5 (i)).

### Natively generated — structural values

**Strings & numbers**

| Data type | Produces | Example phrasings |
|---|---|---|
| Random alphanumeric string | letters + numbers, of a stated length — also the right tool for a **username** or a **password** | *"Enter a randomly generated alphanumeric string into the reference code field"* / *"Fill the password field with a randomly generated 12-character alphanumeric string"* |
| Random numeric string | digits kept as text — also the right tool for a **ZIP code**, an account number, an unformatted SSN, or a **CVV** (any 3–4-digit value) | *"Enter a randomly generated 10-digit numeric string into the account number field"* / *"Fill the ZIP field with a randomly generated 5-digit numeric string"* |
| Random number | an integer of a stated digit count | *"Enter a randomly generated 6-digit number into the PIN field"* / *"Fill the order quantity field with a randomly generated number"* |
| Number in a range | a value between a stated min and max | *"Enter a randomly generated number between 100 and 999 into the amount field"* / *"Fill the age field with a randomly generated number between 18 and 65"* |

**Contact & identity**

| Data type | Produces | Example phrasings |
|---|---|---|
| Random email address | a unique email address (pin the domain — see below) | *"Enter a randomly generated email address into the registration form"* / *"Fill the email field with a randomly generated email address"* |
| Random phone number | a phone number, optional format mask | *"Enter a randomly generated phone number into the contact field"* / *"Fill the mobile number field with a randomly generated phone number in the format XXX-XXX-XXXX"* |
| Random date of birth | a DOB with a configurable age range and format | *"Fill the Date of Birth field with a randomly generated date of birth for someone between 21 and 65 years old"* / *"Enter a randomly generated date of birth into the applicant information form"* |

### No native generator — use a random string, a custom-code step, or a datasource

Names, addresses, businesses, and free text only *look* generatable; none has a built-in generator. Each value instead takes one of two approaches — a quick random string, or a realistic value from a custom-code step or a datasource:

- **Quick — when realism doesn't matter (the common case).** Put a **random string** in the field — *"Enter a randomly generated 8-character alphanumeric string into the first name field."* Unique every run, but it reads like `kW7mQp2x`, not a real value. Most forms never check, so this is the default.
- **Realistic — when the value must look real** (the field validates it, or a downstream system expects real-looking data). Use a **custom-code step** that picks from an inline list and stores it in a local variable, then reference that variable when you fill the field — or draw a row from a **TDM datasource** (`tdm.md`):
  - *"Run a custom-code step that picks a random first name from an inline list of realistic first names — such as Alice, Brian, Clara, David, Elena — and store it in a local variable named firstName"*
  - *"Run a custom-code step that picks a random company name from an inline list — such as Northwind Trading, Acme Labs, Belfour & Co — and store it in a local variable named company"*
  - *"Enter the firstName variable into the first name field and the company variable into the organization field"*

  The custom-code output lands in a variable and every field references it — the same capture-before-reuse discipline as any generated value (`fundamentals.md` §5 (i)). The pattern is identical for a **street address, city, state, country, job title, department**, or a **sentence / paragraph / block of prose**.

**Formatted values** — a *formatted* SSN with dashes (XXX-XX-XXXX), a UUID, an IPv4 address, or a URL — also have no generator: the exact format has to be assembled, so use a **custom-code step** that builds the string into a variable (a raw numeric or alphanumeric string won't satisfy the format). A plain, unformatted SSN is just a numeric string (above). A **credit-card expiry** is a relative future date — state it that way in a stated format (see Dates below).

**Credit-card number** — never enter a real card, and there is **no synthetic-card generator**. A payment flow needs a *designated* test card (a provider's published test number — still never a real card), supplied from a named **masked variable** like any other checkout secret.

A couple of constraints don't fit a table cell:
- **An email a later step reads must be on `functionizeapp.com`.** The built-in reader can only open a `functionizeapp.com` inbox, so pin the domain — *"generate a random email on the functionizeapp.com domain"* — whenever a later step opens that email (2FA, confirmation); omit it and the "open the email" step finds nothing (`specialty-steps.md` §4). If in any doubt a later step reads it, pin the domain anyway.
- **Ask for a numeric *string*, not a number, when leading zeros matter** — ZIP codes, account numbers: *"a randomly generated 9-digit numeric string"* keeps a leading zero that a plain number would drop or render in exponent form.

**Describe the field as a generated value.** Phrase it in English — *"a random 8-character string"* — so the platform produces a fresh value on each run rather than a fixed one.

**Dates — state them as relative intent with a format, and prefer generated over a literal:**
- **Typed (non-calendar) date field** → state the output format so it isn't guessed: *"enter today's date as MM/DD/YY"*, *"enter a date 30 days from today in MM/DD/YYYY format"* — the agent computes and types it (`fundamentals.md` #13).
- **Calendar-picker widget** → *"set the delivery date to 30 days from today using the date picker"* — the agent drives the picker (`fundamentals.md` #13).
- Reserve a fixed literal date for when that exact historical date is the point of the test.

## Specialty surfaces (full how-to in the named reference)

| Capability | Phrase it as / native path | Where |
|---|---|---|
| Outbound HTTP / REST / GraphQL | "Call `POST /orders` with …" — a native API call, always preferred over hand-rolled `fetch()` (CSP/CORS-blocked) | `specialty-steps.md` §1 |
| Database query / verification | "Run a database query to …" | `specialty-steps.md` §2 |
| File upload / download | "Upload the attached file …", "Wait for the download to complete" — name a **real** file source (SKILL.md #23) | `specialty-steps.md` §3 |
| Email-driven flows (confirmation, 2FA) | "Open the email reader and wait for the confirmation email" — `functionizeapp.com` addresses only | `specialty-steps.md` §4 |
| SMS / text-message OTP | "Open the SMS reader and wait for the OTP" (needs a provisioned account phone number — resolve before referencing, SKILL.md #21) | `specialty-steps.md` §4, SMS subsection |
| Custom JavaScript in page context | "Run custom JavaScript that …" — the **fallback** when no native capability covers the case | `specialty-steps.md` §5 |
| Server-side logic, file parsing, Postman/cURL, multi-language code | an **Extension** (Node.js / Python / Go / Java; Postman/cURL imports **added in the UI — not generated from prompt text**) | `specialty-steps.md` §1, §3 |
| Ambiguous runtime decision ("handle whatever popup appears") | describe the decision in English and let the platform's runtime AI resolve it at run time | `fundamentals.md` §3 |

## Not standalone capabilities

**Components** are reusable stored step-sequence blocks (cite a reused one by its exact full name — SKILL.md component rule). **Extensions** are reusable custom code modules at runtime hooks — API calls, file parsing, integrations. They are **different platform concepts**; never describe a component as "also called an extension." Neither is something you phrase as a single native action.
