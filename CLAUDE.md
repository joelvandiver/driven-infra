# CLAUDE.md — Working agreement for driven-infra

This file governs how Claude works in this repository. It is loaded automatically
at the start of any Claude session that touches this repo. These rules override
any default behavior.

## Rule 1 — Implement, but teach everything

- Claude MAY implement in this repo: Terraform, Ansible, Vagrantfiles,
  manifests, scripts, and docs.
- This technology is new to Joel, and the goal of the project is that he fully
  understands everything in it (see `docs/PLAN.md` §"instructive"). So every
  change Claude makes must come with a thorough explanation: what was written,
  why that approach, what the key concepts are, and what the alternatives were.
  No silent edits — a change Joel can't explain afterward is a failed change.
- When explaining, prefer primary references (kubeadm docs, Terraform registry
  docs, Ansible module docs) over blog-style summaries; link them.
- Commits and pushes only when Joel asks.

## Rule 2 — Break work into tasks, and Joel verifies every gate

- When Joel says "next requirement", names a phase, or brings any new piece of
  work: use the `breakdown` skill (`.claude/skills/breakdown/`) to decompose it
  into small, independently verifiable tasks BEFORE implementation begins.
- One task at a time. Do not implement task N+1 while task N is unverified.
- Every task ends with a **verification gate**: a concrete check Joel runs
  himself (a command, an observable state, a re-run showing idempotency).
  Claude may run its own checks while implementing, but the gate belongs to
  Joel: he runs it, shares the output, Claude reviews it, and only then does
  the next task begin.
- Claude never marks a task done on Joel's behalf. Evidence from Joel's machine
  is the ground truth.
- If Joel shares a diff or output, review it critically: name what is correct,
  what is wrong, and *why* — referencing the underlying concept, not just the fix.

## Teaching style

- Always explain the "why" behind a recommendation or an implementation choice.
- Use the pinned versions in `bootstrap/versions.yml` as the single source of
  truth when discussing tool or component versions.
- Keep the phase discipline from `docs/PLAN.md`: each phase ends in a verifiable
  state before the next begins.
