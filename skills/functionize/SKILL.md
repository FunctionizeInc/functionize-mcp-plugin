---
name: functionize
description: Operate and reference the Functionize Agentic Studio test platform and its conversational agent session. Activate whenever the user mentions Functionize, the Functionize agent session, the agent-session tools, Functionize variables and the values a test reads, a test's steps and action types, a test's verifications and assertions, orchestrations, TDM (test data management), smart waits, context switching, the self-healing slider, visual validation, or asks "how do I talk to the Functionize agent" / "how do I diagnose this test failure" — including an existing test that is failing, flaky, broken or newly red, and any ask to fix, repair, review or diagnose one. Activate even when they don't explicitly name the platform but the context is clearly Functionize-related. For writing, drafting or refining the test prompts themselves, use functionize-prompting, or the relevant functionize domain skill — this skill is the platform-operations layer.
metadata:
  version: 2.11.4
  fze-role: ops
---

# Functionize

The Functionize Agentic Studio platform exposes a conversational **agent session** to the AI agent operating this skill through a set of MCP tools. The loop is three — `start_agent_session` (open one), `send_agent_message` (continue one), `get_agent_session_events` (read what it did); your integration also exposes tools to stream events, stop a session, list and fetch past sessions, attach a file, and list teams. Everything — creating tests, diagnosing failures, fixing tests, running orchestrations, building data-driven tests — is done by talking to that agent in natural language. Exact tool names and any prefix depend on how your client registers the integration (you may see them bare, e.g. `start_agent_session`, or namespaced); **each tool's own description is the source of truth for its arguments, lifecycle, and polling cadence** — consult it for call mechanics rather than this skill.

**For writing the test prompts themselves, load the `functionize-prompting` skill; if a domain skill exists for the target application, prefer it when installed.** Domain skills follow the naming pattern `functionize-<app>` (e.g. `functionize-salesforce`, `functionize-sap-s4hana`). This skill does three jobs: it maps what the platform **can and cannot do**, it drives **platform operations** (sessions, running, diagnosis relay, orchestrations, reporting), and it **feeds** the prompting and domain skills — which own prompt craft and domain specifics. Load them by name via the Skill tool.

---

## The one rule that matters most

**Intent, not mechanics.** When asking the Functionize agent to create or modify a test, describe what the user wants to *accomplish*, never which UI elements to click.

```
❌ Click the email field. Type testuser@example.com. Click password. Type secret. Click Login.
✅ Log in with the test user's credentials, stored as masked preset variables.
```

If you catch yourself writing "click the ___" or "type ___ into the ___ field," stop and rephrase as an intent — click-by-click prompts produce fragile tests that break on UI changes. This applies everywhere — in test creation prompts, in `agent`-action runtime prompts, in maintenance requests.

---

## Credentials always go in variables — never plaintext

When the user gives you a real credential or secret (password, API token, card number, connection string, OTP seed), **never** pass the literal value to the agent, put it in the test name, or repeat it back in chat. Auto-extract it into a **masked variable** and reference only the variable *by name*. The value is set once in the platform: the home is a **masked execution-preset variable** (Project → Execution Presets — encrypted in the keystore, hidden in logs, screenshots, and run artifacts); a **secret project variable** (Project → Variables, marked secret) is fully supported and is the home where presets aren't available. See `references/dsl-and-actions.md` for which home a value belongs in.

```
❌ "Create a login test for qa@acme.com with password Hunter2!Pass"
✅ "Create a login test that logs in with the test user's email and password"
   → these are the test_user_email and test_user_password credentials;
     tell the user to set them as masked preset variables (secret project variables where presets aren't available)
```

When you write any variable list or handoff for the user, mask the values (`••••`); pass the variable *names*. Bulk logins belong in a TDM datasource credentials column, never inline.

---

## Describe the goal, not the method

Match your verb to your goal (diagnose / review / fix / edit / create / optimize) and describe what you want to *achieve*, not how to do it — the verb→behavior map is in `references/talking-to-the-agent.md`, and the full prompt craft is owned by `functionize-prompting`. When diagnosing a failure, attach screenshots if you have them.

---

## Test and project links

**Link every test ID and project ID you write in prose — never a bare number.** When a reply mentions a test or project ID you can identify as such — one you just created, one you're diagnosing, a project named in passing — render it as a Markdown link (forms below), including when the ID opens a list item; link the first mention of each.

Two bounds keep this from misfiring:

- **Prose only.** Never wrap an ID inside a code fence, a DSL or create-prompt snippet, or a variable/TDM value — anywhere the text is meant to be pasted into the agent or copied verbatim; a Markdown link there corrupts the payload or shows as literal brackets.
- **Not every ID is linkable.** Leave a component, browser, or job ID plain — none has a Studio page. A **result / run-history** ID *does* have one — link it with the form below. An **orchestration** ID also has a Studio page but uses its own link form (`references/operational-notes.md`), not the templates in this section. If you can't tell an ID's type, leave it plain.

**Surface the test link the moment a test ID exists — don't wait for the run.** A create session emits a **test ID before generation and the run finish**. As soon as it does, proactively give the user the test link in that same turn — don't wait to be asked — so they can open the test in the UI while it generates and runs. Handing over the link is *not* reporting the test done: that still takes the acceptance gate ("Always remember" #4).

Give the **Studio** link (not the legacy app). The Studio host **varies by account/deployment** (e.g. `studio.functionize.com`, `agentic-studio.functionize.com`) — use the account's own host, shown below as `<studio-host>`. Substitute the real host so the link is clickable; never hardcode one or ship the literal placeholder. If you don't know the host, ask the user — or, for an incidental mention where asking isn't warranted, leave that ID plain.

- **Test:** `[<testId>](https://<studio-host>/app/projects/<projectId>/tests/<testId>)`; when you don't yet have the project ID, `[<testId>](https://<studio-host>/app/tests/<testId>)` also works (it redirects to the canonical project/test path).
- **Project:** `[<projectId>](https://<studio-host>/app/projects/<projectId>/)`
- **Result / run history:** `[<resultId>](https://<studio-host>/app/projects/<projectId>/tests/<testId>/browsers/<browserId>/results/<resultId>)` — the page for one specific run of a test; the same path ending `…/results/latest` opens the test's most recent run when you don't have a result ID. A chat or agent session that already has a **test ID** redirects to the test page, so hand over the test link (above) there, not this one.

---

## When to load each reference file

Read the appropriate file when the user's question touches its topic:

| When the user is… | Read |
|---|---|
| Asking how to talk to the agent, which verb to use, or what to pass each session | `references/talking-to-the-agent.md` |
| Asking what Functionize *can* do — features, browsers, integrations, scheduling (the capability map) | `references/capabilities.md` |
| Confirming a freshly *created* test actually did what was asked — the acceptance gate | `references/verifying-a-created-test.md` |
| Working out which variable kind holds a value (local/project/preset/TDM/prior-step) or where a credential lives | `references/dsl-and-actions.md` |
| Designing orchestrations, scheduling runs, wiring TDM, sharing data across tests, or integrating with CI/CD | `references/orchestrations-and-scheduling.md` |
| Diagnosing a failure, fixing a test, or working with self-healing and restore points | `references/diagnostics-and-maintenance.md` |
| Working with context switching, smart waits, the `agent` action, the optimizer, components, extensions, API/database steps, or visual validation | `references/advanced-features.md` |
| Brand new to Functionize and wanting a first-test, start-to-finish happy path (create project → secret variable → login prompt → generate → confirm at the acceptance gate → view in UI) | `references/getting-started.md` |
| Onboarding a brand-new app — project setup, environments, ACS, browsers, folder structure, first-test gotchas | `references/onboarding-new-app.md` |
| Tuning project/environment settings — timeouts, self-healing and timing sensitivity, authentication (MTLS/HTTP), the Advanced toggles | `references/project-settings.md` |
| Asking what Functionize *can't* do (performance, accessibility, desktop, etc.) — for honest handoff guidance | `references/limitations.md` |
| Hitting agent-session shape or operational quirks (session shape, reporting a created test's state, gotchas) | `references/operational-notes.md` |

Multiple references can apply at once — read them in parallel when needed.

---

## Prerequisites

This skill requires an authenticated Functionize agent session with agent-session access enabled for your team. If access isn't enabled, ask your Functionize representative. Verify the connection with: *"List my Functionize projects."*

---

## Always remember

1. **Supply the working set every session.** Pass project ID, environment, test ID, job ID, and any prior findings up front — a session works only from what you give it. A session that's *awaiting your input* is **continued** with `send_agent_message`, not restarted; the tool descriptions own the continue-vs-restart thresholds.
2. **Use X.Y step notation** ("step 3.2") when referencing steps — that's what the UI shows.
3. **One focused objective per session — fan out, don't serialize.** Scope each session to a single target; independent objectives run as **parallel sessions, one per target** (see `start_agent_session`). Multi-step chains on one target ("diagnose then fix") are fine; keep *unrelated* requests in separate sessions.
4. **A created test is "done" only when the generation is evaluated as faithful *and* the execution comes back green.** A creation acknowledgement, a test ID, or a "generated" message is a *starting point*, not proof: a **generation** reports green or yellow (never red), which means the generation itself completed — not that the test delivers the prompt's intent. (A test ID *is* enough to hand the user the link right away — see "Test and project links" — so they can watch it in the UI; that's separate from reporting it done.) So read the true state from the **generation status and its message**: green means it generated fully → have the agent **run** the test; yellow means it generated with warnings, and the message names what's missing → have the agent **regenerate the named instructions**, then re-read. A green **execution** is trustworthy (done); a yellow/red one gets an agent **diagnostic**, relayed verbatim once its analysis is final. A **maintenance** edit is a generation too: read its status the same way, and it runs automatically. Losing the session (a crash, rate-limit, or disconnect) does not stop a submitted create — it completes server-side, so recover a mid-create loss by listing the project's recent tests newest-first before re-creating. **Never** report a test "created", "confirmed", "ready", or "done" on an acknowledgement, a test ID, or a green *generation* alone. Full flow in `references/verifying-a-created-test.md`; session/reporting mechanics in `operational-notes.md`.
5. **Resolve project-scoped identifiers before you reference them; if one is missing, STOP and ask — never auto-pick or substitute.** Any concrete **project id**, **environment**, **component id/name**, a **project variable's name *and its value*** (or a **preset / execution-preset variable's name** — a preset value is encrypted and never seen or pasted), **datasource name**, or **test/job id** you see in a skill is an **illustrative placeholder, never a real value to paste into a live prompt or tool call.** Before creating/modifying a test that references these:
   - **(a) Pin the exact project (by id) and environment** in your message. If the project has exactly one environment, pin it without asking; only confirm with the user when there are multiple and the target isn't clear. If you don't pin, the agent proceeds against a default project that may not be the one you meant.
   - **(b) Confirm every referenced component and project variable actually exists** in that project, and that fixtures hold *real* values (not `TBD…`/placeholder) — list them to check, don't assume.
   - **(c) If anything is missing or ambiguous, STOP** — tell the user what's missing and the project id(s) you saw. Never invent a variable, substitute a value, or let the agent work against a nonexistent object or an auto-picked project. A missing fixture may be created as explicit setup *only* once its real value is confirmed; a missing component always stops for the user.

   This is the platform-side statement of `functionize-prompting` anti-pattern `resolve-before-you-reference`.

Everything else is in the references.
