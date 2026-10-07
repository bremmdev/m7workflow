# m7workflow

Workflow and productivity skills for coding agents: the chores around the code, like keeping the ticket in view while you build. Each skill is a folder with a `SKILL.md` in the open [Agent Skills](https://agentskills.io) format, so the same files work in Claude Code, GitHub Copilot, Cursor, Codex, Gemini CLI and any other harness that reads skills.

For accessibility skills, see [m7skills](https://github.com/bremmdev/m7skills).

## Skills

| Skill | What it does |
| --- | --- |
| [`pbi-context`](skills/pbi-context/SKILL.md) | Loads the Azure DevOps work item behind the current branch, whether the number is a task or a PBI, with its backlog item, sibling tasks, Feature and Epic. Assembles the acceptance criteria from the criteria field and descriptions, and checks the branch's changes against them. Read-only, and never outputs colleagues' names or contact details. |

## Install

### Any agent: the `skills` CLI

```sh
npx skills add bremmdev/m7workflow
```

It asks which skills and which agents to install for. Useful flags:

```sh
npx skills add bremmdev/m7workflow --list                       # show the skills
npx skills add bremmdev/m7workflow -s pbi-context -a cursor     # one skill, one agent
npx skills add bremmdev/m7workflow -g                           # for your user, not this project
```

### By hand

Copy a folder from `skills/` into the folder your agent reads:

| Agent | Project | User |
| --- | --- | --- |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| GitHub Copilot | `.agents/skills/` or `.github/skills/` | `~/.copilot/skills/` |
| Cursor | `.agents/skills/` | `~/.cursor/skills/` |
| Codex | `.agents/skills/` | `~/.codex/skills/` |
| Gemini CLI | `.agents/skills/` | `~/.gemini/skills/` |

Keep the whole folder: `SKILL.md` links to the `references/` and `scripts/` next to it.

## Requirements

`pbi-context` needs Node 18+, `git`, and either `az login` or a personal access token with Work Items (Read). See [`setup.md`](skills/pbi-context/references/setup.md).

## Adding a skill

1. Create `skills/<name>/SKILL.md`. The frontmatter `name` must equal the folder name, and `description` says what the skill does and when to use it.
2. Write it for any harness: no tool names or paths specific to one agent, and paths relative to the skill folder.
3. No organisation names, URLs or credentials in the repository: read them from the environment or the git remote.
4. No personal information about colleagues in a skill's output: no names, e-mail addresses, phone numbers or user ids. Scrub it in the script, not only in the instructions.
5. Add a row to the table above.

## License

[MIT](LICENSE)
