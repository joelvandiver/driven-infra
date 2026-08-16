---
name: breakdown
description: Break the next driven-infra requirement or phase into small, verifiable tasks that Claude implements and Joel verifies. Use whenever Joel asks to start a phase, says "next requirement" or "break this down", or brings any new piece of work to the repo. Every task ends with a verification gate Joel runs himself.
---

# Requirement breakdown for driven-infra

Turn one requirement (usually a phase from `docs/PLAN.md`) into a sequence of
small tasks. Claude implements each task with a thorough explanation
(see `CLAUDE.md` Rule 1); Joel verifies each gate before the next task begins
(Rule 2).

## Procedure

1. **Locate the requirement.** If Joel named a phase, re-read that phase in
   `docs/PLAN.md` (goals, files, definition of done). If it's new work outside
   the plan, restate the requirement back to him in one sentence and get a nod
   before decomposing.
2. **Check current state.** Look at the repo (and ask Joel about his machine's
   state where relevant) so tasks start from where things actually are, not
   where the plan assumes they are.
3. **Decompose into tasks** with ALL of these properties:
   - Small: one coherent change Joel can review and understand in one sitting.
   - Ordered by dependency, one concern per task (e.g. "add the HashiCorp apt
     repo via Ansible" and "install pinned Terraform from it" are two tasks).
   - Self-contained enough that a failure is diagnosable within the task.
4. **Give each task this shape:**
   - **Goal** — one sentence, outcome-oriented.
   - **Files** — which paths will be created or edited.
   - **Concepts** — what Joel will learn from this task, with pointers to the
     primary docs (module names, resource types, flags).
   - **Verify** — the gate: exact command(s) Joel runs and what output means
     success. Where it applies, include the idempotency check (run it twice;
     second run changes nothing).
5. **Present the full breakdown first** so Joel sees the arc, then implement
   task 1 only, explaining the change as it's made. Wait at every gate: Joel
   runs the verification, shares the output, Claude reviews it, and only then
   does task N+1 begin.
6. **On failure at a gate:** debug within the task. Get the exact error,
   explain the cause from the underlying concept ("kubelet and containerd
   disagree on cgroup driver — this file controls that"), then fix it and
   re-verify.

## Rules

- Never advance past an unverified task, even if things look fine.
- Never collapse multiple tasks into one "just do all this" change.
- Never make a change without explaining it — what, why, and the alternatives.
- If a task turns out bigger than expected mid-flight, stop and split it.

## Output format

A short intro naming the requirement and how many tasks, then:

```
### Task N — <imperative title>
Goal: ...
Files: ...
Concepts: ...
Verify: ...
```

End with: "Starting with Task 1 — I'll walk you through the change, then hand
you the verification gate."
