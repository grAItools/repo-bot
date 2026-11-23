# repo-bot

`repo-bot` is a GitHub bot built on [Probot](https://github.com/probot/probot).
It responds to slash commands in pull request comments and is meant to be
deployable in two ways from the same code: as a GitHub Action that other
repositories reference, and as a standalone server.

The project is at the very beginning. There is no code or tooling yet. The
design is in
[`docs/superpowers/specs/2026-11-23-repo-bot-design.md`](docs/superpowers/specs/2026-11-23-repo-bot-design.md).
Read it before any work. The next piece of work is the implementation plan
for that design.

## Development workflow (mandatory)

This repository uses [Superpowers](https://github.com/obra/superpowers) for all
design and development work. The skills are vendored in `.agents/skills/`.

1. At the start of every session, read
   `.agents/skills/using-superpowers/SKILL.md` and follow it.
2. Before building anything that the design does not already cover, use the
   `brainstorming` skill. Do not write code before a design has been approved
   by the maintainer.
3. Follow the skills for planning, test-driven development, debugging, code
   review, and verification as they apply.

Skills refer to each other with a `superpowers:` prefix (for example
`superpowers:brainstorming`). In this repository that means the skill of the
same name in `.agents/skills/`.

Design documents go in `docs/superpowers/specs/` and implementation plans in
`docs/superpowers/plans/`, as the skills specify.

Where the skills offer a choice, these rules decide it:

- `main` changes only through pull requests. Never merge into `main` locally
  and never push to it. When `finishing-a-development-branch` offers its
  options, take the one that pushes the branch and opens a pull request.
- Never file an issue on another project, and never export or share a
  session transcript or diagnostic bundle (as `diagnosing-superpowers` can),
  unless the maintainer asks for exactly that in the same session.
- Set `SUPERPOWERS_DISABLE_TELEMETRY=1` in the environment of anything the
  skills start, including the `brainstorming` visual companion.
- Before committing a change under `.agents/`, run
  `.agents/scripts/sync-skills.sh` and commit the links with it.

## Security

This bot runs with repository write permissions and reacts to comments that
anyone can post, on public repositories that accept pull requests from forks.
Treat every comment, pull request title, body, branch name, and file as
untrusted input. Security requirements are part of the design, not a later
pass.

## Maintaining agent configuration

See [`.agents/README.md`](.agents/README.md) for how skills are stored, linked
into harness directories, and updated.
