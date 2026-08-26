# Advanced Features — what each is for and when it applies

`capabilities.md` lists these as capabilities; this file is the **applicability note** for each — what it does, when to reach for it, and the one caveat that matters. You reach every one of them the same way: describe the intent to the Functionize agent. You do not hand-author the underlying mechanics — how to *phrase* each request (an `agent`-action prompt, component reuse, a wait as intent) is prompt craft and lives in `functionize-prompting`.

---

## Smart waits

A smart wait returns as soon as in-flight network activity settles, so it is faster than a fixed pause for a page whose readiness is network-driven (an SPA route change, a search, a submit-and-redirect). When readiness is **not** network-driven — a WebSocket/SSE stream, a CSS animation, content fetched-then-rendered later, or a page held up by slow tracking pixels — a fixed wait fits instead. Phrase the wait as the intent ("wait until the results load"); `functionize-prompting` covers the phrasing.

## Context switching

The platform auto-follows page loads within the same tab, but a **new tab or window, an iframe, or an OAuth popup** is a separate context it does not enter on its own — that needs an explicit switch (in and back out). Describe it as intent: "sign in through the popup, then return to the main window." Switching only moves the step's focus — it never signs anyone in or out. Any sign-in (through that popup, say) is a separate action from the switch. So a context switch alone cannot make the test act as a different user in the app under test. Because it only moves focus, a switch carries **no smart-wait** — a smart wait fires when a step interacts with an element, not when focus moves. So let the action that opens the new context finish before switching to it; the first interaction after the switch then gets its own smart-wait. A step can only find elements in its active context — an unswitched step is still looking at the previous one.

## The `agent` action — runtime AI inside a step

A step that reads the live page and decides what to do from a natural-language prompt, instead of a pre-determined target. Reach for it when you **cannot predict the element or path at generation time** — the layout varies per environment or region, the flow is multi-condition ("if a cookie banner is present, dismiss it"), or a single step would otherwise need many branches. When the target *is* predictable, an explicit step is cheaper and faster. Its prompt follows the same intent-first discipline as a create prompt (`functionize-prompting`).

## Optimizer

Acts only on **passing** tests and tunes performance (waits, screenshots, pacing) — it never changes test logic, deletes steps, or touches element selection. Fix a failing test first, then optimize once it passes.

## Components

Reusable, named step-sequences for a workflow shared across tests (a login, a checkout address-fill) — **not an extension**. Reach for one when a genuine multi-step sequence repeats across several tests; reference it by name and id. The reuse discipline — same page/same state, a different-shaped flow is a separate component rather than a parameter, don't over-use — is prompt craft in `functionize-prompting`.

## Extensions

Reach for an extension when native steps and a single API call aren't enough — multi-step API work, complex auth (OAuth mint/refresh, signed requests), transforming a response, parsing a file, or overriding pass/fail. **Separate from components:** extensions are custom code modules at runtime hooks, not reusable stored step blocks. Two flavors: built-in actions (a cURL call, a saved Postman collection) and custom serverless code. You **describe the extension you need** and the platform runs it; this skill never teaches you to write one. (What extensions are, and the languages and hooks they support, is in `capabilities.md`.)

## API and database steps

- **API step** — a single-shot HTTP call mid-test (any method, headers, auth via a secret variable), with the response carried into later steps. For anything multi-step or with complex auth, use an extension instead.
- **Database step** — run a query during execution and read the result into a variable, for an end-to-end "did the record actually land" check.

## Custom JavaScript steps

A custom-JavaScript step **reports its own result from what it returns**: a boolean (`true` = pass, `false` = fail), or a JSON-formatted string for a custom message — `return '{ "result": false, "message": "no rows found" }'` (`result` boolean, `message` string; a captured value can be interpolated into the message). Work that finishes **asynchronously** (an API follow-up, a timer) must signal completion through the built-in **`functionizeAsyncCallback(<boolean>)`**, so the step resolves only after the async work settles — a bare `await` does not reliably gate the result. (What custom JS is *for*, and when to reach for it, is prompt craft — `functionize-prompting`.)

## Visual validation

Reach for **step comparison** (against a prior step in the same run) to catch an action's unintended side-effects — did *only* the intended thing change? — which a functional check would miss; a **baseline** or **full-page** check confirms appearance against an approved reference. **The caveat that matters:** never lower the match threshold to force a failing visual step green — confirm first, from the screenshots, that the change is genuinely cosmetic; otherwise a lowered threshold masks a real regression. (The three modes and the tunable threshold are in `capabilities.md`.)
