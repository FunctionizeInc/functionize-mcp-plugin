# Converting an existing Playwright or Selenium script into a Functionize prompt

You have a **test script** (Playwright or Selenium) and want the Functionize agents to generate the
equivalent test. Your job here is one thing: **read the script and lift it to the plain-English,
intent-driven flow** that *Decompose any flow* (SKILL.md) turns into the create-agent prompt. Conversion is a
**pre-step** — lift the script to intent, then run the normal decompose on that intent.

**Scope.** This is the *translation* craft only — script code → intent. It does **not** triage a suite,
bucket tests, gate on missing context, or manage batches; that is bulk-migration work the `functionize-playwright-migrate` / `functionize-selenium-migrate` skills own when installed. When you have one script (or a few)
pasted in a conversation, you convert it here and hand the intent to *Decompose any flow*.

## Read the assertions, not just the actions — or it isn't a test

A script is **actions + assertions**, and the assertions are what make it a test rather than a walk-through.
Carry them across honestly:

- **Carry what each assertion *proves* into a verify — at logical checkpoints, not after every step.** Tell
  the two apart by *what the assertion proves*: a **sync-gate** asserts the precondition for the next action
  (an element visible/enabled before you click it — the auto-wait idiom) → fold it into a smart-wait, or at
  most an anchored wait; a **checkpoint** asserts an outcome the flow produced → keep it as a verify. Don't
  emit a verify per line (defensive-verification-bloat, #1), cap a re-check of a value captured earlier at one
  per test (#12), and don't silently drop what an assertion proved. (The terminal verify is the durable
  outcome — *The converted output…* below owns that rule, #26.)
- **No durable assertion → surface it, don't ship it.** A script that asserts **nothing** — or only things
  also true on an error/redirect page (a URL fragment, `body` visible, a generic element) — converts to a
  green-but-meaningless test. Its Outcome slot is empty (readiness check, SKILL.md): say so plainly — *"this
  has no durable success check; what proves it actually worked?"* — and ask, rather than emit a walk-through
  dressed as a passing test.

## Unwrap the locator into the sentence it implies

This is the heart of reading a script. A locator that **carries a human-meaningful name — the text a user
sees, or the accessible name the author wrote** — lift straight into words:

| Script locator | Intent |
|---|---|
| Playwright `getByRole('button', { name: 'Sign in' })` | the **Sign in** button |
| Playwright `getByLabel('Password')` | the **Password** field |
| Playwright `getByText('Order confirmed')` / `getByPlaceholder` / `getByAltText` / `getByTitle` | the visible text / placeholder / alt / title string |
| Selenium `By.linkText` / `By.partialLinkText` (the anchor's visible text) | the **Sign in** link |
| **any** CSS/XPath whose predicate holds the visible text or accessible name — `//button[text()='Sign in']`, `//a[contains(.,'Order confirmed')]`, `[aria-label='Search']` | that name — lift it |

A **structural locator with no text or name carries no user-visible meaning** — a positional path
(`div > div:nth-child(3)`), a build-generated id (`#x7f3a`, an `id` locator), a bare class, or a
`data-testid`. You **cannot invent intent from it**: if the surrounding code makes the element's purpose
clear (a nearby label, the action that follows), describe that purpose; if it doesn't, **ask the user what the
element is** — never guess a structural selector into a confident-looking wrong step. (This is the
reading-direction of #6: a control's visible label *is* intent; a positional or structural target is not.)

## Drop the harness mechanics — delete, don't transliterate

A script is full of scaffolding that is **not a user step**. Transliterating it into prompt lines is the
single most common way to generate a garbage test (the platform already handles timing and setup).
**Delete**, don't translate:

- **Waits** — `waitForSelector` / `waitForResponse` / `waitForTimeout`, `WebDriverWait`, `sleep`: gone
  (Functionize smart-waits; write a wait only when it is a real user-observable condition — *Every wait needs
  an anchor*, SKILL.md). A wait that was the **only** proof a step succeeded is an assertion in disguise —
  keep it as a **verify** of the durable state it was gating on, not as a wait.
- **Fixtures / hooks / config / `test.step` grouping / `storageState` plumbing** — the *mechanism* drops, but
  keep the **intent** behind it: a login fixture → the login steps (or "the user is already signed in"); a
  DB/API seed → set up that data; env/config → a variable. A hoisted login that never appears in the test
  body is a **precondition**, not a per-step to re-type into every test.
- **JS-executor workarounds** — `page.evaluate` / `executeScript` used to force-click, `scrollIntoView`, or
  set a field's value: drop the JS and write the **plain user action** it stands in for (the platform handles
  interactability and scrolling; a transliterated executor is the mechanics-not-intent trap this whole skill
  refuses, #6). A JS `localStorage`/token auth-seed is the fixture case above. Only a genuinely **non-visible
  read** stays as custom JavaScript (#28).
- **Frame switches** — `frameLocator`, `switchTo().frame`: drop them; the platform enters an iframe from
  intent (describe the field by its visible purpose). The exception is a **cross-origin payment** widget
  (Stripe/Braintree/Adyen/PayPal) — name the frame by its visible purpose (`references/specialty-steps.md`,
  *iframe / embedded content*).

## Read the script's shape, not only its lines

Facts that live outside the action/assertion body and must still be lifted:

- **The entry point.** The opening `page.goto(url)` / `driver.get(url)` is the test's starting URL — lift it,
  so decompose's entry check (#5) resolves instead of re-asking the user for a URL the script already gave.
  (If the converted flow instead starts by running a login component, that component leads and carries its own
  URL — use it, per #5.)
- **A data loop.** A parameterized script — `test.each(rows)`, a `for` over a data array, a TestNG
  `@DataProvider` — is **one** intent test driven by a **TDM datasource**, not one prompt block per row. Write
  the single flow in plain intent and attach the data in the platform (#29, `references/tdm.md`).
- **An ordered / serial block.** `test.describe.serial(...)` / `test.describe.configure({ mode: 'serial' })` —
  or TestNG `@Test(dependsOnMethods=…)` — declares its tests run **in order**, later ones depending on earlier
  (the runner skips the rest once one fails). If they mutate shared state in sequence (create → edit → delete)
  they are **one dependent flow, not independent tests**: compose them into **one ordered test**, or — if too
  long to generate atomically — **segment and chain via an orchestration** (#9), handing each record's id to
  the next (#30). **Never convert the members as independent tests** — that severs the dependency and each
  later test runs with no upstream state. If they are serialized only to dodge a shared-**data** clash but are
  otherwise independent, convert each as its own **self-seeding** test.
- **Two actors, in sequence.** Two browser contexts (`browser.newContext()` twice) or a mid-test credential
  switch is **two users** — for a **sequential** hand-off (maker-checker, a role change), segment into one
  test per actor and chain them via an orchestration (#3 / #37); never a single context-switching flow. (Two
  users acting *at the same time* is the concurrent case below, not this.)

## Recognize what can't be faithfully converted — say so, don't fabricate

Most constructs convert; a few have **no faithful Functionize equivalent**, and for those, **tell the user
honestly** ("this asserts X, which Functionize can't do as written") instead of emitting a confident prompt
that generates a meaningless test. Genuinely unconvertible: an assertion on a **mocked / intercepted**
response (`page.route` / request interception — Functionize drives the *real* app), **arbitrary canvas pixel
reads** (`getImageData`), **pure unit logic** with no UI/API/DB surface, and a **truly concurrent multi-user**
script (two live sessions acting at once — Functionize drives one session; a *sequential* maker-checker split
is the *Two actors* case above, not this).

**Do not mistake these convertible cases for unconvertible ones:**
- A **real, non-mocked** API assertion (`request.get(...)`, `APIRequestContext`) → a native **API-call verify**
  (`references/specialty-steps.md` §1). Only the *mocked* response above is unconvertible.
- **Reading inside a downloaded file** (a CSV cell, a PDF value) → an **Extension** (`references/specialty-steps.md`
  §3); the native download check verifies file **metadata only** — name, size, extension — not contents.
- A **computed CSS / class / attribute** assertion (`toHaveClass('selected')`, an `aria-*` value) → lift it to
  the **visible state it proxies** ("verify the *X* tab is selected"); drop to **custom JavaScript** only for a
  genuinely non-visible residue (#28, `references/fundamentals.md` §4 (a)).
- A **screenshot / visual-snapshot** assertion (`toHaveScreenshot`, `toMatchSnapshot`, a Selenium image
  compare) → a native **visual validation** (phrase it *"visually verify…"*; `references/templates.md`
  Template 8), region-scoped away from volatile or seeded data (#16 / #20). A canvas's rendered *appearance*
  converts the same way — only a programmatic `getImageData` pixel read stays unconvertible.

## The converted output still obeys the anti-patterns

Lifting from a script does not exempt the result from the rules — apply them to what you emit:

- **End on the durable, outcome-specific result** the script proved — content that exists only *after* the
  flow completes, never a transient toast/spinner and never a generic element also present on an
  error/redirect page (#26).
- If the script selected **"the first" / "the most recent" / any** shared record, the converted test must
  **own its data** — seed-and-own or pin by id — not carry the flake over (#20).
- A negative check (`not.toBeVisible`) needs a **positive anchor** proving the page rendered, or it passes on
  a blank page (lone-negation, `references/fundamentals.md`).

## Then decompose

Once the script is lifted to intent — actions as steps, assertions as checkpoint verifies, mechanics dropped,
entry point and data loop captured, locators resolved or flagged — you have an ordinary flow. **Run *Decompose
any flow* (SKILL.md) on it** to produce the final create-agent prompt; the readiness check, the entry-point
(#5) rule, and the outcome rules take over from there. (An ordered chain that had to be segmented, or a
two-actor script, becomes several linked tests wired as an orchestration in the platform — not a single
prompt.) Conversion got you to intent; decompose takes intent to a prompt.
