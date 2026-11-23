# repo-bot design

Status: draft, awaiting maintainer approval. Merging the pull request that
adds this document approves it. Nothing is built before that.

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
- The Actions deployment runs directly on the standard GitHub runner, not in
  a container. Software that a command needs is installed on demand, when
  that command is triggered.
- The server deployment is a Docker container for now. This was given as a
  provisional answer.
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
4. Two deployments from one codebase. The same comment leads to the same
   decision and the same action in both, except for the one difference stated
   in "Limits on use".
5. In the Actions deployment, a command pays only for the software it uses.
   A command that needs no extra software starts as fast as the runner
   allows.
6. A repository that agents can develop safely: blocking quality and security
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
- Running the Actions deployment on self-hosted, Windows, macOS, or Arm
  runners. (Proposed.)
- `@repo-bot` mention syntax.
- Platforms other than GitHub.com. GitHub Enterprise Server is not tested.
- A hosted public instance of the server. Each organisation runs its own.

## Anti-goals

The project has failed, even if the commands work, if any of these is true:

- Someone without write access to a repository can make the bot act, spend
  the repository's LLM budget, change a pull request, or cause software to
  be downloaded.
- Text in a comment or a pull request can change which program runs, which
  arguments it gets, where the LLM key is sent, or what the bot's token is
  used for.
- A command run executes software whose exact content is not fixed by a hash
  committed in this repository.
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
- A JavaScript action runs without an install step, so its code and Node
  dependencies must be bundled and committed.
- PR-Agent is a Python program, version 0.50.0 on PyPI. The bot is Node. They
  can only meet through a process boundary.
- GitHub-hosted runners start clean. Nothing installed in one run is there in
  the next.
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

## Approach

The main design question was how a Node bot runs a Python tool in both
deployments.

### Chosen: JavaScript action with on-demand tools; container for the server

Decided by the maintainer on 2026-11-23.

- In Actions the bot is a bundled JavaScript action on the standard runner.
  When a command needs a tool such as PR-Agent, the bot's own code installs
  it for that run, from a lockfile in this repository, and then runs it.
- On the server the same tools are already installed in the container image,
  built from the same lockfiles.
- The code that decides whether to run a command, and the code that starts
  the tool, are the same in both deployments. Only the step that makes the
  tool available differs, and it sits behind one small interface.

What this costs:

- The install happens in a job that holds secrets, so it has to be locked
  down. See "Tools and provisioning".
- A command that needs PR-Agent waits for the install on every run.
- If the package index is unreachable, those commands fail in Actions.

### Considered and not chosen

- **One container image for both deployments**, run in Actions as a Docker
  container action. One artifact, but every command, including `/help`, waits
  for an image pull on every run, and the image grows with each tool added
  for any command.
- **A second workflow step that runs PR-Agent.** The PR-Agent call would then
  exist twice, in workflow YAML for Actions and in code for the server. The
  two could drift, and YAML is harder to test.
- **Dispatching to a second workflow.** Needs a token that can trigger
  workflows even for the simplest setup, adds a second run per command, and
  still leaves the server needing its own path.

## Design

### Shape

```
comment event
   │
   ▼
entrypoint ──► the same Probot app function
(Actions adapter        │
 or Probot server)      ▼
                 1. filter     is this a new comment on a PR, by a human?
                 2. parse      is the first line a known command?
                 3. config     is the command enabled in this repository?
                 4. authz      may this person run it?
                 5. ack        react to the comment
                 6. provision  make the command's tools available
                 7. run        command handler
                 8. report     react, and comment on failure
```

Steps 1 to 4 are pure decisions with no side effects apart from two read-only
API calls (repository configuration and the commenter's permission). Nothing
is written to GitHub, nothing is downloaded, and no child process starts until
step 4 has passed.

### Parts

Each part has one job and can be tested alone.

| Part | Job |
|------|-----|
| `app` | Registers the `issue_comment.created` handler and wires the parts together. The only part both entrypoints import. |
| Entrypoints | `action`: runs `app` once through the Actions adapter. `server`: runs `app` under Probot's server. Each chooses its tool provisioner and task runner. No other logic. |
| Parser | Turns a comment body into either "not a command" or `{namespace, name}`. Pure function. |
| Command registry | The fixed list of commands the bot knows, each with a handler, a default required permission, the tools it needs, and help text. |
| Repository config | Loads and validates `.github/repo-bot.yml` from the repository's default branch. |
| Authoriser | Decides whether the commenter may run the command. |
| Reporter | Reactions and comments back to the pull request. The only part of the bot that writes to GitHub. (PR-Agent, a separate program, posts its own result.) |
| Tool provisioner | Given a tool name from the registry, returns the path of a ready-to-run executable or a failure reason. Two implementations: install on demand (Actions) and find preinstalled (server). |
| PR-Agent runner | Starts PR-Agent for one pull request and one tool, and returns success or a failure reason. |
| Task runner | Runs a handler to completion. In Actions it runs inline. On the server it puts the work on a bounded in-process queue so the webhook is answered quickly. |
| Settings | Reads deployment settings (LLM endpoint, model, key, timeouts) from environment variables and never logs their values. Settings every command needs are validated at start-up. Settings only some commands need, such as the LLM settings, are validated when such a command runs, so `/help` works without them. |

The tool provisioner and the PR-Agent runner are the only parts that start
processes.

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
file is invalid, no command runs. A commenter who has the default required
permission, `write`, gets a reply that the configuration is invalid. Anyone
else gets nothing.

The LLM endpoint, model, and key are deployment settings, not repository
configuration. They cannot be set from this file.

### Tools and provisioning

A tool is an external program that a command needs. Version 1 has one:
PR-Agent, needed by `/pr:review` and `/pr:describe`. `/help` needs none.

What the repository holds for each tool:

- its exact version
- a lockfile listing every package it depends on, each with an exact version
  and content hashes
- the exact version of the language runtime it runs on

The same lockfiles are used to install on demand in Actions and to build the
server image, so both deployments run the same versions. A check in
continuous integration fails if they differ.

Installing on demand, in Actions (all proposed):

- It happens only after the command has been authorised, and only for the
  tools that command declares.
- It installs into a new private directory under the runner's temporary
  directory, and adds nothing to the system.
- Every package must match a hash in the lockfile. A mismatch stops the
  install and the command fails.
- Only prebuilt packages are accepted. Nothing is compiled and no package
  build script runs.
- The language runtime is pinned, not taken from whatever version the runner
  image happens to have.
- Anything the provisioner itself has to download in order to install (an
  installer, a runtime) is checked against a hash committed in this
  repository before it is used.
- The install process gets a new environment built from an allow-list. It
  contains no LLM key and no GitHub token.
- Nothing is cached between runs. The Actions cache is shared state that a
  less trusted run could alter, so the design does without it.
- It has a time limit.
- Supported platform: GitHub-hosted Linux x86-64 runners.

On the server:

- Tools are installed when the image is built, never while it is running.
- The provisioner only locates the installed tool. If it is missing, the
  command fails and the fault is logged as a broken image.

Start-up targets for the Actions deployment, to be measured during planning
(proposed):

- `/help` adds no more than 5 seconds to the runner's own start-up.
- Installing PR-Agent takes no more than 60 seconds.

If measurement shows a target cannot be met under the rules above, that goes
back to the maintainer before any rule is relaxed.

### Running PR-Agent

- The runner gets the PR-Agent executable from the tool provisioner.
- It starts PR-Agent directly, without a shell, with a fixed argument list:
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

### To verify during planning

Before any PR-Agent or provisioning code is written:

1. How to turn PR-Agent's repository-level settings (`.pr_agent.toml`) off
   completely, and proof that it works. If PR-Agent could read them from a
   pull request branch, a fork could redirect the LLM endpoint and capture
   the key.
2. The exact setting names for an OpenAI-compatible endpoint (base URL, key,
   model name, maximum input size) in PR-Agent 0.50.0.
3. The exact setting names for passing a GitHub token.
4. That `describe` can be limited to what the maintainer wants it to change
   (see Open questions).
5. That every package PR-Agent depends on is available prebuilt for the
   supported platform. If one is not, that goes back to the maintainer.
6. A way to get a pinned Python runtime and a hash-checked install on a
   GitHub-hosted runner that meets the rules in "Tools and provisioning",
   with measured times against the start-up targets.

Findings go into the threat model with a test for each.

### LLM provider

- Settings: base URL, model name, API key, maximum input size. All come from
  the deployment (Actions secrets and variables, or server environment).
- The documented default is the CSCS inference API. Any OpenAI-compatible
  endpoint works, so that repositories outside CSCS can use the bot.
- The base URL must be `https`. (Proposed.)

### Actions deployment

- The repository publishes a JavaScript action for the `node24` runtime, with
  the bot and its Node dependencies bundled into committed files.
- A consuming repository adds one short workflow, copied from a template in
  this repository's documentation. It listens for `issue_comment`, grants
  the permissions below, passes the LLM settings, and uses the action pinned
  to a commit.
- The template skips, at no runner cost, comments that are not on a pull
  request or do not start with a slash. That filter only saves runner time.
  The bot makes every check itself and is safe without it.
- This repository keeps a copy of the template as a real workflow, so that
  continuous integration checks it.
- There is no reusable workflow in version 1. A reusable workflow would have
  to name the action by a commit it cannot know, or by a tag that can move.
  (Proposed.)
- It runs on a GitHub-hosted Linux runner. No container is involved.
- Token permissions requested: `pull-requests: write`, `issues: write`,
  `contents: read`. Nothing else.
- The pull request's code is never checked out.
- Documentation tells consumers to pin the action to a commit.

### Server deployment

- A container image holding the bot and its tools, run as a GitHub App.
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

- On the server, only one run of a given command is active per pull
  request. A second request for the same command on the same pull request
  while one is running or waiting is refused with a reaction. (Proposed.)
- In Actions there is no such limit in version 1, so two runs of the same
  command can overlap. The only built-in mechanism, a workflow concurrency
  group, takes effect before the bot has authorised anyone, which would let
  an outsider's comment displace an authorised command that is waiting.
  (Proposed.)
- There is no other rate limit in version 1. People who can run commands
  already have write access to the repository.

### Reporting

- On accept: an "eyes" reaction on the command comment.
- On success: a "rocket" reaction. PR-Agent's own output is the result.
- On failure: a "confused" reaction and one short comment with a reason
  category (timed out, tool unavailable, provider unavailable, misconfigured,
  internal error) and a reference a maintainer can find in the logs. No raw
  error text.

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
- Tests prove that the environments of the install process and of the tool
  process contain only their allow-listed names, and that the install
  environment holds no secret.
- A test proves that nothing is installed or downloaded for a comment that
  is not an authorised command.
- A test proves that a package whose hash does not match stops the install.
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
  enforced; the parser, authoriser, tool provisioner, and runner at 100% of
  branches.
- Lint and format: ESLint with the strict type-checked rules, Prettier.
- Layout: `src/` for the parts above, `test/`, `docs/`, the tool lockfiles,
  the committed action bundle, `action.yml`, `Dockerfile`,
  `.github/workflows/`.
- Agent layout: already in place (`AGENTS.md`, `CLAUDE.md`, `.agents/`).

Continuous integration, all blocking on pull requests:

- typecheck, lint, format check, tests with coverage, build
- a check that the committed action bundle is exactly what a fresh build
  produces
- a check that the tool lockfiles are complete, hash-locked, and the same
  for Actions and for the server image
- a job that performs a real on-demand install on a GitHub-hosted runner and
  runs the installed tool's version command
- CodeQL for JavaScript/TypeScript and for the workflows themselves
- a workflow linter and a workflow security scanner
- dependency review, and a check that the npm lockfile is in sync
- server image build, Dockerfile lint, and an image vulnerability scan
- the agent-skill link check from the pre-commit configuration

Repository rules:

- All third-party actions pinned to a commit. Workflows default to no
  permissions and request what each job needs.
- `main` is protected: pull requests only, required checks, at least one
  human approval.
- Automated dependency update pull requests, covering npm, the tool
  lockfiles, actions, and the image base.
- Secret scanning with push protection.

Releases (proposed):

- A release is a tag on a commit of `main`, created by a person. The bundle
  at that commit has already been verified by continuous integration.
- Consumers pin the tagged commit in their copy of the workflow template.
- The server image is built from the tagged commit and published to the
  GitHub container registry with a provenance attestation and a software
  bill of materials.

Testing against real services:

- Automated tests never call GitHub or an LLM. They use recorded responses
  and the stand-in executable. The install job above contacts the package
  index and nothing else.
- End-to-end checks against a real repository and a real LLM endpoint are
  run by hand before a release, following a written checklist. (Proposed.)

## Prerequisites that need a person

Agents cannot do these. They block the steps noted.

- Turn on branch protection, secret scanning with push protection, and code
  scanning for `grAItools/repo-bot`. Blocks: the repository rules above.
- Allow the repository to publish packages to the container registry.
  Blocks: publishing the server image.
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
- The bundler for the action.
- The installer used for on-demand installs, the lockfile format, and how
  the pinned runtime is obtained, within the rules in "Tools and
  provisioning".
- The choice of workflow linter, workflow security scanner, Dockerfile
  linter, and image scanner.
- Base images, as long as they are pinned by digest and small.
- Log format, as long as it is structured and masks secrets.
- Queue size, worker count, and the install and PR-Agent time limits, as
  documented defaults that a deployment can change.
- Wording of help text and failure comments.
- The order of work, to be set out in the implementation plan. The plan
  should start with the repository tooling and checks, then the command
  framework with `/help` as a working Actions deployment, then tool
  provisioning and PR-Agent, then the server deployment, then release.

## Calls made in this document

The maintainer should check these while reading.

1. `/help` is included.
2. Version 1 commands take no arguments, and extra text is rejected.
3. Unknown commands are ignored silently; refused commands get a reaction
   only.
4. Required permission defaults to `write` and cannot be lowered.
5. Repository configuration can only enable, disable, and tighten. LLM
   settings are deployment-only.
6. The LLM endpoint is any OpenAI-compatible `https` URL, with CSCS as the
   documented default.
7. The server uses an in-process queue with no external store.
8. TypeScript, Node 24, npm, Vitest, ESLint, Prettier.
9. `main` requires a human approval on every pull request.
10. OpenSpec is not used; Superpowers design documents and plans are the
    record of intent.
11. PR-Agent's repository settings file is off, and consuming repositories
    cannot customise PR-Agent in version 1.
12. `/pr:review` only posts a comment. LLM-written comment text based on
    fork content is an accepted risk; the bot never acts on it.
13. The server requires an allow-list of account owners.
14. On the server, one active run per command per pull request. In Actions,
    no limit. No other rate limit.
15. On-demand installs are hash-locked, prebuilt packages only, with a pinned
    runtime, no cache, no secrets in the install environment, and only after
    authorisation.
16. The server never installs at run time; its image is built from the same
    lockfiles.
17. The Actions deployment supports GitHub-hosted Linux x86-64 runners only.
18. Start-up targets: 5 seconds for `/help`, 60 seconds to install PR-Agent,
    measured during planning.
19. A release is a tag created by a person; the server image is built from
    the tagged commit.
20. End-to-end checks are manual, from a checklist, before each release.
21. Consumers copy a short workflow template; there is no reusable workflow
    in version 1.

## Open questions

None of these blocks the implementation plan or the command framework. Each
names what it does block.

1. **What may `/pr:describe` change?** PR-Agent's describe tool can rewrite
   the pull request title and body and add labels. Should it overwrite the
   author's title and description, or post its description as a comment and
   leave them alone? Blocks: building `/pr:describe`.
2. **Which CSCS model, and what is its input size limit?** The bot needs a
   model name and a maximum input size to configure PR-Agent. Blocks: the
   documented default and the end-to-end checklist.
3. **Can GitHub-hosted runners reach the CSCS endpoint?** If it is only
   reachable from certain networks, the Actions deployment cannot use it
   from GitHub's runners, and the design needs either self-hosted runners or
   a different default provider for that deployment. Blocks: the documented
   default for Actions.
4. **Who provides the LLM key for other organisations' repositories?** The
   design assumes each consuming repository supplies its own endpoint and
   key. If the intent is for grAItools to provide CSCS access to others, that
   is a different and much riskier design. Blocks: nothing, if the assumption
   holds.
5. **Who will do the items under "Prerequisites that need a person", and
   when?** They include registering the GitHub App and providing a sandbox
   repository and an LLM key for testing. Blocks: as listed in that section.
