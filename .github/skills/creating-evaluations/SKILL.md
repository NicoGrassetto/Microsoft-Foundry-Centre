---
name: creating-evaluations
description: "Use when creating or editing an agent's evaluate_agent.py under src/agents/<agent-name>/, or when choosing evaluation rules, schedules, red teaming, or evaluators for an agent."
---

# Creating evaluations

Each agent keeps its evaluation code in its own folder, next to the script that
creates it. There is no separate evaluations folder.

Work in two phases:

1. **List the options.** Find what the pinned SDK versions support for
   evaluation rules, schedules, red teaming, and evaluators, and ask the user
   which ones to use.
2. **Write `evaluate_agent.py`** using only the options the user chose.

## Layout

```
src/agents/<agent-name>/
├── create_agent.py      # Creates the Foundry agent and defines AGENT_NAME
├── evaluate_agent.py    # Creates, registers, and runs the agent's evaluations
├── prompts/
│   └── v1_instructions.md
├── skills/              # Optional
└── create_routine.py    # Optional
data/<data-folder>/datasets/
├── golden_dataset.jsonl
├── validation_dataset.jsonl
├── evaluation_dataset.jsonl
└── synthetic/
```

- `<agent-name>` is both the folder name and the Foundry agent name, such as
  `trail-guide-agent`.
- `<data-folder>` is the agent name without `-agent`, in snake_case:
  `trail-guide-agent` uses `data/trail_guide/` and `weather-agent` uses
  `data/weather/`.
- `evaluate_agent.py` holds all of an agent's evaluation code: datasets,
  evaluators, evals and their runs, evaluation rules, schedules, red teaming,
  and local evaluations.

## Phase 1: List the options

Do this phase before you add or change any evaluation rule, schedule, red team,
or evaluator. You can skip it only if you already listed the options for the
same pinned versions earlier in the conversation.

Microsoft Learn documents only the latest SDK release, and preview APIs change
between releases, so take the options from the versions this project pins.

### 1. Find the pinned versions

Read the `azure-ai-projects`, `azure-ai-evaluation`, and `openai` pins in
`requirements.txt`. If a pin is missing, ask the user which version to use.
Never assume the latest release.

### 2. List what the installed SDK supports

Run this from the repository root. It reads the packages installed in `.venv`
and makes no network calls. If `.venv` is missing, create it and run
`python -m pip install -r requirements.txt` first.

```bash
.venv/bin/python -W ignore - <<'EOF'
import enum
import importlib
import inspect
import typing

import azure.ai.projects.models as models
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from openai.types.eval_create_params import TestingCriterion
from openai.types.evals.run_create_params import DataSource

RESOURCES = ("EvaluationRule", "EvaluationRuleFilter", "Schedule", "RedTeam", "EvaluatorVersion", "EvaluationTaxonomy")
KEYWORDS = ("EvaluationRule", "Trigger", "Recurrence", "ScheduleTask", "RiskCategory", "AttackStrategy",
            "RedTeam", "Taxonomy", "Evaluator", "EvaluationLevel", "EvaluationTarget")
UNRELATED = ("Routine", "VoiceAgent", "AgentInsight", "Realtime")


def fields(cls):
    names = {}
    for base in reversed(cls.__mro__):
        if base.__module__.startswith("azure.ai.projects.models"):
            names.update(getattr(base, "__annotations__", {}))
    return ", ".join(name for name in names if not name.startswith("_"))


def type_values(types):
    return ", ".join(typing.get_type_hints(t)["type"].__args__[0] for t in types)


client = AIProjectClient(
    endpoint="https://example.services.ai.azure.com/api/projects/example",
    credential=DefaultAzureCredential(),
)
for path in ("evaluation_rules", "beta.schedules", "beta.red_teams", "beta.evaluation_taxonomies", "beta.evaluators"):
    operations = client
    for part in path.split("."):
        operations = getattr(operations, part)
    print(f"client.{path}: {', '.join(m for m in dir(operations) if not m.startswith('_'))}")

for name, cls in inspect.getmembers(models, inspect.isclass):
    if name in RESOURCES:
        print(f"{name}({fields(cls)})")
    elif any(k in name for k in KEYWORDS) and not name.startswith(UNRELATED):
        if issubclass(cls, enum.Enum):
            print(f"{name}: {', '.join(str(e.value) for e in cls)}")
        elif cls.__dict__.get("__mapping__"):
            for kind, subclass in cls.__mapping__.items():
                print(f"{name}[{kind}] = {subclass.__name__}({fields(subclass)})")

run_sources = [getattr(models, n) for n in dir(models) if n.endswith("RunDataSource")]
print("Eval run data sources:", type_values([*typing.get_args(DataSource), *run_sources]))
print("OpenAI graders:", type_values(typing.get_args(TestingCriterion)))
local = importlib.import_module("azure.ai.evaluation")
print("Local evaluators:", ", ".join(n for n in dir(local) if n.endswith(("Evaluator", "Grader")) and not n.startswith("_")))
try:
    importlib.import_module("azure.ai.evaluation.red_team")
    print("Local red teaming: available")
except ImportError as error:
    print(f"Local red teaming: unavailable ({error})")
EOF
```

If an azd environment is configured, also list the built-in evaluator catalog.
This call only reads from the project:

```bash
azd exec -- .venv/bin/python -c '
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient(endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
for evaluator in client.beta.evaluators.list(type="builtin"):
    print(evaluator.name, evaluator.categories, evaluator.supported_evaluation_levels)
'
```

### 3. Read the docs for the pinned versions

Read the samples and changelogs at each release tag. They show how each option
is used and which options are new, renamed, removed, or preview. Replace
`<version>` with the pin, such as `2.7.0`:

- `https://github.com/Azure/azure-sdk-for-python/tree/azure-ai-projects_<version>/sdk/ai/azure-ai-projects/samples/evaluations`
- `https://github.com/Azure/azure-sdk-for-python/tree/azure-ai-projects_<version>/sdk/ai/azure-ai-projects/samples/red_team`
- `https://github.com/Azure/azure-sdk-for-python/blob/azure-ai-projects_<version>/sdk/ai/azure-ai-projects/CHANGELOG.md`
- `https://github.com/Azure/azure-sdk-for-python/blob/azure-ai-evaluation_<version>/sdk/evaluation/azure-ai-evaluation/CHANGELOG.md`

Then read Microsoft Learn for concepts, preview status, and region, model, and
cost requirements. Learn describes the latest release, so note anything it
documents that the pinned versions don't have. If a page has moved, search
Learn for its title.

| Category | Microsoft Learn pages |
| --- | --- |
| Evaluation rules | [Set up continuous evaluation](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard#set-up-continuous-evaluation), [Human evaluation](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/human-evaluation) |
| Schedules | [Create a recurring evaluation](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard#create-a-recurring-evaluation) |
| Red teaming | [AI Red Teaming Agent](https://learn.microsoft.com/en-us/azure/foundry/concepts/ai-red-teaming-agent), [Run in the cloud](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-ai-red-teaming-cloud), [Run locally](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-scans-ai-red-teaming-agent) |
| Evaluators | [Built-in evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/built-in-evaluators), [Custom evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/custom-evaluators), [Rubric evaluators](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/rubric-evaluators), [Azure OpenAI graders](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-evaluators/azure-openai-graders) |
| All of them | [Cloud evaluation](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/cloud-evaluation), [Regions, limits, and virtual networks](https://learn.microsoft.com/en-us/azure/foundry/concepts/evaluation-regions-limits-virtual-network) |

### 4. List the options and ask

Start with the pinned versions, then give one table per category, in this
order: evaluation rules, schedules, red teaming, evaluators.

| Option | What it does | Settings | Status | Source |
| --- | --- | --- | --- | --- |
| Human evaluation action | Asks human reviewers to score agent responses with a template | `template_id` | Preview | [Human evaluation](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/human-evaluation) |

- **Evaluation rules:** a row for each event type and each action.
- **Schedules:** a row for each trigger, recurrence, and task type, and the eval
  run data sources a scheduled run can use.
- **Red teaming:** a row for each way to run a scan (a red team, an eval run
  with a red-team data source, or locally), with the targets each one supports.
  Give risk categories and attack strategies as comma-separated lists.
- **Evaluators:** a row for each built-in evaluator category, custom evaluator
  definition type, OpenAI grader, and group of local evaluators.
- **Status** is Preview if the option is under `client.beta`, has "preview" in
  its name, needs `allow_preview=True`, or Learn marks it preview. Otherwise
  it's GA.
- **Source** links to the sample, changelog entry, or page that documents the
  option.

End with the options that Learn documents but the pinned versions don't have.

Then ask the user which options to use in each category. Don't write
`evaluate_agent.py` until they answer.

## Phase 2: Write `evaluate_agent.py`

### Sections

The script is split into sections, and the command line chooses which ones run.
Include only the sections the agent needs. Rules, schedules, red teams, and
evaluators use only the options the user chose in phase 1.

| Section | What it does | Running it again |
| --- | --- | --- |
| `datasets` | Uploads the golden, validation, and evaluation datasets | Uploads only versions that don't exist yet. To change a dataset, bump its version in `DATASET_VERSIONS`. |
| `evaluators` | Creates the custom evaluators | Creates a new version of each custom evaluator, so run it only after changing one. |
| `evals` | Creates the eval | Does nothing if an eval named `EVAL_NAME` exists. `evals.update` can change only an eval's name and metadata, so to change its testing criteria, bump the `v<n>` in `EVAL_NAME`. |
| `runs` | Starts offline eval runs | Starts new runs every time. |
| `rules` | Creates the evaluation rules (online) | Updates the rules in place and points them at the current eval. |
| `schedules` | Creates the schedules (offline, recurring) | Updates the schedules in place and points them at the current eval. |
| `red-teams` | Starts red-team scans | Starts new scans every time. |
| `local` | Runs `azure-ai-evaluation` evaluators on this machine | Runs them again. |

### Template

Follow the shape of `create_agent.py`: module constants, functions marked with
the same comment banners, and a `main()` function.

```python
import argparse
import logging
import os
from pathlib import Path

from azure.ai.evaluation import evaluate
from azure.ai.projects import AIProjectClient
from azure.core.exceptions import ResourceNotFoundError
from azure.identity import DefaultAzureCredential
from openai import OpenAI
from openai.types import EvalListResponse

AGENT_DIR = Path(__file__).resolve().parent
AGENT_NAME = "trail-guide-agent"
AGENT_TITLE = "Trail Guide"
DATASETS_DIR = AGENT_DIR.parents[2] / "data" / "trail_guide" / "datasets"
DATASET_VERSIONS = {"golden": "1", "validation": "1", "evaluation": "1"}
EVAL_NAME = f"{AGENT_TITLE} Quality v1"


def require_eval(openai_client: OpenAI) -> EvalListResponse:
    for evaluation in openai_client.evals.list():
        if evaluation.name == EVAL_NAME:
            return evaluation
    raise SystemExit(f"Eval {EVAL_NAME!r} not found. Run the evals section first.")


# ------- Datasets --------
def update_datasets(client: AIProjectClient, openai_client: OpenAI) -> None:
    for split, version in DATASET_VERSIONS.items():
        name = f"{AGENT_NAME}-{split}"
        try:
            client.datasets.get(name=name, version=version)
        except ResourceNotFoundError:
            client.datasets.upload_file(
                name=name,
                version=version,
                file_path=str(DATASETS_DIR / f"{split}_dataset.jsonl"),
            )
# --------


# ------- Custom evaluators --------
def update_evaluators(client: AIProjectClient, openai_client: OpenAI) -> None:
    client.beta.evaluators.create_version(
        name=f"{AGENT_NAME}-grounding",
        evaluator_version=...,
    )
# --------


# ------- Evals --------
def update_evals(client: AIProjectClient, openai_client: OpenAI) -> None:
    if any(evaluation.name == EVAL_NAME for evaluation in openai_client.evals.list()):
        return
    openai_client.evals.create(
        name=EVAL_NAME,
        data_source_config=...,
        testing_criteria=...,
    )
# --------


# ------- Eval runs (offline/batch) --------
def start_runs(client: AIProjectClient, openai_client: OpenAI) -> None:
    evaluation = require_eval(openai_client)
    dataset = client.datasets.get(name=f"{AGENT_NAME}-golden", version=DATASET_VERSIONS["golden"])
    openai_client.evals.runs.create(
        eval_id=evaluation.id,
        name=...,
        data_source=...,  # Reads dataset.id
    )
# --------


# ------- Evaluation rules (online) --------
def update_rules(client: AIProjectClient, openai_client: OpenAI) -> None:
    evaluation = require_eval(openai_client)
    client.evaluation_rules.create_or_update(
        id=f"{AGENT_NAME}-continuous-quality",
        evaluation_rule=...,  # Its action uses evaluation.id
    )
# --------


# ------- Schedules (offline, recurring) --------
def update_schedules(client: AIProjectClient, openai_client: OpenAI) -> None:
    evaluation = require_eval(openai_client)
    client.beta.schedules.create_or_update(
        schedule_id=f"{AGENT_NAME}-golden-nightly",
        schedule=...,  # Its task uses evaluation.id and a dataset ID
    )
# --------


# ------- Red teaming --------
def start_red_teams(client: AIProjectClient, openai_client: OpenAI) -> None:
    client.beta.red_teams.create(red_team=...)  # Or an eval run with a red-team data source
# --------


# ------- Local evaluations --------
def run_local_evaluations(client: AIProjectClient, openai_client: OpenAI) -> None:
    evaluate(data=str(DATASETS_DIR / "golden_dataset.jsonl"), evaluators=...)
# --------


SECTIONS = {
    "datasets": update_datasets,
    "evaluators": update_evaluators,
    "evals": update_evals,
    "runs": start_runs,
    "rules": update_rules,
    "schedules": update_schedules,
    "red-teams": start_red_teams,
    "local": run_local_evaluations,
}


def main() -> None:
    parser = argparse.ArgumentParser(description=f"Create and run the {AGENT_NAME} evaluations.")
    parser.add_argument(
        "sections",
        nargs="+",
        choices=[*SECTIONS, "all"],
        help="Sections to run. They always run in the order listed.",
    )
    args = parser.parse_args()

    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)-8s %(name)s | %(message)s",
    )

    with (
        DefaultAzureCredential() as credential,
        AIProjectClient(
            endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
            credential=credential,
        ) as client,
        client.get_openai_client() as openai_client,
    ):
        for name, run_section in SECTIONS.items():
            if "all" in args.sections or name in args.sections:
                logging.info("Running section %s", name)
                run_section(client, openai_client)


if __name__ == "__main__":
    main()
```

- `...` stands for the rest of each call's arguments.
- Each section function takes both clients, even if it uses one or neither, so
  `main()` can call any of them the same way.
- Any section can run without the others, so a section never uses values from
  another section. It looks up what it needs by name and stops with a message
  that names the section to run first, as `require_eval` does.
- `AGENT_NAME` matches the constant in `create_agent.py`.
- Add `allow_preview=True` to `AIProjectClient` only when a non-beta call uses a
  preview feature, such as `HumanEvaluationPreviewRuleAction`. `client.beta.*`
  calls don't need it.

### Running it

Agent folder names contain hyphens, so they are not Python packages. Run the
script from the repository root, the same way as `create_agent.py`, and name
the sections to run:

```bash
azd exec -- env PYTHONPATH=src python src/agents/<agent-name>/evaluate_agent.py <section> [<section> ...]
```

- At least one section is required, so a bare command can't start runs or scans
  by accident. Pass `all` to run every section, or `--help` to list them.
- Sections run in the order of the table above, whatever order you give them in.
- No section creates or changes the agent; `create_agent.py` does that.

For example:

```bash
# Upload new dataset versions and change nothing else
azd exec -- env PYTHONPATH=src python src/agents/trail-guide-agent/evaluate_agent.py datasets

# After bumping the eval's version: create the new eval, then point the rules and schedules at it
azd exec -- env PYTHONPATH=src python src/agents/trail-guide-agent/evaluate_agent.py evals rules schedules

# Start an offline run of the current eval
azd exec -- env PYTHONPATH=src python src/agents/trail-guide-agent/evaluate_agent.py runs
```

## Naming

Every name, ID, and display name starts with the agent, so each agent's objects
group together in the project. Build them from `AGENT_NAME` and `AGENT_TITLE`,
for example `f"{AGENT_NAME}-golden"`. Testing criterion names are the exception
because they live inside an eval.

| Object | Field | Pattern | Example |
| --- | --- | --- | --- |
| Dataset | `name` | `<agent-name>-<split>` | `trail-guide-agent-golden` |
| Dataset | `version` | `"1"`, `"2"`, ..., set in `DATASET_VERSIONS` | `"1"` |
| Custom evaluator | `name` | `<agent-name>-<aspect>` | `weather-agent-grounding` |
| Custom evaluator | `display_name` | `<Agent Title> <Aspect>` | `Weather Grounding` |
| Eval | `name` | `<Agent Title> <Purpose> v<n>` | `Trail Guide Quality v1` |
| Testing criterion | `name` | Built-in evaluator ID without `builtin.`, or the custom evaluator's `<aspect>` | `groundedness`, `request_adherence` |
| Eval run | `name` | `<agent-name>-<split>-<agent version or git sha>` | `trail-guide-agent-golden-v3` |
| Evaluation rule | `id` | `<agent-name>-continuous-<purpose>` | `trail-guide-agent-continuous-quality` |
| Evaluation rule | `display_name` | Same as its eval's name | `Trail Guide Quality v1` |
| Schedule | `schedule_id` | `<agent-name>-<split>-<cadence>` | `trail-guide-agent-golden-nightly` |
| Schedule | `display_name` | `<Agent Title> <Split> <Cadence>` | `Trail Guide Golden Nightly` |
| Red team | `display_name` | `<Agent Title> <Purpose> Scan` | `Trail Guide Jailbreak Scan` |

- `<Agent Title>` is the agent name in Title Case without "Agent", such as
  `Trail Guide`.
- `<split>` is `golden`, `validation`, `evaluation`, or `synthetic`, matching the
  data file it comes from.
- Names and IDs are kebab-case: lowercase letters, digits, and hyphens. Testing
  criterion names are snake_case, like the built-in evaluator IDs
  (`builtin.task_adherence` becomes `task_adherence`).
- Never reuse a name for different content. To change a dataset, bump its
  version in `DATASET_VERSIONS`. To change an eval's testing criteria, bump the
  `v<n>` in `EVAL_NAME`. Each `evaluators` run creates a new evaluator version.
