Azure Billing Archival (Serverless Hot/Cold)
 Keep the latest 90 days of billing data in Cosmos DB (hot) and auto-archive older records to Azure Data
 Lake Storage Gen2 (cold). If your API requests an older record, an HTTP Function reads it from the archive
 (and can optionally rehydrate it back into Cosmos).
 ✨ What you get
 • 
• 
Hot storage: Cosmos DB container (
 records ) with 90‑day TTL
 Cold storage: ADLS Gen2 under 
• 

• 
archive/YYYYMM/DD/customerId=<id>/records.jsonl
 Change Feed archiver: Function writes old docs to ADLS
 HTTP Reader: Function to fetch archived records by 
id
 (Optional): Timer backfill job and analytics later
 ✨ Architecture
 Client API → Cosmos (hot)
 ↘ on miss → HTTP Function (archive_reader) → ADLS (cold)
 Cosmos → Change Feed → Archive Writer Function → ADLS
 • 
No API contract change: your app checks Cosmos first; if not found, call the reader.
 📁 Repository Layout
 azure-billing-archival/
 ├─ infra/
 │  └─ terraform/
 │     ├─ providers.tf
 │     ├─ variables.tf
 │     ├─ main.tf
 │     └─ outputs.tf
 └─ functions/
   ├─ shared/
   │  └─ storage.py
   ├─ archive_writer/
   │  ├─ __init__.py
   │  ├─ function.json
   │  └─ utils.py
 1
   ├─ archive_reader/
   │  ├─ __init__.py
   │  └─ function.json
   ├─ host.json
   └─ requirements.txt
 🔍 File-by-file walkthrough
 infra/terraform/providers.tf
 • 
Pins Terraform + AzureRM providers, and enables 
Azure deployment.
 infra/terraform/variables.tf
 • 
Inputs like 
prefix , 
features {} for AzureRM. Required for any
 location , and 
cosmos_ttl_seconds (90 days) for easy customization.
 infra/terraform/main.tf

• 
• 
Resource Group
 Storage Account (ADLS Gen2 enabled) + private container 
Cosmos DB Account + DB + Container
 Partition key: 
archive
 /pk
 default_ttl = 90 days → auto-expire older items
 Application Insights for logs
 Function App (Linux, Consumption plan) with Managed Identity
 App settings (names of Cosmos DB, Storage, container)
 Role assignments for Function to read/write blobs and Cosmos data
 infra/terraform/outputs.tf
 • 
Outputs handy names: 
function_app_name , 
storage_account used later by CLI.
 functions/shared/storage.py
 • 
resource_group , 
Creates an authenticated BlobServiceClient using Managed Identity
 • 
cosmos_account , 
(DefaultAzureCredential).
 ``: builds deterministic path 
archive/YYYYMM/DD/customerId=<id>/records.jsonl using 
• 
• 
createdAt .
 ``: appends one JSON line to that file. (Simple demo approach; in prod prefer Append Blobs or
 Parquet writer with buffered batches.)
 ``: scans blobs under a prefix, streams 
.jsonl , returns the object with matching 
id .
 2
functions/archive_writer/__init__.py
 • 
• 
• 
Cosmos DB Trigger receives change feed batches.
 For each document, if older than retention, writes it to ADLS with 
append_jsonl .
 Deletes are not forced: rely on Cosmos TTL to expire old hot data cleanly.
 functions/archive_writer/utils.py
 • 
``: computes age from 
createdAt ; returns 
True if ≥ 90 days.
 functions/archive_writer/function.json
 • 
• 
Binds the function to Cosmos DB Change Feed using 
Uses DB/container names from 
COSMOS_CONN app setting.
 %COSMOS_DBNAME% and 
%COSMOS_CONTAINER% .
 functions/archive_reader/__init__.py
 • 
• 
• 
• 
• 
• 
HTTP endpoint: 
GET /api/archive-reader?id=<recordId> (or POST body).
 Uses 
scan_for_record to find a row by 
id under prefix 
Returns 
archive/ .
 404 if not found; otherwise returns the JSON doc.
 You can extend this to:
 infer tighter prefixes using date/customer hints,
 rehydrate the doc back to Cosmos with a short TTL after a cold hit.
 functions/archive_reader/function.json
 • 
HTTP trigger (auth level Function by default). You’ll pass a key when calling.
 functions/host.json
 • 
Enables App Insights logging with sampling.
 functions/requirements.txt
 • 
Python dependencies for Azure Functions, Identity, Storage, and Cosmos SDKs.
 🚀 Deploy
 0) Prereqs
 • 
• 
• 
Terraform installed
 Azure CLI installed and logged in (
 az login )
 Python available if you plan to run seed scripts locally
 3
1) Provision Infra
 cd infra/terraform
 terraform init
 terraform apply-auto-approve
 2) Capture outputs (PowerShell)
 $rg
 = terraform output-raw resource_group
 $func = terraform output-raw function_app_name
 $cos = terraform output-raw cosmos_account
 $st
 = terraform output-raw storage_account
 3) App settings for Functions
 $cosConn = az cosmosdb keys list--type connection-strings-g $rg-n $cos--
 query "connectionStrings[0].connectionString"-o tsv
 $cosKey = az cosmosdb keys list--type keys-g $rg-n $cos--query
 "primaryMasterKey"-o tsv
 4) Deploy Functions (zip deploy)
 az functionapp config appsettings set -g $rg-n $func--settings `
 "COSMOS_CONN=$cosConn" `
 "COSMOS_KEY=$cosKey" `
 "COSMOS_ACCOUNT=$cos" `
 "COSMOS_DBNAME=billingdb" `
 "COSMOS_CONTAINER=records" `
 "ARCHIVE_STORAGE_ACCT=$st" `
 "ARCHIVE_CONTAINER=archive" `
 "RETENTION_DAYS=90" `
 "SCM_DO_BUILD_DURING_DEPLOYMENT=true"
 cd ..\..
 if (Test-Path functions.zip) { Remove-Item functions.zip }
 Compress-Archive-Path functions\*-DestinationPath functions.zip
 az functionapp deployment source config-zip-g $rg-n $func--src functions.zip
 Wait \~20–40s for warm-up and package restore.
 4
✅ Test the flow
 A) Insert a test document (>90 days old)
 @'
 from azure.cosmos import CosmosClient
 import os
 acct   = os.environ["COSMOS_ACCOUNT"]
 key    = os.environ["COSMOS_KEY"]
 db     = os.environ.get("COSMOS_DBNAME","billingdb")
 cname  = os.environ.get("COSMOS_CONTAINER","records")
 client = CosmosClient(f"https://{acct}.documents.azure.com:443/", 
credential=key)
 cont = client.get_database_client(db).get_container_client(cname)
 doc = {
  "id": "rec-1001",
  "customerId": "C001",
  "billingMonth": "2023-12",
  "amount": 120.55,
  "createdAt": "2023-12-15T10:03:11Z",
  "pk": "C001|202312"
 }
 cont.upsert_item(doc)
 print("Inserted:", doc["id"])
 '@ | Out-File-Encoding UTF8 seed_cosmos.py
 $env:COSMOS_ACCOUNT = $cos
 $env:COSMOS_KEY
 = $cosKey
 $env:COSMOS_DBNAME = "billingdb"
 $env:COSMOS_CONTAINER = "records"
 python .\seed_cosmos.py
 B) Call the archive reader
 Change Feed will write it to ADLS (because it’s older than 90 days). Give it \~30–60s the first
 time.
 $fnKey = az functionapp function keys list-g $rg-n $func--function-name
 archive_reader--query "keys[0].value"-o tsv
 $readerUrl = "https://$func.azurewebsites.net/api/archive-reader?
 id=rec-1001&code=$fnKey"
 curl $readerUrl
 Result: JSON for 
rec-1001 .
 5
C) (Optional) See blob path
 az storage blob list--account-name $st--container-name archive--prefix
 "archive/"--query "[].name"-o tsv
 🧪 Local dev
 • 
• 
• 
Install local deps: 
pip install -r functions/requirements.txt
 Run locally: 
func start --script-root functions
 Configure local.settings.json (do not commit keys) if needed.
 🔒 Security & hardening
 • 
• 
• 
• 
Use Private Endpoints, Storage firewall, and CMK (customer-managed keys) if required.
 Prefer Append Blobs or Parquet with buffered writes for large volumes.
 Add integrity checks (counts + checksums) and dashboard alerts in App Insights.
 Add a tiny 
{id → blobPath} metadata index if lookups must avoid scanning.
 🧰 Troubleshooting
 • 
• 
``** says empty directory** → run it inside 
infra/terraform where 
.tf files exist.
 Zip deploy works but HTTP 404 → wait 30–60s; make sure the change feed processed the insert;
 confirm blob exists.
 • 
• 
Function import errors → Ensure 
SCM_DO_BUILD_DURING_DEPLOYMENT=true and that 
requirements.txt is present at 
functions/ root.
 Auth issues to Storage → Function App must have system-assigned identity + role 
Data Contributor on the Storage Account (Terraform adds this).
 Storage Blob 
�
� Next features (optional)
 • 
• 
• 
• 
Rehydration: on cold read, write doc back to Cosmos with TTL=24h.
 Backfill Timer: migrate historic data nightly with checksums.
 Parquet format: switch JSONL to Parquet (PyArrow) for 5–10× smaller storage & faster analytics.
 Synapse Serverless: query ADLS directly with pay-per-TB SQL.
 License
 MIT (or your preferred license).
 6
