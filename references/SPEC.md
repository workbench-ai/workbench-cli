# Workbench CLI Specification

## Status

This is the current product contract for `products/workbench-cli`. The CLI is cloud-first: top-level workflow and resource commands target hosted Workbench projects, while local-only execution is explicitly under `workbench dev`.

## Public Model

Durable resources live in Workbench Web:

- `project`: one hosted Workbench project.
- `source`: the pushed project source revision: `workbench.yaml`, candidate files, task files, the authored runtime Dockerfile, and project-contained adapter sources.
- `public project`: a project with `visibility: public`, addressable as `username/project`.
- `star`: a user marker on a public project.
- `fork`: a private project created from the current public source, with upstream metadata.
- `contribution`: a PR-like proposal from a fork back to its upstream public project.
- `candidate`: a generated hosted file snapshot Workbench can inspect, improve, compare, and pull.
- `runtime`: a Dockerfile runtime selected by source and built/reused by hosted push.
- `run`: a hosted `eval`, `improve`, or `compare` workflow.

Local source directories contain `workbench.yaml`, `candidate/`, `tasks/`, an optional runtime Dockerfile such as `environment/Dockerfile`, optional adapter directories, and an optional `.workbench/origin.json`. The origin file records the hosted project id, `owner/project` ref, API base URL, writable state, source revision, and optional upstream fork metadata. Local source state is disposable; hosted project state is durable.

## Configuration

The CLI resolves the API base URL from `WORKBENCH_API_URL`, then `.workbench/origin.json`, then the login config written by `workbench login --base-url URL`, then `https://v2.workbench.ai`.

`workbench login` uses the Workbench web portal OAuth device flow and requires the signed-in user to have a username. Manual pasted bearer tokens are not part of the product contract. `workbench logout` removes the Workbench access token. `workbench whoami` reports the API target, login state, username, and adapter auth state.

`workbench auth connect ADAPTER[/SLOT]` stores user-scoped adapter auth for local development and hosted runs. Every adapter, built-in or external, declares auth methods in its adapter manifest. Env-backed methods store only manifest-declared env names, profile-file methods copy only manifest-declared files from `--profile-root`, and command methods must print one JSON object containing `env` and/or `files`. Default profiles are implicit when an adapter manifest declares auth, including manifest-declared nested adapter refs; multi-slot adapters default each declared slot to profile `default`. Non-default auth profiles are named with `--profile` and referenced from YAML with `auth`, but secret material is never stored in `workbench.yaml`.

## Commands

Supported commands:

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

Bare `workbench` prints help and exits `0`. Unknown commands print a usage error and exit `2`. Mutating hosted commands that support `--dry-run` do not write origin/output files. `workbench projects delete` is the explicit hosted project deletion path; when it deletes the local origin project, it removes `.workbench/origin.json` only after the hosted delete succeeds. Destructive local restore requires either `--dry-run` or `--yes`.

`workbench init --skill` is the default starting point when the candidate is unclear, and is preferred for agent-facing instructions, prompts, examples, or reusable workflow skills. `workbench init --pipeline` is preferred when the pipeline itself is clearly the candidate. `workbench init --command` is the advanced scaffold only when the command-line implementation itself is the candidate. The candidate is the mutable thing Workbench improves; evals live in `tasks`, `run`, and `grade`. Authored specs may mix adapters across phases; for example, a command runner can use a rubric grader. Default to rubric grading for qualitative scoring, and use command grading only for objective deterministic checks, an existing scorer, or an explicitly requested programmatic grading contract.

`workbench check` is the local source and execution-plan primitive. It validates `workbench.yaml` plus the referenced Dockerfile, candidate directory, tasks directory, and project adapter manifests without contacting hosted Workbench. JSON output contains `plan` with the project name, source projection counts, candidate path/editable files, task case count, runtime resources and network policy, objective metric, and proposer/runner/grader adapter summaries.

`workbench adapters create`, `add`, `vendor`, `list`, `inspect`, `update`, and `remove` are the adapter authoring primitives. There is no alias flag: the manifest `id` is the adapter id. Local adapter sources must live inside the project source root; npm and git adapter sources resolve into exact, hashed `workbench.lock` entries. Adapter setup commands are composed onto the image from `runtime.dockerfile`; the Dockerfile owns task dependencies, while adapter manifests own adapter runtime dependencies. Adapter manifest `command` is a shell command string that Workbench runs exactly; if it is omitted, the command defaults to `workbench-adapter-<id>`.

`workbench open` prints a Workbench Web URL and opens the operating-system browser unless `--no-open` is present. `workbench open --json --no-open` resolves a project, run, candidate, or comparison route without launching a browser. Project identity lives in the route path as `/projects/:username/:project`; public project refs use `username/project`; internal `wb_...` ids are still accepted for private owner workflows; run refs resolve to the candidate evaluation task-review modal with the Trace tab active once a task trace exists, candidate refs open the route-native candidate evaluation view, and comparison refs open a comparison route. Hosted commands that create, push, publish, fork, start, watch, show, or cancel resources include a `urls` object in JSON output and print the most useful URL in text output.

## Hosted Workflows

`workbench push` validates the local source, creates or updates the hosted project from `workbench.yaml`, uploads the authored Dockerfile source, candidate directory, tasks directory, and project-contained adapter sources, and writes `.workbench/origin.json`. If `workbench.yaml`, candidate files, task files, adapter source files, authored Dockerfile source, composed runtime Dockerfile, resources, and network policy are unchanged, push returns the current source revision without creating a new source revision or environment version. The YAML `name` is the project name and public handle under the authenticated username; it is immutable after the project is created.

`workbench clone USER/PROJECT [DIR]` downloads the current public project source into a local directory and writes a read-only origin. `workbench pull` reconciles the current local source directory from its origin: a writable origin pulls by hosted project id, and a cloned public origin pulls from `username/project`. Managed source files that disappeared upstream are removed; unrelated local files outside the managed source projection are preserved. Generated candidate files remain under `workbench candidates pull`.

`workbench publish` and `workbench unpublish` toggle project visibility. Public projects are public benchmarks at `/projects/:username/:project`, can be cloned, starred, and forked, and expose the current source through the public source API.

`workbench fork USER/PROJECT [DIR]` creates a private hosted project owned by the authenticated user from the current public source, materializes or updates a writable local checkout, and writes origin upstream metadata. `workbench star` and `workbench unstar` mutate the authenticated user's star on a public project. `workbench contribute` opens a lightweight contribution from the current fork origin back to its upstream project using the fork's current source revision.

`workbench eval` creates a hosted run with `workflow: "eval"`. It evaluates the requested candidate, the active hosted candidate, or the uploaded candidate snapshot if no candidate exists. It queues runner and grader executions for the top-level spec and does not propose changes.

`workbench improve` creates a hosted run with `workflow: "improve"`. It queues proposer executions up to `--budget`, then runner and grader executions for each proposed candidate, and updates hosted candidate history by greedy selection.

`workbench compare` is a result-inspection workflow over completed evaluations. Authored YAML no longer declares comparison variants; adapter, runtime, runner, and grader choices live in the single top-level spec.

`workbench runs show RUN_ID` is the run detail primitive. JSON output contains `{ run, jobs, urls }`; text output summarizes workflow, candidate, samples, trials, job status, duration, cost when available, first failed job error, and hosted URLs. `workbench runs cancel RUN_ID` is the only public cancellation primitive. It finishes the run with `outcome: "cancelled"` and `stoppedReason: "cancelled"`, marks non-terminal jobs `cancelled`, prevents queued jobs from starting, and leaves project and candidate history intact.

All workflows freeze the current source revision, base candidate files, and task files onto the run input. Each direct child directory under `tasks.path` runs as a standalone task. `task.yaml` is parsed as control-plane task text and is not mounted as a runtime file. Task roots may contain only `task.yaml`, optional `input/`, and optional `expected/`; runners see optional `/workspace/input/task/input`, while graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and runner output files mounted at `/workspace/input/runner-output`. Runtime phases receive only named read-only inputs under `/workspace/input` plus a writable `/workspace/output` directory. Proposers see candidate files and prior run traces. Adapter commands receive `WORKBENCH_ADAPTER_REQUEST` and `WORKBENCH_OUTPUT`; simple command adapter users can ignore those helpers and read the mounted paths directly. Adapters may also write `/workspace/output/.workbench/result.json` with `summary`, `feedback`, and role-aware `usage`; Workbench treats that file as internal metadata, not a user-facing runner artifact.

The `--samples` flag is a repeat count over the frozen task set. A materialized evaluation sample may contain many task case results, but comparison and candidate sample counters count only repeat sample indexes, not task executions.

Execution usage is role-aware. Proposer, runner, and grader jobs may each report provider/model, token breakdown, and dollar cost. Provider-reported cost is authoritative; Workbench estimates from its checked-in LiteLLM price snapshot only when an adapter reports enough tokens but no provider cost. Comparison cost is runner plus grader cost per evaluation sample. Candidate total usage includes proposer cost plus every runner and grader sample cost.

## Local Development

`workbench dev improve`, `workbench dev eval`, and `workbench dev compare` run against a local source directory without hosted API calls. They use `--dir`, share the same `workbench.yaml` contract, store local history under `.workbench/`, and automatically run independent local jobs concurrently according to declared execution resources and local host capacity. No user-facing parallelism knob is exposed. Dev runs do not start a browser server; text and JSON outputs include a `workbench dev open --dir ...` command for starting the read-only local view explicitly.

`workbench dev open` starts a read-only local browser server for local development state. The server reads the selected source directory and `.workbench/runtime`, exposes the same browser DTO shape used by hosted Workbench Web, and serves the shared Workbench workspace UI until the command stops. Its project Files surface mirrors the pushed source projection: `workbench.yaml`, files under `candidate.path`, files under `tasks.path`, project-contained adapter files, and the Dockerfile selected by `runtime.dockerfile`. It must not expose hosted projects, cloud auth, queue or worker internals, arbitrary filesystem paths outside the pushed source projection, or browser mutation endpoints.

## Spec Authoring Contract

The hosted API stores the spec as YAML source. The canonical authoring guide lives under [`docs/evals/`](docs/evals/):

- [`docs/evals/spec-syntax.md`](docs/evals/spec-syntax.md) owns the YAML shape.
- [`docs/evals/runner-contract.md`](docs/evals/runner-contract.md) owns staged runtime directories, phase visibility, runner output files, and the grader scorecard file.
- [`docs/evals/from-existing-workflow.md`](docs/evals/from-existing-workflow.md) owns the workflow-wrapping path.
- [`docs/evals/from-file-outputs.md`](docs/evals/from-file-outputs.md) owns file-output task guidance for `.docx`, `.xlsx`, `.pdf`, `.pptx`, and similar files.

The validator accepts only the version-12 YAML shape: `version: 12`, `name`, `description`, `candidate.path`, `candidate.editable`, `tasks.path`, `runtime.dockerfile`, optional `adapters`, `objective`, `propose`, `run`, and `grade`. The required `description` is human project context shown in Workbench Web; optimizer instructions belong in `objective.goal`. Adapter blocks use `use:` and optional `with:` fields. Built-in agent adapters such as `codex`, `claude`, and `pi` are examples of direct adapter ids; custom local, npm, or git adapters use the same shape. The built-in command adapter uses `use: command` with one `with.command` string and no `prepare`, `cwd`, or custom `env` fields. External adapters are declared under `adapters` and referenced by manifest `id`. All authored file paths are workspace-relative literals and must not be globs. Do not put eval-only rubrics, goldens, expected values, or grader code in the candidate unless that evaluator is itself the product being improved. Adapter auth is resolved from user-scoped adapter connections and harness config is resolved by adapters, not stored in `workbench.yaml`. Workbench agent turns run inside the selected sandbox boundary, so adapter descriptors may set sandbox-appropriate harness defaults without adding authoring knobs.

## Runtime Environments

The runtime environment model is Dockerfile-first. Specs declare `runtime.dockerfile`. Local development builds from the authored Dockerfile as needed, then runs adapter setup commands on top. Hosted source push stores the authored Dockerfile in the source revision, creates or reuses the hosted workflow runtime from the composed Dockerfile, normalized resources, adapter setup, and network policy, and runs use the active built version from that pushed source. `workbench envs create` is the manual hosted project environment primitive for inventory and direct environment management outside source push; it does not switch the source-pushed workflow runtime. Missing `runtime.network` means open outbound egress. To restrict network access, set `runtime.network.egress: none` or set `runtime.network.egress: allowlist` with an explicit `allow` list of hosts or host:port entries. Adapter auth does not infer or rewrite network policy. The Dockerfile is the place to pin task tools such as LibreOffice; adapter setup pins agent or adapter CLIs. Networked adapter CLIs and HTTPS tools also depend on native CA certificates; minimal base images may need `ca-certificates`, and `workbench init` templates include it by default.
