---
name: orchestrate-migration
description: Organizes large migrations around shared decisions, mechanical translation, and ordered executable gates. Use when a language, framework, or architecture migration is too large to perform file by file by hand.
---

# Orchestrate a migration (orchestrate-migration)

Keep the result recognizable to current maintainers. Organize the port so each translated item can be compared with its source, then prove that the assembled result works.

## Decide the shape and preservation goal

1. **Choose all-at-once or incremental before translating.** Count the temporary code and short-term complexity incremental migration creates, including code you only hope to delete later. All-at-once still needs multiple work phases.
2. **State what must survive.** Preserve architecture, features, performance, and behavior with a translation-shaped port. Minimize behavioral changes and defer extensive idiomatic redesign.
3. **Do not freeze the product for the migration.** Include displaced bugfixes, security work, and features in the feasibility decision; do not treat their suspension as a free resource.

## Write and review the shared decisions

Before scaling, produce two reusable artifacts:

- **A pattern/type mapping guide.** Record how source constructs map to target constructs. Make translators follow it and reviewers check it.
- **A per-item decision record for hard cases.** Trace actual control flow before deciding ownership, lifetimes, or concurrent and asynchronous retention. Record which item each decision governs.

Have two independent adversarial reviewers challenge the hard-case proposals. Review both artifacts, then review them together for conflicting instructions. Apply corrections and have the supervising human read them.

## Pilot, then translate mechanically

Run the complete production loop on a tiny sample before distributing the migration:

1. Have one implementer translate an item against the source and both guides.
2. Have two or more reviewers independently check behavioral correspondence and guide compliance, without the implementer's rationale.
3. Have one separate fixer apply their findings.

Use the pilot to repair the process before scaling. Then repeat it item by item, keeping diffs translation-shaped and reviewable. Do not treat generated, reviewed files as working software.

## Turn the compiler into a work queue

**Fix dependency structure first.** Classify where cyclically dependent code belongs, write down the plan, then perform the structural repair before distributing compiler errors.

For each package, run the compiler at the start of its repair workflow. Group diagnostics by file and save them. Have an implementer repair that scoped queue, independent reviewers challenge the changes, and a fixer apply feedback.

Keep compiler execution at controlled points and commits at the end of work units. Do not let translation workers independently run builds or manipulate shared repository state. Reject stubs and elaborate workaround defenses that trade away behavior for compilation.

## Advance through executable gates

Pass these gates in order; passing one proves nothing about the next:

1. **Types** — finish compiler repair.
2. **Link** — build the executable and repair linkage failures.
3. **Start** — run it and repair startup failures.
4. **One test** — prove the test runner can execute a test.
5. **The suite** — use bounded local batches to repair failures, then pass the full behavioral suite.
6. **The CI matrix** — complete the full suite on every supported platform; verify tests actually ran without skips.

Save runtime failures with their command and test failures with their test file or platform. Feed those queues through the same implement–review–fix loop.

## Audit semantics, not resemblance

Give reviewers these concrete failure prompts:

- **Build-mode-dependent evaluation:** does a release build erase an assertion's required side effect?
- **Casts and edge cases:** does a conversion now panic where the source truncated or accepted an odd-length input?
- **Capacity and bounds:** did a placeholder change a sizing relationship, or did new bounds checks expose an inherited off-by-one?
- **Evaluation order:** does a transformation now run after argument insertion and alter data it previously left alone?
- **Object lifetime across asynchronous boundaries:** does scope exit free an object that a callback still retains or will free again?

Compare these behaviors with the source and repair discrepancies even when the syntax looks equivalent.

## Separate merging from releasing

Base integration on completed verification and human sampling of code, review effectiveness, and test execution. Make release a separate decision; continue regression repair and hardening after integration.

## Common mistakes

| Mistake | Better |
| --- | --- |
| Choosing incremental migration without counting temporary code | Decide the shape and account for its transitional complexity |
| Combining a port with wholesale redesign or a product freeze | Preserve recognizable behavior and include displaced product work in feasibility |
| Letting every translator invent mappings | Review shared patterns and per-item decisions together |
| Scaling after a generation-only demo | Pilot implementation, independent review, and correction |
| Chasing compiler errors before dependency repair | Resolve structure, then consume saved diagnostics package by package |
| Calling type-check success a working migration | Pass link, startup, test, suite, and full CI gates separately |
| Reviewing syntax alone | Audit build modes, casts, bounds, evaluation order, and async lifetimes |
| Treating a merge as a release | Continue hardening and make the release decision separately |
