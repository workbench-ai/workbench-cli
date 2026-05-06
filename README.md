# Workbench CLI

Create skill-first evals by default, use rubric grading for qualitative scoring, use pipeline or advanced command evals when clearly appropriate, and configure, run, inspect, and export hosted Workbench projects with workbench.

This repository is a generated public skill for AI coding agents. It follows the [Agent Skills](https://agentskills.io) format with `SKILL.md` at the repository root.

## Installation

```bash
npx skills add workbench-ai/workbench-cli
```

## Use When

- Verifying or installing the published `@workbench-ai/workbench-cli` package before hosted setup
- Scaffolding ambiguous candidates with `workbench init --skill NAME --agent codex`, using `workbench init --pipeline NAME --agent codex` when the pipeline is clearly the candidate, and using `workbench init --command NAME` only when the command-line implementation itself is the candidate
- Authoring Workbench eval specs from existing workflows or file-output tasks with rubric grading by default and command grading only for deterministic scoring contracts
- Creating a hosted Workbench project from `workbench.yaml` and linking local source with `workbench launch` or `workbench link PROJECT_ID`
- Listing hosted runtime environments and creating project-specific Dockerfile environments
- Uploading idempotent project source through `workbench sync`
- Starting hosted eval, improve, and compare workflows, watching completion, inspecting candidate files, and exporting selected candidates
- Keeping Workbench Web open in an embedded browser with `workbench open --json --no-open` while an agent drives the CLI
- Visualizing local development state with the `workbench dev open --dir ...` command printed by successful dev runs or by running `workbench dev open --json --no-open` explicitly

## Structure

`SKILL.md` is the installable skill. Supporting directories such as `agents/`, `references/`, and `evals/` are included only when declared by the authored product skill.
