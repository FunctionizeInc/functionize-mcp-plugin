# Operational Notes — Agent-Session Shape, Reporting Accuracy, Gotchas

These are the practical things you'll trip over working with the Functionize agent session itself. Read this if a session is misbehaving or you're trying to figure out the most efficient way to interact.

---

## Agent-session shape

The agent session has three states that matter: it's working, it's waiting for your input, or it's finished. Don't judge a job by the session status — a session can show "working" when the underlying task has actually finished. Judge by the task's own durable result, not the session label.

While it's working (`running`), calling `send_agent_message` returns HTTP 400 — that's expected, not an error; wait until it's awaiting your input before sending.

**Wait on your own turn — never with a background timer.** Read the session's events (`get_agent_session_events`) on the fast-first-then-verb-bucket cadence in that tool's own description, which also says to cap the loop. For a single session you're blocked on, a blocking wait — where your integration offers one — is lowest-latency; when you're driving several sessions at once, read events on demand on your next turn instead. Either way, never spawn a detached/background timer (e.g. a background `sleep`) to pace the wait: each one fires a stale completion notification long after the work has moved on, so a loop of them buries you in false signal.

### Prereq

Agent session access must be enabled for your team. If it isn't, the agent-session tools return a "feature not enabled" error and the skill cannot function. Ask your Functionize representative to enable agent-session access.

### Judging create-completion

**A session-level completion signal is not "the test is done."** Settle a created test through the acceptance gate (`verifying-a-created-test.md`): read the generation status (green, or yellow with its message naming what's incomplete), then read the execution's own terminal result (PASSED / FAILED / ERROR with a real run timestamp). A session event or a `running` / `finished` label reports on the turn, not on the test.

---

## Reporting a created test's state

A **generation** reports green, or yellow with a message naming what's incomplete and why — read it; the full flow is `verifying-a-created-test.md`. Two facts about the session's own signals:

- **Generation reports green or yellow, never red; an execution can come back red.** A green generation means it generated fully — not that the test delivers the prompt's intent — so it is a starting point; a yellow generation returns a message naming what's incomplete, which you regenerate before running. A green **execution** is trustworthy.
- **`test_success` / `test_failed` are non-terminal and fire multiple times per session** — separately for **generation-complete** and **execution-complete**. So a single event is never "done," and the session `status` stays `running` through the post-generation pass — never gate on it. `get_agent_session_events` returns the agent's narration plus a terminal `agent_done` whose `status: success` marks the **turn**, not a test passing.

Then read the execution's own terminal result for the verdict:

- **A "pass" counts only with a real run** — a terminal status (`PASSED` / `FAILED` / `ERROR`) paired with a real, recent run timestamp. No run row, a status that never reached terminal, or a placeholder/sentinel timestamp means no real run happened — disregard the status.
- **A creation acknowledgement or a test ID is not "created."** Never report a test "created", "confirmed", "ready", or "done" off an acknowledgement or the presence of a test ID; settle it through the acceptance gate. (A test ID *is* enough to hand over the test link right away — SKILL.md § "Test and project links" — which is separate from reporting it done.)
- **When you can't confirm, hedge** — "generation still shows in-progress — NOT confirmed complete"; "reported passed, but no real run timestamp — not reporting a pass"; "edit issued, no confirmation returned — change NOT confirmed."

**When regeneration doesn't converge, ask the agent — don't infer the cause.** The acceptance gate's move for a partial generation is to have the agent regenerate the missing instructions (`verifying-a-created-test.md`). If the agent keeps reporting the same instruction as not generated across regenerations, why it won't generate is the **agent's** to diagnose. The complement is **pre-run**: before you write a prompt, the domain skills own the **premise check** — whether a target can exist in the application at all (e.g. `functionize-salesforce`'s object-model reality check: a Lead has no Contacts related list) — so an impossible target never enters the prompt in the first place.

---

## The create job is async — it outlives the session

A create/generation is an **async server-side job**: once submitted it runs and completes on the platform independently of the MCP client and the driving agent/session. Two consequences:

- **Stopping the session doesn't stop the job.** `stop_agent_session` ends the *session* — reach for it only when the user explicitly asks to cancel, never because you infer a session is stuck, idle, or slow; it's terminal (the session can't be resumed). It does **not** abort the async create running server-side, which completes regardless. So after any stop, read the generation status from the platform to see where the create actually landed — the session ending is not the job ending.
- **Losing the session doesn't lose a submitted create.** If the driving session dies mid-create — a crash, a rate/session limit, a disconnect — a create that already reached the server runs to completion there; the death doesn't abort it, so it's not lost or stuck "in-progress." Don't assume it's gone and re-create from scratch (that duplicates collateral and burns another generation/OTP run). Let the platform be the arbiter: list the project's recent tests (and components) newest-first — if the produced artifact is there, clear the acceptance gate on it (§ "Judging create-completion"); only if nothing new landed, treat the create as not landed and create.

---

## Common operational gotchas

- **Large tenants: don't ask the agent to "list everything."** A bare "list all my projects" on a big account returns an unwieldy dump. Narrow the request — by name, folder, or recent activity — so the answer stays focused.

- **One deep topic per turn gets the best answer.** Multi-part deep prompts dilute focus — narrow to one topic per turn for any deep technical question.

- **Refresh long sessions.** After many turns, start a fresh session to keep answers sharp.

- **Don't start ecommerce/search tests via Google.com** — it triggers reCAPTCHA. Start on the target site directly (Amazon, Target, etc.).

- **Don't create multiple login-dependent tests in parallel if they share a single-use credential (MFA/TOTP).** Generation runs trigger live logins concurrently and will clash at the MFA step (and can briefly lock the test user). Create them one at a time, or serialize via an Orchestration for repeat runs.

- **Pure agent tests run on the `live` environment by default.** Match what the user expects — if they think their test should run against `staging`, pass that explicitly. A newly onboarded app may not have a `live` environment at all — pin your actual target environment rather than relying on the default.

- **`fze datasource delete` is global** — it removes the datasource from ALL tests. Use `fze test tdm-detach` to remove a single test's mapping.

- **Native outbound notifications are broad** — email, SMS, Slack (with a separate failure channel), PagerDuty, MS Teams, and more, configured per-orchestration (Notifications tab) and in the team Alerts settings. Build a relay only for things with no connector (scheduled digests, bespoke dashboards). Full connector list + setup: `orchestrations-and-scheduling.md`.

- **A new session starts from a clean context.** Pass IDs and prior findings explicitly; to keep context instead, continue the existing session with `send_agent_message`.

- **The UI's X.Y step index is the canonical step reference.** A narrated "instruction N" can differ from the UI's X.Y numbering — e.g. narrated "instruction 4" may be UI "instruction 8". When you report a step number to a human, quote the UI's X.Y. See `talking-to-the-agent.md` § Step references.

- **Make a `PASS` mean what you intend — a test must verify a real value.** On dynamic-result pages (e.g. a POST-submit calculator whose result page can render blank), a verify step can end up asserting nothing while the run still reports PASS — a test-design matter: author the verify to assert durable content, not a transient or absent element. (See functionize-prompting anti-pattern `dynamic-result-page-false-pass`.)

- **A stored environment URL (e.g. `qa`) can differ from the test target.** The test runs against the start URL you provide in the prompt — set it explicitly to target the environment you intend.

---

## The `fze` CLI

The agent session (natural language) is the primary interface. The `fze` command-line interface is the alternative for scripted and CI/CD use — the same operations are also available through the UI or by describing intent to the agent. It's documented separately — follow the official Functionize CLI documentation for setup.

---

## Tool URL patterns to remember

For the **test and project link** forms (and when to surface them), see "Test and project links" in `SKILL.md` — the single source for those templates.

When linking to an orchestration, use the account's `<studio-host>` and placeholder IDs (as with test/project links):

- `https://<studio-host>/app/orchestrations?orchestrationId=<orchestrationId>` — the orchestration (opens its side panel)
- `https://<studio-host>/app/orchestrations/<orchestrationId>/runs/<runId>` — a specific run's results

The REST API base varies by deployment — confirm it against your Studio API Documentation rather than hardcoding a host. Treat `v4` as the current API version, and confirm any specific path against that documentation too.
