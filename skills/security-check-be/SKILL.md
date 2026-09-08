---
name: security-check-be
description: Security review of backend/server-side changes in any project (.NET, Node, Java, Python, Go, PHP). Use before raising or merging a backend PR, or when asked to security-check an API, controller, service, query, handler, or migration. Covers authn/authz, IDOR and tenant scoping, injection, secrets, crypto, data exposure, SSRF, uploads, dependencies.
---

# Backend security check

Framework-agnostic. Step 1 detects the stack; the checks are the same, only the sink names change.

## 1. Detect stack

Read the manifest (`*.csproj`/`*.sln`, `package.json`, `pom.xml`/`build.gradle`, `requirements.txt`/`pyproject.toml`, `go.mod`, `composer.json`) and note language, framework, data store, and auth mechanism in the report. Then locate the request entry points: controllers/routes/handlers/resolvers/consumers/jobs.

## 2. Scope

Default to **changed code only**:

```bash
git diff --name-only origin/HEAD...HEAD
```

No diff, or not a repo → review the paths the user named; if neither, ask. Read every file in scope, plus the auth/middleware setup it depends on. Never report on a file you did not read.

## 3. Checks

For **every new or changed endpoint**, answer checks 1–3 explicitly — they are where real breaches come from.

| # | Check | Look for |
|---|---|---|
| 1 | **Authentication** | endpoint requires auth, or is deliberately public. Every anonymous/unauthenticated route needs a stated reason. Webhooks must verify a signature or shared secret, not just a hard-to-guess URL |
| 2 | **Authorization** | role/permission/scope enforced server-side for the specific action, not merely "logged in". Write, export, bulk, and admin actions checked separately from read |
| 3 | **IDOR / tenant scoping** | every query and mutation filtered by the caller's tenant/org/owner, not only by the id from the request. Ids from the client are never trusted as authorization |
| 4 | **Injection** | string-concatenated or interpolated SQL; raw/`$where`/JS-eval NoSQL operators; user data merged into a query document; shell/process execution; LDAP; XPath; template rendering of user input. Require parameterized queries |
| 5 | **Mass assignment** | request DTO bound straight onto an entity, letting the client set `role`, `isAdmin`, `tenantId`, `price`, `status`; unvalidated `PATCH`/merge |
| 6 | **Input validation** | required fields, types, lengths, ranges, enums validated at the boundary; pagination capped; no unbounded list/export |
| 7 | **Path traversal / uploads** | user-supplied filename or path in file APIs; content type, extension, and size enforced server-side; uploads stored outside the web root and never executed |
| 8 | **SSRF** | outbound HTTP/fetch/webhook/import target derived from user input without a host allowlist; redirects followed to internal addresses |
| 9 | **Secrets** | credentials, keys, tokens, connection strings hardcoded or committed in config/appsettings/env files; secrets in logs or exception messages; secrets checked into infra manifests |
| 10 | **Crypto** | MD5/SHA-1 or unsalted hashes for passwords (use bcrypt/argon2/PBKDF2); non-cryptographic RNG for tokens, ids, or OTPs; hardcoded key/IV; ECB mode; disabled TLS/certificate validation |
| 11 | **Data exposure** | responses returning whole entities with sensitive fields (hashes, internal ids, other users' data); stack traces or SQL errors returned to the client; API docs/debug endpoints reachable in production |
| 12 | **Deserialization** | polymorphic/type-name deserialization of untrusted JSON, XML external entities, native binary deserialization, unsafe YAML |
| 13 | **Rate limiting / abuse** | login, password reset, OTP, search, export, and public endpoints throttled; no unauthenticated resource-expensive work |
| 14 | **Session & tokens** | token lifetime and revocation; refresh-token rotation; cookies `HttpOnly`, `Secure`, `SameSite`; logout invalidates server-side state |
| 15 | **CORS & headers** | wildcard origin combined with credentials; overly broad allowed origins/methods/headers |
| 16 | **Audit & PII logging** | privileged and personal-data access is auditable; logs do not contain PII, tokens, or full request bodies |
| 17 | **Dependencies** | if the manifest or lockfile changed, run the ecosystem audit (`dotnet list package --vulnerable --include-transitive`, `npm audit --omit=dev`, `pip-audit`, `mvn dependency-check`, `govulncheck`) and report **high/critical only** |
| 18 | **Data-layer changes** | new migration/collection/index — is the data classified, scoped by tenant, and does it need encryption at rest? Destructive migrations reversible? |

## 4. Severity

- **Critical** — unauthenticated or cross-tenant access to data or actions, injection, RCE, committed production secret.
- **High** — missing authorization on a privileged action, IDOR within a tenant, weak password crypto, SSRF, sensitive data in a response.
- **Medium** — missing rate limit, weak validation, PII in logs, permissive CORS, verbose errors.
- **Low** — hardening.

## 5. Report

```
## BE security check — <scope> (<stack>)
Verdict: PASS | PASS WITH FINDINGS | FAIL
Endpoints reviewed: N

### Endpoint matrix
| Endpoint | Auth | Authorization | Tenant/owner scoped |
|----------|------|---------------|---------------------|
| POST /orders/export | JWT | ⚠ role not checked | ✅ |

| Sev | Check | Location | Issue | Fix |
|-----|-------|----------|-------|-----|
| Critical | IDOR | src/Api/OrderController.cs:88 | loads order by client id with no owner filter | add owner/tenant predicate to the query |

### Verified clean
- <checks that ran with no findings>
```

Verdict: any Critical or High → **FAIL**. Only Medium/Low → **PASS WITH FINDINGS**. Nothing → **PASS**.

## 6. Rules

- Trace the actual path: who can call it, with what input, to reach what data. If you cannot state that, drop the finding.
- Never treat a client-supplied id, tenant, role, or price as trusted, and never count frontend validation or a hidden UI control as a control.
- Give each fix as a one-line diff or a named framework mechanism (auth attribute/middleware, parameterized query, explicit DTO, allowlist, `RandomNumberGenerator`).
- Do not change code unless asked. If asked, fix and re-run the check on the result.
- Zero findings is a valid outcome — say so plainly instead of padding the table.
