---
name: run-long-task
description: Carries a large agent effort from a countable completion contract through production, independent review, and verified finish. Use when one agent run or supervised effort must complete a long refactor, migration, backlog, bulk generation, or cleanup task.
---

# Run a long task (run-long-task)

Build a loop that produces bounded work, challenges it independently, applies corrections, and proves the result. Measure progress by completed work and executable evidence, not output volume.

## 1. Frame the completion contract

Before producing artifacts, write down what the effort must make true:

- **Finish line** — name the completed outcome, not an activity such as "keep improving."
- **Proof** — name the checks and observable results that establish that outcome.
- **Boundaries** — name the assigned scope, behavior to preserve, and operations workers may perform.
- **Stop and report** — state when a blocker requires reporting instead of widening scope or forcing a pass.

Make the work queue-shaped: files to transform, failures to repair, or scoped items to complete. Count what remains so progress means a shrinking backlog.

Set the stop condition for this effort; do not import an arbitrary retry count from another project. If it is reached, report the remaining work and missing proof honestly.

For a rewrite, identify the recurring failure class it should prevent and compare alternatives before committing. Include displaced fixes, security work, and features in the cost; use a bounded experiment to test feasibility.

## 2. Choose the work unit and gates

Define what **one unit done** means before assigning work. Include production, independent review, applied corrections, and the checks appropriate to that phase.

Choose units that make failures and edits local enough to inspect:

- Use individual artifacts or files during bulk production.
- Use packages with diagnostics grouped by file during compiler repair.
- Use commands with saved stacktraces during runtime bring-up.
- Use test files or platform failures during verification repair.

Change the unit when the phase changes. A translated file can finish its production loop while the assembled program still cannot run; keep that distinction visible.

Write the gate sequence before starting. For executable code, use types → link → start → one test → full suite → full CI matrix. For other artifacts, identify the corresponding validity, actual-use, and complete verification checks.

Require evidence at each gate. Do not promote a production milestone into proof that a later gate passed.

## 3. Write shared decisions once

Record common patterns and required conventions before parallel production. Give workers the same guide instead of asking each to invent its own rules.

Record difficult item-specific decisions separately. For code, trace actual control flow to understand ownership, escaping references, asynchronous retention, and cleanup before choosing their representation.

Have independent reviewers challenge these decisions and apply their findings. Review general guidance and item-specific records together so they cannot silently contradict each other.

Have the supervising human read the corrected guidance. Use it as an input to both production and review throughout the effort.

For a migration, decide all-at-once versus incremental and account for temporary code. Preserve recognizable structure and behavior; defer extensive idiomatic redesign.

## 4. Pilot the complete loop

Run a tiny sample through the intended production process before scaling:

1. Select a bounded item and give its implementer the required context and shared decisions.
2. Produce the artifact.
3. Obtain independent adversarial reviews against the source of truth.
4. Apply findings through a separate fixer.
5. Exercise the checks and repository boundary defined for that work unit.

Inspect whether the loop produces faithful work and catches mistakes. Repair faulty instructions before repeating them across the backlog.

Use the pilot to test the process, not just generation quality. Claim only the gates it actually exercised; a reviewed sample is not automatically an integrated, runnable result.

## 5. Bound concurrency by infrastructure

Measure repository and worktree size, available disk space, and I/O throughput before increasing parallelism. Let storage capacity and observed read/write performance set the ceiling.

Use a manageable number of worktrees and bounded assignments. Do not assume one worktree per agent fits the disk or that more model calls create more throughput.

Scope worker mutations to assigned output. Keep repository and build operations at controlled phase boundaries so workers do not interfere with each other.

- During production, keep slow shared builds out of individual worker loops.
- During compiler repair, run the shared diagnostic command at the start of the scoped workflow.
- At the end of a work unit, commit only its assigned files at the permitted boundary.
- Never let parallel workers stash, pop stashes, reset, or otherwise manipulate shared state mid-flight.

Recheck resource pressure while scaling. If searches stall or disk fills, correct the infrastructure or concurrency setting rather than feeding more work into the bottleneck.

## 6. Keep authoring and review independent

Use **one implementer, two or more adversarial reviewers, and one fixer** per work unit. Keep the roles separate: the implementer does not review, reviewers do not implement, and the fixer applies findings afterward.

Give reviewers separate context windows containing the artifact or diff and access to prior behavior and shared guidance. Exclude the author's reasoning and rationale: the author wants acceptance; reviewers must seek failure.

Brief reviewers: **"Assume this is wrong and enumerate how it breaks."** Require comparison with the source of truth, not merely internal consistency or a green check.

Preserve this arrangement during repairs and cleanup as well as initial production. Apply findings before counting the unit's review-and-correction work as complete.

See `adversarial-loops` for the focused review procedure.

## 7. Save failures as scoped work queues

Capture diagnostic output at a controlled point instead of letting every worker react to live shared failures.

- For compiler repair, resolve dependency cycles first: classify where the code belongs, record the plan, then refactor. Save subsequent compiler errors by package and file.
- For runtime bring-up, save each failing stacktrace with its command or subcommand.
- For local tests, run bounded batches and save failures with their test files.
- For CI, group failures by supported platform and continue repairs until complete runs pass.

Assign each saved scope to the implement–review–fix loop. Keep diagnostics and review findings as recorded inputs to the next step, not transient conversation.

Run checks at the workflow's controlled boundaries to establish the next backlog. Keep compilation, repository mutation, and worker edits from competing mid-flight.

## 8. Repair the process that repeats mistakes

When the same class of defect recurs, inspect the prompt and review instructions that produced or accepted it. Change that loop and inspect subsequent output to see whether the correction holds.

Reject stubs that make errors disappear by discarding required behavior. Reject paragraph-long comments defending a workaround instead of fixing it.

Apply corrections to affected artifacts through the review loop. Do not leave the generator unchanged while repeatedly hand-fixing its output.

For transformations, direct reviewers toward semantic discrepancies: build-mode-dependent side effects, conversion edge cases, capacity and bounds, evaluation order, and objects retained across asynchronous boundaries.

Compare against the original behavior even when the new artifact looks mechanically equivalent. Syntax resemblance and internal consistency can both hide regressions.

## 9. Advance through actual execution

Reuse existing behavioral checks wherever they remain applicable. Source comparison and independent review supplement those checks; they do not replace execution.

For executable work, establish each gate in order:

1. **Types:** repair diagnostics without deleting required behavior.
2. **Link:** build the executable and resolve linkage failures.
3. **Start:** run the result and investigate immediate failures.
4. **One test:** establish that the test runner can exercise behavior.
5. **The suite:** use bounded batches to converge, then pass the full suite.
6. **The CI matrix:** complete all required tests on every supported platform.

For other work, execute the validity and actual-use checks selected in the contract, then verify the full required scope. Do not substitute an inspected sample for completion.

Isolate resource-heavy tests with operating-system controls for memory, CPU, and processes where needed. Continue monitoring disk and I/O; process isolation does not prevent disk exhaustion.

Investigate build-mode timeouts and semantic differences rather than treating every failure as equivalent. If performance continuity matters, compare it separately from functional correctness.

## 10. Verify the verification with the human

**A green run is not evidence until you confirm the checks actually executed.** Inspect execution results, test counts, skips, and completion of the required scope.

Do not accept deleted tests, skipped checks, passing subsets, or superseded partial CI runs as a full pass. Verify completed runs across the required platform matrix.

Make the supervising human's job explicit: **monitor outputs, sample work, and edit the loop.** Keep the evidence available for them to:

- Read shared decisions and compare sampled diffs with their source of truth.
- Confirm reviewers find discrepancies and enforce the guide.
- Confirm checks execute instead of merely reporting green status.
- Run additional relevant local checks before deciding to integrate.

Use those observations to correct the workflow. Do not ask the human to infer confidence from generated lines, agent counts, or commit volume.

## 11. Separate integration from release

Report which finish line has been reached and the evidence supporting it. Keep **integrated**, **hardened**, and **released** distinct; merging into the main branch is not a versioned release.

Continue hardening after integration. Investigate regressions, address security findings, and use applicable fuzzing and memory checks to expose failures missed by the initial suite.

For fuzz-generated fixes, produce a reproducer and fix for human review. Expand applicable sanitizer or interpreter-based checking as the code permits; do not imply whole-system coverage from a partial check.

Defer extensive redesign until fidelity is established. Make the release decision from the additional evidence rather than inheriting it from the merge decision.

## Do not

- Start a large job with one broad prompt and hope it works.
- Scale production before piloting implementation, review, and correction.
- Let each worker invent shared conventions or let authors review themselves.
- Size concurrency by ambition while ignoring repository size, disk, and I/O.
- Let parallel workers stash, reset, or run uncontrolled shared operations.
- Silence failures with stubs, missing behavior, or elaborate workaround defenses.
- Treat generated artifacts, successful compilation, or green status as verified completion.
- Remove or skip required checks to force a pass, or call a sample the full suite.
- Equate integration with release or stop hardening because the work merged.

## Common mistakes

| Mistake | Better |
| --- | --- |
| Defining effort instead of an outcome | Write a finish line, proof, boundaries, and a stop-and-report rule |
| Counting output volume as progress | Count completed work units and the remaining scoped queue |
| Leaving "one unit done" implicit | Define its production, review, correction, and phase checks |
| Distributing unresolved shared decisions | Record and review common guidance and hard cases together |
| Piloting only artifact generation | Exercise the complete work-unit loop before scaling |
| Adding workers while disk or I/O stalls | Measure the infrastructure ceiling and bound concurrency |
| Letting workers control shared state mid-flight | Schedule shared operations at controlled phase boundaries |
| Reviewing with the author's rationale | Separate contexts and compare against the source of truth |
| Chasing live failures across the whole job | Save diagnostics and assign scoped repair queues |
| Hand-fixing a recurring generated defect | Change the generating or reviewing process and inspect the result |
| Assuming the next gate follows from the last | Execute each gate and keep its evidence distinct |
| Trusting a green badge | Confirm required checks actually ran and completed without skips |
| Leaving the human outside the loop | Have them monitor, sample, and edit the workflow |
| Reporting a merge as a release | Keep integration and release decisions separate and continue hardening |
