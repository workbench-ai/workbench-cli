# Workbench Eval Authoring

This directory is the canonical guide for creating hosted Workbench evaluations. Use it when you need a new `workbench.yaml`, need to run and score a candidate with an existing workflow, or need to evaluate files such as `.docx`, `.xlsx`, `.pdf`, or `.pptx` produced by a candidate.

The candidate is the mutable thing Workbench improves. Evals live in `tasks`, `run`, and `grade`; do not make evaluator code the candidate unless the evaluator itself is the product being improved.

Workbench eval authoring has two normal starting points:

- Existing workflow: start with [from-existing-workflow.md](from-existing-workflow.md) when there is already a script, test command, benchmark harness, or manual scoring process.
- File-output tasks: start with [from-file-outputs.md](from-file-outputs.md) when tasks, outputs, examples, goldens, reports, workbooks, decks, PDFs, or other opaque files affect runtime prerequisites.

Before writing a spec, read:

- [spec-syntax.md](spec-syntax.md) for the local/hosted YAML shape.
- [runner-contract.md](runner-contract.md) for staged directories, phase visibility, and output files.
- [tasks-and-fixtures.md](tasks-and-fixtures.md) for task directory layout and expected files.
- [run-and-inspect.md](run-and-inspect.md) for the hosted CLI loop.

File-specific guidance lives under [file-recipes/](file-recipes/):

- [docx.md](file-recipes/docx.md)
- [xlsx.md](file-recipes/xlsx.md)
- [pdf.md](file-recipes/pdf.md)
- [pptx.md](file-recipes/pptx.md)

The authoring goal is not to make a perfect evaluator on the first pass. First make a small smoke eval that proves Workbench can stage the candidate, run it on a task, keep expected files private to graders, write a finite numeric score, and produce inspectable runner files. Default to rubric grading for qualitative behavior; use command grading only for objective deterministic checks or an existing scorer. Then tighten scoring and task coverage.
