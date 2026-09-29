---
name: api-test-run-debug
description: Runs the API-Automation Postman collection or a single folder with Newman (with CSV data and the QA environment), reads the results, and debugs failures — separating script bugs, test-data errors, environment/auth problems, and real API defects. Use when asked to run tests, execute a folder, check why a test fails, dry-run new scripts, or generate a Newman report.
---

# Run and debug API tests

## Setup

Newman 6 and `newman-reporter-htmlextra` are installed globally under nvm **node 22.22.2**:

```bash
source ~/.nvm/nvm.sh && nvm use 22.22.2 >/dev/null
newman --version
```

Run against the **live** collection when scripts were just edited in Postman: re-export first, or
point Newman at the Postman API URL if the user provides an API key. Otherwise the repo export
is used — say which one you ran.

## Commands

Single folder (the normal case — each leaf folder has its own CSV):

```bash
newman run API-Automation.postman_collection.json \
  -e QA.postman_environment.json \
  --folder "Get-Immigration" \
  -d "Test-data/PIM/Immigration/create-employee-for-get-immigration.csv" \
  -r cli,htmlextra --reporter-htmlextra-export newman-reports/get-immigration.html
```

- `--folder` matches a folder name; parent folders run all children. Folders without a CSV run
  without `-d`.
- Full regression: loop over leaf folders pairing each with its CSV rather than running the whole
  collection with a single `-d` (one CSV can't feed every folder).
- CSVs live in `Test-data/<Module>/<Feature>/`; find a folder's CSV by name with
  `find Test-data -name "<folder-name>.csv"`.
- Add `--verbose` to see request/response bodies; add `--export-environment` to inspect env vars
  after the run.
- Enable script logging with `--env-var debug=true` (scripts read `pm.variables.get("debug")`,
  where the environment value overrides the collection default).

### Dry-run without touching QA

For new scripts, first run against a small Node mock on localhost that returns canned responses
matching the schemas, overriding `--env-var baseURL=http://localhost:<port>`. Put the mock in the
scratchpad, not the repo. This exercises script logic, variable hand-offs and CSV wiring.

Confirm with the user before running folders that create data on a shared QA instance at scale.

## Reading results

Summarise as: folder · iterations · requests · assertions passed/failed, then one line per failure
with request ID, test name, and the assertion message.

## Debugging — classify every failure

| Symptom | Likely class | Check |
|---|---|---|
| `Precondition failed: X not set by PIM-0NN` | Upstream request failed | Look at the earlier request's status/body — fix that first |
| `Test data error: … blank` / JSON parse error in request | Test data | CSV cell empty, wrong quoting, header typo vs `{{var}}` name |
| 401 / `Token request failed` | Environment/auth | `QA.postman_environment.json` creds, `baseURL`, clock; token refresh in collection pre-request |
| `fixture is missing` / schema var undefined | Framework | Collection variable missing — check the variable list was not wiped by a partial patch |
| Schema mismatch on a field | Contract change or schema too strict | Compare actual response vs schema; ask before loosening |
| Value mismatch (expected CSV vs actual) | Real defect **or** API normalises data (trimming, date format, case) | Reproduce with a single manual call (curl setup in **api-exploratory-testing**); check OrangeHRM source for the transformation |
| Record not found in list | Wrong ID saved, pagination, or eventual consistency | Log `recordId` and list length under `debug` |
| Passes alone, fails in full run | Shared state leak | Missing `pm.environment.unset` in the setup request |
| Flaky timing (> 10000 ms) | Environment | Rerun once; report if repeatable |

Debug steps:
1. Rerun only the failing folder with `--verbose`.
2. Read the actual request body and response — don't guess from the test name.
3. Fix the smallest thing: CSV > script > schema. Never weaken an assertion just to make it pass;
   if the API looks wrong, report it as a suspected defect with request, response, and expected value.
4. Rerun the folder, then its parent folder, to confirm no side effects.

## Report back

- Command(s) run and which collection source (live vs export).
- Pass/fail counts, failures with classification and root cause.
- What you changed (and where: Postman, CSV, schema) and what still fails.
- Suspected API defects listed separately, ready to raise as bugs.
