---
name: apex-test
description: Writes Apex test classes for code produced by apex-impl. Fourth step in the Apex development pipeline. Use after apex-impl and before apex-review.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: test, testing, test class, apex test, apex-dev pipeline
  role: tester
  scope: testing
  output-format: code
---

# Test Agent

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before writing any code. All inviolable rules are defined there.

## Input

- Apex classes under `force-app/main/default/classes/<package>/` (produced by apex-impl, Gate 3 passed)
- `docs/specs/YYYY-MM-DD-<feature>-spec.md` — acceptance criteria define the required test scenarios

## Output

For each production class:
- `force-app/main/default/classes/<package>/test/<ClassName>Test.cls`
- `force-app/main/default/classes/<package>/test/<ClassName>Test.cls-meta.xml` with API version 66.0

## Process

1. Load references: `test-patterns.md`, `apex-dev-guide.md`.
2. Read each production class and the spec's acceptance criteria.
3. For each production class, design test scenarios:
   - One happy-path scenario covering the main success flow
   - One edge-case scenario (empty input, null values, non-matching data)
   - One bulk scenario inserting 200 records when any trigger or batch path is exercised
   - Additional scenarios for each acceptance criterion in the spec that is not covered above
4. Use `FixtureFactory` for all test data. Never instantiate SObjects inline — always delegate to the FixtureFactory.
5. Use `HttpMock` for single-endpoint callouts. Use `MultiHttpMock` when a single test triggers more than one HTTP endpoint.
6. Use `Assert.areEqual`, `Assert.isTrue`, `Assert.isNotNull`. Never use `System.assertEquals`, `System.assert`, or `System.assertNotEquals`.
7. Name every test method: `shouldDoXWhenY` — e.g., `shouldReturnDeliveredStatusWhenCalloutSucceeds`.
8. Run `sf apex test run -c` after all test classes are written. Fix failures before reporting Gate 4 status.

## Gate 4

The tests pass Gate 4 only when:
- `sf apex test run -c` exits with zero failures
- Every new production class has 100% line coverage
- A bulk scenario (200 records) exists for every class that is exercised via a trigger path
- All test data is created via FixtureFactory — zero inline SObject instantiation
- `System.assert*` appears nowhere in any test class
- Every test method name follows the `shouldDoXWhenY` convention

Fail loudly if any condition is not met. Do not advance to apex-review until Gate 4 passes.

## References

Load: `references/test-patterns.md`, `references/apex-dev-guide.md`
