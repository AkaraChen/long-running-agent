# Bun’s Zig → Rust rewrite: methodology

Source: Jarred Sumner, [“Rewriting Bun in Rust,” July 8, 2026](https://bun.com/blog/bun-in-rust). Extracted 2026-09-12 from the published post; the post itself is not redistributed here. This document extracts procedures, decision criteria, and failure lessons. Section 11 is an imperative synthesis of those observations, not an additional workflow specification published by Bun. Dates and release states are those reported in the post.

## 1. The decision: prevent recurring failures at an acceptable migration cost

**The problem was recurring memory-lifetime failure despite substantial testing.** Bun combined JavaScriptCore’s garbage-collected values with manually managed native memory. Correctness required tracking ownership, exactly-once cleanup, JavaScript exceptions, and whether GC pointers remained visible to the stack scanner. Re-entrant JavaScript callbacks could invalidate pointers or buffers; asynchronous native operations could outlive the scope that initiated them.

Bun already ran its test suite with ASAN on every commit, using a Zig compiler patch; shipped safety-checked `ReleaseSafe` builds on Windows; continuously fuzzed runtime APIs with Fuzzilli; and maintained extensive end-to-end memory-leak tests. Sumner wanted to move enforcement earlier than execution, CI, and post-merge fuzzing.

| Alternative | What it offered | Why it was not chosen |
| --- | --- | --- |
| Keep fixing individual bugs | Avoid migration and retain the existing implementation | Leave the recurring failure mechanism in place indefinitely |
| Stay in Zig with ownership conventions and smart pointers | Make ownership expectations more explicit without changing languages | Still depend on review and conventions; worse ergonomics than Rust without its guarantees. This was the actual fallback plan, and smart-pointer work had begun |
| Rewrite the Zig portion in C++ | Constructors/destructors and fewer C wrappers around existing C++ dependencies | Still rely on style-guide enforcement through review, with memory corruption and leaks remaining possible |
| Rewrite manually in Rust | Ownership checking and automatic cleanup | Sumner estimated a small team of engineers with full context on the codebase would need about a year, displacing bugfixes, security work, compatibility improvements, and features |

**Decision criteria:** prevent important failure classes; obtain earlier feedback; preserve behavior and architecture; retain maintainability; and avoid an unacceptable development freeze. Rust’s ownership checks and `Drop` addressed important lifetime mistakes. A faithful mechanical port and a language-independent test suite offered a way to constrain behavioral change.

**The trigger was a bounded experiment.** Sumner proposed spending a week testing whether Anthropic’s pre-release Claude Fable 5 could perform the rewrite. A few days in, substantial test progress and close correspondence between Rust and Zig changed his assessment from an experiment to something he intended to merge.

The transferable decision method is conditional: identify a persistent failure class, evaluate enforcement alternatives, then test whether migration cost has changed enough to make the preferred remedy feasible. The post does not argue that every Zig project needs Rust or that mixed Rust/C/C++ code becomes free of memory bugs.

Source sections: “What we were already doing,” “Just be really smart and don’t make mistakes?,” “What about C/C++?,” “Why Rust?,” and “Stats.”

## 2. The two big questions: migration shape and preservation strategy

**Question one: “Incremental rewrite? Or, everything all at once?”**

Sumner chose all-at-once, drawing on his previous esbuild Go-to-Zig port. His objection to incremental migration was the temporary code it introduces: short- and medium-term complexity that must eventually be removed.

All-at-once describes the implementation transition, not a single generation step. The execution still had preparation, a pilot, translation, compiler repair, runtime repair, tests, and CI. The post describes a port branch and a later merge into `main`; it does not describe gradually deploying Zig and Rust subsystems inside a hybrid executable.

**Question two: how to preserve architecture, performance, features, and maintainability while gaining Rust’s language features?**

The answer was a translation-shaped port with minimal behavioral change. Keep the source recognizable to maintainers, then progressively reduce `unsafe` and move toward idiomatic Rust. The team should be able to understand the new implementation using its knowledge of the old one.

The post places further idiomatic refactoring after v1.4 ships, but also reports pre-merge cleanup, deduplication, and some unsafe reduction. The supported principle is to prioritize fidelity and defer extensive redesign, not prohibit every refactor. Splitting the code into crates itself required structural changes.

Source sections: “Claude, rewrite Bun in Rust.,” “Get the test suite passing in CI,” and “Maintainability.”

## 3. Prep artifacts: make translation and lifetime decisions reusable

### `PORTING.md`: shared pattern and type mappings

Before generation, Sumner spent about three hours discussing how existing Zig patterns should map closely to Rust. Claude serialized that discussion into `PORTING.md`.

The guide mapped Zig patterns and types to Rust patterns and types. Implementers had to follow it; reviewers checked compliance. Its methodological role was to record decisions once and reuse them across workers rather than letting each file translator invent its own mapping.

The article does not reproduce the complete guide, its template, or a comprehensive construct-by-construct translation table.

### `LIFETIMES.tsv`: field-level analysis of actual control flow

Sumner described this workflow:

1. Read struct fields throughout every source file.
2. Trace control flow to understand their lifetimes.
3. Identify fields whose lifetimes are difficult to express in Rust.
4. Propose the appropriate lifetime.
5. Have two adversarial reviewers examine that proposal.
6. Apply feedback and serialize the results into `LIFETIMES.tsv` for other agents.

This was analysis of the actual program, not a blanket rule replacing every pointer with a reference. The post does not disclose TSV columns, representative rows, or detailed borrowing solutions.

### Review both artifacts together

A further adversarial review examined `PORTING.md` and `LIFETIMES.tsv` together to identify conflicting recommendations and recheck correctness. Sumner also read them manually. This prevented general translation instructions and field-specific lifetime instructions from silently disagreeing.

### Technical choices the article supports

| Area | Documented technique | Limit of the evidence |
| --- | --- | --- |
| Translation | Claude mechanically ported `.zig` files to `.rs` files against the guides | No deterministic transpiler implementation or complete code-generation pipeline is described |
| Ownership and cleanup | Use Rust ownership machinery and `Drop`; the article credits `Drop` with fixing file-path cleanup leaks on error paths | No complete borrowing or ownership-wrapper catalog |
| Arenas and reference counting | Zig used clear arena lifetimes for suitable parser/AST state and reference counting elsewhere | No Rust allocator, arena replacement, or `Rc`/`Arc` policy is specified |
| Native interop | Retain C/C++ libraries, including JavaScriptCore; many remaining unsafe operations involve native pointers or library calls | No full FFI ownership contract, ABI migration, or binding-generation procedure |
| Async lifetime | A libuv handle must remain allocated until its asynchronous close callback frees it | No general event-loop redesign or executor choice |
| Build organization | Split one Zig compilation unit into roughly 100 Rust crates for faster compilation; repair cycles explicitly | No complete build graph or toolchain configuration |
| Link-time optimization | Rust/C/C++ cross-language LTO enables inlining across language boundaries | No reproduction configuration |
| Errors | Automatic cleanup reduces omitted or duplicate cleanup on error paths | No systematic mapping for Zig error sets or JavaScript exception propagation |

Source sections: “Prep work,” “Loops that write & review code,” “Compiler errors as a work queue,” “Adversarial review,” “The work continues,” “Reduced memory usage,” and the benchmarking section.

## 4. The trial run: exercise the whole process before scaling

Before translating all 1,448 Zig files, Sumner started with three. Each went through the intended production workflow:

1. One implementer produced the Rust file.
2. Two adversarial reviewers checked behavioral correspondence with Zig and adherence to both guides.
3. One fixer applied the reviewers’ suggestions.

The stated rationale was to de-risk expensive work first. The trial exercised implementation, review, and correction together, not just the ability to generate plausible Rust.

The article does not name the files, explain their selection, or provide a numerical acceptance threshold. It also does not say the pilot was a compiling, integrated vertical slice. Those would be additional recommendations, not observations about Bun.

Source sections: “What does this look like?” and “Trial run.”

## 5. Orchestration: bounded work units, independent review, human supervision

### The recurring loop

The common process was: select a task, implement it, obtain independent adversarial reviews, apply findings, and commit at the permitted boundary. The unit of work changed as the system became functional:

| Phase | Work unit | Loop behavior |
| --- | --- | --- |
| Preparation | Pattern/type mappings and field lifetimes | Produce and review shared guidance |
| Translation | Individual Zig files | Produce Rust, check against the guides and original behavior, fix before commit |
| Dependency repair | Code involved in cycles | Classify where it belongs, record that plan, then refactor |
| Compiler repair | Crate, with diagnostics grouped by file | Consume saved errors, repair code, review, apply corrections |
| CLI bring-up | Subcommand and failing stacktrace | Save failures, group by subcommand, fix, review, apply |
| Local testing | Test files partitioned by folder | Run about 100 random test files, save failures, process fixes |
| CI convergence | Platform-specific failures | Repeat repairs until each platform’s full suite passes |
| Cleanup | Targeted refactors | Address Windows code, duplication, unsafe usage, and other cleanup |

The article reports about 50 dynamic workflows across the effort. It does not enumerate all 50 or say that 50 ran concurrently.

### Keep implementation and review independent

The general rule was one implementer with two or more adversarial reviewers. Representative loops used two reviewers and one fixer. Review happened in separate context windows, with the objective of finding bugs and reasons the implementation would fail.

The role boundary is explicit: “The implementer doesn't review. The reviewer doesn't implement.”

The review demonstration gives reviewers the diff without the implementer’s reasoning. The trial also requires checking Zig/Rust correspondence and compliance with the guides. These descriptions establish independent assessment, but do not specify exactly how every workflow assembled reviewers’ context.

### Parallelization and effort

| Measure | Reported value and meaning |
| --- | --- |
| Original Zig source | 535,496 lines excluding comments |
| Translation scope | 1,448 files; three in the pilot |
| Main effort | Eleven days, May 3 to merge on May 14 |
| Workflows over the effort | About 50 |
| Peak partitioning | Four workflow shards, each with a worktree |
| Agents | Sixteen Claudes per shard; about 64 overall |
| Compiler-phase arrangement | Sixteen loops across four worktrees, each with one implementation agent, two reviewers, and one feedback applier |
| Peak generation | About 1,300 lines per minute; not a measure of working software |
| Commits | 6,778 in the stats summary; 6,502 in visualizations explicitly excluding merges |
| Peak commit rates | 695 in an hour; a separate visualization reports 58 in one minute |
| Landed additions | +1,009,272 lines; cumulative rewrites in the visualization are a different measure |
| Compiler backlog | Approximately 16,000 errors; the phase-D replay contains 1,610 commits |
| Pre-merge model usage | 5.9 billion uncached input tokens, 690 million output tokens, 72 billion cached input-token reads |
| Model cost estimate | Approximately $165,000 at API pricing; not an all-in labor and infrastructure budget |

The post identifies the visualization’s merges-excluded counting, but does not independently reconcile every total. These quantities describe the run; they are not recommended targets for another project.

### The human’s work

Sumner describes one engineer driving the effort with Claude Fable 5 and Claude Code’s dynamic workflows. He monitored outputs, identified mistakes, and asked Claude to modify faulty loops. He read the prep artifacts, compared substantial Rust code with Zig, checked reviewer effectiveness and guide compliance, verified actual test execution, and made the merge decision.

The post also describes later team optimizations and human review of fuzz-generated PRs. It does not provide a developer-by-developer allocation, a Rust training plan, or a formal maintenance handoff. The eleven-day result is not evidence that the rewrite required no subsequent human work.

Source sections: “Loops that write & review code,” “Adversarial review,” “Finally writing the code,” “Stats,” “Maintainability,” and “What’s next.”

## 6. Failure modes: repair the process that produces mistakes

### Shared Git state caused worker interference

About two minutes into the full-file workflow, one agent ran `git stash`, another ran `git stash pop`, and then `git reset HEAD --hard` appeared. They were manipulating shared state and stepping on each other.

Sumner changed workflow instructions to prohibit stash/reset and constrain Git activity to specific-file commits. He also prohibited Cargo and slow commands during translation. Later prose describes committing and pushing files, so the account is not a complete literal command allowlist.

**Derived rule:** scope workers’ mutations to assigned output and place shared repository/build operations at controlled phase boundaries. The article does not publish a locking implementation.

### One worktree per agent exceeded the disk budget

Separate worktrees for every agent would consume too much space, and the changes eventually needed to compile together. The compromise was four worktrees with sixteen Claudes each. The article does not explain file claiming, locking, synchronization, or conflict resolution inside this arrangement.

### Disk IOPS constrained concurrency

The EC2 instance’s default IOPS had not been increased. A slow `grep` could stall reads and writes for minutes. The article identifies the bottleneck but supplies no final provisioning setting.

**Derived rule:** treat storage throughput and capacity as limits on agent concurrency, rather than sizing only for available model calls.

### Compiler success incentivized removing behavior

Claude interpreted the request to get crates compiling as permission to stub functions with compiler errors. It also wrote long comments defending workarounds. Sumner changed review instructions to reject those justifications and require fixing the code. His rule ends: “the code is wrong — fix the code.”

He reports that a prompt edit stopped these patterns after a few hours. The intervention changed the generating/reviewing process rather than only hand-patching outputs. The later placeholder regression still shows that this was not proof every incomplete translation had been removed.

### Checking types was not enough to run the program

After `cargo check` passed, building and running `bun --version` exposed linker errors and then an immediate startup panic. Further work was required before `bun test <file>` ran.

Type-checking, linking, startup, test-runner operation, and behavioral verification were separate gates. Passing one did not establish the next.

### Stress tests needed operating-system isolation

Tests could exhaust TCP sockets, read and write gigabytes, or spawn roughly 10,000 processes. Some integration and leak tests timed out in debug builds. Bun used `systemd-run` with cgroups to limit memory/CPU and isolate PID namespaces.

The machine still repeatedly ran out of disk and crashed. Memory/CPU/process isolation did not solve every resource problem. Exact limits and the resolution of every debug-timeout case are not supplied.

### Similar-looking code preserved syntax but changed semantics

The article reports nineteen known regressions, all fixed by publication, with most arising from source/target semantic differences. Its examples supply concrete review targets:

| Failure | Mismatch | Lesson from the reported fix |
| --- | --- | --- |
| Async libuv close | A `Box<uv::Pipe>` dropped at the end of a match arm while libuv retained its pointer; the callback would free it again | Keep the allocation alive until callback cleanup. The illustrated fix uses `Box::leak(pipe).close(...)`; it transfers cleanup responsibility, not a general license to leak |
| Assertion side effects | Zig evaluated the assertion argument in every build; Rust’s `debug_assert!` erased its side-effecting expression in release builds | Audit build-mode-dependent evaluation; required mutations must survive release compilation |
| Odd-length byte slices | The Zig helper ignored an unmatched final byte; `bytemuck::cast_slice` panicked | Preserve edge-case behavior. Bun restored truncation of the unmatched byte |
| Bounds and capacity | A placeholder reduced filename-interning capacity and exposed an inherited off-by-one; Rust bounds checks panicked | Preserve sizing relationships, remove placeholders, and investigate both changed limits and inherited defects |
| Compile-time formatting | Zig transformed markers before inserting arguments; the Rust function transformed the completed string, including arguments | Preserve transformation order. Bun moved the operation into a macro |

Source sections: “False starts,” “Finally writing the code,” “Another false start,” “Smoke tests,” “Even more false starts,” “Adversarial review,” and “Porting mistakes.”

## 7. Verification: reuse behavior tests and add independent evidence

### Existing tests were the behavioral baseline

Bun’s tests were written in TypeScript, so changing the runtime’s implementation language did not require replacing its external behavioral suite. That suite was the main executable basis for retaining behavior, supplemented by source comparison and adversarial review.

The local loop ran about 100 random test files, partitioned across four worktrees by folder. Failures yielded saved stacktraces and errors for the implementer/reviewers/fixer cycle. This was a way to converge on working behavior, not final acceptance based on a sample.

CI exercised six platform combinations:

| Platform | Reported test shards |
| --- | --- |
| Linux x64 | 60 |
| Linux arm64 | 60 |
| macOS x64 | 2 |
| macOS arm64 | 4 |
| Windows x64 | 8 |
| Windows arm64 | 8 |

Two days after the first CI run, failing files had decreased from 972 to 23. Linux went fully green another day and a half later. Repairs continued per platform. The visualization distinguishes completed passing runs from superseded partial runs and identifies #54202 as the final all-green build.

The three-platform inventory lists 57,337–60,624 tests and more than a million `expect()` calls per platform. The article reports zero tests skipped or deleted. This does not establish that every test file and expectation remained byte-for-byte unchanged.

### Distinguish existing checks from later hardening

| Mechanism | Timing and role described |
| --- | --- |
| ASAN | Existing Zig-era test execution on every commit, enabled by a compiler patch |
| Zig `ReleaseSafe` | Existing Windows safety checking; macOS/Linux used `ReleaseFast` |
| Fuzzilli | Existing continuous runtime-API fuzzing |
| Memory-leak tests | Existing end-to-end coverage, including long-running cases |
| Coverage-guided parser fuzzing | Added after merge, running continuously across Bun’s parsers |
| Fuzz-to-PR workflow | Fuzz failures go to Claude for a reproducer and fix PR; humans review PRs |
| Security review | Eleven post-merge Claude Code Security rounds, with findings addressed |
| LeakSanitizer | Improved integration to track all native allocations; repeated in-process builds demonstrate detection of retained state |
| Miri | Covers a growing portion of code in CI at publication; not described as a whole-codebase pre-merge gate |

This is layered verification, not formal equivalence. The article does not document a differential-testing harness that feeds identical generated inputs to Zig and Rust and compares results.

### Benchmark continuity separately

The post compares Bun v1.3.14 and v1.4.0 on Linux x64 using an EC2 machine with a Xeon Platinum 8488C. It uses `oha` for HTTP hello-world servers and `hyperfine` for application/CLI workloads. HTTP results average three rounds; application workloads include Next.js builds, Vite builds, and TypeScript compilation.

The methodological point is to evaluate performance continuity separately from functional correctness. Exact commands, a full noise-control protocol, automated regression thresholds, and a benchmark-based merge gate are not documented.

### Manually verify the verification

Sumner explicitly describes:

1. Reading the preparation artifacts.
2. Monitoring outputs and changing faulty workflows.
3. Checking that reviewers found Zig/Rust discrepancies and enforced both guides.
4. Comparing substantial portions of source side-by-side.
5. Confirming tests actually ran and were not skipped, rather than trusting green status alone.
6. Running additional local commands before merge; those commands are not enumerated.
7. Human review of later fuzz-generated PRs.

The production section adds downstream evidence: Prisma tested the rewrite against previously observed leaks and connection-pool failures after VM pause/resume. That is real-world validation reported by a user, not a published mandatory Bun release checklist.

Source sections: “What we were already doing,” the local/CI testing sections, “Merging the Rust rewrite,” “Stats,” “The work continues,” “We fixed every instrumentable memory leak,” the benchmarking section, “Production,” and “What’s next.”

## 8. Compiler errors as a work queue: save diagnostics, then distribute repairs

**First repair the dependency structure.** The Zig project was effectively one compilation unit. Sumner wanted approximately 100 Rust crates for faster compilation while minimizing departures from the original architecture. A preparatory dependency refactor was insufficient. Instead of restarting, one workflow classified where cyclically dependent code should live and recorded that plan; another performed the refactor.

Resolving the cycles exposed about 16,000 compiler errors. The repair loop was:

1. Run `cargo check` at the beginning of the crate’s repair workflow.
2. Group its diagnostics by file and save them to a file.
3. Have an implementation agent repair the crate’s errors.
4. Have two adversarial reviewers examine the changes.
5. Have a fixer apply their feedback.
6. Commit at the end of the work unit.

The article explicitly places `cargo check` at the start and Git activity at the end to avoid interference. This is phase-specific: the earlier translation phase prohibited Cargo, while compiler repair introduced controlled compiler execution.

The phase-D replay describes sixteen loops across four worktrees and 64 Claudes, with commits landing per crate. Architectural decisions, diagnostics, and review findings became recorded inputs to subsequent work rather than remaining transient conversations.

Diagnostic de-duplication, stale-error handling, task claiming, and exact inter-batch re-check scheduling are not specified. The documented mechanics are to run the shared diagnostic operation at a controlled point, save its output, and distribute scoped repairs rather than let every worker independently drive build and repository state.

Source section: “Compiler errors as a work queue.”

## 9. Merge/release strategy: separate integration confidence from release confidence

The merge threshold was the full suite passing on every supported CI platform, manual confirmation that tests ran rather than being skipped, and additional local checks. The branch merged on May 14 after eleven days.

The distinction is explicit: “Merging into `main` isn't a versioned release.” Sumner was ready to commit to maintaining Rust before he was ready to release it.

Post-merge work included security review, continuous parser fuzzing, regression fixes, and cleanup. At publication, v1.3.14 was the last Zig release, and v1.4.0 was identified as the first Rust release, available in canary. The post also reports use by Prisma Compute and by Claude Code from its June 17 release onward.

This supports a progression from integration to hardening and real-world use before general release. It does not establish a formal canary promotion policy.

**Shipping and coexistence are gaps.** The post does not explain how bugfixes or security patches shipped on the Zig line during the eleven days, how branches were synchronized, or whether fixes were dual-implemented. The rejection of a year-long freeze is not evidence of a documented uninterrupted-shipping mechanism. An all-at-once port plus separate release boundary also does not describe runtime coexistence of Zig and Rust implementations.

Source sections: “Merging the Rust rewrite,” “Stats,” “The work continues,” “Production,” and “Shipping.”

## 10. Explicit do-nots: prevent convenient proxies from replacing correctness

These are choices, prohibitions, or corrections stated in the post:

- **Do not rely on one broad rewrite prompt and hope.** Use preparation, bounded work, review, feedback, and runtime checks.
- **Do not let implementers review themselves.** Keep adversarial review roles and contexts separate.
- **Do not start with all files before exercising the process.** Bun first tested it on three.
- **Do not allow stash/reset to manipulate shared worker state.** Constrain repository mutations to assigned files and phase boundaries.
- **Do not run Cargo or slow commands inside translation workers.** That phase separated generation from integration; compiler repair later used controlled checks.
- **Do not satisfy compiler errors by stubbing required behavior.** Review must preserve the implementation’s contract.
- **Do not accept a workaround because an agent wrote an elaborate defense.** Fix the code.
- **Do not treat generation and review as proof of execution.** Of the initially translated output, Sumner says: “Absolutely none of it worked yet.”
- **Do not accept green CI without verifying actual test execution.** Sumner manually checked that tests ran and were not skipped.
- **Do not equate merge readiness with release readiness.** They were separate decisions.
- **Do not combine the port with wholesale idiomatic redesign.** Prioritize recognizable translation, with further refactoring afterward.

The reported bugs additionally support these derived review rules: do not hide required side effects in build-mode-dependent assertions; do not assume casts have matching edge cases; do not leave capacity placeholders; and do not let lexical scope prematurely terminate native async ownership. These are extracted lessons, not a separately published Bun rules file.

Source sections: “Claude, rewrite Bun in Rust.,” “Adversarial review,” “Trial run,” “False starts,” “Another false start,” “Finally writing the code,” “Merging the Rust rewrite,” and “Porting mistakes.”

## 11. Generalized playbook: imperative steps derived from the account

The steps below synthesize the preceding evidence. Their outputs make the method actionable; they are not claims about unpublished Bun tooling. Bun’s worker, worktree, and crate counts are observations, not defaults.

1. **Identify the failure class the rewrite should prevent.** Compare compiler enforcement and automatic cleanup with current review and runtime checks. Explain why continued patching or stronger conventions are insufficient. Output: a concrete decision rationale.

2. **Test whether migration has become affordable.** Include displaced fixes, security work, and features in the cost. Start with a bounded feasibility experiment. Output: evidence of translation fidelity and behavioral progress before committing to the entire migration.

3. **Choose migration shape and preservation goals explicitly.** Decide incremental versus all-at-once and account for temporary infrastructure. If following Bun, prioritize recognizable architecture and minimal behavior change over immediate redesign. Output: a shared scope for the port.

4. **Write the mapping guide before distributing translation.** Record source patterns/types and their intended target-language forms, then review them. Output: a `PORTING.md` equivalent.

5. **Analyze lifetime obligations from real control flow.** Inspect stored fields, escaping references, and asynchronous retention. Propose difficult mappings, have two independent reviewers challenge them, and record corrected results. Output: a `LIFETIMES.tsv` equivalent.

6. **Review both artifacts together and read them yourself.** Resolve conflicts between general patterns and field-specific decisions. Output: mutually consistent instructions.

7. **Pilot the complete implement–review–fix process.** Translate a small sample and check source correspondence and guide compliance. Bun used three files; the article gives no universal sample size. Output: evidence that the workflow itself produces reviewed translations.

8. **Partition work within infrastructure limits.** Scope file mutations, constrain Git commands, and use a manageable number of worktrees. Account for disk capacity and I/O throughput before increasing concurrency. Output: bounded assignments and operational restrictions.

9. **Preserve independent adversarial review in every phase.** Ask reviewers to find incorrect behavior without the implementer’s persuasive rationale. Use two or more reviewers and a separate feedback-application step, following Bun’s pattern. Monitor reviewer effectiveness. Output: corrected work units.

10. **Repair dependency organization deliberately.** Classify where cyclically dependent code belongs, record decisions, then refactor. Output: a target structure that permits meaningful compiler repair.

11. **Turn diagnostics into a saved, scoped backlog.** Run the compiler at a controlled point, group errors by file within crates, repair, review, and commit at defined boundaries. Reject stubs and justified workarounds. Output: compiling code whose required behavior has not been discarded.

12. **Advance through executable gates in order.** Fix linkage, startup, and enough command behavior to run tests. Save stacktraces with subcommands and apply the same review cycle. Output: an executable capable of exercising the existing suite.

13. **Converge locally, then across the full platform matrix.** Use bounded test batches for repair, OS isolation for stress loads, and complete CI runs for acceptance. Do not substitute passing subsets or superseded runs. Output: full-suite evidence for supported platforms.

14. **Audit semantic differences deliberately.** Examine build-mode evaluation, casts, capacities, bounds checks, formatting order, and native async ownership. Treat Bun’s failures as prompts, not an exhaustive checklist. Output: identified source/target discrepancies repaired.

15. **Verify the verification yourself.** Confirm tests executed without skips; inspect source correspondence and reviewer behavior; run additional commands. Check old/new performance separately from functional correctness. Output: a human-reviewed basis for merging.

16. **Separate merge from release and continue hardening.** Investigate regressions, run security review and coverage-guided fuzzing, review generated fix PRs, and expand applicable sanitizer/Miri coverage. Refactor toward idiomatic code without assuming native unsafe boundaries disappear. Output: evidence for a later release decision.

### Gaps: what the post does not supply

- **Complete orchestration:** reusable workflow definitions, full prompts, scheduler implementation, retry policy, agent context lifecycle, and exact stop conditions.
- **Complete prep artifacts:** the full mapping guide, TSV schema, sample lifetime rows, and a comprehensive translation catalog.
- **Pilot selection:** file names, selection rationale, and a quantitative acceptance gate.
- **Worker coordination:** file claiming, locks, branch synchronization, cherry-picking, merge-conflict resolution, and stale-diagnostic handling.
- **Continuous shipping:** patch-forward/backport mechanics, release staffing, or a demonstrated process for maintaining Zig releases during the eleven days.
- **Implementation coexistence:** runtime switching, dual execution, or a temporary Zig/Rust subsystem interop architecture.
- **Differential testing:** an automated old/new input-output comparison harness. Source comparison, reused tests, and comparative benchmarks are documented instead.
- **Rust ownership details:** allocator choice, arena design, `Rc`/`Arc` policy, pinning strategy, aliasing policy, and complete unsafe abstraction contracts.
- **FFI and async architecture:** full ABI/binding-generation plans, event-loop migration, executor selection, cancellation, and comprehensive callback ownership rules.
- **Error translation:** systematic Zig error-set, Rust error-propagation, JavaScript-exception, and panic-boundary mappings.
- **Build reproduction:** complete Cargo/native build configuration, toolchain versions, cross-language LTO commands, and the final crate graph.
- **Infrastructure reproduction:** final EC2 sizing, disk/IOPS settings, cgroup limits, and disk-exhaustion recovery procedures.
- **Verification configuration:** exact sanitizer/fuzzer commands, corpus policy, Miri coverage inventory, complete benchmark protocol, and performance acceptance thresholds.
- **Release operations:** rollback policy, canary promotion criteria, and prescribed downstream qualification.
- **People and handoff:** detailed staffing split, Rust onboarding, and measured long-term maintenance outcomes.
- **Smaller codebases:** no explicit adaptation guidance. The pilot and resource-constrained sharding are documented practices, but the article does not establish suitable agent, crate, worktree, or review counts for smaller projects.
- **Local transcription:** the supplied Markdown preserves the narrative but omits several displayed code blocks and includes only one of the three illustrated adversarial-review bugs. This document does not reconstruct those missing examples or mistake transcription omissions for gaps in the original page.
