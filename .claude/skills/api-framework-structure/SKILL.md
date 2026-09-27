---
name: api-framework-structure
description: Defines the folder structure, naming, and data-flow conventions of this Postman + Newman API automation framework (OrangeHRM API). Use when adding a new module, folder, CSV, schema, or environment variable, when asked "where does X go", or before any other api-* skill touches the repo or the collection.
---

# API framework structure

This repo is a **Postman collection driven by CSV test data and run with Newman**. There is no
code framework (no package.json, no JS test runner) — everything executable lives inside the
collection's pre-request and test scripts.

## Source of truth

| Artifact | Source of truth | Notes |
|---|---|---|
| Collection | Postman **API-Automation** in Team Workspace (uid `16056352-c7925e32-21ec-48e0-bb7c-94ab75adb207`) | `API-Automation.postman_collection.json` is an **export** and can lag. Re-export after Postman edits before committing. |
| Environment | `QA.postman_environment.json` | Only connection/auth vars. Never commit real secrets beyond what is already there. |
| Test data | `Test-data/*.csv` | One CSV per leaf folder. |
| JSON schemas | Collection variables (`<module>PostSchema`, `<module>GetSchema`) | Stored as stringified JSON. |

When editing via the Postman MCP, `patchCollection`'s `variable` field **replaces the whole list** —
fetch the live collection, merge, send every variable, then verify.

## Repository layout

```
Integration-api/
├── API-Automation.postman_collection.json   # exported collection (tests live here)
├── QA.postman_environment.json              # baseURL, clientID, clientSecret, grantType, userName, password, accessToken
├── Test-data/                                # one CSV per leaf folder, kebab-case, named after the folder
│   └── <folder-name>.csv
├── Test-plans/                               # outputs of api-test-scope / api-test-prioritization
│   └── <module>-scope.md
├── newman-reports/                           # git-ignored run output (htmlextra)
├── .claude/skills/                           # these skills
└── .gitignore                                # ignores newman-reports/, report.html, .~lock.*#, .mcp.json
```

Rules:
- CSVs go in `Test-data/` only — never in the repo root. Flag stray root CSVs or `.~lock.*#` files.
- Reports go in `newman-reports/` (ignored). Do not commit `report.html`.
- `Test-plans/` holds markdown scope/priority docs; create it on first use.

## Collection layout

```
API-Automation
├── Authentication
│   └── AUTH-001 get access token
├── Positive Test cases
│   └── <Module>                       e.g. PIM
│       └── <Feature>                  e.g. Create-Employee, Get-Employee, Update-employee
│           └── <Sub-feature>          e.g. Immigration
│               ├── Add-<Sub-feature>  self-contained leaf folder = one scenario
│               └── Get-<Sub-feature>
└── Negative Test cases                (create when the first negative test is written)
    └── <Module>/<Feature>/<scenario>  mirrors the positive tree
```

- **Leaf folder = one scenario = one CSV.** Every leaf folder creates its own data (e.g. its own
  employee) so it can run alone with `--folder`.
- Folder names: kebab-case, Title-Case words allowed (`Add-Immigration`, `get-all-employees`).
- Request names: `<ID> <plain sentence>` — `PIM-026 add immigration for created employee`.
- IDs: `<MODULE>-NNN`, sequential **across the whole collection**, never reused. Find the highest
  existing ID before adding (`AUTH-001`, `PIM-001…`). Setup requests get IDs too.

## Collection-level scripts (do not duplicate in requests)

- **Pre-request:** refreshes the OAuth token via `/oauth/issueToken` 60 s before `tokenExpiresAt`.
- **Tests:** response time < 10000 ms, `Content-Type` is JSON, status < 500.

## Variable scopes

| Scope | Used for | Example |
|---|---|---|
| Environment | Connection/auth; IDs passed between requests in a folder | `baseURL`, `accessToken`, `emp_number`, `immigration_record_id` |
| Collection | Schemas, shared fixtures, `debug` flag | `immigrationGetSchema` |
| Local (`pm.variables`) | Per-request generated values | `employeeId` |
| Iteration data | Expected/input values from CSV | `imm_number` |

- Hand-off IDs are snake_case env vars; if a folder has two actors use role names
  (`supervisor_emp_number`, `subordinate_emp_number`).
- Unique employee IDs: `` `<PREFIX>-${Date.now().toString().slice(-6)}` `` with a module prefix
  (IMM, IMMGET, DEP, …).

## CSV conventions

- File name = leaf folder name + `.csv`.
- Header row, all values double-quoted except numbers that go into the body unquoted.
- Shared employee columns: `firstName,middleName,lastName,locationId,joinedDate,chkLogin,autoGenerateEmployeeId`.
- Module columns prefixed: `imm_*`, `dep_*`, `dd_*`, plus `<prefix>_successMessage` where the API returns one.
- Multi-actor folders prefix by role: `supervisorFirstName`, `subordinateFirstName`.
- One row = one iteration. Add rows for data variations instead of duplicating requests.

## When asked to set up a new module

1. Confirm module name and ID prefix with the user.
2. Create `Test-plans/<module>-scope.md` (via **api-test-scope**).
3. Add folders to the live collection following the tree above.
4. Add schema collection variables (full-list patch).
5. Add CSVs to `Test-data/`.
6. Re-export the collection into the repo.
