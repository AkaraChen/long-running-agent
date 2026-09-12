# Write Goal

A plugin that helps users turn a rough intention into a well-specified goal:
a completion contract with a clear finish line, proof, boundaries, and stop
rule.

It contains a single skill, [write-goal](skills/write-goal/SKILL.md), a
faithful copy of the upstream skill from the **Kimi Code open-source
repository** by **Moonshot AI**.

## The skill

- [write-goal](skills/write-goal/SKILL.md): help the user craft a well-specified
  `/goal` objective for goal mode — what must become true, how that truth is
  proven, where the work may and may not reach, and when to stop and report
  instead of grinding on. It works with the user to draft and refine the
  objective, then starts it once the wording is approved.

## Compatibility

The original skill text refers to Kimi Code's `/goal`, `AskUserQuestion`, and
`CreateGoal` capabilities. Other agent hosts may require equivalent tools or
adaptations to start goals. This copy preserves the upstream behavior and does
not add cross-host integration.

## Install as a plugin

This repository is both the skill source and a single plugin named
`write-goal`. The skill lives at `skills/write-goal/SKILL.md`.

### Claude Code

```sh
claude plugin marketplace add AkaraChen/long-running-agent
claude plugin install write-goal@long-running-agent
```

Local check without installing:

```sh
claude --plugin-dir .
```

Then invoke the skill as `/write-goal:write-goal`. `claude plugin validate .`
checks the marketplace catalog.

### Codex

```sh
codex plugin marketplace add AkaraChen/long-running-agent
codex plugin add write-goal@long-running-agent
```

A checkout of this repo also exposes the plugin through
`.agents/plugins/marketplace.json`. Restart Codex after adding the marketplace,
then enable `write-goal`. Invoke the skill as `$write-goal`.

## Source / Attribution

This skill is copied from the **Kimi Code open-source repository** by
**Moonshot AI**.

- Upstream repository: https://github.com/MoonshotAI/kimi-code
- Upstream commit: `f12d59e089e2531a33fbca30b26ffeabd5862b45`
- Original file: [packages/agent-core-v2/src/features/skill/catalog/builtin/write-goal.md](https://github.com/MoonshotAI/kimi-code/blob/f12d59e089e2531a33fbca30b26ffeabd5862b45/packages/agent-core-v2/src/features/skill/catalog/builtin/write-goal.md)
- Copied on: 2026-09-07
- License: [MIT](LICENSE), Copyright (c) 2026 Moonshot AI.

The original skill text is preserved unchanged and packaged as
`skills/write-goal/SKILL.md`. The upstream TypeScript registration wrapper is
not included because it belongs to Kimi Code's runtime.