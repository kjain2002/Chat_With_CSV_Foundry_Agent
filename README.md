# Data Q&A Agent Notebook

> Upload a spreadsheet (CSV or Excel), ask questions about it in everyday language, and get answers, charts, or new files back — powered by **Microsoft Foundry Agent** - New Experience (via code).

---

## Quick-Start Checklist

If you already have Azure and Python set up, here's the short version:

| Step | What to do |
|------|-----------|
| 1 | Clone this repo and `cd` into the folder |
| 2 | Create a Python environment (see [Step 2](#step-2--set-up-python) below) |
| 3 | `pip install -r requirements.txt` |
| 4 | Copy `.env.template` to `.env` and fill in your values |
| 5 | Open `data_qa_agent.ipynb` in VS Code, pick the **sample_foundry_v2** kernel, run all cells |

New to this? Follow the full guide below from the top.

---

## What's in this folder?

| File / Folder | Purpose |
|---------------|---------|
| `data_qa_agent.ipynb` | The main notebook you'll run |
| `requirements.txt` | Python packages (pinned versions) |
| `.env.template` | Template — copy to `.env` and fill in your values |
| `.env` | Your local secrets (git-ignored) |
| `sample_data/sample_sales.csv` | Example dataset to try first |
| `output/` | Where the agent saves generated files |

---

## Full Setup Guide

### Step 1 — Azure Prerequisites

You'll need these **before** touching any code:

- [ ] An **Azure subscription**
- [ ] An **Azure AI Foundry resource** with a **project** inside it
- [ ] A **deployed model** in that project (e.g. `gpt-4.1`)
- [ ] **Azure CLI** installed ([install guide](https://learn.microsoft.com/cli/azure/install-azure-cli))

Once you have those, log in:

```powershell
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID_OR_NAME>"
```

#### 1a. Create a Storage Account

The Foundry new experience needs a **storage account** connected to your project so the agent can read/write files.

> **Why?** Classic Foundry had built-in storage. The new experience does not — you bring your own.

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

#### 1b. Grant the Project Permission to Use That Storage

> **Key detail:** Foundry has two managed identities — one for the *resource* and one for the *project*.
> You need the **project** identity. That's the one Code Interpreter actually uses.

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
