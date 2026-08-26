# Talking to the Functionize Agent

Everything is done by chatting with the Functionize agent in natural language. This file covers what the agent can do, how to phrase requests, and what context the agent needs every time.

---

## The agent session

Supply the relevant identifiers up front (project ID, environment, test ID) — a new session works only from what you give it. A session left *awaiting your input* is **continued** with `send_agent_message`, not restarted; independent objectives run as **parallel sessions, one per target**. The session tools' own descriptions own the lifecycle — when to continue, restart, or fan out. Further session-behavior notes: operational-notes.md.

---

## Verb choice changes what the agent does

Match the verb to your goal — it's the single biggest lever on getting the right result in one turn.

| Verb / phrasing | What the agent does |
|---|---|
| "Why is test X failing?" / "What's the root cause?" | Read-only analysis — produces a root-cause report, makes no changes |
| "Review test X" / "Does it actually cover the flow?" | Read-only analysis of a test that is not necessarily failing — the agent reads the test and reports what it does and does not cover. Relay the report; the judgement is the agent's, not yours |
| "Fix test X" / "Make it pass" | Diagnosis + repair + validation re-run |
| "Change step 5 to…" / "Add a wait after step 3" / "Update the URL" | Direct edits to specific actions / settings |
| "Create a test that…" | Creates a new test from instructions |
| "Make test X faster" | Optimizes a passing test — tunes waits and screenshots without changing test logic |
| "Update the runtime version" | Bumps the runtime version |

If you're not sure whether you want diagnosis or a fix, ask for the fix — a fix request includes diagnosis as a first step.

---

## What to pass every session

Include the relevant identifiers up front — a new session works only from what you provide:

- **Project ID** — unless there's only one project on the team
- **Environment** — defaults to `live` if you don't say. Common alternatives: `staging`, `prod`, `sandbox`. Match what the user expects — if they expect `staging`, pass it explicitly.
- **Test ID** — for any operation on an existing test
- **Job ID / Exec ID** — for execution-specific analysis (diagnosing a particular run)
- **Prior findings** — if a diagnostic already ran and you now want a fix, paste the diagnostic conclusion so the fix doesn't redo it

---

## Step references — use X.Y notation

Use **X.Y notation** (e.g. "step 3.2") when referencing steps — that's what the UI shows. Flat ordinals ("step 18") are the internal numbering, which differs from the UI's.

**The focus step** is the single step that names the gap — the first failing step, or the first zero-step / stalled instruction — the one you lead with when you report a REWORK or a stopped generation. Quote it in X.Y form. ("Lead with the focus step X.Y" elsewhere in this skill means this step.)

**When relaying a step/instruction number to a human, anchor it to the UI's X.Y index.** A narrated *grouped* count can differ from the X.Y the user sees — what's narrated as "instruction 4" may be "instruction 8" in the UI. Pass the narrated number straight through and it lands wrong in a bug report. So before you hand a number to a person (bug report, status update), **confirm it against the test's step list / X.Y index** and quote that; if you only have the narrated number, say so and note it may differ from the UI.

---

## Prompt length guidance

| Task | Sweet spot |
|---|---|
| Test creation | 100–300 words |
| Diagnostics / maintenance | 50–150 words |

Under 30 words → too little to target precisely. Over 500 → usually means mechanics are sneaking in.

---

## The #1 rule: intent, not mechanics

**Never** write click-by-click for test creation — mechanical steps produce fragile tests that break on UI changes. This applies everywhere you phrase a prompt — test-creation *and* runtime `agent`-action prompts. **For the full prompt-craft treatment — how to phrase an intent, verification phrasing, capture/reuse, data isolation, and the anti-patterns — load the `functionize-prompting` skill**, which owns it. This file stays on the operations side: which verb to use, what to pass, and how to route a diagnostic or maintenance request.

---

## Diagnostic prompts — what to include

| Field | Why |
|---|---|
| Test ID | Required |
| Timeline ("started failing yesterday; last pass May 19") | Enables comparison against the last good run |
| App-side changes you know about ("header redesign deployed Monday") | Narrows the search space |
| Manual-testing data point ("works fine in the browser") | Eliminates real-app-bug as the root cause |
| Failure-location hint ("seems to fail before checkout loads") | Narrows where to look |
| Environment | Removes the lookup |

### Mediocre

> "Test {TEST_ID} failed. Can you tell me why?"

### Great

> "Diagnose test {TEST_ID}. It's been passing for months but started failing consistently as of yesterday (May 20). Last successful run was May 19. The app team deployed a new header/footer redesign on Monday — not sure if related. On manual inspection the checkout flow works fine; the failure seems to happen somewhere before checkout loads. Environment: staging."

---

## Maintenance prompts — hand off prior diagnosis

If a diagnostic already ran, paste the conclusion into the maintenance request so the agent doesn't redo it.

### Mediocre

> "Fix test {TEST_ID}."

### Great

> "Fix test {TEST_ID}. A diagnostic already ran — step 3.2 (the 'Add to Cart' button click) is failing because the new header redesign pushed the button lower on the page and the step is now matching a different button in the new nav bar. The diagnostic recommended adjusting the element selection on step 3.2. Please fix and run a validation execution to confirm."

---

## Common preventable failure patterns

- **Asking "what happened?" when you want a fix.** That asks for read-only analysis — use "fix" to get the repair.
- **Wrong step numbering** — UI shows X.Y, internal numbering is different. Use the UI form.
- **Over-specifying the method instead of the outcome** — just describe what you want to happen and let the agent carry it out.
- **Forgetting the project ID** in a multi-project team.
- **Assuming a new session remembers a past one** — a fresh session starts blank; to keep context, *continue* the existing session with `send_agent_message` rather than starting over.
- **Inventing credentials, URLs, or generator functions** that don't exist. Pass real values; the supported generator set lives in `functionize-prompting`.
- **Multi-topic deep prompts in one turn** — one deep topic per turn gets the sharpest answer.

---

## What you get back

The Functionize agent returns its analysis as a report — a diagnostic names the failing step and a recommended fix; a maintenance report states each change and includes a restore point. Your job is to relay it faithfully, and only once it is final; see `references/diagnostics-and-maintenance.md` for how to route a failure to the agent and relay the result.
