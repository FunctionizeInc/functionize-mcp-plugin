# Action phrasebook — one copy-paste example per action type

The companion to `native-actions.md`: that file catalogs *what the platform does natively and its boundaries*; this one shows *what a great prompt line looks like* for each. Load it when you want a ready phrasing to adapt, or to check your line matches the house style.

Every line is **plain-English intent, never syntax** — describe what you want and the platform picks the action and materialises any token/operator. Replace `quoted labels`, `<placeholders>`, and variable names with the app's real values; project-scoped values (components, project variables, datasources) must already exist in the project — resolve before referencing (`SKILL.md` #21).

## UI interactions

```
Click the "Place Order" button
Double-click the "report.pdf" row in the file list
Right-click the "Sales Q3" node in the tree
Hover over the "Account" menu
Enter "wireless headphones" in the search field
Press Enter
Select "United States" from the Country dropdown
Choose the "Express" shipping radio
Check the "I agree to the terms" box
Uncheck "Remember me"
Leave "Subscribe to newsletter" as-is
In the Tags multi-select, choose "Priority", "Billing", and "VIP"
Drag the "Design spec" card onto the "Done" column
Sign the document on the signature canvas
Scroll to the "Pricing" section
Resize the window to 375 x 812
```

Typeahead / lookup — a decision rule tied to observable UI state, not a count:
```
In the Country field, type the full value "United States"; if it auto-completes to a single match, press Tab to confirm; if a suggestions dropdown appears, select "United States" from it
```

## Navigation & waits

```
Navigate to https://app.example.com/orders
Refresh the page
Click the browser Back button and verify the previous page is shown
Wait for the results table to appear, then click the first row
Wait up to 30 seconds for the "Processing" banner to clear
Wait until the Submit button is visible and enabled, then click it
```
Never a bare `Wait 5 seconds` — always anchor a wait on a visible element or state.

## Verification

```
Verify the order total reads "$29.99"                              (exact)
Verify the page shows "Order confirmed"                            (substring)
Verify the order ID matches the format ORD- followed by 8 digits   (pattern)
Verify the confirmation message is displayed on the page           (presence)
Verify the confirmation total equals the orderTotal variable       (against a captured value)
Verify the confirmation page loaded (it shows an order number) AND no error banner is displayed   (absence + positive anchor)
Verify the grand total is greater than the subtotal               (two on-page values, no capture)
Visually verify the checkout page region matches the baseline      (visual)
```

Refusal / error-path — assert the complement pair, anchored to a positive state:
```
Submit the login form with an invalid password
Verify the login page is still displayed (the email field is still visible), the error reads "Incorrect username or password", and the URL did not change to /dashboard
```

## Flow control & variables

```
If you see the "Session expired" dialog, click "Log in again"   (a conditional — its own line, runs only when the condition holds)
Dismiss the cookie consent banner if it appears                    (a "handle it" — vague or variant UI, described in English)
For each row in the results table, verify it shows a status of "Active"   (loop)
Delete each row until the "No items" empty-state message appears   (repeat-until)
Verify the order number is not empty and capture it as a local variable named orderId   (verify-capture an element read)
Enter the orderId variable in the search box and press Enter       (reuse a captured local)
Log in with the password from the masked preset variable named test_user_password   (masked preset variable)
Enter the password from the masked preset variable named admin_password   (masked preset variable)
After placing the order, verify the confirmation number is not empty and capture it as a local variable named orderId   (cross-test hand-off — step 1: native verify-capture to a local)
Run custom JavaScript to write the orderId variable into the existing project variable named last_order_id   (cross-test hand-off — step 2: custom-JS writes the local into the pre-created project variable)
```

## Context switching, dialogs, session state

```
Open the email reader in a new tab
Switch back to the application tab
Accept the confirmation dialog
Dismiss the alert
Enter "Test note" in the browser prompt
Set the session cookie from the masked preset variable named session_token
Set the local-storage key auth_state from the masked preset variable named session_token
```
Iframe-embedded fields (rich-text editors, embedded forms) are handled for you — describe the field by its visible purpose; you don't name the frame. Cross-origin payment widgets are the exception — name the frame by purpose; see `specialty-steps.md` §4.

## Generated values — say the shape; the platform generates it

```
Generate a random 8-character string and enter it in the Username field
Generate a random whole number between 1 and 100 and enter it in the Quantity field
Generate a random email on the functionizeapp.com domain and set it as a local variable named signupEmail
Generate a random 12-character alphanumeric string, capture it as a local variable named signupPassword, then enter it in the password and confirm-password fields   (a confirm field reuses the captured value — never re-generate)
Generate a random phone number in the format XXX-XXX-XXXX and enter it in the Phone field
Generate a random date of birth for an age between 18 and 65, formatted MM/DD/YYYY, and enter it in the Date of Birth field
Generate a random 9-digit numeric string and enter it in the Account Number field   (SSN/ZIP/account — keeps leading zeros)
Enter a random 8-character string in First Name and another random 8-character string in Last Name   (no native name generator — for realistic-looking names use a custom-code step: native-actions.md)
Enter a date 30 days from today in MM/DD/YYYY format in the Start Date field   (typed relative date)
Set the delivery date to 30 days from today using the date picker  (calendar widget)
```
Pin `functionizeapp.com` whenever a later step reads the email. Never enter a real credit card — there is no synthetic-card generator; a valid-format card is a designated test card supplied from a masked variable. Full random-data catalog in `native-actions.md`.

## API calls — native, never a custom-code `fetch()`

```
Call POST /orders on the Orders API base URL (a preset variable), sending Authorization: Bearer using the token from the masked preset variable named orders_api_token and Content-Type: application/json; the JSON body seeds one order line with quantity 2. Verify the response order id (response data.id) is not empty and set it as a local variable named orderId
Call GET /orders/{the orderId variable} on the same Orders API base URL with the same bearer token; verify the response status is 200 and the body's status field is "confirmed"
Call DELETE /orders/{the orderId variable} on the same Orders API base URL with the same bearer token; verify the response status is 200
```
The path, method, query params, response-field names, and status codes are literal contract — write them verbatim; describe only the request body in prose. (OData: `GET /sap/opu/odata/sap/API_SALES_ORDER_SRV/A_SalesOrder?$filter=SalesOrder eq '<the captured order number>'&$select=SalesOrder,SoldToParty`. Postman/cURL run via an Extension added in the UI — not generated from prompt text.)

## Database

```
Run a database query on the orders database connection (a project variable) to select the row where id = the orderId variable, and verify its status column is "shipped"
Run a database query to delete the seeded rows for the orderId variable   (dedicated teardown — never a trailing UI delete)
```

## Files — an upload needs a real, provisioned source (never a bare "upload a file")

```
Upload the attached file to the import form                        (a file attached at create time)
Upload the CSV file from the datasource to the import form         (a datasource fixture)
Click Export and download the CSV, then upload the downloaded file to the Import form   (download → re-upload)
Wait for the CSV download to complete and capture the filename, then verify the filename starts with "report-"
Upload the file using JavaScript                                   (fallback when the input is hidden/JS-driven)
```
Reading data *inside* a downloaded file needs an Extension.

## Email / SMS driven flows — `functionizeapp.com` reader only

```
Open the email reader in a new tab and wait up to 60 seconds for the verification email with subject containing "Verify your account"
Click the verification link in the email body
Open the SMS reader and wait up to 60 seconds for the OTP, then enter it in the code field   (needs a provisioned account phone number)
```

## Custom JavaScript — the fallback when no native path covers it

```
Run custom JavaScript to compute the expected total from the orderSubtotal variable — apply 8.25% tax, round to 2 decimals, format as "$1,234.56" — and set it as a local variable named expectedTotal
```

## Login / authentication

```
Run the "<your login component name>" component (ID <component-id-from-your-project>) to authenticate
No login required — browse as a guest
```
SSO/MFA/2FA/TOTP/OTP route through a login component or the email/SMS reader; a TOTP code has no typeable step. Credentials always come from **named masked variables** — a masked preset variable, or a secret variable where presets aren't available — never inline (`authenticating-test-users.md`).
