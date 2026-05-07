# Workbench CLI

The `workbench` CLI is cloud-first. Normal commands target hosted Workbench projects; local execution is an explicit development mode under `workbench dev`.

The public model is intentionally small:

- project: one hosted Workbench project
- source directory: local `workbench.yaml`, `candidate/`, `tasks/`, optional runtime Dockerfile such as `environment/Dockerfile`, and optional project-contained adapter sources
- origin: `.workbench/origin.json`, which binds a source directory to `username/project`
- public project: a project with `visibility: public`, addressable as `username/project`
- star: a user marker on a public project
- fork: a private project created from the current public source
- contribution: a PR-like proposal from a fork back to its upstream public project
- candidate: a `skill`, `pipeline`, or advanced `command` under `candidate/`
- runtime: Dockerfile runtime selected by `runtime.dockerfile`
- run: a hosted `eval`, `improve`, or `compare` workflow
- candidate snapshot: generated hosted files that can be inspected, previewed, and pulled

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
workbench login [--base-url URL] [--no-open] [--json]
workbench logout [--json]
workbench whoami [--dir DIR] [--json]
workbench push [--dir DIR] [--project PROJECT] [--dry-run] [--json]
workbench clone USER/PROJECT [DIR] [--dry-run] [--json]
workbench pull [--dir DIR] [--dry-run] [--json]
workbench publish [--dir DIR] [--project PROJECT] [--json]
workbench unpublish [--dir DIR] [--project PROJECT] [--json]
workbench fork USER/PROJECT [DIR] [--name NAME] [--json]
workbench star USER/PROJECT [--json]
workbench unstar USER/PROJECT [--json]
workbench contribute [--dir DIR] [--title TEXT] [--body TEXT] [--json]
workbench eval [--dir DIR] [--project PROJECT] [--candidate ID] [--samples N] [--no-push] [--watch] [--dry-run] [--json]
workbench improve [--dir DIR] [--project PROJECT] [--budget N] [--samples N] [--no-push] [--watch] [--dry-run] [--json]
workbench compare [--dir DIR] [--project PROJECT] [--candidate ID] [--samples N] [--no-push] [--watch] [--dry-run] [--json]
workbench watch [RUN_ID] [--dir DIR] [--project PROJECT] [--interval-ms N] [--timeout-ms N] [--json]
workbench init [DIR] --skill NAME --agent ADAPTER [--from PATH] [--example] [--json]
workbench init [DIR] --pipeline NAME --agent ADAPTER [--from PATH] [--example] [--json]
workbench init [DIR] --command NAME [--from PATH] [--example] [--json]
workbench check [--dir DIR] [--json]
workbench adapters add SOURCE [--dir DIR] [--json]
workbench adapters create PATH [--dir DIR] [--json]
workbench adapters vendor SOURCE PATH [--dir DIR] [--json]
workbench adapters list [--dir DIR] [--json]
workbench adapters inspect ID [--dir DIR] [--json]
workbench adapters update [--dir DIR] [--json]
workbench adapters remove ID|SOURCE [--dir DIR] [--json]
workbench status [--dir DIR] [--project PROJECT] [--json]
workbench open [PROJECT|RUN_ID|CANDIDATE_ID|COMPARISON_ID] [--dir DIR] [--project PROJECT] [--no-open] [--json]
workbench projects list [--json]
workbench projects show [PROJECT] [--dir DIR] [--project PROJECT] [--json]
workbench projects delete [PROJECT] [--dir DIR] [--project PROJECT] [--dry-run] [--json]
workbench envs list [--dir DIR] [--project PROJECT] [--json]
workbench envs create --name NAME --dockerfile PATH|- [--dir DIR] [--project PROJECT] [--cpu N] [--memory-gb N] [--disk-gb N] [--timeout-minutes N] [--network off|on] [--dry-run] [--json]
workbench runs list [--dir DIR] [--project PROJECT] [--json]
workbench runs show [RUN_ID] [--dir DIR] [--project PROJECT] [--json]
workbench runs cancel RUN_ID [--dir DIR] [--project PROJECT] [--json]
workbench candidates list [--dir DIR] [--project PROJECT] [--json]
workbench candidates files CANDIDATE_ID [--dir DIR] [--project PROJECT] [--json]
workbench candidates preview CANDIDATE_ID --path PATH [--dir DIR] [--project PROJECT] [--output PATH|-] [--json]
workbench candidates pull CANDIDATE_ID --out DIR [--dir DIR] [--project PROJECT] [--dry-run] [--json]
workbench logs [RUN_ID] [--dir DIR] [--project PROJECT] [--json]
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
workbench auth connect ADAPTER[/SLOT] [--dir DIR] [--method METHOD] [--profile PROFILE] [--profile-root DIR] [--local-only] [--json]
workbench auth disconnect ADAPTER[/SLOT] [--profile PROFILE] [--local-only] [--json]
workbench --help
workbench --version
```

`workbench check` validates the local source projection and summarizes the execution plan without contacting hosted Workbench. JSON output includes `plan` with source file counts, candidate path/editable files, task case count, runtime resources and network policy, objective metric, and proposer/runner/grader adapter summaries.

## Normal Hosted Flow

Create a local source directory, push a hosted project, and run the first
improvement loop. `push` uploads the initial source state.

```bash
workbench login
workbench auth connect codex
workbench init --skill invoice-review --agent codex
workbench push
workbench improve --budget 1 --samples 1 --watch
workbench open --json --no-open
```

The `workbench.yaml` `name` is the hosted project name. It must be unique for
the authenticated username and cannot be changed by pushing a renamed source; create
a new project when you need a new name. The required `description` is shown in
the hosted project dashboard and Overview tab so people can understand what the
project evaluates without opening the YAML.

Hosted push is idempotent. `workbench push`, and the automatic push performed by
hosted `eval`, `improve`, and `compare`, upload one complete source revision:
`workbench.yaml`, candidate files, task files, project-contained adapter files,
the authored Dockerfile source, resources, and network policy. If that source is
unchanged, Workbench does not create a new source revision or Dockerfile
environment version, and the hosted active candidate remains the default basis
for the next run.

Run focused hosted workflows when you need them:

```bash
workbench eval --samples 1 --watch
workbench improve --budget 2 --samples 1 --watch
workbench compare --samples 1 --watch
```

Hosted Workbench may require a positive Workbench credit balance before a run starts. New hosted billing accounts receive a $5 credit grant, and users can add prepaid credit through Stripe top-ups. Workbench does not re-bill the model or harness provider cost; users pay those providers through adapter auth such as OAuth or API keys. Workbench charges a hosted execution service fee for completed hosted jobs under the configured billing formula. Backend provider/resource details are not part of the user-facing bill.

Watched runs wait until the hosted run finishes. Use `--timeout-ms N` only when a script needs an explicit client-side deadline.

For `eval`, `improve`, `compare`, and their `dev` forms, `--samples` is the repeat count for the whole task set. Task count is reported separately in the evaluation detail; a two-task run with `--samples 1` should show `samples 1/1`, not two samples.

Inspect and pull results:

```bash
workbench runs list
workbench runs show <run-id>
workbench runs cancel <run-id>
workbench candidates list
workbench candidates files <candidate-id>
workbench candidates preview <candidate-id> --path workbench-run-summary.md --output -
workbench candidates pull <candidate-id> --out ./best-candidate
```

Use `--project PROJECT` for one-off hosted targeting. Otherwise commands use `.workbench/origin.json`, written by `workbench push` or `workbench clone`. `PROJECT` may be the internal `wb_...` id or an owner-visible project name when the API can resolve it; public refs use `username/project`.

Publish and collaborate on benchmarks:

```bash
workbench publish
workbench clone alice/invoice-review ./invoice-review
workbench star alice/invoice-review
workbench fork alice/invoice-review ./invoice-review-fork
workbench contribute --title "Improve grader coverage"
```

`workbench publish` makes the origin project public at `/projects/:username/:project`. Public projects can be cloned, starred, and forked. `workbench clone` downloads the current source, and `workbench pull` reconciles the current source from the local origin. `workbench fork` creates a private hosted copy, materializes a writable local checkout with upstream metadata, and `workbench contribute` opens a lightweight proposal from the fork's current source revision back to that upstream.

Delete hosted projects explicitly:

```bash
workbench projects delete <project-name> --dry-run
workbench projects delete <project-name>
```

If the deleted project is the local origin project, the CLI removes `.workbench/origin.json` after the hosted delete succeeds.

Hosted `push`, `publish`, `fork`, `eval`, `improve`, `compare`, `watch`, `runs show`, and `runs cancel` outputs include Workbench Web URLs. In JSON output they live under `urls`. In text output the CLI prints the most useful next URL, such as `Open project:`, `Open run:`, `Open evaluation:`, or `Open comparison:`.

Hosted `--watch` commands and `workbench watch` are client-side polling only. Stopping the command does not cancel the hosted run; use `workbench runs cancel RUN_ID` to cancel a hosted run.

Use `workbench open --json --no-open` to resolve a project, run, candidate, or comparison to a Workbench Web route without launching a separate operating-system browser:

```bash
workbench open --json --no-open
workbench open <run-id> --json --no-open
workbench open <candidate-id> --json --no-open
```

The returned URL uses `/projects/:username/:project`; `username/project` opens a public project, internal project ids are still accepted for private owner workflows, run ids resolve to the candidate evaluation task-review modal with the Trace tab active once a task trace exists, candidate ids open candidate evaluation, and comparison ids open a comparison route. Use `workbench open` without `--no-open` only when the operating-system browser is the intended target.

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

Successful `workbench dev eval`, `workbench dev improve`, and `workbench dev compare` runs finish and exit. Text output prints an `Open local view:` command and lifecycle note; JSON output returns the same information at `localView.command` and `localView.note`.

Use `workbench dev open` directly to visualize existing local development state from `.workbench/runtime` in a read-only browser. It starts a small local server for the selected `--dir`, uses the same Workbench workspace UI as hosted Workbench Web, and must stay running while the local web view is in use; stopping or killing it stops the server and the page will stop working. It does not contact the hosted API. The project Files tab mirrors the source projection that `workbench push` uploads: `workbench.yaml`, `candidate.path`, `tasks.path`, project-contained adapter files, and the authored `runtime.dockerfile`, not arbitrary root-level repo files. Use `workbench open` only for hosted project URLs.

Prefer `workbench init --skill` when the candidate is unclear or when the thing being improved is agent-facing instructions, prompts, examples, or a reusable workflow skill. Use `workbench init --pipeline` when the pipeline itself is the candidate. Use `workbench init --command` only when the command-line workflow itself is the candidate. Specs can mix adapters across phases; choose `run` and `grade` adapters independently from the candidate type. Default to rubric grading for qualitative scoring, and use command grading only for objective deterministic checks, an existing scorer, or an explicitly requested programmatic grading contract.

## Environments

List built-in and project-specific runtimes:

```bash
workbench envs list
workbench envs list --project <project-name>
```

Create a manual hosted Dockerfile environment:

```bash
workbench envs create --project <project-name> --name invoice-runtime --dockerfile ./Dockerfile --memory-gb 4 --disk-gb 10 --timeout-minutes 20 --network off --json
```

Specs normally declare `runtime.dockerfile`; hosted source push stores that authored Dockerfile in the source revision and creates or reuses the hosted workflow runtime from the composed Dockerfile after adapter setup commands are applied. Project-contained adapter sources are uploaded as part of the source revision and are included in source export/pull. Missing `runtime.network` means open outbound egress. Use `egress: none` for offline runs, or `egress: allowlist` with an explicit `allow` list of hosts or host:port entries when you want sandbox-level restriction. Adapter auth never infers or rewrites that policy. Use `workbench envs create` directly only when you want to manage a manual hosted project environment outside source push; it does not switch the source-pushed workflow runtime.

## Adapters

Built-in adapters are available by id: `codex`, `claude`, `pi`, `command`, and `rubric`. Local adapters live inside the project source root and declare `workbench.adapter.yaml`.

```bash
workbench adapters create adapters/my-agent
workbench adapters add ./adapters/my-agent
workbench adapters inspect my-agent --json
workbench adapters update
```

Adapter setup commands run on top of the image from `runtime.dockerfile`. The Dockerfile should contain task and workflow dependencies; adapter manifests own adapter runtime dependencies such as agent CLIs. Adapter manifest `command` is run exactly as a shell command string; omit it only when setup installs the default `workbench-adapter-<id>` command.

Minimal base images can omit native CA certificates. If your runtime uses networked adapters or HTTPS tools, install `ca-certificates` in the Dockerfile; `workbench init` templates include it by default.

## Auth And API Target

Default login targets the hosted Workbench control plane:

```bash
workbench login
workbench whoami
workbench auth connect codex
```

For local Workbench Web development:

```bash
workbench login --base-url http://127.0.0.1:3000 --no-open
WORKBENCH_API_URL=http://127.0.0.1:3000 workbench projects list
```

The CLI resolves the API base URL from `WORKBENCH_API_URL`, then `.workbench/origin.json`, then the login config, then `https://v2.workbench.ai`.

Adapter auth is explicit and method-based. Every adapter, built-in or external, declares auth methods in its adapter manifest. Env-backed methods store only manifest-declared env names from the current environment, file-backed methods copy only manifest-declared files from `--profile-root`, and command-backed methods must print one JSON object with `env` and/or `files`. For example, `OPENAI_API_KEY=... workbench auth connect codex --method api-key` stores only `OPENAI_API_KEY`, and `ACME_API_KEY=... workbench auth connect acme --dir . --method api-key` stores only the declared `ACME_API_KEY`. Multi-slot adapters use `adapter/slot`, such as `workbench auth connect deployer/github --method oauth --profile prod`; if YAML omits `auth`, each declared slot uses profile `default`. When the CLI is logged in to Workbench, adapter auth is uploaded for hosted runs and `workbench whoami --dir .` reports both local and hosted status, including auth required by manifest-declared nested adapter refs. Adapter auth profile names may be referenced from YAML with `auth`, but default profiles are implicit for adapters that declare auth and secret material is never stored in `workbench.yaml`.

## Spec And Eval Authoring

Specs remain first-class for advanced workflows. The shared local/hosted validator accepts only the version-12 shape: `version: 12`, `name`, `description`, `candidate.path`, `candidate.editable`, `tasks.path`, `runtime.dockerfile`, optional `adapters`, `objective`, `propose`, `run`, and `grade`. The required `description` is human project context for Workbench Web; keep optimizer instructions in `objective.goal`. The candidate is the mutable source under improvement; evals live in `tasks`, `run`, and `grade`. Task roots may contain only `task.yaml`, optional `input/`, and optional `expected/`; `task.yaml` is control-plane text, runners see optional `/workspace/input/task/input`, and graders also see optional `/workspace/input/task/expected` plus runner output files at `/workspace/input/runner-output`. Built-in agent adapters such as `codex`, `claude`, and `pi` are direct adapter ids; custom local, npm, or git adapters use the same shape. The built-in command adapter uses `use: command` with one `with.command` string and no `prepare`, `cwd`, or custom `env` fields. External adapters are declared under `adapters` and referenced by manifest `id`; `npm:` and `git:` sources resolve into exact `workbench.lock` entries. Harness defaults are owned by adapters rather than by project YAML.

Use [`evals/spec-syntax.md`](evals/spec-syntax.md) for the complete shared YAML shape and [`evals/runner-contract.md`](evals/runner-contract.md) for the proposer, runner, grader, staged input, and output-file contract. Use [`evals/from-existing-workflow.md`](evals/from-existing-workflow.md) when wrapping an existing benchmark or test command, and [`evals/from-file-outputs.md`](evals/from-file-outputs.md) when tasks or outputs involve `.docx`, `.xlsx`, `.pdf`, `.pptx`, or similar files.
