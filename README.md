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

## Pull updates on other machines

```
/plugin marketplace update my-skills-marketplace
```

Bump `version` in `my-skills/.claude-plugin/plugin.json` if you want installs pinned
to explicit releases; otherwise updates track the latest commit.
