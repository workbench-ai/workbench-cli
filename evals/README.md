# Workbench CLI Skill-Eval Catalog

This directory holds the product-local ergonomics catalog for the `workbench` CLI skill. It tests whether the skill drives the shared local/hosted CLI correctly and whether agents use the eval-authoring docs when creating project evaluations.

It is not the public guide for creating Workbench project evaluations. Eval authoring guidance lives under `products/workbench-cli/docs/evals/` and is copied into installed skills as `references/docs/evals/`.

The evals intentionally cover the hosted deployment contract, local/hosted targeting rules, and eval-authoring entry points:

- configuring a hosted API target
- verifying or installing the published CLI package
- authoring evals from existing workflows
- authoring evals from `.docx`, `.xlsx`, `.pdf`, or `.pptx` file outputs
- creating a project and applying a spec
- creating project-specific Dockerfile environments
- syncing idempotent hosted source
- starting and watching eval/improve/compare workflows
- opening or returning Workbench Web URLs so an agent can keep an embedded browser synchronized with CLI work
- inspecting, previewing, and exporting hosted candidates

They should reward the public hosted workflow and eval-authoring behavior. Local `workbench init`, `workbench check`, `workbench dev improve`, checkpoint, and restore are valid public CLI behavior, but this catalog remains focused on hosted deployment tasks and eval authoring.

Validate the catalog from `products/workbench-cli`:

```bash
pnpm cli-skill-evals:validate
```
