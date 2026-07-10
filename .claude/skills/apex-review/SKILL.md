---
name: apex-review
description: Reviews all Apex code for Clean Code, SOLID, and base-framework compliance. Auto-fixes up to 3 times before blocking. Final step in the Apex development pipeline before code is ready for deploy.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: review, code review, quality gate, apex-dev pipeline, lint
  role: reviewer
  scope: quality
  output-format: report
---

# Review Agent

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before reviewing any code. This agent enforces every rule defined there.

## Input

- All production Apex classes under `force-app/main/default/classes/<package>/`
- All test Apex classes under `force-app/main/default/classes/<package>/test/`
- `docs/specs/YYYY-MM-DD-<feature>-spec.md` and `docs/specs/YYYY-MM-DD-<feature>-arch.md`

## Output

**On success (Gate 5 passed):** No file written. Report "Gate 5 passed — all dimensions clean."

**On block (4th attempt):** `.superpowers/review/YYYY-MM-DD-<feature>-review.md` containing:
- One entry per unresolved violation: file path, line number, dimension (Clean Code / SOLID / base-framework), violation description, specific fix suggestion

## Review Checklist

Run this checklist against every production and test class:

### Clean Code
- [ ] No acronyms or abbreviations in any class, method, variable, or parameter name (see `references/apex-dev-guide.md` "No Acronyms" section for full list)
- [ ] Methods are small and operate at a single level of abstraction
- [ ] Zero explanatory comments — only "why" comments for non-obvious constraints
- [ ] No magic numbers or hardcoded strings
- [ ] No dead code (unreachable branches, unused variables, unused imports)

### SOLID
- [ ] **SRP:** Each class has one reason to change — verifiable by its one-sentence responsibility statement from the arch doc
- [ ] **OCP:** New behavior is added via extension (subclassing or interface implementation), not modification of existing classes
- [ ] **DIP:** Classes depend on `ILogger`, `AbstractRepository`, `Filter`, `AbstractOutboundCommand`, or `Command` interfaces — not on concrete implementations

### base-framework
- [ ] `Lists` and `Maps` utilities used for collection operations — no custom helpers
- [ ] `Filter` used for all filter 
- [ ] `Logger` used for all diagnostic output — zero `System.debug` calls
- [ ] `TypeFactory` used wherever dynamic class instantiation occurs
- [ ] No framework infrastructure reimplemented

## Auto-Fix Protocol

```
Attempt 1:
  Run full checklist against all classes.
  Auto-fix these violations automatically:
    - Rename identifiers containing acronyms (e.g., accId → accountId, sfId → salesforceId)
    - Replace System.debug(x) with Logger.log(x)
    - Replace custom List/Map iteration helpers with Lists.* / Maps.* equivalents
  Run: sf apex test run -c
  If Gate 5 passes → DONE. Report "Gate 5 passed on attempt 1."

Attempt 2 (violations remain after attempt 1):
  Auto-fix these violations:
    - Extract private methods to resolve SRP violations (oversized methods doing multiple things)
    - Replace hardcoded strings/numbers with named constants
  Run: sf apex test run -c
  If Gate 5 passes → DONE. Report "Gate 5 passed on attempt 2."

Attempt 3 (violations remain after attempt 2):
  Auto-fix these violations:
    - Introduce constructor injection to resolve DIP violations (concrete dependencies → abstract parameters)
  Run: sf apex test run -c
  If Gate 5 passes → DONE. Report "Gate 5 passed on attempt 3."

Attempt 4 → BLOCK:
  Write .superpowers/review/YYYY-MM-DD-<feature>-review.md
  List every unresolved violation: file, line, dimension, description, fix suggestion
  Report: "Gate 5 blocked after 3 auto-fix attempts. See review report."
  Await human decision. Take no further action.
```

## Auto-Fix Constraints

- Never change a public method signature (name, parameters, return type)
- Never change a class name
- Never change a Custom Metadata field reference
- Every auto-fix attempt must be followed by `sf apex test run -c` — never skip the test run
- If a test fails after an auto-fix, revert the fix that caused the failure before proceeding to the next attempt

## References

Load: `references/apex-dev-guide.md`, `references/apex-base-framework.md`, `references/repository.md`, `references/trigger-patterns.md`, `references/test-patterns.md`
