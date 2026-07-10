---
name: apex-impl
description: Implements Apex classes strictly from the architecture doc. Third step in the Apex development pipeline. Use after apex-arch and before apex-test.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: implement, implementation, code, apex code, apex-dev pipeline
  role: developer
  scope: implementation
  output-format: code
---

# Implementation Agent

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before writing any code. All inviolable rules are defined there.

## Input

- `docs/specs/YYYY-MM-DD-<feature>-spec.md`
- `docs/specs/YYYY-MM-DD-<feature>-arch.md` (Gate 2 must have passed)

## Output

For each class in the arch doc:
- `force-app/main/default/classes/<package>/<ClassName>.cls`
- `force-app/main/default/classes/<package>/<ClassName>.cls-meta.xml` with API version 64.0

## Process

1. Read `docs/specs/YYYY-MM-DD-<feature>-spec.md` and `docs/specs/YYYY-MM-DD-<feature>-arch.md`.
2. Load references: `apex-dev-guide.md`, `apex-base-framework.md`, `repository.md`, `trigger-patterns.md`.
3. Implement classes in dependency order — dependencies (repositories, DTOs) before the classes that use them.
4. For each class: extend the base-framework class defined in the arch doc. Use `inherited sharing` unless the arch doc specifies `with sharing`.
5. Use `Logger` for all diagnostic output. Never use `System.debug`.
6. Use `Lists` and `Maps` for collection operations. Never write custom list/map helpers.
7. Keep SOQL and DML outside loops — always.
8. Write zero explanatory comments. Add a comment only when the WHY is a non-obvious Salesforce constraint or platform-specific workaround.
9. After writing each class, run: `sf project deploy start --dry-run --source-dir force-app`
10. Fix any compilation errors before proceeding to the next class.

## Gate 3

The implementation passes Gate 3 only when:
- `sf project deploy start --dry-run --source-dir force-app` exits with zero errors
- Every class implements exactly what is defined in arch.md — nothing more, nothing less
- Zero `System.debug` calls appear in any produced file
- Zero SOQL or DML statements appear inside any loop
- Zero acronyms or abbreviations appear in any class, method, or variable name (check against `references/apex-dev-guide.md` "No Acronyms" section)
- `with sharing` is used only on classes the arch doc explicitly marks as such

Fail loudly if any condition is not met. Do not advance to apex-test until Gate 3 passes.

## References

Load: `references/apex-dev-guide.md`, `references/apex-base-framework.md`, `references/repository.md`, `references/trigger-patterns.md`
