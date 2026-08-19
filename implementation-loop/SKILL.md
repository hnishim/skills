---
name: implementation-loop
description: Execute an explicitly approved implementation Plan through a bounded custom-agent loop. Use when the user invokes $implementation-loop or asks Codex to carry out an already approved Plan with the Luna implementer and Sol reviewer, including tests, review, fixes, and re-review. Do not use to create, negotiate, approve, or materially change a Plan, or when no identifiable approved Plan exists.
---

# Implementation Loop

Treat the approved Plan as the source of truth. Orchestrate the work from the parent thread; delegate implementation and review to the named custom agents.

## Entry gate

1. Identify one concrete Plan in the current task or an explicitly referenced artifact.
2. Confirm that the user explicitly approved that Plan or explicitly invoked this skill to execute that identifiable Plan after it was presented.
3. Capture its scope, acceptance criteria, constraints, required validation, and user-owned changes that must remain untouched.
4. If the Plan is absent, ambiguous, still under discussion, or materially incomplete, stop before spawning an agent. State what is missing and ask only for the decision needed to proceed.
5. Do not create, expand, reinterpret, or approve the Plan inside this skill. A necessary material deviation requires user approval and ends the current loop.

## Agent contract

- Spawn the custom agent named `implementer` for all code and file changes. Do not substitute the parent agent or a generic worker merely because it is available.
- After implementation, spawn the custom agent named `reviewer` for each review cycle. Do not ask the implementer to review its own work.
- Run agents serially because the reviewer depends on the completed implementation and current diff.
- Give each agent the approved Plan, repository/worktree boundaries, relevant AGENTS.md instructions, and the previous agent's structured result. Do not send unrelated conversation history.
- Reuse the same implementer thread for fixes when possible so it retains implementation context. Use a fresh reviewer thread for each cycle so every review evaluates the current state independently.
- Wait for each required result before continuing. Do not claim completion while an agent is still running.

## Workflow

1. Record the pre-existing worktree state and distinguish user-owned changes from changes authorized by the Plan.
2. Spawn `implementer` with the approved Plan and ask it to implement only that Plan, run proportionate tests, and return its required structured report.
3. If the implementer reports `BLOCKED`, stop and report the blocker. Do not ask the reviewer to judge an incomplete or indeterminate implementation.
4. Assemble the review packet:
   - approved Plan and acceptance criteria;
   - current diff and changed-file list, including relevant pre-existing changes;
   - implementer report and exact test outcomes;
   - repository conventions and applicable instructions.
5. Spawn a fresh `reviewer` with the review packet. Count this as one review cycle.
6. Parse the reviewer's exact `status` field.
   - `PASS`: verify that the reported tests and worktree state still match the reviewed revision, then finish.
   - `CHANGES_REQUIRED`: if fewer than three review cycles have occurred, send only the approved Plan plus the reviewer's actionable findings to the existing implementer thread. Ask it to fix those findings without unrelated refactoring, rerun affected tests, and return a new structured report. Then return to step 4.
   - Missing, malformed, or contradictory status: ask the same reviewer once to restate the result in the required schema. If it remains invalid, stop and report the protocol failure.
7. After a third `CHANGES_REQUIRED`, stop without further edits. Escalate the remaining findings and test state to the user.

One review cycle means one completed reviewer verdict. The initial review is cycle 1, so at most two automatic fix rounds follow it before the cycle-3 verdict.

## Review policy

Require the reviewer to evaluate only material, evidence-backed issues in these categories:

- correctness and edge cases;
- conformance with the approved Plan and acceptance criteria;
- scope creep or unrelated changes;
- regressions and compatibility;
- test adequacy and observed test failures;
- security, privacy, error handling, and robustness;
- consistency with repository conventions and applicable instructions.

Use `CHANGES_REQUIRED` only when at least one actionable finding blocks safe acceptance of the approved Plan. Suggestions, preferences, and speculative improvements belong in `non_blocking_notes` and must not prevent `PASS`. Every blocking finding must identify category, severity, evidence with a file/symbol or test reference, impact, and a concrete required change. Do not accept findings that expand the approved Plan unless they address a direct regression, security issue, or unmet acceptance criterion.

## Safety and stop conditions

- Preserve unrelated and pre-existing user changes. Never reset, discard, overwrite, stage, commit, or publish them unless the Plan explicitly requires it.
- Do not commit, push, open a pull request, write to external systems, or perform destructive actions unless the approved Plan explicitly authorizes them.
- Stop for missing authority, destructive work, external side effects, persistent test-environment failure, conflicting instructions, or a material Plan deviation.
- If review findings conflict with the approved Plan, do not choose silently. Report the conflict to the user.

## Completion report

Report:

- final state: `PASS`, `BLOCKED`, or `REVIEW_LIMIT_REACHED`;
- review cycles used;
- files changed and concise implementation summary;
- tests run with pass/fail/not-run status;
- remaining findings, risks, or unverified behavior;
- confirmation that unrelated user changes were preserved, or the exact exception.

Do not add an invitation to continue or propose unrelated follow-up work.
