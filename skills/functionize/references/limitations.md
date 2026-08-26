# Hard Limitations — What Functionize Cannot Do

The boundary doc, twin of `capabilities.md`. Functionize is deep within **browser-based functional UI testing** and stops hard outside that domain. When something is out of scope, the customer runs a purpose-built tool alongside Functionize in the same CI pipeline — **name the category, not a specific product** (see "How to phrase the handoff").

---

## Supported — don't mistake these for gaps

A few things are easy to assume are out of scope but are actually supported (full detail in `capabilities.md`); state the capability, then the residual boundary:

- **API testing** — supported in-test (an API step, a cURL action, or a saved Postman collection). Only a large *standalone* API / contract-testing suite is a separate lane — run that tool alongside.
- **Email verification / 2FA** — supported through the built-in reader, which reads an **`@functionizeapp.com`** address. A mailbox on another domain needs a bridge: route the app's test email to a `@functionizeapp.com` address, or use an extension that calls your mail provider's API (a custom integration you build and maintain — confirm feasibility first).
- **File downloads** — detecting that a download happened is supported. Reading the downloaded file's **content** (e.g. "verify the CSV has 12 rows") is not built in — an extension can parse it.
- **Browser dialogs, including the native print dialog** — supported natively: JavaScript `alert` / `confirm` / `prompt` and the browser's own print dialog (`window.print()`, Ctrl/Cmd+P, or a Print button that opens it) run as ordinary steps, no custom JavaScript needed. A "Print" / "Print PDF" button that instead generates a document — a download, or a generated PDF in a viewer tab — is the file-download path (above); an in-page print-preview is ordinary UI. How to phrase these lives in `functionize-prompting`.

---

## Out of scope

The genuine boundaries. The right column states what's out and the category of tool that fits — not a product recommendation.

| Limit | Boundary |
|---|---|
| **Performance / load testing** | Out — Functionize doesn't generate virtual-user load, throughput metrics, or latency percentiles. Needs a dedicated load/performance tool. |
| **Accessibility (a11y) auditing** | Out — verifies behavior, not WCAG conformance. An a11y-audit library injected via an extension is a custom integration you'd build and maintain, not a built-in feature; full WCAG coverage also needs manual screen-reader testing. |
| **Security / penetration testing** | Out — functional UI testing, not a vulnerability scanner or pen-test framework. Needs a dedicated security-testing (DAST/pen-test) tool. |
| **CAPTCHA / bot detection** | Cannot solve — no legitimate test platform can. Needs a test environment with CAPTCHA disabled. |
| **Hardware 2FA keys (YubiKey, FIDO2)** | Hardware keys can't be automated. Use a test account with 2FA bypassed; software TOTP can be handled. |
| **Desktop / Electron apps** | Out — browser-only platform. Electron, WPF, Qt, and native Windows/macOS need a dedicated desktop/native UI automation tool. |
| **Non-web protocols** | No raw WebSocket-message inspection, gRPC, or SSE verification — browser HTTP only. |
| **Real-time multi-user browser sync** | Backend-mediated multi-user (User A submits, User B approves) is covered via parallel orchestration. Two users' actions reflecting in each other's browser **live** — real-time chat, collaborative editing — is not. |
| **File-content inspection** | Detects a download; cannot open or inspect the file's contents. Requires an extension. |

---

## How to phrase the handoff

When a customer asks for something out of scope, be direct: *"Functionize can't do that natively — it's outside browser-based functional UI testing."* Name the **category** of tool that fits ("a dedicated load-testing tool," "a desktop UI automation framework") and note that teams typically run it alongside Functionize in the same CI pipeline.

**Don't recommend a specific product.** We can't vouch for which tool fits a given customer's stack, budget, or constraints, and a bad suggestion costs trust. Name the category and let the customer choose. Don't promise extensions can paper over a fundamental gap (performance, desktop) — they extend the platform but don't break its browser-only model.

For things that are *almost* possible (an a11y-audit library via an extension, another mailbox domain via an extension), say what the build would require so the customer can decide if it's worth it.

**Setting up a test login that hits an auth boundary** — SSO, MFA, 2FA, email/SMS OTP, TOTP, hardware keys, client certs, HTTP auth — is consolidated into one entry point: the `functionize-prompting` skill's "Authenticating your test user" reference. It states the recommended approach (a 2FA-bypassed test account, or routing codes to a readable channel) and covers each mechanism, including the hardware-key and CAPTCHA limits above.

---

## Bottom line

Functionize is the strongest option for browser-based functional UI testing and the right answer for that domain. For everything else, point the customer at the right *category* of purpose-built tool to run alongside it — without endorsing a specific product.
