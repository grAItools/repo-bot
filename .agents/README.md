# .agents/

Harness-agnostic agent configuration, following the
[AGENTS.md](https://agents.md) and [Agent Skills](https://agentskills.io)
conventions. The layout is modelled on the grAItools
[harness-agnostic project blueprint](https://github.com/grAItools/project-blueprints/blob/main/knowledge/harness-agnostic-project.md).

## Supported harnesses

No harness has been validated against this repository yet. When one is, record
it here.

| Harness | Tested version | Validated on | Adapters | Limitations |
|---------|----------------|--------------|----------|-------------|
| Claude Code | not yet validated | | `CLAUDE.md` imports `AGENTS.md` and the bootstrap skill; `.claude/skills/` links | |

## Layout

- `skills/`: shared skills (committed), one directory with a `SKILL.md` each
- `skills-local/`: private skills (gitignored)
- `scripts/sync-skills.sh`: links skills into harness directories that do not
  read `.agents/skills/` themselves
- `licenses/`: licenses of vendored third-party skills

## Third-party skills

| Source | Version | Commit | License |
|--------|---------|--------|---------|
| [obra/superpowers](https://github.com/obra/superpowers) `skills/` | 7.0.2 | `22f0b463e7a5e579904dd6211a520321035526f7` | MIT, see `licenses/superpowers-LICENSE` |

The skills are an unmodified copy of the upstream `skills/` directory at that
commit. They are vendored, not installed as a plugin, so that every harness
gets the same reviewed version and nothing updates on its own.

Upstream starts each session by injecting the `using-superpowers` skill through
a harness plugin hook. Here that is replaced by the instruction in the root
`AGENTS.md` and, for Claude Code, by an import in `CLAUDE.md`.

The `brainstorming` skill has an optional visual companion that starts a local
server and loads a logo from the upstream author's website. Set
`SUPERPOWERS_DISABLE_TELEMETRY=1` to turn that request off.

### Updating Superpowers

1. Check out the new upstream commit somewhere outside this repository.
2. Review the diff of its `skills/` directory against `.agents/skills/`.
3. Replace `.agents/skills/` with the new copy and update the table above.
4. Run `.agents/scripts/sync-skills.sh` and commit the skills and links together.

## Adding a skill

1. Create `.agents/skills/<name>/SKILL.md` with `name` and `description`
   frontmatter.
2. Run `.agents/scripts/sync-skills.sh`. It exits 1 after creating links, until
   they are staged.
3. Commit the skill and its links.

## Rules

- Edit shared skills here, not through the `.claude/skills/` links.
- Do not edit vendored Superpowers skills in place. Project-specific guidance
  goes in `AGENTS.md` or in a separate skill.
