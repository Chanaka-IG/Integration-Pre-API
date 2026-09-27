---
name: api-test-scripting
description: Writes Postman request pre-request/test scripts and CSV test data for the P1 and P2 scenarios of an API test scope, following this repo's conventions (self-contained folders, sequential IDs, env-var hand-offs, iteration-data assertions, schema checks). Use when asked to write/automate/script API tests, add requests to the API-Automation collection, or implement scenarios from Test-plans/<module>-scope.md.
---

# API test scripting (P1 + P2 only)

Inputs: `Test-plans/<module>-scope.md` with confirmed priorities (**api-test-prioritization**).
Script **only P1 and P2** rows. If asked to script P3/P4, say they are out of scope for this skill
and confirm with the user first.

Follow **api-framework-structure** for folders, IDs, CSVs and variable scopes. The reference
implementation is `Positive Test cases/PIM/Update-employee/Immigration` — copy its shape.

## Workflow

1. Fetch the live collection (Postman MCP `getCollection`) — the repo export may be stale.
2. Find the highest existing `<MODULE>-NNN` and continue from there.
3. Per scenario, create a **self-contained leaf folder**: setup request(s) → action request →
   (verification GET if read-after-write).
4. Create the CSV in `Test-data/<folder-name>.csv`.
5. Add/merge schema collection variables (send the full variable list).
6. Dry-run (see **api-test-run-debug**) before telling the user it is done.
7. Update the scope doc: fill in the collection IDs for scripted rows.
8. Re-export the collection to `API-Automation.postman_collection.json`.

## Request body

Reference data with `{{var}}`. Strings quoted, numbers/booleans unquoted:

```json
{
    "firstName": "{{firstName}}",
    "employeeId": "{{employeeId}}",
    "chkLogin": {{chkLogin}},
    "type": {{imm_type}}
}
```

Any unquoted variable needs a pre-request guard against a blank CSV cell (invalid JSON otherwise).

## Setup request (creates the parent)

```js
// Pre-request
// clear values left over from earlier folders or earlier runs
pm.environment.unset("emp_number");
pm.environment.unset("<child>_record_id");

pm.variables.set("employeeId", `<PREFIX>-${Date.now().toString().slice(-6)}`);
```

```js
// Tests
pm.test("Employee is created", function () {
    pm.response.to.have.status(201);
});

const responseData = (pm.response.json() || {}).data || {};

pm.test("Response contains empNumber", function () {
    pm.expect(responseData.empNumber, "empNumber").to.exist.and.not.be.empty;
});

// side effect: consumed by PIM-0XX
if (responseData.empNumber) {
    pm.environment.set("emp_number", responseData.empNumber);
}
```

## Dependent request

```js
// Pre-request
const empNumber = pm.environment.get("emp_number");
if (!empNumber) {
    throw new Error("Precondition failed: emp_number not set by PIM-0XX");
}
```

## Assertions

```js
const response = pm.response.json() || {};
const responseData = response.data || {};
const norm = v => (v === null || v === undefined) ? "" : String(v);
const expected = key => norm(pm.iterationData.get(key));

pm.test("<Record> is added", function () {
    pm.response.to.have.status(201);
});

pm.test("Response matches the <record> schema", function () {
    const raw = pm.collectionVariables.get("<module>PostSchema");
    pm.expect(raw, "<module>PostSchema fixture is missing").to.be.a("string").and.not.empty;
    pm.response.to.have.jsonSchema(JSON.parse(raw));
});

pm.test("<Record> details match what was sent", function () {
    pm.expect(norm(responseData.number), "number").to.eql(expected("imm_number"));
});
```

For list GETs: find the exact record by the id saved in the POST step, assert it exists, and only
run field checks inside `if (selectedRecord) { ... }`.

## Negative (P2) tests

- Place under `Negative Test cases/<Module>/<Feature>/<scenario>` (mirror the positive tree).
- Still self-contained: create a valid parent first, then send the bad request.
- Put the invalid value and expected result in the CSV (`expectedStatus`, `expectedError`) so one
  folder covers several invalid values as iterations.
- Assert: exact status code (400/401/404/422 — not just "not 2xx"), error body shape/message, and
  where possible that nothing was created (follow-up GET).
- The collection-level "Response is JSON" test runs on every response — if an error response is
  not JSON, note it as a finding instead of weakening the collection test.

## Rules

- Test names are plain sentences describing the expectation ("Record is linked to the created employee").
- Every assertion has a message argument naming the field.
- No hard-coded expected values in scripts — they come from the CSV or a previous response.
- Env writes (`pm.environment.set`) sit **outside** `pm.test`, commented `// side effect: consumed by …`.
- Don't repeat collection-level checks (response time, JSON, < 500) in requests.
- Use `console.log` only behind `if (pm.variables.get("debug"))` (resolves env or collection scope).
- No `setTimeout`/sleeps; no `postman.setNextRequest` unless the user asks.
