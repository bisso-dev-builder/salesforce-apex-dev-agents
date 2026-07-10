---
name: apex-dev
description: Orchestrates the full Apex development pipeline. Dispatches apex-spec, apex-arch, apex-impl, apex-test, and apex-review in sequence. Use with a user story to run the complete cycle. Supports manual, semi-automatic, and checkpoint modes.
metadata:
  version: "1.0.0"
  domain: platform
  triggers: apex-dev, full pipeline, develop apex, new integration, new feature, apex development cycle
  role: orchestrator
  scope: orchestration
  output-format: summary
---

# Apex Development Orchestrator

## Contract

Read `apex-patterns/SKILL.md` — specifically the `## Agent Contract` section — before taking any action. This orchestrator enforces every rule defined there through the gates of each dispatched skill.

## Invocation Modes

### Mode A — Manual

Invoke each skill individually. No need to call `apex-dev`.

```
/apex-spec    → produces docs/specs/YYYY-MM-DD-<feature>-spec.md; review Gate 1
/apex-arch    → produces docs/specs/YYYY-MM-DD-<feature>-arch.md; review Gate 2
/apex-impl    → produces .cls files; review Gate 3
/apex-test    → produces test .cls files; review Gate 4
/apex-review  → auto-fix loop; Gate 5
```

Use when: exploratory work, surgical changes, or when you want full manual control at every gate.

### Mode B — Semi-automatic

```
/apex-dev "user story text"
```

Orchestrates all five skills in sequence. Halts only when a gate fails after the skill's maximum retry attempts.

Use when: you trust the pipeline and want hands-off execution on the happy path.

### Mode C — Checkpoints

```
/apex-dev --checkpoint <stage1>,<stage2> "user story text"
```

Like Mode B, but pauses at named stages for human review even when the gate passes. Available stage names: `spec`, `arch`, `impl`, `test`, `review`.

Example:
```
/apex-dev --checkpoint arch,review "user story"
```

Use when: an architect must approve the design before implementation, or a tech lead must sign off before marking the cycle complete.

## Orchestration Process (Modes B and C)

The pipeline is strictly serial. Each stage waits for the previous gate to pass.

```
1. Parse invocation: extract user story text, --checkpoint list (if any).

2. Invoke apex-spec with the user story.
   Await: docs/specs/YYYY-MM-DD-<feature>-spec.md
   Check: Gate 1 conditions (see apex-spec skill)
   If --checkpoint spec: pause. Print "Gate 1 passed. Awaiting review before apex-arch."
                         Resume on human confirmation.

3. Invoke apex-arch with the spec doc.
   Await: docs/specs/YYYY-MM-DD-<feature>-arch.md
   Check: Gate 2 conditions (see apex-arch skill)
   If --checkpoint arch: pause. Print "Gate 2 passed. Awaiting review before apex-impl."
                         Resume on human confirmation.

4. Invoke apex-impl with spec doc + arch doc.
   Await: .cls files under force-app/
   Check: Gate 3 conditions (see apex-impl skill)
   If --checkpoint impl: pause. Print "Gate 3 passed. Awaiting review before apex-test."
                         Resume on human confirmation.

5. Invoke apex-test with produced .cls files.
   Await: test .cls files under force-app/.../test/
   Check: Gate 4 conditions (see apex-test skill)
   If --checkpoint test: pause. Print "Gate 4 passed. Awaiting review before apex-review."
                         Resume on human confirmation.

6. Invoke apex-review with all code.
   Auto-fix loop runs inside apex-review (max 3 attempts).
   Check: Gate 5 conditions (see apex-review skill)
   If --checkpoint review: pause. Print "Gate 5 passed. Awaiting final review."
                           Resume on human confirmation.

7. On full success: print cycle summary:
   - Spec: docs/specs/YYYY-MM-DD-<feature>-spec.md
   - Arch: docs/specs/YYYY-MM-DD-<feature>-arch.md
   - Classes created: [list]
   - Test classes created: [list]
   - Gate 5: passed on attempt N
   - Ready for: sf project deploy start --source-dir force-app
```

## On Gate Failure

When any skill reports a gate failure after exhausting its retry attempts:
1. Print the gate failure report (file, line, violation, suggestion).
2. State which stage failed and what the unresolved violation is.
3. Await human input. Do not attempt to re-run the failed skill automatically.

**Resume after manual fix:**
```
/apex-dev --resume <stage>
```
Re-runs the named stage and all subsequent stages. Does not re-run earlier stages.
Valid stage names: `spec`, `arch`, `impl`, `test`, `review`

**Abort cycle:**
```
/apex-dev --abort
```
Stops the orchestrator. No files are deleted. Produced artifacts remain in place.

## Pipeline is Strictly Serial

No stage runs until the previous gate has passed. There is no parallel execution between stages. This is a hard constraint — not a configuration option.
