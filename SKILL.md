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

- source directory: local `workbench.yaml`, `candidate/`, `tasks/`, optional `environment/Dockerfile`, and optional `.workbench/link.json`
- project: the hosted Workbench project
- candidate: a mutable `skill`, `pipeline`, or advanced `command`
- environment: the Dockerfile runtime selected by `environment.dockerfile`
- run: hosted `eval`, `improve`, or `compare`
- candidate: generated hosted output snapshot that can be inspected, previewed, and pulled

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

- Prefer `workbench init --skill NAME --agent codex|claude|pi` when the candidate is unclear or when the candidate is agent-facing instructions, prompts, examples, or a reusable workflow skill. Use `workbench init --pipeline NAME --agent codex|claude|pi` when the pipeline itself is clearly the candidate. Use `workbench init --command NAME` only when the command-line implementation itself is the candidate. The candidate is the mutable thing Workbench improves; evals live in `tasks`, `run`, and `grade`. Specs can mix adapters across phases, such as a command runner with a rubric grader or an agent runner with a command grader. Then run `workbench launch` to create, sync, and connect a local source directory to a hosted project. The `workbench.yaml` name is the hosted project name.
- Default to `grade: use: rubric` for qualitative or judgment-based scoring, including task fit, output quality, writing quality, formatting quality, visual quality, and reviewer-style criteria. Use `grade: use: command` only for narrow objective checks, an existing deterministic scorer, or an explicitly requested programmatic grading contract. Do not write a custom grader just because a task uses binary files; have the runner emit inspectable files under `/workspace/output` and let the rubric grader judge them.
- Use `workbench eval`, `workbench improve`, and `workbench compare` for hosted workflows. Add `--watch` when waiting for completion is requested.
- Treat hosted sync as idempotent: unchanged `workbench.yaml`, candidate files, task files, Dockerfile source, resources, and network policy must not create a new source revision or reset the hosted active candidate.
- Use `workbench dev improve`, `workbench dev eval`, and `workbench dev compare` only when local development without hosted API calls is explicitly requested.
- Use `workbench check` to validate local `workbench.yaml`.
- Treat the `workbench.yaml` `name` as the hosted project name. It must be unique and is not renamed by `workbench sync`.
- Treat current specs as version 11: every candidate has `candidate.path` and `candidate.editable`; tasks live under `tasks.path`; the Dockerfile runtime lives at `environment.dockerfile`; objective metadata lives under `objective`; execution adapters live at top-level `propose`, `run`, and `grade`.
- Treat task files by projection: each task root may contain only `task.yaml`, optional `input/`, and optional `expected/`; `task.yaml` is control-plane task text, runners see optional `/workspace/input/task/input`, and graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and `/workspace/input/runner-output` (runner output).
- Treat phase visibility as exact: proposers see `/workspace/input/candidate` and `/workspace/input/traces` (prior run event streams); runners see `/workspace/input/candidate` and optional `/workspace/input/task/input`; graders see optional `/workspace/input/task/input`, optional `/workspace/input/task/expected`, and `/workspace/input/runner-output`. No phase receives `workbench.yaml` or `task.yaml` as a mounted file.
- Avoid reward-hacking eval layouts: put only real supporting workflow files under `tasks/<case>/input/`, and put goldens, expected values, private rubrics, tolerances, expected files, and task-specific grader scripts for intentional command graders under `tasks/<case>/expected/`. Put the task instruction in `task.yaml`; do not duplicate it into `input/request.md` unless a command runner specifically needs an ordinary file. Do not tell runners to inspect expected files or public inputs that are really answer keys.
- Treat command adapters as a single-command surface: `use: command` with `with.command` only. Do not author Workbench `prepare`, `cwd`, or custom `env` fields.
- Treat environment network as explicit sandbox policy: omitted `environment.network` means open outbound egress, `egress: none` means offline, and `egress: allowlist` requires an explicit `allow` host list. Provider auth does not infer network hosts; Bedrock, proxies, Azure, private gateways, and other custom endpoints must be listed only when the user deliberately chooses allowlist egress.
- Use `workbench envs list` and `workbench envs create --project ID --name NAME --dockerfile PATH|-` for runtime environments.
- When authoring or editing `environment/Dockerfile` for networked agent providers or HTTPS tools, ensure minimal or slim base images install `ca-certificates`; `workbench init` generated Dockerfiles do this by default.
- Use `workbench pull` for hosted source download. Use `workbench candidates list`, `workbench candidates files`, `workbench candidates preview`, and `workbench candidates pull` for generated candidate inspection and retrieval.
- Use `workbench projects delete PROJECT_ID --dry-run` to preview hosted project deletion, then `workbench projects delete PROJECT_ID` only when deletion is explicitly requested.
- Use `workbench dev candidates`, `workbench dev files`, `workbench dev preview`, and `workbench dev restore --dry-run|--yes` for local development history.
- Prefer `--json` when another tool or agent needs structured output.
- Use `--dir DIR` to target a source directory.
- Treat `--samples` as repeat attempts over the task set, not as task count. A run with two task directories and `--samples 1` should report one evaluation sample, with task-level detail under the candidate evaluation.
- Interpret execution cost by role: comparison cost is runner plus grader cost per sample, while candidate totals also include proposer cost. Provider-reported cost wins; Workbench estimates from its checked-in LiteLLM price snapshot only when the adapter reports tokens but no provider cost.
- Treat low scores and rubric criteria with `pass: false` as quality signals, not execution failures. Use scores, metrics, and criteria to explain quality; reserve failed/error language for runner, grader, provider, sandbox, timeout, validation, or runtime errors.

## Browser Companion

Keep Workbench Web open while you work when the user's environment has an embedded or in-app browser. Prefer this pattern:

1. Run `workbench open --json --no-open` after launch/link and sync, before a candidate exists.
2. Read the returned `url`.
3. Navigate the embedded browser to that URL.

After watched `eval`, `improve`, or `compare` workflows, use `workbench open <candidate-id> --json --no-open` when the command output includes a candidate id. It returns the candidate evaluation route with the hosted project selected. If no embedded browser is available, use `workbench open` without `--no-open` or print the URL for the user.

For explicit local development, `workbench dev eval`, `workbench dev improve`, and `workbench dev compare` run to completion and exit. Text output prints an `Open local view:` command, and JSON output returns the same command at `localView.command`. Run that command, or run `workbench dev open --json --no-open`, to start the read-only local browser view over the selected `--dir` and `.workbench/runtime`. It does not contact hosted Workbench Web. The local project Files tab mirrors the synced source projection (`workbench.yaml`, `candidate.path`, `tasks.path`, and `environment.dockerfile`) rather than the full local repo root.

## Task Recipes

Setup and create:

```bash
workbench auth login
workbench auth connect codex
workbench init --skill my-eval --agent codex
workbench launch
workbench open --json --no-open
```

Edit an existing project:

```bash
workbench check
workbench sync --json
workbench open --json --no-open
```

Run evaluation and improvement loops:

```bash
workbench eval --samples 1 --watch --json
workbench improve --budget 1 --samples 1 --watch --json
workbench compare --samples 1 --watch --json
workbench open <candidate-id> --json --no-open
```

Inspect and export:

```bash
workbench open <candidate-id> --json --no-open
workbench candidates list
workbench candidates files <candidate-id>
workbench candidates preview <candidate-id> --path workbench-run-summary.md --output -
workbench candidates pull <candidate-id> --out ./best-candidate
```

Use `workbench link PROJECT_ID` when a hosted project already exists. Use `--project ID` for one-off commands without writing `.workbench/link.json`.

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
