# Workbench CLI

Create skill-first evals by default, use rubric grading for qualitative scoring, use pipeline or advanced command evals when clearly appropriate, and configure, run, inspect, and export hosted Workbench projects with workbench.

This repository is a generated public skill for AI coding agents. It follows the [Agent Skills](https://agentskills.io) format with `SKILL.md` at the repository root.

## Installation

```bash
npx skills add workbench-ai/workbench-cli
```

## Use When

- Verifying or installing the published `@workbench-ai/workbench-cli` package before hosted setup
- Scaffolding ambiguous candidates with `workbench init --skill NAME --agent ADAPTER`, using `workbench init --pipeline NAME --agent ADAPTER` when the pipeline is clearly the candidate, and using `workbench init --command NAME` only when the command-line implementation itself is the candidate
- Authoring Workbench eval specs from existing workflows or file-output tasks with required project descriptions, rubric grading by default, and command grading only for deterministic scoring contracts
- Creating or updating a hosted Workbench project from `workbench.yaml` with `workbench push` and `.workbench/origin.json`
- Listing hosted runtime environments and creating manual hosted environments from Dockerfiles
- Publishing, cloning, starring, forking, and contributing to public benchmark projects addressed as `username/project`
- Uploading idempotent project source through `workbench push`
- Starting hosted eval, improve, and compare workflows, watching completion, inspecting candidate files, and exporting selected candidates
- Keeping Workbench Web open in an embedded browser with `workbench open --json --no-open` while an agent drives the CLI
- Visualizing local development state with the `workbench dev open --dir ...` command printed by successful dev runs or by running `workbench dev open --json --no-open` explicitly

## Structure

`SKILL.md` is the installable skill. Supporting directories such as `agents/`, `references/`, and `evals/` are included only when declared by the authored product skill.
