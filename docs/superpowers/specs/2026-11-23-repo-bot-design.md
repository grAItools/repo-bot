# repo-bot design

Status: draft, awaiting maintainer approval. Nothing is built until it is approved.

Date: 2026-11-23

## How to read this document

Three kinds of statement appear here, and they are marked:

- **Decided**: the maintainer said so.
- **Proposed**: a call made while writing this document. The maintainer can
  overrule any of them. They are collected in "Calls made in this document".
- **Open**: only the maintainer can answer, and the answer changes what gets
  built. They are collected in "Open questions".

## Intent

`repo-bot` is a GitHub bot that lets people trigger automation on a pull
request by writing a slash command in a comment. The first two commands ask an
LLM to review or describe the pull request.

The maintainer wants one bot, developed in one repository, that any other
repository can use in either of two ways:

- **Actions deployment.** A repository adds a small workflow that references
  `grAItools/repo-bot`. No server is involved. This is the easy way in.
- **Server deployment.** The same bot runs as a long-lived GitHub App server
  for organisations that want faster responses and one installation for many
  repositories.

The reason for building it on Probot is that both deployments can share the
same handler code, so behaviour and security checks cannot drift apart.

The project will be developed entirely by coding agents. The maintainer
requires very high code quality, and security above all, because the bot holds
write permissions and reacts to text that strangers can post.

## Decided

- The bot is called `repo-bot` and is built on Probot.
- It deploys both through GitHub Actions and as a server, from the same code.
- It lives at `grAItools/repo-bot`, is public, and must be usable from any
  repository that wants it.
- Commands are slash commands in pull request comments.
- Version 1 includes `/pr:review` and `/pr:describe`. They run the `review`
  and `describe` tools of [PR-Agent](https://github.com/qodo-ai/pr-agent),
  using the CSCS inference server as the LLM provider.
- Target repositories are public and accept pull requests from forks. Future
  commands may need to run or modify pull request code, so the security model
  has to be strict from the start.
- The server deployment target is a Docker container for now. This was given
  as a provisional answer.
- The repository uses a harness-agnostic agent layout inspired by the grAItools
  project blueprint, and Superpowers as the development workflow.

## Goals

1. A command framework: parse a comment, decide whether it is a command,
   check that the commenter may run it, run it, and report the result on the
   pull request.
2. `/pr:review` and `/pr:describe`, backed by PR-Agent and a configurable
   OpenAI-compatible LLM endpoint, with CSCS as the documented default.
3. `/help`, listing the commands enabled in the current repository.
   (Proposed.)
4. Two deployments from one codebase, with identical command behaviour.
5. A repository that agents can develop safely: blocking quality and security
   checks, a written threat model, and tests for every security rule.

## Non-goals for version 1

- Commands that check out, build, run, or push pull request code. The
  framework must not make them impossible, but none ship in version 1.
  See "Future commands that touch pull request code".
- Automatic behaviour without a command, such as reviewing every new pull
  request, labelling, or assigning reviewers.
- Other PR-Agent tools (`improve`, `ask`, and the rest).
- Passing user-supplied options through to PR-Agent.
- Letting a consuming repository customise PR-Agent (extra instructions,
  language, review sections). PR-Agent's behaviour is fixed by the bot in
  version 1. (Proposed.)
- Rate limiting beyond the duplicate-run rule in "Limits on use".
- `@repo-bot` mention syntax.
- Platforms other than GitHub.com. GitHub Enterprise Server is not tested.
- A hosted public instance of the server. Each organisation runs its own.

## Anti-goals

The project has failed, even if the commands work, if any of these is true:

- Someone without write access to a repository can make the bot act, spend
  the repository's LLM budget, or change a pull request.
- Text in a comment or a pull request can change which program runs, which
  arguments it gets, where the LLM key is sent, or what the bot's token is
  used for.
- A secret (LLM key, GitHub token, App private key, webhook secret) appears
  in a log, a comment, or an error message.
- The Actions and server deployments make different authorisation decisions
  for the same comment.
- A security rule exists only in prose and has no test.

## Constraints

Checked on 2026-11-23:

- Probot 14.3.7 is ESM-only and needs Node `^20.18.1 || >=22`.
- `@probot/adapter-github-actions` 5.0.1 depends on Probot `^14.0.2`.
- GitHub Actions JavaScript actions must use `node24`. Node 20 was removed
  from runners in September 2026.
- PR-Agent is a Python program, version 0.50.0 on PyPI. The bot is Node. They
  can only meet through a process boundary.
- In the Actions deployment the workflow trigger is `issue_comment`. That
  event runs the workflow from the default branch with access to secrets,
  and anyone can post a comment. The fork-checkout protection added to
  `actions/checkout` does not cover it.
- The workflow's `GITHUB_TOKEN` cannot push to fork branches, and its
  actions do not trigger other workflows.
- The Actions adapter delivers one event per run and only supports events
  that Actions exposes. Server-only features (custom HTTP routes, acting on
  other installations, in-memory state across events) must not be needed by
  any command.

## Approaches

The main design question is how a Node bot runs a Python tool in both
deployments. Three ways were considered.

### A. One container image, PR-Agent as a subprocess (proposed)

The bot and a pinned PR-Agent are built into one container image. The bot
starts PR-Agent as a child process. The same image is the server and is also
what the GitHub Action runs (a Docker container action pointing at the
published image by digest).

- For: one artifact and one code path for the security-sensitive step in both
  deployments. PR-Agent's version and its dependencies are pinned and scanned
  once. Consumers install nothing.
- Against: every command in the Actions deployment pays an image pull, an
  estimated 20 to 40 seconds, including `/help`. Docker container actions run
  only on Linux runners.

### B. JavaScript action plus a separate PR-Agent step

The Action is a bundled JavaScript action that validates the command and sets
outputs. A following workflow step, in a reusable workflow, runs PR-Agent.
The server still uses a subprocess or a sidecar.

- For: fast start for cheap commands.
- Against: the PR-Agent invocation exists twice, once in YAML and once in
  code. The two can drift, and YAML is harder to test. This works against the
  reason for choosing Probot.

### C. Dispatch to a second workflow

The bot validates the command and sends a `repository_dispatch` event; a
second workflow runs PR-Agent.

- For: clean separation between the privileged listener and the worker.
- Against: needs a token that can trigger workflows (a GitHub App or a
  personal token) even for the simplest setup, adds a second run per command,
  and still leaves the server needing its own path.

**Proposed: A.** The maintainer has not yet chosen. Approving this document
approves A. If the image pull time turns out to be unacceptable, the fallback
is a JavaScript action for commands that do not need PR-Agent, kept as a later
change, not part of version 1.

## Design

### Shape

```
comment event
   │
   ▼
entrypoint ──► the same Probot app function
(Actions adapter        │
 or Probot server)      ▼
                 1. filter   is this a new comment on a PR, by a human?
                 2. parse    is the first line a known command?
                 3. config   is the command enabled in this repository?
                 4. authz    may this person run it?
                 5. ack      react to the comment
                 6. run      command handler
                 7. report   react, and comment on failure
```

Steps 1 to 4 are pure decisions with no side effects apart from two read-only
API calls (repository configuration and the commenter's permission). Nothing
is written to GitHub and no child process starts until step 4 has passed.

### Parts

Each part has one job and can be tested alone.

| Part | Job |
|------|-----|
| `app` | Registers the `issue_comment.created` handler and wires the parts together. The only part both entrypoints import. |
| Entrypoints | `action`: runs `app` once through the Actions adapter. `server`: runs `app` under Probot's server. No logic of their own. |
| Parser | Turns a comment body into either "not a command" or `{namespace, name}`. Pure function. |
| Command registry | The fixed list of commands the bot knows, each with a handler, a default required permission, and help text. |
| Repository config | Loads and validates `.github/repo-bot.yml` from the repository's default branch. |
| Authoriser | Decides whether the commenter may run the command. |
| Reporter | Reactions and comments back to the pull request. The only part that writes user-visible output. |
| PR-Agent runner | Starts PR-Agent for one pull request and one tool, and returns success or a failure reason. The only part that starts a process. |
| Task runner | Runs a handler to completion. In Actions it runs inline. On the server it puts the work on a bounded in-process queue so the webhook is answered quickly. |
| Settings | Reads deployment settings (LLM endpoint, model, key, timeouts) from environment variables, validates them at start-up, and never logs their values. |

### Command grammar

- Only the first line of the comment is considered.
- The line must match, in full: a slash, an optional namespace followed by a
  colon, and a name. Namespace and name are lowercase letters, digits, and
  hyphens. Example: `/pr:review`, `/help`.
- Version 1 commands take no arguments. A first line with anything after the
  command name is rejected with a short reply that the command takes no
  arguments. It is not run with the extra text ignored.
- An unknown command is ignored silently, so that other bots' slash commands
  in the same repository do not produce noise.
- Edited comments are never treated as commands.

### Who may run a command

- The comment must be on a pull request, not an issue.
- The sender must be a user, not a bot or an App.
- The commenter's permission on the repository is read from the GitHub API
  at the time of the comment. The `author_association` field in the event is
  not used for the decision.
- Each command has a required permission. The default for all version 1
  commands is `write`. A repository may raise it in its configuration but
  may not lower it below `write` in version 1. (Proposed.)
- A refused command gets a single reaction and no comment, so that the bot
  cannot be used to generate comment spam.

### Repository configuration

An optional `.github/repo-bot.yml` in the consuming repository, read only from
the default branch, never from the pull request. It can:

- enable or disable individual commands (all version 1 commands are enabled
  by default)
- raise the required permission for a command

It is validated against a strict schema. Unknown keys are an error. If the
file is invalid, the bot does nothing except report that its configuration is
invalid, to someone who would have been allowed to run the command.

The LLM endpoint, model, and key are deployment settings, not repository
configuration. They cannot be set from this file.

### Running PR-Agent

- PR-Agent is installed in the image at an exact version with hash-checked
  dependencies.
- The runner starts it directly, without a shell, with a fixed argument list:
  the pull request URL built from the event's repository and number, and the
  tool name taken from the command registry. No text from the comment or the
  pull request is ever part of the arguments.
- The child process gets a new environment built from an allow-list: the
  GitHub token for this event, the LLM settings, and what Python needs to
  run. The bot's own environment is not inherited.
- The run has a time limit. On timeout the process is killed and the failure
  is reported.
- PR-Agent's output is captured, never echoed to a comment, and passed
  through secret masking before it reaches a log.
- PR-Agent posts its own review or description on the pull request. The bot
  reports only success or failure.
- PR-Agent's repository-level settings file is turned off in both
  deployments. The bot supplies the whole PR-Agent configuration. (Proposed.)
- PR-Agent is configured so that `/pr:review` only posts a comment. It adds
  no labels and changes nothing else on the pull request. (Proposed.)

What PR-Agent writes comes from an LLM that has read text a fork author
controls, so the content of its comment cannot be trusted. The design accepts
that risk for comment text and nothing more: the bot never reads PR-Agent's
output to decide what to do, and PR-Agent is given no ability to act beyond
the comment (and, for `describe`, whatever the maintainer allows under Open
questions). The threat model records this as an accepted risk.

**To verify during planning, before any PR-Agent code is written:**

1. How to turn PR-Agent's repository-level settings (`.pr_agent.toml`) off
   completely, and proof that it works. If PR-Agent could read them from a
   pull request branch, a fork could redirect the LLM endpoint and capture
   the key.
2. The exact setting names for an OpenAI-compatible endpoint (base URL, key,
   model name, maximum input size) in PR-Agent 0.50.0.
3. The exact setting names for passing a GitHub token.
4. That `describe` can be limited to what the maintainer wants it to change
   (see Open questions).

Findings go into the threat model with a test for each.

### LLM provider

- Settings: base URL, model name, API key, maximum input size. All come from
  the deployment (Actions secrets and variables, or server environment).
- The documented default is the CSCS inference API. Any OpenAI-compatible
  endpoint works, so that repositories outside CSCS can use the bot.
- The base URL must be `https`. (Proposed.)

### Actions deployment

- The repository publishes a Docker container action and a reusable workflow
  that wraps it with the right trigger filter and permissions.
- A consuming repository adds a workflow of a few lines that listens for
  `issue_comment` and calls the reusable workflow, passing its LLM secret.
- The workflow skips, at no runner cost, comments that are not on a pull
  request or do not start with a slash.
- Token permissions requested: `pull-requests: write`, `issues: write`,
  `contents: read`. Nothing else.
- The pull request's code is never checked out.
- Documentation tells consumers to pin the action to a commit.

### Server deployment

- The same image, started in server mode, as a GitHub App.
- Probot verifies webhook signatures. Unsigned or wrongly signed requests are
  rejected.
- For each event the server uses a short-lived installation token scoped to
  the event's repository, and gives PR-Agent that token and no other.
- Long work runs on a bounded queue with a fixed number of workers. When the
  queue is full, new commands are refused with a reply saying so.
- The container runs as a non-root user with a read-only root filesystem.
- A health endpoint reports whether the process is up. It exposes nothing
  else.
- App permissions requested: pull requests read and write, issues read and
  write, contents read, metadata read. Events: issue comment.
- The server spends its operator's LLM key on behalf of every repository it
  serves. It therefore only acts for accounts on an allow-list of owners set
  by the operator. An event from any other installation is dropped. The
  allow-list is required: the server refuses to start without one.
  (Proposed.)

### Limits on use

- Only one run of a given command is active per pull request. A second
  request for the same command on the same pull request while one is running
  is refused with a reaction. In Actions this is a workflow concurrency group
  per pull request and command, without cancelling the run in progress. On
  the server the queue enforces it. (Proposed.)
- There is no other rate limit in version 1. People who can run commands
  already have write access to the repository.

### Reporting

- On accept: an "eyes" reaction on the command comment.
- On success: a "rocket" reaction. PR-Agent's own output is the result.
- On failure: a "confused" reaction and one short comment with a reason
  category (timed out, provider unavailable, misconfigured, internal error)
  and a reference a maintainer can find in the logs. No raw error text.

### Future commands that touch pull request code

Not built in version 1, but the framework reserves the rule now: the process
that holds write credentials or secrets never executes code from a pull
request. A command that needs to run such code must do it in a separate job or
sandbox with no secrets and a read-only token, and pass results back as data.
Any such command needs its own design document.

## Security

Security requirements are written before code and are part of the definition
of done.

- `docs/security/threat-model.md` lists what is protected, who might attack
  it, each entry point, and each rule that defends it. Every rule names the
  test that enforces it.
- `SECURITY.md` explains how to report a vulnerability.
- A regression suite of hostile inputs covers at least: commands from users
  without access, from bots, on issues, in edited comments; extra text after
  a command; shell and option characters; very long input; unusual Unicode,
  including look-alike characters and control characters; and malformed
  repository configuration.
- A test proves the child process environment contains only the allow-listed
  names.
- A test proves secrets do not appear in logs or comments on any failure
  path.
- Every finding from a review gets a regression test before it is closed.

## Repository and tooling

The blueprint has no Node profile, so this section defines one with the same
strictness as its Python baseline. (All proposed.)

- Language: TypeScript in strict mode, ESM, Node 24.
- Package manager: npm with a committed lockfile. Install scripts disabled.
- Tests: Vitest, with recorded GitHub API responses for handler tests and a
  stand-in `pr-agent` executable for runner tests. Coverage threshold
  enforced; the parser, authoriser, and runner at 100% of branches.
- Lint and format: ESLint with the strict type-checked rules, Prettier.
- Layout: `src/` for the parts above, `test/`, `docs/`, `action.yml`,
  `Dockerfile`, `.github/workflows/`.
- Agent layout: already in place on the `setup/superpowers` branch
  (`AGENTS.md`, `CLAUDE.md`, `.agents/`).

Continuous integration, all blocking on pull requests:

- typecheck, lint, format check, tests with coverage, build
- CodeQL for JavaScript/TypeScript and for the workflows themselves
- a workflow linter and a workflow security scanner
- dependency review, and a check that the lockfile is in sync
- image build, Dockerfile lint, and an image vulnerability scan
- the agent-skill link check from the pre-commit configuration

Repository rules:

- All third-party actions pinned to a commit. Workflows default to no
  permissions and request what each job needs.
- `main` is protected: pull requests only, required checks, at least one
  human approval.
- Automated dependency update pull requests.
- Secret scanning with push protection.

Releases:

- Each release publishes the image to the GitHub container registry with a
  provenance attestation and a software bill of materials.
- `action.yml` names the image by digest, so the image has to exist before
  the commit that refers to it. A release is therefore a pull request: the
  image is built from the current `main`, and a release pull request changes
  only the version and the digest in `action.yml`. A human merges it, and the
  merge commit is tagged. The image is one commit behind the tag, and that
  commit contains no code. Nothing bypasses branch protection. (Proposed.)
- Consumers pin the tagged commit. The reusable workflow refers to the action
  in the same repository at the same commit.

Testing against real services:

- Automated tests never call GitHub or an LLM. They use recorded responses
  and the stand-in executable.
- End-to-end checks against a real repository and a real LLM endpoint are
  run by hand before a release, following a written checklist. (Proposed.)

## Prerequisites that need a person

Agents cannot do these. They block the steps noted.

- Turn on branch protection, secret scanning with push protection, and code
  scanning for `grAItools/repo-bot`. Blocks: the repository rules above.
- Allow the repository to publish packages to the container registry.
  Blocks: releases.
- Provide a sandbox repository, and a fork of it, for the end-to-end
  checklist. Blocks: the first release.
- Provide an LLM endpoint and key for the end-to-end checklist. Blocks: the
  first release.
- Register a GitHub App for the server deployment. Blocks: end-to-end checks
  of the server.

## Development workflow

- Superpowers skills, as set out in `AGENTS.md`: plan from this design,
  test-first implementation, review, and verification before anything is
  called done.
- Each pull request gets a security-focused review by an agent that did not
  write the change, in addition to the human approval.

## Left to the builder

- Module boundaries inside `src/` and exact type definitions.
- The schema library for configuration and settings validation.
- The choice of workflow linter, workflow security scanner, Dockerfile
  linter, and image scanner.
- Base images, as long as they are pinned by digest and small.
- Log format, as long as it is structured and masks secrets.
- Queue size, worker count, and the PR-Agent time limit, as documented
  defaults that a deployment can change.
- Wording of help text and failure comments.
- The order of work, to be set out in the implementation plan. The plan
  should start with the repository tooling and checks, then the command
  framework with `/help`, then PR-Agent, then the server deployment, then
  release.

## Calls made in this document

The maintainer should check these while reading.

1. Approach A: one image, PR-Agent as a subprocess, Docker container action.
2. `/help` is included.
3. Version 1 commands take no arguments, and extra text is rejected.
4. Unknown commands are ignored silently; refused commands get a reaction
   only.
5. Required permission defaults to `write` and cannot be lowered.
6. Repository configuration can only enable, disable, and tighten. LLM
   settings are deployment-only.
7. The LLM endpoint is any OpenAI-compatible `https` URL, with CSCS as the
   documented default.
8. The server uses an in-process queue with no external store.
9. TypeScript, Node 24, npm, Vitest, ESLint, Prettier.
10. `main` requires a human approval on every pull request.
11. OpenSpec is not used; Superpowers design documents and plans are the
    record of intent.
12. PR-Agent's repository settings file is off, and consuming repositories
    cannot customise PR-Agent in version 1.
13. `/pr:review` only posts a comment. LLM-written comment text based on
    fork content is an accepted risk; the bot never acts on it.
14. The server requires an allow-list of account owners.
15. One active run per command per pull request; no other rate limit.
16. Releases go through a human-merged pull request that sets the image
    digest; the image is one code-free commit behind the tag.
17. End-to-end checks are manual, from a checklist, before each release.

## Open questions

1. **What may `/pr:describe` change?** PR-Agent's describe tool can rewrite
   the pull request title and body and add labels. Should it overwrite the
   author's title and description, or post its description as a comment and
   leave them alone?
2. **Which CSCS model, and what is its input size limit?** The bot needs a
   model name and a maximum input size to configure PR-Agent.
3. **Can GitHub-hosted runners reach the CSCS endpoint?** If it is only
   reachable from certain networks, the Actions deployment cannot use it
   from GitHub's runners, and the design needs either self-hosted runners or
   a different default provider for that deployment.
4. **Who provides the LLM key for other organisations' repositories?** The
   design assumes each consuming repository supplies its own endpoint and
   key. If the intent is for grAItools to provide CSCS access to others, that
   is a different and much riskier design.
5. **Who will do the items under "Prerequisites that need a person", and
   when?** They include registering the GitHub App and providing a sandbox
   repository and an LLM key for testing.
