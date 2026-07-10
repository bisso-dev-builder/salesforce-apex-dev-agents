---
name: apex-arch
description: Validates framework adherence and defines class architecture from a spec doc. Second step in the Apex development pipeline. Use after apex-spec and before apex-impl.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: architecture, design, arch, class design, apex-dev pipeline
  role: architect
  scope: planning
  output-format: markdown
---

# Architecture Agent

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before taking any action. All inviolable rules are defined there.

## Input

- `docs/specs/YYYY-MM-DD-<feature>-spec.md` (Gate 1 must have passed)

## Output

`docs/specs/YYYY-MM-DD-<feature>-arch.md` with exactly these five sections:

1. **Class List** — every class and interface to be created or modified; for each: exact class name, sharing model (`inherited sharing` or `with sharing`), base-framework class extended or interface implemented, one-sentence responsibility statement
2. **Dependencies** — for each class: what it depends on, stated as abstractions not concretes
3. **Framework Mapping** — table: requirement → framework component that covers it
4. **Custom Metadata Records** — list of `EventConfiguration__mdt`, `EventOutboundConfig__mdt`, or `EventOutboundEnvironment__mdt` records needed, with key field values
5. **File Paths** — exact path under `force-app/main/default/classes/` for each class

## Process

1. Read `docs/specs/YYYY-MM-DD-<feature>-spec.md`.
2. Load references: `apex-base-framework.md`, `repository.md`, `trigger-patterns.md`.
3. For each requirement in the spec: identify the base-framework pattern that covers it. Use it. Do not design a new class when an existing framework abstraction already handles the problem.
4. For each new class: write a one-sentence responsibility statement. If you cannot write one clear sentence, the class has too many responsibilities — split it.
5. Check for duplication: if the framework already provides a capability (e.g., `EventDequeuer`, `EventExecutor`, `Callout`), do not recreate it.
6. List all Custom Metadata records required, with key field values filled in.
7. Write arch doc to `docs/specs/YYYY-MM-DD-<feature>-arch.md`.

## Gate 2

The arch doc passes Gate 2 only when all of the following are true:
- Every class has a one-sentence responsibility statement
- Every dependency is stated as an abstraction (interface or abstract class), not a concrete type, where the framework provides one
- No framework infrastructure is being reimplemented — if the framework covers it, the table shows it
- All required Custom Metadata records are listed with key field values
- No class, method, or parameter name contains an acronym or abbreviation

Fail loudly if any condition is not met. Do not advance to apex-impl until Gate 2 passes.

## References

Load: `references/apex-base-framework.md`, `references/repository.md`, `references/trigger-patterns.md`
