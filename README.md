# Data Q&A Agent Notebooks

> Upload a spreadsheet (CSV or Excel), ask questions about it in everyday language, and get answers, charts, or new files back — powered by **Microsoft Foundry Agent** (via code).

This repo contains **two notebooks** that do the same thing using different Foundry APIs:

| Notebook | Foundry API | File upload approach | Multi-turn |
|----------|-------------|---------------------|------------|
| `data_qa_agent.ipynb` | **New Experience** (Responses API + Conversations) | File bound to **Code Interpreter tool** → agent must be **republished** each time a new file is uploaded | Conversations API |
| `data_qa_agent_classic.ipynb` | **Classic** (Threads / Runs / Messages) | File attached to the **message** via `MessageAttachment` → agent is created **once** and never changes | Threads |

### Why two notebooks?

The **new experience** (Responses API + Conversations) is the recommended path going forward — it offers versioned agents, modern state management, and new features. However, Code Interpreter files can only be bound at the agent level, which means a new agent version must be published every time you upload a different file.

The **classic** API (deprecated, retiring March 31 2027) lets you attach files directly to a message in a thread. The agent definition never changes — you create it once, and every subsequent file is just a message attachment. This is simpler for the "upload different CSVs ad-hoc" scenario but is on a deprecation path.

---

## Quick-Start Checklist

If you already have Azure and Python set up, here's the short version:

**New Experience notebook:**

| Step | What to do |
|------|-----------|
| 1 | Clone this repo and `cd` into the folder |
| 2 | Create a Python environment (see [Step 2](#step-2--set-up-python) below) |
| 3 | `pip install -r requirements.txt` |
| 4 | Copy `.env.template` to `.env` and fill in your values |
| 5 | Open `data_qa_agent.ipynb` in VS Code, pick the **sample_foundry_v2** kernel, run all cells |

**Classic notebook:**

| Step | What to do |
|------|-----------|
| 1 | Clone this repo and `cd` into the folder |
| 2 | Create a Python environment (see [Step 2](#step-2--set-up-python) below) |
| 3 | `pip install -r requirements_classic.txt` |
| 4 | Copy `.env.template` to `.env` and fill in your values |
| 5 | Open `data_qa_agent_classic.ipynb` in VS Code, pick the kernel, run all cells |

New to this? Follow the full guide below from the top.

---

## What's in this folder?

| File / Folder | Purpose |
|---------------|---------|
| `data_qa_agent.ipynb` | **New Experience** notebook (Responses API + Conversations) |
| `data_qa_agent_classic.ipynb` | **Classic** notebook (Threads / Runs / Messages) |
| `requirements.txt` | Python packages for the new experience notebook |
| `requirements_classic.txt` | Python packages for the classic notebook |
| `.env.template` | Template — copy to `.env` and fill in your values |
| `.env` | Your local secrets (git-ignored) |
| `sample_data/sample_sales.csv` | Example dataset to try first |
| `output/` | Where the agent saves generated files |

---

## Classic vs. New Experience — How File Uploads Work

This is the most important difference between the two notebooks:

### New Experience (`data_qa_agent.ipynb`)

```
Upload file → Bind file to Code Interpreter tool → Republish agent version → Create/reuse conversation → Generate response
```

- Files for Code Interpreter **must** be bound to the agent's tool configuration via `AutoCodeInterpreterToolParam(file_ids=[...])`.
- This means every time you upload a new CSV/Excel file, a **new immutable agent version** is published.
- Multi-turn context is managed via the **Conversations API** (`openai.conversations.create()`, `openai.conversations.items.create()`).
- Follow-up questions within the same file don't require republishing — they reuse the same conversation.

### Classic (`data_qa_agent_classic.ipynb`)

```
Upload file → Attach to message → Run agent on thread → Done
```

- Files are uploaded and attached to the **message** via `MessageAttachment(file_id=..., tools=CodeInterpreterTool().definitions)`.
- The agent is created **once** and **never modified** regardless of how many files are uploaded.
- Multi-turn context is managed via **Threads** (`agents_client.threads.create()`, `agents_client.messages.create()`).

### Summary

| Aspect | New Experience | Classic |
|--------|---------------|---------|
| SDK | `azure-ai-projects` 2.x | `azure-ai-agents` 1.x |
| Agent creation | `create_version()` — immutable, versioned | `create_agent()` — mutable, created once |
| File upload | Bound to agent tool config → **republishes agent** | `MessageAttachment` on message → **no agent change** |
| Multi-turn state | Conversations API | Threads |
| Response generation | `responses.create()` | `runs.create_and_process()` |
| File download | `containers.files.content.retrieve()` | `files.get_content()` |
| Status | **GA — recommended** | **Deprecated** (retiring March 31 2027) |

---

## Full Setup Guide

### Step 1 — Azure Prerequisites

You'll need these **before** touching any code:

- [ ] An **Azure subscription**
- [ ] An **Azure AI Foundry resource** with a **project** inside it
- [ ] A **deployed model** in that project (e.g. `gpt-4.1`)
- [ ] **Azure CLI** installed ([install guide](https://learn.microsoft.com/cli/azure/install-azure-cli))
- [ ] *(Optional)* **Azure Developer CLI (`azd`)** installed ([install guide](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd))

Once you have those, log in:

```powershell
# Azure CLI
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID_OR_NAME>"

# (Optional) Azure Developer CLI
azd auth login
```

#### 1a. Create a Storage Account

The Foundry new experience needs a **storage account** connected to your project so the agent can read/write files.

> **Why?** Classic Foundry had built-in storage. The new experience does not — you bring your own.

<details>
<summary><strong>Option A: Azure CLI (<code>az</code>)</strong></summary>

```powershell
# ---- Replace these with your own values ----
$SUB_ID              = az account show --query id -o tsv
$RG                  = "<YOUR_RESOURCE_GROUP>"
$LOCATION            = "<YOUR_REGION>"              # e.g. eastus2
$FOUNDRY_RESOURCE    = "<YOUR_FOUNDRY_RESOURCE_NAME>"
$PROJECT_NAME        = "<YOUR_PROJECT_NAME>"
$STORAGE_ACCT        = "<yourstorageacctname>"       # 3-24 chars, lowercase, globally unique
# ---------------------------------------------

az storage account create `
  --name $STORAGE_ACCT `
  --resource-group $RG `
  --location $LOCATION `
  --sku Standard_LRS `
  --kind StorageV2 `
  --min-tls-version TLS1_2
```

</details>

<details>
<summary><strong>Option B: Azure Developer CLI (<code>azd</code>)</strong></summary>

`azd` doesn't have a direct `storage account create` command, but you can provision one via Bicep/Terraform by initializing a template. For a quick one-off, use `az` (above) or create a minimal Bicep file:

```bicep
// infra/storage.bicep
param location string = resourceGroup().location
param storageName string

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
  }
}
```

Then deploy:

```powershell
azd provision
# or manually:
az deployment group create `
  --resource-group $RG `
  --template-file infra/storage.bicep `
  --parameters storageName=$STORAGE_ACCT
```

</details>

#### 1b. Grant the Project Permission to Use That Storage

> **Key detail:** Foundry has two managed identities — one for the *resource* and one for the *project*.
> You need the **project** identity. That's the one Code Interpreter actually uses.

<details>
<summary><strong>Option A: Azure CLI (<code>az</code>)</strong></summary>

```powershell
# Get the project's managed identity
$PROJECT_MSI = az rest --method GET `
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$FOUNDRY_RESOURCE/projects/$PROJECT_NAME`?api-version=2025-04-01-preview" `
  --query "identity.principalId" -o tsv

# Give it read/write access to the storage account
az role assignment create `
  --assignee-object-id $PROJECT_MSI `
  --assignee-principal-type ServicePrincipal `
  --role "Storage Blob Data Contributor" `
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT"
```

*Optional — verify it worked:*

```powershell
az role assignment list `
  --assignee-object-id $PROJECT_MSI `
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT" `
  --output table
```

</details>

<details>
<summary><strong>Option B: Azure Developer CLI (<code>azd</code>) + Bicep</strong></summary>

Add a role assignment to your Bicep template:

```bicep
// infra/role-assignment.bicep
param storageAccountName string
param projectPrincipalId string

var storageBlobDataContributorId = 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' existing = {
  name: storageAccountName
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storage.id, projectPrincipalId, storageBlobDataContributorId)
  scope: storage
  properties: {
    principalId: projectPrincipalId
    principalType: 'ServicePrincipal'
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', storageBlobDataContributorId)
  }
}
```

Then provision with `azd provision`, or deploy manually:

```powershell
az deployment group create `
  --resource-group $RG `
  --template-file infra/role-assignment.bicep `
  --parameters storageAccountName=$STORAGE_ACCT projectPrincipalId=$PROJECT_MSI
```

</details>

#### 1c. Connect Storage in Foundry Portal

1. Open https://ai.azure.com
2. Go to **Management Center** → **Connected resources**
3. Click **+ New connection** → **Data** → **Azure Blob Storage**
4. Select the storage account you just created
5. Set authentication to **Microsoft Entra ID**

> Allow a few minutes for permission propagation before uploading files.

---

### Step 2 — Set Up Python

Pick **one** of the two options below. Both target **Python 3.14.3** (same as `albemarle_web_app`).

#### Option A: Conda (Recommended)

**Don't have Conda?** Install Miniconda first:

```powershell
winget install -e --id Anaconda.Miniconda3
```

Then open a **new** terminal and run:

```powershell
conda create -n sample_foundry_v2 python=3.14.3 ipykernel -y
conda activate sample_foundry_v2
pip install -r requirements.txt
```

#### Option B: Python venv

**Don't have Python 3.14?** Install it first:

```powershell
winget install -e --id Python.Python.3.14
```

Then create and activate the virtual environment:

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1          # PowerShell
# or: .venv\Scripts\activate.bat      # cmd prompt
python -m pip install --upgrade pip
pip install -r requirements.txt
python -m ipykernel install --user --name sample_foundry_v2 --display-name "Python (sample_foundry_v2)"
```

---

### Step 3 — Configure Environment Variables

```powershell
Copy-Item .env.template .env -Force
```

Open `.env` and fill in both values:

| Variable | What to put |
|----------|-------------|
| `FOUNDRY_PROJECT_ENDPOINT` | `https://<resource>.services.ai.azure.com/api/projects/<project>` |
| `FOUNDRY_MODEL_DEPLOYMENT_NAME` | The model you deployed, e.g. `gpt-4.1` |

You can find both in the Foundry portal under your project's **Overview** page.

---

### Step 4 — Run the Notebook

1. Open **`data_qa_agent.ipynb`** in VS Code
2. In the top-right kernel picker, select **Python (sample_foundry_v2)**
3. Run cells from top to bottom

The cells do this in order:

| Cell(s) | What happens |
|---------|-------------|
| 1 | Installs Python packages |
| 2 | Loads your `.env` settings |
| 3 | Publishes a new agent version to Foundry |
| 4 | Connects to the OpenAI Responses API |
| 5+ | Upload a file, ask questions, download results |

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---------|-------------|-----|
| `400 invalid_payload: Not allowed when agent is specified` | Sending `tools` alongside `agent_reference` in the API call | This is already fixed in the notebook. If you see it in custom code, remove the top-level `tools` parameter. |
| File upload succeeds but analysis fails | Storage not connected, or permissions haven't propagated | Re-check [Step 1b](#1b-grant-the-project-permission-to-use-that-storage) and [Step 1c](#1c-connect-storage-in-foundry-portal). Wait a few minutes and retry. |
| `Authorization failed` or `Access denied` | Your user account doesn't have a Foundry role | Run the command below to grant yourself access. |
| No file downloaded after asking the agent to generate one | Agent didn't produce a file annotation | Rephrase your prompt to explicitly ask for a saved/exported file. |

**Grant yourself Foundry access** (if needed):

```powershell
$ME = az ad signed-in-user show --query id -o tsv

az role assignment create `
  --assignee-object-id $ME `
  --assignee-principal-type User `
  --role "Azure AI User" `
  --scope "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$FOUNDRY_RESOURCE"
```
