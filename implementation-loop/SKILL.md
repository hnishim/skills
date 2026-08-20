---
name: implementation-loop
description: Execute an explicitly approved implementation Plan through a test-first, bounded custom-agent loop. Use when the user invokes $implementation-loop or asks Codex to carry out an already approved Plan with implementer-authored tests, independent test review, implementation, review, fixes, and re-review. Do not use to create, negotiate, approve, or materially change a Plan, or when no identifiable approved Plan exists.
---

# Implementation Loop

Treat the approved Plan as the source of truth. Establish an independently reviewed acceptance-test baseline before implementation, then orchestrate the bounded implementation loop from the parent thread.

## Entry gate

1. Identify one concrete Plan in the current task or an explicitly referenced artifact.
2. Confirm that the user explicitly approved that Plan or explicitly invoked this skill to execute that identifiable Plan after it was presented.
3. Capture its scope, acceptance criteria, constraints, required validation, and user-owned changes that must remain untouched.
4. Record the effective model and reasoning effort for the parent, `implementer`, and `reviewer` at loop start when the harness exposes them. The `implementer` custom-agent configuration must use reasoning effort `medium`; the reviewer remains independently configured. If a value is unavailable, record `unknown`; never infer historical settings from a later configuration snapshot.
5. Before spawning an agent, derive an acceptance-and-risk matrix from the Plan. Map every acceptance criterion and non-functional risk to one or more automated checks, fault-injection checks, or explicit manual acceptance checks. Include boundary, malformed-input, dependency, cleanup, signal/interruption, security/privacy, documentation-consistency, and external-system risks where applicable.
6. If the Plan is absent, ambiguous, still under discussion, materially incomplete, not testable as written, or leaves a product/security/failure-handling policy undecided, stop before spawning an agent. State what is missing and ask only for the decision needed to proceed.
7. Do not create, expand, reinterpret, or approve the Plan inside this skill. A necessary material deviation requires user approval and ends the current loop.

## Agent contract

- The names `implementer` and `reviewer` are fixed custom-agent identifiers; their model, reasoning effort, and sandbox settings are defined in the custom agent TOML files. Keep `implementer` at reasoning effort `medium` unless the user explicitly changes that policy.
- Spawn the custom agent named `implementer` for all test, fixture, implementation, and documentation changes. Do not substitute the parent agent or a generic worker merely because it is available.
- Use the custom agent named `reviewer` first to review the tests-only change, then for implementation review. Do not ask the implementer to approve its own tests or implementation.
- Run agents serially because each reviewer depends on a completed tests-only or implementation diff.
- Give each agent the approved Plan, repository/worktree boundaries, relevant AGENTS.md instructions, and the previous agent's structured result. Do not send unrelated conversation history.
- Reuse the same implementer thread across test design, implementation, and fixes when possible. Reuse the test reviewer only within the tests-only gate; use a fresh reviewer for each post-implementation review cycle.
- Wait for each required result before continuing. Do not claim completion while an agent is still running.

## Agent lifecycle and cleanup

- Keep at most one implementer and one reviewer active at a time.
- Reuse the same implementer thread throughout a Plan unless it is blocked, unavailable, or its context is no longer reliable.
- Reuse one reviewer throughout the test-design gate so it can verify its own findings consistently. After closing that reviewer, start one fresh reviewer per implementation-review cycle.
- Close the test reviewer when the tests-only gate reaches a terminal status. Close each implementation reviewer immediately after its verdict.
- Retain the implementer through the tests-only gate and implementation loop. Close it after `PASS`, `BLOCKED`, `TEST_DESIGN_BLOCKED`, `REVIEW_LIMIT_REACHED`, or when the loop is interrupted.
- Before resuming an interrupted or previously stopped loop, close stale completed or interrupted agents from that loop and confirm that no reviewer remains active.
- On any terminal or interrupted path, clean up all agents spawned by this loop before reporting the result.

## Phase 1: tests-only gate

1. Record the pre-existing worktree state and distinguish user-owned changes from changes authorized by the Plan. Attach the acceptance-and-risk matrix to the test-design packet.
2. Spawn `implementer` with the approved Plan and require a tests-only change. It may add or edit tests, fixtures, fakes, and test-only helpers, but it must not add or edit production code, runtime configuration, user documentation, generated production artifacts, or external systems.
3. Require the implementer to translate every acceptance criterion and risk-matrix row into observable checks, including success paths, boundary and malformed inputs, dependency and transport failures, cleanup failures, signal/interruption behavior, security/privacy behavior, documentation consistency, and non-mutating compile or static checks. Record any behavior that cannot be automated as a manual acceptance check instead of simulating success.
4. Require the implementer to run the existing suite and the new tests separately. Existing tests must still pass. Classify each new result as `EXPECTED_FAIL`, `UNEXPECTED_FAIL`, or `PASS`, and explain why each expected failure demonstrates missing production behavior rather than a broken test. Tests do not need to pass before implementation when an `EXPECTED_FAIL` intentionally proves an absent behavior; they must pass before implementation review.
5. If the implementer reports `BLOCKED`, stop. If it changes non-test files, return the boundary violation once and require it to remove only its own out-of-scope changes before review; if the violation remains, stop as `BLOCKED`. Do not begin implementation.
6. Spawn one `reviewer` with the Plan, acceptance-and-risk matrix, tests-only diff, existing and new test results, manual acceptance checks, repository instructions, and pre-existing changes. The reviewer must inspect the tests independently and verify coverage of the matrix, not merely approve test syntax or source-string assertions. It must perform a complete review rather than stopping after the first finding.
7. Parse the test reviewer's exact `status`:
   - `TESTS_APPROVED`: close the test reviewer and continue to Phase 2.
   - `TESTS_CHANGES_REQUIRED`: if fewer than two test-review verdicts have occurred, send only its actionable findings to the existing implementer, request tests-only corrections, rerun the affected tests, and return to step 6 using the same reviewer.
   - `PLAN_INCOMPLETE`: close all loop agents, stop as `BLOCKED`, and report the exact missing decision; neither agent may choose the requirement.
   - Missing, malformed, or contradictory status: ask the same reviewer once to restate the result. If it remains invalid, stop and report the protocol failure.
8. After a second `TESTS_CHANGES_REQUIRED`, stop as `TEST_DESIGN_BLOCKED` without production changes.
9. On `TESTS_APPROVED`, record the approved test diff, changed-file list, acceptance-and-risk matrix, manual checklist, and content hashes as the locked acceptance baseline. Later implementation may add non-weakening tests but must not delete, weaken, skip, or modify an approved test. If an approved test is proven invalid, stop as `BLOCKED` and request a new test-design gate instead of changing it inside the implementation loop.

The tests-only gate is separate from the three implementation-review cycles. It has at most two reviewer verdicts and must complete before any production implementation begins.

## Phase 2: implementation and bounded review

1. Send the approved Plan, locked test baseline, acceptance-and-risk matrix, manual acceptance checks, repository boundaries, and test-review result to the same `implementer`. Ask it to implement only the Plan, make the approved tests pass, add any non-weakening regression tests, and run all permitted validation.
2. Require an implementer self-check before review: compare the full diff with the Plan and matrix, confirm the locked tests are unchanged, exercise each defined failure injection including cleanup and signal/interruption paths where applicable, run safe compile/static checks even when live E2E is prohibited, verify documentation matches actual persistence and failure behavior, and distinguish tested behavior from unverified live behavior.
3. If the implementer reports `BLOCKED`, a locked-test change, or a material Plan deviation, stop before implementation review and report it.
4. Assemble the implementation review packet:
   - approved Plan, acceptance criteria, and acceptance-and-risk matrix;
   - locked test baseline and hash comparison;
   - current diff and changed-file list, including relevant pre-existing changes;
   - implementer report and exact test outcomes;
   - manual acceptance checks and what remains unverified;
   - repository conventions and applicable instructions.
5. Spawn a fresh `reviewer` with the packet. Count this as one implementation-review cycle. Require a complete, evidence-backed review of the full diff, tests, fault-injection and signal paths, documentation consistency, manual checklist, and scope boundaries. The reviewer must return all material findings discoverable in that pass, rather than intentionally deferring them.
6. Parse the reviewer's exact `status`:
   - `PASS`: verify that the reported tests, locked-test hashes, and worktree state still match the reviewed revision, then finish.
   - `CHANGES_REQUIRED`: if fewer than three implementation-review cycles have occurred, send only the approved Plan plus the matrix and reviewer's actionable findings to the existing implementer. Ask it to add a failing regression test for each reproducible defect before fixing it, preserve the locked baseline, rerun affected and full tests, and return a new structured report. Classify each finding as an acceptance gap, implementation defect, documentation mismatch, or unresolved policy decision. Then return to step 4.
   - Missing, malformed, or contradictory status: ask the same reviewer once to restate the result. If it remains invalid, stop and report the protocol failure.
7. After a third `CHANGES_REQUIRED`, stop without further edits as `REVIEW_LIMIT_REACHED`. This is an incomplete, non-accepting state, not a successful completion. Escalate the remaining findings, risk classification, manual checklist, and test state to the user. Do not mark the Plan complete or defer a high-severity correctness, security, privacy, or data-loss finding without explicit user risk acceptance.

One implementation-review cycle means one completed post-implementation reviewer verdict. The initial implementation review is cycle 1, so at most two automatic fix rounds follow it before the cycle-3 verdict.

## Test review policy

Require the test reviewer to verify all of the following:

- every Plan acceptance criterion maps to an observable automated or manual check;
- tests specify public behavior and externally visible side effects without unnecessarily fixing an internal design;
- success, empty/boundary input, malformed data, dependency failure, cleanup failure, and security/privacy paths are covered where applicable;
- fakes and mocks preserve the relevant contract and can inject each required failure independently;
- new tests fail for the intended missing behavior, not because of syntax errors, invalid fixtures, missing test dependencies, or unrelated environment failures;
- existing behavior and unrelated user changes remain intact;
- manual checks identify real launcher, GUI, permission, network, or external-system validation that static tests cannot prove;
- the tests do not weaken, reinterpret, or expand the Plan.
- the acceptance-and-risk matrix has no unexplained row; every manual check names the real launcher, permission, network, external-system, or signal behavior that static tests cannot prove;
- cleanup and abnormal termination are tested or explicitly listed as manual/unverified, and documentation accurately states temporary persistence and residual risks.

Use `TESTS_CHANGES_REQUIRED` only for actionable gaps that could allow an incorrect implementation to pass. Return all material findings in one verdict. Use `PLAN_INCOMPLETE` when a test would require choosing a product, security, compatibility, or failure-handling policy not fixed by the Plan.

## Review policy

Require each implementation reviewer to evaluate only material, evidence-backed issues in these categories:

- correctness and edge cases;
- conformance with the approved Plan and acceptance criteria;
- scope creep or unrelated changes;
- regressions and compatibility;
- test adequacy and observed test failures;
- security, privacy, error handling, and robustness;
- consistency with repository conventions and applicable instructions.

Use `CHANGES_REQUIRED` only when at least one actionable finding blocks safe acceptance of the approved Plan. Suggestions, preferences, and speculative improvements belong in `non_blocking_notes` and must not prevent `PASS`. Every blocking finding must identify category, severity, evidence with a file/symbol or test reference, impact, and a concrete required change. Return all material findings in one verdict; do not intentionally defer discoverable issues to later cycles. Do not accept findings that expand the approved Plan unless they address a direct regression, security issue, or unmet acceptance criterion.

## Definition of done

Declare `PASS` only when all of the following hold:

- every acceptance-and-risk matrix row is covered by a passing automated/fault-injection check or a completed manual acceptance check;
- the locked acceptance tests are unchanged and pass, and added regression tests pass;
- static checks and proportionate runtime checks pass, with failures classified and reported;
- documentation matches actual persistence, cleanup, permissions, and residual-risk behavior;
- no material reviewer findings remain and no high-severity risk is being silently deferred.

## Safety and stop conditions

- Preserve unrelated and pre-existing user changes. Never reset, discard, overwrite, stage, commit, or publish them unless the Plan explicitly requires it.
- Do not commit, push, open a pull request, write to external systems, or perform destructive actions unless the approved Plan explicitly authorizes them.
- During the tests-only gate, do not permit production or documentation changes. During implementation, do not permit changes that weaken the locked acceptance baseline.
- Stop for missing authority, destructive work, external side effects, persistent test-environment failure, conflicting instructions, or a material Plan deviation.
- If review findings conflict with the approved Plan, do not choose silently. Report the conflict to the user.

## Completion report

Report:

- final state: `PASS`, `BLOCKED`, `TEST_DESIGN_BLOCKED`, or `REVIEW_LIMIT_REACHED` (the latter is explicitly incomplete);
- effective model and reasoning settings recorded at loop start, including `unknown` values;
- test-review verdicts and implementation-review cycles used;
- locked acceptance-test files and whether their hashes remained unchanged;
- acceptance-and-risk matrix coverage and completed/unverified manual checks;
- files changed and concise implementation summary;
- tests run with pass/fail/not-run status;
- remaining findings, risks, or unverified behavior;
- confirmation that unrelated user changes were preserved, or the exact exception.

Do not add an invitation to continue or propose unrelated follow-up work.
