# Caldo Skills

## Install

Add this repo as a plugin marketplace:

```
/plugin marketplace add caldoai/skills
```

Then install a skill with the command in its row of the table below.

To install by hand instead, copy a skill's folder into `~/.claude/skills/`.

## Skills

| Skill | Description | Install |
| --- | --- | --- |
| [restructure-codebase](skills/restructure-codebase/SKILL.md) | Refactor a codebase for agents. | `/plugin install restructure-codebase@caldo-skills` |

## Adding a skill

1. Create `skills/<name>/SKILL.md`.
2. Add an entry for it to `plugins` in [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json).
3. Add a row to the table above.

## License

[MIT](LICENSE)
