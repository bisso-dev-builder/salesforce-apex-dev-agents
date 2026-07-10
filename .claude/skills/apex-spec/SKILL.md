---
name: apex-spec
description: Transforms a user story into a structured spec doc. First step in the Apex development pipeline. Use before apex-arch, apex-impl, apex-test, or apex-dev.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: spec, specification, user story, requirements, apex-dev pipeline
  role: analyst
  scope: planning
  output-format: markdown
---

# Spec Agent

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before taking any action. All inviolable rules are defined there.

## Input

User story in natural language with acceptance criteria. Provide inline or reference a file path.

## Output

`docs/specs/YYYY-MM-DD-<feature>-spec.md` with exactly these five sections:

1. **Problem Statement** — what business problem this solves, in one paragraph
2. **Salesforce Context** — objects, fields, triggers, custom metadata types, and existing classes affected
3. **Framework Components** — which base-framework pieces apply (e.g., `EventQueue`, `AbstractOutboundCommand`, `TriggerHandler`, `AbstractRepository`)
4. **Acceptance Criteria** — numbered list; each item must be independently testable as a pass/fail condition
5. **Out of Scope** — explicit list of what this feature does NOT cover

## Process

1. Read the user story carefully. Extract the business intent — what problem does the user need solved?
2. Identify affected Salesforce objects, fields, and existing framework components.
3. Map the user story to framework components: which part of the base-framework already handles the problem? Which parts need to be extended?
4. Write acceptance criteria as testable statements: "The system does X when Y" or "Given Z, when A, then B."
5. Surface any ambiguities. Ask for resolution before writing the spec — do not assume.
6. Write spec to `docs/specs/YYYY-MM-DD-<feature>-spec.md`.

## Gate 1

The spec passes Gate 1 only when all of the following are true:
- All five sections are present and filled (no section is empty or contains "TBD")
- Every acceptance criterion is independently testable — someone can verify it as pass or fail without reading the code
- Framework components are identified by class name (not vague terms like "the integration layer")
- Affected Salesforce objects and fields are listed explicitly
- No ambiguity remains — all edge cases mentioned in the user story are addressed

Fail loudly if any condition above is not met. Do not advance to apex-arch until Gate 1 passes.

## References

None — this agent works with natural language only. Do not load any Apex reference files.
