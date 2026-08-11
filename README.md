# caatinga-skill

A Claude Code **plugin** providing a skill for Caatinga (`@caatinga/cli`, `ctg`), a TypeScript-first deployment orchestration toolkit for Stellar/Soroban.

This repo is itself a plugin **marketplace**: [github.com/Dione-b/caatinga-skill](https://github.com/Dione-b/caatinga-skill).

## Structure

```
ctg-skills/
├── .claude-plugin/
│   ├── plugin.json        # plugin manifest (name, description, version)
│   └── marketplace.json   # marketplace manifest — lets this repo be added as a plugin source
├── skills/
│   └── caatinga/
│       ├── SKILL.md       # the skill itself (frontmatter + instructions)
│       └── evals.json     # eval prompts used to validate the skill's behavior
└── README.md
```

This follows Claude Code's plugin layout: a `.claude-plugin/plugin.json` manifest at the repo root, with skills under `skills/<skill-name>/SKILL.md`. `marketplace.json` turns the repo itself into a marketplace containing one plugin (`caatinga-skill`), so it can be added and installed directly from Git without a separate marketplace repo.

## Integrating the plugin

### Option A — Add directly from this repo (Git)

In Claude Code:

```
/plugin marketplace add Dione-b/caatinga-skill
/plugin install caatinga-skill@caatinga-skill
```

The first argument to `/plugin install` is the plugin name, the part after `@` is the marketplace name — both are `caatinga-skill` here since the repo defines a marketplace with one plugin of the same name.

### Option B — Local development install

Clone this repo and point Claude Code at it directly for local iteration:

```bash
git clone git@github.com:Dione-b/caatinga-skill.git
```

```
/plugin marketplace add /path/to/caatinga-skill
/plugin install caatinga-skill
```

Changes to `skills/caatinga/SKILL.md` are picked up without repackaging anything — no zip/build step.

### Verifying it loaded

Open Claude Code in a project and ask something skill-relevant (e.g. "deploy my Soroban contract" in a repo with `caatinga.config.ts`). Claude should reference Caatinga-specific guidance (e.g. `ctg deploy`, `caatinga.artifacts.json`) rather than generic Stellar CLI advice. You can also run `/plugin` to confirm `caatinga-skill` is listed as installed.

## Adding or updating a skill

1. Create a new folder under `skills/` named after the skill: `mkdir skills/my-skill`
2. Add `skills/my-skill/SKILL.md` with YAML frontmatter (`name`, `description`) followed by the skill's instructions in Markdown.
3. Optionally add `skills/my-skill/evals.json` with prompts + expected behavior to regression-test the skill's triggering and guidance.
4. No packaging step needed — the plugin is loaded straight from the repo/directory structure.

## Running evals

Each `evals.json` lists prompts with expected outcomes (including negative cases — prompts that should *not* trigger the skill). There's no automated runner in this repo yet; evals are meant to be run manually or wired into whatever harness you use to validate skill behavior before shipping changes to `SKILL.md`.
