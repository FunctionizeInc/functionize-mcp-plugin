# Diagnosing and Maintaining Tests — route to the agent, relay the result

The Functionize agent owns diagnosis and repair. Your job is to **route** a failing or warning test to it and **relay** its result — you do not self-diagnose. No result-code decision trees, no scoring the platform's confidence, no reverse-engineering a fix from a raw code by hand: the platform runs the analysis and its classification evolves with the runtime. This mirrors the acceptance gate (`verifying-a-created-test.md`): a green execution is done; a yellow or red one goes to the agent for a diagnostic, which you relay verbatim once it is final.

**The one exception is the connection itself.** If Functionize or the MCP is down or won't connect, there is no agent to route to — so here, and only here, you may help: offer connection-troubleshooting to get the session back (confirm the MCP server is running and configured, re-establish the integration, retry the session), framed as **restoring access**, not as diagnosing a test. This is scoped to the connection only — a failing run, an incomplete generation, or an environmental wall (a CAPTCHA, a Conditional-Access block) is not this case and still goes to the agent.

---

## My test went red — route it to the agent

1. **Note the *first* failed step's X.Y number** — the focus step. Failures cascade (one broken step makes the rest fail too), so the first red step is the real cause and the rest are fallout. Lead with it when you report.
2. **Ask the Functionize agent for a diagnostic** — describe the goal ("diagnose why test `<id>` in project `<id>` is failing") and attach a screenshot if you have one (upload it with `upload_session_file` and pass the returned `attachment_context_id` on your next message — see that tool's own description). It runs root-cause analysis and returns a read-only report naming the failing step, the cause, and a recommended fix.
3. **Relay the agent's diagnostic** — faithfully, and only once it is final. A high-level progress note ("analyzing the test now") is fine; the diagnosis itself waits until it is complete, so you only ever relay settled information. When you want the fix applied, hand the agent its own diagnosis so it doesn't redo the analysis.

## Maintenance — fix plus auto-validation

A fix (maintenance) is a **generation plus execution combined**: the agent applies the change and **re-runs automatically**, returning a report that states each change as before → after, includes a **restore point** for one-step undo, and confirms the validation run. You do not issue a separate run. Read the auto-run's result and handle it exactly as above — green is done; a yellow or red gets another diagnostic.

---

## Where your test results live in the UI

Orient by *what* you're looking for — exact labels and layouts change between releases:

- **PASS / FAIL status** — open a project's **Test Listing**; the **Browsers** column shows each test's latest-run status per enabled browser (PASS, FAIL, PENDING, INCOMPLETE, WARNING).
- **The steps** — open a test to see its steps, numbered in **X.Y form** (the canonical numbering; see `talking-to-the-agent.md` § "Step references").
- **The failed step** — open the failed test's **Browser** tab and expand the failed action for its error detail and the before/after screenshots.
- **The run detail** — each execution records its result, browser, environment, timestamp, and per-step outcome; use it to compare a failing run against the last passing one.
- **Live Debug** — watch a test run live on a clean VM to see exactly where and why it breaks.

---

## Capability facts

- **Self-healing** — tests auto-adapt to UI change; sensitivity is a **0–10 lever in Settings** (default 5, balanced) settable at team / project / test / action scope. To change it, ask the agent or set it in Settings; lower = more resilient but likelier to accept the wrong element, higher = stricter.
- **Restore points** — every change to a test is restorable for instant rollback; a maintenance report hands you the restore point. Not git-style snapshots — each restorable event captures a before/after delta.
- **Tolerance modes** — a test can continue past a failed step, or skip a dispensable one, instead of halting. Phrase the tolerance as **intent** ("if a promotional overlay appears, dismiss it and continue"), not by flag name — the phrasing craft is in `functionize-prompting`.
- **Control flow** — the platform has **first-class conditional** primitives (a real "if" — no native "else", so a two-way is written as two conditionals) plus loop primitives; there is **no native try/catch**. To build control flow, describe the intent to the agent — the conditional phrasing craft is in `functionize-prompting` §3.
- **Optimizer** — tunes a passing test's performance without changing its logic (see `advanced-features.md`).
