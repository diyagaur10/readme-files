# Accounting Ledger Service

> **Version**: See `changelog.md` for change history  
> **Default Port**: `10087`  
> **Runtime**: Python 3.10 / Flask 1.1.4  
> **Deployment**: Dockerized, AWS (EC2 + S3 + DynamoDB)

---

## Table of Contents

1. [Service Overview](#1-service-overview)
2. [Project Structure & Module Responsibilities](#2-project-structure--module-responsibilities)
3. [High-Level Architecture](#3-high-level-architecture)
4. [End-to-End Service Flow](#4-end-to-end-service-flow)
5. [Critical Code Walkthrough](#5-critical-code-walkthrough)
6. [Configuration & Environment Setup](#6-configuration--environment-setup)
7. [Failure Scenarios & Debugging Guide](#7-failure-scenarios--debugging-guide)
8. [Operational & Infra Considerations](#8-operational--infra-considerations)
9. [Local Development & Runbook](#9-local-development--runbook)

---

## 1. Service Overview

### Business Problem

In the InsentiHub incentive-compensation platform, agents earn commissions and entitlements across multiple payout lifecycle stages. Each financial event (provisioning, accrual, tax liability, settlement) must generate corresponding **double-entry accounting ledger records** in the client's General Ledger (GL) system. Without these records, the finance team cannot reconcile payouts, comply with GST/TDS regulations, or produce accurate financial statements.

### Core Responsibilities

| Responsibility | Description |
|---|---|
| **GL Entry Generation** | Produces balanced CR/DR pairs for every accounting rule invocation |
| **Multi-Stage Support** | Handles Provisional, Accrual, Settlement, and Adjustment stages |
| **Tax-Aware Entries** | Generates GST (RCM/FCM), Withholding Tax, and Local Tax entries |
| **Idempotency / Deduplication** | Prevents duplicate ledger records by comparing against existing DB entries |
| **Bulk Upload Orchestration** | Creates a Batch, uploads a CSV file, and triggers bulk processing via the platform's batch API |

### Explicit Non-Responsibilities

- **Does NOT** calculate entitlement amounts (handled by upstream entitlement services)
- **Does NOT** trigger payout cycle creation or progression
- **Does NOT** directly write to the GL database — it delegates to the platform's BulkCUD (Bulk Create-Update-Delete) batch pipeline
- **Does NOT** send notifications or emails

### System Context

```
Payout Cycle Trigger
       |
       v
 [Accounting Trigger Action Service]
       |  POST /accounting-ledger-service/<payout_cycle_id>/<gl_acc_rule_id>
       v
 [accounting-ledger-service]  <-- THIS SERVICE
       |
       |-- Reads payout cycle & entitlement data (BulkRead API / S3)
       |-- Looks up GL rule master & GL mapping tables (GLCache)
       |-- Generates balanced CR + DR CSV rows
       |
       v
 [Platform Batch/BulkCUD API]
       |
       v
 [accountingledger Object in Platform DB]
```

---

## 2. Project Structure & Module Responsibilities

```
accounting-ledger-service-develop/
├── accounting_ledger_service/          # Main application package
│   ├── controller.py                   # Flask entry point & HTTP route
│   ├── service_handler.py              # Top-level orchestration / handler
│   ├── FactoryPattern.py               # Factory: selects correct ledger impl
│   ├── account.py                      # Abstract base class AccountLedger + AgentLevelAccountLedger
│   ├── GLCache.py                      # In-memory cache for GL master data
│   ├── Util.py                         # BulkRead API helper, payout cycle fetcher
│   ├── custom_code.py                  # Specialised data-to-file transformers
│   └── impl/                           # Concrete ledger implementations
│       ├── Accrual.py                  # AgentPayble, WithholdTax, LocalTax, GST
│       ├── Provisional.py              # ProvisionalBase, ParentTxn, ChildTxn, AgentTxn
│       ├── Settlement.py               # BankPayble (COMMPAY)
│       ├── Adjustment.py               # TaxableAdjustment, NonTaxableAdjustment
│       ├── BulkBatchAction.py          # Batch creation & bulk upload orchestration
│       └── __init__.py
├── Tests/                              # Unit tests
│   ├── test_Accrual.py
│   ├── test_Adjustment.py
│   ├── test_Provisional.py
│   ├── test_Settlement.py
│   └── test_Util.py
├── test/                               # Integration / manual test fixtures
│   ├── event*.json                     # Sample event payloads per GL rule
│   └── test.py
├── Dockerfile
├── run.sh                              # Docker run script (used by CI/CD)
├── build.sh
├── requirements.txt
├── pip.conf                            # Internal PyPI mirror config
└── changelog.md
```

### Architectural Pattern

The service follows a **Layered + Strategy/Factory** architecture:

```
HTTP Layer          → controller.py  (Flask route)
Orchestration Layer → service_handler.py  (event wiring)
Domain Layer        → account.py  (abstract base, common pipeline)
Strategy Layer      → impl/*.py  (concrete GL rule implementations)
Infrastructure Layer→ GLCache.py, Util.py, BulkBatchAction.py  (external calls)
```

The Factory Pattern (`FactoryPattern.py`) selects the correct concrete class at runtime based on the `gl_acc_rule_id` passed in the request. All concrete classes share a common processing pipeline defined in `AccountLedger.createAccountingUploadFile()`.

---

## 3. High-Level Architecture

### Entry Point

One REST endpoint accepts all requests:

```
POST /accounting-ledger-service/<payout_cycle_id>/<gl_acc_rule_id>
```

**Required HTTP Headers:**

| Header | Description |
|---|---|
| `client` | Client/tenant identifier (e.g., `kli`, `ifl`) |
| `domain` | Domain identifier (e.g., `incentihub`) |
| `stage_id` | Lifecycle stage (e.g., `provisional`, `accrual`, `settlement`) |
| `stage` | Deployment environment tag (e.g., `CLIENT_SB`, `QA`) |

### External Dependencies

| Dependency | Usage |
|---|---|
| **BulkRead API** | Paginated reads for entitlement, GL rule master, GL maps |
| **Platform Batch API** | Creates batch, adds parallel task, triggers processing |
| **AWS S3** | Downloads provisional calculation report CSV files |
| **AWS DynamoDB** | Runtime properties store (via `property-fetcher` library) |
| **Platform Object API** | Reads payout cycle data (single object GET) |
| **`accountingledger` read** | Checks for existing ledger records during deduplication |

### Data Flow Diagram

```
HTTP POST
   |
   v
[controller.py]
   | – extract headers (client, domain, stage_id, stage)
   | – call service_handler.accounting_handler(event)
   v
[service_handler.py]
   | – load properties from DynamoDB
   | – fetch payoutcycle via Object Read API
   | – FactoryPattern.getType() → selects impl class
   | – AccountLedger.start(event)
   v
[AccountLedger.start()]
   | – getData()      → fetches raw data (BulkRead API or S3)
   | – createAccountingUploadFile()  → builds CR+DR CSV in chunks
   | – callBulkCUD()  → uploads via BulkBatchAction
   v
[BulkBatchAction]
   | – createBatch()
   | – createBatchParallelTask()
   | – documentcud()  (uploads CSV)
   | – mock_bulkaction_statusupdate_trigger()
   v
[Platform Bulk Processing → accountingledger records created]
```

---

## 4. End-to-End Service Flow

### Step 1 — Request Reception (`controller.py`)

The Flask route extracts `payout_cycle_id` and `gl_acc_rule_id` from the URL path, plus tenant headers. It assembles an `event` dict and delegates to `service_handler.accounting_handler()`.

### Step 2 — Properties Load (`service_handler.py`)

`property_fetcher.load_properties(event)` reads from AWS DynamoDB using the tenant context. These properties drive all configurable behaviour (URLs, batch parameters, column mappings, etc.).

### Step 3 — Payout Cycle Fetch (`Util.fetch_payoutcycle`)

If `payoutcycle` is not already in the event (it won't be on a normal API call), the service performs a GET to the Platform Object Read API using the `Payoutcycle.ObjectRead.URL` property. The result is stored in `event["payoutcycle"]`.

### Step 4 — Factory Selection (`FactoryPattern.getType`)

`GLCache` is initialised here. It immediately fetches:
- All `accountingglrulemaster` records (paginated)
- All `masterglclientglmap` records (paginated)

The factory validates:
1. `gl_acc_rule_id` is not None
2. `stage_id` is not None
3. The rule is `applicable == True` in the GL rule master
4. The rule's configured `stage` matches the requested `stage_id` (case-insensitive)

If any check fails, an exception is raised immediately.

Based on `gl_acc_rule_id`, the correct concrete class is selected:

| GL Rule IDs | Class | Stage |
|---|---|---|
| `PRVNCTXN`, `PRVNCTXNREC` | `ChildTxn` | provisional |
| `PRVNPTXN`, `PRVNPTXNREC` | `ParentTxn` | provisional |
| `PRVNAGT`, `PRVNAGTREC` | `AgentTxn` | provisional |
| `COMMACCRUEAGT`, `COMMACCRUEAGTREC` | `AgentPayble` | accrual |
| `COMMACCRUEWITHOLD`, `COMMACCRUEWITHOLDREC` | `WithholdTax` | accrual |
| `COMMACCRUELT`, `COMMACCRUELTREC` | `LocalTax` | accrual |
| `COMMACCRUEVATRCM`, `COMMACCRUEVATRCMREC` | `GST` (RCM) | accrual |
| `COMMACCRUEVATFCM`, `COMMACCRUEVATFCMREC` | `GST` (FCM) | accrual |
| `TaxableCommExtra`, `TaxableCommLess` | `TaxableAdjustment` | accrual |
| `NonTaxableCommExtra`, `NonTaxableCommLess` | `NonTaxableAdjustment` | accrual |
| `COMMPAY`, `COMMPAYREJ`, `COMMPAYREJ1` | `BankPayble` | settlement |

**Reversal flag**: `isReversalEntry` is set to `True` when the rule ID ends with `REC`, `Less`, `REJ`, or `REJ1`. This inverts the amount condition operator from `>` to `<` and flips DR/CR logic in some implementations.

### Step 5 — Data Acquisition (`getData`)

Two strategies exist depending on the class hierarchy:

**A. `AgentLevelAccountLedger.getData()` (Accrual, Settlement, Adjustment)**  
Calls `getBulkReadAPI()` with a paginated POST to fetch `agententitlement` or `agentpaymentledgertxn` records. Each page is written to a temp CSV file via a callback (`agentLevelDataToFile`).

**B. `ProvisionalBase.getData()` (Provisional)**  
Downloads a pre-computed calculation report CSV from **AWS S3**. The S3 path comes from `payoutcycle.cyclereview.document_attributes` (either `parent_calculation_report` or `transaction_calculation_report`). File is saved locally under `/tmp/payoutcycle/<cycleid>/`.

### Step 6 — CSV Construction (`createAccountingUploadFile`)

Processes the raw data file in configurable chunks (`FILE_READ_CHUNK_SIZE`, default 10,000 rows):

1. **`chunkPreprocessingHook`** — optional per-implementation filtering/transformations
2. **Column renaming** — driven by `<task_id>.column_convert` property
3. **Set common fields** — `compensation_header`, `compensation_entity`, `payout_cycle_id`, `stage`, `gl_acc_rule_id`, `dr_or_cr=CR`, `incenti_txn_code`, `incenti_gl_code`, `accounting_level`
4. **`populateRuleSpecificData` (CR side)** — looks up `client_gl_code` from GLCache
5. **`populateAmtAndDate`** — sets `amount` and `accounting_date` fields
6. **Drop rows with null `accounting_date`** — silently skipped, logged
7. **Drop rows with `amount == 0`** — only positive amounts flow through
8. **Round amounts** to `RoundOFF_DIGITS` decimals (default 2)
9. **`RemoveDuplicates`** — if enabled in `remove_duplicate_applicable_list`, fetches existing CR records and removes matching ones from the new batch (Counter-based matching, not keyed by ID)
10. **Create DR copy** — `df_dr = filtered_df_cr.copy()` then `dr_or_cr = DR`, GL code flipped to debit GL
11. **`chunkPreprocessingHookDR`** — optional DR-side transformations
12. **`populateRuleSpecificData` (DR side)**
13. **Column filtering** — keeps only columns listed in `<task_id>.column_to_keep` property
14. Write CR rows then DR rows to the output CSV (append mode after first chunk)

### Step 7 — Bulk Upload (`callBulkCUD` → `BulkBatchAction`)

1. **`createBatch`** — POST to platform batch API. If batch already exists (422 + "already exists"), treated as success.
2. **`createBatchParallelTask`** — PUT to add a parallel task to the batch (identified by `task_id`)
3. **`documentcud`** — multipart POST to upload the CSV file; the CSV is sent as `file=1.csv`
4. **Sleep** — waits `Wait_Time_Before_Upload_Trigger` seconds (default 30s) before triggering
5. **`create_mock_statusupdate_payload`** — fetches batch state from Object Read API and constructs the trigger payload
6. **`mock_bulkaction_statusupdate_trigger`** — POSTs to `DocumentBatchProcessing.URL` to start asynchronous bulk processing

### Key Branching Points

| Decision | Logic |
|---|---|
| Provisional vs Accrual/Settlement data source | Class hierarchy — `ProvisionalBase` uses S3, `AgentLevelAccountLedger` uses BulkRead |
| Reversal entry | `isReversalEntry` flag set from rule ID suffix; flips amount operator and GL code |
| GST type (RCM vs FCM) | `setGSTType()` sets `gst_type`; different data sources and GL code lookups |
| Settlement nature_of_payment | Driven by `settlement.<task_id>.add_nature_of_payment_list` property |
| Accounting date source | `accounting_date_type.mapping` property → `object_id` + `attribute_id` pair |

---

## 5. Critical Code Walkthrough

### `GLCache` — The Performance Cornerstone

**File**: `accounting_ledger_service/GLCache.py`

Instantiated once per request in `FactoryPattern.getType()`. On construction it eagerly loads:
- **`accountingglrulemaster`** — all GL rules (keyed by `gl_acc_rule_id`)
- **`masterglclientglmap`** — all master-to-client GL code mappings (keyed by composite: `incenti_gl_code + compensation_header + compensation_entity + nature_of_payment`)

Lazy-loaded on demand:
- **`accounting_lt_gl_map_records`** — LT (local tax) GL maps; loaded by `LocalTax.getPayload()`
- **`accounting_adjustment_gl_map_records`** — adjustment GL maps; loaded by `TaxableAdjustment.getPayload()`
- **`accounting_gst_gl_map`** — GST GL maps; loaded by `GST.setGSTType()`

> **⚠️ Important**: GLCache holds **all GL master data in memory** for the lifetime of a single request. There is no cross-request caching. Every API call re-fetches the entire master dataset. This is safe for correctness but can be slow if master tables are large.

### `AccountLedger.RemoveDuplicates` — Idempotency Guard

**File**: `accounting_ledger_service/account.py`, lines 230–318

Enabled only if `task_id` appears in the `remove_duplicate_applicable_list` property.

Algorithm:
1. Fetches all existing CR ledger records for this `payout_cycle_id` + `gl_acc_rule_id` + `agent_codes` via BulkRead
2. Builds a `Counter` of existing records keyed by the deduplication columns (configurable via `<task_id>.duplicate_match_key_columns` or `duplicate_match_key_columns_by_gl_acc_rule_id`)
3. Iterates new records; for each, if a matching existing record count exists, marks the new record for removal and decrements the counter (handles multiple occurrences correctly)

> **Inferred from code**: The deduplication only applies to the **CR side**. DR side is generated as a copy of the CR side after deduplication, so it is implicitly deduplicated.

> **Known gap**: `if "No record found for object_id" in str(e)` — if the BulkRead returns no records (first-time run), this is treated as a valid empty result and the new records pass through unchanged.

### `AccountLedger.createAccountingUploadFile` — Main Pipeline

**File**: `accounting_ledger_service/account.py`, lines 68–215

The most complex method. Key non-obvious behaviours:

- **`is_first_call` flag** controls whether to write the CSV header (first chunk → `to_csv(…)`, subsequent chunks → `to_csv(mode='a', header=False)`)
- **`amount` is always made absolute** (`abs(row.amount)`) before filtering — this means sign is irrelevant; amounts are passed as absolute values and direction is encoded in `dr_or_cr`
- **`accounting_txn_id` is in `mandatory_columns` but never populated by this service** — it is expected to be populated by the downstream BulkCUD processing pipeline using the platform's transaction ID generation
- **`checkIfFIleEmpty`** returns `False` if the file does not exist (all chunks were empty, no records), which causes an exception to be raised

### `BulkBatchAction.mock_bulkaction_statusupdate_trigger` — Trigger Mechanism

**File**: `accounting_ledger_service/impl/BulkBatchAction.py`, lines 82–91

The term "mock" is a misnomer in production context. This method calls the real `DocumentBatchProcessing.URL` endpoint to trigger the actual batch processing. The name originates from early development where this simulated a Lambda/API Gateway callback. **Inferred from code**: In production, this URL points to a real processing trigger endpoint (possibly a Lambda via API Gateway or an NLB endpoint).

### `GST.chunkPreprocessingHook` — CGST/SGST/IGST Row Expansion

**File**: `accounting_ledger_service/impl/Accrual.py`, lines 156–167

When from_state == to_state (intra-state transaction), two GST components apply: CGST and SGST. The hook duplicates such rows and tags one as `CGST` and the duplicate as `SGST`. Inter-state rows get tagged as `IGST`. This doubles the row count for intra-state GST transactions.

### `ProvisionalBase.populateClientGL` — Parameterised GL Code Lookup

**File**: `accounting_ledger_service/impl/Provisional.py`, lines 66–87

Provisional entries support parameterised GL code selection. Each `masterglclientglmap` record can have `input_params` — a JSON array of `{column: value}` dicts. The method iterates these conditions against the current row. The **first matching** condition wins. If no conditions match, falls back to the default `client_gl_account_code`. This is used to route different transaction types to different GL codes within the same compensation header.

---

## 6. Configuration & Environment Setup

### Runtime Environment Variables

| Variable | Required | Description |
|---|---|---|
| `FLASK_APP` | Yes | Must be `accounting_ledger_service/controller.py` |
| `FLASK_RUN_PORT` | Yes | Port to expose; production default is `10087` |
| `FLASK_ENV` | Yes | `development` or `production` |
| `DEPLOYMENT_TYPE` | Yes | `PROD` enables production Flask environment |
| `AWS_DEFAULT_REGION` | Yes | AWS region; default in run.sh is `ap-south-1` |
| `service_name` | Yes | Must be `accounting_ledger` (used by `property-fetcher`) |
| `resources` | Yes | Must be `logger,shared` (used by internal library) |
| `CONTAINER_NAME` | Yes | `accounting_ledger_service` (used for container identification) |
| `ALS_PORT` | Yes (run.sh) | Port to bind on the host; mapped to container `FLASK_RUN_PORT` |

### DynamoDB Properties

All runtime configuration is stored in a DynamoDB `properties` table. The partition key is `<domain>#<client>#<stage>` and the sort key is `accounting_ledger#<property_key>`.

Critical properties:

| Property Key | Type | Description |
|---|---|---|
| `BulkRead.URL` | String | Base URL for paginated bulk reads |
| `BulkRead.Method` | String | HTTP method (default `POST`) |
| `BulkRead.Headers` | JSON template | Headers with `<domain>` and `<client>` placeholders |
| `BULK_READ_BATCH_SIZE` | Integer | Page size for paginated API calls |
| `Payoutcycle.ObjectRead.URL` | URL template | `<payout_cycle_id>` placeholder |
| `Batch.ObjectRead.URL` | URL template | `<batch_id>` placeholder |
| `accounting_date_type.mapping` | JSON | Maps `accounting_date_type` → `{object_id, attribute_id}` |
| `FILE_READ_CHUNK_SIZE` | Integer | Pandas chunk size (default 10,000) |
| `RoundOFF_DIGITS` | Integer | Decimal rounding (default 2) |
| `mandatory_columns` | CSV string | Columns that must exist in output; nulled if missing |
| `remove_duplicate_applicable_list` | JSON array | List of `task_id`s where dedup is active |
| `duplicate_match_key_columns_by_gl_acc_rule_id` | JSON object | Per-rule dedup key columns |
| `BatchParallelTask_core_attributes` | JSON template | `<task_id>` placeholder for batch task setup |
| `Wait_Time_Before_Upload_Trigger` | Integer | Sleep seconds before trigger (default 30) |
| `mock_statusupdate_payload` | JSON template | Payload template with `<batch_data>`, `<batch_id>`, `<domain>`, `<client>`, `<timestamp>` |
| `DocumentBulkUpload.*` | Multiple | URLs, methods, headers, payloads for batch/file upload APIs |
| `DocumentBatchProcessing.URL` | String | URL to trigger bulk processing |
| `DocumentBatchProcessing.METHOD` | String | HTTP method (default `POST`) |
| `<task_id>.column_convert` | JSON object | Column rename map for specific rule |
| `<task_id>.column_to_keep` | CSV string | Columns to retain in output CSV |
| `settlement.<task_id>.add_nature_of_payment_list` | JSON object | Incenti GL codes needing nature_of_payment for settlement |
| `GetagentLevelDataToFile.exclude_columns` | CSV/list | Columns exempt from NaN fill in agent entitlement reads |

### Internal Libraries

These are fetched from an internal PyPI mirror configured in `pip.conf`:

| Library | Version | Purpose |
|---|---|---|
| `HttpCaller` | >=1.1.2.4 | HTTP client wrapper (`CustomHttpTemplate`, `HttpImpl`) |
| `logger` | >=1.3.1.0 | Structured logging (overrides built-in `print`) |
| `property-fetcher` | >=1.1.4.0 | DynamoDB-backed runtime config loader |
| `sqsSnsMessage` | >=1.1.2.0 | SQS/SNS utilities (imported but not directly invoked in reviewed code) |
| `common-filters` | >=1.0.0.0 | Flask request/response filters (`apply_filters`) |
| `sqsExtendedClient` | 1.0.0.0 | Extended SQS client |
| `botoinator` | 0.0.6 | Boto3 utility |

---

## 7. Failure Scenarios & Debugging Guide

### Common Failure Modes

#### 1. `accounting datetype mapping not added in properties`
**Cause**: `accounting_date_type.mapping` property is missing or does not contain an entry for the `gl_rule_master.accounting_date_type` value.  
**Debug**: Check DynamoDB for the property. Verify the map includes the correct key format: `<task_id>.<accounting_date_type>` (preferred) or just `<accounting_date_type>`.  
**Fix**: Add the correct mapping entry: `{"object_id": "payoutcycle", "attribute_id": "cycle_end_date"}`.

#### 2. `gl_acc_rule_id or Stage not configured or gl_acc_rule_id not applicable`
**Cause**: Mismatch between requested `stage_id` and the `stage` configured in `accountingglrulemaster`, or `applicable` is `False`.  
**Debug**: Verify the GL rule master record in the platform for the tenant. The `stage` comparison is case-insensitive (`.lower()==stage_id`).  
**Fix**: Update the GL rule master record or correct the `stage_id` header in the calling service.

#### 3. `NO records found with for the requested accounting code`
**Cause**: After all chunk processing, the output CSV is empty. Could mean: no entitlement records match the where-clause, all amounts are zero, all `accounting_date` values are null, or all records were filtered as duplicates.  
**Debug**: Check the bulk read where-clause (e.g., `net_accrued_amount > 0`). Verify entitlement records exist for the payout cycle. Check if dedup removed everything.  
**Fix**: Confirm upstream entitlement calculation has run.

#### 4. `Error calling bulk read API for accountingledgerread`
**Cause**: The existing ledger read (deduplication check) failed with an unexpected error.  
**Debug**: Look at the specific exception from the API call. Check `accountingledgerread` service repository configuration.  
**Note**: If the error message contains "No record found for object_id", it is treated as "first run" and is **not** an error.

#### 5. S3 Download Failures (`storeS3FileLocally`)
**Cause**: IAM permissions for the EC2/container role are insufficient, or the S3 object does not exist (payout cycle review hasn't generated the report yet).  
**Debug**: Check boto3 error code in logs (`404` = file missing, `403` = permission denied). Verify the `cyclereview.document_attributes` in the payout cycle object.  
**Fix**: Ensure the provisional calculation pipeline has completed before this service is called.

#### 6. `Error creating batch` / `Error creating BatchParallelTask`
**Cause**: The platform batch API is unreachable or returned a non-200, non-422 status.  
**Debug**: Check `DocumentBulkUpload.CreateBatch.URL` property. Inspect HTTP response in logs.  
**Note**: 422 "already exists" is treated as success (idempotent batch creation).

#### 7. `error processing bulk CUD action`
**Cause**: The bulk processing trigger failed (`DocumentBatchProcessing.URL` returned non-200).  
**Debug**: Check the platform's batch processing service. Inspect batch status for `batch_id = accounting_<cycleid>`.  
**Note**: At this point the CSV file has already been uploaded — retrying the full flow will attempt batch recreation (idempotent) and re-upload.

### Unsafe Assumptions / Edge Cases

| Scenario | Risk |
|---|---|
| Multiple concurrent calls for same payout cycle + rule | Race condition in dedup: both calls read existing records simultaneously, both may proceed |
| Large payout cycles (>1M agents) | Temp CSV files can grow very large; container disk space may be insufficient |
| GST `from_state`/`to_state` not in `gstglmap` | Raises exception at `calcClientGLCode`; all agents in that state will fail |
| `accounting_txn_id` not populated | Intentional — this field is expected to be auto-generated by the platform |
| `nature_of_payment` null for settlement | Has fallback: defaults to empty string for GL code lookup |

### Debugging Using Logs

The service uses the internal `logger` library (overrides built-in `print`). All `print()` calls are structured log statements.

Key log patterns to search for:

```bash
# Request start
grep "EVENT:" logs

# GL cache size (implicit in HTTP 200 responses)
grep "accounting_gl_rule_master" logs

# Chunk processing progress
grep "DataFrame has been successfully saved" logs

# Deduplication
grep "first time records" logs   # First-run dedup (no existing records)

# Batch upload progress
grep "processing complete" logs   # Successful completion

# Errors
grep "logtype='fatal'" logs       # Fatal errors from controller
```

---

## 8. Operational & Infra Considerations

### Startup Sequence

1. Container starts Flask with `flask run --host=0.0.0.0` on `FLASK_RUN_PORT`
2. No warm-up or pre-loading occurs — GL master data is fetched on the **first request**
3. Timezone is set to `Asia/Kolkata` (IST) in the Docker image

### Shutdown

- Flask process handles `SIGTERM` gracefully (standard Flask behaviour)
- No in-flight request state needs cleanup (all processing is synchronous within a request)
- Temp CSV files in the container's working directory may be orphaned if the process is killed mid-request (cleanup is in the `finally` block of `AccountLedger.start()`)

### Health Check

There is **no dedicated `/health` endpoint**. Health is implicitly confirmed by the container being reachable on the exposed port.

> **Inferred from code**: For production, add a simple `/health` GET route. Alerting should be based on error rates in logs.

### Performance Profile

| Operation | Typical impact |
|---|---|
| GLCache init (per request) | 2 paginated API calls at startup; cost grows with master data size |
| BulkRead API calls | Dominant time cost; proportional to number of entitlement records |
| S3 download (Provisional) | Network-bound; depends on S3 region proximity |
| Pandas chunked processing | CPU-bound; `FILE_READ_CHUNK_SIZE=10000` is a key tuning knob |
| Sleep before trigger | Fixed `Wait_Time_Before_Upload_Trigger` seconds (default 30s) added to every request |

### Scaling Considerations

- The service is **stateless** between requests — horizontal scaling is safe
- **Concurrency within a single payout cycle should be avoided**: multiple simultaneous requests for the same `payout_cycle_id` + `gl_acc_rule_id` may create duplicate batch records or race in deduplication
- Disk I/O is significant: ensure the container's ephemeral storage is sufficient for large CSVs
- The 30-second sleep is a hard bottleneck per invocation; for large payout cycles with many rules, this significantly extends total processing time

### Resource Sensitivity

| Resource | Sensitivity |
|---|---|
| Disk (ephemeral) | HIGH — temp CSV files created in working directory |
| Memory | MEDIUM — Pandas DataFrames held per chunk; GLCache held per request |
| Network | HIGH — multiple paginated API calls + S3 download |
| CPU | LOW-MEDIUM — Pandas operations on chunks |

---

## 9. Local Development & Runbook

### Prerequisites

- Python 3.10
- Docker (for container-based runs)
- AWS credentials configured (`~/.aws/credentials` or env vars) with access to:
  - DynamoDB table (`properties`)
  - S3 (for provisional stage only)
- Access to the internal PyPI mirror (configure `pip.conf` with your credentials)
- Network access to the platform's BulkRead, Object Read, and Batch APIs

### Running Directly (without Docker)

```bash
cd accounting-ledger-service-develop/

# Install dependencies (internal pip mirror required)
pip install -r requirements.txt

# Set environment variables
export service_name=accounting_ledger
export resources=logger,shared
export FLASK_APP=accounting_ledger_service/controller.py
export FLASK_RUN_PORT=10087
export AWS_DEFAULT_REGION=ap-south-1

# Run
cd accounting_ledger_service
flask run --host=0.0.0.0
```

### Running via Docker

```bash
cd accounting-ledger-service-develop/

# Build image
docker build -t accounting_ledger_service .

# Run (replace ALS_PORT with your local port)
export ALS_PORT=10087
bash run.sh
```

### Testing a Single GL Rule

Use the sample payloads in `test/`:

```bash
# Example: trigger COMMACCRUEAGT rule
curl --location --request POST \
  'http://localhost:10087/accounting-ledger-service/<payout_cycle_id>/COMMACCRUEAGT' \
  --header 'client: <your_client>' \
  --header 'domain: incentihub' \
  --header 'stage_id: accrual' \
  --header 'stage: CLIENT_SB'
```

Or run `service_handler.py` directly for local debugging (edit the `test` dict at the bottom of the file):

```bash
cd accounting_ledger_service
python service_handler.py
```

### Running Unit Tests

```bash
cd Tests/
python -m pytest test_Accrual.py test_Adjustment.py test_Provisional.py test_Settlement.py test_Util.py -v
```

### Test Data & Mock Dependencies

- `test/event*.json` files contain sample event payloads for each GL rule type
- `test/data.json` contains sample entitlement data
- For Provisional tests, mock the S3 download or provide a local CSV at the expected path

### Common Developer Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Wrong `stage_id` header | `gl_acc_rule_id not applicable` error | Check GL rule master `stage` field |
| Missing DynamoDB property | `KeyError` or `NoneType` exceptions | Add property to DynamoDB for the tenant |
| Running from wrong directory | `ModuleNotFoundError` | Run flask from inside `accounting_ledger_service/` |
| pip.conf not configured | `pip install` fails | Configure internal PyPI mirror credentials |
| S3 file not present | Silent empty `filepath` + eventual exception | Ensure provisional calculation pipeline ran first |
| `accounting_date_type` not in mapping | Exception on `AccountLedger.__init__` | Add correct entry to `accounting_date_type.mapping` property |

### Adding a New GL Rule

1. Determine which base class the new rule inherits from:
   - Agent-level data from API → subclass `AgentLevelAccountLedger`
   - Report file from S3 → subclass `ProvisionalBase`
2. Implement at minimum:
   - `getPayload()` — BulkRead where-clause and select-clause
   - `getObjctId()` — the object ID to query
   - `agentLevelDataToFile()` — callback to transform API response to CSV
   - `populateAmtAndDate()` — set `amount` and `accounting_date` columns
   - `populateRuleSpecificData()` (optional) — override if GL code lookup differs from standard
3. Add `gl_acc_rule_id` handling in `FactoryPattern.getType()` (`impl/` class must be imported)
4. Add GL rule master record in the platform (with correct `stage`, `applicable=True`, GL codes configured)
5. Configure `accounting_date_type.mapping` in DynamoDB for the new rule
6. Add unit test in `Tests/` mirroring the existing test structure

---

## Appendix — GL Rule ID Reference

```
Provisional Stage:
  PRVNCTXN      Child transaction ledger (normal)
  PRVNCTXNREC   Child transaction ledger (reversal)
  PRVNPTXN      Parent transaction ledger (normal)
  PRVNPTXNREC   Parent transaction ledger (reversal)
  PRVNAGT       Agent-level provisional (normal)
  PRVNAGTREC    Agent-level provisional (reversal)

Accrual Stage:
  COMMACCRUEAGT       Agent payable (normal)
  COMMACCRUEAGTREC    Agent payable (reversal)
  COMMACCRUEWITHOLD   Withholding tax (normal)
  COMMACCRUEWITHOLDREC Withholding tax (reversal)
  COMMACCRUELT        Local tax (normal)
  COMMACCRUELTREC     Local tax (reversal)
  COMMACCRUEVATRCM    GST Reverse Charge Mechanism (normal)
  COMMACCRUEVATRCMREC GST Reverse Charge Mechanism (reversal)
  COMMACCRUEVATFCM    GST Forward Charge Mechanism (normal)
  COMMACCRUEVATFCMREC GST Forward Charge Mechanism (reversal)
  TaxableCommExtra    Taxable commission adjustment (extra)
  TaxableCommLess     Taxable commission adjustment (less/reversal)
  NonTaxableCommExtra Non-taxable commission adjustment (extra)
  NonTaxableCommLess  Non-taxable commission adjustment (less/reversal)

Settlement Stage:
  COMMPAY     Bank payment entries
  COMMPAYREJ  Payment rejection (reversal)
  COMMPAYREJ1 Payment rejection variant (reversal)
```
