# Inventory Sync — Oracle WMS → SFCC
**MuleSoft Senior Consultant Exercise | Retailer Digital Transformation**

---

## Business Problem

The retailer runs ~750 stores across the country. Every night, stores update inventory via POS → Oracle WMS by 1 AM. **SFCC (Salesforce Commerce Cloud)** needs that data before the e-commerce day starts, to power:

- **ROPIS** – Reserve Online, Pick Up In Store
- **BOPIS** – Buy Online, Pick Up In Store
- **SFS** – Ship From Store

A full inventory CSV (~200K products × 750 stores = up to 150M rows, potentially **1 GB**) is generated from Oracle WMS at 1 AM and dropped to an SFTP folder. SFCC expects one separate CSV per store, named `inventory_<storeId>_<DD-MM-YYYY>.csv`.

---

## Solution Architecture

```
[Oracle WMS]
     │ Nightly 1 AM — generates full CSV
     ▼
[SFTP / Local Folder]  ◄─── Source
     │
     │  4 AM Cron Trigger (Mule Scheduler)
     ▼
┌──────────────────────────────────────────────────────┐
│          inventory-sync-scheduler-flow               │
│  1. Check file exists?                               │
│     • NO  → Email alert → backend-support@retailer   │
│     • YES → Start Batch Job (streaming)              │
└─────────────────────┬────────────────────────────────┘
                      │ Streams CSV (never loads fully in memory)
                      ▼
┌──────────────────────────────────────────────────────┐
│       Mule Batch Job: inventory-split-batch-job      │
│  ┌──────────────────────────────────────────────┐    │
│  │ Step 1: validate-record-step                 │    │
│  │   • All fields mandatory                     │    │
│  │   • Quantity = whole number                  │    │
│  │   • Type = Standard | Backorder              │    │
│  │   • Tags each record: _isValid true/false    │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ Step 2: write-store-file-step (valid only)   │    │
│  │   • APPEND to inventory_<storeId>_date.csv   │    │
│  │   • One file per unique StoreId              │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ Step 3: collect-invalid-step (invalid only)  │    │
│  │   • Logs invalid records                     │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ on-complete                                  │    │
│  │   • Log batch summary stats                  │    │
│  │   • Email invalid records if any             │    │
│  │   • Archive source file                      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
                      │
                      ▼
[SFTP Target / Local Output Folder]
     inventory_WHBKS_29-03-2026.csv
     inventory_FBVTS_29-03-2026.csv
     inventory_<storeId>_<date>.csv  × 750
```

---

## Key Design Decisions

### 1. Mule Batch Framework — for 1 GB on 0.2 vCore
The requirement of processing a ~1 GB CSV on just **0.2 vCore** is the most challenging NFR. The solution uses **Mule Batch Job** because:

- Batch processes records in **streaming chunks** (blockSize = 500), never loading the full file into heap.
- Records are processed in **parallel across blocks** automatically by the Batch framework.
- Batch provides **built-in failure tracking** per record; `maxFailedRecords=-1` ensures the job never stops early on bad records.
- The `file:read` uses `streaming=true` MIME type so the CSV is parsed as a lazy iterator — no memory spike.

### 2. APPEND Mode File Write (Store Splitting)
Instead of grouping all records per store in memory first (which would OOM on large files), each valid record is immediately **appended** to its store-specific file. This is memory-constant regardless of file size.

### 3. Validation in DataWeave (Step 1)
All three validation rules are implemented in a single DataWeave transform, producing `_isValid` and `_validationErrors` fields. Downstream batch steps use `acceptExpression` to route valid vs. invalid records — no if/else branching in Java or Groovy.

### 4. Global + Local Error Handling
- **Local** `try/error-handler` blocks wrap every major flow section.
- **Global** `error-handler` catches typed errors not handled locally: `SFTP:CONNECTIVITY`, `FILE:ACCESS_DENIED`, `EMAIL:SEND`, `MULE:RETRY_EXHAUSTED`, etc.
- **Notification flows** are reusable sub-flows — called from anywhere in the app without code duplication.
- Email failures are swallowed with `on-error-continue` to prevent an alert loop.

### 5. Secure Configuration
All secrets (SFTP passwords, SMTP password) use `${}` property placeholders resolved from:
- Local: `config.yaml` (never commit real passwords)
- Production: **Anypoint Secrets Manager** / CloudHub Secure Properties

---

## Non-Functional Requirements Coverage

| NFR | Implementation |
|-----|----------------|
| **Throughput — 1 GB / 30 min / 0.2 vCore** | Streaming CSV read + Batch blockSize=500; records never fully loaded into heap |
| **Security** | SFTP over SSH, SMTP/TLS, passwords in Secrets Manager |
| **Reliability** | SFTP reconnect (3×, 5s), `maxFailedRecords=-1` (never stops early), source file archived after success |
| **Scalability / Seasonal Peaks** | Batch auto-parallelism; stateless design; deploy more replicas on CloudHub during peaks |
| **Availability** | E-commerce SFCC is decoupled — sync failure does not take down SFCC; last-known-good files remain |
| **Observability** | Structured logs with correlationId at every step; email alerts for ops team |
| **Error Recovery** | Manual reprocessing: drop corrected file to source folder; next scheduler run will pick it up |
| **Transaction Management** | Batch provides built-in record-level failure tracking; source file only archived after successful batch |

---

## Project Structure

```
inventory-sync/
├── pom.xml
└── src/
    ├── main/
    │   ├── mule/
    │   │   ├── global-config.xml          # SFTP, File, Email connector configs
    │   │   ├── inventory-sync-main.xml    # Scheduler, batch job, file flows
    │   │   ├── notification-flows.xml     # Email alert sub-flows (3 types)
    │   │   └── error-handler.xml          # Global typed error handler
    │   └── resources/
    │       └── config.yaml                # All app properties
    └── test/
        ├── munit/
        │   └── inventory-sync-munit-suite.xml   # 7 MUnit test cases
        └── resources/
            └── test-inventory.csv               # Sample data (valid + invalid rows)
```

---

## Running Locally (Demo Mode)

1. **Prerequisites**: Anypoint Studio 7.x, JDK 8+, Maven 3.6+

2. **Create input/output folders**:
   ```bash
   mkdir -p /tmp/inventory-sync/input /tmp/inventory-sync/output /tmp/inventory-sync/input/archive
   ```

3. **Place today's inventory file** (adjust date):
   ```bash
   cp src/test/resources/test-inventory.csv \
      /tmp/inventory-sync/input/full_inventory_$(date +%d-%m-%Y).csv
   ```

4. **Run in Anypoint Studio**: Right-click project → Run As → Mule Application

5. **Check output**:
   ```bash
   ls /tmp/inventory-sync/output/
   # inventory_WHBKS_<date>.csv
   # inventory_FBVTS_<date>.csv
   ```

6. **Run MUnit tests**: Right-click `src/test/munit/*.xml` → Run MUnit Suite

---

## Enhancements Beyond Requirements

1. **Source file archival** — processed files moved to `archive/` folder, preventing reprocessing.
2. **Idempotency** — output files are date-stamped; re-running on same date overwrites cleanly.
3. **Case-insensitive type validation** — `Standard`, `STANDARD`, `standard` all pass (defensive against WMS formatting quirks).
4. **Correlation ID in all logs** — enables distributed tracing across CloudHub log streams.
5. **Reusable notification sub-flows** — 3 separate email alert types, each independently testable.
6. **SFTP + File connector** — switch between local (dev) and SFTP (prod) by changing config only, no code change.
