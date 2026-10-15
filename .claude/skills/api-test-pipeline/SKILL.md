---
name: api-test-pipeline
description: Runs the whole API test pipeline end to end for one endpoint, given only the API (method + path, e.g. "[GET] /api/subunits") — exploration on QA → test scope → prioritization → P1/P2 scripting in the Postman collection → Newman run/debug → review — one stage after another without stopping between stages. Use when the user gives an API and asks for "the full pipeline", "do the whole process", "explore, scope, script and review", or "run it until the review is done".
---

# API test pipeline (orchestrator)

You are the **pipeline driver**. The user provides **only the API**. You run every stage in order,
hand each stage's output to the next, and finish with the review. Do not pause for confirmation
between stages: the stage skills' "ask whether to proceed" endings are **skipped** in this pipeline.

```
1 api-exploratory-testing → 2 api-test-scope → 3 api-test-prioritization
  → 4 api-test-scripting → 5 api-test-run-debug → 6 api-test-review → 7 hand-off
```

Each stage follows its own skill **exactly**. Load it with the Skill tool when you reach it, then
follow its instructions. This file only adds the glue: inputs, hand-offs, defaults for questions,
gates, and the final report. Conventions come from **api-framework-structure** and
**api-automation-best-practices**. Read both before stage 1.

## 0. Input and pre-flight

**Input:** method + path, e.g. `[GET] /api/subunits`. Derive the rest yourself:

| Item | How |
|---|---|
| Module | From the OrangeHRM source (controller plugin) and existing `docs/<Module>/` / collection folders (e.g. `/api/subunits` → Admin). |
| Feature folder | Reuse an existing `docs/<Module>/<Feature>/` + `Test-data/<Module>/<Feature>/` if the resource already has one (e.g. `Subunits`); otherwise Title-Case kebab of the resource. |
| ID prefix | `<MODULE>-NNN` — next free id from the **live** collection (and the memory note, if any). |
| Scope/Exp ID prefix | Reuse the prefix of sibling docs for the same resource (e.g. `SUB-` → `SUB-GET-Snn`, `SUB-GET-Enn`). |
| File names | Naming rules in **api-exploratory-testing** / **api-test-scope** / **api-test-prioritization** (`…-Test-plan-<verb>-<resource>-scope.md`, same description in the prioritization file). |

Ask the user **only** if the method or path is missing, or the module can't be determined from the
source. Everything else is derived.

Pre-flight (fail fast, report and stop if broken):
1. QA token can be issued (setup in **api-exploratory-testing**). A 403 `licence invalid` → stop, the user must renew the licence.
2. Newman works: nvm node 22.22.2, and `export NODE_OPTIONS="--no-network-family-autoselection --dns-result-order=ipv4first"`.
3. Postman MCP reachable: get the live collection map and find the highest `<MODULE>-NNN`.
4. Compare the live structure with `API-Automation.postman_collection.json` (names/order) and the live
   variable list with the export's. If they differ, note it. Variables that requests reference but that exist
   nowhere are a **finding** to fix in stage 4 (it happened with `subunit*Schema`).

Keep a running **pipeline log** in the scratchpad (`$SP/pipeline-log.md`): stage, outputs, decisions,
open issues. It feeds the final report.

## 1. Exploration — `api-exploratory-testing`

- Output: `docs/<Module>/<Feature>/<MODULE>-exploration-<method>-<slug>.md`. If it exists, re-explore
  and update it in place, but only where it's stale or incomplete.
- Read the OrangeHRM source first (controller, validator, service/DAO, apidoc block) so each probe
  has an expected result.
- For GET/list endpoints create your own fixtures (parent/child records, employees if a filter needs
  them) so read-after-write, filters and ordering are tested on known data, then clean up.
- **Gate:** every checklist category has a row or "N/A — reason"; suspected defects listed; data cleaned up
  (or the leftovers listed).

## 2. Scope — `api-test-scope`

- Input: the stage-1 doc only (check its `Endpoint key`).
- Mark scenarios already automated by other folders with their collection IDs.
- Skip the closing "proceed to prioritization?" question.

## 3. Prioritization — `api-test-prioritization`

- Generate the prioritization doc from the scope doc (same rows, Priority filled, summary + rationale).
- Skip the "confirm P1/P2" question. Write the chosen defaults under **Assumptions** instead (see
  "Open questions" below).

## 4. Scripting — `api-test-scripting` (P1 + P2 only)

Proven workflow for this repo:
1. **Generate, don't hand-type.** Write a Python generator in the scratchpad that produces:
   request specs (export shape: `name`, `event[{listen, script{exec,type}}]`, `request{auth?, method, header, body?, url{raw,host,path}}`, `response: []`),
   the CSVs in `Test-data/<Module>/<Feature>/`, schema strings, and a **run collection** (repo export
   + new folders + merged variables). Run it with `python3 -I`.
2. `node --check` every generated script.
3. Query params are added in the pre-request with `pm.request.url.addQueryParams` (never `?{{query}}` in the URL).
4. Every leaf folder creates its own data and deletes it at the end (employees: terminate). It must
   run alone and twice in a row.
5. Known-defect rows: assert the **documented** behaviour, add a `<prefix>_knownDefect` column, and end the test name with `[known defect Dn]`.
6. Push to Postman only **after** stage 5 passes on the run collection (see stage 5):
   - Create the folders yourself (`createCollectionFolder`, parent folder id without the owner prefix).
   - Write one MCP payload file per request, then delegate the `createCollectionRequest` calls to a
     **subagent**: sequential, verbatim, and on a timeout check the folder before retrying (the MCP often times out
     but still creates the request). Never create duplicates.
   - Variables: `patchCollection` **replaces the whole list**. Merge with the live list, write the full list to
     a file, and let a subagent send it verbatim. Then re-fetch (`model=minimal`) and diff the live list against the file.
     It must be identical.
   - Spot-check one pushed request (`getCollectionRequest populate=true`) against the spec.
7. Re-export: write the run collection to `API-Automation.postman_collection.json`
   (`json.dumps(indent=2, ensure_ascii=False)`, no trailing newline). Check
   `git diff --diff-algorithm=patience --stat` shows insertions only (plus intended variable changes).
8. Add an **Automation status** section (scope ID → collection IDs → folder/CSV row → QA status) to both the
   scope and prioritization docs.

## 5. Run and debug — `api-test-run-debug`

- Run each new leaf folder with its CSV against QA, from the run collection: negatives (no data) first, then
  the positives. Use `-r cli,json` and inspect the JSON report:
  - check the URLs actually sent (query strings, empty values);
  - check conditional tests really ran (count assertion names per request). Note: `pm.sendRequest`
    calls are listed as extra executions of the current request. That isn't a double run.
- Classify every failure. Only known-defect rows may stay red. Fix script/data bugs and re-run.
- Re-run existing folders that share anything you changed (schemas, variables) to prove no regression.
- **Gate:** all assertions pass except documented known defects.

## 6. Review — `api-test-review`

- Review the new folders against the checklist, using the specs/live requests and CSVs.
- Fix every Blocker and Major you find (in the generator → regenerate → re-push only the changed
  requests → syntax-check; re-run the folder if the change affects behaviour and doesn't create more
  undeletable data). Then report them as "fixed".
- Produce the review report in that skill's format with a verdict.

## 7. Hand-off (final message)

Report, concisely:
- Docs created (3), CSVs, folders + collection IDs, next free ID.
- QA results table per folder (assertions passed/failed, known defects).
- Suspected defects (one line each).
- Review findings by severity and the verdict.
- Every default you chose for an open question, so the user can change it.
- Data left on QA (e.g. terminated employees).
- Ask whether to commit (never commit or push without being asked; the branch is the current one, not `main`).
Also update the project memory (next free ID, endpoint facts, new gotchas).

## Open questions — defaults, not stops

The stage skills say "ask the user". In this pipeline, decide with these defaults, record them in the
prioritization doc's **Assumptions** and in the final report, and keep going:

| Question | Default |
|---|---|
| Spec vs actual differs (suspected defect) | Assert the documented/source behaviour as a known-defect test |
| Behaviour only observed (source silent) | Assert observed behaviour; mark "observed only" |
| Undocumented behaviour that looks wrong but isn't clearly specified | Record as P3 / open question; don't script |
| 403 / role tests without non-admin credentials | Mark **blocked**, don't script |
| Error message wording oddities | Assert status + key fields; exact text only when stable |
| Scenario would leave undeletable data | Prefer fewer CSV rows; otherwise skip and note |

## When you MUST stop and ask

- Method/path missing, or module not determinable.
- Anything creating more than ~20 records in one go, bulk deletes, or touching data you didn't create.
- Changing shared QA settings (e.g. organisation flags).
- Environment broken (licence, credentials, MCP down) beyond a retry.
- A test can't be made independent/deterministic.
- Removing or rewriting existing tests or variables, rather than restoring something missing.
