---
name: api-automation-best-practices
description: General API test automation best practices applied to this Postman + Newman framework (OrangeHRM API) — test independence, determinism, assertion depth, contract/schema testing, negative and security coverage, test data lifecycle, secrets handling, flakiness prevention, maintainability, and CI/reporting. Use when asked "what's the best practice for…", when designing a new test approach, when deciding how to structure assertions or data, when hardening flaky tests, or as the principles behind api-test-scripting and api-test-review.
---

# API automation best practices

These are the **principles**. Repo-specific mechanics live in the other skills — don't restate them,
point to them:

| Need | Skill |
|---|---|
| Where files/folders/variables go | **api-framework-structure** |
| What to test | **api-test-scope** → **api-test-prioritization** |
| How to write the scripts | **api-test-scripting** |
| Checking tests against standards | **api-test-review** |
| Running and diagnosing | **api-test-run-debug** |

When a principle here conflicts with an established repo convention, follow the convention and
point out the conflict to the user — don't silently diverge.

## 1. Every test is independent and repeatable

- A leaf folder must pass when run **alone**, **in any order**, and **twice in a row**.
- Create the data you need; never rely on records that "already exist" in QA or on a previous folder.
- Generate unique values (timestamp suffixes) for anything with a uniqueness constraint.
- Unset hand-off variables at the start of a folder so stale values from an earlier run can't give a false pass.
- A dependent request that is missing its precondition must **fail loudly** (throw), not skip quietly.

## 2. Deterministic — no flakiness by design

- No sleeps or fixed waits. If the API is eventually consistent, raise it as a finding; don't paper over it.
- Don't assert on list position, default sort order, or total counts in shared environments —
  find records by the id you created.
- Don't assert exact timestamps; assert format, or a range around "now".
- A flaky test is a bug. Fix the root cause or quarantine it with a note; never add retries to hide it.

## 3. Assert what matters, precisely

For each response, check in this order:

1. **Status code — exact** (`201`, not "2xx"; `422`, not "not 200").
2. **Contract** — JSON schema for the whole body (catches missing/renamed/retyped fields).
3. **Data** — the fields you sent come back unchanged; server-generated fields (ids) exist and have the right type.
4. **Side effects** — read-after-write GET confirms the change persisted; for negatives, confirm nothing was created.

Also:
- One behaviour per `pm.test`, named as a plain sentence of the expectation.
- Every `pm.expect` carries a message naming the field, so a failure says *what* broke.
- Expected values come from test data or earlier responses — never hard-coded in the script.
- Avoid weak assertions: `to.exist` alone, `to.be.ok`, `length > 0`, "status is not 500".
- Guard field checks so a missing record produces one clear failure, not a cascade of `TypeError`s.
- Try to cover the scope of the test plan with a single request, rather than splitting it into multiple requests that each assert one thing.

## 4. Schemas are contracts

- Keep schemas strict enough to catch drift: `required` for mandatory fields, correct `type`s,
  `additionalProperties` left permissive unless the API guarantees a closed shape.
- Nullable fields must be typed `["string", "null"]` — not dropped from the schema.
- Separate schemas for POST vs GET responses when their shapes differ.
- When a schema check fails, decide first whether it's an API change or a stale schema — don't just loosen it.

## 5. Cover more than the happy path

Weight effort by priority (P1/P2 first), but a mature suite covers:

| Category | Examples |
|---|---|
| Validation | missing mandatory field, wrong type, over max length, invalid enum/date format |
| Boundary | min/max length, zero, empty string vs null, leap dates, today/future dates |
| Auth | no token, expired/invalid token, user without permission (401 vs 403) |
| Not found | non-existent or deleted parent/record ids (404) |
| Integrity | duplicate unique values, child record linked to the right parent, delete cascades |
| Security basics | special characters/HTML/SQL-ish input stored and returned verbatim, not executed or 500 |

Negative tests assert the **specific** status and error message/shape, and drive multiple invalid
inputs as CSV iterations instead of duplicated requests.

## 6. Test data lifecycle

- Data-driven: inputs **and** expected results live in the CSV; one row = one iteration.
- Keep data realistic but obviously synthetic (recognisable prefixes) so it's easy to find and purge.
- Don't put personal data or production copies in CSVs.
- Clean up when the API supports it and the record would otherwise break later runs;
  otherwise rely on unique values and document the leftover data.
- Blank CSV cells feeding unquoted JSON values need a guard — invalid JSON is a script bug, not an API defect.
- Use the same CSV for positive and negative tests of the same feature, with a `valid` column to filter rows.
- If a test needed a value from the pre-request, store it in an env var and assert it in the test, rather than hard-coding it in the CSV.
- Try to get test data from the CSV rather than generating it in the pre-request, so the test is deterministic and repeatable. If there is nothing to do, you can add test data in the pre-request, but it should be a last resort.

## 7. Secrets and environments

- Tokens are fetched at runtime (collection pre-request), never pasted into requests or committed.
- Credentials live only in the environment file; don't add new secrets to the repo — use Newman
  `--env-var` or CI secrets for anything new.
- Nothing environment-specific (host, ids, user names) is hard-coded in requests — use variables,
  so the same suite runs against another environment unchanged.
- Never point a write-heavy run at production.

## 8. Maintainability

- Put shared behaviour at the highest sensible level (collection/folder scripts), and don't repeat it per request.
- Keep scripts short and uniform — copy the reference folder's shape so every folder reads the same.
- Small helpers (`norm`, `expected`) over clever abstractions; readability beats DRY in test code.
- Stable, sequential IDs and descriptive names make reports and traceability to the scope doc trivial.
- Treat the live Postman collection as source of truth; re-export before committing so the repo diff reflects reality.
- Logs only behind a `debug` flag; no noisy `console.log` in committed scripts.

## 9. Execution, CI and reporting

- Dry-run new or changed folders (against a local mock where possible) before touching QA.
- Run P1 on every change/PR, full suite on a schedule.
- Fail the build on any assertion failure; don't use `--suppress-exit-code` in CI.
- Publish an htmlextra report as a build artifact; keep reports out of git.
- When a test fails, classify it before acting: script bug, test data, environment/auth, or real API defect
  (see **api-test-run-debug**). Only the last one becomes a bug report.

## 10. Know when not to automate

- One-off exploratory checks, rarely-changing admin config, or scenarios needing manual/visual
  judgement are usually P3/P4 — record them in the scope doc rather than scripting them.
- If a test can't be made independent and deterministic, flag it to the user instead of committing a flaky one.

## Applying this skill

When asked for advice: answer with the relevant principle(s) above, then show how it maps to this
repo (the folder, variable, or script pattern), citing the owning skill.
When reviewing or designing: list each deviation as *principle → what's wrong → concrete fix*.
