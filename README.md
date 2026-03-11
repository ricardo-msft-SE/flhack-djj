# RAG Policy Assistant — Build Guide

> **Target audience:** Engineers and technical architects building a Retrieval-Augmented Generation (RAG) system for internal policy Q&A.

---

## Overview

This guide explains how to implement the RAG architecture sketched below using **Azure AI Foundry**, **VS Code**, and Azure CLIs. The system lets users query 30+ organizational policy PDFs and web-sourced policy data through a chat interface backed by Azure OpenAI (AOAI) and Azure AI Search.

### Choose your path

Each step offers two tracks — pick the one that fits your workflow:

| Track | Who it's for | What you write |
|---|---|---|
| **⚡ Programmatic** | Config-only, minimal scripting | CLI one-liners, `azd` templates, MCP prompts |
| **🔧 SDK / Code** | Full control, custom logic | Python SDK, YAML DAGs, Jinja2 prompts |

> **TL;DR for the programmatic path:** `azd init --template` → `azd up` → Foundry built-in ingestion pipeline → GitHub Copilot + MCP for day-2 ops. No custom Python required.

![RAG Architecture](./assets/architecture-sketch.jpg)

---

## Architecture

| Layer | Component | Azure Service |
|---|---|---|
| **LLM** | GPT model hosted at NWRDC | Azure OpenAI (AOAI) |
| **Retrieval** | Semantic + vector search | Azure AI Search |
| **Knowledge Base** | Chunked, vectorized policy content | AI Search Index |
| **Ingestion — PDFs** | 30+ policy & procedure documents | Azure AI Foundry Data Ingestion |
| **Ingestion — Web** | Live web policy pages | Custom crawler → AI Search indexer |
| **Orchestration** | Flow connecting retrieval → generation | Azure AI Foundry Prompt Flow |
| **Auth** | Secretless service-to-service | Managed Identity / DefaultAzureCredential |

### Data Flow

```
User Question (Q)
      │
      ▼
  ┌───────────────────────────────────────────────────┐
  │                       RAG                         │
  │                                                   │
  │  ┌──────────────────┐       ┌──────────────────┐  │
  │  │   AOAI (NWRDC)   │──────▶│   AI SEARCH      │  │
  │  │   LLM  GPT-x     │◀──────│                  │  │
  │  └──────────────────┘       └────────┬─────────┘  │
  │           │                          │             │
  │           │                 ┌────────▼─────────┐  │
  │           │                 │       KB          │  │
  │           │                 │  (vector index)   │  │
  │           │                 └────────▲─────────┘  │
  │  ┌────────┴────────┐                 │             │
  │  │  WEB POLICY     │─────────────────┤             │
  │  │  DATA           │                 │             │
  │  └─────────────────┘        ┌────────┴─────────┐  │
  │                             │       PDFs        │  │
  │                             │  30+ Policies &   │  │
  │                             │  Procedures       │  │
  │                             └──────────────────┘  │
  └───────────────────────────────────────────────────┘
      │
      ▼
  Answer (A) returned to User
```

---

## Prerequisites

| Tool | Minimum Version | Install |
|---|---|---|
| Azure CLI (`az`) | ≥ 2.60.0 | `winget install Microsoft.AzureCLI` |
| Azure Developer CLI (`azd`) | ≥ 1.9.0 | `winget install Microsoft.Azd` |
| Python | ≥ 3.11 | `winget install Python.Python.3.11` |
| Prompt Flow CLI (`pfcli`) | latest | `pip install promptflow[azure]` |
| VS Code | latest | [code.visualstudio.com](https://code.visualstudio.com) |

**VS Code extensions (install once):**

```bash
code --install-extension ms-azure-devops.azure-pipelines
code --install-extension prompt-flow.prompt-flow
code --install-extension ms-python.python
code --install-extension ms-azuretools.vscode-azurefunctions
```

### MCP Servers (Programmatic path)

For the config-only track, install the Azure MCP server so GitHub Copilot Agent Mode can provision and query resources on your behalf without you writing any SDK code.

```bash
# Install the Azure MCP server globally
npx @azure/mcp@latest server install

# Verify the server is registered
npx @azure/mcp@latest server list
```

After installation, open VS Code, switch Copilot to **Agent Mode**, and you can issue natural-language commands like:

> *"Create an Azure AI Search service named srch-rag-policy in resource group rg-rag-policy, Standard tier, East US"*

The MCP server translates the prompt to the correct `az` REST calls and executes them. The sub-sections below show the equivalent explicit commands for auditability.

**Useful MCP toolsets for this solution:**

| MCP server | Covers |
|---|---|
| `@azure/mcp` (Azure core) | Resource groups, RBAC, subscriptions |
| Azure AI Foundry MCP | Hub/project creation, model deployments, online endpoints |
| Azure AI Search MCP | Index creation, indexer management, datasource registration |

---

## Step 1 — Provision Infrastructure

> **⚡ Programmatic path — provision everything in two commands**
>
> Use the official `azd` RAG template. It creates the resource group, Foundry hub + project, AOAI deployments, AI Search service, and wires up managed identity — no manual `az` commands needed.
>
> ```bash
> # Authenticate
> az login
> azd auth login
>
> # Initialise from the community RAG template
> mkdir rag-policy && cd rag-policy
> azd init --template azure-samples/azure-openai-rag-workshop
>
> # Provision all resources in one shot (prompts for subscription / location)
> azd up
> ```
>
> Skip to [Step 2](#step-2--ingest-policy-documents-into-the-knowledge-base) after `azd up` succeeds. The template outputs all endpoint URLs as environment variables you can reference directly.
>
> **Day-2 changes via MCP (no portal, no extra commands):**
> Open Copilot Agent Mode and say: *"Scale my AI Search service srch-rag-policy to 2 replicas"* — the Azure MCP server handles the REST call.

---

The steps below describe the **equivalent explicit CLI commands** for environments where the template cannot be used (air-gapped, custom naming, partial adoption).

### 1.1 Authenticate

```bash
az login
azd auth login
```

### 1.2 Create a Resource Group

```bash
az group create \
  --name rg-rag-policy \
  --location eastus
```

### 1.3 Create an Azure AI Foundry Hub and Project

```bash
# Hub (shared infrastructure: networking, storage, key vault)
az ml workspace create \
  --kind hub \
  --resource-group rg-rag-policy \
  --name aih-rag-hub

# Project (scoped workspace for this solution)
HUB_ID=$(az ml workspace show \
  --name aih-rag-hub \
  --resource-group rg-rag-policy \
  --query id -o tsv)

az ml workspace create \
  --kind project \
  --resource-group rg-rag-policy \
  --name aip-rag-policy \
  --hub-id "$HUB_ID"
```

### 1.4 Deploy an Azure OpenAI Model

> If your AOAI resource is already provisioned at NWRDC, skip the `create` step and record the endpoint.

```bash
# Create the AOAI resource
az cognitiveservices account create \
  --name aoai-rag-policy \
  --resource-group rg-rag-policy \
  --kind OpenAI \
  --sku S0 \
  --location eastus

# Deploy the chat model
az cognitiveservices account deployment create \
  --resource-group rg-rag-policy \
  --name aoai-rag-policy \
  --deployment-name gpt-policy-chat \
  --model-name gpt-4o \
  --model-version "2024-08-06" \
  --model-format OpenAI \
  --sku-capacity 30 \
  --sku-name Standard

# Deploy the embedding model (used for vectorization)
az cognitiveservices account deployment create \
  --resource-group rg-rag-policy \
  --name aoai-rag-policy \
  --deployment-name text-embedding-3-large \
  --model-name text-embedding-3-large \
  --model-version "1" \
  --model-format OpenAI \
  --sku-capacity 120 \
  --sku-name Standard
```

### 1.5 Create Azure AI Search

```bash
az search service create \
  --name srch-rag-policy \
  --resource-group rg-rag-policy \
  --sku Standard \
  --location eastus \
  --partition-count 1 \
  --replica-count 1
```

---

## Step 2 — Ingest Policy Documents into the Knowledge Base

> **⚡ Programmatic path — use Foundry's built-in ingestion pipeline**
>
> Azure AI Foundry ships a managed **data ingestion + vectorization** pipeline. Point it at your files; it handles chunking, embedding, and index population with zero Python.
>
> ```bash
> # 1. Register your PDF folder as a Foundry data asset
> az ml data create \
>   --name policy-pdfs \
>   --path ./policies/ \
>   --type uri_folder \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
>
> # 2. Kick off the built-in vectorization pipeline
> az ml job create \
>   --file - <<'EOF'
> $schema: https://azuremlschemas.azureedge.net/latest/pipelineJob.schema.json
> type: pipeline
> experiment_name: policy-ingestion
> settings:
>   default_compute: serverless
> jobs:
>   ingest:
>     type: command
>     component: azureml://registries/azureml/components/llm_rag_crack_and_chunk_and_embed/versions/latest
>     inputs:
>       input_data:
>         type: uri_folder
>         path: azureml:policy-pdfs@latest
>       embeddings_model: azure_open_ai://deployment/text-embedding-3-large/model/text-embedding-3-large
>       asset_uri: azureml://datastores/workspaceblobstore/paths/policy-chunks
>       chunk_size: "1024"
>       chunk_overlap: "256"
>   index_and_register:
>     type: command
>     component: azureml://registries/azureml/components/llm_rag_update_acs_index/versions/latest
>     inputs:
>       embeddings: ${{parent.jobs.ingest.outputs.embeddings}}
>       acs_config:
>         endpoint: https://srch-rag-policy.search.windows.net
>         index_name: policy-index
> EOF
> ```
>
> Monitor progress:
>
> ```bash
> az ml job show --name <job-name> \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
> ```
>
> **Via Copilot + MCP:** *"Create an AI Search index named policy-index on srch-rag-policy with a content field and a 3072-dimension vector field using hnsw"* — the Azure AI Search MCP issues the index-creation REST call for you.

---

The steps below describe the **SDK / code track** for full control over chunking strategy, filtering, and custom field schemas.

### 2.1 PDF Ingestion

#### Option A — VS Code (Foundry Portal / AI Extension)

1. Open the **Azure AI Foundry** extension in VS Code.
2. Connect to the project `aip-rag-policy`.
3. Navigate to **Data** → **Add a data source** → **Upload files**.
4. Upload your PDF files from the `./policies/` directory.
5. Configure chunking: size = `1024` tokens, overlap = `256`.
6. Select **Azure AI Search** as the target, create index `policy-index`.
7. Choose `text-embedding-3-large` as the embedding model.

#### Option B — CLI / Python SDK

```bash
pip install azure-ai-projects azure-search-documents azure-identity openai
```

**`scripts/ingest_pdfs.py`**

```python
import os
import glob
from pathlib import Path

from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SimpleField, SearchableField,
    SearchField, SearchFieldDataType, VectorSearch,
    HnswAlgorithmConfiguration, VectorSearchProfile,
)
from openai import AzureOpenAI

SEARCH_ENDPOINT = os.environ["AZURE_SEARCH_ENDPOINT"]
AOAI_ENDPOINT   = os.environ["AZURE_OPENAI_ENDPOINT"]
INDEX_NAME      = "policy-index"
EMBED_MODEL     = "text-embedding-3-large"
CHUNK_SIZE      = 1024  # characters (adjust to tokens if using tiktoken)
CHUNK_OVERLAP   = 256

credential = DefaultAzureCredential()

aoai = AzureOpenAI(
    azure_endpoint=AOAI_ENDPOINT,
    azure_ad_token_provider=lambda: credential.get_token(
        "https://cognitiveservices.azure.com/.default"
    ).token,
    api_version="2024-08-01-preview",
)

def embed(text: str) -> list[float]:
    return aoai.embeddings.create(input=text, model=EMBED_MODEL).data[0].embedding

def chunk_text(text: str) -> list[str]:
    chunks, start = [], 0
    while start < len(text):
        end = min(start + CHUNK_SIZE, len(text))
        chunks.append(text[start:end])
        start += CHUNK_SIZE - CHUNK_OVERLAP
    return chunks

# ── Ensure the index exists ──────────────────────────────────────────────────
index_client = SearchIndexClient(endpoint=SEARCH_ENDPOINT, credential=credential)
if INDEX_NAME not in [idx.name for idx in index_client.list_indexes()]:
    index = SearchIndex(
        name=INDEX_NAME,
        fields=[
            SimpleField(name="id", type=SearchFieldDataType.String, key=True),
            SearchableField(name="content", type=SearchFieldDataType.String),
            SimpleField(name="sourcefile", type=SearchFieldDataType.String, filterable=True),
            SearchField(
                name="contentVector",
                type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
                searchable=True,
                vector_search_dimensions=3072,
                vector_search_profile_name="hnsw-profile",
            ),
        ],
        vector_search=VectorSearch(
            algorithms=[HnswAlgorithmConfiguration(name="hnsw")],
            profiles=[VectorSearchProfile(name="hnsw-profile", algorithm_configuration_name="hnsw")],
        ),
    )
    index_client.create_index(index)

# ── Ingest PDFs ──────────────────────────────────────────────────────────────
search_client = SearchClient(endpoint=SEARCH_ENDPOINT, index_name=INDEX_NAME, credential=credential)

for pdf_path in glob.glob("./policies/*.pdf"):
    import pdfplumber  # pip install pdfplumber
    with pdfplumber.open(pdf_path) as pdf:
        full_text = "\n".join(page.extract_text() or "" for page in pdf.pages)

    for i, chunk in enumerate(chunk_text(full_text)):
        doc_id = f"{Path(pdf_path).stem}_{i}"
        search_client.upload_documents([{
            "id": doc_id,
            "content": chunk,
            "sourcefile": Path(pdf_path).name,
            "contentVector": embed(chunk),
        }])
        print(f"  Indexed chunk {i} of {Path(pdf_path).name}")
```

Run:

```bash
export AZURE_SEARCH_ENDPOINT="https://srch-rag-policy.search.windows.net"
export AZURE_OPENAI_ENDPOINT="https://aoai-rag-policy.openai.azure.com"

python scripts/ingest_pdfs.py
```

### 2.2 Web Policy Data Ingestion

For live web-sourced policy pages, create an AI Search **web crawler indexer** or use a custom scraper that pushes chunks into the same `policy-index`.

```bash
# Register a web datasource with the AI Search REST API
SEARCH_ADMIN_KEY=$(az search admin-key show \
  --resource-group rg-rag-policy \
  --service-name srch-rag-policy \
  --query primaryKey -o tsv)

# For a custom scraper, push to the same index using the Python SDK above
# For scheduled web crawling, configure an AI Search indexer pointing at the datasource
```

> **Tip:** In production, avoid storing the admin key. Prefer managed identity with the `Search Service Contributor` role assigned to your ingestion service principal.

---

## Step 3 — Build the RAG Orchestration Flow

> **⚡ Programmatic path — clone a pre-built RAG flow template**
>
> Microsoft publishes a battle-tested RAG Prompt Flow template in the `azureml` registry. Import it directly — no DAG authoring, no Python retrieval node to write.
>
> ```bash
> # Pull the community RAG flow from the registry
> pfcli flow clone \
>   --source azureml://registries/azureml/models/rag-flow-template/versions/latest \
>   --output rag-policy-flow
>
> cd rag-policy-flow
>
> # Wire up your endpoints via environment variables (no secret storage)
> cat > .env <<'EOF'
> AZURE_OPENAI_ENDPOINT=https://aoai-rag-policy.openai.azure.com
> AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-policy-chat
> AZURE_OPENAI_EMBED_DEPLOYMENT=text-embedding-3-large
> AZURE_SEARCH_ENDPOINT=https://srch-rag-policy.search.windows.net
> AZURE_SEARCH_INDEX=policy-index
> EOF
>
> # Test immediately — no code changes required for the basic RAG pattern
> pfcli flow test --flow . \
>   --input question="What is the user account deletion policy?"
> ```
>
> **Customise prompts without touching code:** Open the flow in VS Code with the **Prompt Flow** extension, click the `generate_answer` node, and edit the system prompt inline in the visual editor. Changes are saved back to the YAML.
>
> **Via Copilot + MCP:** In Agent Mode say: *"Add a filter to my Prompt Flow's retrieve_context node that keeps only chunks where sourcefile contains 'HR'"* — Copilot reads your `flow.dag.yaml`, generates the diff, and applies it.

---

The steps below describe the **manual DAG definition** for teams that need custom retrieval logic.

### 3.1 Scaffold the Prompt Flow

```bash
# Initialise a chat flow skeleton
pfcli flow init --flow rag-policy-flow --type chat
cd rag-policy-flow
```

### 3.2 Add the AOAI Connection

```bash
pfcli connection create \
  --file connection.yaml \
  --set api_key="$(az cognitiveservices account keys list \
      --name aoai-rag-policy \
      --resource-group rg-rag-policy \
      --query key1 -o tsv)"
```

**`connection.yaml`** (committed without the key — key injected at runtime via managed identity in production):

```yaml
name: aoai_connection
type: AzureOpenAI
api_base: https://aoai-rag-policy.openai.azure.com/
api_type: azure
api_version: "2024-08-01-preview"
```

### 3.3 Define the Flow DAG

**`flow.dag.yaml`**

```yaml
inputs:
  question:
    type: string

outputs:
  answer:
    type: string
    reference: ${generate_answer.output}

nodes:
  - name: retrieve_context
    type: python
    source:
      type: code
      path: retrieve_context.py
    inputs:
      question: ${inputs.question}
      index_name: policy-index
      top_k: 5

  - name: generate_answer
    type: llm
    source:
      type: code
      path: generate_answer.jinja2
    inputs:
      deployment_name: gpt-policy-chat
      question: ${inputs.question}
      context: ${retrieve_context.output}
    connection: aoai_connection
    api: chat
```

### 3.4 Retrieval Node

**`retrieve_context.py`**

```python
import os
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from openai import AzureOpenAI

def retrieve_context(question: str, index_name: str, top_k: int = 5) -> str:
    credential = DefaultAzureCredential()

    aoai = AzureOpenAI(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_ad_token_provider=lambda: credential.get_token(
            "https://cognitiveservices.azure.com/.default"
        ).token,
        api_version="2024-08-01-preview",
    )

    embedding = aoai.embeddings.create(
        input=question,
        model="text-embedding-3-large",
    ).data[0].embedding

    search_client = SearchClient(
        endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
        index_name=index_name,
        credential=credential,
    )

    results = search_client.search(
        search_text=question,
        vector_queries=[
            VectorizedQuery(
                vector=embedding,
                k_nearest_neighbors=top_k,
                fields="contentVector",
            )
        ],
        select=["content", "sourcefile"],
        top=top_k,
    )

    return "\n\n---\n\n".join(
        f"[{r['sourcefile']}]\n{r['content']}" for r in results
    )
```

### 3.5 Answer Generation Prompt

**`generate_answer.jinja2`**

```jinja2
system:
You are a policy assistant for the organization. Answer the user's question
using ONLY the policy context provided below. Cite the source document name
when referencing specific policies.

If the answer is not found in the provided context, respond with:
"That information is not available in the current policy documents.
Please contact HR or your compliance team."

Context:
{{ context }}

user:
{{ question }}
```

---

## Step 4 — Run and Test Locally

> **⚡ Programmatic path — test and evaluate with pure CLI, no test harness code**
>
> ```bash
> # Single-turn interactive test
> pfcli flow test --flow rag-policy-flow \
>   --input question="What is the process for user account deletion?"
>
> # Batch eval — streams per-row results to stdout
> pfcli run create \
>   --flow rag-policy-flow \
>   --data tests/eval_questions.jsonl \
>   --column-mapping question='${data.question}' \
>   --stream
>
> # Attach a built-in groundedness evaluator (no custom eval code needed)
> pfcli run create \
>   --flow azureml://registries/azureml/models/gpt-groundedness/versions/latest \
>   --data tests/eval_questions.jsonl \
>   --run <previous-run-id> \
>   --column-mapping question='${data.question}' answer='${run.outputs.answer}'
> ```
>
> **Generate a starter eval dataset via Copilot Chat (no scripting):**
> Paste this into Copilot Chat and save the output to `tests/eval_questions.jsonl`:
> > *"Generate 10 Q&A pairs in JSONL format (`question`, `ground_truth` fields) covering HR policy topics: account deletion, PTO, data retention, acceptable use."*
>
> **VS Code shortcut:** Install the **Prompt Flow** extension → right-click any node → **Run from here** to replay a single step with cached upstream outputs.

Standard commands for reference:

```bash
# Interactive single-turn test
pfcli flow test --flow . \
  --input question="What is the process for user account deletion?"

# Batch evaluation against a JSONL test set
pfcli run create \
  --flow . \
  --data tests/eval_questions.jsonl \
  --column-mapping question='${data.question}' \
  --stream
```

In VS Code, open the flow with the **Prompt Flow** panel to trace each node, inspect intermediate outputs, and iterate on prompts without leaving the editor.

---

## Step 5 — Deploy to Azure AI Foundry

> **⚡ Programmatic path — `azd deploy` or single `az ml` sequence**
>
> If you used the `azd` template in Step 1, deployment is one command after any flow or config change:
>
> ```bash
> azd deploy
> ```
>
> `azd` reads your `azure.yaml`, builds the container, pushes to Azure Container Registry, creates/updates the managed online endpoint, and shifts traffic — all automatically.
>
> For environments not using the `azd` template:
>
> ```bash
> az ml online-endpoint create \
>   --name rag-policy-endpoint \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
>
> az ml online-deployment create \
>   --file deployment.yaml \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy \
>   --all-traffic
> ```
>
> **Canary / blue-green via CLI (no portal):**
>
> ```bash
> # Deploy v2 with 10 % traffic
> az ml online-deployment create \
>   --file deployment-v2.yaml \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
>
> az ml online-endpoint update \
>   --name rag-policy-endpoint \
>   --traffic "rag-policy-deployment=90 rag-policy-deployment-v2=10" \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
>
> # Promote v2 to 100 % after validation
> az ml online-endpoint update \
>   --name rag-policy-endpoint \
>   --traffic "rag-policy-deployment-v2=100" \
>   --resource-group rg-rag-policy \
>   --workspace-name aip-rag-policy
> ```
>
> **Via Copilot + MCP:** *"Deploy my Prompt Flow in rag-policy-flow to the rag-policy-endpoint endpoint in my aip-rag-policy Foundry project"* — the Foundry MCP server handles the REST calls.

---

The steps below describe the explicit artifact-build + deploy sequence for full control.

### 5.1 Build a Deployable Artifact

```bash
pfcli flow build \
  --source . \
  --output ../dist/rag-policy-flow \
  --format docker
```

### 5.2 Create a Managed Online Endpoint

```bash
az ml online-endpoint create \
  --name rag-policy-endpoint \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy
```

### 5.3 Deploy

**`deployment.yaml`**

```yaml
name: rag-policy-deployment
endpoint_name: rag-policy-endpoint
model:
  path: ../dist/rag-policy-flow
environment_variables:
  AZURE_OPENAI_ENDPOINT: https://aoai-rag-policy.openai.azure.com
  AZURE_SEARCH_ENDPOINT: https://srch-rag-policy.search.windows.net
instance_type: Standard_DS3_v2
instance_count: 1
```

```bash
az ml online-deployment create \
  --file deployment.yaml \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy \
  --all-traffic
```

### 5.4 Smoke Test

```bash
ENDPOINT_URL=$(az ml online-endpoint show \
  --name rag-policy-endpoint \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy \
  --query scoring_uri -o tsv)

az ml online-endpoint invoke \
  --name rag-policy-endpoint \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy \
  --request-file tests/sample_request.json
```

**`tests/sample_request.json`**

```json
{
  "question": "What is the process for user account deletion?"
}
```

---

---

## Day-2 Operations — Programmatic Reference

Common operational tasks expressible as single CLI commands or MCP prompts. No portal, no custom scripts.

### Update the system prompt without redeploying

```bash
# Edit generate_answer.jinja2 locally, validate, then push
pfcli flow test --flow rag-policy-flow   # confirm locally
azd deploy                               # push updated flow artifact
```

Or in Copilot Agent Mode: *"Update the system prompt in generate_answer.jinja2 to also cite the section number of the policy, then redeploy"*

### Re-index after new PDFs are added

```bash
# Upload new files to the Foundry data asset blob location
az storage blob upload-batch \
  --source ./policies/new/ \
  --destination "<blob-container-url>"

# Resubmit the ingestion pipeline (picks up new files automatically)
az ml job create --file ingestion-pipeline.yaml \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy
```

### Verify ingestion with a spot-check query

```bash
SEARCH_KEY=$(az search query-key list \
  --resource-group rg-rag-policy \
  --service-name srch-rag-policy \
  --query "[0].key" -o tsv)

curl -s -X POST \
  "https://srch-rag-policy.search.windows.net/indexes/policy-index/docs/search?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: $SEARCH_KEY" \
  -d '{"search": "account deletion", "top": 3, "select": "sourcefile,content"}' \
  | jq '.value[] | .sourcefile'
```

> Use a **query key** (read-only) rather than the admin key for spot-checks and application use.

### Scale the endpoint

```bash
az ml online-deployment update \
  --name rag-policy-deployment \
  --endpoint-name rag-policy-endpoint \
  --instance-count 3 \
  --resource-group rg-rag-policy \
  --workspace-name aip-rag-policy
```

---

## Security Checklist

| Control | Implementation |
|---|---|
| **No secrets in code** | Use `DefaultAzureCredential` — no API keys in source |
| **Managed Identity** | Assign system-assigned identity to the Foundry deployment; grant `Cognitive Services OpenAI User` and `Search Index Data Reader` roles |
| **Content Safety** | Enable Azure AI Content Safety on the AOAI deployment (prompt shield, jailbreak detection) |
| **Network isolation** | Restrict AI Search to private endpoint or IP allowlist |
| **Least-privilege RBAC** | `Cognitive Services OpenAI User` for inference, `Search Index Data Reader` for retrieval, `Storage Blob Data Reader` for ingestion |
| **Input validation** | Validate and sanitize user input before passing to the LLM to prevent prompt injection |
| **Audit logging** | Enable diagnostic settings on AOAI and AI Search → Log Analytics workspace |

---

## Repository Structure

```
.
├── README.md
├── assets/
│   └── architecture-sketch.jpg
├── scripts/
│   ├── ingest_pdfs.py          # PDF chunking + vectorization → AI Search
│   └── ingest_web.py           # Web policy scraper → AI Search
├── rag-policy-flow/
│   ├── flow.dag.yaml           # Prompt Flow DAG definition
│   ├── retrieve_context.py     # Vector retrieval node
│   ├── generate_answer.jinja2  # System + user prompt template
│   └── connection.yaml         # AOAI connection descriptor (no secrets)
├── infra/
│   ├── main.bicep              # IaC: AOAI + AI Search + Storage + Foundry
│   └── main.parameters.json
├── tests/
│   ├── eval_questions.jsonl    # Batch evaluation dataset
│   └── sample_request.json     # Endpoint smoke test payload
└── deployment.yaml             # Foundry managed online deployment spec
```

---

## Key References

- [Azure AI Foundry documentation](https://learn.microsoft.com/azure/ai-foundry/)
- [Prompt Flow — VS Code extension](https://marketplace.visualstudio.com/items?itemName=prompt-flow.prompt-flow)
- [Azure AI Search — vector search overview](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/)
- [DefaultAzureCredential (Python)](https://learn.microsoft.com/azure/developer/python/sdk/authentication-overview)
- [Azure AI Content Safety](https://learn.microsoft.com/azure/ai-services/content-safety/)
