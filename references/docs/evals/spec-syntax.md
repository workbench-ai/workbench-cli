# Workbench Spec Syntax

Workbench stores one `workbench.yaml` spec per project. The canonical authored shape is version 12:

- `name` is the unique project handle.
- `description` explains what the project evaluates and is shown in Workbench Web.
- `candidate` declares the directory under improvement and the editable paths inside it.
- `tasks.path` points at task directories. Each direct child must contain a `task.yaml`.
- `runtime.dockerfile` declares the Dockerfile used locally and remotely.
- `objective` declares the optimization goal, metric, and direction.
- `propose`, `run`, and `grade` declare the adapters for each phase.

The candidate is never "the eval" unless the evaluator itself is the product being improved. Evals are defined by tasks plus the `run` and `grade` blocks.

Use `workbench check --dir <source-dir>` locally or `workbench push --project <project-id> --dry-run` before uploading hosted source.

## Minimal Shape

```yaml
version: 12
name: workflow-quality
description: Evaluate whether the workflow skill completes representative tasks with useful inspectable outputs.
candidate:
  path: candidate/workflow-skill
  editable:
    - SKILL.md
tasks:
  path: tasks
runtime:
  dockerfile: environment/Dockerfile
  resources:
    cpu: 2
    memoryGb: 4
    timeoutMinutes: 20
objective:
  goal: Improve the skill so it completes the task reliably and produces useful output files.
  metric: score
  direction: maximize
propose:
  use: codex
  with:
    model: gpt-5.4-mini
run:
  use: codex
  with:
    instructions: Use the candidate skill, task instruction, and any optional public task input to complete the task. Write durable output files under /workspace/output.
    model: gpt-5.4-mini
grade:
  use: rubric
  with:
    instructions: Grade only from the task instruction, optional public task input, expected files, runner output files, and the assigned criteria.
    judge:
      use: codex
      with:
        model: gpt-5.4-mini
    criteria:
      - id: correctness
        description: The produced output satisfies the task requirements.
```

## Required Fields

- `version`: must be `12`.
- `name`: short project-readable name.
- `description`: short human-readable explanation of what the project evaluates. Workbench shows it in the project dashboard and Overview tab. Keep optimizer instructions in `objective.goal`, not in `description`.
- `candidate.path`: workspace-relative candidate directory. For skills, this is the whole skill directory. Do not put eval-only rubrics, goldens, or grader code in the candidate.
- `candidate.editable`: literal paths inside `candidate.path` that the proposer may change.
- `tasks.path`: workspace-relative task directory.
- `runtime.dockerfile`: workspace-relative Dockerfile path.
- `objective.goal`: success target for proposer, reviewers, and humans.
- `objective.metric`: numeric metric to optimize.
- `objective.direction`: `maximize` or `minimize`.
- `propose`: proposer adapter, such as `codex`, `claude`, `pi`, `command`, or an installed adapter id.
- `run`: runner adapter, such as `codex`, `claude`, `pi`, `command`, or an installed adapter id.
- `grade`: grader adapter, such as `rubric`, `command`, or an installed adapter id.

Stale fields are rejected, including `candidate.kind`, `candidate.optimization`, `defaults`, `comparisons`, top-level `environment`, task-level `environment`, task-level `run`, task-level `grade`, top-level `evaluation`, top-level `cases`, phase-level `run.instructions`, phase-level `grade.instructions`, and the old `run.use: agent` wrapper.

## Runtime Network

If `runtime.network` is omitted, Workbench uses open outbound egress. This is the default because agent adapters, package managers, browser tools, private gateways, and custom endpoints are ordinary runtime concerns. Adapter auth never infers or rewrites network policy.

Use `egress: none` only for offline deterministic runs:

```yaml
runtime:
  dockerfile: environment/Dockerfile
  network:
    egress: none
```

Use `egress: allowlist` only when you want sandbox-level host restriction. The `allow` entries are hosts or host:port entries such as `api.example.com` or `proxy.example.com:8443`:

```yaml
runtime:
  dockerfile: environment/Dockerfile
  network:
    egress: allowlist
    allow:
      - api.example.com
      - proxy.example.com:8443
```

When using provider-specific endpoint families such as Claude through Bedrock, list those hosts yourself if you choose `allowlist`; for example, `bedrock-runtime.us-east-1.amazonaws.com`.

## Tasks

`tasks.path` points to a directory whose direct child directories are standalone task runs:

```text
workbench.yaml
candidate/
  invoice-review/
    SKILL.md
tasks/
  invoice-001/
    task.yaml
    expected/
      rubric.md
  invoice-002/
    task.yaml
    input/
      invoice.pdf
    expected/
      rubric.md
```

Each `task.yaml` contains only control-plane task text. It is parsed by Workbench and is not mounted as a runtime file:

```yaml
task: |-
  Review the invoice and identify billing issues.
```

All tasks use the same top-level `runtime`, `run`, and `grade` adapters.

Task runtime files use fixed directory names. `input/` is optional and contains public files mounted for both runner and grader phases. `expected/` is optional and contains grader-only files such as goldens, rubrics, tolerances, or task-specific grader scripts for intentional command graders. Root task files other than `task.yaml` are invalid.

## Path Resolution

All authored paths are literal workspace-relative paths. They must not be absolute, empty, `.`/`..`, or globs containing `*` or `?`.

- `candidate.path` is resolved from the workspace root.
- `candidate.editable[]` entries are resolved inside `candidate.path`.
- `tasks.path` is resolved from the workspace root.
- `runtime.dockerfile` is resolved from the workspace root.
- Workbench uploads the whole candidate directory and the whole tasks directory. There are no include globs.

## Runners

Agent runners use direct agent adapters. Agent CLI versions are installed by adapter setup on top of `runtime.dockerfile`; task and workflow tools belong in the Dockerfile. Adapter credentials come from `workbench auth connect`, not from YAML.

```yaml
run:
  use: codex
  with:
    instructions: Use /workspace/input/candidate as the candidate directory, the task instruction, optional public task input under /workspace/input/task/input, and /workspace/output for durable output files.
    model: gpt-5.4-mini
```

Command runners run inside the composed Dockerfile runtime from `/workspace`. The command adapter has only one authored field, `command`; setup belongs in the command itself, a checked-in script, or the runtime Dockerfile.

```yaml
run:
  use: command
  with:
    command: python /workspace/input/candidate/run.py
```

## Graders

Rubric graders are agent-backed and are the default for qualitative or judgment-based scoring, including task fit, output quality, writing quality, formatting quality, visual quality, and reviewer-style criteria. Workbench runs one grader per runner sample. The judge sees the current task files, runner output files, and the full rubric, then returns one score plus complete criterion JSON that Workbench validates and wraps into the standard scorecard.

```yaml
grade:
  use: rubric
  with:
    instructions: Grade only from the task instruction, optional public task input, expected files, runner output files, and the assigned criteria.
    judge:
      use: codex
      with:
        model: gpt-5.4-mini
    criteria:
      - id: quality
        description: The result satisfies the task requirements and is easy to inspect.
        weight: 1
```

Command graders run inside the composed Dockerfile runtime and write `/workspace/output/scorecard.json`. Use them only for narrow objective checks, an existing deterministic scorer, or an explicitly requested programmatic grading contract. Do not write a custom command grader just because a task produces binary files; emit inspectable runner output files and grade qualitative behavior with a rubric.

```yaml
grade:
  use: command
  with:
    command: python /workspace/input/task/expected/grade.py
```

## External Adapters

Built-in adapters are available by id: `codex`, `claude`, `pi`, `command`, and `rubric`. Project-contained adapters are listed in top-level `adapters` and referenced by the manifest `id`:

```yaml
adapters:
  - ./adapters/my-agent
run:
  use: my-agent
  with:
    instructions: Complete the task and write durable files under /workspace/output.
    mode: strict
```

Each adapter directory declares `workbench.adapter.yaml`:

```yaml
id: my-agent
protocol: workbench.adapter.v1
setup:
  - npm install --global .
command: workbench-adapter-my-agent
```

An adapter can declare nested adapter invocations with `refs`, using JSON pointers relative to its `with` object:

```yaml
id: my-rubric
protocol: workbench.adapter.v1
setup:
  - npm install --global .
command: workbench-adapter-my-rubric
refs:
  - /judge
```

Adapter setup commands are composed onto the image from `runtime.dockerfile`; the Dockerfile owns task dependencies, while the adapter manifest owns adapter runtime dependencies. If `command` is omitted it defaults to `workbench-adapter-<id>`; when `command` is present, Workbench runs that shell command string exactly and does not synthesize a fallback binary. The adapter command receives `WORKBENCH_ADAPTER_REQUEST`, a JSON file containing the execution purpose, adapter `with` config, optional auth material under `auth.self` and `auth.adapters`, task text when applicable, and staged filesystem paths. It writes phase outputs under `WORKBENCH_OUTPUT` using the same output contract as built-ins: `candidate_patch.json` for propose, ordinary files for run, and `scorecard.json` for grade. Optional adapter metadata goes in `WORKBENCH_OUTPUT/.workbench/result.json` for summary, feedback, and usage/cost reporting.

If an adapter manifest declares auth, the default profile is implicit. This also applies to nested adapter invocations reached through manifest `refs`; nested defaults must be explicit in the authored `with` object, such as `with.judge.use: codex`, rather than hidden by Workbench core. For multi-slot manifests, omitted `auth` means every declared slot uses profile `default`. Use `auth: prod` or a slot map only when a phase or nested invocation needs a non-default profile. `workbench auth connect ADAPTER[/SLOT] --method METHOD` validates the chosen method against the manifest before storing env vars, profile files, or command-emitted auth bundles.
