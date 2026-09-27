---
name: api-test-review
description: Reviews Postman API tests in the API-Automation collection (and their CSVs/schemas) against this repo's standards — structure, naming, IDs, independence, assertion quality, data handling, and coverage of P1/P2 scope — and reports findings by severity. Use when asked to review tests, check tests against standards, audit a folder, or before committing/raising a PR for new API tests.
---

# API test review

Review the requested folder(s); default to folders changed since the last commit
(`git diff` on the collection export, or compare the live collection with the export).
Standards come from **api-framework-structure** and **api-test-scripting** — read both first.

Review against the **live** collection when possible; note if the repo export differs.

## Checklist

### Structure & naming
- [ ] Folder in the right place (`Positive Test cases` / `Negative Test cases` / module / feature).
- [ ] Leaf folder is self-contained: creates its own parent data; runs alone with `--folder`.
- [ ] Request names `<ID> <plain sentence>`; IDs sequential, unique, no gaps introduced by reuse; typos fixed.
- [ ] CSV exists in `Test-data/`, named after the folder, columns prefixed per convention.

### Independence & state
- [ ] Setup request unsets hand-off env vars before creating data.
- [ ] Dependent requests guard preconditions with `throw new Error("Precondition failed: … not set by <ID>")`.
- [ ] Unique IDs generated (`<PREFIX>-<timestamp>`), no fixed employee IDs that collide on rerun.
- [ ] No reliance on data from another folder or a previous run.

### Assertions
- [ ] Exact status code asserted on every request.
- [ ] Schema validation on action/GET requests, and it fails loudly if the schema var is missing.
- [ ] Field values compared to CSV/previous response, not hard-coded.
- [ ] List responses: specific record found by ID, field checks guarded by `if (selectedRecord)`.
- [ ] Negative tests assert specific status + error content, and ideally that nothing was created.
- [ ] Every `pm.expect` has a message; test names read as plain-sentence expectations.
- [ ] No assertion that always passes (e.g. `to.exist` on a value already defaulted to `{}`, empty `pm.test`).

### Data & scripts
- [ ] Unquoted body variables guarded against blank CSV cells.
- [ ] `norm()` used where null/""/undefined should be equal.
- [ ] Env writes outside `pm.test`, with `// side effect:` comment.
- [ ] No duplicated collection-level checks; no stray `console.log` outside `debug`.
- [ ] No secrets or tokens hard-coded.

### Coverage
- [ ] Every P1/P2 row in `Test-plans/<module>-scope.md` maps to a collection ID, or is marked as not yet scripted.
- [ ] Read-after-write exists for each create endpoint.

## Report format

Group by severity; each finding cites the request ID and quotes the offending line.

```
## Review: <folder>  (<n> requests, <m> CSVs)

### Blocker   — test gives wrong result or breaks other tests
- PIM-031 · assertion compares to hard-coded "Son" instead of dep_relationship → passes for wrong data. Fix: …

### Major     — violates standard, reduces reliability
### Minor     — naming, comments, style
### Coverage gaps — P1/P2 scope rows with no test

Verdict: Ready / Ready after fixes / Not ready
```

Do not edit the collection during a review unless the user asks for fixes; if they do, apply the
fixes, then re-run via **api-test-run-debug**.
