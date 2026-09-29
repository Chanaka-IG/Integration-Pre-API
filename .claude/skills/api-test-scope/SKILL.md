---
name: api-test-scope
description: Builds the API test scope for a module or endpoint (e.g. PIM Immigration, Leave, Attendance) — enumerates endpoints and test scenarios across positive, negative, validation, auth, boundary, and data-integrity categories, using the OrangeHRM source/docs plus the api-exploratory-testing findings, and writes docs/<Module>/<Feature>/<module>-<Test-plan><description>-scope.md. Runs after api-exploratory-testing. Use when asked to "create test scope", "what should we test for <endpoint>", "list test cases for <API>", or before writing new tests.
---

# Functional Tester Agent
You are a **Senior API automation Designer** — you think like a real user AND a malicious user.

## Task
Create test scenarios for the given API endpoint(s) and write them in a markdown table in `docs/<Module>/<Feature>/<module>-<Test-plan><small description about the API use>-scope.md` (see **api-framework-structure**). Include all positive, negative, validation, auth, boundary, and data-integrity scenarios you can think of. Base request fields, responses, status codes and error messages on the **api-exploratory-testing** findings (actual behaviour), checked against the OrangeHRM code and https://api.orangehrm.com/ (expected behaviour).

## Inputs
1. **From the user: only the API** — endpoint URL/path, method and module name. If any of these
   is missing, ask. Do **not** ask the user for the request body, response, status codes or error
   messages.
2. **Everything else comes from the exploration doc** from **api-exploratory-testing**: request
   fields, response shape/schema, status codes, validation rules, error messages, suspected defects.
   - Work out the exact file name from the given API using the naming rule in
     **api-exploratory-testing** (`docs/<Module>/<Feature>/<MODULE>-exploration-<method>-<endpoint-slug>.md`),
     e.g. `POST /api/v2/pim/employees/{empNumber}/immigrations` →
     `docs/PIM/Immigration/PIM-exploration-post-employees-immigrations.md`. One doc per endpoint + method.
   - Look in `docs/<Module>/<Feature>/`; if you're not sure of the feature folder, `find docs -name "<file name>"`.
     Write the scope doc into the same folder.
   - Read **only** the doc(s) matching the given API, and check its `Endpoint key` line matches.
     Never use exploration docs for other endpoints or methods, even in the same module.
   - Missing → **run api-exploratory-testing first** with the same API, then continue with the
     scope — don't build the scope from assumptions.
3. The OrangeHRM code and https://api.orangehrm.com/ are the check on what the behaviour *should*
   be (see step 1 below).

# API test scope

Output: `docs/<Module>/<Feature>/<module>-<Test-plan><small description about the API use>-scope.md` (see **api-framework-structure**). Scope only — no scripts here. Priorities are assigned afterwards by **api-test-prioritization**.

## 1. Gather the facts first

Do not invent endpoints or fields. Use two sources together:
1. **Expected behaviour** — OrangeHRM source via the OrangeHRM Code MCP (`search_code` for the API
   controller/validator) and https://api.orangehrm.com/: required fields, validation rules, max
   lengths, enums, and error messages.
2. **Actual behaviour** — the exploration doc: observed status codes, exact error messages,
   response shape/schema, normalisation, and its "Notes for scope".

How to combine them:
- Source and exploration agree → use it as the expected result (quote exact codes/messages).
- They differ → expected result comes from the source/docs; add the row to **Suspected defects**
  with the Exp ID, and ask the user which is correct if the source is ambiguous.
- Only exploration covers it (source silent) → use the observed behaviour, mark it
  "observed only" in Test data notes, and list it in **Open questions** for confirmation.
- Neither covers it → Open questions; re-run **api-exploratory-testing** on that point if needed.

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
Source: <spec/ticket/code refs> · Exploration: <exploration doc file name(s)> · Date: <YYYY-MM-DD>

## Endpoints
| Method | Path | Purpose | Success |
|---|---|---|---|

## Scenarios
| Scope ID | Endpoint | Category | Scenario | Expected result | Test data notes | Priority |
|---|---|---|---|---|---|---|
| IMM-S01 | POST /api/employees/{emp}/immigration | Happy path | Add record with all fields | 201, record returned with recordId | all imm_* columns | _tbd_ |

```

## Suspected defects
| Scope ID | Exp ID | Expected (source/docs) | Actual (exploration) |
|---|---|---|---|

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
