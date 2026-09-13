---
name: adversarial-loops
description: Runs independent adversarial review and correction loops to catch failures beyond compilation and passing tests. Use when an agent produces a large volume of consequential code, configs, or migrations that will be merged or released.
---

# Run adversarial reviews (adversarial-loops)

Make review try to disprove the artifact's correctness. Compiling and passing tests do not replace comparison with the behavior and shared decisions the artifact must preserve.

## Separate the contexts

**Never let the author review their own work.** The author wants the result accepted; the reviewer must want to find what is wrong. Give them separate context windows and separate objectives.

Give each reviewer the diff or artifact and access to the source of truth: prior behavior, original source, and the shared guide. Do not pass along the author's reasoning, rationale, or persuasive explanation. Review the work on its evidence.

## Fix the roles

Use this arrangement for each work unit:

1. **One implementer** produces the artifact against the source of truth and shared guide. The implementer does not review.
2. **Two or more adversarial reviewers** independently enumerate bugs and reasons the artifact fails. Reviewers do not implement.
3. **One fixer** applies the findings in a separate correction step.

Keep these boundaries when the work changes from initial production to repairs. Do not collapse review and correction into an author's self-check.

## Give reviewers an adversarial brief

Use this brief, not "check for issues":

> Assume this is wrong and enumerate how it breaks. Compare the artifact with prior behavior and the shared guide. Find discrepancies, missing behavior, and reasons it will fail even if it compiles and its tests pass.

Require findings to identify the failing behavior or violated decision. An artifact can be internally consistent and still contradict its source of truth.

Have the fixer address the findings in the artifact. Do not count an explanation of why a discrepancy is convenient as a correction.

## Reject convenient substitutes for correctness

- **Reject stubs that silence the compiler.** Restore the required behavior instead of removing the compiler's reason to complain.
- **Reject paragraph-long defenses of workarounds.** If the workaround needs that defense, fix the code.
- **Reject green status as proof.** Check what actually ran and compare the artifact against its required behavior.

## Repair the generating process

When a class of mistake recurs, change the prompt or workflow that produces or accepts it. Make the review brief reject that failure class explicitly, and apply corrections to the affected work.

Inspect subsequent outputs to see whether the change works. Do not spend the run hand-fixing the same mistake while leaving its generator intact.

## Keep the human sampling

Keep diffs, review findings, and execution results available to the supervising human. Have them:

- Read some diffs against the original source or required behavior.
- Confirm reviewers actually catch discrepancies and enforce the shared guide.
- Confirm tests executed and were not skipped behind a green result.

Use what the human finds to edit the loop. Review volume alone does not establish reviewer effectiveness.

## Common mistakes

| Mistake | Better |
| --- | --- |
| Asking the author to review its own output | Use separate reviewers with separate contexts |
| Giving reviewers the author's rationale | Give the artifact and source of truth without the author's reasoning |
| Asking reviewers to "check for issues" | Tell them to assume the work is wrong and enumerate how it breaks |
| Checking only internal consistency | Compare against prior behavior and the shared guide |
| Having reviewers implement their findings | Assign one fixer to the separate correction step |
| Accepting stubs, elaborate defenses, or green status | Require preserved behavior and actual execution evidence |
| Repeatedly patching one generated mistake | Repair the prompt or process and inspect subsequent output |
| Treating automated review as self-validating | Have the human sample diffs, findings, and executed checks |
