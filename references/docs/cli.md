# Workbench CLI

The `workbench` CLI is cloud-first. Normal commands target hosted Workbench projects; local execution is an explicit development mode under `workbench dev`.

The public model is intentionally small:

- project: one hosted Workbench project
- source directory: local `workbench.yaml`, `candidate/`, `tasks/`, and optional `environment/Dockerfile`
- link: `.workbench/link.json`, which binds a source directory to a hosted project
- candidate: a `skill`, `pipeline`, or advanced `command` under `candidate/`
- environment: Dockerfile runtime selected by `environment.dockerfile`
- run: a hosted `eval`, `improve`, or `compare` workflow
- candidate: generated hosted file snapshot that can be inspected, previewed, and pulled

## Install

Node.js 18+ is required.

```bash
npm install -g @workbench-ai/workbench-cli
workbench --version
```

If npm reports an auth or registry error, configure the package registry your
organization provides for the `@workbench-ai` scope, then rerun the same install
command.

## Commands

```bash
workbench <command> [options]
workbench launch [--dir DIR] [--dry-run] [--json]
workbench link PROJECT_ID [--dir DIR] [--dry-run] [--json]
workbench sync [--dir DIR] [--project ID] [--dry-run] [--json]
workbench pull --out DIR [--dir DIR] [--project ID] [--revision ID] [--dry-run] [--json]
workbench eval [--dir DIR] [--project ID] [--candidate ID] [--samples N] [--no-sync] [--watch] [--dry-run] [--json]
workbench improve [--dir DIR] [--project ID] [--budget N] [--samples N] [--no-sync] [--watch] [--dry-run] [--json]
workbench compare [--dir DIR] [--project ID] [--candidate ID] [--samples N] [--no-sync] [--watch] [--dry-run] [--json]
workbench watch [RUN_ID] [--dir DIR] [--project ID] [--interval-ms N] [--timeout-ms N] [--json]
workbench init [DIR] --skill NAME --agent codex|claude|pi [--from PATH] [--example] [--json]
workbench init [DIR] --pipeline NAME --agent codex|claude|pi [--from PATH] [--example] [--json]
workbench init [DIR] --command NAME [--from PATH] [--example] [--json]
workbench check [--dir DIR] [--json]
workbench unlink [--dir DIR] [--dry-run] [--json]
workbench status [--dir DIR] [--project ID] [--json]
workbench open [RUN_ID|CANDIDATE_ID] [--dir DIR] [--project ID] [--no-open] [--json]
workbench projects list [--json]
workbench projects show [PROJECT_ID] [--dir DIR] [--project ID] [--json]
workbench projects delete [PROJECT_ID] [--dir DIR] [--project ID] [--dry-run] [--json]
workbench envs list [--dir DIR] [--project ID] [--json]
workbench envs create --name NAME --dockerfile PATH|- [--dir DIR] [--project ID] [--cpu N] [--memory-gb N] [--disk-gb N] [--timeout-minutes N] [--network off|on] [--dry-run] [--json]
workbench runs list [--dir DIR] [--project ID] [--json]
workbench runs show [RUN_ID] [--dir DIR] [--project ID] [--json]
workbench candidates list [--dir DIR] [--project ID] [--json]
workbench candidates files CANDIDATE_ID [--dir DIR] [--project ID] [--json]
workbench candidates preview CANDIDATE_ID --path PATH [--dir DIR] [--project ID] [--output PATH|-] [--json]
workbench candidates pull CANDIDATE_ID --out DIR [--dir DIR] [--project ID] [--dry-run] [--json]
workbench logs [RUN_ID] [--dir DIR] [--project ID] [--json]
workbench dev eval [--dir DIR] [--candidate ID] [--samples N] [--json]
workbench dev improve [--dir DIR] [--budget N] [--samples N] [--json]
workbench dev compare [--dir DIR] [--candidate ID] [--samples N] [--json]
workbench dev checkpoint [--dir DIR] [--json]
workbench dev restore [--dir DIR] [--candidate ID] [--dry-run] [--yes] [--json]
workbench dev candidates [--dir DIR] [--json]
workbench dev files [--dir DIR] [--candidate ID] [--json]
workbench dev preview [--dir DIR] [--candidate ID] --path PATH [--output PATH|-] [--json]
workbench dev comparisons [--dir DIR] [--json]
workbench dev open [--dir DIR] [--host HOST] [--port N] [--no-open] [--json]
workbench auth login [--base-url URL] [--no-open] [--json]
workbench auth logout [--json]
workbench auth status [--json]
workbench auth connect PROVIDER [--method METHOD] [--profile-root DIR] [--api-key-env NAME] [--aws-profile NAME] [--aws-region REGION] [--local-only] [--json]
workbench auth disconnect PROVIDER [--local-only] [--json]
workbench --help
workbench --version
```

## Normal Hosted Flow

Create a local source directory, launch a hosted project, and run the first
improvement loop. `launch` uploads the initial source state.

```bash
workbench auth login
workbench auth connect codex
workbench init --skill invoice-review --agent codex
workbench launch
workbench improve --budget 1 --samples 1 --watch
workbench open --json --no-open
```

The `workbench.yaml` `name` is the hosted project name. It must be unique for
your account and cannot be changed by syncing a renamed source; create a new
project when you need a new name.

Hosted sync is idempotent. `workbench sync`, and the automatic sync performed by
hosted `eval`, `improve`, and `compare`, upload one complete source revision:
`workbench.yaml`, candidate files, task files, Dockerfile source, resources, and
network policy. If that source is unchanged, Workbench does not create a new setup
revision or Dockerfile environment version, and the hosted active candidate
remains the default basis for the next run.

Run focused hosted workflows when you need them:

```bash
workbench eval --samples 1 --watch
workbench improve --budget 2 --samples 1 --watch
workbench compare --samples 1 --watch
```

Watched runs wait until the hosted run finishes. Use `--timeout-ms N` only when a script needs an explicit client-side deadline.

Inspect and pull results:

```bash
workbench runs list
workbench candidates list
workbench candidates files <candidate-id>
workbench candidates preview <candidate-id> --path workbench-run-summary.md --output -
workbench pull --project <project-id> --out ./source-copy
workbench candidates pull <candidate-id> --out ./best-candidate
```

Use `--project ID` for one-off hosted targeting or `workbench link PROJECT_ID` to store the project in `.workbench/link.json`.

Delete hosted projects explicitly:

```bash
workbench projects delete <project-id> --dry-run
workbench projects delete <project-id>
```

If the deleted project is the local linked project, the CLI removes `.workbench/link.json` after the hosted delete succeeds.

Use `workbench open --json --no-open` when an agent should keep the user's embedded browser on the matching Workbench Web project without launching a separate OS browser. The returned URL uses `/projects/:projectId`; candidate ids open the candidate evaluation view, and run ids open the candidates view.

## Agent Browser Workflow

Agents should keep the CLI and Workbench Web in sync. Prefer an embedded or in-app browser when the agent host provides one:

```bash
workbench open --json --no-open
workbench open <candidate-id> --json --no-open
```

Navigate the embedded browser to the returned URL after launch/link, sync, hosted eval, hosted improve, hosted compare, and candidate inspection. Use `workbench open` without `--no-open` only when no embedded browser is available and the operating system browser is the best fallback.

After watched workflows, prefer `workbench open <candidate-id> --json --no-open` when the JSON output includes a `candidateId`. Use `workbench open --json --no-open` for project setup and sync states before a candidate exists.

For `eval`, `improve`, `compare`, and their `dev` forms, `--samples` is the repeat count for the whole task set. Task count is reported separately in the evaluation detail; a two-task run with `--samples 1` should show `samples 1/1`, not two samples.

## Local Development Flow

Use `dev` when iterating without the hosted API:

```bash
workbench init --command local-command-eval
workbench check
workbench dev improve --budget 1 --samples 1
workbench dev candidates
workbench dev open
```

Local development commands use `--dir DIR`. They run independent local jobs automatically according to the same generic dependency and resource contract as hosted execution: runner jobs for separate tasks or samples can overlap, and grader jobs start as soon as their runner dependency succeeds. There is no separate parallelism flag; concurrency comes from declared job resources and local host capacity.

Successful `workbench dev eval`, `workbench dev improve`, and `workbench dev compare` runs finish and exit. Text output prints an `Open local view:` command, and JSON output returns the same command at `localView.command`.

Use `workbench dev open` directly to visualize existing direct local development state from `.workbench/runtime` in a read-only browser. It starts a small local server for the selected `--dir`, uses the same Workbench workspace UI as hosted Workbench Web, and does not contact the hosted API. The project Files tab mirrors the source projection that `workbench sync` uploads: `workbench.yaml`, `candidate.path`, `tasks.path`, and `environment.dockerfile`, not arbitrary root-level repo files. Use `workbench open` only for hosted project URLs.

Prefer `workbench init --skill` when the candidate is unclear or when the thing being improved is agent-facing instructions, prompts, examples, or a reusable workflow skill. Use `workbench init --pipeline` when the pipeline itself is the candidate. Use `workbench init --command` only when the command-line workflow itself is the candidate. Specs can mix adapters across phases; choose `run` and `grade` adapters independently from the candidate type. Default to rubric grading for qualitative scoring, and use command grading only for objective deterministic checks, an existing scorer, or an explicitly requested programmatic grading contract.

## Environments

List built-in and project-specific runtimes:

```bash
workbench envs list
workbench envs list --project <project-id>
```

Create a custom Dockerfile environment:

```bash
workbench envs create --project <project-id> --name invoice-runtime --dockerfile ./Dockerfile --memory-gb 4 --disk-gb 10 --timeout-minutes 20 --network off --json
```

Specs normally declare `environment.dockerfile`; hosted source sync creates or reuses the hosted environment from that Dockerfile plus normalized resources and network policy. Missing `environment.network` means open outbound egress. Use `egress: none` for offline runs, or `egress: allowlist` with an explicit `allow` list of hosts or host:port entries when you want sandbox-level restriction. Provider auth never infers or rewrites that policy. Use `workbench envs create` directly when you want to manage a project environment outside sync.

Minimal base images can omit native CA certificates. If your runtime uses networked agent providers or HTTPS tools, install `ca-certificates` in the Dockerfile; `workbench init` templates include it by default.

## Auth And API Target

Default login targets the hosted Workbench control plane:

```bash
workbench auth login
workbench auth status
workbench auth connect codex
```

For local Workbench Web development:

```bash
workbench auth login --base-url http://127.0.0.1:3000 --no-open
WORKBENCH_API_URL=http://127.0.0.1:3000 workbench projects list
```

The CLI resolves the API base URL from `WORKBENCH_API_URL`, then the linked project URL, then the login config, then `https://v2.workbench.ai`.

Agent provider auth is explicit and method-based. `workbench auth connect codex` and `workbench auth connect claude` default to OAuth profile auth. Codex OAuth stores `.codex/auth.json`; Claude OAuth stores `.claude.json` plus a portable Claude `setup-token` OAuth token from `.claude/oauth-token`, `CLAUDE_CODE_OAUTH_TOKEN`, or the interactive `claude setup-token` flow. Use `--method api-key` to store `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`. Use `workbench auth connect claude --method bedrock` to store Bedrock credentials from AWS env vars or `--aws-profile`. When the CLI is logged in to Workbench, the same connection is uploaded for hosted runs. Provider auth is never stored in `workbench.yaml`.

## Spec And Eval Authoring

Specs remain first-class for advanced workflows. The shared local/hosted validator accepts only the version-11 shape: `version: 11`, `name`, `candidate.path`, `candidate.editable`, `tasks.path`, `environment.dockerfile`, `objective`, `propose`, `run`, and `grade`. The candidate is the mutable source under improvement; evals live in `tasks`, `run`, and `grade`. Task roots may contain only `task.yaml`, optional `input/`, and optional `expected/`; `task.yaml` is control-plane text, runners see optional `/workspace/input/task/input`, and graders also see optional `/workspace/input/task/expected` plus runner output files at `/workspace/input/runner-output`. Agent runners are direct provider adapters (`codex`, `claude`, or `pi`); command adapters use `use: command` with one `with.command` string and no `prepare`, `cwd`, or custom `env` fields. Provider harness defaults are owned by the sandbox adapter rather than by project YAML.

Use [`evals/spec-syntax.md`](evals/spec-syntax.md) for the complete shared YAML shape and [`evals/runner-contract.md`](evals/runner-contract.md) for the proposer, runner, grader, staged input, and output-file contract. Use [`evals/from-existing-workflow.md`](evals/from-existing-workflow.md) when wrapping an existing benchmark or test command, and [`evals/from-file-outputs.md`](evals/from-file-outputs.md) when tasks or outputs involve `.docx`, `.xlsx`, `.pdf`, `.pptx`, or similar files.
