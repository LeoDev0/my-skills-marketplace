# my-skills-marketplace

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
for my personal skills. Installing once on a machine makes the skills available and
Claude Code manages the files — no manual copying into `~/.claude/skills/`. Pulling
new commits to other machines is two short commands (see [How updates reach other
machines](#how-updates-reach-other-machines) below).

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
/reload-plugins
```

(Replace `LeoDev0/my-skills-marketplace` with the actual GitHub repo if different.)

## Add a new skill

1. Put the skill in `my-skills/skills/<skill-name>/SKILL.md`.
2. Commit and push. **No manifest edits** — `marketplace.json` and `plugin.json`
   never change when you add a skill; the `skills/` directory is auto-discovered.

That's the whole workflow. Pushing is the only manual step.

## How updates reach other machines

`plugin.json` intentionally has **no `version` field**. Per [Claude Code's
plugin marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces#version-resolution-and-release-channels),
when `version` is omitted and the marketplace is hosted in git, *every commit
automatically counts as a new version*. So each push is picked up without any
version bookkeeping.

On machines where `my-skills` is installed, there are **two paths** to pull a new
commit. Neither is fully zero-touch on a third-party marketplace like this one — see
the auto-update note below.

**Manual (works out of the box):**

```
/plugin marketplace update my-skills-marketplace   # fetches updates to disk
/reload-plugins                                    # loads them into the running session
```

The first command refreshes the marketplace catalog and pulls updated plugin
content; the second activates it in the current session. Restarting Claude Code
works in place of `/reload-plugins`. See
[Apply plugin changes without restarting](https://code.claude.com/docs/en/discover-plugins#apply-plugin-changes-without-restarting).

**Opt-in auto-update (per machine, once):**

Per [Configure auto-updates](https://code.claude.com/docs/en/discover-plugins#configure-auto-updates),
*third-party marketplaces have auto-update disabled by default* — only the official
Anthropic marketplace is on by default. To enable it for this one: run `/plugin`,
go to the **Marketplaces** tab, select `my-skills-marketplace`, and choose
**Enable auto-update**. After that, Claude Code refreshes the marketplace at
startup; if any plugins changed it shows a notification prompting you to run
`/reload-plugins` to load them. **The reload step is still required even with
auto-update on** — auto-update fetches, it doesn't activate.

> Do **not** add a `version` field back unless you specifically want to pin
> installs to explicit releases — doing so disables per-commit auto-updates and
> requires bumping the field on every change.
