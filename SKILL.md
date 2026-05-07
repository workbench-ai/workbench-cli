---
name: workbench-cli
description: >-
  Use this skill for configuring, creating evaluations for, running, inspecting,
  comparing, improving, or exporting Workbench projects with the `workbench`
  CLI, including evals based on existing workflows or file outputs such as DOCX,
  XLSX, PDF, and PPTX files.
---

# Workbench CLI

Use the `workbench` CLI as the cloud-first Workbench interface. Normal commands target hosted projects; local execution is development mode under `workbench dev`.

The public resources are:

- source directory: local `workbench.yaml`, `candidate/`, `tasks/`, optional runtime Dockerfile such as `environment/Dockerfile`, optional project-contained adapter directories, and optional `.workbench/origin.json`
- project: the hosted Workbench project
- public project: a hosted project with `visibility: public`, addressable as `username/project`
- star: a user marker on a public project
- fork: a private project created from the current public source
- contribution: a PR-like proposal from a fork back to its upstream public project
- candidate: a mutable `skill`, `pipeline`, or advanced `command`
- runtime: the Dockerfile runtime selected by `runtime.dockerfile`
- run: hosted `eval`, `improve`, or `compare`
- candidate snapshot: generated hosted output files that can be inspected, previewed, and pulled

## CLI Availability

Before using hosted project commands, verify that the `workbench` binary is available:

```bash
workbench --version
```

If the command is missing, install the published CLI:

```bash
npm install -g @workbench-ai/workbench-cli
workbench --version
```

If npm reports an auth or registry error, report that npm needs registry access for the `@workbench-ai` scope, then stop cleanly instead of falling back to a local monorepo build.

## Working Rules

- Prefer `workbench init --skill NAME --agent ADAPTER` when the candidate is unclear or when the candidate is agent-facing instructions, prompts, examples, or a reusable workflow skill. Use `workbench init --pipeline NAME --agent ADAPTER` when the pipeline itself is clearly the candidate. Use `workbench init --command NAME` only when the command-line implementation itself is the candidate. The candidate is the mutable thing Workbench improves; evals live in `tasks`, `run`, and `grade`. Specs can mix adapters across phases, such as a command runner with a rubric grader or an agent runner with a command grader. Then run `workbench push` to create or update the hosted project and write `.workbench/origin.json`. The `workbench.yaml` name is the hosted project name.
- Default to `grade: use: rubric` for qualitative or judgment-based scoring, including task fit, output quality, writing quality, formatting quality, visual quality, and reviewer-style criteria. Use `grade: use: command` only for narrow objective checks, an existing deterministic scorer, or an explicitly requested programmatic grading contract. Do not write a custom grader just because a task uses binary files; have the runner emit inspectable files under `/workspace/output` and let the rubric grader judge them.
- Use `workbench eval`, `workbench improve`, and `workbench compare` for hosted workflows. Add `--watch` when waiting for completion is requested.
- Treat project-contained adapter directories listed in `adapters` as pushed source. `workbench push`, `workbench clone`, `workbench pull`, and Workbench Web source export include those adapter files alongside `workbench.yaml`, candidate files, task files, and the Dockerfile.
- Treat watched hosted commands as client-side polling only. Stopping `workbench eval --watch`, `workbench improve --watch`, `workbench compare --watch`, or `workbench watch` does not cancel the hosted run; use `workbench runs cancel RUN_ID` to cancel it.
- Treat hosted command URLs as first-class output: `push`, `publish`, `fork`, `eval`, `improve`, `compare`, `watch`, `runs show`, and `runs cancel` include a `urls` object in JSON and print the most useful Workbench Web URL in text output.
- Treat hosted push as idempotent: unchanged `workbench.yaml`, candidate files, task files, project-contained adapter files, authored Dockerfile source, composed runtime, resources, and network policy must not create a new source revision or reset the hosted active candidate.
- Use `workbench dev improve`, `workbench dev eval`, and `workbench dev compare` only when local development without hosted API calls is explicitly requested.
- Use `workbench check` to validate the local source and inspect the execution plan before pushing; JSON output includes a `plan` with project description, source counts, candidate path/editable files, task cases, runtime resources/network, objective metric, and proposer/runner/grader adapters.
- Treat the `workbench.yaml` `name` as the hosted project name and public handle. It must be unique for the authenticated username, appears in Workbench Web URLs as `/projects/:username/:project`, and is not renamed by `workbench push`. Treat the required `description` as human project context shown in Workbench Web; keep optimizer instructions in `objective.goal`.
- Treat current specs as version 12: every spec has `name` and `description`; every candidate has `candidate.path` and `candidate.editable`; tasks live under `tasks.path`; the Dockerfile runtime lives at `runtime.dockerfile`; objective metadata lives under `objective`; execution adapters live at top-level `propose`, `run`, and `grade`.
- Treat task files by projection: each task root may contain only `task.yaml`, optional `input/`, and optional `expected/`; `task.yaml` is control-plane task text, runners see optional `/workspace/input/task/input`, and graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and `/workspace/input/runner-output` (runner output).
- Treat phase visibility as exact: proposers see `/workspace/input/candidate` and `/workspace/input/traces` (prior run event streams); runners see `/workspace/input/candidate` and optional `/workspace/input/task/input`; graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and `/workspace/input/runner-output`. No phase receives `workbench.yaml` or `task.yaml` as a mounted file.
- Avoid reward-hacking eval layouts: put only real supporting workflow files under `tasks/<case>/input/`, and put goldens, expected values, private rubrics, tolerances, expected files, and task-specific grader scripts for intentional command graders under `tasks/<case>/expected/`. Put the task instruction in `task.yaml`; do not duplicate it into `input/request.md` unless a command runner specifically needs an ordinary file. Do not tell runners to inspect expected files or public inputs that are really answer keys.
- Treat command adapters as a single-command surface: `use: command` with `with.command` only. Do not author Workbench `prepare`, `cwd`, or custom `env` fields.
- Treat runtime network as explicit sandbox policy: omitted `runtime.network` means open outbound egress, `egress: none` means offline, and `egress: allowlist` requires an explicit `allow` host list. Adapter auth does not infer network hosts; Bedrock, proxies, Azure, private gateways, and other custom endpoints must be listed only when the user deliberately chooses allowlist egress.
- Treat adapter auth as manifest-owned. Run `workbench auth connect ADAPTER[/SLOT] --method METHOD` after the adapter is listed in `workbench.yaml`; default profiles are implicit for adapters that declare auth, including manifest-declared nested adapter refs and every declared multi-slot auth slot, while non-default profiles are referenced with `auth` in YAML. Nested adapters must still be explicit adapter invocations in `with`, such as `with.judge.use: codex`; Workbench does not invent hidden provider defaults.
- Use `workbench envs list` and `workbench envs create --project PROJECT --name NAME --dockerfile PATH|-` for manual hosted runtime environments. Manual environment creation does not switch the source-pushed workflow runtime; `workbench push` owns that runtime.
- When authoring or editing the runtime Dockerfile for task tools or HTTPS workflows, ensure minimal or slim base images install `ca-certificates`; `workbench init` generated Dockerfiles do this by default. Adapter runtime dependencies such as agent CLIs belong in adapter setup, not in the project Dockerfile. Adapter manifest `command` is run exactly as a shell command string; omit it only when setup installs the default `workbench-adapter-<id>` command.
- Use `workbench clone USER/PROJECT` to download the current public source with a read-only origin, and `workbench pull` to reconcile the managed source set from its origin, including upstream deletions. Use `workbench candidates list`, `workbench candidates files`, `workbench candidates preview`, and `workbench candidates pull` for generated candidate inspection and retrieval.
- Use `workbench publish`, `workbench unpublish`, `workbench star`, `workbench unstar`, `workbench fork USER/PROJECT [DIR]`, and `workbench contribute` for public benchmark collaboration; fork materializes a writable local checkout.
- Use `workbench projects delete PROJECT --dry-run` to preview hosted project deletion, then `workbench projects delete PROJECT` only when deletion is explicitly requested.
- Use `workbench runs show RUN_ID` for hosted run/job detail and `workbench runs cancel RUN_ID` to stop a running hosted workflow. Do not delete a project to stop a run.
- Use `workbench dev candidates`, `workbench dev files`, `workbench dev preview`, and `workbench dev restore --dry-run|--yes` for local development history.
- Prefer `--json` when another tool or agent needs structured output.
- Run Workbench project commands from the source directory containing `workbench.yaml`, or pass `--dir DIR` explicitly. Bare commands like `workbench check`, `workbench push`, `workbench pull`, `workbench eval`, and `workbench improve` target the current directory.
- Treat `--samples` as repeat attempts over the task set, not as task count. A run with two task directories and `--samples 1` should report one evaluation sample, with task-level detail under the candidate evaluation.
- Interpret execution cost by role: comparison cost is runner plus grader cost per sample, while candidate totals also include proposer cost. Provider-reported cost wins; Workbench estimates from its checked-in LiteLLM price snapshot only when the adapter reports tokens but no provider cost.
- Treat hosted billing separately from harness spend: users pay model or harness providers through adapter auth, while Workbench may require a positive credit balance and charge a hosted execution service fee for completed hosted jobs under the configured billing formula. New hosted billing accounts receive a $5 credit grant, users can add prepaid credit through Stripe top-ups, and Workbench credit applies only to the Workbench service fee. Do not expose backend provider, VM, host, CPU, memory, disk, or lease details as user-facing billing concepts.
- Treat low scores and rubric criteria with `pass: false` as quality signals, not execution failures. Use scores, metrics, and criteria to explain quality; reserve failed/error language for runner, grader, provider, sandbox, timeout, validation, or runtime errors.

## Browser Companion

When the host provides an embedded or in-app browser, opening Workbench Web is an action to take, not just a URL to report. Keep the browser synchronized with the CLI while you work:

1. After hosted `push`, `publish`, `fork`, `eval`, `improve`, `compare`, `watch`, `runs show`, or `runs cancel` commands return a `urls` object, navigate the embedded browser to the most relevant returned URL before finalizing.
2. Make the browser visibly open to the Workbench page so the user can see it; do not only resolve, print, or mention the URL when an embedded browser is available.
3. Use `workbench open --json --no-open` only when resolving a project, run, candidate, or comparison id later.
4. If no embedded browser capability is available, use `workbench open` without `--no-open` or report the URL for the user.

After watched `eval`, `improve`, or `compare` workflows, use `urls.candidateEvaluation` when the command output includes a candidate id. When validating traces or debugging behavior, also use `urls.run` or `workbench open <run-id> --json --no-open` and click into the actual task run trace. If the run has not produced a task trace yet, use `workbench runs show <run-id>` for operational detail.

For explicit local development, `workbench dev eval`, `workbench dev improve`, and `workbench dev compare` run to completion and exit. Text output prints an `Open local view:` command and lifecycle note; JSON output returns the same information at `localView.command` and `localView.note`. Run that command, or run `workbench dev open --json --no-open`, to start the read-only local browser view over the selected `--dir` and `.workbench/runtime`. `workbench dev open` is the local web server and must stay running while the local web view is in use; stopping or killing it stops the server and the page will stop working. It does not contact hosted Workbench Web. The local project Files tab mirrors the pushed source projection (`workbench.yaml`, `candidate.path`, `tasks.path`, project-contained adapter files, and the authored `runtime.dockerfile`) rather than the full local repo root.

## Task Recipes

Setup and create:

```bash
workbench login
workbench auth connect codex
workbench init --skill my-eval --agent codex
workbench push --json
```

Edit an existing project:

```bash
workbench check
workbench push --json
```

Run evaluation and improvement loops:

```bash
workbench eval --samples 1 --watch --json
workbench improve --budget 1 --samples 1 --watch --json
workbench compare --samples 1 --watch --json
workbench runs show <run-id> --json
workbench runs cancel <run-id>
```

Inspect and export:

```bash
workbench open <candidate-id> --json --no-open
workbench candidates list
workbench candidates files <candidate-id>
workbench candidates preview <candidate-id> --path workbench-run-summary.md --output -
workbench candidates pull <candidate-id> --out ./best-candidate
```

Use `workbench clone USER/PROJECT` for public source download and `workbench fork USER/PROJECT [DIR]` when edits should be pushed back through a contribution. Use `--project PROJECT` for one-off private owner commands without writing `.workbench/origin.json`; internal `wb_...` ids are accepted.

## Evaluation Creation

When creating a Workbench eval, first read [references/docs/evals/README.md](references/docs/evals/README.md). In the authored repo tree, the same content lives under `docs/evals/`.

- Use [references/docs/evals/spec-syntax.md](references/docs/evals/spec-syntax.md) before writing or editing `workbench.yaml`.
- Use [references/docs/evals/runner-contract.md](references/docs/evals/runner-contract.md) before writing runner/grader commands or the `/workspace/output` runner-file and `/workspace/output/scorecard.json` grader contracts.
- Use [references/docs/evals/from-existing-workflow.md](references/docs/evals/from-existing-workflow.md) when there is already a benchmark, test suite, script, or manual scoring workflow.
- Use [references/docs/evals/from-file-outputs.md](references/docs/evals/from-file-outputs.md) when tasks or outputs involve documents, spreadsheets, PDFs, decks, goldens, examples, or other file outputs with format-specific runtime prerequisites.
- For file-output task evals, load only the matching recipe from `references/docs/evals/file-recipes/`: `docx.md`, `xlsx.md`, `pdf.md`, or `pptx.md`.

Do not treat `skills/workbench-cli/evals/` as project eval examples. That directory is the skill-ergonomics catalog for testing this skill.

## Local References

- Read [references/docs/cli.md](references/docs/cli.md) for the canonical CLI guide. In the authored repo tree, the same content lives under `docs/cli.md`.
- Read [references/SPEC.md](references/SPEC.md) when you need the command and spec contract.
- Read [references/docs/evals/](references/docs/evals/) when you need to create or explain Workbench eval specs, runner commands, tasks, or file-output task recipes.
