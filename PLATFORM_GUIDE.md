# Microsoft Data & AI Strategy Platform — Complete Guide

> **Repo:** [aqa100/Data-and-AI-Strategy-Platform](https://github.com/aqa100/Data-and-AI-Strategy-Platform)  
> **Based on:** Microsoft's internal data estate journey  
> **Deployment model:** GitHub Actions + Azure Bicep  
> **Environments supported:** Development · Test · Production

---

## Table of Contents

1. [What Is This Platform?](#1-what-is-this-platform)
2. [Architecture Overview](#2-architecture-overview)
3. [Azure Components](#3-azure-components)
4. [Repository Structure](#4-repository-structure)
5. [Prerequisites](#5-prerequisites)
6. [Deployment: Step-by-Step](#6-deployment-step-by-step)
7. [Feature Flags Reference](#7-feature-flags-reference)
8. [Variable Configuration](#8-variable-configuration)
9. [GitHub Secrets Reference](#9-github-secrets-reference)
10. [RBAC & Permissions](#10-rbac--permissions)
11. [Networking: Private Endpoint Setup](#11-networking-private-endpoint-setup)
12. [Post-Deployment Tasks](#12-post-deployment-tasks)
13. [Data Ingestion: Control Table Examples](#13-data-ingestion-control-table-examples)
14. [GitHub Actions Workflows](#14-github-actions-workflows)
15. [Known Issues](#15-known-issues)
16. [Contributing](#16-contributing)

---

## 1. What Is This Platform?

The **Microsoft Data & AI Strategy Platform** is a fully automated, Infrastructure-as-Code (IaC) accelerator for deploying a modern Azure data platform. It targets organizations that want to move from foundational data work to AI innovation — quickly and repeatably.

### The Three Pillars

| Pillar | Focus |
|--------|-------|
| **People** | Build data acumen, foster a data-driven culture, identify SMEs, grow a Center of Enablement |
| **Process** | Central data governance with federated elements: data products, lineage, quality, lifecycle, access control |
| **Platform** | Scalable Azure infrastructure with shared services, self-serve ingestion, federated compute, and AI integration |

### Key Capabilities

- **Feature-flag driven deployment** — turn any Azure service on/off per environment
- **Metadata-driven ingestion** — ADF/Databricks pipelines controlled via a central SQL control table
- **Multi-environment support** — dev, test, and production via the same pipeline
- **Optional private networking** — deploy entirely behind VNets with private endpoints
- **Built-in RBAC** — role assignments for managed identities and AAD groups configured automatically
- **OpenAI/ML integration** — optional Azure OpenAI, Azure ML, and Cognitive Services components

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Governance (Purview)                 │
├──────────────┬──────────────────┬───────────────────────────┤
│  Data Sources│  Shared Services │     Federated Compute      │
│  (Oracle,    │  - Metadata-     │  - Databricks              │
│   SQL, REST, │    driven ingest │  - Synapse                 │
│   M365, CSV, │  - ADF Pipelines │  - Azure ML                │
│   Files...)  │  - Key Vault     │  - Logic Apps              │
├──────────────┴──────────────────┴───────────────────────────┤
│          Data Lake (Landing → Raw → Staging → Curated)      │
├─────────────────────────────────────────────────────────────┤
│        Consumers (Power BI, Fabric, Synapse, Databricks)    │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Layers

| Layer | Container | Purpose |
|-------|-----------|---------|
| **Landing** | `landing` | Raw ingested files from source systems |
| **Raw** | `raw` | Validated, format-converted files (Parquet) |
| **Staging** | `staging` | Delta tables with deduplication and partitioning |
| **Curated** | `curated` | Business-ready, enriched data products |

---

## 3. Azure Components

| Component | Azure Service | Feature Flag |
|-----------|--------------|--------------|
| Data Lake | ADLS Gen2 | `DeployDataLake` |
| Landing Storage | Azure Blob Storage | `DeployLandingStorage` |
| Key Vault | Azure Key Vault | `DeployKeyVault` |
| Metadata DB | Azure SQL (MetadataControl) | `DeployAzureSQL` |
| Orchestration | Azure Data Factory | `DeployADF` |
| Big Data Compute | Azure Databricks | `DeployDatabricks` |
| Analytics | Azure Synapse Analytics | `DeploySynapse` |
| Governance | Microsoft Purview | `DeployPurview` |
| Workflow Automation | Logic App (Standard) | `DeployLogicApp` |
| AI/ML | Azure Machine Learning | `DeployMLWorkspace` |
| AI Cognitive | Cognitive Services | `DeployCognitiveService` |
| Fabric | Microsoft Fabric Capacity | `DeployFabricCapacity` |
| Monitoring | Log Analytics Workspace | `DeployLogAnalytics` |
| Streaming | Event Hub + Stream Analytics | `DeployEventHubNamespace` |
| Generative AI | Azure OpenAI + AI Search | `DeployOpenAIServiceAndAiSearch` |

---

## 4. Repository Structure

```
├── .github/workflows/
│   ├── deployment_orchestrator.yml     # Main entry point — triggers all envs
│   ├── deployment_environment.yml      # Per-environment deployment logic
│   ├── cicd_azuresql.yml               # Azure SQL schema CI/CD
│   ├── cd_adf.yml                      # ADF artifact deployment
│   ├── cd_synapse_workspace.yml        # Synapse artifact deployment
│   ├── cd_logicapp.yml                 # Logic App deployment
│   ├── consumer_orchestrator.yml       # Consumer workspace orchestration
│   └── ...
├── DeploymentComponents/
│   ├── bicep_templates/                # Bicep IaC for all services
│   ├── bicep_parameters/               # Per-env Bicep parameter files
│   │   ├── dev/
│   │   ├── test/
│   │   └── prod/
│   ├── variables/
│   │   ├── general_feature_flags/      # ✏️ Which services to deploy
│   │   │   ├── feature_flags_dev.json
│   │   │   ├── feature_flags_test.json
│   │   │   └── feature_flags_prod.json
│   │   ├── general_variables/          # ✏️ Resource names & locations
│   │   │   ├── variables_dev.json
│   │   │   ├── variables_test.json
│   │   │   └── variables_prod.json
│   │   ├── entra_assignments/          # ✏️ AAD group IDs for RBAC
│   │   ├── networking_setup/           # ✏️ VNet/subnet config (private mode)
│   ├── azure_sql_artifacts/            # SQL scripts & control table setup
│   ├── synapse_adf_artifacts/          # ADF/Synapse pipeline definitions
│   ├── databricks/                     # Databricks notebooks
│   ├── logicapp_standard/              # Logic App workflow definitions
│   ├── mlops_artifacts/                # Azure ML pipelines
│   ├── ingestion_patterns/             # Example ingestion patterns
│   ├── consumers/                      # Consumer workspace templates
│   ├── rbac.md                         # RBAC assignments reference
│   └── networking.md                   # Networking requirements
├── DataStrategyBacklog.xlsx            # Azure DevOps importable backlog
└── README.md
```

---

## 5. Prerequisites

### Azure

- [ ] Azure subscription with **Owner** rights (or pre-created resource groups with Owner rights)
- [ ] Azure AD group created for admin team members
- [ ] Required resource providers registered:
  - `Microsoft.EventGrid`
  - `Microsoft.Purview` *(if deploying Purview)*
  - `Microsoft.EventHub` *(if deploying Purview or Event Hub)*
  - `Microsoft.Databricks` *(if deploying Databricks)*

Register via CLI:
```bash
az provider register --namespace Microsoft.EventGrid --wait
az provider register --namespace Microsoft.Databricks --wait
az provider register --namespace Microsoft.Purview --wait
```

### GitHub

- [ ] Forked this repository into your own GitHub account
- [ ] GitHub environments created: `development`, `test`, `production`
- [ ] Secrets configured per environment (see [Section 9](#9-github-secrets-reference))
- [ ] Service Principal with federated OIDC credential (see [Section 6](#6-deployment-step-by-step))

### Tools (local setup)

```bash
az --version      # Azure CLI 2.x+
gh --version      # GitHub CLI
```

---

## 6. Deployment: Step-by-Step

### Step 1 — Create a Service Principal

```bash
az ad sp create-for-rbac \
  --name "DataStrategyPlatform-SP" \
  --role "Owner" \
  --scopes "/subscriptions/<SUBSCRIPTION_ID>"
```

Save the output `appId` — this is your `SERVICE_PRINCIPAL_CLIENT_ID`.

### Step 2 — Create Federated OIDC Credential

Replace `<APP_ID>` and `<YOUR_GITHUB_ORG/REPO>` accordingly. Create one credential **per environment**:

```bash
# For 'development' environment
az ad app federated-credential create \
  --id <APP_ID> \
  --parameters '{
    "name": "DataStrategyPlatform-development",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<YOUR_GITHUB_ORG/REPO>:environment:development",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

Repeat with `subject` pointing to `environment:test` and `environment:production` as needed.

### Step 3 — Create AAD Groups

```bash
az ad group create --display-name "Data Strategy Admins" --mail-nickname "DataStrategyAdmins"
az ad group create --display-name "Data Strategy Shared Services" --mail-nickname "DataStrategySharedServices"

# Add yourself
az ad group member add --group "<GROUP_ID>" --member-id $(az ad signed-in-user show --query id -o tsv)
```

### Step 4 — Create GitHub Environments

```bash
gh api repos/<ORG/REPO>/environments/development --method PUT
gh api repos/<ORG/REPO>/environments/test --method PUT
gh api repos/<ORG/REPO>/environments/production --method PUT
```

### Step 5 — Set GitHub Secrets

```bash
REPO="<ORG/REPO>"
ENV="development"

gh secret set TENANT_ID             --body "<TENANT_ID>"      --env $ENV --repo $REPO
gh secret set SUBSCRIPTION_ID       --body "<SUBSCRIPTION_ID>" --env $ENV --repo $REPO
gh secret set SERVICE_PRINCIPAL_CLIENT_ID --body "<SP_CLIENT_ID>" --env $ENV --repo $REPO

# Optional: only if deploying VMs with Bastion
gh secret set VM_USERNAME --body "<username>" --env $ENV --repo $REPO
gh secret set VM_PASSWORD --body "<password>" --env $ENV --repo $REPO

# Optional: only if using a separate DNS zone subscription
gh secret set DNS_ZONE_SUBSCRIPTION_ID --body "<dns_sub_id>" --env $ENV --repo $REPO
```

### Step 6 — Configure Feature Flags

Edit `DeploymentComponents/variables/general_feature_flags/feature_flags_dev.json`:

```json
{
  "DeployDevEnvironment": true,
  "DeployResourcesWithPublicAccess": true,
  "DeployWithCustomNetworking": false,
  "DeployDataLake": true,
  "DeployADF": true,
  "DeployDatabricks": true,
  "DeployPurview": true,
  ...
}
```

> See [Section 7](#7-feature-flags-reference) for the full flag table.

### Step 7 — Configure Resource Variables

Edit `DeploymentComponents/variables/general_variables/variables_dev.json`:

```json
{
  "azureResourceLocation": "eastus2",
  "PrimaryRgName": "rg-dataplatform-dev",
  "dataFactoryName": "adf-myplatform-dev",
  "keyVaultName": "kv-myplatform-dev",
  "dataLakeName": "myplatformdevdl",
  "landingStorageName": "myplatformdevlnd",
  ...
}
```

> See [Section 8](#8-variable-configuration) for naming constraints.

### Step 8 — Configure Entra Assignments

Edit `DeploymentComponents/variables/entra_assignments/variables_dev.json`:

```json
{
  "Entra_Group_Admin": {
    "Group_ID": "<AAD_GROUP_OBJECT_ID>",
    "Group_Name": "Data Strategy Admins"
  },
  "Entra_Group_Shared_Service": {
    "Group_ID": "<AAD_GROUP_OBJECT_ID>",
    "Group_Name": "Data Strategy Shared Services"
  }
}
```

### Step 9 — Commit and Trigger

```bash
git add DeploymentComponents/variables/
git commit -m "Configure dev environment for deployment"
git push origin main

# Trigger the deployment
gh workflow run deployment_orchestrator.yml --repo <ORG/REPO> --ref main

# Watch progress
gh run list --repo <ORG/REPO> --workflow deployment_orchestrator.yml
gh run watch <RUN_ID> --repo <ORG/REPO>
```

---

## 7. Feature Flags Reference

File: `DeploymentComponents/variables/general_feature_flags/feature_flags_<env>.json`

| # | Flag | Description |
|---|------|-------------|
| 1 | `DeployDevEnvironment` | Deploy the dev environment |
| 2 | `DeployTestEnvironment` | Deploy the test environment |
| 3 | `DeployProdEnvironment` | Deploy the production environment |
| 4 | `DeployResourcesWithPublicAccess` | Public network access (`0.0.0.0/0`). Credentials still required |
| 5 | `DeployWithCustomNetworking` | Deploy behind VNet with private endpoints |
| 6 | `Assign_RBAC_On_Deployment` | Auto-assign RBAC roles during deployment |
| 7 | `ServicePrincipalHasOwnerRBACAtSubscription` | Required for Purview ingestion private endpoints |
| 8 | `DeployFabricCapacity` | Deploy Microsoft Fabric Capacity |
| 9 | `DeployDataLake` | Deploy ADLS Gen2 Data Lake |
| 10 | `DeployLandingStorage` | Deploy Landing Blob Storage |
| 11 | `DeployKeyVault` | Deploy Azure Key Vault |
| 12 | `DeployAzureSQL` | Deploy Azure SQL Server + MetadataControl DB |
| 13 | `DeployAzureSQLArtifacts` | Deploy SQL tables and stored procedures |
| 14 | `UseDatabricksForIngestionNotebooks` | Use Databricks (true) or Synapse (false) notebooks |
| 15 | `DeployADF` | Deploy Azure Data Factory |
| 16 | `DeployADFArtifacts` | Deploy ADF pipelines and linked services |
| 17 | `DeploySynapse` | Deploy Azure Synapse Analytics |
| 18 | `DeploySynapseSparkPools` | Deploy Synapse Spark pools |
| 19 | `DeploySynapseSqlPools` | Deploy Synapse Serverless SQL pools |
| 20 | `DeploySynapseArtifacts` | Deploy Synapse pipelines/linked services |
| 21 | `DeploySynapseWithDataExfiltrationProtection` | Enable exfiltration protection (disables AML orchestration) |
| 22 | `DeployPurview` | Deploy Microsoft Purview |
| 23 | `DeployDatabricks` | Deploy Azure Databricks |
| 24 | `DeployDatabricksCompute` | Deploy Databricks all-purpose compute cluster |
| 25 | `DatabricksUsesUnityCatalog` | Enable Unity Catalog on Databricks workspace |
| 26 | `DeployLogicApp` | Deploy Logic App (Standard) |
| 27 | `DeployLogicAppArtifacts` | Deploy Logic App workflow definitions |
| 28 | `DeployLogAnalytics` | Deploy Log Analytics Workspace |
| 29 | `DeployMLWorkspace` | Deploy Azure Machine Learning Workspace |
| 30 | `DeployMLCompute` | Deploy ML compute clusters |
| 31 | `DeployCognitiveService` | Deploy Azure Cognitive Services |
| 32 | `DeployEventHubNamespace` | Deploy Azure Event Hub Namespace |
| 33 | `DeployStreamAnalytics` | Deploy Stream Analytics job |
| 34 | `DeployOpenAIServiceAndAiSearch` | Deploy Azure OpenAI + AI Search |
| 35 | `DeployOpenAIDemoApp` | Deploy the OpenAI demo web app |

---

## 8. Variable Configuration

File: `DeploymentComponents/variables/general_variables/variables_<env>.json`

### Naming Constraints

| Constraint | Applies To |
|-----------|-----------|
| Letters and numbers only, 3–24 chars | `dataLakeName`, `landingStorageName`, `logicAppStorageName`, `mlStorageName` |
| 3–24 chars (hyphens allowed) | `keyVaultName` |
| Letters and numbers only | `mlContainerRegistryName`, `fabricCapacityName` |
| Globally unique | All storage accounts, SQL servers, Key Vaults, ADF, Purview, Databricks |
| Always `MetadataControl` | `azureSQLServerDBName` |

> ⚠️ If Key Vault or Container Registry are deleted and redeployed, **change the name** — soft delete prevents reuse.

### Example Configuration

```json
{
  "azureResourceLocation": "eastus2",
  "PrimaryRgName": "rg-dataplatform-dev",
  "LogicAppRgName": "rg-logicapp-dev",
  "MlRgName": "rg-ml-dev",

  "fabricCapacityName": "myorgdevfabric",
  "fabricSKU": "F2",

  "dataFactoryName": "adf-myorg-dev-001",
  "azureSQLServerName": "sql-myorg-dev-001",
  "azureSQLServerDBName": "MetadataControl",
  "keyVaultName": "kv-myorg-dev-001",
  "dataLakeName": "myorgdevdl001",
  "landingStorageName": "myorgdevlnd001",

  "databricksWorkspaceName": "dbw-myorg-dev",
  "purviewName": "pv-myorg-dev-001",
  "logAnalyticsName": "log-myorg-dev",

  "logicAppName": "la-myorg-dev",
  "logicAppServicePlanName": "laplan-myorg-dev",
  "logicAppStorageName": "myorgdevla001",
  "logicAppInsightsName": "lai-myorg-dev",

  "mlWorkspaceName": "ml-myorg-dev",
  "mlStorageName": "myorgdevml001",
  "mlContainerRegistryName": "myorgdevacr001",
  "mlWorkspaceKeyVaultName": "kv-myorg-ml-001"
}
```

---

## 9. GitHub Secrets Reference

Secrets are set **per environment** (development / test / production).

| Secret | Required | Description |
|--------|----------|-------------|
| `TENANT_ID` | ✅ Always | Azure Active Directory Tenant ID |
| `SUBSCRIPTION_ID` | ✅ Always | Azure Subscription ID |
| `SERVICE_PRINCIPAL_CLIENT_ID` | ✅ Always | App ID of the Service Principal |
| `DNS_ZONE_SUBSCRIPTION_ID` | If private endpoints | Subscription hosting the Private DNS Zones |
| `VM_USERNAME` | If deploying VMs/Bastion | VM administrator username |
| `VM_PASSWORD` | If deploying VMs/Bastion | VM administrator password |

---

## 10. RBAC & Permissions

The following RBAC assignments are made automatically when `Assign_RBAC_On_Deployment: true`.

### AAD Admin Group
- Storage Blob Data Contributor on Data Lake, Landing, Logic App, and ML storage
- Key Vault Secrets Officer
- Azure SQL Database Administrator
- Purview Root Collection Administrator
- Synapse Administrator
- Cognitive Services User

### ADF Managed Identity
- Storage Blob Data Contributor (Data Lake, Landing)
- Key Vault Secrets User
- SQL read/execute on MetadataControl
- Purview Data Curator
- Contributor on Databricks Workspace

### Synapse Managed Identity
- Storage Blob Data Contributor (Data Lake, Landing)
- Key Vault Secrets User
- SQL read/execute on MetadataControl
- Purview Data Curator
- Contributor on Azure ML Workspace

### Logic App Managed Identity
- Synapse Contributor + Credential User
- Contributor on ADF
- SQL read/execute on MetadataControl

### Azure ML Managed Identity
- Storage Blob Data Contributor on Data Lake
- AcrPull on ML Container Registry
- Key Vault Administrator

### Additional Optional Roles (`entra_assignments` variables)
- `Assign_RBAC_for_Governance` — Governance group on storage
- `Assign_RBAC_for_Publishers` — Data Publishers with path-level write access
- `Assign_RBAC_for_Producers` — Data Producers with path-level write access
- `Assign_RBAC_for_Consumers` — Data Consumers with read access
- `Assign_RBAC_for_CICD_Service_Principal` — CI/CD SP assignments

---

## 11. Networking: Private Endpoint Setup

Enable by setting `DeployWithCustomNetworking: true` in feature flags.

### Minimum IP Requirements

| Subnet | Resource | Min IPs |
|--------|----------|---------|
| Private Endpoint | Synapse | 3 |
| Private Endpoint | Purview | 4 |
| Private Endpoint | Landing/Data Lake Storage | 5 |
| Private Endpoint | Key Vault | 1 |
| Private Endpoint | SQL Server | 1 |
| Private Endpoint | ML Workspace + Storage | 6 |
| Private Endpoint | ADF + Logic App Storage | 6 |
| **Data Subnet total** | | **31 IPs** (/26 recommended) |
| Logic App Subnet | Logic App | 32 IPs (/27 minimum) |

### Required Private DNS Zones

| Resource | DNS Zone |
|----------|----------|
| Azure SQL | `privatelink.database.windows.net` |
| Storage (Blob, DFS, Table, Queue, File) | `privatelink.blob/dfs/table/queue/file.core.windows.net` |
| Key Vault | `privatelink.vaultcore.azure.net` |
| ADF | `privatelink.datafactory.azure.net` |
| Purview | `privatelink.purview.azure.com` + `privatelink.purviewstudio.azure.com` |
| Synapse | `privatelink.sql/dev/azuresynapse.net` |
| Databricks | `privatelink.azuredatabricks.net` |
| Event Hub | `privatelink.servicebus.windows.net` |
| Cognitive Services | `privatelink.cognitiveservices.azure.com` |
| Azure ML | `privatelink.api.azureml.ms` + `privatelink.notebooks.azure.net` |
| Container Registry | `privatelink.azurecr.io` |

> ⚠️ Do **not** deploy new DNS zones if integrating with an existing VNet — use the existing zones.

---

## 12. Post-Deployment Tasks

### Azure SQL — Run on every environment

Connect using AAD authentication (SQL Auth is disabled):

```sql
EXEC [dbo].[AddManagedIdentitiesAsUsers]
```

### Synapse — Run in Synapse Serverless DB `StoredProcDB`

```sql
EXEC [dbo].[AddManagedIdentitiesAsUsers]
```

If deploying Logic App:
```sql
-- Run pre-created script in Synapse portal:
RunForLogicApp
```

### Purview — Manual steps

1. Add ADF and Synapse managed identities as **Data Curators** in the Root Collection
2. When Lake DBs are created, grant Purview scan access:

```sql
CREATE LOGIN [PurviewAccountName] FROM EXTERNAL PROVIDER;
CREATE USER  [PurviewAccountName] FOR LOGIN [PurviewAccountName];
ALTER ROLE db_datareader ADD MEMBER [PurviewAccountName];
```

3. *(Private networking only)* Approve pending Private Endpoint connections for Purview's managed Storage Account and Event Hub in the Azure Portal.
4. *(Private networking only)* Set up a [Managed VNET Integration Runtime](https://learn.microsoft.com/en-us/azure/purview/catalog-managed-vnet) and a [Self-Hosted Integration Runtime](https://learn.microsoft.com/en-us/azure/purview/catalog-private-link-end-to-end).

---

## 13. Data Ingestion: Control Table Examples

All ingestion is driven by records in `dbo.ControlTable` in the `MetadataControl` Azure SQL database.

### Pipeline Flow

```
Source → [PL_1_Source_to_Landing_Step1] → Landing Container
Landing → [PL_2_Process_Landed_Files_Step2] → Raw Container
Raw     → [PL_3_MoveToStaging_Step2]        → Staging Container (Delta)
```

Each dataset needs **one control table record per pipeline stage**.

---

### Source to Landing: Oracle (DateTime watermark)

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "schema": "dbo", "table": "SalesOrders",
     "query": "SELECT * FROM dbo.SalesOrders WHERE ModifiedDate > TO_TIMESTAMP(''WATERMARKVALUE'', ''YYYY-MM-DD HH24:MI:SS.FF'')" }',
  '{ "keyVaultSecretName": "oracle-conn-secret" }',
  '{ "watermark_column": "ModifiedDate", "watermark_column_data_type": "Datetime", "partitioningOption": "None" }',
  '{ "fileName": "SalesOrders_YYYYMMDDHHMMSS.parquet", "folderPath": "finance/oracle/", "container": "landing" }',
  '', '', '',
  'PL_1_Source_to_Landing_Step1',
  'TR_SalesOrders_Daily',
  '{ "ingestionPattern": "Oracle" }',
  0, 1, '{}', 1
)
```

### Source to Landing: M365

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "tableName": "TeamsActivity", "scopeFilter": "", "filterOnDate": true }',
  '',
  '{ "dateFilterColumn": "ReportDate", "startDate": "2024-01-01", "endDate": "2024-12-31",
     "watermark_column_data_type": "DateTime",
     "outputColumns": [{"name": "UserId"}, {"name": "ReportDate"}, {"name": "MessageCount"}] }',
  '{ "fileName": null, "folderPath": "hr/m365/teamsactivity/", "container": "landing" }',
  '', '', '',
  'PL_1_Source_to_Landing_Step1',
  'TR_M365_Weekly',
  '{ "ingestionPattern": "M365" }',
  0, 1, '{}', 1
)
```

---

### Landing to Raw: CSV (Incremental)

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.csv", "folderPath": "finance/sap/%", "container": "landing" }',
  '',
  '{ "fileType": "delimitedText", "delimiter": ",", "compression": "None" }',
  '{ "fileName": null, "folderPath": "finance/sap/orders/", "container": "raw" }',
  '', '', '',
  'PL_2_Process_Landed_Files_Step2',
  'TR_FileCreated_Landing_EventLog',
  '{ "dataLoadingBehavior": "Copy_to_Raw", "loadType": "incremental" }',
  0, 1, '{}', 1
)
```

### Landing to Raw: Parquet (Full Load)

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.parquet", "folderPath": "finance/sap/%", "container": "landing" }',
  '',
  '{ "fileType": "parquet", "compression": "" }',
  '{ "fileName": null, "folderPath": "finance/sap/orders/", "container": "raw" }',
  '', '', '',
  'PL_2_Process_Landed_Files_Step2',
  'TR_FileCreated_Landing_EventLog',
  '{ "dataLoadingBehavior": "Copy_to_Raw", "loadType": "full" }',
  0, 1, '{}', 1
)
```

### Landing to Raw: Excel (Multiple Sheets)

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.xlsx", "folderPath": "%/%", "container": "landing" }',
  '',
  '[{ "SheetName": "Sheet1", "HeaderRow": "1" }, { "SheetName": "Sheet2", "HeaderRow": "1" }]',
  '{ "fileName": null, "folderPath": null, "container": "landing" }',
  '', '', '',
  'PL_2_Process_Landed_Files_Step2',
  'TR_FileCreated_Landing_EventLog',
  '{ "dataLoadingBehavior": "Extract_Excel_Sheets" }',
  0, 1, '{}', 1
)
```

### Landing to Raw: PDF/Image (Form Recognizer)

```sql
-- Supported models: prebuilt-invoice, prebuilt-receipt, prebuilt-tax.us.w2, prebuilt-idDocument, prebuilt-businessCard
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.PDF", "folderPath": "%/%", "container": "landing" }',
  '',
  '{ "model": "prebuilt-invoice" }',
  '{ "fileName": null, "folderPath": null, "container": "landing" }',
  '', '', '',
  'PL_2_Process_Landed_Files_Step2',
  'TR_FileCreated_Landing_EventLog',
  '{ "dataLoadingBehavior": "Form_Recognizer_Extraction" }',
  0, 1, '{}', 1
)
```

---

### Raw to Staging: Parquet → Delta (with partitioning)

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.parquet", "folderPath": "finance/sap/%", "container": "raw" }',
  '',
  '{ "primary_key_cols": "[''OrderId'']",
     "partition_cols": "[''CalcYear'', ''CalcMonth'']",
     "date_partition_column": "OrderDate",
     "file_type": "" }',
  '{ "fileName": null, "folderPath": "finance/sap/orders/", "container": "staging" }',
  '', '', '',
  'PL_3_MoveToStaging_Step2',
  'TR_FileCreated_Raw_EventLog',
  '{ "dataLoadingBehavior": "Copy_to_Staging" }',
  0, 1, '{}', 1
)
```

### Raw to Staging: JSON → Delta

```sql
INSERT INTO [dbo].[ControlTable] VALUES (
  '{ "fileName": "%.json", "folderPath": "hr/m365/%", "container": "raw" }',
  '',
  '{ "primary_key_cols": "[''UserId'', ''ReportDate'']",
     "partition_cols": "[''CalcYear'', ''CalcMonth'']",
     "date_partition_column": "ReportDate",
     "file_type": "json" }',
  '{ "fileName": null, "folderPath": "hr/m365/teamsactivity/", "container": "staging" }',
  '', '', '',
  'PL_3_MoveToStaging_Step2',
  'TR_FileCreated_Raw_EventLog',
  '{ "dataLoadingBehavior": "Copy_to_Staging" }',
  0, 1, '{}', 1
)
```

---

## 14. GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `deployment_orchestrator.yml` | Manual (`workflow_dispatch`) | **Main entry point** — detects which environments to deploy and fans out |
| `deployment_environment.yml` | Called by orchestrator | Deploys all Bicep infrastructure for one environment |
| `cicd_azuresql.yml` | Push / manual | CI/CD for Azure SQL schema (tables, stored procs) |
| `cd_adf.yml` | Push / manual | Deploy ADF pipeline artifacts |
| `cd_synapse_workspace.yml` | Push / manual | Deploy Synapse workspace artifacts |
| `cd_logicapp.yml` | Push / manual | Deploy Logic App workflow definitions |
| `consumer_orchestrator.yml` | Manual | Orchestrate consumer workspace creation |
| `consumer_creation.yml` | Called by consumer orchestrator | Create individual consumer workspaces |
| `consumer_powerbi_deployment.yml` | Manual | Deploy Power BI reports |
| `open_ai_resources.yml` | Manual | Deploy OpenAI + AI Search resources |
| `machine_learning_resources.yml` | Manual | Deploy ML workspace and compute |
| `logic_app_resources.yml` | Manual | Deploy Logic App and associated resources |

### Triggering the Main Deployment

```bash
gh workflow run deployment_orchestrator.yml \
  --repo <ORG/REPO> \
  --ref main

# Monitor
gh run watch <RUN_ID> --repo <ORG/REPO>
```

> ⚠️ Always use **"Run workflow"** from the Actions tab or `gh workflow run`. Do NOT use the "Re-run" button — it will not pick up updated variable files.

---

## 15. Known Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Power App for Acquisition Services has limited functionality | Open | Only basic metadata collection; manual config steps required |
| Cannot add multiple file names/sources to Acquisition Service App | Open | No resolution currently due to resource constraints |
| Acquisition Services App needs to be fully metadata-driven | Open | Drop-down values should be table-driven |
| Unity Catalog requires an existing metastore | — | Disable `DatabricksUsesUnityCatalog` if no metastore exists in your tenant |
| Key Vault / Container Registry soft delete | — | Change resource name if redeploying after deletion |
| Purview requires Subscription Owner for ingestion private endpoints | — | Set `ServicePrincipalHasOwnerRBACAtSubscription: true` if SP has Owner rights |

---

## 16. Contributing

This project follows the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-improvement`
3. Commit with clear messages
4. Open a Pull Request — a CLA bot will guide you through signing the Contributor License Agreement

For questions: [opencode@microsoft.com](mailto:opencode@microsoft.com)

---

*Generated for deployment on Azure subscription `QC-SUB-SANDBOX-DM` · Environment: `development` · Region: `eastus2`*
