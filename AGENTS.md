# Agent Notes

- Keep skill sources in `skills/` at `skills/<name>/SKILL.md`. Do not add grouping folders under `skills/`.
- `write-goal` is a faithful copy of the upstream Kimi Code skill; do not rewrite the skill body to add cross-host integration. Compatibility notes belong in `README.md`.
- Keep the upstream `LICENSE` (MIT, Copyright (c) 2026 Moonshot AI) intact and keep the attribution in `README.md`.
- Plugin manifests live at the repo root: `.claude-plugin/` for Claude Code, `plugin.json` plus `.codex-plugin/` for Codex / Agent Plugins, and `.agents/plugins/marketplace.json` for the Codex marketplace catalog. Do not nest `skills/` inside those manifest directories.
- Keep manifest fields consistent across `plugin.json`, `.claude-plugin/`, `.codex-plugin/`, and `.agents/plugins/`: `name: write-goal`, version `0.1.0`, author `AkaraChen`, homepage/repository `https://github.com/AkaraChen/write-goal-skill`.
- Do not add a root `docs/` directory or `install.sh`.