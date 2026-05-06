# From Existing Workflow

Use this path when there is already a script, test suite, benchmark command, or manual scoring process. The goal is to wrap that workflow in the Workbench runner/grader contract without changing it more than needed.

## Process

1. Identify the command that already produces output files or pass/fail signals.
2. Put mutable files under the candidate directory.
3. Put public fixtures under `tasks/<case>/input/` and private expected outputs, tolerances, or task-specific grader scripts for intentional command graders under `tasks/<case>/expected/`.
4. Write a runner command that executes the existing workflow and writes inspectable files under `/workspace/output`.
5. Reuse the existing deterministic scorer as a command grader when one exists, or use a rubric grader when the workflow output needs qualitative review. Command graders read `/workspace/input/task/input`, `/workspace/input/task/expected`, and `/workspace/input/runner-output`, then write `/workspace/output/scorecard.json`.
6. Validate the spec, sync source, and run one sample.

## Adapter Pattern

The runner can preserve the existing command. The grader translates its result into Workbench scorecard JSON:

```python
import json
import subprocess
from pathlib import Path

candidate_dir = Path("/workspace/input/candidate")
output_dir = Path("/workspace/output")
output_dir.mkdir(parents=True, exist_ok=True)

completed = subprocess.run(
    ["python", "scripts/check.py"],
    cwd=candidate_dir,
    text=True,
    capture_output=True,
)

(output_dir / "workflow-result.json").write_text(json.dumps({
    "exitCode": completed.returncode,
    "stdout": completed.stdout[-4000:],
    "stderr": completed.stderr[-4000:],
}), encoding="utf8")
```

Put task-specific grading logic under `tasks/<case>/expected/grade_workflow.py` only for intentional command graders, usually to preserve an existing deterministic scoring workflow. The command grader writes:

```python
import json
from pathlib import Path

workflow_result = json.loads((Path("/workspace/input/runner-output") / "workflow-result.json").read_text())
expected_dir = Path("/workspace/input/task/expected")
score = 1.0 if workflow_result["exitCode"] == 0 else 0.0
(Path("/workspace/output") / "scorecard.json").write_text(json.dumps({
    "score": score,
    "summary": "Existing workflow reached full score." if score == 1.0 else "Existing workflow scored below full credit.",
    "feedback": workflow_result,
}, indent=2), encoding="utf8")
```

## Spec

```yaml
version: 11
name: existing-workflow
candidate:
  path: candidate/existing-workflow
  editable:
    - scripts/run_existing_workflow.py
tasks:
  path: tasks
environment:
  dockerfile: environment/Dockerfile
objective:
  goal: Improve the existing workflow wrapper.
  metric: score
  direction: maximize
propose:
  use: command
  with:
    command: python -c "import json; from pathlib import Path; p=Path('/workspace/input/candidate/scripts/run_existing_workflow.py'); content=p.read_text().rstrip() + '\n# Workbench candidate revision.\n'; Path('/workspace/output/candidate_patch.json').write_text(json.dumps({'files':[{'path':'scripts/run_existing_workflow.py','content':content,'encoding':'utf-8'}],'fileChanges':['scripts/run_existing_workflow.py']}), encoding='utf-8')"
run:
  use: command
  with:
    command: python /workspace/input/candidate/scripts/run_existing_workflow.py
grade:
  use: command
  with:
    command: python /workspace/input/task/expected/grade_workflow.py
```

Use [runner-contract.md](runner-contract.md) to keep the phase outputs valid.
