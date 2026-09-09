---
name: security-check-frontend
description: Security review of frontend/client-side changes in any web project (Angular, React, Vue, Svelte, plain JS/TS). Use before raising or merging a frontend PR, or when asked to security-check UI, component, template, or browser code. Covers XSS, secrets, token handling, route/permission guards, redirects, browser storage, PII logging, dependencies.
---

# Frontend security check

Framework-agnostic. Step 1 detects the stack; the checks are the same, only the sink names change.

## 1. Detect stack

Read the manifest (`package.json`, `deno.json`, `composer.json`) for the framework, and note it in the report. Map the generic sinks to that framework:

| Generic sink | Angular | React | Vue | Plain JS |
|---|---|---|---|---|
| Raw HTML | `[innerHTML]`, `bypassSecurityTrust*` | `dangerouslySetInnerHTML` | `v-html` | `.innerHTML`, `insertAdjacentHTML` |
| Template/attr binding | `[href]` `[src]` | `href={}` `src={}` | `:href` `:src` | `setAttribute` |
| Navigation | `Router.navigateByUrl` | `navigate()`, `history.push` | `router.push` | `location.href` |
| DOM escape hatch | `ElementRef.nativeElement`, `Renderer2` | `ref.current`, `findDOMNode` | `$refs`, `$el` | `document.*` |

## 2. Scope

Default to **changed code only**:

```bash
git diff --name-only origin/HEAD...HEAD
```

No diff, or not a repo → review the paths the user named; if neither, ask. Read every file in scope. Never report on a file you did not read.

## 3. Checks

Grep for candidates, then **read the surrounding code**. A hit is a finding only if untrusted data — user input, API response, route/query param, `postMessage`, uploaded file, rich-text/WYSIWYG content, URL fragment — can reach it.

| # | Check | Look for |
|---|---|---|
| 1 | **XSS — raw HTML** | the framework's raw-HTML sink (table above) fed by anything but a literal |
| 2 | **XSS — sanitizer bypass** | explicit trust/bypass calls, `sanitize: false`, disabled escaping in a template engine |
| 3 | **Code eval** | `eval(`, `new Function(`, string-arg `setTimeout`/`setInterval`, dynamic `import(` of a variable |
| 4 | **URL injection** | `href`/`src`/`action`/`formaction`/`srcdoc` bound to a variable → must reject `javascript:`, `data:`, `vbscript:` |
| 5 | **Open redirect** | navigation or `location` assignment fed by a query/route param without an internal-path allowlist |
| 6 | **Secrets in client code** | API keys, tokens, passwords, private keys, connection strings in source, env/config files, or build output. Anything shipped to the browser is public |
| 7 | **Token handling** | access/refresh tokens in `localStorage`/`sessionStorage`/URL/non-`HttpOnly` cookie; auth headers attached to third-party origins; token in logs |
| 8 | **Sensitive browser storage** | PII (name, email, national/social id, address, financials, health) cached client-side without need; not cleared on logout |
| 9 | **Route/permission guards** | every new protected route has its guard/loader/middleware; lazy routes not left open |
| 10 | **Client-only authorization** | permission enforced only by hiding/disabling UI → flag that the server must enforce it, naming the endpoint |
| 11 | **Data in URLs** | PII, ids, or tokens in query strings (they land in logs, referrers, history) |
| 12 | **PII / token logging** | `console.*`, leftover `debugger`, analytics or error-reporting SDKs receiving personal data |
| 13 | **Uploads** | type/size checked client-side **and** mirrored server-side; uploaded HTML/SVG never rendered inline |
| 14 | **Cross-origin** | `postMessage` without an origin check; `window.opener` use; `target="_blank"` without `rel="noopener noreferrer"`; `<iframe>` without `sandbox`; CORS `credentials` to a wildcard origin |
| 15 | **Error surfaces** | server errors or stack traces rendered to the user |
| 16 | **Dependencies** | if the manifest or lockfile changed, run the ecosystem audit (`npm audit --omit=dev`, `pnpm audit --prod`, `yarn npm audit`) and report **high/critical only** |
| 17 | **Client-side validation** | present for UX, but never the only check — confirm a server-side counterpart exists |

## 4. Severity

- **Critical** — exploitable now: stored/reflected XSS reachable by another user, shipped secret, token exposure.
- **High** — exploitable with a precondition: open redirect, unguarded privileged route, PII in storage or logs.
- **Medium** — weakens defence: missing `noopener`, unsandboxed iframe, verbose errors, missing validation.
- **Low** — hardening.

## 5. Report

```
## Frontend security check — <scope> (<framework>)
Verdict: PASS | PASS WITH FINDINGS | FAIL
Files reviewed: N

| Sev | Check | Location | Issue | Fix |
|-----|-------|----------|-------|-----|
| High | Open redirect | src/auth/login.ts:42 | navigates to `returnUrl` from the query string | allowlist internal paths only |

### Verified clean
- <checks that ran with no findings>
```

Verdict: any Critical or High → **FAIL**. Only Medium/Low → **PASS WITH FINDINGS**. Nothing → **PASS**.

## 6. Rules

- State the concrete path from untrusted input to sink. If you cannot describe it, drop the finding — no pattern-match-only reports.
- Give each fix as a one-line diff or a named safe alternative (escape/sanitize API, text binding, guard, allowlist, `HttpOnly` cookie).
- Do not change code unless asked. If asked, fix and re-run the check on the result.
- Zero findings is a valid outcome — say so plainly instead of padding the table.
