Cost Optimization Challenge: Managing Billing Records in Azure Serverless Architecture

We have a serverless architecture in Azure, where one of our services stores billing records in Azure Cosmos DB. The system is read-heavy, but records older than three months are rarely accessed.
Over the past few years, the database size has significantly grown, leading to increased costs. We need an efficient way to reduce costs while maintaining data availability.

Current System Constraints
1. 
Record Size: Each billing record can be as large as 300 KB.
2. 
Total Records: The database currently holds over 2 million records.
3. 
Access Latency: When an old record is requested, it should still be served, with a response time in the order of seconds.

Solution Requirements

Please propose a detailed solution to optimize costs while ensuring the following

1. 
Simplicity & Ease of Implementation – The solution should be straightforward to deploy and maintain.
2. 
No Data Loss & No Downtime – The transition should be seamless, without losing any records or requiring2 service downtime.
3. 
No Changes to API Contracts – The existing read/write APIs for billing records must remain unchanged

Bonus Points

Include an architecture diagram illustrating your proposed solution.
Provide pseudocode, commands, or scripts for implementing core logic (such as data archival, retrieval, and cost optimization st rategies).

Solution:
Absolutely — since the **current data resides in Cosmos DB**, you’ll need a **one-time migration** of all **existing records (older than 3 months or all 2M)** to **Azure Blob Storage**, and then **deprecate Cosmos DB**.

Let’s break this down with zero downtime and no API contract changes.

---

## ✅ Final Plan Summary

### 🔄 One-Time Migration (Cosmos DB → Blob Storage)

Migrate all records to Blob Storage as JSON files using a **simple Python or Azure Function app**.

### 🏃 Ongoing Storage

Once migration is complete:

* Store **all new records directly to Blob Storage (Hot tier)**
* **Auto-tier them to Cool tier** using lifecycle rules

### 📞 Read & Write API remains unchanged

Logic layer (e.g., Azure Function) handles reads/writes transparently — no client-side changes.

---

## 🔁 Migration Logic: Cosmos DB → Blob Storage

### 🔹 Setup

* Enable **Azure Blob Storage container** called `billing-records`
* (Optional) Add `billing/yyyy-mm/dd/recordId.json` structure
* Write a **batch job** (Azure Function, Azure Data Factory, or Python script)

---

### ✅ Python Migration Script (Serverless-friendly)

```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json
from datetime import datetime

# Cosmos DB config
COSMOS_ENDPOINT = "<cosmos-endpoint>"
COSMOS_KEY = "<cosmos-key>"
DATABASE_NAME = "BillingDB"
CONTAINER_NAME = "Records"

# Blob config
BLOB_CONN_STR = "<azure-storage-connection-string>"
BLOB_CONTAINER_NAME = "billing-records"

# Initialize clients
cosmos_client = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
container = cosmos_client.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)

blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
blob_container = blob_service.get_container_client(BLOB_CONTAINER_NAME)

# Loop through and migrate records
query = "SELECT * FROM Records"
for record in container.query_items(query, enable_cross_partition_query=True):
    record_id = record["id"]
    timestamp = record.get("timestamp", datetime.utcnow().isoformat())
    date = datetime.fromisoformat(timestamp)
    blob_path = f"billing/{date.year}-{date.month:02}/{date.day:02}/{record_id}.json"

    # Upload to blob
    blob_container.upload_blob(blob_path, json.dumps(record), overwrite=True)

    # Optionally delete from Cosmos after upload
    container.delete_item(item=record_id, partition_key=record['partitionKey'])

print("Migration complete.")
```

---

## 🗂️ Lifecycle Rule (Tier to Cool After 90 Days)

Create once and forget:

```bash
az storage account management-policy create \
  --account-name mystorageaccount \
  --policy '{
    "rules": [
      {
        "name": "moveToCoolAfter90Days",
        "enabled": true,
        "type": "Lifecycle",
        "definition": {
          "filters": {
            "blobTypes": ["blockBlob"],
            "prefixMatch": ["billing/"]
          },
          "actions": {
            "baseBlob": {
              "tierToCool": {
                "daysAfterModificationGreaterThan": 90
              }
            }
          }
        }
      }
    ]
  }'
```

---

## 📞 Updated Function App (API Logic)

### 🔹 Write Billing Record

```python
def write_billing_record(record):
    date = datetime.fromisoformat(record["timestamp"])
    blob_path = f"billing/{date.year}-{date.month:02}/{date.day:02}/{record['id']}.json"
    blob_container.upload_blob(blob_path, data=json.dumps(record), overwrite=True)
```

### 🔹 Read Billing Record

```python
def get_billing_record(record_id, date):
    blob_path = f"billing/{date.year}-{date.month:02}/{date.day:02}/{record_id}.json"
    try:
        blob = blob_container.get_blob_client(blob_path)
        return json.loads(blob.download_blob().readall())
    except:
        raise ValueError("Record not found")
```

---

## ✅ Final Result After Migration

| Component      | Role                               |
| -------------- | ---------------------------------- |
| Cosmos DB      | ✅ Deprecated after migration       |
| Blob Hot Tier  | Stores new records (≤90 days)      |
| Blob Cool Tier | Auto-tiered for older records      |
| API Layer      | Same endpoints, logic changed only |
| Cost Savings   | Up to 90% lower than Cosmos DB     |
