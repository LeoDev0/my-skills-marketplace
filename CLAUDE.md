# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) that distributes one plugin (`my-skills`) — a personal collection of Claude Code skills. The repo holds only markdown + JSON manifests; **there is no build, no tests, no lint, no CI**. Output is consumed by Claude Code, not compiled.

## Architecture — why each file exists

The repo wires together three pieces; understanding why each is there keeps changes scoped:

- `.claude-plugin/marketplace.json` — the **marketplace catalog**. Lists the one plugin via a relative path. Written once; **does not change when a skill is added**.
- `my-skills/.claude-plugin/plugin.json` — the **plugin manifest**. Intentionally has **no `version` field**. Per the [docs on version resolution](https://code.claude.com/docs/en/plugin-marketplaces#version-resolution-and-release-channels), omitting `version` on a git-hosted plugin makes every commit count as a new version — that is how this plugin auto-publishes on every push. Adding `version` back pins installs to explicit releases and disables per-commit auto-updates; see the warning at the bottom of `README.md`.
- `my-skills/skills/<name>/SKILL.md` — **each skill**. The `skills/` directory is auto-discovered by the plugin loader, which is why no manifest edits are needed per skill.

The whole-repo design goal is *"add a skill = drop a directory + push."* Anything that would require editing `marketplace.json` or `plugin.json` to add a skill is a regression of that goal.

## SKILL.md house style

All skills here follow the same shape — read any existing one (e.g. `my-skills/skills/claude-github-code-review/SKILL.md`) for the full template before drafting a new one. Key conventions:

- **YAML frontmatter** with `name`, `description`, optionally `license`. The `description:` field is what drives Claude's **auto-invocation** decision:
  - A situational description (e.g. *"Use when writing, reviewing, or refactoring code…"*) invites Claude to auto-trigger the skill when the task matches.
  - A description framed as a specific action with `/slash-name <args>` syntax steers toward slash-only invocation. Use this for skills that take side-effecting actions (e.g. posting to GitHub) — auto-fire is undesirable for those.
- **Numbered Workflow** section with concrete steps.
- **Rules and gotchas** section enumerating constraints, edge cases, and known failure modes — this is where most of the skill's leverage actually lives.
- Tight, opinionated, single-invocation scope. Each skill encodes one repeatable workflow, not general assistance.

Only the frontmatter `name` + `description` are read for the auto-trigger decision; the body is loaded only after the skill is invoked. So changing *when* a skill fires means editing `description`, not the workflow text.

## Adding a new skill

1. Create `my-skills/skills/<skill-name>/SKILL.md` with frontmatter following the conventions above.
2. Commit and push. **Do not edit `marketplace.json` or `plugin.json`** — the `skills/` directory is auto-discovered.

`README.md` § "How updates reach other machines" documents the consumer-side update flow (`/plugin marketplace update` + `/reload-plugins`, plus the opt-in auto-update toggle).

## Sanity checks

The only mechanical check that meaningfully applies is JSON validity of the two manifests:

```bash
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json')); json.load(open('my-skills/.claude-plugin/plugin.json'))"
```

For end-to-end testing without GitHub round-trips, add this directory as a local marketplace from another Claude Code session: `/plugin marketplace add /home/leo/code/my-skills-marketplace`.