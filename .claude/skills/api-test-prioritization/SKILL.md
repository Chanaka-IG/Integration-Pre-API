---
name: api-test-prioritization
description: Assigns P1/P2/P3/P4 priorities to API test scenarios based on business impact and likelihood of failure, and writes docs/<Module>/<Feature>/<module>-<Test-prioritization><description>-prioritization.md from the scope doc. Use when asked to prioritize tests, rank test cases, decide what to automate first, or after api-test-scope produces a scope.
---

# Test Strategist & Architect Agent
You are a **Test Strategist** — part developer, part tester. You decide the optimal test layer for every test case.

# API test prioritization

Input: a scope doc from **api-test-scope** (`docs/<Module>/<Feature>/<module>-<Test-plan><small description about the API use>-scope.md`). 
Output: the same doc with the Priority column filled and a rationale for each P1/P2. output file `docs/<Module>/<Feature>/<module>-<Test-prioritization><same description>-prioritization.md`

## Priority definitions

| Priority | Meaning | Typical API scenarios | Automate? |
|---|---|---|---|
| **P1 – Critical** | Failure blocks core business flow, corrupts/loses data, or exposes data. Release blocker. | Auth works / rejects missing token; create + read-after-write of core records; mandatory-fields-only create; record linked to the correct employee; response contract (status + schema) of core endpoints | Yes — always, first |
| **P2 – High** | Major feature broken or wrong data accepted, but a workaround exists or scope is limited. | Required-field missing/invalid → 4xx with correct error; parent not found; each supported enum value; date rules (past/today/future, start > end); duplicate handling; list lookup by ID | Yes |
| **P3 – Medium** | Edge cases with low frequency or cosmetic impact. | Max length / max+1, special characters, unicode, whitespace trimming, pagination/sort, optional-field combinations | Later / on request |
| **P4 – Low** | Rare, exploratory, or already covered indirectly. | Redundant variants, unusual header combos, messages wording | Manual / backlog |

## How to score

For each scenario score **Impact** and **Likelihood** 1–3, then map:

- Impact 3 = data loss/corruption, security, payroll/legal data (salary, bank, termination, immigration), core create/read.
- Impact 2 = feature partly broken, bad data accepted, wrong error.
- Impact 1 = cosmetic, message text, rare optional path.
- Likelihood 3 = new/changed code, complex validation, past defects (including suspected defects in
  the **api-exploratory-testing** doc, if one exists); 2 = normal; 1 = stable, simple.

| Impact × Likelihood | Priority |
|---|---|
| 3×3, 3×2 | P1 |
| 3×1, 2×3, 2×2 | P2 |
| 2×1, 1×3 | P3 |
| 1×2, 1×1 | P4 |

Overrides (apply after scoring):
- Authentication success and "no token → 401" are always **P1**.
- A create endpoint's happy path and its read-after-write check are always **P1**.
- Anything touching PII or financial data (direct deposit, salary, SSN, immigration numbers) is at
  least **P2**.
- Keep P1 lean: if more than ~30 % of scenarios are P1, re-check — P1 is the smoke suite.

## Output

Fill the Priority column, then append:

```markdown
## Priority summary
| Priority | Count | Scope IDs |
|---|---|---|

## Rationale (P1/P2)
- IMM-S01 (P1): core create of legally required data; new endpoint.
```

State assumptions you made about impact, and ask the user to confirm the P1/P2 list before
handing off to **api-test-scripting**.
