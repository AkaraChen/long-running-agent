# Long Running Agent

A plugin of skills for long-running agent work: writing the completion contract
before a long run, then running the loop that carries it to a verified finish.

## The skills

- [write-goal](skills/write-goal/SKILL.md): help the user craft a well-specified
  `/goal` objective for goal mode — what must become true, how that truth is
  proven, where the work may and may not reach, and when to stop and report
  instead of grinding on. It works with the user to draft and refine the
  objective, then starts it once the wording is approved.
- [run-long-task](skills/run-long-task/SKILL.md): the complete loop for a large
  agent effort — frame the contract, choose the work unit and the gates, write
  the shared decisions once, pilot the loop, bound concurrency by
  infrastructure, keep authoring and review independent, save failures as scoped
  work queues, verify the verification, and separate integration from release.
- [adversarial-loops](skills/adversarial-loops/SKILL.md): the review loop that
  catches what compiling and passing tests do not — separate contexts, one
  implementer plus two or more adversarial reviewers plus a fixer, and repairing
  the process that keeps producing the same mistake.
- [orchestrate-migration](skills/orchestrate-migration/SKILL.md): the
  migration-shaped cut — shared mapping and per-item decision records, a pilot,
  mechanical translation, the compiler as a work queue, and ordered executable
  gates.

`write-goal` produces the contract; `run-long-task` runs it. `adversarial-loops`
and `orchestrate-migration` are focused cuts of the same loop, for review work
and for migrations.

## Methodology sources

`references/` holds methodology write-ups that skills in this repo draw on.
They are research notes, not skills, and are not part of the plugin surface.

- [references/bun-in-rust-methodology.md](references/bun-in-rust-methodology.md):
  how Bun was ported from Zig to Rust in eleven days with supervised agent
  workflows — the decision criteria, the prep artifacts, the orchestration loop,
  the failure modes that forced process changes, and the verification strategy.
  Extracted from Jarred Sumner's
  [“Rewriting Bun in Rust”](https://bun.com/blog/bun-in-rust).
  `run-long-task`, `adversarial-loops` and `orchestrate-migration` are all
  extracted from it.

## Compatibility

The `write-goal` skill text refers to Kimi Code's `/goal`, `AskUserQuestion`, and
`CreateGoal` capabilities. Other agent hosts may require equivalent tools or
adaptations to start goals. This copy preserves the upstream behavior and does
not add cross-host integration. The other three skills are host-agnostic.

## Install as a plugin

This repository is both the skill source and a single plugin named
`long-running-agent`. The skills live at `skills/<name>/SKILL.md`.

### Claude Code

```sh
claude plugin marketplace add AkaraChen/long-running-agent
claude plugin install long-running-agent@long-running-agent
```

Local check without installing:

```sh
claude --plugin-dir .
```

Then invoke a skill as `/long-running-agent:write-goal` or
`/long-running-agent:run-long-task`. `claude plugin validate .` checks the
marketplace catalog.

### Codex

```sh
codex plugin marketplace add AkaraChen/long-running-agent
codex plugin add long-running-agent@long-running-agent
```

A checkout of this repo also exposes the plugin through
`.agents/plugins/marketplace.json`. Restart Codex after adding the marketplace,
then enable `long-running-agent`. Invoke a skill as `$write-goal` or
`$run-long-task`.

## Source / Attribution

The `write-goal` skill is copied from the **Kimi Code open-source repository** by
**Moonshot AI**.

- Upstream repository: https://github.com/MoonshotAI/kimi-code
- Upstream commit: `f12d59e089e2531a33fbca30b26ffeabd5862b45`
- Original file: [packages/agent-core-v2/src/features/skill/catalog/builtin/write-goal.md](https://github.com/MoonshotAI/kimi-code/blob/f12d59e089e2531a33fbca30b26ffeabd5862b45/packages/agent-core-v2/src/features/skill/catalog/builtin/write-goal.md)
- Copied on: 2026-09-07
- License: [MIT](LICENSE), Copyright (c) 2026 Moonshot AI.

The original skill text is preserved unchanged and packaged as
`skills/write-goal/SKILL.md`. The upstream TypeScript registration wrapper is
not included because it belongs to Kimi Code's runtime.

`run-long-task`, `adversarial-loops` and `orchestrate-migration` were extracted
from the methodology document in `references/`, which cites the source post.
