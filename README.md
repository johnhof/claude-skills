# claude-skills

Personal Claude Code skill library. These skills extend Claude's behavior for common workflows.

## Skills

| Skill | Description |
|-------|-------------|
| [arbiter](./arbiter/SKILL.md) | Interactive local diff review via the Arbiter UI |
| [implement](./implement/SKILL.md) | Multi-agent parallel implementation workflow with TDD, review cycles, and automated PR creation |
| [pr-helper](./pr-helper/SKILL.md) | Automated PR comment triage, CI failure resolution, and merge polling |

## Usage

Skills are loaded automatically by Claude Code from `~/.claude/skills/`. To install, clone this repo and symlink or copy the skill directories there:

```bash
git clone https://github.com/johnhof/claude-skills ~/.claude/skills
```

Or to keep them in sync:
```bash
# from ~/.claude/skills/
git pull
```
