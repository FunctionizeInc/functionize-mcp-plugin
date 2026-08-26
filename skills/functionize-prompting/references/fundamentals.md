# Fundamentals — Anatomy, Verification, Conditionals, Capture, Anti-Patterns, Course-Correction

This is the core reference for everything you need to write a solid Functionize create-agent prompt. Load it when the user asks anything structural about a prompt — how to lay it out, how to phrase a verification, how to handle conditionals, how to capture and reuse data, what mistakes to avoid, or how to fix a prompt that produced a wrong step.

---

## 1. Prompt anatomy — the structural template

A Functionize create-agent prompt is a sequence of natural-language instruction lines that the create agent converts into test steps. The structure is deliberately simple: **chronological workflow order, one logical objective per line, no preamble, no postamble**.

### Section order

There is no concept of "setup" / "teardown" sections. The agent sees a linear sequence. Conventional order:

1. Authentication first (if the app requires it). Omit when the test is guest-only or starts already logged in.
2. Core workflow in chronological order.
3. A **non-asserting teardown** step comes last only when it reconciles state that outlives the run VM — a persistent-state reset for cross-run isolation. **Don't append a bare client-side logout**: the run VM is torn down after the test, so a logout clears nothing the tear-down doesn't (include logout only when logout is itself the behaviour under test). And **don't delete created data in a trailing UI step** — a trailing UI delete is *skipped whenever the flow fails earlier*, so it leaves behind exactly the residue it was meant to remove, and a test that depends on its own trailing cleanup to stay re-runnable is not isolated. Put data cleanup where it runs reliably and independently of the happy path: a dedicated API/DB teardown step, or a separate teardown test in the orchestration (§4(b)). Better still, make the test not need cleanup at all — give each run its own uniquely-named data so re-runs don't collide (the isolation contract, SKILL.md #20).

### Line length and granularity

One sentence with **one logical objective** per top-level line. Longer is acceptable when needed to disambiguate, but prefer brevity.

| Line style | When |
|---|---|
| Single-action top-level line: `Click "Place Order"` | Navigation, search, single clicks |
| Compound line with `-` sub-bullets | Multi-field forms, configuration panels |
| Top-level line with capture clause | When a later step needs a value from this step |

Pre-capture any value a later line will consume: a later line can reference an earlier value **only if that earlier line explicitly captured it**. A forward reference alone won't pull a value backward — if line 14 verifies against a total, line 8 must say "capture the total for later verification."

### Top-level lines are plain; `-` is a sub-instruction marker

A top-level instruction is a **plain line with no leading `-`** — one objective per line, and the create agent turns each into its own step. A `-` prefix means the opposite: **a line starting with `-` (or a space) is grouped under the preceding top-level instruction**, as a *sub*-step of it. So `-` is reserved for splitting **one** compound instruction into its parts — *fill this form*, then its fields — never for enumerating the workflow.

**Do not prefix every instruction with `-`.** With no plain top-level line to attach to, the dashed lines have no proper parent — so they generate as a single step instead of one each, and a whole workflow shrinks to a single step (the exact failure the grouping rule above describes). Dashes split one line into fields; plain lines carry the workflow. Keep the fields of a single record together under one lead line:

```
Fill in the shipping address form:
- First Name: use a random 6-character string
- Last Name: "FZE-" followed by a random 6-character string
- Street: 123 Test Street
- City: Springfield
- State: IL
- ZIP: 62701
- Email: the email from the preset variable named checkout_email
```

### Bullet vs prose

- Use prose for navigation, search, and single-action steps; use `-` prefixed bullets **only** for the sub-fields of a compound lead line — **structured multi-field input** (forms, configuration panels). A list of *independent* checks is **not** such a compound line — see "One check per verify step" below.
- **Never use numbered bullets** — they conflict with the step indices in the generated test.

### Repeating line items (multi-line orders, multi-product quotes/invoices)

A `-` sub-bullet block describes the fields of **one** record (the form above is one shipping address). It is **not** how you express **N repeating rows** — a multi-line sales order, a multi-product quote/invoice, a cart with several items. A single `-` block describes one record's fields, so several line items packed into it collapse to a single row — and the test then "passes" while exercising a **single-line** order, so the real risk (line-level pricing, tax, availability, totals) is never tested.

**Express each line item as its own top-level instruction**, then verify per line and on the rolled-up total:

```
Add order line 1: material "FG-100", quantity 10
Add order line 2: material "FG-200", quantity 5
Add order line 3: material "FG-300", quantity 2
Verify the order shows 3 line items
Verify line 1 net value equals unit price × 10, and capture the order net total
```

- One instruction per line keeps each row a distinct, retry-able action and lets you verify rows independently.
- **Verify the line count** (`shows 3 line items`) so a silently-dropped row fails the test instead of passing.
- **Variable / data-driven counts** → drive the lines from a **TDM line-item table** (one datasource row per line; see `tdm.md`) and iterate, rather than hardcoding N instructions. Capture the resulting count and assert it.
- This pattern is the backbone of SAP sales/purchase orders (`functionize-sap-s4hana` recipes) and Salesforce CPQ quote lines (`functionize-salesforce` business-processes) — both cross-link here.

### Blank lines

Blank lines don't signal sub-task boundaries — the create agent ignores them, so they never convey structure or grouping. Because they're inert, use them for readability: in an assembled prompt you show the user, **separate logical objectives with a blank line** so it reads as distinct blocks rather than a wall of text (short illustrative fragments needn't bother). Keep a compound line's `-` sub-fields tight under their lead — no blank line inside a single block. For an author-side section label or note, write an HTML comment, not a blank line.

### HTML comments — the only metadata channel

`<!-- ... -->` comments don't become steps. Use them for:
- Test name, expected duration, persona
- List of variables the test requires (so the human or AI maintaining the test can see at a glance)
- Forward-dependency notes — which line captures what, which line consumes it
- *Intentionally NOT asserted* — assertions the test deliberately does **not** make, each with its reason, **when** a tempting-but-wrong one exists (§5 #25)
- Anything you want preserved for the next author but not turned into a step

### Opening / closing conventions

**There are none.** The first non-blank, non-comment line is instruction 1. Don't add `Start test`, `End test`, `Test complete` — they become spurious steps.

### Annotated template — e-commerce guest checkout

```
<!--
  Test name: "Guest Checkout — Standard Product, Credit Card"
  Expected duration: ~45–60 seconds
  Persona: Anonymous first-time buyer
  Required preset variables (named, set masked in Functionize):
    checkout_email
    checkout_card
    checkout_expiry
    checkout_cvv
  Key forward-dependencies:
    Cart line captures the product NAME → verified on the confirmation page
    Payment line captures the GRAND total (after shipping) → verified on the confirmation page
-->

Search for "wireless headphones" using the store search bar

Select the first product result and add it to cart with:
- Quantity: 1
- Size: Medium (only if a size selector is shown)

Proceed to the shopping cart page

Verify the cart shows the product name and capture it, setting it as a local variable named productName

Verify the cart line total equals the product unit price times the quantity

Click "Checkout as Guest"

Fill in the shipping address form:
- First Name: use a random 6-character string
- Last Name: "FZE-" followed by a random 6-character string
- Street: 123 Test Street
- City: Springfield
- State: IL
- ZIP: 62701
- Email: the email from the preset variable named checkout_email

Click "Continue to Shipping"

Select the cheapest available shipping method and click "Continue to Payment"

On the payment page, verify the order grand total — the final amount due including shipping, not the subtotal — is not empty and set it as a local variable named grandTotal

Enter payment details:
- Card Number: the card from the masked preset variable named checkout_card
- Expiry: the value from the masked preset variable named checkout_expiry
- CVV: the value from the masked preset variable named checkout_cvv

Click "Place Order"

Verify the order confirmation page is displayed

Verify an order number is shown

Verify the confirmation shows the productName variable from the cart

Verify the grand total shown equals the grandTotal variable from the payment page
```

### Waits and synchronization

Most steps need no explicit wait — the platform waits smartly for elements and pages. When you DO need to synchronize on something asynchronous, phrase it as a **condition**, not a fixed sleep:

| Don't (fixed sleep — flaky and slow) | Do (condition-based) |
|---|---|
| `Wait 2 seconds, then click Submit` | `After the results table appears, click Submit` |
| `Wait 5 seconds for the page` | `Wait for the loading spinner to disappear` |
| `Wait 10 seconds for processing` | `Wait up to 30 seconds for the "Processing" banner to clear` |

- **Every wait needs an *anchor*: wait for the specific element your next step will use — until it is *visible* — not "the page."** "Wait for the page to load," "wait for processing to finish," "wait for the result to render" name no concrete state, so there is nothing specific to wait for. Name the exact control the next action touches and wait for it to be **visible**; you don't need to add "interactable" — the click or type step handles element-readiness itself. `Wait until the "Place Order" button is visible, then click it`. The wait and the next step then share one concrete anchor. **Exception — a control that renders *disabled until it becomes actionable*** (e.g. a submit button gated by form validation): wait until it is **visible and enabled**, since "visible" alone can pass while it's still disabled.
- **After an async action (submit, convert, generate), anchor the wait on a *persistent* element that exists once it has finished** — a result-page section header, the created record's link, the destination page's stable heading — never a transient spinner or status message. If the action ran in a **modal or overlay, wait for it to close first**, then wait for the persistent element on the page beneath. (UI form of anti-pattern #22.)
- A bare `Wait 2 seconds` is simultaneously too long (slows every run) and too short (flakes when the app is slower) — name the **state** you're waiting for instead.
- The cleanest implicit wait is the **"after X loads/appears" clause** on the *next* action (e.g. "After the search results load, click the first result") — it gates that step on the condition with no separate wait step (smell #2 in §5).
- This is the same construct as the email/SMS readers' **"wait up to N seconds for the … to arrive"** (specialty-steps) — a bounded wait on a condition, applied to UI.
- **A time *ceiling on a condition* is fine; a *bare fixed sleep* is not.** "Wait **up to** 30s **for** the banner to clear" ends the moment the condition is met (the number is just a safety cap) — that's allowed and good. "Wait 30 seconds" always burns the full 30s and still races a slower app — that's the banned fixed sleep. The test is whether a **condition** ends the wait, not whether a number appears. **For a *long or open-ended* wait — an agent generation, a full test run, a batch job — the ceiling is not just fine but *required*: with no explicit generous cap the wait expires early against the short default timeout (failing before the operation has actually finished), or stalls the run. Cap it — *"wait **up to** 10 minutes for the run to reach a terminal result."***
- If a flow is *genuinely* slow app-wide, raise the project's timeout settings rather than sprinkling waits into the prompt.

### Selecting from lists — dropdowns, radios, checkboxes, typeaheads

These are in almost every form, and each control type wants different phrasing. Match the phrasing to the control type — mis-typing is a top cause of "the step did nothing," most often phrasing a native dropdown as "type X" (a native `<select>` takes a chosen option, not typed text).

| Control | Phrase it as | Notes |
|---|---|---|
| **Native `<select>` dropdown** | `Select "United States" from the Country dropdown` | "Select … from the … dropdown" picks the option. Do **not** say "type United States" — a native `<select>` takes a chosen option, not typed text. |
| **Typeahead / combobox / autocomplete** | `In the Country field, type the full value "United States" and select the matching result` | **A decision rule tied to observable UI state** (not a count of matches). Full rule — the Tab-vs-dropdown mechanics and the duplicate-names exception — in "Searching for a record," below. |
| **Radio button** | `Choose the "Express" shipping option` / `Select the "Credit Card" radio` | One choice per group; name the option label. |
| **Checkbox** | `Check the "I agree to the terms" box` · `Uncheck "Remember me"` · `Leave "Subscribe to newsletter" as-is` | State the desired **end state** with check/uncheck — idempotent, works regardless of the default. Don't blindly "click" a box whose initial state you don't control. To **not touch** a box, say **"leave it as-is"** (not "leave it unchecked", which is ambiguous — set vs. no-op vs. assert). To **assert** a box's state, use a separate verify step. |
| **Multi-select** | `In the Tags multi-select, choose "Priority", "Billing", and "VIP"` | Name **each** value explicitly; don't write "select the tags." |

When a value won't appear, suspect a **dependent/cascading** control (the parent field gates it) — set the controlling field first (Salesforce dependent picklists, `functionize-salesforce` how-to-prompt §4, are the canonical case).

### Submitting after typing — prefer Enter over a button

After typing a value into a box, **submit by pressing Enter (Return) rather than clicking a "Search" / "Submit" / "Go" button** — unless the test specifically needs that button. Enter is more robust: it doesn't depend on locating the right control (search and submit buttons vary by label and icon, and a page often has several).
```
Enter the oppName variable in the global search box and press Enter
```
Click the button only when (a) the test is specifically exercising that button, or (b) the field doesn't submit on Enter (a multi-line textarea, or a form where Enter does nothing). A distinct action button that isn't submit-after-typing — `"Sign In"`, `"Place Order"`, `"Convert"` — is unaffected; click it.

### Searching for a record — type the full value

When you search for a record (or fill a typeahead/lookup), **type the complete value, not a fragment.** A fragment leaves a dynamically generated suggestions list — often with multiple, rotating entries — to choose from, a top source of flake (the option you want shifts position, or several match). The full value narrows toward the exact record. In a **search box**, press Enter to submit, then open the single result. In a **lookup/typeahead field**, give a **decision rule tied to observable UI state, not a count**: if the field **auto-completes to a single match**, press **Tab** to confirm (Tab, not Enter — Enter can submit the parent form); if a **dropdown list of suggestions appears instead**, select the intended one from it, because Tab would grab whichever auto-completes first. Don't type a bare fragment when the full value would resolve on its own — **but in an org you know has duplicate names, do the reverse: type a fragment to bring up the suggestions list, select the intended record, and don't Tab** (the full value could auto-complete to the wrong duplicate, and the fragment is what forces the list to appear so you can pick).

```
Type the full account name "Acme Corporation" in the global search box and press Enter, then open the single matching result; if the search results page appears instead, select the matching record from the list
```
- **Type the full value + Enter**, then **open the single result**.
- **Fallback for a non-unique value:** *"if the search results page appears instead, select the matching record from the list"* — covers an org where the value isn't unique and search lands on a list rather than the record (the §3 conditional/fallback construct).
- For a record you created, search by the **variable** you set at creation (§4) — its **unique suffix** guarantees a single match, so the fallback is belt-and-suspenders there.

### Formats and locale

State both **input** and **verify** values in the app's **displayed** format for the target environment — `"$1,234.56"`, `"25%"`, `"(415) 555-0199"`, and locale dates (`MM/DD/YYYY` vs `DD/MM/YYYY`). Cross-environment runs (a top reason to variabilize, §5(e)) break on **format, not logic**: a hardcoded `Verify the total equals "$29.99"` (an exact match) fails on a EU-locale render of `"29,99 €"` even though the amount is right.

- **Currency / numbers where the symbol or separator varies** → prefer a substring or pattern match over an exact match (e.g. a money pattern like `\$[\d,]+\.\d{2}`) — but anchor it to the digits that matter so it actually matches a real value.
- **Dates** → prefer the **date picker** over typing, phrased as a *preference with fallback* (full rule: #13); ISO (`YYYY-MM-DD`) is a good typed fallback the agent can reformat. Avoid time-of-day-sensitive `equals` across timezones (a DateTime stored UTC renders differently per user TZ).
- **Truly cross-locale** assertions → parameterize the expected value per environment in a named **non-masked preset variable** (typically one preset per environment) rather than hardcoding one locale's string.

---

## 2. Verification phrasing — say what you want checked

The create agent maps your English to the right kind of check. Pick the phrasing that produces the behavior you want.

### One check per verify step

Assert **one field or element per verification line.** Don't bundle several distinct checks into one line — `verify the record shows the name, the Stage, and the Amount` — because a bundled line generates as a **single** verify step that may not assert all three, so some go unchecked while the step still passes green. **The dashed form bundles the same way** — a `Verify … :` / `… shows:` lead-in with checks dashed beneath (*"Verify the confirmation shows:"* then *"- an order number"*, *"- the total"*) is one bundled step, not one per line; `-` marks the fields of a compound *input* (§1), never a checklist of independent verifies. Split into one plain verify line per field:
```
Verify the Opportunity Name field shows the oppName variable
Verify the Opportunity Stage field shows "Value Proposition"
Verify the Opportunity Amount field shows the oppAmount variable
```

### Phrase → what it checks

| Phrase pattern | What it checks |
|---|---|
| "verify X is `value`" / "confirm X equals `value`" / "check X reads `value`" | Exact string match |
| "verify X contains `value`" / "confirm X includes `value`" / "check the page shows `text`" | Substring match |
| "verify X matches the pattern …" / "confirm X follows the format …" | Pattern match |
| "verify X is present" / "confirm X exists" / "check X is displayed" | Presence only — no text check |
| "verify X shows the [name] variable" / "confirm X equals the [name] variable" | Compares against a value set earlier as a named local variable (never a loose step-reference — see below) |
| "visually verify X" / "X matches the baseline" / "X looks the same as [earlier]" | Screenshot comparison |
| "verify X is NOT displayed" / "confirm X is not present" / "no X appears" | Absence check |

### The verb decides whether a step asserts — only `Verify` does

The **verb** decides whether a step records a pass/fail or just synchronizes:

| Phrase pattern | Terminal-step safe? |
|---|---|
| "Verify / Confirm / Check [outcome condition]" | ✅ Yes — smart-waits for the condition, then records a hard assertion (PASS or FAIL) |
| "Wait until / Wait for [condition]" | ❌ No — synchronizes only; if the condition never appears it times out **softly** and the run still reports passed |

Both wait for the condition; only a `Verify` records whether it was met. **Terminal-step rule: the last meaningful step of every test must be a `Verify` that asserts the durable, outcome-specific result** — the authenticated sidebar / protected content (login), a created record's id or field value (CRUD), a confirmation total or status (checkout / submission), rendered structural content (marketing). Never a `Wait until` (a soft timeout the run passes straight through), a bare action (click / navigate / type), a transient toast / spinner, or a generic element that also renders on an error / redirect page. A test without a terminal `Verify` cannot fail on its own outcome — that missing assertion is what lets a flow that silently didn't complete still report passed. Full rule, mechanics, and edge cases: §5(i) #26.

### Loose vs tight presence

The key distinction is **"on the page"** vs **"the value."**

**Loose (presence + substring):**
```
Verify the confirmation message is displayed on the page
```
Checks element presence with substring matching — not locked to a single exact string. "On the page" signals broad presence.

**Tight (exact field-level match):**
```
Verify the order total reads "$29.99"
```
Checks the specific element for an exact match. The quoted value signals "I know the exact string."

**Pattern (flexible format check):**
```
Verify the order ID matches the format ORD- followed by 8 digits
```
Checks against the pattern `ORD-\d{8}`. The phrase "matches the format" is the trigger. (That's a *display* verify; for a **guarded capture** of a runtime value, use `contains` instead — see §4 (a).)

### Text matching — case and whitespace

A text verify is **case-sensitive by default**, and **internal whitespace is preserved** — `"Closed  Won"` (two spaces) does not match `"Closed Won"` (one), and `"closed won"` does not match `"Closed Won"`. Two consequences:

- When the app's casing is the thing under test, assert the exact rendered string. When it isn't, prefer a partial match on a stable fragment over an exact match that a trivial casing or spacing drift would flap — describe the intent ("verify it contains X") and the platform selects the comparison; you never hand-write the operator.
- To match **case-insensitively** on purpose, pick a case-insensitive operator — an ignore-case equality or a case-insensitive regex — rather than an exact match; say what you want and the platform selects the operator.

### Cross-step value comparison — always via a named local variable

To verify a value on the current page against a value from earlier in the same test, **set the earlier value as a named local variable and verify against that variable:**

1. Earlier: `set [the value] as a local variable named [name]` — or `capture [the value] and set it as a local variable named [name]` when reading it off the page (a value read off the page and then compared is guarded on capture — verify its shape in the same instruction, §4 (a)).
2. Later: `verify [field] shows the [name] variable` (or `equals` / `contains` / `greater than` / `less than` the [name] variable).

```
Verify the order total is not empty and set it as a local variable named orderTotal at checkout
... later ...
Verify the confirmation total equals the orderTotal variable
```

**Verify against the named variable, never a loose step-reference** ("matches the value from step 5," "the same order id as the previous step") **or a prose description of the value** ("the captured Opportunity name," "the name used"): the agent has no handle to reach back for and regenerates a fresh value, so the check lands on something that never appeared (§4(a) has the full mechanism). If no earlier line set the variable, there's nothing to compare against.

**Never hardcode a value that already appeared earlier in the same test.** If a later check must assert a value the test itself entered or produced upstream, re-stating it as a literal — `verify the Amount reads "$25,000.00"` — is a regression: it duplicates the source of truth and breaks on formatting/locale (an exact-match literal that the create agent flags as fragile). Pick by how the value originated — each routes through a **named local variable**, never a step-reference:

- **A value you entered or authored** → set it as a local variable at the point you enter it, then verify against it: `verify the Amount field shows the oppAmount variable`.
- **A value that only appeared on screen upstream** (a system-generated total, id, or timestamp) → **capture it into a local variable** at that point (guarding its shape, since it is read off the page and then compared — §4 (a)), then verify against the variable: `verify the total is not empty and set it as a local variable named orderTotal` … `verify the confirmation total equals the orderTotal variable`.

The comparison is **not limited to equality** — `equals`, `contains`, `greater than`, `less than` (and the other operators) all work against the variable; name the operator you mean.

**Comparing two values both already on the page needs no capture.** When both operands are visible on the *current* screen, compare them directly by naming both fields and the operator — no variable in between: `Verify the grand total is greater than the subtotal`, `Verify the quantity in stock is greater than or equal to the quantity ordered`, `Verify the shipping address equals the billing address`, `Verify the item-count badge matches the number of rows in the line-items table`. Capture into a variable only for a value that appeared on an *earlier* page or step (above) — don't detour through a variable for two values you can see at once.

### Visual validation

**Functional (text/value-based) — the default:**
```
Verify the cart summary table is displayed with the correct line items
```
Produces a standard content check.

**Visual (screenshot comparison) — triggered by visual keywords:**
```
Visually verify the checkout page layout matches the expected design
```
Generates a baseline comparison. Keywords "visually," "visual appearance," "looks the same," "layout matches" trigger the visual path.

```
Verify this page looks the same as the cart page captured earlier
```
Generates a step-to-step visual comparison linking to the earlier capture step.

**The rule:** default is functional. Add "visually" / "visual" / "looks like" to route to visual validation.

**The match tolerance is an editor knob, not prompt text.** State *what* to compare in English — the whole page or, better, a region scoped away from volatile content. To make a check stricter or looser, region-scope it rather than writing a tolerance into the instruction.

When the page shows record-specific volatile data (a seeded order id, timestamps, randomized names), either region-scope the visual check away from those fields (`templates.md` Template 8, region-only comparison) or pin to a record with stable display values — otherwise the baseline diffs every run for the same reason #16 warns about live feeds, now applied to data your own test seeded (the isolation contract, SKILL.md #20).

### Negative verification

```
Verify the error banner is not displayed
Confirm the "Out of Stock" label is not present on the product page
Check that no validation error message appears after submitting the form
```

The trigger is "not" paired with presence verbs ("is not shown," "is not displayed," "is not present," "no … appears"). The check is a presence test with negation — it confirms the element is absent.

**Pair a standalone or state-change absence with a positive anchor** — otherwise it can pass without proving anything when the whole page failed to load (an absent element is "absent" on a blank/crashed page too). When the absence *is* the point (after an action, or the only check on the page), name a positive state that proves the page rendered: `After the cart is emptied, verify the "Remove item" button is not displayed AND the "Your cart is empty" message appears` — not a bare `Verify the "Remove item" button is not displayed`. (This is the same rule the refusal-test section applies to error paths, below.)

#### Error-path / refusal tests — assert the complement pair, anchored to a positive state

A test that checks the app *refuses* something (bad login, invalid input, blocked transition, permission denied) is the single most common place a test passes while testing nothing. Two phrasings pass without proving the refusal happened — refuse both:

| Passes even when the control is broken | Why it proves nothing |
|---|---|
| `Verify an error message appears` | An error appears for *any* reason, including the wrong one. A page-not-found or an unrelated validation error passes this. |
| `Verify you are not on the dashboard` (lone negation) | True before login, true on a network error, true on a blank page. Absence alone proves nothing happened — not that the *right* thing happened. |

A correct refusal test asserts **both halves of the complement pair**, and anchors them to a **positive state that proves the page actually rendered**:

1. **The forbidden transition did NOT happen** — named concretely: "still on the login page," "the wizard did not advance past step 2," "the record was not created."
2. **The specific, expected error text** — the actual message, matched as a substring, not "an error."
3. **A positive anchor first** — a value that is present only when the page you expect actually loaded, so the negative assertions aren't passing against a crashed/blank page.

```
Submit the login form with an invalid password
Verify the login page is still displayed (the email field is still visible) and the error reads "Incorrect username or password", and the URL did not change to /dashboard
```

```
Save the opportunity with a close date in the past
Verify the record was NOT saved — the edit modal is still open AND the inline error contains "Close Date cannot be in the past"
```

Contrast: `Submit an invalid password and verify an error appears` passes if the form crashes, if any validation fires, or if the page 500s — none of which is "login was correctly refused." Name the page you must still be on, the transition that must not have happened, and the exact message.

The same shape applies to positive-with-absence checkpoints: pair the absence with the positive state it qualifies — `Verify the confirmation page loaded (it shows an order number) AND no error banner is displayed`, not a bare `Verify no error banner is displayed`.

### Contrasting examples

| Phrasing | Generates |
|---|---|
| `Verify the page title reads "Checkout — Acme Store"` | exact match on the page title |
| `Verify the page title contains "Checkout"` | substring match — matches "Checkout — Acme Store," "Secure Checkout," etc. |
| `Verify the confirmation total equals the orderTotal variable` (set on the review page) | Value check against a named local variable — the reliable cross-step form |
| `Visually verify the cart page layout matches the baseline` | Visual baseline comparison |
| `Verify the "Item removed" confirmation banner is not displayed after closing the cart` | Existence check with negation |

---

## 3. Conditional, fallback, and recovery phrasing

When the flow isn't always the same, you have two tools:

- A **conditional** — a standalone instruction that runs only when a condition *you can name* holds: *"If you see X, do Y."* Reach for it when a specific, nameable thing may or may not be on the page.
- A **"handle it" instruction** — natural-language phrasing for a genuinely vague or heuristic situation the agent must assess visually: *"Handle any popup that appears."* Reach for it when you **can't** state a clean condition.

### Writing a conditional

**One conditional, one line.** A conditional is its own instruction — never glued onto a normal step. If you've written a compound line with a conditional buried in it, pull the conditional onto its own line:

```
❌  Open the dashboard, dismissing the "What's new" dialog if it appears, then open Reports
✅  Open the dashboard
    If you see the "What's new" dialog, close it
    Open Reports
```

When the condition isn't met, the step simply doesn't run and raises no error.

**Positive or negative.** State the condition either way — *"If you see X, do Y"* or *"If you don't see X, do Y."* Prefer the positive form where you can: a negative can read as true before a slow element has finished rendering, so never hang a load-bearing action on a bare *"if you don't see X"* — and if you must, bound how long to wait before concluding X is absent.

**No "otherwise."** There is no else. A genuine two-way choice is **two separate conditional lines**, and only for *incidental* UI:

```
If you see the compact toolbar, open the "⋯" overflow menu and click "Export"
If you see the full toolbar, click "Export" directly
```

The two conditions must be **true complements** (X / not-X), not two unrelated positives, so they don't overlap or leave a gap. **Never split the behavior under test this way** — if *which side runs* is the thing you're proving, seed it instead (next rule).

**Name the condition on something you can point to** — *"If you see the 'Session expired' dialog, click 'Log in again'."* A specific dialog, banner, or element, not a vibe.

### A conditional can't carry the behavior under test

Conditionals and "handle it" steps are for genuinely **optional, incidental** UI that may or may not appear and *isn't what you're testing* — a cookie banner, a promo popup, a first-visit interstitial. **Never put the behavior the test exists to prove behind one.**

A **conditionalized assertion silently passes when its condition is absent** — and with a bare *"if"* now the natural form, this is the easy trap to fall into:

```
❌  If a discount appears in the cart, verify it is 10%
```
If a regression stops the discount from ever applying, the condition is simply never true, the step never runs, and the test reports green **having asserted nothing** — it passes *because* the thing under test broke. Nothing in the line even hints the check was skipped.

```
✅  Add the "SAVE10" promo code to the cart
    Verify the cart shows a 10% discount line
```
You control the precondition (you applied the code), so the discount is **required**, not optional — assert it head-on, no *"if."* The rule: **if you can't predict which branch must run, you don't have a test — you have an observation.** Seed the precondition until the expected outcome is required, then verify it directly. (A test's terminal outcome verify is likewise never conditional — #26.)

### When you can't name the condition — "handle it"

When the situation is too vague to state as a named condition, describe it in natural language and let the agent assess the page:

```
Handle any promotional popup that may appear after login — dismiss it and return to the main page
```

**The cleanest cookie-banner / popup dismissal** is a "handle it," not a named conditional — cookie banners vary too much to pin to one element:
```
Dismiss the cookie consent banner if it appears
```
Better than forcing a rigid condition on one banner variant (`If a banner with text "We use cookies" and an "Accept All" button is visible…`) — the agent assesses the page visually and handles variants gracefully. **Bound the wait when the element often *isn't* there:** an unqualified "if it appears" can make the agent wait the full missing-element timeout for a popup that never shows, so cap it — `Close the promotional modal if it appears within 5 seconds, then continue`. (A banner that *reliably* appears needs no bound.)

**"Whichever is present"** is a handle-it too: `Click "Sign In" or "Log in", whichever is present` — the agent picks the one that exists, which you can't pre-determine.

**Recovery / retry** is a handle-it — and name the failure mode it targets:
```
Click "Sign In". If Sign In is rejected with a wrong-credentials error, verify the error message is shown and stop with a clear failure — do not retry. If the click itself fails on a page error or timeout, reload the login URL and retry the sign-in once.
```
A *functional* failure (bad credentials) is a real result to assert and fail on — retrying it away masks the defect (the same swallow trap as above). An *infrastructure* failure (page error, timeout) is the only kind worth a reload-and-retry.

**Flatten nested conditions into independent lines.** Because each conditional stands alone, what used to nest becomes separate lines — or a single handle-it when the cases are all "dismiss whatever shows up":
```
If you see a promotional overlay, dismiss it
If you see a loyalty popup, dismiss it
If you see a survey modal, dismiss it
```
or simply: `Dismiss any promotional overlays, loyalty popups, or survey modals that appear`.

---

## 4. Capture and reuse patterns

The full phrasing surface for "capture X and use it later."

### (a) Within-test capture → reuse (local variables)

**Phrasing pattern:** `set [the value] as a local variable named [name]`, then reuse it as `the [name] variable`. For a value read from the page or a response, combine the read with the set: `capture [the value] and set it as a local variable named [name]`. The verb **set** is what makes the platform store the value; use it explicitly, and always give the variable a concrete name.

#### A value worth capturing is a value worth verifying — native verify-capture is the default

A value the test reads at runtime can be empty or malformed at the instant it's read, and a bare *capture it as X* leaves the create agent to decide *which* element to read and *how*. So for any value read from a **discrete, individually-verifiable element** — a page element, or a targetable surface like the email reader — the default is **one shape, every time, regardless of how a later step uses it**:

```
Verify the order number is not empty and capture it as orderId
```

The verify **is** the capture mechanism: it names the element and how success is judged, and the same instruction stores the value into the named variable. This is the *capture* case of *prefer the platform's native capability, fall back only when needed* (SKILL.md) — the native element verify-capture is the default; hand-written custom JavaScript is a fallback (below). There is no per-capture "do I guard this one?" decision to make for an element value — you write the one shape.

**The source decides the shape.** The rule keys on what you *read* from a source, so a value the test *generates* itself is a separate case:

| Source | Capture shape |
|---|---|
| A **discrete element** — a page field, an email-reader value you can target, or an individually-verifiable **API/response field** | **native verify-capture** — the one shape above |
| A **native non-element source** — the URL, a downloaded filename (§(c)) | **plain capture**, no verify — the platform read is unambiguous and there is no element to point a verify at; a *"verify the URL is not empty"* guard is a tautology that doesn't protect the part you care about (the id segment) |
| A value the test **generates itself** — a random email/string it produces, *not read from any source* | **plain capture** — there is no source element to verify; capture it once and reference the variable (#17) |
| Something **a native element verify genuinely can't reach** | **custom JavaScript**, and only here — see *When JavaScript is the fallback* below |

**Why verify a value you're only going to re-enter?** Because the uniform shape removes a branch the create agent (and you) could otherwise judge two ways — the reproducibility the one shape buys. The underlying fact is still true: a value **compared** later (verify/assert) — or reused nowhere — passes **silently** when empty, whereas one **re-entered** into a field (enter/type/fill) fails **loudly** at the enter step on its own. But that silent-vs-loud distinction is no longer a decision you make: you verify-capture the element either way, and for the re-entered case the verify earns its keep by pinning the native element read rather than by catching a silent empty. This also folds in the **write-only** case — an **element** value written to a project variable ((b)), a TDM datasource column (`tdm.md`), or handed to a chained test is never read back on that line, so an empty write persists silently; the uniform rule already verify-captures it, so "write-only is guarded" falls out of the rule rather than being a rule you apply separately (§(b) and #29). (A write-only value from a *non-element* source — say an id parsed from the URL — follows the plain-capture carve-out above; guard its shape with a fixed-marker *contains* only if the downstream consumer needs it.)

**Already verified on the same line?** If the capture's own line already asserts the value's content (*verify the cart shows the correct product name, and capture it*), that check **is** the verify — don't add a redundant *is not empty* (bloat, #1).

Write it as **one combined instruction** — the verify and the capture on the same line, never a standalone verify bullet before a separate capture (as one instruction it reads as a native verify-capture, not verification bloat — SKILL.md #1).

**Verify the shape, not the value.** When you verify an element value, the shape is the value's *durable* property — it has content, or it contains a fixed marker. The exact value is the part that changes on the next run, which is why you're storing it. An exact-value check passes right after generation and fails the moment the value rotates; a shape check survives.

**Pick the shape with a plain operator — no regex.** The default, for any value with no fixed marker, is **is not empty** — it confirms content was captured (an empty field fails it) and reads plainly for a non-coder:

| What you know about the value | Use | Example phrasing |
|---|---|---|
| Nothing fixed — a name, an id, an OTP, free text | **is not empty** | "verify the field is not empty and capture it as generatedName" |
| A **fixed substring** always appears — a prefix, a domain, a status word | **contains** that substring | "contains ORD-", "contains @functionizeapp.com", "contains Complete" |
| It must **not** be a known-bad value | **does not equal / does not contain** | "does not equal Error", "does not contain failed" |
| It relates to a value **captured earlier in this test** | **compare to that earlier step** | "is greater than the total captured in the previous step" |

You describe the check in plain words — never a regex, never an operator code. (The no-regex rule is for the **capture guard**; §2's pattern/format verifies still apply to an ordinary *display* check you are not capturing.) **Two phrasings to avoid:** *"contains a digit"* pins the specific digit the generator sees at generation time (it breaks the moment a later value lacks that digit), and *"matches a pattern…" / "is an N-digit number"* generates a **regex**, not a plain operator. A number needs no special check — when it's guarded, *is not empty* guards it like anything else. **"Compare to an earlier step" is a cross-step comparison, not a source-shape guard** — it counts against #12's one-per-test budget.

**When JavaScript is the fallback.** Custom JavaScript is the capture path **only** when a native element verify genuinely can't reach the value — never as a shortcut. Three cases: (a) the value is **embedded inside a larger block of free text** and must be parsed out (a code mid-sentence, not a discrete element); (b) it must be **computed or transformed** (arithmetic, a date computation, a signature); (c) it lives in a **surface with no traversable DOM** (a canvas, a non-traversable Shadow DOM). Anywhere the value is a discrete element, the native verify-capture above is the move — a bare *capture it* there both drops the verify and can lead to gratuitous custom JavaScript where a native read would have done it.

**The self-verify smell.** If a captured value's *only* downstream use is verifying it equals itself, the verify proves little on its own — *is not empty* is its minimum sanity floor, and the vacuous self-check is worth rethinking — including a verify whose only job is to echo the captured value back to the user. (The always-verify default also closes a gap the earlier per-downstream-use model left open: a value re-entered into a field that *silently accepts empty* — an optional field, a blank-query search — is now caught at the capture verify, not left to the test's terminal outcome verify, #26.)

**The email / OTP case is read off-page.** A received code is read from the **email reader**, not a page field: open the reader, wait for the message, then verify-capture the code (the reader does not auto-capture; see `specialty-steps.md` §4 and anti-pattern #27). The reader exposes the code as a **discrete, targetable element**, so it takes the uniform shape like any element — *verify the verification-code element is not empty and capture it as [name]*. Fall back to the JavaScript parse only when a particular email embeds the code **mid-sentence in free-text prose** with no distinct element (the free-text case above). This off-page source is why the rule spans *any* runtime element read, not just on-page ones.

**Persisting a capture across runs:** verify-capture it **locally** first (the uniform rule), then **write it to a project variable in a separate explicit step** ((b) below) — a local `set` creates the variable inline, but a project variable is platform-managed, so you can't capture straight into project scope.

```
Select the first product result and add it to cart, and verify the product name and price are not empty and set them as local variables named productName and unitPrice, for later verification
```

Later in the test:

```
Verify the cart shows the productName variable
Verify the cart line total matches the unitPrice variable times the quantity
```

**Always set and name the variable for any value you reuse — on both sides.** Loose prose reads as a comment, not a variable, whether it's how you try to **store** the value (*"remember this name"*, *"the value above"*, *"the same name"*) or how a later step tries to **recall** it (*"verify the remembered name"*, *"the name from earlier"*, *"that project name"*): no variable is created on the store side, and nothing resolves on the recall side (anti-pattern #17). **Pick a concrete variable name at the point you set it, suggest it in the prompt, and reference that exact name — as `the [name] variable` — every time after** — a value captured but then recalled only as "the remembered name" is as broken as one never captured. This matters most when:

- **Reused far from its capture**, or referenced more than once — a later bare "the captured email" is ambiguous when more than one capture is in play.
- **Multiple captures live at once** (an email *and* an order number *and* a name) — "the captured value" is ambiguous; named variables `signupEmail` / `orderId` are not. Naming an explicit variable is what anti-pattern #17 calls for — an unnamed value can't be reliably referenced again, so name it once and reuse that name.
- **Chained / cross-test reuse** — a value consumed in a different test must be a named handle (a chain variable or a project variable; see (b) below and #20).

```
Sign up with a generated email and set it as a local variable named `signupEmail`
... later ...
Log in with the `signupEmail` variable and verify the account page greets that same address
```

The named handle is then the *only* thing later steps use to reference the value — never re-type the address, re-generate it, or re-read it from the page.

**Reference form — the other side.** Every later use names the variable the **same** way and **flags that it's a variable** — `the [name] variable` — whether you're entering it or verifying against it:
```
Enter the oppName variable into the global search box and press Enter
Verify the Opportunity Name field shows the oppName variable
```
The **read-side** reference is **identical for local and project scope** — `the [name] variable`, same everywhere. The **write side differs, and not symmetrically**: a local variable is created inline by `set … as a local variable`, but a project variable **cannot be set or captured inline** — you capture to a **local** variable first, then **write** that local into the pre-created project variable in a **separate** step (anti-pattern **#30**; §(b)). The one hard requirement: the **exact same name** appears where you set it and in every later use (a name mismatch fails quietly — the step only *warns* — so keep it verbatim).

Capture is also how a test *owns* the record it mutates (the isolation contract, SKILL.md #20): after seeding/creating a record, capture its id, then have every later edit/verify/cleanup step reference that captured id rather than re-finding the record by position or recency.

**A captured value persists across tab/window switches for the whole run.** You can capture a value in one tab and reference it in another after switching context (e.g. read a code in the email-reader tab, use it back in the app tab) — captured values are run-scoped, so you don't need to re-derive one just because the flow crossed tabs. (For *cross-test* durability, use a project variable — §(b) below.)

For precise cross-step reference when multiple captures exist:

```
On the shipping method page, verify the shipping cost is not empty and set it as a local variable named shippingCost; later, verify the confirmation shipping cost equals the shippingCost variable (not the cart subtotal)
```

**When a page has more than one money field (subtotal, shipping, tax, grand total), name the *same* one on both the capture and the verify — or the comparison is meaningless.** "Capture the total / verify the total matches" doesn't say *which* total, so the capture and the verify can land on different money fields (subtotal vs. grand total — they differ by shipping + tax): a silently hollow assertion that passes against the wrong number. Name the **same** field on both sides so both resolve to the one you mean. Say which total: `verify the grand total (the final amount due, not the subtotal) is not empty and set it as a local variable named grandTotal` … `verify the confirmation grand total equals the grandTotal variable`. Capture the comparable value at a point where it's final — e.g. the grand total only exists *after* shipping and tax are applied, so capture it on the payment/review page, not the cart.

**Local vs. project is your authoring decision, made from the test plan — not the create agent's.** The create agent sees only this one prompt; it has no view of your other tests, so it cannot choose the scope and cannot act on a conditional like *"use a project variable if another test needs it."* You decide as you author, from the test plan: a value used only inside this test → **local** ((a) above); a value your test plan hands to another test or case → **project** ((b) below). Write the **concrete** choice into the prompt.

**A third home — the preset variable — holds the values a human sets, and is never written from a test.** An **execution-preset** variable is a human-set runtime value, resolved per run from the selected preset. It is the modern home for **secrets** (mark it **masked** — encrypted in the keystore, hidden in logs/screenshots/artifacts) and for **stable per-environment config** (non-masked — an instance/base URL, a region), with typically one preset per environment selected at run time (where presets aren't available a secret falls back to a **secret variable**). It is **read-only** from a test: reference it by name, in words — `enter the password from the masked preset variable named admin_password` — and never create or write it from a prompt (confirm it exists before referencing, #21). A preset variable and a project variable are **different homes**: reference a value at the scope where it lives — a same-named variable at the **wrong** scope resolves **empty** at run time (the scope-match bite of #21 — `functionize` → `dsl-and-actions.md`). Presets are for human-set input; a value the *test itself* produces for another test stays a **project** variable ((b) below), because a test cannot write to a read-only preset.

**Worked example — referencing a masked preset variable:**
```
<!-- admin_password is a masked execution-preset variable: a human sets it in the platform
     (Project → Execution Presets), it is read-only from the test, and it is never set or
     created from this prompt. Confirm it exists for the target preset before running (#21). -->
Log in as the admin user, entering the password from the masked preset variable named admin_password
Verify the admin console loads and the header shows the Administration menu
```
Use this shape rather than a project-variable login copied and relabeled — the preset, not the prompt, supplies the value.

### (b) Cross-test capture (project variables for orchestrations)

**Phrasing pattern — always two steps.** A runtime value reaches a project variable in **two** explicit steps: **verify-capture it into a local variable first**, then **write that local variable into the existing project variable** (that write step is a **custom-JavaScript step** — native capture only ever creates a local, so the project-variable write is the one place custom JS is the mechanism; `specialty-steps.md` §5). You **cannot capture straight into project scope** — a local `set` creates its variable inline, but a project variable is platform-managed and pre-created, so a single *"verify X and write it into the project variable"* line skips the local landing the write needs (an empty or unbound result). Two steps, every time:

```
After placing the order, verify the confirmation number is not empty and capture it as a local variable named orderId; then write the orderId variable into the existing project variable named last_order_id so downstream tests can reference it
```

- ❌ *verify the confirmation number is not empty and write it into the project variable named last_order_id* — one clause, no local capture; the runtime read has nowhere to land.
- ✅ *verify the confirmation number is not empty and capture it as a local variable named orderId; then write the orderId variable into the existing project variable named last_order_id.*

To hand a value to a later test in the same orchestration, write it into a (pre-created) project variable — the durable channel for passing data across tests — and the dependent test reads that variable by name. A handed-off value is **write-only** — a store this test never reads back — so the uniform native verify-capture (§4 (a)) applies as it does to any element value (it is the local-capture step above); the write-only case is *why* the verify matters here, since an empty write hands the next test nothing, silently.

**The project variable must already exist in the project.** Unlike a local variable (which `set` creates inline), a project variable is platform-managed — the create agent can't create one from a prompt. Create it once in project settings first (a one-time platform step), then **write** to it from the test. (This is the write-side companion to #21: a project variable is a project-scoped fact.)

### (c) Extracting from non-element sources

These are the **native non-element sources** of §4 (a)'s boundary table: the platform read is unambiguous and there is no element to point a verify at, so they are a **plain capture** — the verify-capture default is for element values, and forcing a *"verify the URL is not empty"* here would be a tautology. (A value from these sources that a later step compares is checked by that downstream compare.)

**From the URL:**
```
After the redirect completes, capture the order ID from the current URL
```
The platform reads the value out of the URL for you.

**From a downloaded file:**
```
After the report downloads, extract the report ID from the downloaded CSV filename
```
Filename extraction is native; file *content* extraction requires an Extension (see specialty-steps.md).

### (d) Page state for visual comparison

The forward-dependency constraint matters here. The capture instruction must explicitly say "for later visual comparison" — otherwise the generated step is a normal functional verify with no screenshot baseline to link to.

**Correct — capture line includes the intent:**
```
Verify the shopping cart summary is correct and capture the page state for visual step comparison in a later instruction
```

**Correct — comparison line references the captured state:**
```
Visually verify the checkout review page matches the page state captured from the cart page
```

If you omit the capture clause on the earlier line, the visual comparison on the later line has nothing to link to.

### Summary

| Scenario | Phrase triggers | Generates |
|---|---|---|
| Same test, same run | "set [X] as a local variable named [name]" | a variable you reuse as "the [name] variable" |
| Across tests in an orchestration | "write [X] into the existing project variable named [name]" | a durable variable the next test reads by name (create the project variable first) |
| From URL | "extract [X] from the URL" | the value captured from the address |
| From downloaded filename | "extract [X] from the downloaded file's name" | native — no extra tooling |
| From file *contents* | (requires an Extension) | an Extension step (see `specialty-steps.md`) |
| For visual comparison | "capture the page state for visual step comparison" on capture line + "visually verify against the captured page state" on comparison line | Screenshot + visual step comparison |

---

## 5. Prompt anti-patterns and smells

These look reasonable but produce bad tests. Refuse them and offer the correct version.

### (a) Defensive verification bloat

**Bad — verify after every click:**
```
Click "Add to Cart"
Verify the "Added to Cart" toast appears
Click "View Cart"
Verify the cart page loads
Verify the product name is displayed
Verify the quantity is 1
Verify the price is $29.99
Click "Checkout"
Verify the checkout page loads
Verify the shipping section is displayed
Verify the payment section is displayed
...
```

**Why it degrades quality:** Intermediary verifications add overhead and a 15-step flow can balloon to 40+ steps. When one of them fails (transient toast, slow page load), the test stops cold. The verifications aren't testing business logic — they're testing that the application reacted to a click.

**Good — verify at logical checkpoints:**
```
Search for "wireless headphones"
Select the first result and add to cart
Proceed to checkout as guest
Fill in the shipping address
Select standard shipping and continue to payment
Enter payment details
Place the order
Verify the order confirmation page shows the correct order number
Verify the order confirmation page shows the correct product name
Verify the order confirmation page shows the correct total
```

Single verification checkpoint at the outcome — three discrete asserts, one per field (§2), all at the end. If the flow breaks earlier, the run fails on the step that couldn't complete — a clear failure with a screenshot, not a misleading mid-flow assertion.

### (b) Vague omnibus verification

**Bad:** `Verify everything on the checkout page looks correct`

**Why it degrades quality:** "Everything" and "correct" are undefined, so the check lands on the entire page body and the failure message is meaningless: "Page body does not equal expected." You can't debug that.

**Good:** name the specific fields — and give each its own plain `Verify` line (one check per verify, §2), never one bundled line:
```
Verify the checkout page shows the correct product name
Verify the checkout page shows the correct quantity
Verify the checkout page shows the correct order total
```

Each field is a discrete verify. A failure tells you exactly which value was wrong.

### (c) Multiple unrelated flows in one test

**Bad:**
```
Log in as an admin
Create a new product listing
Search for the product as a guest
Add it to cart
Check out
Log in as admin again
Delete the product
Check inventory reports
```

**Why it degrades quality:** The test has no single reason to fail. If the product creation breaks, all guest-flow and admin-teardown steps fail too — but the guest flow itself was fine. You can't run them independently, can't retry just one, and the 60+ step test takes minutes.

**Good:** Three separate tests — `Admin: Create Product`, `Guest: Purchase Product`, `Admin: Delete Product`. Run them sequentially in an orchestration. Each fails independently with a clear cause.

### (d) Excessive nesting of conditionals

(Covered in section 3 above — each conditional is its own line, so nesting flattens into independent conditional lines, or a single "handle it" when the cases are all "dismiss whatever appears".)

### (e) Hardcoding data instead of variables

**Bad:**
```
Enter email: testuser_abc123@example.com
Enter card number: 4111111111111111
Enter shipping address: 123 Main St, Springfield, IL 62701
```

**Why it degrades quality:** The test fails when the hardcoded email is already registered, when the payment processor rejects duplicate transactions from the same card, or when the address needs to change across environments. Every data change requires editing the prompt and regenerating steps.

**Good:**
```
Enter email: the email from the preset variable named checkout_email
Enter card: the card from the masked preset variable named checkout_card
Enter shipping address: the address from the preset variable named shipping_address
```

One variable change updates all tests using it. Environment-specific values work automatically.

**The inverse also over-reaches — a value must *earn* its variable.** Variabilizing is need-based, not reflexive. A value earns one when it is a **secret** (always — masked preset, *Credentials & secrets* / SKILL.md #4), **reused** by a later step (capture it, §4 (a) / #17), **differs per environment**, **format- or locale-sensitive** (money, dates — "Formats and locale" above), a **fixture** — an org- or environment-varying identifier or target (resolve it, never inline it, #21) — an **entry / base URL** (§5 (f)), or would otherwise **collide** if hardcoded. A plain, non-secret literal with **none of those** stays inline — even one used at a single point; minting a preset for it is machinery it hasn't earned, and a user pasting five fixed config values for one throwaway test does not want five presets. Provenance and sensitivity decide it, not use-count: a one-off signup email is still a variable (it **collides** on re-run), while a fixed search term or a one-off note stays inline.

### (f) Under-specified start URL or auth state

**Bad — no auth context, no URL:**
```
Navigate to the dashboard and create a new report
```

**Why it degrades quality:** Without an explicit entry point and auth state, the prompt leaves open whether the test starts already authenticated, what URL the dashboard is at, and whether there's a login gate — so a spurious login step can appear or generation can open on the wrong URL. Supply both (next).

**Good — explicit auth and entry point:**
```
Navigate to the reports dashboard at https://app.example.com/reports (user is already authenticated)
Create a new report with title "Q1 Summary"
```

**The entry point depends on whether a component leads the test.** Every created test begins at an entry URL, and how it's set has two cases:

- **A component is the first step (typically a login component)** → the component **carries its own URL, and that is the entry point** — use it **unless the author specifically specifies a different starting URL**. Absent such an override, **don't set, supply, look up, or default a starting URL, and never fall back to the project/environment URL**; just reference the component as the first step and let its embedded URL be the entry, **regardless of how many environments the project has**. (Every component embeds a URL; for a login component that URL is the login page.)
- **No component leads the test** → there is no embedded URL to inherit, so the entry URL is a **project-scoped fact you resolve** (an application of #21): use the known entry URL when the project has one environment; **ask which environment** when it has two or more; and if no URL was provided, **ask for the username, password, and starting URL** (credentials go in named masked variables — a masked preset variable, or a secret variable where presets aren't available — never inline) — don't invent any of them.

The entry URL falls back to the project/environment URL when nothing else sets it — that's the intended default. The case to get right is making sure that fallback lands on the app: an environment URL is often an SSO/IdP host or a portal/landing page rather than the app, so if the entry defaults there, generation opens on the wrong page and downstream steps look for the app where it isn't (a loose "verify the app loaded" can even pass there without proving the app opened). When a component leads, the fallback never applies — its own URL is the entry (unless you specifically override it). When none does, resolve the entry per the cases above rather than leaving it to the default.

**For a login artifact that carries its own entry URL** (a login test or a login component), keep that URL *live*: when the prompt names a project variable for the app URL, the generated first (load-URL) step must hold that project-variable reference, never a hardcoded literal, so the entry URL tracks the variable across environments. A hardcoded literal instead of that reference won't track the variable after a repoint (an environment migration or host change) — so phrase the load-URL step to reference the project variable, and the entry stays live across environments (#17).

### (g) Subtle smells beyond the obvious

**1. Anchorless element references**
```
Click the "Submit" button
```
Three submit buttons on the page, and nothing in the instruction says which one. **Fix:** `Click the "Submit" button in the shipping address section` — the section anchor names the target precisely.

**2. Implicit page state assumptions**
```
Click the first product in the search results
```
Assumes search results exist. What if the search returned zero results? **Fix:** `After the search results load, click the first product`. The "after … loads" clause adds an implicit wait for results.

**3. Contextless tab/window references**
```
Switch to the email reader tab and find the verification link
```
No prior instruction opened the email reader tab. **Fix:** The instruction that precedes it must say `Open the email reader in a new tab`.

**4. Over-specifying mechanics**
```
Click the "#add-to-cart" button with CSS selector ".btn-primary"
```
A hardcoded CSS path locks the step to one DOM shape and breaks on any UI change. **Fix:** `Add the product to the cart` — at intent level it adapts when the UI shifts.

**5. Environment-agnostic file paths**
```
Upload the spreadsheet from /Users/alice/Desktop/test-data.csv
```
File paths don't exist on the execution server. **Fix:** Use a TDM datasource or a Functionize-uploaded file (see specialty-steps.md → file ops).

### (h) Acquiring a stateful/shared record by position or recency instead of owning it — the isolation contract

**Bad — grab whatever record happens to be on top:**
```
Navigate to the Orders list
Open the first order and click "Cancel Order"
Verify the order status shows "Cancelled"
```

**Why it degrades quality:** "the first order" fails on **state**, **contention**, or **identity**, and mutating a record the test didn't create pollutes shared environments — *every test owns the data it touches.* (Full failure-mode breakdown + the canonical-violation list: anti-pattern #20 below.)

**Good — seed the record, capture its id, act on that id (see templates.md, Template 7):**
```
Call POST /orders at the base URL from the preset variable named api_base_url with ... and verify the response orderId is not empty and set it as a local variable named `orderId`
Navigate to the https://app.example.com/orders/ page for the order id in the orderId variable
Click "Cancel Order"
Verify the order (the orderId variable) status shows "Cancelled"
```

**Also good — pin a known, dedicated record by a stable identifier:**
```
Open the order whose number is in the project variable <your_order_id_var> and click "Cancel Order"
```
or, in a list, anchor by the record's identifying value rather than position.

**Not this anti-pattern:** browsing STATIC, read-only catalog content — "click the first product in the search results" (smell #2 above) — is fine and correct (see anti-pattern #19). The line is whether the test **mutates** the record or shares it with other writers. Static read-only → relative selection. Stateful/shared-mutable → seed it or pin it (SKILL.md #20).

**Capture-on-creation, then seed-then-search — the captured value is the test's only handle.** *Whenever a record of any type is created, capture its identifying value at the moment of creation* — the **name you set** (give it a meaningful, scenario-readable prefix + a dynamic unique suffix) **or** a **system-generated id** (order number, document number, record id). That captured value is then the **only** thing later steps use to **find AND verify** the record — by searching for it, opening it, and asserting against it — whether in **this same test** or a **chained downstream test** (hand it off via a chain variable or a project variable; see #9 and #20). Never re-derive, re-guess, or re-search by position/recency/a free-text field.

The unique suffix is also what makes the later **search return *exactly one* record**: a *static* name (`"Acme Renewal Opp"`) accumulates one row **per run**, so the search returns **many** and the test opens the wrong record or fails on an ambiguous multi-match; the unique suffix guarantees the search resolves to the **single** record this run owns.
```
Create a new opportunity named "Acme-Renewal-" followed by a random 6-character string, and set the name as a local variable named `oppName`
... later in the same test ...
Search for the `oppName` variable in the global search box and press Enter, then open the single matching result
Verify the opportunity (the `oppName` variable) shows Stage "Closed Won"
```
If your **test plan** hands this opportunity to another test or case, set it as a **project** variable instead (§(b)) — a scope decision you make from the plan when you author, since the create agent can't see your other tests.

Search and verify on an indexed/unique field, never a free-text field that could match other rows. (Same rule for a generated email/username you'll reuse, and for a system id like an order number — capture it once at creation, reuse the captured value everywhere after; #17.)

When that list can **paginate**, narrowing to the captured value is not only about resolving a single match — it is about **correctness**: an un-narrowed presence/absence check sees only the current page, so a present row on a later page reads as absent and an absence check passes hollow (#31).

**Verifying a row is *in* a list — own it first, then narrow.** The same contract governs a **presence** check, not only a mutation. The trap: asserting membership on a row the test **neither created nor pinned**.

**Bad — assert on a pre-existing row matched by arbitrary text:**
```
Open the Projects list
Verify a row containing the text "Acme Corp" is shown in the list
```
It proves nothing durable: no reason such a row exists, no reason it still exists next run, and — on a paginatable list — no reason it sits on the page you can see. *"Contains some text"* is not an outcome.

**Good — own the row, then narrow to it.** Two ways to own it:
- **Seed it in-test** under a name carrying a unique suffix, then search to the single row you own; best-effort teardown only where the data must be reclaimed — the **unique suffix**, not the delete, is what keeps re-runs isolated (never a trailing UI delete, §1).
- **Pin an existing record** by a **stable identifier** — a durable business id, or a dedicated fixture a separate setup test or the user provisions under a reserved name (e.g. `"<Scenario> fixture — do not delete"`) — then search that **exact** value so the list resolves to the one row.

```
Generate a unique project name — the prefix "Smoke-" plus a random 6-character string — and set it as a local variable named `projectName`
Create a project with the name from the `projectName` variable
Search the Projects list for the `projectName` variable and wait up to 30 seconds for a matching row to appear
Verify the matching row shows the `projectName` variable
```

Either way, search on the unique/reserved value and verify **that rendered row**, never the raw on-screen list. **When the claim is only "the list is populated"** — not that a specific record is present — assert it **structurally** ("at least one row renders", #16); don't seed or search for a named row you don't actually need.

### (i) Anti-pattern catalog (#9–#22, #25, #26, #31, #32) — full write-ups

`SKILL.md` lists the title of each of these and points here for the detail. Each keeps its `#N` label — cross-referenced as `functionize-prompting #N` across the skill family and from elsewhere in this file. The condensed framings of #1–#8 stay inline in `SKILL.md`; their concepts are also taught above in (a)–(g).

**#9 — One giant long flow when it could be segmented** — a flow long enough that it cannot be generated atomically puts every downstream instruction behind its first weak link — one unready prerequisite near the top and the rest of the chain never gets built. For long E2Es, **segment at clean boundaries and chain via an orchestration** that hands off the captured id (e.g. an order number) — the downstream test must act on that handed-off id, not re-locate the record by position or recency (see #20) — and make/seed prerequisites solid *before* generating. This is distinct from #3 (which is about mixing unrelated flows) — here it's one coherent flow that's simply too long to generate atomically. Segment a multi-section or multi-tab tour into separate tests up front rather than planning to retry it.

**#10 — Leaving business-critical master data unspecified** — in complex enterprise apps (SAP especially), an unpinned "pick a material / customer / G/L account from the value help" can land on an invalid row (a template material not extended to the org, a chart-level account not in the company code) — wasting a run on a bad record. **Pin known-good IDs as project variables** and reference them directly; if discovery is unavoidable, add an explicit **validate-and-retry** instruction (e.g. *"verify the line received item category TAN; if not, pick a different material and retry — do not silently accept a bad row"*). Concrete pinned data + fail-fast validation is the single biggest reliability lever for ERP prompts.

**#11 — A verify that can't run against the real value on a dynamic-result page** — on a POST-submit result page the result can fail to render or be navigated away from before it's read (a POST result isn't preserved across navigation), so the verify must run against the value on the page that appears immediately after submit. Two defenses in the prompt: (a) for result-on-submit pages, instruct *"read the result on the page that appears immediately after submit — do NOT navigate away or reload, the result is not preserved across navigation"*; (b) add a fail-fast clause *"if the result/value cannot actually be read, report that it did not render — do not mark the step complete."* When reviewing a generated test, **confirm the verify step actually exists and asserts a real value**.

**#12 — Verifying a value captured earlier in the same run is the single most generation-expensive verification** — verifying a value captured earlier (e.g. "the confirmation shows the from/to account") works and is genuinely dynamic (adapts across runs), but limit it to **one** dynamic-value check per test; verify the *stable* parts (headings, fixed amounts like "$100.00") with normal text/contains and only cross-reference the one value that truly must match.

**#13 — Expecting direct typing to fill date/time picker fields** — on an entry/leaving date+time+AM/PM picker form, typed values can land in the *wrong* field (a date typed into the time box; a leaving-date field showing a time), and AM/PM radios are hard to hit — the picker is the more reliable path. Don't over-specify mechanics. Phrase the **method as a preference with explicit fallback latitude, not a mandate** — *"set the date using the date picker **if possible; fall back to text entry or another method if needed**"* — so it can recover if the picker isn't present or won't take the value. The date picker is the right *default*, but locking it in removes the latitude to fall back — keep the picker as the preference, allow the fallback. Pin the method hard only when the user explicitly requires one specific mechanism. Place the entry and leaving values as **separate** instruction lines so they don't bleed into each other. For a **relative** date, state it as **intent** — *"set the delivery date to 30 days from today"* / *"a date 5 days in the future"* — and let the agent compute and navigate to it — and **when the value may be *typed*** (a typed field, or a picker with a text-entry fallback), **state the expected output format** (*"30 days from today in MM/DD/YYYY format"*) so a typed value isn't guessed; only hand-write a custom-JavaScript-computed date when you also need the exact value to **verify** later, or for a typed (non-calendar) field. Cross-month navigation works on its own — state the relative date as intent; a custom-JavaScript-computed date is needed only when you must verify the exact value later.

**#14 — A blanket "dismiss the cookie banner if it appears" on a site that has no banner.** On a site with no consent banner, a blanket dismiss-interstitial step can click a stray link and navigate *away from the target page*. Only add a banner-dismiss line for sites that actually show one (typically GDPR-region sites) — there it works cleanly against a clear Reject/Accept control. If you're unsure, scope it tightly to the consent control (e.g. *"if a cookie-consent dialog appears, click its Reject/Accept button"*) rather than a generic "dismiss any banner," which the agent may satisfy by clicking the wrong element.

**#15 — Driving on-site search when a stable deep-link exists — especially on bot-protected sites.** On bot-protected public sites, on-site search can trigger a **"client challenge" CAPTCHA** and a category click can hit a **human-verification** page — either of which stops the flow before the verification instructions are produced. Where the target has a predictable URL (a product/article/package permalink), instruct the agent to **navigate directly to that URL** instead of typing into site search. It's faster, more deterministic, and sidesteps search-box bot challenges. Reserve the search flow for when *exercising search itself* is the point of the test.

**#16 — Verifying volatile, live content on a dynamic feed/listing.** An exact current value baked into a verify — a feed's "#1 story title", a specific commenter name, or a listing's exact "N Results" count — breaks on the next run. For feeds, search results, counts, dates, "latest" anything: phrase the check **structurally** — *"verify the list shows multiple ranked stories, each with a score"*, *"verify at least one result is displayed"* — never the specific text that happens to be there today. (This is the over-specific twin of anti-pattern #2's vagueness — aim for structural-but-specific.) **A "non-empty" check must resolve to a *visibly rendered* item, not merely a present container** — a "no results" empty-state, a skeleton/placeholder loader, or a spinner all satisfy "the list is on the page" while proving nothing rendered. Anchor the check to a real row/result/item (*"verify at least one result **row** is displayed"*), so an empty-state or still-loading skeleton **fails** instead of passing. **The inverse is just as data-dependent — asserting the empty-state message itself.** A verify that the list *shows* *"No items found."* holds only while the data happens to be empty; the first row anyone adds turns the test red through no fault of its own (exact-copy of a transient state). Assert an empty-state **only when the test owns that emptiness** — it created and then cleared the specific rows it asserts on, which is the owned-absence pattern in #31 (narrow by the unique captured identifier, then assert the positive empty-state). Otherwise assert **stable structure present regardless of data** — the page heading, the primary create/add control — never the population-of-the-moment.

**#17 — A runtime value referenced without capturing it into a variable.** Any value the test produces or receives at runtime — a generated random, a captured order ID or confirmation total, or a received OTP / verification code — must be **set once as a named variable and referenced by that variable** in every later line. Two phrasing patterns to get right. **Mode A — uncaptured:** a generated random referenced twice bare produces two different values — the email on the registration line and *"log back in with the same email"* on a later line are two separate generations, so the login no longer matches the account that was registered (same for a generated username or password). Fix: capture once, name the variable, reference the variable in every later line; never reference the same random twice bare. **Mode B — phrase the reuse so the enter step uses the live captured value:** phrase the capture as a native verify-capture — *verify the source is not empty and capture it as a local variable, then enter the stored variable* (#28) — so the enter step references the stored variable. The ambiguous *"read the code from the email and enter it"* (or a bare *"capture the code from the email body"*) can instead produce a one-time literal in place of a live read (see anti-pattern #27, extract-don't-snapshot — the companion rule that names the phrasing cause and its prevention). This matters most for a fast-rotating value like an OTP, but a reused order ID or confirmation total follows the same discipline. The **entry URL of a login artifact** is that discipline applied to the entry point: for a login test or component that carries its own entry URL, keep it a live project-variable reference so it tracks across environments — full rule in §5 (f).

**#18 — Naming a page/section/component the prompt hasn't confirmed exists on the target.** If a prompt names sections that **don't exist** on the target (e.g. "Modal", "Loading Images" on a site that exposes different ones), a named-but-absent section isn't silently skipped — without a skip clause the run keeps looking for it instead of moving on. Two defenses: (a) name only sections you've verified are present; (b) add a skip clause — *"if a named section is not present on the page, skip it and continue; do not search repeatedly."* A named target has no implicit "skip if absent"; you supply that latitude with the skip clause.

**#19 — Hardcoding a *specific* item in a browse/listing flow instead of selecting relatively.** On browse/listing flows, verifies/clicks pinned to a specific named product or a fixed grid position are brittle — they break when the catalog reorders or the item moves. For browse/category/search flows, instruct relative selection — *"open the **first** product in the grid"*, *"verify the result list shows the product you added"* (cross-referencing the captured name from the add-to-cart step — but per #12 keep to **one** such cross-step verify) — rather than asserting a hardcoded product name or a fixed row. Reserve a specific-item assertion for when that exact item is the point of the test.

**#20 — Selecting a STATEFUL or SHARED record by position, recency, or "any" instead of owning it — the isolation contract.** *Every test owns the data it touches.* It either **SEEDS** the record it will mutate (create it via API/DB/UI in-test, capturing the generated id) or **PINS** read-only/static data by a **stable identifier**. Picking the record to edit/cancel/delete/approve by "the first order in the list", "the most recent user", "any open ticket", "Recently Viewed", or a fixed grid row is the **canonical violation** — it fails on **state** (the record may already be in the asserted state → a pass that proves nothing), **contention** (another run or a tester sharing the environment mutates it mid-run), or **identity** (a different record every run). It is the **#1 source of latent flake**, and "have someone pre-create a specific record" is **not** the fix — it just relocates the contention to a shared fixture every run fights over. Field pattern: ❌ *"open the first order in the list and cancel it"* / *"edit the most recently created user"*; ✅ *"create an order via the API and capture its id, then cancel **that** order"* (seed-and-own — see `templates.md` Template 7) or ✅ *"open the order whose number is in the project variable <your_order_id_var>"* / *"in the row for the captured email, click Edit"* (pin-by-id — pin the row by its identifying value; the pinned variable is project-scoped, so confirm it exists per #21). This is what makes a suite **parallel-safe, order-independent, and re-runnable**. **Boundary with #19:** #19 is correct that for a *browse/listing flow over STATIC, read-only catalog content* you SHOULD select relatively ("open the first product in the grid") — nothing is mutated and the catalog is shared-read-only. #20 governs the opposite case: a record your test will **change**, or that other runs also write. Static read-only → relative selection (#19). Stateful/shared-mutable → seed it or pin it (#20). If a stateful record genuinely cannot be seeded or pinned and must be discovered at runtime, gate the mutation behind a **validate-and-retry** check (cf. #10): confirm the acquired record matches the expected, owned criteria before acting — never silently mutate "whatever was first." Adjacent: #10 (pin known-good master data), #17 (capture a self-generated value before reusing it).

**#21 — Treating a PROJECT-SCOPED identifier as a known constant — copying an example id/name out of documentation, or referencing a component / project variable / environment / datasource / project the target hasn't been confirmed to contain.** *Resolve, don't assume.* Component **names**, project-variable **names *and their values***, environment names, datasource names, and the **target project itself** are tenant- and project-specific facts that live **in the project, never in this skill** — so a concrete identifier you see in any example here is illustrative, never real. For example: a documentation **example** component name and an **invented** fixture variable dropped verbatim into a prompt against an **auto-picked** project — none of the three exist there, so generation can't resolve the nonexistent component and the fixture stays an unresolved placeholder, so the instruction can't do what it intends. **The gate — before generating any test that references these:** (a) **pin the exact project (by id) and environment yourself — never let the agent auto-pick**; (b) **verify every referenced component and project variable actually exists in that project — at the SCOPE the test references it**, and that each fixture variable holds a **real value, not a placeholder**. *Scope-match matters:* a same-named variable at the wrong scope (preset vs project) resolves to **empty** at runtime — the field receives nothing. Keep each value at the scope its reference resolves at, and confirm both the name **and** the scope: a human-set secret in a **masked preset variable** (or a secret variable where presets aren't available), stable per-environment config in a **non-masked preset variable**, and an inter-test data pipe in a **project variable** (presets are read-only). Full home table: `functionize` → `dsl-and-actions.md`. (c) if anything is **missing or ambiguous, STOP and ask** — report exactly what's missing and the project id(s) you saw; **never substitute a value, never invent a variable, never loop searching for a missing object, never silently auto-pick a project.** **Placeholder convention for prompts you hand the user:** write every project-scoped identifier as an unmistakable placeholder — `run the "<your login component name>" component`, the project variable named `<your_product_fixture>` — so an example can never be copied as a live value. Adjacent: #10 (pin known-good master data — values), #18 (don't name a target not confirmed to exist — UI), #20 (own your data — records).

**#22 — Treating an async / agentic "generate" or "create" action as synchronous, and verifying it by a progress/status indicator instead of the durable produced artifact.** Modern flows increasingly kick off background work — an AI assistant that *generates* content from a prompt, an async import/build/export job, a "create" that streams a result — where the triggering control returns immediately but the *result* lands seconds-to-minutes later, often with a backend-assigned name/id. Two traps follow. **(a) Don't chain the next step as if the result already exists.** A follow-on action ("open the created item", "delete it", "edit it") issued right after the trigger races a not-yet-produced artifact — the row may not be in the list yet, and its identifier may be assigned by the backend, not your prompt. Wait for the durable produced artifact to appear (the created record's link/row, the finished output panel, the downloaded file), and **capture its real id/name then** (cf. #17) before acting on it. **(b) Don't anchor the verify on a transient status/progress indicator.** A spinner, a "Generating…/Running…" label, a progress bar, or an in-progress *status field* is not proof of success — and such status fields frequently **don't settle on a clean terminal value** (they can linger in an intermediate "in progress"/"awaiting input" state after the real work finished, or auto-dismiss like a toast — cf. the transient-verify trap). Verify the **durable downstream effect**: the new row present in the list (narrow to it first — a raw list shows only its current page; #31), the generated output rendered with its expected **structure** (assert shape, not the exact generated text — #16), the file on disk, the record at its persisted end-state. ❌ *"click Generate, then verify the status shows 'Completed'"* / *"create the item, then immediately delete it from the list"*; ✅ *"submit the request, wait for the produced result to appear (the created-item link / an output panel with at least one result), capture its id, then act on and verify **that** artifact."* Adjacent: #9 (segment long async flows and chain via the captured id), #11 (a status badge is not a verify), #16 (assert generated content structurally), #17 (capture the produced id before reusing it).

**#25 — Writing author or environment context, or an author conditional, as a prompt instruction line.** The create agent turns **every line of the prompt into a test step**. So a line that isn't an action or an assertion still becomes one — and shows up as a real, customer-visible step in the generated test. Three shapes leak this way:

- **Account / environment configuration** — *"(this account is 2FA-bypassed)"*, *"SSO is disabled in this environment"*, *"credentials are pre-seeded"*, *"feature flag X is on"*. These are facts about the fixture, not things the test does. Left in a body line they generate as steps: a *"(this account is 2FA-bypassed)"* note becomes a step that "documents" the account config, and a paired author note like *"verify no OTP/2FA prompt appeared"* generates as a real negative assertion the test was never meant to make.
- **Setup rationale / notes to yourself** — why the test is shaped a certain way, reminders, TODOs.
- **Author conditionals addressed to the human or the agent** — *"if an OTP prompt appears, stop and tell me"*, *"let me know if the totals don't match"* (a note *to you* — asking the run to halt or report back — not a test branch). These read as instructions to the author, but the agent turns them into steps. **A genuine test-branch conditional is not this, and is legitimate:** *"if you see the 'Session expired' dialog, click 'Log in again'"* is a real, runnable step the test should contain — write it as a standalone conditional (§3), don't comment it out. The test to tell them apart: *who does the "if" address?* The human/author (halt, tell me) → out; the running test (do Y when X is on the page) → a real step.

**The rule: a prompt line states only what the test *does* (an action) or *verifies* (an assertion).** Everything else goes in a **stripped `<!-- comment -->` header** — the parser removes HTML comments, so a comment can never become a step (see §1 "HTML comments — the only metadata channel") — **or is omitted entirely**. Put the required-variables list, the expected duration, forward-dependencies, and any environment/config context in that comment header; leave the numbered body as pure actions and verifies.

```
✅  <!-- Auth: test account is 2FA-bypassed; email + password only, no second factor.
        Required preset variables: studioLoginEmail, studioLoginPwd (masked). -->
    Run the "Studio Smoke Login" component.
    Verify the Studio Home dashboard is displayed.

❌  Log in with the studioLoginEmail and studioLoginPwd variables.
    (this account is 2FA-bypassed)          ← config fact → generates as a step
    Verify no OTP or 2FA prompt appeared.    ← author note → generates as a bogus assertion

❌  Run the "Studio Smoke Login" component (username + password, no 2FA).   ← inline aside on a valid action line → still generates verbatim as a stray-comment step
✅  Run the "Studio Smoke Login" component.                                 ← "(username + password, no 2FA)" is fixture context → show-and-confirm note, never the prompt
```

**Distinct from #8.** #8 is about *how to mark* author metadata (use HTML comments, not blank lines). #25 is the failure that follows when you *don't*: the metadata **leaks into an executable step**. #8 gives you the channel; #25 says use it (or omit) rather than writing context inline.

**Login / auth-setup case (the motivating one).** The precondition that a login can actually complete — the account is **2FA-bypassed**, or its code reaches a channel the reader can read (an `@functionizeapp.com` mailbox, a provisioned SMS number) — is stated as **teaching prose** in `authenticating-test-users.md` (the one auth entry point), and confirmed at the acceptance gate (`functionize` → `verifying-a-created-test.md`). In the **prompt itself** that precondition is fixture context: put it in the comment header or omit it — never as a step, and never as a "verify there was no second-factor prompt" assertion the test wasn't asked to make.

**Recording a deliberate non-assertion — the *Intentionally NOT asserted* header note.** *"Or omit"* loses the *rationale*, so a later editor — or the create agent on a regenerate — re-adds the fragile assertion the author knew to avoid. When the dropped assertion is **non-obvious and tempting**, record it in the comment header as an *Intentionally NOT asserted* note — each dropped assertion with its one-line reason — instead of silently omitting it. The recurring shapes: an **auto-dismissing success banner** that can appear even when nothing persisted (the durable outcome verify is the real proof — #26 bans ending on a transient toast); a value the **acting role never entered or cannot see**; a **secondary-surface** confirmation whose re-read latency is unconfirmed, where the primary durable signal lives elsewhere; a default or field with **no documented surface to read**.

The **fact** that a signal is non-durable, role-invisible, or unsurfaced comes from the loaded **app-context / domain skill** (its durable-success-signal facts — move 4 of the core's *Decompose any flow*); this note is only the discipline of **surfacing** that fact consistently, not a place to restate app facts. It stays a **header note — never a body line** (that reopens the bogus-step trap above). Skip it when there is no such tempting exclusion; it is not a block every test carries.

```
✅  <!-- Intentionally NOT asserted:
        - the order's default warranty term — no documented surface shows it, so there is nothing to read.
        - the "Paid" badge on the orders list — a valid secondary confirmation, but on a different
          surface whose re-read latency is unconfirmed; the order detail page is the primary durable
          signal (verified above). -->
```

**Boundary with #5.** #5 tells you to state a genuine **start-state precondition the generation depends on** — *"(user is already authenticated)"*, *"No login required — browse as a guest"* — because it changes *which steps get generated* (whether a login step is added). That is a generation directive, keep it. The test for #25: does the note change *what steps exist* (a #5 start-state directive — keep) or is it a *fact / rationale / author-conditional* that shouldn't be a step at all (#25 — comment or omit)? *"2FA-bypassed"* does not change what the login test does (email + password either way); *"already authenticated"* does (skip the login). Adjacent: #5 (start-state that is intent), #8 (comments are the metadata channel), §3 (a conditional can't carry the behavior under test).

**#26 — A test whose terminal step is not a durable-outcome verify.** Every test's final meaningful step must be a **`Verify`** that asserts the durable result the test set out to produce — never a bare action, a soft wait, or a transient check. This is the authoring-side complement to the acceptance gate (the `functionize` skill confirms a created test *did* what was asked; this makes the test *able to say so*).

- **Why the verb matters.** A `Verify` step **smart-waits for the condition and records a hard assertion** (PASS or FAIL). A `Wait until` step **synchronizes only** — if the condition never appears it times out *softly* and the run still reports passed. Both wait; only a `Verify` asserts. A wait at the end of a test is a silent no-op — that is exactly what lets a login with an empty/unresolved credential report passed without authenticating. A terminal `Verify` turns that same "we never actually did the thing" into a hard, findable FAILED.
- **Content must be durable and outcome-specific** — content that exists only *after* the flow completed: the authenticated sidebar / protected content (login), the created record's identifier or a field value (CRUD), a confirmation total or status (checkout / submission), the rendered structural content (a marketing site). **Not** a generic element that also renders on an error / redirect page, a transient toast / spinner, or a URL pattern. This is what keeps the rule from collapsing into a hollow "append any verify" (a bare *"verify the page is displayed"* passes on an error page too).
- **Never itself conditional.** The terminal verify sits in the main flow, not behind an *"if"* — a verify inside a conditional asserts nothing whenever its condition is absent (the swallow trap, §3). Seed the precondition so the outcome is required, then verify it head-on.

```
✅  Verify the authenticated Studio Home is displayed — the left sidebar and Home landing content are present
❌  Wait until the authenticated Home is shown            ← wait: soft timeout, run still passes
❌  Click "Sign in"                                        ← last step an action, no outcome check
❌  Verify the page has loaded                             ← generic: passes on error/redirect pages
❌  Wait until the success toast disappears                ← transient
```

**Edge cases that still satisfy the rule:** negative-auth ends on a verify of the **error** state (*"Verify the login error 'Invalid credentials' is displayed"*); a **non-asserting step may trail the outcome verify only when it reconciles state that outlives the run VM** — isolation or cross-test plumbing: a persistent-state reset (an account-state reset; preferably a dedicated teardown or a separate teardown test per §1 *Section order* / §4(b) — never a trailing UI delete), a **#30** two-step project-variable hand-off write, or a **server-side session invalidation** (a concurrent-session or seat-cap logout). Such trailing plumbing may carry its **own** self-verify, which does not substitute for the outcome verify; a bare **client-side logout is not** such a step (the VM tear-down already clears the session), so it appears only when logout is itself the behaviour under test. A **structural site** with no back-channel ends on a verify of rendered content (*"Verify the article heading 'Q3 Earnings' and the first three paragraphs are present"*). **Distinct from:** #1 (don't verify after *every* click → this says DO verify once, at the end, at the outcome), #2 (not a vacuous omnibus → verify the *specific* durable result), #11 (not volatile/generic → verify *durable, outcome-specific* content). See also §2 "The verb decides whether a step asserts."

**#31 — Asserting a record is present or absent by reading the list as it sits instead of narrowing to the row.** (First own the row you assert on: a membership check that matches a pre-existing row by arbitrary text is a #20 violation, taught at §5(h).) A list, table, or grid that paginates (or lazy-loads / virtualizes its rows) shows only its **current page**, so a membership assertion read off the visible rows is blind to the rest. A **presence** check false-**fails** when the target sits on a later page (the `Verify` smart-waits, times out, and reports FAILED though the record exists). An **absence** check — the one you write after a delete — false-**passes**, silently: the row isn't on page 1, so *"verify no row named X"* goes green while X survives on page 2 and the delete may never have happened. **Absence is the dangerous half** — a false green that proves nothing (the pagination twin of #16's empty-container false-pass). Never assert membership against whichever rows happen to be on screen; **narrow to the target first, then assert on the narrowed result.** Strongest option first:

- **Verify at the record's own detail page** when it has a stable address and you hold its identifier — navigate directly by the captured id / URL (§5(h)) and assert there. Most robust: it bypasses the list *and* its search index, so neither pagination nor eventual-consistency (#22) can reach it. Use it whenever the real claim is *"the record exists / has this state"* rather than *"it appears in this list."*
- **When list membership is itself the point** (or there is no detail route), narrow the list and assert on a **rendered row**, not the container (#16):
  - a **search box** → type the record's **unique captured identifier** (the id, or the unique-suffixed name captured at creation — §5(h); a shared or partial value re-opens the multi-match trap), submit, **wait up to a bounded time for a matching row to appear**, then verify that row;
  - else a **column filter** on the identifying field — same bounded-wait-then-verify-the-row. (A **sort** only reorders — on an N-page grid the target still sits on whatever page its key falls, so sorting does not narrow; don't rely on it.)
- **Absence (after a delete) — the owned-absence pattern:** narrow by the same **unique captured identifier**, **wait up to a bounded time for its row to clear** (a delete is not always instantly reflected in a list or its index — #22), then assert the **positive empty-state** — *"the 'No matching records' message is shown"* — paired with a proof the filtered query actually ran (the search field still holds the value). A bare *"the list is empty"* / *"no row named X"* re-opens the §2 lone-negation trap: it also passes on a still-loading, errored, or never-rendered list.

Field pattern:
❌ *"Open the Orders list and verify a row for the orderId variable is present"* (sees the current page only) / *"verify no row for the orderId variable appears"* (hollow — passes when it's on a later page).
✅ *"Navigate to the order's page by the orderId variable and verify its status shows 'Cancelled'"* (detail page — no list, no pagination).
✅ *"Search the Orders list for the orderId variable, wait up to 30 seconds for a matching row to appear, then verify that row shows 'Cancelled'."*
✅ *"After deleting it, filter the Orders list by the orderId variable, wait up to 30 seconds for its row to clear, then verify the 'No orders found' empty-state is shown and the filter still holds the orderId variable."*

**Don't narrow-first when** the list, its pagination, or its sort order is **itself under test** (searching destroys the state you're checking), or for a **static, read-only browse** flow where relative selection is correct (#19). This governs a **specific, owned or known** record's membership — not volatile content asserted structurally (#16). Adjacent: **§5(h)** (find your owned record by its unique captured value), **#20** (own or pin the record you assert on), **#22** (a just-created/just-deleted row may not be in the list or its index yet — bound the wait), **#16** (assert a rendered row, not a container), **§2** (anchor an absence to a positive state).

**#32 — Asserting a gate denies without first confirming the precondition that makes the denial meaningful.** A permission or feature-gate test asserts the *wrong* outcome on purpose — a role gate, a feature-flag gate, or a permission boundary blocks the action, and the "blocked" state is what you verify. The trap this names sits **upstream of the denial**: a blocked-looking screen can come from a **broken precondition** rather than the access-control decision under test. Authentication is the usual culprit — a silently-failed login (an OTP that never arrived, a wrong credential, an expired session) leaves the app logged-out or on an error page, and navigating to the protected surface from there yields a redirect, a 404, or a "not available" screen **structurally identical** to the intended gate. The `Verify` passes; the test proved nothing about authorization — it exercised a failed login. **Fix: positively assert the precondition before inverting the verify.** Right after login, assert the authenticated state only a real session renders (the signed-in home shell, the session-bearing nav, a personalized element) so a login failure fails *there*, loudly, instead of masquerading as the gate downstream. **A second precondition applies whenever the verify asserts something *absent*:** the gated field/attribute/record/feature must actually *exist* — and, for a field, be *populated* — for a privileged persona, or the absence is **hollow** (a never-populated field reads identically to a suppressed one, so the test passes without exercising the gate). Pin a read-only fixture on which an admin *can* see the value; never seed a fresh, blank record for an absence test, because that record's missing value proves nothing about access control. The denial assertion itself is then an ordinary refusal test — assert its complement pair anchored to a positive state (the gate's own signal present, the gated action's controls absent on a page that rendered), exactly as the **Error-path / refusal tests** rule in §2 and the terminal-outcome rule (#26) already require; that half is not restated here. **Scope:** this governs authZ and feature-gate tests, where login is *meant* to succeed and access control is the thing under test. It does **not** apply to a bad-login (authN-failure) test — there the failed login *is* the outcome, there is no "intended persona authenticated" precondition to assert, and the refusal is verified per §2 / #26. The denial signal is whatever the app makes durable — a denial screen, a withheld control, an action left in an unapproved state — not necessarily a dedicated "not available" page. ❌ *"log in as a low-privilege user, go to the restricted area, verify the denial screen."* ✅ *"log in as that user; verify the authenticated home renders (login really succeeded); go to the restricted area; verify its denial per the refusal-test rule — the gate's own signal present and the gated action's controls absent on a page that rendered."* Adjacent: **§2** (Error-path / refusal tests — the complement-pair + positive-anchor rule this defers to for the denial half), #26 (a durable, outcome-specific terminal verify), #5 (a start-state precondition the generation depends on), #20 (own/know the persona you act as, never "any" user).

---

## 6. Course-correction — phrasing a corrective instruction

When a step is wrong, the failing run goes to the **Functionize agent**: it diagnoses what failed, and you and the agent decide the repair — regenerate, a surgical step/field edit, or delete-and-recreate. Neither the diagnosis nor the repair choice is the prompt author's to infer from the artifact — ask the agent, don't infer from the run (`functionize` → `references/diagnostics-and-maintenance.md`). What this section owns is the one prompt-craft part: **if the repair is a corrective instruction, phrase it as intent, not selectors** — the same craft as any prompt, scoped to the broken line plus one line of surrounding context. (A create-agent test can be repaired by editing the prompt and regenerating — which rebuilds all steps, discarding manual step edits — or by editing steps; a recorded test has no underlying prompt, so its fixes are per-step edits — a platform capability, not a judgment you make about a run.)

### Fixing "picked the wrong button" without mechanical selectors

**Bad fix (locks to brittle DOM):**
```
Click the button with text "Proceed to Payment" using CSS selector #checkout-payment-btn
```

**Good fix (intent-driven, updated instruction):**
```
Click the "Proceed to Payment" button in the checkout sidebar, not the "Continue Shopping" link in the header
```

The negation clause disambiguates without specifying mechanics — the step regenerates and targets the right element.

### Fix-prompt vs creation-prompt — what's different

| Aspect | Creation prompt | Fix prompt |
|---|---|---|
| Context | Self-contained | References existing workflow; only changes the broken part |
| Scope | Full workflow | Broken line(s) plus one line of surrounding context |
| Detail | Intent-level | More specific — disambiguation clues based on what went wrong |

**Example — fixing a wrong-button click:**

Original (the generated step clicked "Continue Shopping"):
```
Click the checkout button
```

Fixed instruction:
```
Proceed to checkout by clicking the "Proceed to Payment" button in the order summary sidebar (not the "Continue Shopping" link in the page header), then enter the shipping address
```

Adds: location clue, negation clause, and explicit action linking.

### Before/after examples

**1. Wrong button clicked:**

Before: `Click the checkout button` → the generated step clicked "Continue Shopping"

After: `Click the "Proceed to Checkout" button in the cart summary panel, not the "Continue Shopping" link in the navigation bar`

**2. Missing forward-dependency capture:**

Before — verify used a hardcoded number because no earlier step captured the total:
```
Verify the order confirmation page is displayed
Verify the confirmation shows the correct order total
```

After — change the preceding line first:
```
On the review page, verify the order total is not empty and set it as a local variable named orderTotal
```
Then: `Verify the confirmation total equals the orderTotal variable`.

**3. Switched verification mode (visual to functional):**

Before: `Verify the page layout matches the design` → generated a visual baseline comparison; the customer wanted functional checks.

After (one check per verify step — §2):
```
Verify the page heading shows the correct title
Verify the primary navigation links are present
Verify the footer shows the copyright text
```
