# Verification findings for the repo-bot design

Date: 2026-11-23

These are the results of the eight checks listed under "To verify during
planning" in [the design](2026-11-23-repo-bot-design.md). The implementation
plans cite this document by item number.

## How to read this document

Each statement carries one of three marks:

- **Observed**: I ran it and saw the result. The method is given.
- **Read**: taken from source code or documentation, not run.
- **Not verified**: nobody has checked it. Each has a task in a plan.

Everything about PR-Agent refers to version 0.50.0 from PyPI, installed from
prebuilt packages on CPython 3.12.13, Linux x86-64. Line numbers are from
that wheel. A different version needs the checks repeated.

## Summary

| Item | Result |
|------|--------|
| 1. Configuration inputs | The design names seven inputs. There are more. All found are closed by a fixed set of settings, listed below. Observed for the important ones. |
| 2. LLM setting names | `config.model`, `openai.api_base`, `openai.key`, `config.custom_model_max_tokens`, `config.max_model_tokens`. Observed, except that the size limit bounds the prompt. |
| 3. GitHub token setting | `github.user_token` with `github.deployment_type = "user"`. Observed. |
| 4. Text with publishing off | Works for `review` and `describe` through a wrapper. No write call reaches GitHub. Observed. |
| 5. Prebuilt packages | Available, but the package set in the design is wrong: 100 packages, not 95. Observed. |
| 6. Pinned runtime and install time | About 4 seconds on a GitHub-hosted runner, against a 60 second target. Observed. |
| 7. Outbound connections | With two environment variables and the fixed settings, a run contacts the GitHub API and the LLM endpoint only. Observed for one small pull request. |
| 8. Token permissions | GitHub's documentation says reacting to a comment needs `issues: write`. Read, not yet tried with a real token. |

Nothing found contradicts a Decided item, one of the 26 proposals, or an
anti-goal. Three statements in the design are inaccurate and are listed
under "Corrections to the design".

## Method

A scratch directory outside the repository held:

- PR-Agent installed with `uv pip install --require-hashes --only-binary :all:`
  from a lockfile generated for `pr-agent[github]==0.50.0`.
- A stand-in GitHub API and a stand-in OpenAI-compatible endpoint on the
  loopback interface. Both logged every request. The GitHub stand-in answered
  the read calls PR-Agent makes for one pull request with one changed file,
  and answered every non-GET request with 403.
- A prototype wrapper script that sets PR-Agent's settings, calls
  `PRAgent().handle_request(...)`, and writes the result to a file.
- The wrapper was run with `env -i` and an allow-list of variables, under
  `strace -f -e trace=connect`, so every connection attempt was recorded.

The stand-in pull request had hostile content in the places an attacker
controls: a `.pr_agent.toml` that sets `publish_output = true` and extra
instructions, an `AGENTS.md`, and a linked issue.

None of this code is in the repository. The plans rebuild the parts that
become tests.

Two read-only source audits (configuration inputs, and outbound
connections) were done by separate agents. Their claims are marked Read
unless a run confirmed them.

## Item 1: configuration and prompt-context inputs

### What the defaults do (Observed)

With the wrapper's settings for repository files and context files left at
PR-Agent's defaults, one `review` run:

- fetched `.pr_agent.toml` and `AGENTS.md` from the repository
- put the hostile text from both into the prompt sent to the LLM
- took `publish_output = true` from the repository's file, and then tried to
  write: two `POST .../issues/7/comments`, one `PUT .../issues/7/labels`,
  one `POST /graphql`

So a repository file can turn publishing back on. `config.publish_output` is
not one of PR-Agent's host-only settings (Read: `pr_agent/config_security.py`).
Turning repository settings off is what protects the publishing setting.

### Inputs found beyond the design's seven

| Input | Setting | Default | How it was checked |
|-------|---------|---------|--------------------|
| Linked issues. Issues named by `#N` or a URL in the pull request title or description, or by a number in the branch name, are fetched and added to the prompt. Applies to `describe` as well. | `pr_reviewer.require_ticket_analysis_review`, `config.extract_issue_from_branch` | `true`, `true` | Observed: with the default, the run fetched `GET /repos/owner/repo/issues/5`. With it off, no such request. |
| Earlier review findings, parsed from existing comments by the bot's login, and replies in resolved review threads | `pr_reviewer.persistent_finding_state`, `pr_reviewer.max_previous_findings_chars` | `true`, `8000` | Read. Also inactive while publishing is off. |
| Skip rules. The pull request's title, labels, author, or branch can make PR-Agent skip the run. | `config.ignore_pr_title` and five related lists | `["^\\[Auto\\]"]`, others empty | Read |
| Skills loaded from host directories | `skills.enabled`, `skills.paths` | `false`, `[]` | Read |
| A continuous integration artifact file added to the prompt | `[artifacts]` section, and `ARTIFACT_PATH`-style variables | off | Read. Only reachable through `cli.py` and the Action runner, which the wrapper does not use. |
| Extra and system instructions | `pr_reviewer.extra_instructions`, `pr_description.extra_instructions`, `config.system_instructions` | empty | Observed through the environment channel, below. |
| Prompt templates, which are ordinary settings sections | `pr_review_prompt.*`, `pr_description_prompt.*` | shipped files | Read. Closed by closing the file and environment channels. |
| A direct read of `PR_AGENT_CONFIG_BRANCH` | environment | unset | Read: `git_providers/git_provider.py:52`. Observed harmless once repository settings are off. |
| `GITHUB_ACTIONS`, which changes whose comments PR-Agent trusts as its own | environment | unset | Read: `git_providers/github_provider.py:994` |
| A `.env` file in the working directory or a parent, loaded by `litellm` on import | `LITELLM_MODE` | loads | Read: `litellm/__init__.py:29` |

One input has no switch: the pull request's branch name is put in the
prompt, together with the title, description, diff, and commit messages.
That is content from the pull request itself, which the design already
accepts as untrusted.

### The environment channel (Observed)

Dynaconf runs with no prefix, so every environment variable of the form
`SECTION__KEY` becomes a setting. With `PR_REVIEWER__EXTRA_INSTRUCTIONS` set
in the child's environment and that setting not fixed by the wrapper, the
text reached the prompt.

When the wrapper fixes a setting, the wrapper wins. A run with
`CONFIG__PUBLISH_OUTPUT=true`, `CONFIG__USE_REPO_SETTINGS_FILE=true`,
`PR_REVIEWER__EXTRA_INSTRUCTIONS`, `PR_AGENT_CONFIG_BRANCH`, and
`GITHUB_ACTIONS=true` all present in the environment still ended with
publishing off, no repository file fetched, no hostile text in the prompt,
and no write call.

The reason is the order of events (Read: `git_providers/utils.py`): PR-Agent
reloads environment variables only after it merges a settings file. With
every file source off, nothing reloads them after the wrapper has set its
values.

### The fixed settings

The wrapper sets these before calling PR-Agent. The effective-settings test
asserts each one after a run, not before.

```
config.git_provider = "github"
config.publish_output = false
config.publish_output_progress = false
config.use_repo_settings_file = false
config.use_global_settings_file = false
config.global_settings_repo = ""
config.use_group_settings_file = false
config.enable_per_directory_settings = false
config.extra_config_url = ""
config.config_branch = ""
config.repo_context_files = []
config.repo_context_sibling_repos = []
config.secret_provider = ""
config.fallback_models = []
config.num_retries = 0
config.retry_same_model_on_timeout = false
config.extract_issue_from_branch = false
config.branch_issue_regex = ""
config.description_issue_regex = ""
config.system_instructions = ""
config.response_language = "en-US"
config.ignore_pr_title = []
config.ignore_pr_labels = []
config.ignore_pr_authors = []
config.ignore_pr_source_branches = []
config.ignore_pr_target_branches = []
config.ignore_repositories = []
config.enable_custom_labels = false
config.enable_ai_metadata = false
github.deployment_type = "user"
github.publish_as_check_run = false
pr_reviewer.require_ticket_analysis_review = false
pr_reviewer.extra_instructions = ""
pr_reviewer.persistent_finding_state = false
pr_reviewer.max_previous_findings_chars = 0
pr_reviewer.structural_reach_input_file = ""
pr_description.extra_instructions = ""
artifacts.enable = false
skills.enabled = false
skills.paths = []
push_outputs.enable = false
otel.is_enabled = false
```

`config.num_retries` and `config.fallback_models` are here for a second
reason. With the defaults, one failing LLM call became ten: five to the
configured model and five to `gpt-6-luna`, PR-Agent's default fallback, sent
to the same endpoint (Observed). With the values above it is one call.

### Not verified

- Dynaconf's own variables ending in `_FOR_DYNACONF`. The child environment
  must not contain them. The allow-list test covers this.
- The prompt template sections were not individually asserted. The plan's
  effective-settings test compares them with the shipped files.
- The list above comes from reading the source of one version. A later
  version can add an input. The design already requires repeating the
  search on every version change.

## Item 2: settings for an OpenAI-compatible endpoint

Observed, against the stand-in endpoint:

| Purpose | Setting | Value |
|---------|---------|-------|
| Model | `config.model` | `openai/<model name>` |
| Base URL | `openai.api_base` | the endpoint's base URL, including `/v1` if the endpoint has it |
| Key | `openai.key` | sent as `Authorization: Bearer <key>` |
| Input size | `config.custom_model_max_tokens` and `config.max_model_tokens` | the model's limit, in tokens |

The request went to `<base URL>/chat/completions` with the model name
without the `openai/` prefix. The request body held `messages`, `model`,
and `temperature`.

Not verified: that the two size settings actually bound the prompt for a
large pull request. `config.max_model_tokens` defaults to 32000 and caps
every model. The plan tests this with a large diff.

Read, not observed: PR-Agent can also call `<base URL>/responses/input_tokens`
to count tokens. It is the same host. An endpoint that does not have it
returns an error that PR-Agent must tolerate; the plan tests this.

## Item 3: passing a GitHub token

Observed: `github.user_token` with `github.deployment_type = "user"`. The
token was sent as `Authorization: token <value>` on every request.
`github.base_url` sets the API address and defaults to
`https://api.github.com`.

## Item 4: the text with publishing off

### It works (Observed)

For both tools, the wrapper got the Markdown text from
`get_settings().data["artifact"]` after `handle_request` returned.

GitHub requests made by a `review` run and by a `describe` run, complete:

```
GET /repos/owner/repo
GET /repos/owner/repo/pulls/7              (twice)
GET /repos/owner/repo/pulls/7/commits      (twice)
GET /repos/owner/repo/languages
GET /repos/owner/repo/pulls/7/files
GET /repos/owner/repo/compare/<base>...<head>
GET /repos/owner/repo/contents/app.py?ref=<head>
GET /repos/owner/repo/contents/app.py?ref=<base>
```

No request with another method. One request to the LLM endpoint.

### What the wrapper must do

These are requirements on the wrapper, each found by a failed attempt or by
reading the code path.

1. Do not go through `pr_agent/cli.py`. Its `run` function sets
   `config.publish_output` to true in plain-diff mode (line 241), reads
   `PR_AGENT_CONFIG_BRANCH` and `PR_AGENT_EXTRA_CONFIG_URL`, and injects the
   artifact context. Call `PRAgent().handle_request` directly.
2. Read one setting from the global settings object before copying it. A
   deep copy of settings that were never read is empty, and the run fails
   with `AttributeError: litellm`. (Observed.)
3. Run inside `request_cycle_context({"settings": <the copy>})`, as
   `cli.py` does, and set the fixed and per-run settings on that copy.
4. Call `handle_request(pr_url, [tool], propagate_tool_errors=True)`. Without
   it PR-Agent logs a failure and returns as if it had succeeded.
5. Treat the run as failed unless `handle_request` returned `True` and the
   artifact is a non-empty string. PR-Agent returns without an artifact when
   the pull request has no files or the model returns nothing.
6. Call PR-Agent's `setup_logger` first. Before that, the logging library's
   default handler prints tracebacks with the values of local variables.
   (Observed in a failing run.)

In `pr_reviewer.py` the artifact is stored when `should_publish` is false.
With `config.publish_output = false` that is always the case, because
`should_publish` is `publish_output and (...)` (line 392). The other exits
from `run` are in requirement 5.

### What the text contains (Observed)

- The `describe` text begins with the pull request's own description,
  copied in, under "User description". Mentions in it are intact.
- Both texts can hold mentions written by the LLM.
- Both hold HTML (`<table>`, `<details>`) and links to
  `https://github.com/<owner>/<repo>/pull/<n>/files#diff-...`.

So the Reporter's mention handling must cover the whole text, including the
copied description.

### An alternative that was not taken

PR-Agent 0.50.0 has a `--json-output <file>` option that works with
`--pr_url`. It writes structured data, not the Markdown, and it only exists
on the `cli.py` path that requirement 1 rules out. The wrapper stays, as
proposal 12 says.

### Not verified

- Failure paths inside PR-Agent that publish. With publishing off, the code
  read guards each `publish_comment` call with `config.publish_output`. The
  runs above included an LLM failure (HTTP 500) and produced no write call.
  Other failures (a malformed model reply, a GitHub error part-way) were not
  tried. The plan's no-write test covers them.
- A pull request with many files. PR-Agent can read files in batches through
  `POST /graphql` (Read: `github_provider.py:1351`). That is a read, but it
  is a POST. The no-write test must tell a GraphQL query from a mutation
  instead of rejecting every POST.

## Item 5: prebuilt packages

Observed:

- `pr-agent==0.50.0` alone does not include PyGithub. The GitHub provider
  fails with "module 'github' is not installed". PyGithub is the optional
  extra `github`.
- `pr-agent[github]==0.50.0` resolves to **100** packages. All are available
  as prebuilt wheels for CPython 3.12 on Linux x86-64
  (`--only-binary :all:` succeeded with `--require-hashes`).
- The five additions are `pygithub`, `pynacl`, `cryptography`, `cffi`, and
  `pycparser`.
- Installed size: 327 MB including the bytecode Python writes on first run.

## Item 6: pinned runtime, hash-checked install, and times

### The method that meets the rules

1. Download the `uv` release archive from GitHub. Check its SHA-256 against a
   committed value.
2. Download a `python-build-standalone` archive. Check its SHA-256 against a
   committed value.
3. `uv venv --python <extracted python>`, then
   `uv pip install --require-hashes --only-binary :all: -r <lockfile>`.

Both downloads are checked by the bot's own code before use, which is what
the design asks for. `uv` can also download Python by itself, but then the
check happens inside `uv`, and I could not confirm it from the binary.
Downloading the archive directly avoids relying on it. The install ran with
`UV_PYTHON_DOWNLOADS=never`.

Values used:

| What | Version | SHA-256 |
|------|---------|---------|
| `uv-x86_64-unknown-linux-gnu.tar.gz` | 0.11.32 | `aab924fd522efd06f1c5f3b93a243864fc453132c94b2dc49f1371b528a4b967` |
| `cpython-3.12.13+20260718-x86_64-unknown-linux-gnu-install_only_stripped.tar.gz` | 3.12.13, release 20260718 | `5854aa6ec71cad00334d5065633c210b2e7feb40956767a59a91791cadcf0b79` |

Both hashes match the checksum files published with the releases.

The install environment was built with `env -i` and held only `PATH`,
`HOME`, `TMPDIR`, `UV_NO_CACHE=1`, `UV_NO_CONFIG=1`, and
`UV_PYTHON_DOWNLOADS=never`. Everything went into a new directory under
`RUNNER_TEMP`.

### Times on a GitHub-hosted runner (Observed)

Runner `ubuntu-24.04`, 4 CPUs, image `20261116`. Workflow run
[45138446380](https://github.com/grAItools/repo-bot/actions/runs/45138446380),
three jobs, 100 packages:

| Job | `uv` download | Python download | Install | Total | First start of PR-Agent | Second start |
|-----|---------------|-----------------|---------|-------|-------------------------|--------------|
| 1 | 0.7 s | 0.8 s | 2.2 s | 3.7 s | 5.0 s | 2.3 s |
| 2 | 0.7 s | 0.9 s | 2.2 s | 3.8 s | 5.1 s | 2.3 s |
| 3 | 0.7 s | 0.9 s | 2.5 s | 4.1 s | 6.2 s | 3.0 s |

The target of 60 seconds for installing PR-Agent is met with a wide margin.

The first start of PR-Agent takes 5 to 6 seconds because Python compiles
the packages. That happens once per run and is outside the install target,
but it is part of what a user waits for.

An earlier run,
[45137611460](https://github.com/grAItools/repo-bot/actions/runs/45137611460),
measured the 95 packages without PyGithub: 3.9 s total.

The workflow that produced these numbers used no actions, no permissions,
and no secrets. It was on this branch for two commits and has been removed.
Its logs expire with GitHub's retention period.

### Not verified

- The target for `/help` (no more than 5 seconds added to the runner's
  start-up). It cannot be measured before the action exists. It is a task in
  plan 1.
- Times under load, or on a day when PyPI or GitHub releases are slow. Three
  jobs on one morning are a small sample.
- Hosts contacted by the install. Expected: `github.com` and its release
  download host, `pypi.org`, `files.pythonhosted.org`. Not traced.

## Item 7: outbound connections of a run

### Observed

With `LITELLM_LOCAL_MODEL_COST_MAP=True` and the fixed settings, `strace`
recorded no connection attempt to any address other than the two loopback
stand-ins, for `review`, for `describe`, and for a `review` whose LLM call
failed.

Without `LITELLM_LOCAL_MODEL_COST_MAP`, the same run looked up a name
through DNS and connected to four addresses in `185.199.108.0/22` on port
443, which is where `raw.githubusercontent.com` is served. This is the
price list `litellm` fetches on import.

### Read

- `tiktoken` does not download in this setup. PR-Agent points
  `TIKTOKEN_CACHE_DIR` at encoding files bundled inside `litellm`
  (`pr_agent/algo/token_handler.py:19-38`). `cl100k_base`, `o200k_base`, and
  `p50k_base` are bundled. PR-Agent asks for `o200k_base` unless the model
  name contains `gpt`.
- A model name that `tiktoken` maps to an encoding that is not bundled
  (`gpt2`, for example) would try to download. Names containing `llama-2`,
  `llama-3`, or `replicate` make `litellm` try to download a tokenizer from
  `huggingface.co`, unless `litellm.disable_hf_tokenizer_download` is true.
- Ticket systems (Jira, Linear, Asana) are contacted only when their tokens
  are set and ticket analysis is on. Both are off.
- Output sinks (`push_outputs`), OpenTelemetry export, and `litellm`
  callbacks are off by default and stay off.
- `litellm` sends LLM requests through `aiohttp` and ignores proxy
  variables unless told otherwise.

### What the child environment and wrapper need

Environment: `LITELLM_LOCAL_MODEL_COST_MAP=True`, `LITELLM_MODE=PRODUCTION`
(stops the `.env` load), `LITELLM_LOCAL_ANTHROPIC_BETA_HEADERS=True`,
`HF_HUB_OFFLINE=1`. Not present: `TIKTOKEN_CACHE_DIR`, `DATA_GYM_CACHE_DIR`,
`CUSTOM_TIKTOKEN_CACHE_DIR`.

Wrapper: `litellm.disable_hf_tokenizer_download = True`. The working
directory is a new, empty directory.

Settings validation: reject a model name that would select an encoding
that is not bundled.

### Not verified

- A run on a large pull request, or one that takes a code path the small
  one did not. The plan's test asserts the connection list on every
  lockfile change.
- Whether `tokenizers` honours `HF_HUB_OFFLINE` on the `from_pretrained`
  path. The `litellm` switch is the control that was read.
- On the server, PR-Agent writes bytecode and may write cache files.
  Where it writes under a read-only root filesystem was not examined. Plan 3
  covers it.

## Item 8: token permissions for the bot's own calls

Read, from GitHub's REST documentation on 2026-11-23:

| Call | Permission the documentation requires |
|------|---------------------------------------|
| Get a user's repository permission | Metadata: read |
| Get a pull request | Pull requests: read |
| Get repository content (`.github/repo-bot.yml`) | Contents: read |
| Create an issue comment (on a pull request) | Issues: write **or** Pull requests: write |
| Create a reaction on an issue comment | Issues: write |

By the documentation, `pull-requests: write` is enough to comment but not to
react. The design adds `issues: write` "only if planning shows that reacting
to, or commenting on, a pull request comment needs it". The documentation
says reacting does.

Not verified: none of this was tried with a real token. Plan 1 grants
`issues: write` and includes a task that tries the reaction with
`pull-requests: write` alone on this repository, so the permission is kept
or dropped on evidence.

## Corrections to the design

These are inaccuracies in statements of fact. None changes a decision.

1. **Package count.** The design says 95 packages and 272 MB in three
   places. For a working GitHub provider it is 100 packages and 327 MB with
   bytecode. The lockfile must be built from `pr-agent[github]`.
2. **Number of inputs.** The design says PR-Agent takes settings or prompt
   text from seven places. Item 1 lists more. The design's rule (each is off
   or pinned, and the effective settings are tested) covers them.
3. **Where the configuration lives.** The design says the environment
   allow-list is what closes the environment-variable input. That is true,
   and it is not sufficient alone: the wrapper must also fix the settings,
   because a repository file can otherwise change `publish_output`. The
   fixed settings live in the wrapper, and the allow-list keeps other
   variables out.

## Still open with the maintainer

Unchanged by this work:

- The CSCS endpoint URL, model name, and input size limit (open questions 1
  and 2).
- Whether a GitHub-hosted runner can reach the CSCS endpoint (open
  question 1).
- The items under "Prerequisites that need a person".
