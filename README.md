# my-skills-marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
for my personal skills. Installing once on a machine makes the skills available, and
they auto-update from this repo — no manual copying into `~/.claude/skills/`.

## Structure

```
.claude-plugin/marketplace.json   catalog (lists the one plugin; written once)
my-skills/                        the plugin
├── .claude-plugin/plugin.json    plugin manifest
└── skills/                       every skill dir here is auto-discovered
    ├── claude-github-code-review/
    └── karpathy-guidelines/
```

## Install on a new machine

```
/plugin marketplace add LeoDev0/my-skills-marketplace
/plugin install my-skills@my-skills-marketplace
```

(Replace `LeoDev0/my-skills-marketplace` with the actual GitHub repo if different.)

## Add a new skill

1. Put the skill in `my-skills/skills/<skill-name>/SKILL.md`.
2. Commit and push. **No manifest edits** — `marketplace.json` and `plugin.json`
   never change when you add a skill; the `skills/` directory is auto-discovered.

That's the whole workflow. Pushing is the only manual step.

## How updates reach other machines

`plugin.json` intentionally has **no `version` field**. Per Claude Code's docs,
when `version` is omitted and the marketplace is hosted in git, *every commit
automatically counts as a new version*. So each push is picked up without any
version bookkeeping.

On machines where `my-skills` is installed, Claude Code runs **background
marketplace auto-updates at startup** — they pull new/changed skills on the next
launch with no action required. This repo being public means no token or auth is
needed for that.

To force a refresh mid-session without restarting:

```
/plugin marketplace update my-skills-marketplace
```

> Do **not** add a `version` field back unless you specifically want to pin
> installs to explicit releases — doing so disables per-commit auto-updates and
> requires bumping the field on every change.
