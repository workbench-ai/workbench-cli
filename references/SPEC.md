# Workbench CLI Specification

## Status

This is the current product contract for `products/workbench-cli`. The CLI is cloud-first: top-level workflow and resource commands target hosted Workbench projects, while local-only execution is explicitly under `workbench dev`.

## Public Model

Durable resources live in Workbench Web:

- `project`: one hosted Workbench project.
- `source`: the synced project source revision: `workbench.yaml`, candidate files, task files, and the Dockerfile runtime.
- `candidate`: a generated hosted file snapshot Workbench can inspect, improve, compare, and pull.
- `environment`: a Dockerfile runtime selected by source and built/reused by hosted sync.
- `run`: a hosted `eval`, `improve`, or `compare` workflow.

Local source directories contain `workbench.yaml`, `candidate/`, `tasks/`, optional `environment/Dockerfile`, and an optional `.workbench/link.json`. The link file records the hosted project id and API base URL. Local source state is disposable; hosted project state is durable.

## Configuration

The CLI resolves the API base URL from `WORKBENCH_API_URL`, then the linked project URL, then the login config written by `workbench auth login --base-url URL`, then `https://v2.workbench.ai`.

`workbench auth login` uses the Workbench web portal OAuth device flow. Manual pasted bearer tokens are not part of the product contract.

`workbench auth connect PROVIDER` stores user-scoped agent provider auth for local development and hosted runs. The default method is provider-native OAuth profile auth; `--method api-key` stores Codex or Claude API-key auth; descriptor-provided env methods such as `workbench auth connect claude --method bedrock` store only their allowlisted env bundle. Provider auth is never project-scoped and is never stored in `workbench.yaml`.

## Commands

Supported commands:

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

Bare `workbench` prints help and exits `0`. Unknown commands print a usage error and exit `2`. Mutating hosted commands that support `--dry-run` do not contact the API or write link/output files. `workbench projects delete` is the explicit hosted project deletion path; when it deletes the locally linked project, it removes `.workbench/link.json` only after the hosted delete succeeds. Destructive local restore requires either `--dry-run` or `--yes`.

`workbench init --skill` is the default starting point when the candidate is unclear, and is preferred for agent-facing instructions, prompts, examples, or reusable workflow skills. `workbench init --pipeline` is preferred when the pipeline itself is clearly the candidate. `workbench init --command` is the advanced scaffold only when the command-line implementation itself is the candidate. The candidate is the mutable thing Workbench improves; evals live in `tasks`, `run`, and `grade`. Authored specs may mix adapters across phases; for example, a command runner can use a rubric grader. Default to rubric grading for qualitative scoring, and use command grading only for objective deterministic checks, an existing scorer, or an explicitly requested programmatic grading contract.

`workbench open` prints a Workbench Web URL and opens the operating-system browser unless `--no-open` is present. Agents should use `workbench open --json --no-open` when they will navigate an embedded browser themselves. Project identity lives in the route path as `/projects/:projectId`; candidate refs open the route-native candidate evaluation view, and run refs open the candidates view for that project.

## Hosted Workflows

`workbench launch` validates the local source, creates the hosted project from `workbench.yaml`, uploads the declared Dockerfile environment, candidate directory, and tasks directory, and writes the local project link. `workbench sync` performs one idempotent source upload for an existing linked or specified hosted project. If `workbench.yaml`, candidate files, task files, Dockerfile source, resources, and network policy are unchanged, sync returns the current source revision without creating a new source revision or environment version. The YAML `name` is the project name; it is unique per owner and immutable after launch.

`workbench pull` downloads the synced hosted source files for the current or selected source revision into a local directory and links that directory to the hosted project. Generated candidate files remain under `workbench candidates pull`.

`workbench eval` creates a hosted run with `workflow: "eval"`. It evaluates the requested candidate, the active hosted candidate, or the uploaded candidate snapshot if no candidate exists. It queues runner and grader executions for the top-level spec and does not propose changes.

`workbench improve` creates a hosted run with `workflow: "improve"`. It queues proposer executions up to `--budget`, then runner and grader executions for each proposed candidate, and updates hosted candidate history by greedy selection.

`workbench compare` is a result-inspection workflow over completed evaluations. Authored YAML no longer declares comparison variants; provider, environment, runner, and grader choices live in the single top-level spec.

All workflows freeze the current source revision, base candidate files, and task files onto the run input. Each direct child directory under `tasks.path` runs as a standalone task. `task.yaml` is parsed as control-plane task text and is not mounted as a runtime file. Task roots may contain only `task.yaml`, optional `input/`, and optional `expected/`; runners see optional `/workspace/input/task/input`, while graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and runner output files mounted at `/workspace/input/runner-output`. Runtime phases receive only named read-only inputs under `/workspace/input` plus a writable `/workspace/output` directory. Proposers see candidate files and prior run traces. Authored commands do not receive `WORKBENCH_*` helper environment variables.

The `--samples` flag is a repeat count over the frozen task set. A materialized evaluation sample may contain many task case results, but comparison and candidate sample counters count only repeat sample indexes, not task executions.

Execution usage is role-aware. Proposer, runner, and grader jobs may each report provider/model, token breakdown, and dollar cost. Provider-reported cost is authoritative; Workbench estimates from its checked-in LiteLLM price snapshot only when an adapter reports enough tokens but no provider cost. Comparison cost is runner plus grader cost per evaluation sample. Candidate total usage includes proposer cost plus every runner and grader sample cost.

## Local Development

`workbench dev improve`, `workbench dev eval`, and `workbench dev compare` run against a local source directory without hosted API calls. They use `--dir`, share the same `workbench.yaml` contract, store local history under `.workbench/`, and automatically run independent local jobs concurrently according to declared execution resources and local host capacity. No user-facing parallelism knob is exposed. Dev runs do not start a browser server; text and JSON outputs include a `workbench dev open --dir ...` command for starting the read-only local view explicitly.

`workbench dev open` starts a read-only local browser server for direct local development state. The server reads the selected source directory and `.workbench/runtime`, exposes the same browser DTO shape used by hosted Workbench Web, and serves the shared Workbench workspace UI. Its project Files surface mirrors the synced source projection: `workbench.yaml`, files under `candidate.path`, files under `tasks.path`, and the Dockerfile selected by `environment.dockerfile`. It must not expose hosted projects, cloud auth, queue or worker internals, arbitrary filesystem paths outside the synced source projection, or browser mutation endpoints.

## Spec Authoring Contract

The hosted API stores the spec as YAML source. The canonical authoring guide lives under [`docs/evals/`](docs/evals/):

- [`docs/evals/spec-syntax.md`](docs/evals/spec-syntax.md) owns the YAML shape.
- [`docs/evals/runner-contract.md`](docs/evals/runner-contract.md) owns staged runtime directories, phase visibility, runner output files, and the grader scorecard file.
- [`docs/evals/from-existing-workflow.md`](docs/evals/from-existing-workflow.md) owns the workflow-wrapping path.
- [`docs/evals/from-file-outputs.md`](docs/evals/from-file-outputs.md) owns file-output task guidance for `.docx`, `.xlsx`, `.pdf`, `.pptx`, and similar files.

The validator accepts only the version-11 YAML shape: `version: 11`, `name`, `candidate.path`, `candidate.editable`, `tasks.path`, `environment.dockerfile`, `objective`, `propose`, `run`, and `grade`. Adapter blocks use `use:` and optional `with:` fields. Agent runners are direct provider adapters (`codex`, `claude`, or `pi`); command adapters use `use: command` with one `with.command` string and no `prepare`, `cwd`, or custom `env` fields. All authored file paths are workspace-relative literals and must not be globs. Do not put eval-only rubrics, goldens, expected values, or grader code in the candidate unless that evaluator is itself the product being improved. Provider auth is resolved from user-scoped provider connections and harness config is resolved by sandbox adapters, not stored in `workbench.yaml`. Workbench agent turns run inside the selected sandbox boundary, so provider descriptors may set sandbox-appropriate harness defaults without adding authoring knobs.

## Runtime Environments

The runtime environment model is Dockerfile-first. Specs declare `environment.dockerfile`. Local development builds from that Dockerfile as needed. Hosted source sync creates or reuses a project environment from the Dockerfile, normalized resources, and network policy, and runs use the active built version. Missing `environment.network` means open outbound egress. To restrict network access, set `environment.network.egress: none` or set `environment.network.egress: allowlist` with an explicit `allow` list of hosts or host:port entries. Provider auth does not infer or rewrite network policy; for example, Claude Bedrock auth runs Claude Code with Bedrock credentials, and a restricted Bedrock project must list its Bedrock runtime host itself. The Dockerfile is the place to pin provider CLI versions and install task tools such as LibreOffice. Networked provider CLIs and HTTPS tools also depend on native CA certificates; minimal base images may need `ca-certificates`, and `workbench init` templates include it by default.
