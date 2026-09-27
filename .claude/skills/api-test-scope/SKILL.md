---
name: api-test-scope
description: Builds the API test scope for a module or endpoint (e.g. PIM Immigration, Leave, Attendance) — enumerates endpoints and test scenarios across positive, negative, validation, auth, boundary, and data-integrity categories, and writes Test-plans/<module>-scope.md. Use when asked to "create test scope", "what should we test for <endpoint>", "list test cases for <API>", or before writing new tests.
---

# API test scope

Output: `Test-plans/<module>-scope.md` (see **api-framework-structure**). Scope only — no scripts
here. Priorities are assigned afterwards by **api-test-prioritization**.

## 1. Gather the facts first

Do not invent endpoints or fields. Collect, in order of preference:
1. The user's spec / Jira ticket / API doc (ask for it if not given).
2. Existing requests in the collection for the same resource (method, URL, body, responses).
3. OrangeHRM source via the OrangeHRM Code MCP (`search_code` for the API controller/validator)
   to learn required fields, validation rules, max lengths, enums, and error messages.
4. A real call against QA only if the user allows it.

Record for each endpoint: method, path, auth, path/query params, request fields
(type, required, max length, enum/format), success status, response shape, known error responses.

## 2. Enumerate scenarios per endpoint

Walk every category; write "N/A — reason" rather than silently skipping one.

| Category | What to cover |
|---|---|
| **Happy path** | All fields; mandatory fields only; each supported enum value; read-after-write (GET returns what POST saved) |
| **Field validation** | Each required field missing / empty / null; wrong type; invalid format (dates, country codes, emails) |
| **Boundaries** | Max length and max+1; min values; dates: past/today/future, leap day, start > end |
| **Special data** | Supported special characters, unicode, leading/trailing spaces, SQL/HTML-like strings (expect safe storage/rejection, not execution) |
| **Resource state** | Parent not found (bad `emp_number`), deleted/terminated parent, duplicate create, update/delete of non-existent child |
| **Auth / access** | No token, expired/invalid token, insufficient scope/role |
| **Contract** | Response schema, status codes, headers, error body shape |
| **Lists / queries** | Empty list, pagination/limit/offset, filters, sort, record-specific lookup by ID |
| **Side effects** | Record linked to correct parent; other records untouched; counts change as expected |

Out of scope by default (list them explicitly): performance/load, UI, security penetration beyond
basic auth checks.

## 3. Write the scope doc

```markdown
# <Module> API test scope
Source: <spec/ticket/code refs> · Date: <YYYY-MM-DD>

## Endpoints
| Method | Path | Purpose | Success |
|---|---|---|---|

## Scenarios
| Scope ID | Endpoint | Category | Scenario | Expected result | Test data notes | Priority |
|---|---|---|---|---|---|---|
| IMM-S01 | POST /api/employees/{emp}/immigration | Happy path | Add record with all fields | 201, record returned with recordId | all imm_* columns | _tbd_ |

## Out of scope
## Open questions
```

- Scope IDs (`<PREFIX>-Snn`) are planning IDs; collection IDs (`PIM-NNN`) are assigned only when
  scripted.
- One row per distinct behaviour. Data variations of the same behaviour become CSV rows, not
  extra scenarios — note them in "Test data notes".
- Mark scenarios already automated with the existing collection ID.
- Put anything you could not confirm in **Open questions** and ask the user, rather than guessing
  expected results.

Finish by summarising counts per category and asking whether to proceed to prioritization.
