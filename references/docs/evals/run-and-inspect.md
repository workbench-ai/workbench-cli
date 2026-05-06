# Run And Inspect

Use this loop after writing `workbench.yaml`, candidate files, and tasks.

## Check And Launch

```bash
workbench check
workbench launch
```

If the hosted project already exists, `workbench launch` fails on the duplicate `workbench.yaml` name. Use `workbench link <project-id>` or pass `--project <project-id>` to each hosted command when you intentionally want to update an existing project. Fix validation errors before running. If the runtime is custom, keep `environment.dockerfile` pointed at the local Dockerfile; hosted sync builds or reuses the project environment version from that Dockerfile.

## Start Smoke Workflows

Run one sample first:

```bash
workbench eval --samples 1 --watch --json
workbench improve --budget 1 --samples 1 --watch --json
workbench open <candidate-id> --json --no-open
```

`--samples` counts repeat attempts over the selected task set. If a project has ten tasks and you run `--samples 1`, Workbench should report one completed sample for the candidate while the task table shows one case result per task.

Cost metrics are sample-based in comparisons. The comparison cost column is runner plus grader execution cost for each evaluation sample. Candidate totals also include proposer execution cost. If a provider reports cost directly, Workbench uses that value; otherwise it estimates from the checked-in LiteLLM model price snapshot when the adapter reports enough token detail.

Use the `candidateId` from the watched JSON output when opening the browser. If a run is still pending and no candidate id exists yet, use `workbench open --json --no-open` to open the project.

If the sample errors, inspect the proposer, runner, grader, runtime environment, and output-writing logic. Most first-pass errors are one of:

- the command path assumes a different working directory
- the configured command grader writes score JSON somewhere other than `/workspace/output/scorecard.json`
- the result is missing the configured finite numeric objective metric
- the Dockerfile runtime is missing a parser or renderer for the produced files
- the runner writes no non-internal files under `/workspace/output`

## Inspect The Candidate

```bash
workbench candidates list --json
workbench candidates files <candidate-id> --json
workbench open <candidate-id> --json --no-open
workbench candidates preview <candidate-id> --path eval-summary.md --output -
workbench candidates pull <candidate-id> --out ./best-candidate
```

Use the first run to verify that score direction, task expected files, durable runner files, and candidate file changes are all understandable before increasing budget or samples. When an embedded browser is available, navigate it to the URLs returned by `workbench open --json --no-open` so the user can inspect Workbench Web while the CLI loop continues.
