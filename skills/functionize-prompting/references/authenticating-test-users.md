# Authenticating Your Test User — One Entry Point

Almost every test starts by logging in. When the login has SSO, MFA, 2FA, client certificates, or HTTP auth, this page is the **single starting point**: it states the recommended approach and routes you to the reference that covers your mechanism.

> **Precondition — confirm the login can actually complete before you promise the test.** The account must reach a code channel Functionize can read — an **`@functionizeapp.com` mailbox** for email codes, a **provisioned phone number** for SMS — **or be 2FA-bypassed**. A login whose code can't be retrieved won't complete: the built-in email reader reads **only** `@functionizeapp.com` mailboxes, so on any other address it can't see the mailbox and times out — the OTP never arrives. End the login on a **verify of the authenticated state** (the signed-in home shell) — a login step alone doesn't prove authentication, but a verify of the signed-in state fails loudly if the app isn't logged in.

---

## Start here — the recommended approach

1. **Always put the credentials in named masked variables, never in the prompt.** This is non-negotiable and applies to every method below. The username, password, IdP password, OTP seed, client-cert material — all of it goes in a **masked execution-preset variable**, or a **secret variable** where presets aren't available, referenced only by name. See the "Credentials & secrets" rule in this skill's SKILL.md.

2. **Use a dedicated test account whose login is as simple as your app allows.** The most reliable test login is a plain username + password. Wherever your app permits it, give the test account a configuration that **skips the second factor** — a test user with **2FA bypassed**. This is the cleanest path and avoids the timing and provisioning complexity of live codes.

3. **If the flow must exercise a real second factor, route the code to a channel Functionize can read** — don't try to defeat it:
   - **Email codes** → send them to a `@functionizeapp.com` address so the built-in email reader can receive them.
   - **SMS / text codes** → the account needs a **provisioned phone number** configured first (a one-time account setup); then the built-in SMS reader can receive them.
   - **TOTP authenticator codes** → there is **no typeable TOTP step**; the code can't be entered inline. Route the whole login through a **login component** — see the required pattern below; prefer a 2FA-bypassed account if you can.

4. **What you cannot automate (design around it):** **hardware security keys** (YubiKey / FIDO2) can't be driven by any browser automation, and **CAPTCHA / bot detection** can't be solved by any legitimate test platform. For both, you need a test environment configured without them, or a test account exempt from them.

> **Capture a one-time code into a variable, then enter that variable.** When the login captures a one-time code (email/SMS 2FA), phrase it as a native verify-capture — *verify the verification-code element is not empty and capture it as a variable, then enter the stored variable* (#28, `fundamentals.md` #17) — so the enter step uses the live captured value, not a re-typed one. The ambiguous *"read the code and enter it"* can instead produce a one-time literal in place of a live read.

> **Required pattern for SSO + a second factor: name the login component.** When login combines SSO with any real second factor (MFA / 2FA / TOTP), make the **first step a login component, invoked by its exact full name** — it owns the entry URL, the SSO redirect, credential entry, and the second-factor branch. If you don't have the real name at authoring time, emit an explicit **placeholder** — *`the "<login component name>" component`* — and flag *resolve before running* (never leave the login vague, never invent the name — #21). **Do not fall back to typing the second factor inline** (e.g. *"enter the TOTP code from the secret"*): no built-in step types a TOTP code — the code must come from a login component (preferred) or a TOTP Extension.

> **One test = one authenticated session.** Log in once, as the persona the test is about. Don't log out and back in as a second user mid-test — that's a multi-flow test (SKILL.md anti-pattern #3). If a second actor is needed, seed their state or chain via an orchestration.

---

## By mechanism — where each is documented

| Your login uses… | Approach | Where it's written up |
|---|---|---|
| **Plain username + password** | Reference both from named masked variables (masked preset, or secret project where presets aren't available); verify the dashboard loads | `templates.md` → Template 1 (Login + dashboard verification) |
| **SSO redirect** (your IdP), no second factor | Describe the redirect explicitly: click "Sign in with SSO", enter credentials on the provider page, verify the app's dashboard page loads. **If the SSO login adds a real second factor (MFA/2FA/TOTP), use a login component instead** (the required-pattern rule above) | `templates.md` → Template 1, SSO variations |
| **Email 2FA / verification code** | the native email reader + multi-tab; the address **must** be `@functionizeapp.com`; add an explicit step to capture the code and reuse it | `specialty-steps.md` §4 ("Phrasing for 2FA login flow" / "The `@functionizeapp.com` constraint") |
| **SMS / text-message OTP** | the native SMS reader + multi-tab; **requires a provisioned account phone number** — resolve that it exists before promising the test (SKILL.md #21) | `specialty-steps.md` §4 ("SMS / text-message OTP") |
| **TOTP authenticator app** | No typeable TOTP step — route the login through a **login component**; prefer a 2FA-bypassed test account if you can | the required-pattern rule above; the `functionize` skill → `limitations.md` (TOTP-via-Extension) |
| **Hardware key (YubiKey / FIDO2)** | Not automatable — use a 2FA-bypassed test account | the `functionize` skill → `limitations.md` |
| **CAPTCHA / bot wall on login** | Not solvable — needs a test environment with it disabled | the `functionize` skill → `limitations.md` |
| **Mutual TLS (client certificate)** | Set the MTLS client key + certificate on the project's Auth tab (platform config, not prompt text) | the `functionize` skill → `project-settings.md` ("Auth tab") |
| **HTTP basic auth** | Set "Additional HTTP Authentication" per URL on the project's Auth tab; credentials still in named masked variables | the `functionize` skill → `project-settings.md` ("Auth tab") |
| **A reusable login flow shared across tests** | Invoke the login **component** by its exact full name (confirmed to belong to the target project, #21); a component carries its own entry URL | SKILL.md anti-pattern #21 + the component guidance in the Style rules |

---

## Two things that bite during login

- **The starting URL.** A login component carries its own entry URL — use it, and don't also set or default a starting URL (a created test otherwise defaults to the project/environment URL, which is frequently an IdP/portal host rather than the app). **And keep that entry URL live:** when the prompt names a project variable for the app URL, the generated first step must reference it, never a hardcoded literal (keep the entry URL live). Full rule: SKILL.md anti-patterns #5 and #17, `fundamentals.md` §5 (f).
- **State explicitly when no login is needed.** For genuinely unauthenticated flows (guest browse), say so explicitly — *"No login required — browse as a guest"* — or a login step you didn't ask for may be generated, filled with invented credentials to satisfy it (SKILL.md anti-pattern #5).
- **End on a `Verify` of the authenticated state — never a `Wait until`.** A login flow's terminal step must assert authenticated state (the sidebar, the user menu, protected content) with a **`Verify`**, not a `Wait until`. The distinction is load-bearing: a login that silently fails to authenticate (wrong credentials, a domain mismatch, an OTP the reader can't retrieve, a credential referenced at the wrong scope) times out *softly* on a `Wait` and the run still reports passed; a `Verify` of the authenticated state turns that same silent non-auth into a recorded FAILED (SKILL.md anti-pattern #26).
