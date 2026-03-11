# RAG Policy Assistant — Build Guide

> **Target audience:** Engineers and technical architects building a Retrieval-Augmented Generation (RAG) system for internal policy Q&A.

---

## Overview

This guide explains how to implement the RAG architecture sketched below using **Azure AI Foundry**, **VS Code**, and Azure CLIs. The system lets users query 30+ organizational policy PDFs and web-sourced policy data through a chat interface backed by Azure OpenAI (AOAI) and Azure AI Search.

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

---

## Step 1 — Provision Infrastructure

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
