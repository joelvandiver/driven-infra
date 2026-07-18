---
name: breakdown
description: Break the next driven-infra requirement or phase into small, verifiable tasks that Joel implements himself. Use whenever Joel asks to start a phase, says "next requirement" or "break this down", or brings any new piece of work to the repo. Guide only — never implement the tasks.
---

# Requirement breakdown for driven-infra

Turn one requirement (usually a phase from `docs/PLAN.md`) into a sequence of
small tasks Joel can implement and verify himself. This skill produces a
*breakdown*, never an implementation — see `CLAUDE.md` Rule 1.

## Procedure

1. **Locate the requirement.** If Joel named a phase, re-read that phase in
   `docs/PLAN.md` (goals, files, definition of done). If it's new work outside
   the plan, restate the requirement back to him in one sentence and get a nod
   before decomposing.
2. **Check current state.** Look at the repo (and ask Joel about his machine's
   state where relevant) so tasks start from where things actually are, not
   where the plan assumes they are.
3. **Decompose into tasks** with ALL of these properties:
   - Small: roughly 15–60 minutes of Joel's time each.
   - Ordered by dependency, one concern per task (e.g. "add the HashiCorp apt
     repo via Ansible" and "install pinned Terraform from it" are two tasks).
   - Self-contained enough that a failure is diagnosable within the task.
4. **Give each task this shape:**
   - **Goal** — one sentence, outcome-oriented.
   - **Files** — which paths Joel will create or edit.
   - **Guidance** — the concepts involved and pointers to the primary docs
     (module names, resource types, flags). Hints, not solutions.
   - **Verify** — the gate: exact command(s) Joel runs and what output means
     success. Where it applies, include the idempotency check (run it twice;
     second run changes nothing).
   - **Teaches** — the one thing Joel should understand after this task.
5. **Present the full breakdown first** so Joel sees the arc, then guide
   task 1 only. Wait at every gate: Joel runs the verification, shares the
   output, Claude reviews it, and only then does task N+1 begin.
6. **On failure at a gate:** debug within the task. Ask for the exact error,
   reason from the concept ("kubelet and containerd disagree on cgroup driver —
   which file controls that?"), and let Joel make the fix.

## Rules

- Never write the task's files, run its commands for Joel, or commit/push its
  results. Chat-only fragments are the ceiling (per `CLAUDE.md`).
- Never collapse multiple tasks into one "just do all this" message.
- Never advance past an unverified task, even if Joel seems confident.
- If a task turns out bigger than an hour mid-flight, stop and split it.

## Output format

A short intro naming the requirement and how many tasks, then:

```
### Task N — <imperative title>
Goal: ...
Files: ...
Guidance: ...
Verify: ...
Teaches: ...
```

End with: "Start with Task 1 — tell me when you have the verification output."
