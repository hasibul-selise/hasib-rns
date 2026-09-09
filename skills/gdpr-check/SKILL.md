---
name: gdpr-check
description: GDPR / data-protection review of code changes in any project. Use when a change touches personal data — new fields, forms, exports, emails, logs, analytics, cookies, third-party SDKs, retention or deletion logic — or when asked to check GDPR, privacy, consent, PII handling, or data-subject rights compliance.
---

# GDPR / data-protection check

Engineering-level review. It flags compliance risk in code; it is not legal advice. Anything ambiguous goes to the DPO / legal owner, named as such in the report.

## 1. Scope

Default to **changed code only**:

```bash
git diff --name-only origin/HEAD...HEAD
```

No diff, or not a repo → review the paths the user named; if neither, ask.

## 2. Find the personal data

Before judging anything, list what personal data the change touches. Personal data = anything relating to an identifiable person, including indirect identifiers (user id, device id, IP, cookie id).

Scan the diff for: models/entities/DTOs, migrations and schemas, form fields, API payloads, exports and reports, email/SMS/push templates, log and telemetry statements, analytics and tag scripts, cookies and browser storage, and third-party SDK or API calls.

Mark **special-category** data explicitly — health, biometrics, genetics, race/ethnicity, religion or belief, political opinion, trade-union membership, sex life or orientation — plus criminal-offence data and children's data. These need a stronger basis and tighter handling.

If the change touches no personal data, say so and stop. That is a complete, valid result.

## 3. Checks

| # | Principle | Check |
|---|---|---|
| 1 | **Lawful basis & purpose** | each new data element has a stated purpose and a basis (consent, contract, legal obligation, legitimate interest). Data collected for one purpose is not silently reused for another |
| 2 | **Data minimisation** | every new field is necessary. No "collect it in case we need it", no full payload capture where an id would do |
| 3 | **Consent** | where consent is the basis: freely given, specific, granular, opt-**in** (no pre-ticked boxes, no bundling), recorded with timestamp/version/scope, and withdrawable as easily as it was given. Marketing/newsletter sends check a current opt-in |
| 4 | **Cookies & trackers** | non-essential cookies, pixels, analytics, session recording, and third-party scripts load **only after** consent; essential-only by default |
| 5 | **Transparency** | new processing is reflected in the privacy notice; the UI says what is collected and why at the point of collection |
| 6 | **Data-subject rights** | access/export, rectification, erasure, restriction, objection, and portability still work for the new data. New fields appear in the export/deletion paths — a new table or third-party copy that deletion does not reach is a finding |
| 7 | **Erasure is real** | delete actually erases or irreversibly anonymises, including backups policy, caches, search indexes, logs, and processor copies. A soft-delete flag kept forever is not erasure |
| 8 | **Retention** | a retention period exists and is enforced (TTL, scheduled purge). No new indefinitely-retained store, log, or audit table of personal data |
| 9 | **Security of processing** | encryption in transit and at rest, pseudonymisation where it works, access limited to those who need it. Defer sink-level detail to `security-check-frontend` / `security-check-backend` |
| 10 | **Logs & telemetry** | no PII, credentials, or full request/response bodies in logs, error trackers, APM, or analytics events. Emails and ids masked |
| 11 | **Processors & transfers** | any new third-party recipient (SDK, API, queue, storage, AI/LLM provider, email/SMS gateway) — is there a DPA, is it in the processor register, and is a transfer mechanism in place if data leaves the EU/EEA? Sending personal data to an external model or analytics service is a transfer |
| 12 | **Automated decisions & profiling** | scoring, ranking, segmentation, or automated eligibility affecting people — needs a basis, and human review plus explanation where it has legal or significant effect |
| 13 | **Accuracy** | users can correct their data; imported or synced data has a source of truth |
| 14 | **Privacy by default** | least-sharing defaults, minimal visibility, shortest sensible retention chosen without user action |
| 15 | **Accountability** | processing record (ROPA) updated; a DPIA raised for large-scale, special-category, or systematic-monitoring processing; breach detectability via audit trail |

## 4. Severity

- **Critical** — special-category or large-scale personal data processed with no basis; personal data sent to an undocumented third party or outside the EU/EEA with no mechanism; trackers firing before consent.
- **High** — erasure or export misses the new data; no retention limit; PII in logs or analytics; pre-ticked or bundled consent; marketing without a checked opt-in.
- **Medium** — excess fields collected; privacy notice not updated; unclear purpose; missing masking.
- **Low** — documentation, naming, ROPA hygiene.

## 5. Report

```
## GDPR check — <scope>
Verdict: PASS | PASS WITH FINDINGS | FAIL
Personal data touched: <fields> (special category: yes/no)
New processors / transfers: <list or none>

| Sev | Principle | Location | Issue | Fix |
|-----|-----------|----------|-------|-----|
| High | Erasure | src/user/delete.ts:31 | new `marketing_prefs` table not removed on account deletion | add cascade / purge step |

### Verified clean
- <principles checked with no findings>

### For legal / DPO
- <questions code cannot settle: basis, DPIA need, transfer mechanism>
```

Verdict: any Critical or High → **FAIL**. Only Medium/Low → **PASS WITH FINDINGS**. Nothing → **PASS**.

## 6. Rules

- Name the actual field and file. "Handles PII" is not a finding; "`ssn` written to the application log at auth.ts:44" is.
- Do not invent legal conclusions. Findings state the engineering gap; open legal questions go under **For legal / DPO**.
- If the project's jurisdiction adds requirements (UK GDPR, Swiss FADP, CCPA), apply GDPR as the baseline and note the local delta rather than assuming it.
- Do not change code unless asked. If asked, fix and re-run the check on the result.
- "No personal data in this change" is a complete result.
