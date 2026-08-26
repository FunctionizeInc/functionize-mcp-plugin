# Getting Started — Your First Test, Start to Finish

New to Functionize? This is the happy path: a single login test, from an empty project to a green run you can see in the UI. Get *this* working before you create anything else (`onboarding-new-app.md` § "First 1–5 tests" — why one-at-a-time beats a batch). The deeper "how" for each step lives in its own reference; this page connects them in order.

---

## What you'll do

1. Create a project and pick an environment.
2. Add your test user's credentials as **preset** variables (password masked, in Execution Presets).
3. Write the login prompt (start from a ready-made template).
4. Generate the test.
5. Watch it generate, then run.
6. Read the result and view it in the UI.

A login test is the right first test for almost any app: it's the shortest critical-path flow, and nearly every other test depends on being logged in.

---

## Step 1 — Create a project and pick an environment

A **project** groups the tests, variables, and settings for one application. An **environment** is the URL that project's tests run against (e.g. your staging or production instance).

- If your app has a single URL, one environment is all you need.
- If you deploy dev / staging / prod separately, create one environment per target — each gets its own URL and its own variable values.

Full decision walkthrough (timeouts, private-network access via ACS, browser choice, folder structure): `onboarding-new-app.md`.

---

## Step 2 — Add your test user as preset variables

Your test needs a login, but the password must **never** be written into the test text. Instead, store it once in the platform and reference it by name.

In **Project → Execution Presets**, create a preset and add two variables, marking the password **masked**:

| Variable name | Value | Mark |
|---|---|---|
| `test_user_email` | your test account's email | — |
| `test_user_password` | its password | **masked** (encrypted, hidden in logs and screenshots) |

You'll refer to these in the prompt *by name* — the test user's email and password — and the platform substitutes the real values at run time. This is the credentials rule the whole platform runs on (see SKILL.md "Credentials always go in variables — never plaintext").

> If execution presets aren't available in your project, add the same two variables in **Project → Variables** and mark the password **secret** instead. Everything below works the same.

**Which test account to use** — if your login has SSO, MFA, or 2FA, read `functionize-prompting`'s "Authenticating your test user" reference first; the short version is to use a test account with 2FA bypassed, or route the code to a `@functionizeapp.com` inbox.

---

## Step 3 — Write the login prompt

You describe the test in plain language — the **goal**, not the clicks. Functionize generates the steps. Start from the ready-made login template and adjust it to your app: **`functionize-prompting` → `references/templates.md` → Template 1 (Login + dashboard verification).**

It looks like this (swap in your URL and the labels your UI actually shows):

```
Navigate to https://app.example.com/login

Enter the test user's email into the email field and the test user's password into the password field — both are the masked preset variables you created in Step 2 (test_user_email, test_user_password)

Click the "Sign In" button

Verify the dashboard page loads and shows the navigation bar and primary content for an authenticated user
```

Notice what it does *not* contain: no CSS selectors, no "type into the third input," no fixed waits. That's intentional — intent-level prompts produce tests that self-heal when the UI shifts. For the why and the full prompt-writing craft, load the `functionize-prompting` skill.

---

## Step 4 — Generate the test

Hand the prompt to Functionize to turn into steps. Generating a test also produces its first execution — so you get the generated steps and a first run together.

---

## Step 5 — Watch it generate, then run

Generation is not instant; a simple login takes a short while, and longer flows take longer. While it runs, the test shows as generating. When it settles you have both a generated test and its first real run.

Don't treat the first sign of life as "done." A **green generation** means only that the steps were built — not that they pass; a test is finished only when generation has completed **and** the run has produced a terminal (green) result. If you're driving this through the agent session rather than the UI, the precise signals to watch — and the false-finishes to avoid — are in `operational-notes.md` § "Reporting a created test's state" and SKILL.md "Always remember" #4 (the acceptance gate).

---

## Step 6 — Read the result and view it in the UI

When the run finishes, the test reports a status:

- **PASSED** — the flow ran and its verifications held. Open the test to see each step with its screenshot and confirm the login actually reached the dashboard.
- **FAILED** — a step didn't pass. Open the failed step to see the screenshot and the reason.

Where each of these lives on screen — the status badge, the step list, the per-step screenshot, the run detail — is laid out in `diagnostics-and-maintenance.md` § "Where your test results live in the UI." If your first run is red, start at that file's § "My test went red — route it to the agent."

A green login test is done proving what you wanted when its **final step verifies durable authenticated state** — the dashboard actually loaded, not just a toast. Before you trust it as a template for the rest of your suite, clear the acceptance gate in `verifying-a-created-test.md` (read the generation status, then read the run), and confirm that terminal verify is in place. That's what makes the green trustworthy, and it saves you from building twenty tests on a shaky foundation.

---

## Where to go next

| You want to… | Go to |
|---|---|
| Set up timeouts, private-network access, browsers, folders | `onboarding-new-app.md` |
| Write more prompts (checkout, signup-with-2FA, data-driven) | `functionize-prompting` → `references/templates.md` |
| Handle SSO / MFA / 2FA on login | `functionize-prompting` → "Authenticating your test user" |
| Confirm a created test really did what you asked | `verifying-a-created-test.md` |
| Debug a red test | `diagnostics-and-maintenance.md` |
| Run tests on a schedule or in CI | `orchestrations-and-scheduling.md` |
