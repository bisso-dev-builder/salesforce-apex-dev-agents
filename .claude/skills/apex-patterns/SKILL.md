---
name: apex-patterns
description: Writes and debugs Apex code, builds Lightning Web Components, optimizes SOQL queries, implements triggers, batch jobs, platform events, and integrations on the Salesforce platform. Use when developing Salesforce applications, customizing CRM workflows, managing governor limits, bulk processing, or setting up Salesforce DX and CI/CD pipelines. Within the apex development pipeline, this skill is a constitutional reference — not invoked directly. Use /apex-dev to run the full pipeline.
license: MIT
metadata:
  author: https://github.com/ercarval
  version: "2.0.0"
  domain: platform
  triggers: Salesforce, Apex, Lightning Web Components, LWC, SOQL, SOSL, Visualforce, Salesforce DX, governor limits, triggers, platform events, CRM integration, Sales Cloud, Service Cloud
  role: expert
  scope: implementation
  output-format: code
  related-skills: apex-dev, apex-spec, apex-arch, apex-impl, apex-test, apex-review
---

# Salesforce Developer

## Agent Contract

These rules are inviolable across all Apex development on this project. Every agent in the apex development pipeline reads this section before acting. No agent may skip or override these rules.

### Clean Code
- No acronyms, initialisms, or abbreviated words in any identifier (class, method, variable, parameter) — see `references/apex-dev-guide.md` "No Acronyms" section for the full offender list
- Methods have a single level of abstraction and a single reason to exist
- No explanatory comments — only "why" comments for non-obvious constraints or Salesforce-specific workarounds
- No magic numbers or hardcoded strings; use named constants
- No dead code

### SOLID
- **SRP:** Every class has one reason to change. Write a one-sentence responsibility statement; if you cannot write one, split the class.
- **OCP:** Extend behavior via the framework's extension points (abstract classes, interfaces). Never modify existing framework classes.
- **DIP:** Depend on framework abstractions (`ILogger`, `AbstractRepository`, `AbstractOutboundCommand`, `Command`), not on concrete implementations.

### base-framework Adoption
- Use `Lists` and `Maps` utilities instead of custom collection helpers
- Use `Logger` for all diagnostic output — never `System.debug`
- Use `TypeFactory` for dynamic class instantiation
- Extend `AbstractRepository` for all DML and SOQL
- Extend `AbstractOutboundCommand` for all outbound HTTP integrations
- Use `TriggerHandler` + trigger action pattern for all trigger logic
- Never reimplement what the base-framework already provides

### Apex Platform Rules
- Bulkify all code — SOQL and DML outside loops, always
- Use `inherited sharing` by default; use `with sharing` only for TriggerHandler, RestResource, and Controller classes
- Use Salesforce DX for all deployments (`sf project deploy start`)
- Write test classes with 100% coverage; use FixtureFactory for test data

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Apex Dev Guide | `references/apex-dev-guide.md` | Coding convention |
| Base Apex Framework | `references/apex-base-framework.md` | Writing Apex code in the integration framework project |
| Adopting Repository Pattern | `references/repository.md` | How to use Repository Pattern |
| Adopting Trigger Patterns | `references/trigger-patterns.md` | How to use Trigger Patterns (TriggerHandler, Filter, Enricher, Publisher, Validator) |
| Adopting Test Patterns | `references/test-patterns.md` | How to use Test Patterns (FixtureFactory, StringGenerator, Mocker, Fake Classes) |
| Lightning Web Components | `references/lightning-web-components.md` | LWC framework, component design, events, wire service |
| SOQL/SOSL | `references/soql-sosl.md` | Query optimization, relationships, governor limits |
| Deployment & DevOps | `references/deployment-devops.md` | Salesforce DX, CI/CD, scratch orgs, metadata API |

## Constraints

### MUST DO
- Apply all rules in the Agent Contract above
- Bulkify Apex code — collect IDs/records before loops, query/DML outside loops
- Use `with sharing` for: `TriggerHandler`, `RestResource`, `Controllers` — otherwise use `inherited sharing`
- Always use Trigger Patterns in Trigger Scope
- Use Repository Pattern for all DML and SOQL/SOSL
- Use FixtureFactory Pattern for Test Data
- Write test classes with 100% code coverage, including bulk scenarios
- Use selective SOQL queries with indexed fields; leverage relationship queries
- Use appropriate async processing (batch, queueable, future) for long-running work
- Use Salesforce DX for source-driven development and metadata deployment

### MUST NOT DO
- Use acronyms, initialisms, or abbreviated words in class names, method names, or variable names
- Execute SOQL/DML inside loops
- Hard-code IDs or credentials in code
- Create recursive triggers without safeguards
- Skip field-level security and sharing rules checks
- Use deprecated Salesforce APIs or components
