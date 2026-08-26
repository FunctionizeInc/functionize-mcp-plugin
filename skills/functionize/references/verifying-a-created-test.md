# Verifying a Created Test — read the generation status, then trust the execution

A generation status is not a verdict. A **generation** — an initial create, or a **maintenance** edit — reports **green or yellow, never red**: green means it generated fully and the test looks good; yellow means it generated with warnings and comes with a **message naming what's incomplete and why**. **Neither is a terminal "success"** — the test still has to run. So before a created test is done, **read the generation status and its message**. An **execution** — the run, after generation — *can* come back red, so a green execution is trustworthy. The whole model: **read the generation status; trust the execution's status.** This is not distrust of the agent — a generation status simply isn't a run yet.

## Generation

- A generation completes **green** (generated fully, test looks good) or **yellow** (generated with warnings, and the platform returns a **message naming what's incomplete and why**). Neither is terminal — the test still needs to be run.
- **Read the status and its message.**
- **Green** → run the test (next section).
- **Yellow** → the message names what's missing; ask the agent to **regenerate the named instructions**, then re-read.

## Execution

- **Green** → done, report success.
- **Yellow or red** → ask the agent to **run a diagnostic** on the test. When the diagnostic finishes, relay the **final result verbatim** to the user — never relay intermediate findings or partial output. A high-level progress note ("analyzing the test now") is fine; the diagnosis itself waits until it is complete, so you only ever relay correct, settled information.

## Maintenance

A **maintenance** request — the agent adds or removes steps on an existing test — is a **generation plus execution combined**: read its generation status the same way, and it **runs automatically after regeneration**, so you do not issue a separate run. Read the auto-execution's result and handle it exactly as above.

## Rules

- **No artifact audit. No independent back-channel read. No self-diagnosis.** The Functionize agent owns diagnosis; your job is to **read the generation status (and its message) and the execution's terminal result, and relay the agent's diagnostic** for a yellow or red. Reading the run's own terminal result (green/red) to report it is not self-diagnosis.

For the **underlying event signals** behind this flow — why `test_success`/`test_failed` are non-terminal and fire more than once, why `agent_done` marks the turn and not the test, and why a "pass" counts only with a real run timestamp — see `operational-notes.md` § "Reporting a created test's state". This file is the workflow; that section is the signal-level detail.
