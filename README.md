# ctg-skills

Claude Code skills for [Caatinga](https://github.com/) (`@caatinga/cli`, `ctg`), a TypeScript-first deployment orchestration toolkit for Stellar/Soroban.

## Structure

```
ctg-skills/
├── caatinga/
│   ├── SKILL.md        # the skill itself (frontmatter + instructions)
│   └── evals.json       # eval prompts used to validate the skill's behavior
├── caatinga.skill        # packaged zip of caatinga/ (built artifact, do not hand-edit)
└── README.md
```

Each skill lives in its own directory named after the skill (matching the `name:` field in its `SKILL.md` frontmatter). This is the layout Claude Code expects when skills are installed individually or loaded from a plugin/marketplace — one folder per skill, each containing exactly one `SKILL.md`.

The `.skill` file is a zip archive of a skill's folder (currently just `caatinga/SKILL.md`) and is a build output, not a source file. Regenerate it after editing `caatinga/SKILL.md` rather than editing the zip directly:

```bash
cd caatinga && zip -r ../caatinga.skill SKILL.md && cd ..
```

## Integrating a skill

### Option A — Project-level (this repo only)

Copy the skill folder into your project's `.claude/skills/` directory:

```bash
mkdir -p /path/to/your-project/.claude/skills
cp -r caatinga /path/to/your-project/.claude/skills/
```

Claude Code will pick it up automatically the next time it lists available skills in that project.

### Option B — User-level (all your projects)

Copy into your global skills directory instead:

```bash
mkdir -p ~/.claude/skills
cp -r caatinga ~/.claude/skills/
```

### Option C — Install the packaged `.skill` file

If you have the `.skill` archive (e.g. shared by a teammate or downloaded), unzip it into either skills directory above:

```bash
unzip caatinga.skill -d ~/.claude/skills/
```

This produces `~/.claude/skills/caatinga/SKILL.md`, identical to Option B.

### Verifying it loaded

Open Claude Code in a project and ask something skill-relevant (e.g. "deploy my Soroban contract" in a repo with `caatinga.config.ts`). Claude should reference Caatinga-specific guidance (e.g. `ctg deploy`, `caatinga.artifacts.json`) rather than generic Stellar CLI advice. You can also check the skills list via the `/skills`-style listing Claude Code surfaces in its system context.

## Adding or updating a skill

1. Create a new folder named after the skill: `mkdir my-skill`
2. Add `my-skill/SKILL.md` with YAML frontmatter (`name`, `description`) followed by the skill's instructions in Markdown.
3. Optionally add `my-skill/evals.json` with prompts + expected behavior to regression-test the skill's triggering and guidance.
4. Package it if you want a distributable artifact: `cd my-skill && zip -r ../my-skill.skill SKILL.md && cd ..`

## Running evals

Each `evals.json` lists prompts with expected outcomes (including negative cases — prompts that should *not* trigger the skill). There's no automated runner in this repo yet; evals are meant to be run manually or wired into whatever harness you use to validate skill behavior before shipping changes to `SKILL.md`.
