---
name: azure-ai-projects-notebooks
description: "Use when creating or editing any notebook under notebooks/. Defines the notebook template for one azure-ai-projects client module: a single McKinsey-style title followed by the module name, a version-pinned API reference link, a first code cell of imports and environment variables, then one subtitled section per public method whose cell passes every signature parameter with an inline comment."
---

# azure-ai-projects notebooks

Each notebook in `notebooks/` covers one `azure-ai-projects` client module, such
as `client.agents`, and shows how to call each of its public methods.

## Structure

1. A title
2. A link to the module's API reference and the SDK version
3. A code cell with the imports and environment variables
4. For each public method, a subtitle and a code cell that calls it

## 1. Title

The notebook has a single `#` heading: a McKinsey-style action title, then the
module name.

```markdown
# Agents are named, versioned definitions that apps invoke by name: `client.agents`
```

An action title states the takeaway in one short sentence. It is not a topic
label. For example, write "Deployments name the models your project can call",
not "Deployments overview".

## 2. API reference and version

Put this line directly under the title:

```markdown
**API reference:** [`AgentsOperations`](https://learn.microsoft.com/en-us/python/api/azure-ai-projects/azure.ai.projects.operations.agentsoperations?view=azure-python) · **Version:** [`azure-ai-projects==2.7.0`](https://github.com/Azure/azure-sdk-for-python/tree/azure-ai-projects_2.7.0/sdk/ai/azure-ai-projects)
```

- The API reference link goes to the module's own page.
  [`docs/azure-ai-projects.md`](../../../docs/azure-ai-projects.md) lists the
  page for every module.
- The version is the pin in `requirements.txt`. Microsoft Learn shows only the
  latest release, so the version link goes to that release's tag. Update the
  link whenever the pin changes.

## 3. Imports and environment variables

The first code cell contains every import and reads every environment variable
the notebook uses. It also creates the client that the other cells share.

```python
import os

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition
from azure.identity import DefaultAzureCredential

PROJECT_ENDPOINT = os.environ["AZURE_AI_PROJECT_ENDPOINT"]
MODEL_DEPLOYMENT_NAME = os.environ["AZURE_DEPLOYMENT_NAME"]

client = AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential())
```

- Use the variable names from `.env.example`. Never hard-code endpoints, keys,
  or IDs.
- Add `allow_preview=True` only when a non-beta module uses a preview feature.
  `client.beta.*` modules don't need it.

## 4. One section per public method

List the module's public methods from the installed package, using the class
named on its API reference page:

```bash
python -c "import azure.ai.projects.operations as o; print([m for m in dir(o.AgentsOperations) if not m.startswith('_')])"
```

For each method, add a `##` subtitle with the method's name, followed by a code
cell that calls it:

```markdown
## `create_version`
```

```python
agent_version = client.agents.create_version(
    agent_name="notebook-demo-agent",  # Agent name; the first version creates the agent
    definition=PromptAgentDefinition(  # What the agent is: its model and instructions
        model=MODEL_DEPLOYMENT_NAME,
        instructions="Answer in one short sentence.",
    ),
    content_type="application/json",  # Request body format; keep the default
    metadata={"source": "notebook"},  # Up to 16 key-value pairs for your own use
    description="Created from the client.agents notebook",  # Human-readable description
    blueprint_reference=None,  # Agent identity blueprint to link; None links none
    draft=None,  # Preview: True saves a draft that "latest" resolution skips
)
print(agent_version.name, agent_version.version)
```

- Pass every parameter in the method's signature by keyword. Put each one on
  its own line with a short inline comment that says what it does.
- Skip `self` and `**kwargs`. For overloaded methods, use the typed keyword
  form, not the raw `body` form.
- Order the sections so the notebook runs from top to bottom: creates come
  first and deletes come last.
- Keep the output short: print one or two fields. Never print credentials,
  keys, or connection strings.
- Clear all outputs before you commit.
