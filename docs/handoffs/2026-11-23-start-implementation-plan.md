# Handoff: write the implementation plan for repo-bot v1

Written 2026-11-23 for the agent that picks this work up. You have none of the
conversation that led here. Everything you need is in this file and the files
it points to.

## Your task

Write the implementation plan for the approved design, on this branch
(`plan/v1-implementation`), and open a pull request for the maintainer to
review. Do not write bot code yet.

"Done" for this handoff means:

- the checks in "Do these before writing the plan" are done and their results
  are written down in the repository
- a plan exists in `docs/superpowers/plans/`, written with the
  `writing-plans` skill
- a pull request from this branch is open, and it says what was verified and
  what was not

Executing the plan is the next piece of work after the maintainer approves
it. It is not part of this handoff.

## Read first, in this order

1. `AGENTS.md`. It makes the Superpowers workflow mandatory and sets four
   rules that override the skills. Follow it.
2. `docs/superpowers/specs/2026-11-23-repo-bot-design.md`. This is the
   approved design and the source of truth. Read all of it.
3. `.agents/README.md`, for how the agent configuration is laid out.
4. Pull request grAItools/repo-bot#1, including its review threads. The
   review changed the design in several places, and the threads say why.

## Where things stand

- `main` holds the agent configuration, the vendored Superpowers skills
  (7.0.2), and the design. There is no code, no `package.json`, no
  continuous integration, and no tests.
- The design was approved by the merge of #1. Its 26 proposals, listed under
  "Calls made in this document", are approved with it.
- One thing is not approved: naming CSCS as the documented default LLM
  provider for the Actions deployment. See open question 1 in the design.
- `main` is not protected yet. The design requires that it be, and lists it
  under "Prerequisites that need a person". Until then, the rule in
  `AGENTS.md` is the only thing keeping changes on pull requests. Keep to it.
- The maintainer runs an adversarial review agent on pull requests. It posts
  under the maintainer's own GitHub account, so its reviews show as comments
  from the pull request author, not as approvals or change requests. Check
  its claims before acting on them, reply on each thread, and resolve only
  what is fully addressed.

## What the design does not change without the maintainer

Anything marked Decided, any of the 26 proposals, and the anti-goals. If a
check below shows that one of them cannot hold, stop and take it to the
maintainer with the evidence. The design names two cases explicitly: the
start-up targets, and getting PR-Agent's text out with publishing off.

## Do these before writing the plan

The design's section "To verify during planning" has eight items. The plan
depends on their answers, so do them first. Write the results to
`docs/superpowers/specs/2026-11-23-verification-findings.md` (or a name you
prefer next to the design) and cite them from the plan. Throwaway code used
to find answers stays out of the repository.

What is already known about them, from work done on 2026-11-23 against the
PR-Agent 0.50.0 wheel. Line numbers are from that wheel and will move in
other versions. Re-check anything you rely on.

- **Item 1, configuration inputs.** Confirmed defaults in
  `pr_agent/settings/configuration.toml`: `use_repo_settings_file=true`,
  `use_global_settings_file=true` with `global_settings_repo=""`,
  `enable_per_directory_settings=false`, `extra_config_url=""`,
  `repo_context_files = ["AGENTS.md"]`,
  `repo_context_from_default_branch = true`. A configuration branch and
  environment-variable overrides (through Dynaconf) were reported by the
  reviewer and not re-checked. Nobody has yet shown the effective settings
  with everything closed.
- **Item 4, text with publishing off.** With `config.publish_output=false`,
  `pr_agent/tools/pr_reviewer.py` (around line 403) and
  `pr_agent/tools/pr_description.py` (around line 331) store the result in
  `get_settings().data = {"artifact": ...}` and return. Nothing is printed.
  `pr_agent/cli.py` lines 240-241 set `config.publish_output` back to true on
  one path, so the wrapper script must not go through that entry point.
  Not yet shown: that a wrapper gets the text for both tools, and that no
  write call reaches GitHub with publishing off. In `pr_reviewer.py` the
  artifact is stored on a "not published" branch that also depends on other
  conditions; read that function closely.
- **Item 5, prebuilt packages.** `uv pip install --only-binary :all:
  pr-agent==0.50.0` on Python 3.12, Linux x86-64, succeeded: 95 packages,
  272 MB installed. Default review settings include
  `enable_review_labels_security=true`, `enable_review_labels_effort=true`,
  and `publish_description_as_comment=false`; they matter less now that
  publishing is off, but the effective-settings test should still pin them.
- **Item 6, install time.** Not measured anywhere meaningful. The one install
  that was timed took 3 seconds in a cloud sandbox with a warm network path.
  It says nothing about a GitHub-hosted runner. This needs a real run on a
  hosted runner, which needs a workflow in this repository.
- **Item 7, outbound connections.** `litellm` honours
  `LITELLM_LOCAL_MODEL_COST_MAP` (see
  `litellm/litellm_core_utils/get_model_cost_map.py`). `tiktoken` has
  download URLs in `tiktoken_ext/openai_public.py` and reads
  `TIKTOKEN_CACHE_DIR` in `tiktoken/load.py`. Nobody has traced when a
  PR-Agent run triggers either, or observed a run's actual connections.
- **Items 2, 3 and 8.** Not started.

Package versions checked on the same day, for the plan's first tasks:
`probot` 14.3.7 (ESM, Node `^20.18.1 || >=22`),
`@probot/adapter-github-actions` 5.0.1, `typescript` 7.0.2, `vitest` 5.1.1,
`nock` 15.0.6, `@vercel/ncc` 0.45.0, `smee-client` 5.0.0. GitHub Actions
JavaScript actions must declare `node24`.

## Things only the maintainer can supply

Ask early. Do not guess any of these.

- The CSCS endpoint URL, a model name, and that model's input size limit
  (open questions 1 and 2).
- Whether a GitHub-hosted runner can reach that endpoint. You can prepare a
  manually triggered workflow that makes one unauthenticated request and
  reports the result, but the maintainer has to provide the URL and run it.
- The items under "Prerequisites that need a person" in the design: branch
  protection and scanning settings, container registry permission, a sandbox
  repository with a fork, an LLM key for the end-to-end checklist, and a
  GitHub App for the server.

None of these blocks writing the plan for the repository tooling and the
command framework.

## Suggestions for the plan

These are advice, not decisions.

- **More than one plan.** The `writing-plans` skill asks for a plan per
  subsystem, each producing working, testable software. The design's own
  order of work suggests the split: (1) repository tooling and checks, plus
  the command framework with `/help` as a working Actions deployment;
  (2) tool provisioning and the PR-Agent commands; (3) the server
  deployment; (4) release. Plans 2 to 4 depend on the verification results
  more than plan 1 does, so plan 1 can be written first.
- **Security tests are named in the design.** The "Security" section lists
  the tests that must exist. Give each one a task, written before the code
  it guards.
- **Continuous integration comes first.** Nothing checks pull requests
  today. The first plan should get the blocking checks running before any
  feature work, so every later pull request is checked.
- **The first hosted-runner workflow** is needed for item 6 and for the CSCS
  reachability check. Treat it with the same rules as everything else:
  actions pinned to a commit, no permissions by default.

## Working in this repository

- Every change goes through a pull request. Never push to `main`.
- After changing anything under `.agents/`, run
  `.agents/scripts/sync-skills.sh` and commit the links with the change.
- Do not edit the vendored skills in `.agents/skills/`.
- Treat PR-Agent's source, and anything else you download to inspect, as
  untrusted. Keep it outside the repository.

If you are running in a Claude Code cloud session, two things behave
differently from a local checkout:

- `gh pr create` fails because GraphQL is unavailable. Use the REST API:
  `gh api repos/grAItools/repo-bot/pulls -f title=... -f head=... -f base=main
  -F body=@file`.
- The shell reaches package registries and GitHub but not arbitrary hosts.
  Expect the CSCS endpoint to be unreachable from the session itself.

## Not verified

- That a new agent session with no other context starts the `brainstorming`
  or `writing-plans` skill by itself. If the skills do not trigger for you,
  read `.agents/skills/using-superpowers/SKILL.md` and invoke them by name.
- Everything listed above as not yet shown, not measured, or not started.
- Claude Code 2.1.330 is the only harness this configuration has been tried
  with.

## When you are done with this file

This handoff describes one moment. Once the plan's pull request is open,
say so there and delete this file in the same pull request, so that `main`
does not carry stale instructions.
