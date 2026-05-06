# Runtime Contract

Workbench version 11 compiles `workbench.yaml` into proposer, run, and grade executions. Hosted Workbench stores each executable node as one durable `execute` job with a `purpose` of `propose`, `run`, or `grade`.

Proposer executions run once per trial and write a candidate patch. Run executions use the current candidate and one current task to write ordinary output files. Grade executions score the immutable runner output files mounted from the run phase. Rubric grading uses one grader execution per runner sample; the judge prompt contains the full rubric and the runtime validates that every criterion receives a score and rationale. Provider-backed `codex`, `claude`, `pi`, and `rubric` adapters run through their declared harness. Command adapters run inside the Dockerfile environment declared by `environment.dockerfile`.

During hosted execution, command phases and agent harnesses may publish live `job_progress` event batches before the terminal job output exists. These batches are best-effort UI progress and trace deltas. The completed job output remains the authoritative candidate patch, runner files, or scorecard.

Agent-backed phases also report execution usage on the completed job. The durable shape is role-aware: proposer jobs write proposer usage, runner jobs write runner usage, and grader jobs write grader usage. Provider-reported cost is authoritative. When the provider does not report cost, Workbench estimates from its checked-in LiteLLM price snapshot using the exact model string emitted by the adapter. Comparison cost is runner plus grader cost per evaluation sample; candidate total cost also includes the proposer.

## Staged Filesystem

Every execution receives a minimal filesystem:

- `/workspace/input/<name>`: read-only declared phase inputs.
- `/workspace/output`: writable output directory.

Authored commands do not receive `WORKBENCH_*` helper environment variables. Read from the mounted paths directly.

Phase visibility is exact:

| Purpose | Visible inputs | Required output |
| --- | --- | --- |
| `propose` | `/workspace/input/candidate`, `/workspace/input/traces` | `/workspace/output/candidate_patch.json` |
| `run` | `/workspace/input/candidate`, `/workspace/input/task/input` | one or more non-internal files under `/workspace/output` |
| `grade` | `/workspace/input/task/input`, `/workspace/input/task/expected`, `/workspace/input/runner-output` | `/workspace/output/scorecard.json` |

No phase receives `workbench.yaml` or `task.yaml` as a mounted file. `task.yaml` is parsed by Workbench as control-plane task text; command runners that need detailed task instructions should receive them as ordinary public files such as `/workspace/input/task/input/request.md`. No phase receives a broad archive, spec JSON, job JSON, prompt file, optional support directory, or reward file. The sandbox control request is removed before any authored command or agent harness runs. Inputs are read-only and mutations under `/workspace/input` are ignored.

This boundary is an authoring contract, not a scoring shortcut. Put only real workflow inputs under `input/`; put goldens, expected values, hidden rubrics, and task-specific grader code for intentional command graders under `expected/`. Runners should not be instructed to inspect `expected/`, and graders should not reward output that merely proves the runner copied private targets.

Agent prompt text is intentionally thin. Proposers receive only the objective goal, metric, candidate root, traces root, editable paths, and candidate patch output contract. Runners receive only `run.with.instructions`, candidate/task/output roots, and the rule that non-internal `/workspace/output` files are persisted for grading and inspection. Rubric graders receive only `grade.with.instructions`, the active criterion list, task and runner-output roots, and the minimal score-plus-criteria JSON contract; Workbench wraps that result into `scorecard.json`. File contents are not interpolated into phase prompts; agents and commands inspect the mounted folders directly.

`/workspace/input/traces` is proposal-only. It contains generic event streams and job summaries from terminal prior executions in the same run, plus any completed trace files already emitted under `.workbench/traces/`. Runner and grader phases never receive this folder.

Spec file references are resolved before these directories are staged:

- `candidate.path`, `tasks.path`, and `environment.dockerfile` are workspace-relative literal paths.
- `candidate.editable[]` entries are literal paths inside `candidate.path`.
- Workbench uploads the whole candidate directory and the whole tasks directory. At runtime, task files are projected by convention: `input/` is runner-visible, `expected/` is grader-only, and root task files other than `task.yaml` are invalid. There are no include globs.
- `propose`, `run`, and `grade` identify adapters by `use` and configure them with `with`.

## Outputs

`candidate_patch.json` describes changes to candidate files:

```json
{
  "files": [
    {
      "path": "run.py",
      "content": "print('updated')\n",
      "encoding": "utf-8"
    }
  ],
  "fileChanges": ["run.py"],
  "summary": "Updated runner behavior."
}
```

`fileChanges` is required and must list the changed candidate-relative paths.

Run executions do not write a manifest. Any non-internal file under `/workspace/output` is persisted, mounted into graders at `/workspace/input/runner-output`, and shown in the Files tab for inspection. Internal control files such as `.workbench/**`, `candidate_patch.json`, `scorecard.json`, sandbox metadata, command stdout/stderr logs, and exit-code files are filtered from the user-facing file set.

`scorecard.json` is the grader result. `score` is required and must be finite.

```json
{
  "score": 0.82,
  "metrics": {
    "format_similarity": 0.82,
    "criterion__required_tables": 1
  },
  "summary": "Matched required tables and headings; missed footer formatting.",
  "cases": [
    {
      "id": "task",
      "status": "completed",
      "metrics": { "format_similarity": 0.82 }
    }
  ],
  "feedback": {
    "notes": "Footer was missing."
  }
}
```

Fields:

- `score`: required numeric metric used by `objective.metric: score`.
- `metrics`: optional additional numeric metrics. Keys prefixed with `criterion__` render as criteria metrics.
- `summary`: optional short human-readable result.
- `cases`: optional task-level results, feedback, and criterion scores. Case status is operational: use `completed` for a valid scorecard and `error` only for grader/runtime errors. Low scores and criteria with `pass: false` stay in metrics and criteria, not case status. File inspection is handled by the runner output file set, not by case-level file references.
- `feedback`: optional structured JSON for diagnostics.

If any phase exits non-zero, Workbench marks the execution failed. Command propose and grade executions also fail when the required phase JSON is missing or invalid. Run executions fail when they publish no non-internal files.

## Inspectable Files

If a generated report, normalized text dump, screenshot, workbook, or debug file should be inspectable after the run, write it into `/workspace/output` from the runner. Do not create a separate manifest to describe it. Workbench forwards those files to the grader at `/workspace/input/runner-output` and exposes the same file set in the browser.

A practical pattern is:

1. Runner creates the primary output and any supporting summaries under `/workspace/output`.
2. Rubric graders compare `/workspace/input/runner-output` against `/workspace/input/task/input`, `/workspace/input/task/expected`, and rubric criteria.
3. Command graders write `scorecard.json` only when the score is objective and deterministic or wraps an existing scorer; rubric graders return score plus criterion results and the adapter writes the standard scorecard.

## Command Shape

Prefer checked-in runner scripts and Dockerfile-pinned tools. Command phases have one command string and always execute from `/workspace`; there are no Workbench `prepare`, `cwd`, or custom `env` fields. Grade commands are intentionally narrower than run commands: they see only public task input, grader-only expected files, and runner output files, so put task-specific grader code under `tasks/<case>/expected/` or bake reusable grader code into the Docker image. Use command grading only for objective deterministic checks or an existing scorer. For qualitative scoring, keep extraction and normalization in runner output files or expected files, then use a rubric grader.

```yaml
environment:
  dockerfile: environment/Dockerfile
run:
  use: command
  with:
    command: python /workspace/input/candidate/run.py
grade:
  use: command
  with:
    command: python /workspace/input/task/expected/grade.py
```

Use shell glue only inside the single command when it genuinely clarifies the phase. Put dependencies in `environment/Dockerfile` instead of installing them during every evaluation job.
