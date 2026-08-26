# Variables a Test Uses — kinds, scope, and where credentials live

The values a test carries live in one of five **variable kinds**, distinguished by lifetime and scope. You need these for three things: putting **credentials in variables**, **resolving a reference before you use it**, and phrasing prompts that reference data. What a test can otherwise *contain* — its full action surface — is in `capabilities.md`; the authoring craft (how to phrase it, the generator catalog) lives in `functionize-prompting`. You never hand-author the underlying syntax; describe intent and the Functionize agent materializes it.

---

## The five variable kinds

| Kind | Lifetime | Notes |
|---|---|---|
| **Execution-preset** | Human-set config, resolved per run from the selected preset | The modern home for the values a human sets: **secrets** (mark **masked**) and **per-environment config** (non-masked). Multiple presets per project; the preset is chosen at run time. Read-only from a test. |
| **Project** | Persists across runs, scoped to (project, environment) | The home for **inter-test data pipes** (a value one test writes, a later test reads) and the **fallback home for secrets** where presets aren't available. Can be marked secret. |
| **Local** | Single test execution; destroyed at run end | Scratch values captured and reused within one test. |
| **Prior-step capture** | Within the current test | A value captured from an earlier step's element (its text, value, an attribute) and reused downstream. |
| **TDM row** | Current row during a TDM orchestration run | The datasource row's columns; see `orchestrations-and-scheduling.md`. |

---

## A variable's home — which kind holds which value

Pick a variable's **home** by how the value is *set* and *used*:

| The value is… | Home | Mark |
|---|---|---|
| A **secret** a human sets (password, SSO/IdP password, MFA/TOTP seed, OAuth secret, API token, connection string, card number) | **Execution-preset variable** — a **secret variable** where presets aren't available | **masked** / **secret** |
| **Stable per-environment config** a human sets (org/instance URL, API base URL, region, a non-secret account) | **Execution-preset variable** — typically one preset per environment | non-masked |
| A value **one test produces at run time and a later test reads** (an inter-test data pipe in an orchestration) | **Project variable** — presets are **read-only** config a test can't write | secret if sensitive |
| A value **captured and reused within the same test** (an id, total, OTP) | **Local variable** | — |
| **Bulk / data-driven rows** (many logins, search terms) | **TDM datasource** | — |

**Database credentials are the exception — not a variable at all.** A database's host, name, and password live in a **platform database connection**, referenced by name from the query step (the create agent never sees the password); do not put them in a preset (`functionize-prompting` → `references/specialty-steps.md` §2). The "connection string" example above is the generic API/service case, not a database.

**Execution presets are the modern home for the values a human sets** (Project → Execution Presets). A preset is a named collection of runtime variables; a project can hold several (e.g. staging, production), and the preset is **selected at run or orchestration time**, so one test runs against many environments with no edit. Any preset variable can be **masked** — encrypted in the keystore and hidden in logs, screenshots, and run artifacts — which makes a masked preset variable the cleanest home for a secret. **Project-variable secrets remain fully supported** and are the home where presets aren't available.

**Reference a value at the scope where it lives.** A masked preset variable, a secret variable, and an inter-test project-variable pipe are different homes; a reference resolves only at the scope where the value actually sits. A same-named variable at the **wrong** scope (preset vs project) resolves **empty**, and the run can report passed without ever using the value — so confirm each referenced name *and its scope* before you rely on it (SKILL.md § "Resolve project-scoped identifiers"; `functionize-prompting` anti-pattern `resolve-before-you-reference`). This is about **scope-matching, not masking**: a value can be marked secret at either scope; the fix is to keep it at the scope the reference resolves at.

**Inter-test data pipes stay project variables.** Because a preset is read-only human-set config, a value a test *writes* at run time for a later test to read cannot live in a preset. Capture it into a local variable, then a **custom-JavaScript step** writes that local into a **pre-existing** project variable — native capture only ever creates a local. Project variables are otherwise **read** anywhere in a test. (Full pattern: `orchestrations-and-scheduling.md`.)
