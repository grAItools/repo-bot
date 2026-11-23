# Repository Tooling and Command Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A checked repository and a working Actions deployment of `repo-bot` whose only command is `/help`, with the whole authorisation path in place and tested.

**Architecture:** A pure decision function (`decide`) receives a read-only view of GitHub and returns what to do. Only after it returns does the `Reporter`, the single part that writes to GitHub, do anything. The Probot app function wires these together and is run once per event by the Actions adapter from a bundled, committed JavaScript file.

**Tech Stack:** Node 24, TypeScript 7.0.2 (strict, ESM), npm, Probot 14.3.7, `@probot/adapter-github-actions` 5.0.1, zod 4.6.7, yaml 2.9.1, Vitest 5.1.1, nock 15.0.6, ESLint 10.15.0 with typescript-eslint 8.73.0, Prettier 3.10.1, `@vercel/ncc` 0.45.0.

**Spec:** [`docs/superpowers/specs/2026-11-23-repo-bot-design.md`](../specs/2026-11-23-repo-bot-design.md). Verification results: [`docs/superpowers/specs/2026-11-23-verification-findings.md`](../specs/2026-11-23-verification-findings.md), cited below as "findings, item N".

This is plan 1 of 4. [`2026-11-23-v1-roadmap.md`](2026-11-23-v1-roadmap.md) says what the other three cover.

## Global Constraints

- Node 24 everywhere. `action.yml` declares `runs.using: node24`.
- TypeScript strict mode, ESM only (`"type": "module"`).
- npm with a committed lockfile. Install scripts disabled: `.npmrc` contains `ignore-scripts=true`.
- Every version in `package.json` is exact, with no `^` or `~`.
- All third-party actions pinned to a commit. Workflows start with `permissions: {}` and each job requests what it needs.
- `main` changes only through pull requests. Never push to `main`.
- No write to GitHub (comment, reaction, label, edit), no download, and no child process before the permission check has passed. A commenter who fails it gets no comment and no reaction.
- Required permission is `write` or `admin`, read from the `permission` field of the collaborator-permission API. `author_association` is not used. The check fails closed.
- Repository configuration is read from `.github/repo-bot.yml` on the default branch only.
- The pull request's code is never checked out.
- No secret appears in a log, a comment, or an error message.
- Automated tests never call GitHub or an LLM.
- Coverage: `src/parser.ts` and `src/decide.ts` at 100% of branches. Whole project at 90% of lines and branches.
- Every rule in `docs/security/threat-model.md` names a test that exists.
- Set `SUPERPOWERS_DISABLE_TELEMETRY=1` for anything the skills start.

## Calls made in this plan

The design leaves these open or silent. The maintainer can overrule any of them in review.

1. A command that the repository has disabled is ignored silently, like an unknown command. The design lists no reply for it.
2. If `.github/repo-bot.yml` cannot be read for a reason other than "not found" (a server error, a rate limit), it is treated like an invalid file: no command runs, and only a commenter with `admin` gets a reply.
3. When several reasons not to run apply, the reply names one, in this order: invalid configuration, extra text, closed pull request.
4. The configuration file is invalid if it is larger than 65536 bytes, or is not a regular file.
5. Schema library: zod. Bundler: `@vercel/ncc`. Workflow linter: actionlint. Workflow security scanner: zizmor.
6. `issues: write` is granted, because GitHub's documentation says reacting to a comment needs it (findings, item 8). Task 14 tries without it and removes it if the reaction still works.
7. The action's exit code is 1 when a command fails or a report cannot be posted, so the workflow run shows the failure.

## Pull requests

The plan is delivered as four pull requests, in this order. Each one is opened only after the previous one has merged.

| Pull request | Tasks | Why separate |
|--------------|-------|--------------|
| A | 1 | The maintainer can run the CSCS check while the rest is built. |
| B | 2, 3, 4 | From here on, every pull request is checked. |
| C | 5 to 11 | The command framework, with no deployment. |
| D | 12, 13, 14 | The Actions deployment and its live checks. |

## Review Focus

Inputs and failures the design implies but does not spell out. Each has a test in the task named.

1. A comment whose first line ends with a carriage return (`/help\r\n`), as browsers on Windows send it. The command runs. Task 6.
2. Reading `.github/repo-bot.yml` fails with a server error or a rate limit. No command runs; an `admin` gets the invalid-configuration reply; anyone else gets nothing. Task 9.
3. The bot's own write is refused (403), for example because the workflow lacks a permission. The remaining reports are still attempted, nothing is thrown, the refusal is logged, and the exit code is 1. Task 10.
4. `.github/repo-bot.yml` is a directory, a symbolic link, a submodule, or larger than 65536 bytes. It is invalid. Task 8.
5. The comment's author is missing from the event (a deleted account). The event is ignored without an exception. Task 9.

## File Structure

```
.github/dependabot.yml                  dependency update configuration
.github/workflows/ci.yml                blocking checks for pull requests
.github/workflows/codeql.yml            code scanning for TypeScript and workflows
.github/workflows/probe-endpoint.yml    manual reachability check for an LLM endpoint
.github/workflows/repo-bot.yml          this repository's copy of the consumer template
.npmrc, .nvmrc, .prettierrc.json, .prettierignore
action.yml                              the published action
dist/                                   committed bundle of src/entrypoints/action.ts
docs/security/threat-model.md           rules, each naming its test
docs/usage/actions.md                   consumer documentation with the workflow template
eslint.config.js, tsconfig.json, vitest.config.ts, package.json, package-lock.json
SECURITY.md
src/parser.ts                           comment body -> command or nothing
src/commands/registry.ts                command types and the registry
src/commands/help.ts                    the /help command
src/repo-config.ts                      parse and validate .github/repo-bot.yml
src/github-reader.ts                    the read-only GitHub calls
src/decide.ts                           filter, parse, config, authorise -> Decision
src/reporter.ts                         the only writes to GitHub
src/task-runner.ts                      TaskRunner interface, inline implementation
src/tools.ts                            ToolProvisioner interface, empty implementation
src/settings.ts                         deployment settings from the environment
src/secrets.ts                          secret masking
src/app.ts                              the Probot app function
src/entrypoints/action.ts               Actions entrypoint
test/                                   one test file per source file, plus:
test/hostile-inputs.test.ts             regression suite of hostile inputs
test/no-write.test.ts                   no write without authorisation
test/architecture.test.ts               structural rules
test/threat-model.test.ts               every rule names an existing test
test/workflow-template.test.ts          documentation template equals the real workflow
test/fixtures/                          recorded GitHub API responses and event payloads
```

---

### Task 1: Endpoint reachability workflow

Answers open question 1 of the design. The maintainer runs it with the CSCS URL.

**Files:**
- Create: `.github/workflows/probe-endpoint.yml`

**Interfaces:**
- Produces: a manually triggered workflow with one input, `url`.

- [ ] **Step 1: Write the workflow**

```yaml
name: probe-endpoint
on:
  workflow_dispatch:
    inputs:
      url:
        description: "https URL to request once, without credentials"
        required: true
        type: string
permissions: {}
jobs:
  probe:
    runs-on: ubuntu-24.04
    timeout-minutes: 2
    steps:
      - name: Request the URL once
        env:
          PROBE_URL: ${{ inputs.url }}
        run: |
          case "$PROBE_URL" in https://*) ;; *) echo "URL must start with https://"; exit 2 ;; esac
          curl --silent --show-error --output /dev/null --max-time 20 --proto '=https' \
            --write-out 'http_code=%{http_code} remote_ip=%{remote_ip} time_total=%{time_total}\n' \
            -- "$PROBE_URL" || echo "curl_exit=$?"
```

The input reaches the shell only through `env`, never by `${{ }}` inside `run`.

- [ ] **Step 2: Check it locally**

Run: `python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/probe-endpoint.yml'))"`
Expected: no output, exit 0.

- [ ] **Step 3: Commit, open pull request A**

```bash
git add .github/workflows/probe-endpoint.yml
git commit -m "Add manual workflow to check that a runner reaches an LLM endpoint"
```

The pull request description asks the maintainer to run the workflow from the Actions tab after merging, with the CSCS base URL, and to report the output line. Any HTTP status, including 401 or 404, means the endpoint is reachable. `curl_exit=6`, `7`, or `28` means it is not.

---

### Task 2: Project scaffold and local checks

**Files:**
- Create: `package.json`, `package-lock.json`, `.npmrc`, `.nvmrc`, `tsconfig.json`, `eslint.config.js`, `.prettierrc.json`, `.prettierignore`, `vitest.config.ts`, `src/secrets.ts`, `test/secrets.test.ts`
- Modify: `.gitignore` (add `node_modules/`, `coverage/`)

**Interfaces:**
- Produces: npm scripts `typecheck`, `lint`, `format:check`, `format`, `test`, `build`, `check`. `check` runs the first three, then `test`.
- Produces: `createMasker(secrets: readonly string[]): (text: string) => string` in `src/secrets.ts`. It replaces every occurrence of each non-empty secret with `***`. Secrets shorter than 4 characters are ignored.

- [ ] **Step 1: Write `package.json`**

`"type": "module"`, `"private": true`, `"engines": { "node": ">=24" }`. Dependencies, exact: `probot` 14.3.7, `@probot/adapter-github-actions` 5.0.1, `zod` 4.6.7, `yaml` 2.9.1. Development dependencies, exact: `typescript` 7.0.2, `vitest` 5.1.1, `@vitest/coverage-v8` 5.1.1, `nock` 15.0.6, `eslint` 10.15.0, `typescript-eslint` 8.73.0, `prettier` 3.10.1, `@vercel/ncc` 0.45.0, `@types/node` 26.6.6.

Scripts:

```json
{
  "typecheck": "tsc --noEmit",
  "lint": "eslint .",
  "format": "prettier --write .",
  "format:check": "prettier --check .",
  "test": "vitest run --coverage",
  "build": "ncc build src/entrypoints/action.ts --out dist --no-cache --license licenses.txt",
  "check": "npm run typecheck && npm run lint && npm run format:check && npm run test"
}
```

`.npmrc`: `ignore-scripts=true` and `save-exact=true`. `.nvmrc`: `24`.

- [ ] **Step 2: Write the tool configuration**

- `tsconfig.json`: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `module` and `moduleResolution` `nodenext`, `target` `es2024`, `noEmit`. Includes `src`, `test`, and the root configuration files.
- `eslint.config.js`: `tseslint.configs.strictTypeChecked` and `stylisticTypeChecked`, with `projectService: true`. Ignores `dist/`, `coverage/`, `.agents/`, `.claude/`.
- `.prettierignore`: `dist/`, `coverage/`, `package-lock.json`, `.agents/`, `.claude/`, `docs/superpowers/`.
- `vitest.config.ts`: coverage provider `v8`, `include: ["src/**"]`, thresholds `lines: 90`, `branches: 90`, and per-file thresholds of `branches: 100` for `src/parser.ts` and `src/decide.ts`. Set `passWithNoTests: false`.

- [ ] **Step 3: Write the failing test `test/secrets.test.ts`**

```ts
import { expect, test } from "vitest";
import { createMasker } from "../src/secrets.js";

test("replaces every occurrence of every secret", () => {
  const mask = createMasker(["ghs_abc123", "sk-xyz789"]);
  expect(mask("a ghs_abc123 b sk-xyz789 c ghs_abc123")).toBe("a *** b *** c ***");
});
test("ignores empty and very short secrets", () => {
  expect(createMasker(["", "abc"])("abc and more")).toBe("abc and more");
});
test("treats secrets as plain text, not patterns", () => {
  expect(createMasker(["a.c$^(x"])("a.c$^(x abcx")).toBe("*** abcx");
});
test("masks the longer secret when one contains another", () => {
  expect(createMasker(["token", "token-long"])("token-long")).toBe("***");
});
```

- [ ] **Step 4: Run it to see it fail**

Run: `npm install && npx vitest run test/secrets.test.ts`
Expected: FAIL, cannot resolve `../src/secrets.js`.

- [ ] **Step 5: Implement `createMasker` in `src/secrets.ts`**

Sort secrets longest first, then `replaceAll` each as a literal string.

- [ ] **Step 6: Run all local checks**

Run: `npm run check`
Expected: typecheck, lint, format check, and tests pass. Coverage thresholds for `src/parser.ts` and `src/decide.ts` are not evaluated yet because the files do not exist.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json .npmrc .nvmrc tsconfig.json eslint.config.js .prettierrc.json .prettierignore vitest.config.ts .gitignore src/secrets.ts test/secrets.test.ts
git commit -m "Add TypeScript project scaffold, local checks, and secret masking"
```

---

### Task 3: Continuous integration

**Files:**
- Create: `.github/workflows/ci.yml`, `.github/workflows/codeql.yml`, `.github/dependabot.yml`

**Interfaces:**
- Produces: check names `check`, `workflows`, `skills`, `dependency-review`, `codeql (javascript-typescript)`, `codeql (actions)`. Task 12 adds `bundle`. The maintainer makes these required when turning on branch protection.

Pinned actions, looked up on 2026-11-23. Re-check each hash against the tag before use.

| Action | Tag | Commit |
|--------|-----|--------|
| `actions/checkout` | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |
| `actions/setup-node` | v7.1.0 | `949feb2413d6458794dcd2491c4babbbce0c15c1` |
| `github/codeql-action` | v4.39.4 | `1bd86f2fdb4b86187fa4059f16f31a3202acc9ea` |
| `actions/dependency-review-action` | v5.0.1 | `bbc44d14a204704ccc493733616cd1e93a0321ec` |
| `zizmorcore/zizmor-action` | v0.9.0 | `bd12b495c90993a4301c73398142281a61499af3` |

actionlint 1.7.12 is downloaded as a release archive, `actionlint_1.7.12_linux_amd64.tar.gz`, and checked against SHA-256 `8aca8db96f1b94770f1b0d72b6dddcb1ebb8123cb3712530b08cc387b349a3d8` before it runs.

- [ ] **Step 1: Write `.github/workflows/ci.yml`**

Triggers: `pull_request`, and `push` to `main`. Top level `permissions: {}`. Every job: `runs-on: ubuntu-24.04`, `timeout-minutes: 10`, `permissions: { contents: read }` unless stated. Every `actions/checkout` step sets `persist-credentials: false`.

| Job | Steps |
|-----|-------|
| `check` | checkout; setup-node with `node-version-file: .nvmrc`; `npm ci`; `npm run check` |
| `workflows` | checkout; download and hash-check actionlint, run `./actionlint -color`; `zizmorcore/zizmor-action` with `advanced-security: false` and `persona: pedantic` |
| `skills` | checkout; `.agents/scripts/sync-skills.sh` |
| `dependency-review` | only on `pull_request`; checkout; `actions/dependency-review-action` with `fail-on-severity: moderate` |

`npm ci` fails when `package-lock.json` and `package.json` disagree, which is the lockfile check the design asks for.

- [ ] **Step 2: Write `.github/workflows/codeql.yml`**

Triggers: `pull_request`, `push` to `main`, and a weekly `schedule`. One job named `codeql` with a matrix over `language: [javascript-typescript, actions]`, `build-mode: none`, permissions `contents: read` and `security-events: write`. Steps: checkout, `github/codeql-action/init`, `github/codeql-action/analyze`.

- [ ] **Step 3: Write `.github/dependabot.yml`**

Version 2. Ecosystems `npm` and `github-actions`, both at directory `/`, weekly. Plans 2 and 3 add the tool lockfile and the image base.

- [ ] **Step 4: Lint the workflows locally**

Run: download actionlint as in the workflow, then `./actionlint`
Expected: no output, exit 0.

- [ ] **Step 5: Commit and push; read the result**

```bash
git add .github/workflows/ci.yml .github/workflows/codeql.yml .github/dependabot.yml
git commit -m "Add blocking checks: typecheck, lint, tests, workflow scanning, CodeQL"
```

Expected on the pull request: all six checks named above report success. If zizmor reports a finding in `probe-endpoint.yml`, fix the workflow; do not add an ignore.

---

### Task 4: Threat model and security policy

**Files:**
- Create: `docs/security/threat-model.md`, `SECURITY.md`, `test/threat-model.test.ts`

**Interfaces:**
- Produces: the rule table format that later tasks and plans append to.

`docs/security/threat-model.md` has these sections: what is protected (the token, later the LLM key, the repository's content and its pull requests, the LLM budget); who might attack (a commenter without access, a fork author, a compromised dependency); entry points (comment body, pull request fields, repository configuration file, the environment); accepted risks; and the rule table.

The rule table has exactly these columns:

```markdown
| ID | Rule | Test |
|----|------|------|
| R1 | Secrets are masked before text reaches a log. | `test/secrets.test.ts` > `replaces every occurrence of every secret` |
```

- [ ] **Step 1: Write the failing test `test/threat-model.test.ts`**

It parses the rule table and, for each row, asserts that the file named in the Test column exists and contains a `test(` or `it(` call whose title is the quoted name.

```ts
test("every rule names a test that exists", () => { /* fails listing each missing pair */ });
test("rule IDs are unique and sequential", () => { /* R1, R2, ... with no gap */ });
test("the table has at least one rule", () => {});
```

- [ ] **Step 2: Run it to see it fail**

Run: `npx vitest run test/threat-model.test.ts`
Expected: FAIL, `docs/security/threat-model.md` not found.

- [ ] **Step 3: Write `docs/security/threat-model.md` and `SECURITY.md`**

The threat model starts with rule R1 above. `SECURITY.md` tells reporters to use GitHub's private vulnerability reporting for this repository and not to open a public issue.

- [ ] **Step 4: Run the test**

Run: `npx vitest run test/threat-model.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit, open pull request B**

```bash
git add docs/security/threat-model.md SECURITY.md test/threat-model.test.ts
git commit -m "Add threat model whose rules are checked against existing tests"
```

From this task on, a task that adds a security rule adds its row to the table in the same commit as the test.

---

### Task 5: Settings

**Files:**
- Create: `src/settings.ts`, `test/settings.test.ts`

**Interfaces:**
- Produces:

```ts
export interface BaseSettings { readonly runRef: string }
export class SettingsError extends Error { readonly names: readonly string[] }
export function readBaseSettings(env: Readonly<Record<string, string | undefined>>): BaseSettings;
```

`runRef` is the reference a maintainer can find in the logs. In Actions it is `GITHUB_RUN_ID`. It must match `/^[A-Za-z0-9_-]{1,64}$/`.

- [ ] **Step 1: Write the failing tests**

```ts
test("reads the run reference from GITHUB_RUN_ID", () => {
  expect(readBaseSettings({ GITHUB_RUN_ID: "12345" })).toEqual({ runRef: "12345" });
});
test("a missing or malformed value throws SettingsError naming the variable", () => {
  for (const env of [{}, { GITHUB_RUN_ID: "" }, { GITHUB_RUN_ID: "a b" }, { GITHUB_RUN_ID: "x".repeat(65) }]) {
    expect(() => readBaseSettings(env)).toThrowError(SettingsError);
  }
});
test("the error message contains variable names and never values", () => {
  try { readBaseSettings({ GITHUB_RUN_ID: "secret value" }); expect.unreachable(); }
  catch (e) { expect(String(e)).toContain("GITHUB_RUN_ID"); expect(String(e)).not.toContain("secret value"); }
});
```

- [ ] **Step 2: Run to see them fail.** Run: `npx vitest run test/settings.test.ts`. Expected: FAIL, module not found.
- [ ] **Step 3: Implement `readBaseSettings` with a zod schema.** Build the error from the issue paths only.
- [ ] **Step 4: Run to see them pass.** Expected: PASS.
- [ ] **Step 5: Commit.** Add threat-model rule "Settings errors name variables and never include values" pointing at the third test.

```bash
git add src/settings.ts test/settings.test.ts docs/security/threat-model.md
git commit -m "Add deployment settings with value-free errors"
```

---

### Task 6: Parser

**Files:**
- Create: `src/parser.ts`, `test/parser.test.ts`, `test/hostile-inputs.test.ts`

**Interfaces:**
- Produces:

```ts
export type ParseResult =
  | { readonly kind: "none" }
  | { readonly kind: "command"; readonly id: string; readonly hasExtraText: boolean };
export function parseCommand(body: string): ParseResult;
```

`id` is `name` or `namespace:name`, for example `help` and `pr:review`.

Rules, in order:

1. Take the text before the first `\n`. Remove one trailing `\r` if present. Remove trailing spaces and tabs.
2. If it matches `/^\/(?:[a-z0-9-]+:)?[a-z0-9-]+$/`, the result is a command without extra text.
3. If it matches `/^\/((?:[a-z0-9-]+:)?[a-z0-9-]+)[ \t]+\S.*$/`, the result is that command with `hasExtraText: true`.
4. Anything else is `none`.

Both patterns are linear. Do not add a quantifier inside a group that is itself repeated.

- [ ] **Step 1: Write the failing tests in `test/parser.test.ts`**

```ts
test.each([
  ["/help", "help"], ["/pr:review", "pr:review"], ["/pr:describe\nmore text", "pr:describe"],
  ["/help\r\nmore", "help"], ["/help  \t", "help"], ["/a-b:c-d9", "a-b:c-d9"],
])("%j is the command %s without extra text", (body, id) => {
  expect(parseCommand(body)).toEqual({ kind: "command", id, hasExtraText: false });
});
test.each([["/help me", "help"], ["/pr:review --fast", "pr:review"], ["/help\tx\r\n", "help"]])(
  "%j is the command %s with extra text", (body, id) => {
    expect(parseCommand(body)).toEqual({ kind: "command", id, hasExtraText: true });
  });
test.each([
  "", "help", " /help", "\n/help", "//help", "/", "/Help", "/pr:", "/:review", "/pr::review",
  "/pr:review:x", "/help!", "/pr_review", "text /help", "/help/x", "/help;id", "/help|cat",
])("%j is not a command", (body) => {
  expect(parseCommand(body)).toEqual({ kind: "none" });
});
```

- [ ] **Step 2: Write the failing hostile-input tests in `test/hostile-inputs.test.ts`**

```ts
describe("parser", () => {
  test.each([
    ["full-width slash", "／help"], ["Cyrillic e in the name", "/hеlp"],
    ["zero-width space after the name", "/help​"], ["zero-width joiner inside", "/he‍lp"],
    ["right-to-left override", "/‮help"], ["NUL after the name", "/help\u0000"],
    ["non-breaking space before text", "/help x"], ["line separator", "/help x"],
    ["byte order mark first", "﻿/help"], ["combining mark", "/helṕ"],
    ["vertical tab", "/help\u000Bx"], ["form feed", "/help\u000C"],
  ])("%s is not a command", (_name, body) => {
    expect(parseCommand(body)).toEqual({ kind: "none" });
  });
  test.each(["/help ; rm -rf /", "/help $(id)", "/help `id`", "/help --config=x", "/help | cat", "/help > /etc/passwd"])(
    "%j is recognised only as a command with extra text", (body) => {
      expect(parseCommand(body)).toEqual({ kind: "command", id: "help", hasExtraText: true });
    });
  test("a 65536-character body is parsed in under 50 ms", () => {
    for (const body of ["/help " + "a".repeat(65530), "/" + "a".repeat(65535), "/" + "a:".repeat(32767), "/help" + " ".repeat(65531)]) {
      const start = performance.now(); parseCommand(body); expect(performance.now() - start).toBeLessThan(50);
    }
  });
});
```

- [ ] **Step 3: Run to see them fail.** Run: `npx vitest run test/parser.test.ts test/hostile-inputs.test.ts`. Expected: FAIL, module not found.
- [ ] **Step 4: Implement `parseCommand`** from the four rules.
- [ ] **Step 5: Run to see them pass.** Run: `npm run test`. Expected: PASS, and `src/parser.ts` at 100% branches.
- [ ] **Step 6: Commit.** Add threat-model rules "Only the first line is considered, and it must match the grammar in full" and "Look-alike and invisible characters never form a command", each pointing at one test above.

```bash
git add src/parser.ts test/parser.test.ts test/hostile-inputs.test.ts docs/security/threat-model.md
git commit -m "Add command parser with hostile-input regression suite"
```

---

### Task 7: Command registry, tool and task interfaces, and `/help`

**Files:**
- Create: `src/commands/registry.ts`, `src/commands/help.ts`, `src/tools.ts`, `src/task-runner.ts`, `test/registry.test.ts`, `test/help.test.ts`, `test/task-runner.test.ts`

**Interfaces:**
- Produces, in `src/tools.ts`:

```ts
export type ToolName = "pr-agent";
export type ProvisionResult =
  | { readonly ok: true; readonly path: string }
  | { readonly ok: false; readonly reason: "tool-unavailable" | "timed-out" };
export interface ToolProvisioner { provide(tool: ToolName): Promise<ProvisionResult> }
export const noToolsProvisioner: ToolProvisioner; // always { ok: false, reason: "tool-unavailable" }
```

- Produces, in `src/task-runner.ts`:

```ts
export type Admission = "accepted" | "queue-full" | "already-running";
export interface TaskRunner {
  admit(key: string): Admission;
  run(key: string, task: () => Promise<void>): Promise<void>;
}
export function createInlineTaskRunner(): TaskRunner; // admit always "accepted"; run awaits the task
```

- Produces, in `src/commands/registry.ts`:

```ts
export type Permission = "write" | "admin";
export type FailureReason =
  | "timed-out" | "tool-unavailable" | "provider-unavailable"
  | "misconfigured" | "interrupted" | "internal-error";
export type CommandResult =
  | { readonly kind: "success"; readonly body?: string }
  | { readonly kind: "failure"; readonly reason: FailureReason };
export interface CommandContext {
  readonly owner: string; readonly repo: string; readonly pullNumber: number;
  readonly headSha: string;
  readonly enabledCommands: readonly Command[];
  readonly provisioner: ToolProvisioner;
}
export interface Command {
  readonly id: string;
  readonly requiredPermission: Permission;   // "write" for every version 1 command
  readonly tools: readonly ToolName[];
  readonly summary: string;                  // one line for /help
  run(context: CommandContext): Promise<CommandResult>;
}
export interface Registry { get(id: string): Command | undefined; all(): readonly Command[] }
export function createRegistry(commands: readonly Command[]): Registry; // throws on a duplicate or malformed id
export const defaultRegistry: Registry;      // contains helpCommand only, in this plan
```

- Produces, in `src/commands/help.ts`: `export const helpCommand: Command` and `export function renderHelp(commands: readonly Command[]): string`.

`renderHelp` output, exactly, for the commands `help` and `pr:review`:

```markdown
**repo-bot** commands enabled in this repository:

- `/help`: List the commands enabled in this repository.
- `/pr:review`: <summary of pr:review>

Write a command alone on the first line of a pull request comment.
```

Commands are listed sorted by `id`. The summary of `help` is `List the commands enabled in this repository.`

- [ ] **Step 1: Write the failing tests**

```ts
// test/registry.test.ts
test("get returns a command by id and undefined otherwise", () => {});
test("createRegistry rejects duplicate ids", () => {});
test("createRegistry rejects an id the parser could not produce", () => {
  // for each of "Help", "pr:", "a b", "": expect(() => createRegistry([{...cmd, id}])).toThrow()
});
test("the default registry holds help, which needs no tools and requires write", () => {
  const help = defaultRegistry.get("help");
  expect(help?.tools).toEqual([]); expect(help?.requiredPermission).toBe("write");
});
// test/help.test.ts
test("renderHelp lists commands sorted by id in the fixed format", () => { /* toBe(the exact text above) */ });
test("help returns its text for the enabled commands only", async () => {
  // run with enabledCommands [helpCommand]; expect { kind: "success", body: renderHelp([helpCommand]) }
});
// test/task-runner.test.ts
test("the inline runner admits everything and awaits the task", async () => {});
test("the inline runner lets a task's rejection propagate", async () => {});
```

- [ ] **Step 2: Run to see them fail.** Run: `npx vitest run test/registry.test.ts test/help.test.ts test/task-runner.test.ts`. Expected: FAIL, modules not found.
- [ ] **Step 3: Implement the four modules.** `createRegistry` validates each id against the parser's pattern from Task 6; export that pattern from `src/parser.ts` as `COMMAND_ID_PATTERN` and use it in both places.
- [ ] **Step 4: Run to see them pass.** Expected: PASS.
- [ ] **Step 5: Commit**

```bash
git add src/commands src/tools.ts src/task-runner.ts src/parser.ts test/registry.test.ts test/help.test.ts test/task-runner.test.ts
git commit -m "Add command registry, /help, and the tool and task interfaces"
```

---

### Task 8: Repository configuration

**Files:**
- Create: `src/repo-config.ts`, `test/repo-config.test.ts`
- Modify: `test/hostile-inputs.test.ts`

**Interfaces:**
- Consumes: `Registry`, `Permission` from Task 7.
- Produces:

```ts
export interface CommandConfig { readonly enabled: boolean; readonly requiredPermission: Permission }
export type RepoConfig = ReadonlyMap<string, CommandConfig>;   // one entry per registry command
export type RepoConfigResult =
  | { readonly kind: "ok"; readonly config: RepoConfig }
  | { readonly kind: "invalid" };
export const MAX_CONFIG_BYTES = 65536;
export function parseRepoConfig(text: string | null, registry: Registry): RepoConfigResult;
```

`text` is `null` when the file does not exist; the result is then every command enabled with its own `requiredPermission`.

File format:

```yaml
commands:
  help:
    enabled: true
  "pr:review":
    enabled: false
    permission: admin
```

Schema, strict at every level: the top level may hold only `commands`. Its keys must be ids in the registry. Each value may hold only `enabled` (boolean) and `permission` (`write` or `admin`). An empty file, or one holding only comments, is valid and means defaults. Parse with `yaml`'s `parse(text, { schema: "core", maxAliasCount: 0, uniqueKeys: true })`.

- [ ] **Step 1: Write the failing tests in `test/repo-config.test.ts`**

```ts
test("no file means every command enabled with its default permission", () => {});
test("an empty file and a comment-only file mean defaults", () => {});
test("a command can be disabled", () => {});
test("permission can be raised to admin", () => {});
test("permission write is accepted and changes nothing", () => {});
test.each([
  ["unknown top-level key", "commandz: {}"],
  ["unknown command", "commands: { nope: { enabled: true } }"],
  ["unknown command key", "commands: { help: { enable: true } }"],
  ["permission below write", "commands: { help: { permission: read } }"],
  ["permission none", "commands: { help: { permission: none } }"],
  ["permission maintain", "commands: { help: { permission: maintain } }"],
  ["enabled as a string", 'commands: { help: { enabled: "false" } }'],
  ["enabled as yes", "commands: { help: { enabled: yes } }"],
  ["top level is a list", "- commands"],
  ["top level is a scalar", "true"],
  ["commands is null", "commands:"],
  ["not YAML", "commands: {"],
])("%s is invalid", (_name, text) => {
  expect(parseRepoConfig(text, defaultRegistry)).toEqual({ kind: "invalid" });
});
```

- [ ] **Step 2: Add the hostile configuration tests to `test/hostile-inputs.test.ts`**

```ts
describe("repository configuration", () => {
  test.each([
    ["alias expansion", "a: &a [x, x]\ncommands: *a"],
    ["duplicate keys", "commands: { help: { enabled: false } }\ncommands: { help: { enabled: true } }"],
    ["a custom tag", "commands: !!js/function 'x'"],
    ["a merge key", "base: &b { enabled: true }\ncommands: { help: { <<: *b } }"],
    ["a key named __proto__", "commands: { __proto__: { enabled: true } }"],
    ["two documents", "commands: {}\n---\ncommands: {}"],
    ["larger than the limit", "# " + "x".repeat(MAX_CONFIG_BYTES)],
  ])("%s is invalid", (_name, text) => {
    expect(parseRepoConfig(text, defaultRegistry)).toEqual({ kind: "invalid" });
  });
  test("parsing never throws", () => {
    for (const text of ["\u0000", "﻿", "{".repeat(10000), "- ".repeat(10000)]) {
      expect(() => parseRepoConfig(text, defaultRegistry)).not.toThrow();
    }
  });
});
```

- [ ] **Step 3: Run to see them fail.** Run: `npx vitest run test/repo-config.test.ts test/hostile-inputs.test.ts`. Expected: FAIL, module not found.
- [ ] **Step 4: Implement `parseRepoConfig`.** Check the byte length first. Catch every parser error and return `invalid`. Use `parseAllDocuments` to reject more than one document.
- [ ] **Step 5: Run to see them pass.** Expected: PASS.
- [ ] **Step 6: Commit.** Add threat-model rules "Repository configuration can only disable a command or raise its permission" and "A malformed configuration file is invalid and never throws".

```bash
git add src/repo-config.ts test/repo-config.test.ts test/hostile-inputs.test.ts docs/security/threat-model.md
git commit -m "Add strict repository configuration parsing"
```

---

### Task 9: Reading GitHub, and the decision

**Files:**
- Create: `src/github-reader.ts`, `src/decide.ts`, `test/github-reader.test.ts`, `test/decide.test.ts`, `test/fixtures/` (recorded responses)
- Modify: `test/hostile-inputs.test.ts`

**Interfaces:**
- Consumes: `parseCommand` (Task 6), `Registry`, `Command` (Task 7), `parseRepoConfig`, `RepoConfig` (Task 8).
- Produces, in `src/github-reader.ts`:

```ts
export interface Target { readonly owner: string; readonly repo: string; readonly issueNumber: number }
export interface GitHubReader {
  /** The raw `permission` field. Rejects on any error. */
  getPermission(username: string): Promise<string>;
  /** Rejects on any error. */
  getPull(): Promise<{ readonly state: string; readonly headSha: string }>;
  /** null when the file does not exist (404). Rejects on any other error, on a non-file, and above MAX_CONFIG_BYTES. */
  getConfigText(): Promise<string | null>;
}
export function createGitHubReader(octokit: ProbotOctokit, target: Target): GitHubReader;
```

Calls: `GET /repos/{owner}/{repo}/collaborators/{username}/permission`, `GET /repos/{owner}/{repo}/pulls/{pull_number}`, `GET /repos/{owner}/{repo}/contents/.github/repo-bot.yml` with no `ref`, so GitHub uses the default branch. `getConfigText` accepts only a response whose `type` is `file` and whose `encoding` is `base64`.

- Produces, in `src/decide.ts`:

```ts
export interface CommentEvent {
  readonly action: string; readonly isPullRequest: boolean;
  readonly senderType: string | undefined; readonly authorType: string | undefined;
  readonly authorLogin: string | undefined; readonly viaApp: boolean;
  readonly body: string;
}
export function toCommentEvent(payload: unknown): CommentEvent;   // never throws
export type IgnoreReason =
  | "not-created" | "not-pull-request" | "not-user" | "not-command"
  | "unknown-command" | "disabled" | "permission-denied" | "lookup-failed";
export type RejectReason = "config-invalid" | "extra-text" | "pr-closed";
export type Decision =
  | { readonly kind: "ignore"; readonly why: IgnoreReason }
  | { readonly kind: "reject"; readonly why: RejectReason; readonly commandId: string }
  | { readonly kind: "fail"; readonly reason: "internal-error"; readonly commandId: string }
  | { readonly kind: "run"; readonly command: Command; readonly headSha: string;
      readonly enabledCommands: readonly Command[] };
export function decide(event: CommentEvent, reader: GitHubReader, registry: Registry): Promise<Decision>;
```

`decide` receives a `GitHubReader` and nothing else that can reach GitHub. It cannot write.

The algorithm, which the tests pin:

```
1. action != "created"                         -> ignore not-created
2. !isPullRequest                              -> ignore not-pull-request
3. senderType != "User" or authorType != "User"
   or viaApp or authorLogin missing            -> ignore not-user
4. parseCommand(body) is none                  -> ignore not-command
5. registry.get(id) is undefined               -> ignore unknown-command
   (no API call has been made up to here)
6. config = parseRepoConfig(getConfigText())   ; a rejection counts as invalid
7. required = invalid ? "admin" : config[id].requiredPermission
8. if valid and !config[id].enabled            -> ignore disabled
9. permission = getPermission(authorLogin)     ; a rejection -> ignore lookup-failed
   permission not in none/read/write/admin     -> ignore lookup-failed
   permission below required                   -> ignore permission-denied
10. invalid                                    -> reject config-invalid
11. hasExtraText                               -> reject extra-text
12. pull = getPull()                           ; a rejection -> fail internal-error
    pull.state != "open"                       -> reject pr-closed
13. run, with headSha = pull.headSha and the enabled commands from config
```

Order of permissions: `none` < `read` < `write` < `admin`.

- [ ] **Step 1: Record the fixtures**

Create `test/fixtures/permission-{admin,write,read,none}.json`, `pull-open.json`, `pull-closed.json`, `pull-draft.json`, `pull-locked.json`, `contents-config.json`, `contents-directory.json`, `contents-symlink.json`, `contents-submodule.json`, and `issue-comment-created.json`, each with the shape GitHub's REST documentation gives for the endpoint, cut down to the fields read. `permission-write.json` also has `"role_name": "maintain"`, and `permission-read.json` has `"role_name": "triage"`, to pin that `role_name` is not read.

- [ ] **Step 2: Write the failing tests in `test/github-reader.test.ts`**, with nock:

```ts
test("getPermission returns the permission field and ignores role_name", async () => {});
test("getPermission rejects on 403, 404, 429, 500, a timeout, and a body without the field", async () => {});
test("getPull returns state and head sha", async () => {});
test("getConfigText decodes a file from the default branch", async () => { /* nock asserts no ref query */ });
test("getConfigText returns null on 404", async () => {});
test("getConfigText rejects on 403, 429, and 500", async () => {});                    // Review Focus 2
test.each(["directory", "symlink", "submodule"])("getConfigText rejects a %s", async () => {}); // Review Focus 4
test("getConfigText rejects a file above MAX_CONFIG_BYTES", async () => {});           // Review Focus 4
test("no request uses a method other than GET", async () => {});
```

- [ ] **Step 3: Write the failing tests in `test/decide.test.ts`**, with a hand-written fake `GitHubReader` that counts calls:

```ts
test.each([
  ["edited comment", { action: "edited" }, "not-created"],
  ["deleted comment", { action: "deleted" }, "not-created"],
  ["comment on an issue", { isPullRequest: false }, "not-pull-request"],
  ["sender is a bot", { senderType: "Bot" }, "not-user"],
  ["author is a bot", { authorType: "Bot" }, "not-user"],
  ["author is an organisation", { authorType: "Organization" }, "not-user"],
  ["posted through an App", { viaApp: true }, "not-user"],
  ["author missing", { authorLogin: undefined, authorType: undefined }, "not-user"],  // Review Focus 5
  ["not a command", { body: "hello" }, "not-command"],
  ["unknown command", { body: "/deploy" }, "unknown-command"],
])("%s is ignored as %s without any API call", async (_n, overrides, why) => {
  // expect(decision).toEqual({ kind: "ignore", why }); expect(reader.calls).toEqual([]);
});
test.each(["none", "read"])("permission %s is denied", async () => {});
test.each(["write", "admin"])("permission %s runs /help with the head sha", async () => {});
test.each(["maintain", "triage", "", "WRITE", "owner"])("permission value %j is refused", async () => {});
test("a failed permission lookup is refused", async () => {});
test("admin required: write is denied, admin runs", async () => {});
test("a disabled command is ignored before the permission lookup", async () => {});
test("invalid configuration: admin is told, write gets nothing", async () => {});
test("unreadable configuration is treated as invalid", async () => {});                // Review Focus 2
test("extra text: an authorised commenter is told, an unauthorised one gets nothing", async () => {});
test.each(["closed", "merged"])("a %s pull request is not run and the commenter is told", async () => {});
test.each(["draft", "locked"])("a %s pull request runs", async () => {});
test("a failed pull request read is an internal error for an authorised commenter", async () => {});
test("invalid configuration is reported before extra text, and extra text before closed", async () => {});
test("the enabled commands exclude disabled ones", async () => {});
test("author_association is never read", () => { /* toCommentEvent output has no such field */ });
test("toCommentEvent never throws", () => { /* null, {}, [], "x", payloads missing each object */ });
```

- [ ] **Step 4: Add hostile event tests to `test/hostile-inputs.test.ts`**

```ts
describe("events", () => {
  test("a login with path characters is passed to the reader unchanged and URL-encoded by the client", async () => {
    // login "../../x?y#z": nock sees /collaborators/..%2F..%2Fx%3Fy%23z/permission
  });
  test("a payload with wrong field types is ignored", () => { /* body: 5, issue: "x", comment: null */ });
});
```

- [ ] **Step 5: Run to see them fail.** Run: `npx vitest run test/github-reader.test.ts test/decide.test.ts test/hostile-inputs.test.ts`. Expected: FAIL, modules not found.
- [ ] **Step 6: Implement both modules.** `toCommentEvent` validates the payload with a zod schema and maps a failed parse to an event that step 1 or 3 ignores.
- [ ] **Step 7: Run to see them pass.** Run: `npm run test`. Expected: PASS, `src/decide.ts` at 100% branches.
- [ ] **Step 8: Commit.** Add threat-model rules for: permission read from the API field, fail closed, bots and Apps ignored, edited comments ignored, configuration from the default branch only, invalid configuration falls back to `admin`.

```bash
git add src/github-reader.ts src/decide.ts test/github-reader.test.ts test/decide.test.ts test/hostile-inputs.test.ts test/fixtures docs/security/threat-model.md
git commit -m "Add read-only GitHub access and the authorisation decision"
```

---

### Task 10: Reporter

**Files:**
- Create: `src/reporter.ts`, `test/reporter.test.ts`

**Interfaces:**
- Consumes: `Target` (Task 9), `FailureReason` (Task 7), `RejectReason` (Task 9), `Admission` (Task 7).
- Produces:

```ts
export type NotRunReason = RejectReason | "queue-full" | "already-running";
export interface ReportTarget extends Target { readonly commentId: number; readonly commandId: string; readonly runRef: string }
export interface Reporter {
  accepted(): Promise<void>;                       // "eyes"
  succeeded(body?: string): Promise<void>;         // "rocket", then the comment if body is given
  failed(reason: FailureReason): Promise<void>;    // "confused", then one comment
  notRun(why: NotRunReason): Promise<void>;        // "confused", then one comment
  /** true if any write was refused or failed */
  readonly hadErrors: boolean;
}
export function createReporter(octokit: ProbotOctokit, target: ReportTarget, log: (message: string) => void): Reporter;
```

Calls: `POST /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions` and `POST /repos/{owner}/{repo}/issues/{issue_number}/comments`.

No method throws. A failed write is logged with its HTTP status and the name of the call, never with the response body, and sets `hadErrors`. A failed reaction does not stop the comment that follows.

Comment texts, exactly, with `<id>` the command id and `<ref>` the run reference:

| Case | Text |
|------|------|
| `extra-text` | `` `/<id>` takes no arguments. Nothing was run. `` |
| `pr-closed` | `` `/<id>` was not run: this pull request is closed. `` |
| `config-invalid` | `` `/<id>` was not run: `.github/repo-bot.yml` on the default branch is invalid or could not be read. No command runs until it is fixed. `` |
| `queue-full` | `` `/<id>` was not run: too many commands are waiting. Try again later. `` |
| `already-running` | `` `/<id>` was not run: the same command is already running on this pull request. `` |
| failure | `` `/<id>` failed: <category>. Reference for maintainers: `<ref>`. `` |

Failure categories: `timed-out` is `timed out`, `tool-unavailable` is `tool unavailable`, `provider-unavailable` is `provider unavailable`, `misconfigured` is `misconfigured`, `interrupted` is `interrupted`, `internal-error` is `internal error`.

- [ ] **Step 1: Write the failing tests**, with nock recording the order of requests:

```ts
test("accepted posts an eyes reaction on the command comment", async () => {});
test("succeeded posts a rocket reaction and no comment without a body", async () => {});
test("succeeded posts a rocket reaction and then the body as one comment", async () => {});
test.each([/* the six failure reasons with their categories */])("failed(%s) posts confused and the fixed text", async () => {});
test.each([/* the five not-run reasons with their texts */])("notRun(%s) posts confused and the fixed text", async () => {});
test("a refused reaction does not prevent the comment, is logged, and sets hadErrors", async () => {});   // Review Focus 3
test("a refused comment does not throw, is logged, and sets hadErrors", async () => {});                  // Review Focus 3
test("a network error on a write does not throw", async () => {});
test("the log line for a refused write holds the status and no response body", async () => {
  // nock replies 403 with body containing "SECRET-IN-BODY"; expect logged text not to contain it
});
```

- [ ] **Step 2: Run to see them fail.** Expected: FAIL, module not found.
- [ ] **Step 3: Implement `createReporter`.**
- [ ] **Step 4: Run to see them pass.** Expected: PASS.
- [ ] **Step 5: Commit.** Add the threat-model rule "Failure comments hold a category and a reference, never raw error text".

```bash
git add src/reporter.ts test/reporter.test.ts docs/security/threat-model.md
git commit -m "Add the reporter, the only part that writes to GitHub"
```

---

### Task 11: The app, and the no-write guarantee

**Files:**
- Create: `src/app.ts`, `test/app.test.ts`, `test/no-write.test.ts`, `test/architecture.test.ts`

**Interfaces:**
- Consumes: everything from Tasks 5 to 10.
- Produces:

```ts
export interface AppDeps {
  readonly registry: Registry;
  readonly taskRunner: TaskRunner;
  readonly provisioner: ToolProvisioner;
  readonly settings: BaseSettings;
  /** Called once per handled event with true if a command failed or a report could not be posted. */
  readonly onOutcome?: (failed: boolean) => void;
}
export function createApp(deps: AppDeps): (app: Probot) => void;
```

Behaviour of the `issue_comment.created` handler:

```
decision = decide(toCommentEvent(payload), reader, registry)
ignore  -> log the reason at debug level; nothing else
reject  -> reporter.notRun(why)
fail    -> reporter.failed(reason)
run     -> key = `${owner}/${repo}#${number}:${command.id}`
           admission = taskRunner.admit(key)
           not accepted -> reporter.notRun(admission)
           accepted     -> reporter.accepted()
                           taskRunner.run(key, async () => {
                             result = await command.run(context)     // an exception counts as internal-error
                             success -> reporter.succeeded(result.body)
                             failure -> reporter.failed(result.reason)
                           })
```

The handler never rejects. An exception anywhere after the decision is logged by type and message through the masker and reported as `internal-error`.

- [ ] **Step 1: Write the failing tests in `test/app.test.ts`**, driving a `Probot` instance with `probot.receive` and nock:

```ts
test("/help by a writer: eyes, then rocket, then one comment with the help text", async () => {});
test("/help lists only enabled commands", async () => {});
test("a command that throws is reported as an internal error with the run reference", async () => {});
test("a command that returns a failure is reported with its category", async () => {});
test("queue-full and already-running admissions are reported and the command does not run", async () => {});
test("onOutcome receives false on success and true on failure or on a refused write", async () => {});
test("the handler does not reject when GitHub returns 500 to everything", async () => {});
test("a thrown error containing the token is logged masked and not posted", async () => {});
```

- [ ] **Step 2: Write the failing tests in `test/no-write.test.ts`**

Each case sends one event and asserts on every request nock saw.

```ts
const notAuthorised = [
  ["a reader", "/help", "read"], ["an outsider", "/help", "none"],
  ["a reader with extra text", "/help now", "read"],
  ["a writer when admin is required", "/help", "write" /* config raises help to admin */],
  ["a writer with invalid configuration", "/help", "write" /* config invalid */],
  ["anyone when the permission lookup fails", "/help", "error-500"],
  ["anyone when the permission value is unknown", "/help", "maintain"],
  ["a bot", "/help", "write" /* sender type Bot */],
  ["an unknown command by an admin", "/deploy", "admin"],
  ["a disabled command by an admin", "/help", "admin" /* config disables help */],
  ["an edited comment by an admin", "/help", "admin" /* action edited */],
  ["a comment on an issue by an admin", "/help", "admin"],
];
test.each(notAuthorised)("%s causes no write, no reaction, and no provisioning", async () => {
  // every recorded request has method GET
  // the provisioner spy was never called
  // the task runner spy was never called
});
test("no reaction or comment is requested before the permission lookup has returned", async () => {
  // delay the permission response; assert no POST was seen while it was pending
});
```

- [ ] **Step 3: Write the failing tests in `test/architecture.test.ts`**

```ts
test("only src/reporter.ts calls a writing GitHub method", () => {
  // read every file under src/ except reporter.ts; none matches
  // /\.(create|update|delete|add|remove|set|replace|merge|lock|unlock)[A-Z]\w*\(/ on an octokit value,
  // and none contains "request(" with a method other than GET
});
test("src/decide.ts imports nothing that can write", () => {
  // its imports are exactly: ./parser.js, ./repo-config.js, ./commands/registry.js, ./github-reader.js (types), zod
});
test("nothing under src/ imports node:child_process", () => {});   // plan 2 narrows this to two files
```

- [ ] **Step 4: Run to see them fail.** Run: `npx vitest run test/app.test.ts test/no-write.test.ts test/architecture.test.ts`. Expected: FAIL, `src/app.ts` not found.
- [ ] **Step 5: Implement `createApp`.**
- [ ] **Step 6: Run everything.** Run: `npm run check`. Expected: PASS with all coverage thresholds met.
- [ ] **Step 7: Commit, open pull request C.** Add threat-model rules: "A comment that is not an authorised command causes no write and no provisioning", "Nothing is written before the permission check returns", "Only the reporter writes to GitHub", "Secrets do not reach logs or comments on a failure".

```bash
git add src/app.ts test/app.test.ts test/no-write.test.ts test/architecture.test.ts docs/security/threat-model.md
git commit -m "Wire the command pipeline and prove it writes nothing unauthorised"
```

---

### Task 12: Actions entrypoint, `action.yml`, and the committed bundle

**Files:**
- Create: `src/entrypoints/action.ts`, `action.yml`, `dist/` (build output), `test/action-entrypoint.test.ts`
- Modify: `.github/workflows/ci.yml` (add job `bundle`), `.gitattributes` (mark `dist/**` as generated)

**Interfaces:**
- Consumes: `createApp` (Task 11), `readBaseSettings` (Task 5), `createInlineTaskRunner`, `noToolsProvisioner`, `defaultRegistry` (Task 7), `createMasker` (Task 2).
- Produces: `export async function main(env: NodeJS.ProcessEnv, write: (line: string) => void): Promise<number>`, the exit code. The file's last statement is `process.exitCode = await main(process.env, console.log)`.

`action.yml`:

```yaml
name: repo-bot
description: Slash commands in pull request comments
inputs:
  token:
    description: Token the bot uses for GitHub. The default is the workflow's token.
    default: ${{ github.token }}
    required: false
runs:
  using: node24
  main: dist/index.js
```

`main` does this, in order:

1. Read the token from `INPUT_TOKEN`. If it is non-empty, write `::add-mask::<token>` so the runner masks it in every later log line.
2. `readBaseSettings(env)`. On `SettingsError`, write the message and return 1.
3. Call the adapter's `run(createApp({...}))` with the inline task runner, `noToolsProvisioner`, and `defaultRegistry`.
4. Return 1 if `onOutcome` reported a failure, otherwise 0.

The adapter reads the token from `INPUT_TOKEN` and the event from `GITHUB_EVENT_NAME` and `GITHUB_EVENT_PATH` by itself (read from its source, version 5.0.1). It logs and swallows errors, which is why the exit code comes from `onOutcome`.

A spike on 2026-11-23 bundled Probot 14.3.7 and the adapter with `ncc` 0.45.0 into `dist/index.js` (2.2 MB) plus four small files, and the bundle handled an `issue_comment` event when run outside `node_modules`. It started in 0.2 seconds.

- [ ] **Step 1: Write the failing tests in `test/action-entrypoint.test.ts`**

```ts
test("masks the token before anything else is written", async () => {
  // first line written is "::add-mask::ghs_testtoken"
});
test("returns 1 and names the variable when GITHUB_RUN_ID is missing", async () => {});
test("handles /help from an event file and returns 0", async () => {
  // GITHUB_EVENT_PATH points at test/fixtures/issue-comment-created.json; nock serves the API
});
test("returns 1 when the command fails", async () => {});
test("returns 0 and writes nothing to GitHub for an unauthorised comment", async () => {});
test("the token never appears in what is written, apart from the add-mask line", async () => {});
```

- [ ] **Step 2: Run to see them fail.** Expected: FAIL, module not found.
- [ ] **Step 3: Implement `main` and write `action.yml`.**
- [ ] **Step 4: Run to see them pass.** Expected: PASS.
- [ ] **Step 5: Build and smoke-test the bundle**

Run:

```bash
npm run build
cp -r dist "$(mktemp -d)/dist"   # run the copy, away from node_modules
```

then run the copied `dist/index.js` with `env -i` holding `PATH`, `INPUT_TOKEN=x`, `GITHUB_RUN_ID=1`, `GITHUB_EVENT_NAME=issue_comment`, and `GITHUB_EVENT_PATH` pointing at a fixture whose body is `hello`.
Expected: exit code 0, no network request attempted (the comment is not a command).

- [ ] **Step 6: Add the `bundle` job to `ci.yml`**

Steps: checkout, setup-node, `npm ci`, `npm run build`, then `git diff --exit-code -- dist` and `test -z "$(git status --porcelain -- dist)"`. The job fails if the committed bundle differs from a fresh build in any byte or file.

- [ ] **Step 7: Commit**

```bash
git add src/entrypoints/action.ts action.yml dist .gitattributes .github/workflows/ci.yml test/action-entrypoint.test.ts
git commit -m "Add the Actions entrypoint and its committed, verified bundle"
```

Expected on the pull request: the `bundle` check passes. If it fails with a difference on the runner but not locally, the build is not reproducible; find the cause before going on. Do not regenerate the bundle on the runner.

---

### Task 13: Workflow template and consumer documentation

**Files:**
- Create: `.github/workflows/repo-bot.yml`, `docs/usage/actions.md`, `test/workflow-template.test.ts`
- Modify: `AGENTS.md` (replace the "very beginning" paragraph with the current state and the `npm run check` command), `.agents/README.md` (its "Local checks" section says there is no continuous integration)

**Interfaces:**
- Produces: the template consumers copy.

The template in `docs/usage/actions.md`, in a fenced `yaml` block:

```yaml
name: repo-bot
on:
  issue_comment:
    types: [created]
permissions: {}
jobs:
  repo-bot:
    if: github.event.issue.pull_request && startsWith(github.event.comment.body, '/')
    runs-on: ubuntu-24.04
    timeout-minutes: 5
    permissions:
      contents: read        # read .github/repo-bot.yml from the default branch
      pull-requests: write  # post the result comment
      issues: write         # react to the command comment
    steps:
      - uses: grAItools/repo-bot@<full commit hash>
```

`.github/workflows/repo-bot.yml` in this repository is the same, except that the single step is replaced by two: `actions/checkout` at the pinned commit with `persist-credentials: false`, then `uses: ./`. For `issue_comment`, checkout takes the default branch, never the pull request.

The documentation states: pin to a full commit hash; the `if` only saves runner time and the bot is safe without it; no concurrency group, with the design's reason; `.github/repo-bot.yml` and its two options; how a commenter without permission sees nothing at all.

- [ ] **Step 1: Write the failing test `test/workflow-template.test.ts`**

```ts
test("the documented template equals this repository's workflow except for the steps", () => {
  // parse both as YAML; delete jobs["repo-bot"].steps from each; expect deep equality
});
test("the documented template pins the action to a 40-character hash placeholder or hash", () => {});
test("this repository's workflow never references the pull request head", () => {
  // the text contains none of: "head.sha", "head.ref", "refs/pull", "pull_request_target"
});
test("the template grants exactly contents: read, pull-requests: write, issues: write", () => {});
test("the template has no concurrency key", () => {});
```

- [ ] **Step 2: Run to see it fail.** Expected: FAIL, files not found.
- [ ] **Step 3: Write the workflow and the documentation; update `AGENTS.md` and `.agents/README.md`.** Run `.agents/scripts/sync-skills.sh` and commit any link changes with them.
- [ ] **Step 4: Run to see it pass.** Run: `npm run check`. Expected: PASS.
- [ ] **Step 5: Commit, open pull request D**

```bash
git add .github/workflows/repo-bot.yml docs/usage/actions.md test/workflow-template.test.ts AGENTS.md .agents/README.md
git commit -m "Add the consumer workflow template and run it in this repository"
```

Expected on the pull request: all checks pass, including actionlint and zizmor on the new workflow.

---

### Task 14: Live checks on this repository

These need pull request D merged, because `issue_comment` workflows run from the default branch. They post real comments. Ask the maintainer before posting, and say which pull request will be used.

**Files:**
- Modify: `docs/superpowers/specs/2026-11-23-verification-findings.md` (record results under items 6 and 8), and, depending on step 3, `.github/workflows/repo-bot.yml`, `docs/usage/actions.md`, `test/workflow-template.test.ts`

- [ ] **Step 1: `/help` works**

On an open pull request in this repository, a user with write access comments `/help`.
Expected: an "eyes" reaction, then a "rocket" reaction, then one comment listing `/help`. The workflow run succeeds.

- [ ] **Step 2: Measure the start-up target**

Read the step timings of that run. Record the duration of the `uses: ./` step.
Expected: 5 seconds or less. If it is more, stop and take the numbers to the maintainer; the design says no rule is relaxed without them.

- [ ] **Step 3: Find out whether `issues: write` is needed**

On a branch, remove `issues: write` from `.github/workflows/repo-bot.yml`, and have the maintainer merge it. Comment `/help` again.

- If the reactions still appear: the permission is not needed. Remove it from the template and the test in the same pull request, and record the result.
- If the reactions are refused (the run fails with exit code 1 and the comment still appears): restore the permission and record the result.

- [ ] **Step 4: A commenter without access sees nothing**

From an account with no access to the repository, comment `/help` on the same pull request.
Expected: the workflow runs and succeeds; no reaction and no comment appear. If no such account is available, record this step as not done.

- [ ] **Step 5: Commit the recorded results**

```bash
git add docs/superpowers/specs/2026-11-23-verification-findings.md
git commit -m "Record live results for /help start-up time and token permissions"
```

---

## What this plan does not cover

- `/pr:review`, `/pr:describe`, tool provisioning, the wrapper script, LLM settings: plan 2.
- The server deployment, the bounded queue, the owner allow-list, the container image and its checks: plan 3.
- Releases and the end-to-end checklist: plan 4.
- Branch protection, secret scanning, and code scanning settings: the maintainer, listed in the design under "Prerequisites that need a person". Task 3 gives the check names to require.
