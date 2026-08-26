# Ready-to-Paste Prompt Templates

Starting points for the most common test types. Edit the variables and labels to match your application.

**Before pasting any template:**

1. Create the variables listed in the header block. **Preset variables** go in Project → Execution Presets (mark secrets **masked**); **project variables** go in Project → Variables (used for inter-test data pipes, and as the secret home where presets aren't available). Which home a value belongs in: the `functionize` skill → `references/dsl-and-actions.md`.
2. Update URLs to point at your actual application.
3. Update element labels (button names, field names) to match what your UI actually shows.
4. Add or remove instruction lines for steps your app doesn't have.

Don't paste these verbatim — they're templates, not finished tests.

---

## Template 1 — Login + dashboard verification

The simplest critical-path test. Start here when onboarding a new application.

```
<!--
  Test name: "Login — Standard User"
  Expected duration: ~15–20 seconds
  Persona: Standard authenticated user
  Required preset variables (Project → Execution Presets; secret variables where presets aren't available):
    test_user_email     — non-masked (account identifier)
    test_user_password  — MASKED (encrypted; hidden in logs/screenshots)
-->

Navigate to https://app.example.com/login

Enter the email from the preset variable named test_user_email into the email field and the password from the masked preset variable named test_user_password into the password field

Click the "Sign In" button

Verify the dashboard page loads and shows the navigation bar and primary content for an authenticated user
```

**Variations:**

If your login uses a username instead of email:
```
Enter the username from the preset variable named test_username into the username field
```

If you need to skip a cookie banner on first visit:
```
Navigate to https://app.example.com/login
Dismiss the cookie consent banner if it appears
```

If your app has **SSO with no second factor** (Okta, Auth0, etc.) — describe the redirect explicitly:
```
Click "Sign in with SSO"
On the SSO provider page, enter the email and password from the preset variables named test_user_email and test_user_password (test_user_password marked masked)
Verify the dashboard page loads at https://app.example.com/
```

If your SSO login **adds a real second factor (MFA / 2FA / TOTP)** — a second factor can't be typed as an inline step, so route the whole login through a login component cited by its exact full name:
```
Log in by running the "<login component name>" component, which owns the entry URL, the SSO redirect, credential entry, and the MFA/second-factor branch
Verify the dashboard loads for the authenticated user
```
Resolve the component's real name in the target project before running — never leave the placeholder (#21). If no such component exists yet, create it once in Functionize (a component is built on the platform, not authored in a prompt), then cite it here.

---

## Template 2 — E-commerce guest checkout

Search → add to cart → checkout → confirmation. Captures product details for end-of-flow verification.

```
<!--
  Test name: "Guest Checkout — Standard Product, Credit Card"
  Expected duration: ~45–60 seconds
  Persona: Anonymous first-time buyer
  Required preset variables (Project → Execution Presets):
    checkout_email    — non-masked (test buyer email)
    checkout_card     — MASKED (a designated test card a real gateway accepts)
    checkout_expiry   — MASKED
    checkout_cvv      — MASKED
  Forward-dependencies:
    Cart line captures the product NAME → verified on the confirmation page
    Payment line captures the GRAND total (after shipping) → verified on the confirmation page
-->

Navigate to https://shop.example.com

Search for "wireless headphones" using the store search bar

Select the first product result and add it to cart with:
- Quantity: 1
- Size: Medium (only if a size selector is shown)

Proceed to the shopping cart page

Verify the cart shows the product name and capture it, setting it as a local variable named productName for later verification

Verify the cart line total equals the product unit price times the quantity

Click "Checkout as Guest"

Fill in the shipping address form:
- First Name: a random 6-character string
- Last Name: "FZE-" followed by a random 6-character string
- Street: 123 Test Street
- City: Springfield
- State: IL
- ZIP: 62701
- Email: the email from the preset variable named checkout_email

Click "Continue to Shipping"

Select the cheapest available shipping method and click "Continue to Payment"

On the payment page, verify the order grand total — the final amount due including shipping, not the subtotal — is not empty and set it as a local variable named grandTotal for later verification

<!-- Card fields sit in a cross-origin payment IFRAME (Stripe/Braintree/Adyen…). Name the frame by purpose here ("In the payment iframe, enter the card number…") — see specialty-steps "iframe / embedded content." -->
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

---

## Template 3 — Signup with email verification (2FA / confirmation link)

Registration flow that requires a verification email and clicking a link back to the app.

```
<!--
  Test name: "Signup — New User with Email Verification"
  Expected duration: ~60–90 seconds
  Persona: First-time visitor
  Notes:
    - Uses a functionizeapp.com email because the built-in email reader only handles that domain
    - Multi-tab flow: app tab + email reader tab
  Required preset variables (Project → Execution Presets):
    signup_password — MASKED; never hardcode the password in the prompt
-->

Navigate to https://app.example.com/signup

Fill in the signup form:
- First Name: a random 6-character string
- Last Name: "FZE-" followed by a random 6-character string
- Email: a freshly generated random email on the functionizeapp.com domain and set it as a local variable named signupEmail
- Password: the password from the masked preset variable named signup_password
- Confirm Password: the same password from the masked preset variable named signup_password

Accept the terms and conditions checkbox

Click the "Create Account" button

Verify the page shows a message asking the user to check their email

Open the email reader in a new tab and wait up to 60 seconds for the verification email with subject containing "Verify your account"

Click the verification link in the email body

In the opened tab, verify "Account verified" or "Welcome" appears

Log in with the signupEmail variable and the password from the masked preset variable named signup_password

Verify the dashboard loads
```

**Variation — 2FA code instead of clickable link:**

```
On the login page, enter the email and password from the login variables

Click "Sign In"

Verify the page asks for a verification code

Open the email reader in a new tab and wait up to 30 seconds for the 2FA email with subject containing "Your verification code"

In the email reader, verify the verification-code element is not empty and capture it as a local variable named otp

Switch back to the application tab

Enter the otp variable into the code field

Click "Verify"

Verify the dashboard loads
```

The "Enter the otp variable" step references the `otp` variable, not a re-typed value (anti-pattern #17).

---

## Template 4 — Forgot password / reset flow

Common but tricky — needs careful tab management.

```
<!--
  Test name: "Forgot Password — Standard User"
  Expected duration: ~45 seconds
  Required preset variables:
    reset_test_email — non-masked; uses a functionizeapp.com address so the email reader can pick it up
-->

Navigate to https://app.example.com/login

Click "Forgot Password"

Enter the email from the preset variable named reset_test_email into the email field

Click "Send Reset Link"

Verify the page shows a confirmation that a reset email has been sent

Open the email reader in a new tab and wait up to 60 seconds for the password reset email with subject containing "Reset your password"

Click the reset link in the email

In the opened tab, enter a new password — "NewPass_" followed by a random 4-character string and a "!" — into both the new password and confirm password fields and set it as a local variable named newPassword

Click "Reset Password"

Verify the page shows that the password has been reset

Log in with the email from the preset variable named reset_test_email and the newPassword variable

Verify the dashboard loads
```

**Important:** since the new password is randomly generated and only known within this test run, you can only verify the login works within the same test. The password is gone after the test ends. If you need persistent reset testing, capture the new password into a project variable via a custom-JavaScript step.

---

## Template 5 — Multi-step form with validation checks

A common pattern for onboarding wizards, application forms, or checkout flows that span multiple pages.

```
<!--
  Test name: "Account Setup Wizard — Happy Path"
  Expected duration: ~90 seconds
  Persona: New customer completing onboarding
-->

Navigate to https://app.example.com/onboarding

Verify the wizard shows "Step 1 of 4" and the "Personal Information" heading

Fill in personal information:
- First Name: a random 6-character string
- Last Name: "FZE-" followed by a random 6-character string
- Date of Birth: 01/15/1990
- Phone: 555-0100

Click "Continue"

Verify the wizard shows "Step 2 of 4" and the "Address" heading

Fill in the address:
- Street: 123 Test Street
- City: Springfield
- State: IL
- ZIP: 62701
- Country: United States

Click "Continue"

Verify the wizard shows "Step 3 of 4" and the "Preferences" heading

Select communication preferences:
- Email Updates: checked
- SMS Updates: unchecked
- Preferred Contact Time: Morning

Click "Continue"

Verify the wizard shows "Step 4 of 4"

Verify the "Review" heading is displayed

Verify the Review step shows Preferred Contact Time "Morning"

Click "Submit"

Verify the page shows "Account setup complete"

Verify an account number is displayed
```

**Validation-failure variation** — testing that the form blocks invalid input:

```
Navigate to https://app.example.com/onboarding

Try to click "Continue" without filling in any fields

Verify the wizard is still on "Step 1 of 4" (it did NOT advance) AND validation error messages appear under the First Name, Last Name, and Phone fields

Enter "abc" in the Phone field

Click "Continue"

Verify the wizard is still on "Step 1 of 4" AND a validation error under the Phone field contains "valid phone"
```

> **Formats & locale (fundamentals §1).** The DOB (`01/15/1990`), phone (`555-0100`), and any currency/number values above assume a **US-locale** render. For cross-environment runs, enter and verify in the app's displayed format — prefer the date picker over typing (state it with fallback latitude — #13), verify formatted numbers by a stable substring or pattern rather than an exact match, and parameterize locale-varying expected values into named non-masked preset variables (typically one preset per environment). The negative checks model the complement pair (§2): assert the forbidden transition did NOT happen *and* the specific error — never a lone "an error appears."

---

## Template 6 — Data-driven login test (using TDM)

Same login test, executed once per row in a TDM datasource. The prompt is intent-driven; TDM attachment happens after generation.

**Prompt:**
```
<!--
  Test name: "Login — Data-Driven (TDM)"
  Expected duration per row: ~15 seconds
  TDM datasource attachment (configure after generation):
    Column "email" → email input field's value
    Column "password" → password input field's value
    Column "expected_display" → dashboard greeting verification's expected text
-->

Navigate to https://app.example.com/login

Enter the email into the email field and the password into the password field

Click "Sign In"

Verify the dashboard loads and shows a greeting with the expected display name
```

**Companion datasource (login-test-users.csv):**

```csv
email,password,expected_display
alice@example.com,Pass1!,Welcome Alice
bob@example.com,Pass2!,Welcome Bob
carol@example.com,Pass3!,Welcome Carol
```

> A TDM datasource is the *only* acceptable home for multiple login credentials — they live in the data file, not the test instructions, and the prompt never names a literal value. Use disposable QA accounts, and upload the file as a datasource rather than committing real passwords to a repo.

**After test generation,** in the Functionize platform:
1. Upload the CSV as a datasource named `login-test-users`.
2. Find the generated test steps for email input, password input, and the dashboard verification.
3. Attach the matching columns.
4. Create a TDM-mode orchestration that points at this test and the datasource.

See `tdm.md` for the full TDM walkthrough.

---

## Template 7 — API setup + UI exercise + DB verification

End-to-end test that seeds backend state via API, exercises the UI, and verifies database state afterward.

```
<!--
  Test name: "Order Creation — API Seed + UI Verify + DB Audit"
  Expected duration: ~30 seconds
  Required preset variables (Project → Execution Presets; typically one preset per environment):
    api_base_url  — non-masked (per-environment API base URL)
    api_token     — MASKED (encrypted; hidden in logs/screenshots)
  Notes:
    Requires database connection "staging-db" configured in environment settings.
-->

Call POST /seed at the base URL from the preset variable named api_base_url, with an Authorization: Bearer header using the token from the masked preset variable named api_token, and body {"scenario": "fresh_customer", "tier": "premium"}; verify the response's customerId and sessionToken are not empty and set them as local variables named customerId and sessionToken

Navigate to https://app.example.com/orders/new, passing the sessionToken variable as the session query parameter

Verify the page loads with the customerId variable displayed

Select the product "Premium Widget" from the catalog

Set the quantity to 3

Click "Place Order"

Verify the order confirmation page is displayed and capture the order number and set it as a local variable named orderId

Query the "staging-db" database: SELECT id, customer_id, status, total FROM orders WHERE id = the orderId variable, and verify the customer_id column equals the customerId variable and the status column equals CREATED

Call DELETE on the /orders endpoint to clean up the test order — the base URL from the preset variable named api_base_url, the order id from the orderId variable, and auth via an Authorization: Bearer header using the masked preset variable named api_token
```

**SQL quoting note:** the example above leaves the captured order id un-quoted because the order ID is a numeric column in this hypothetical schema. If your `id` column is a string/UUID type, wrap the captured value in single quotes in the query (`WHERE id = '…'`). Mismatched quoting is a common silent failure — the query returns zero rows and the verify step fails with a confusing message.

---

## Template 8 — Visual regression baseline

Capture a baseline screenshot, then verify the same page matches it on future runs.

```
<!--
  Test name: "Visual — Checkout Page Layout"
  Expected duration: ~20 seconds
  Notes:
    First run establishes the baseline.
    Subsequent runs compare against it.
    To intentionally update the baseline (after a planned design change), use the platform's "Update Baseline" action on this step.
-->

Navigate to https://app.example.com/login

Log in with the email and password from the preset variables named test_user_email and test_user_password (test_user_password marked masked)

Add a product to the cart by navigating to https://app.example.com/products/123 and clicking "Add to Cart"

Navigate to https://app.example.com/checkout

Wait until the "Place Order" button is visible (the checkout page has finished rendering)

Visually verify the checkout page layout matches the baseline
```

**Visual-comparison variants:**

**Step-to-step comparison (compare two pages within the same test):**

```
Visit the cart page

Verify the cart summary is displayed and capture the page state for visual step comparison

Visit the checkout review page

Visually verify the checkout review page matches the page state captured from the cart page
```

Use this to verify information shown across multiple screens stays consistent.

**Region-only comparison:**

```
Visually verify the order summary panel on the right side of the checkout page matches the baseline
```

Naming a specific region helps the comparison ignore unrelated changes elsewhere on the page.

---

## How to adapt these templates

1. **Replace the URLs** — `https://app.example.com` → your application's URL.
2. **Replace the project variable names** — e.g. `checkout_email` → whatever you've named your variables.
3. **Replace the element labels** — "Sign In," "Place Order," etc. → whatever your UI actually shows.
4. **Add the `<!-- comment -->` header** documenting variables needed and forward-dependencies.
5. **Remove steps your app doesn't have**, but never delete the verification at the end.

If your test type isn't covered above, write a new one — describe the workflow in plain English and assemble the right pattern from these primitives.

---

## Domain-specific extensions

These templates are domain-agnostic. For some domains, dedicated sibling skills add **domain-shaped templates** (process catalogs, persona-role-gating, cross-module verification) on top of these primitives:

- **SAP S/4HANA Cloud testing** — see the **functionize-sap-s4hana** skill. Adds: process catalog for the 6 SAP E2E processes (O2C, S2P, R2R, D2O, L2C, R2R-HR); Fiori app + business role + document-type reference; cross-module FI verification template; tenant-config placeholders for SAP business roles and master data.

Three patterns from `functionize-sap-s4hana` generalize beyond SAP and are worth knowing:

1. **Cross-module verification** — when one system action posts to another system (logistics → financial accounting, e-commerce → ERP, CRM → marketing automation, app → analytics), verify the downstream document was created with the expected linkage and content. The **functionize-sap-s4hana** skill's references/templates.md §5 (cross-module verification) shows the shape.
2. **Persona-role-gating** — for authorization-aware apps, every test step is gated by a specific user role. Capturing which role drives which step in the `<!-- header -->` and at handoff points makes role-related test failures debuggable. See the **functionize-sap-s4hana** skill's references/tenant.md §4 (business roles assigned to test users).
3. **9-step prompt-expansion method** — for complex domains, a checklist of "what does the skill need to decide" before writing the prompt (process, scope, prerequisites, persona path, apps, document chain, verifications, gotchas, format) keeps expansions repeatable. See the **functionize-sap-s4hana** skill's § 9-step method.
