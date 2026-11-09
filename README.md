# File_Validation_Dispatcher

**File Validation & Routing Framework** — accepts files from four heterogeneous sources, validates them against versioned, metadata-driven contracts (file type, size, emptiness, duplication, schema match, column order, optional regional/country residency), routes **PASS** files to per-file-type subfolders of the Databricks Intake volume and **FAIL** files to Quarantine, and **certifies end-to-end resilience** by reconciling every uploaded file against Databricks bronze processing.

## Pipeline

```
4 sources → EVENT INGESTION (canonical FILE_ARRIVED) → STAGE (GridFS)
→ IDENTIFY (explicit → filename pattern → header fingerprint → content sniff)
→ VALIDATE (FILE → HEADER → CONTENT rules; plugin validators; WARN = shadow mode)
→ ROUTE (priority policies → transfer manifest → Intake/Quarantine, checksum verified)
→ Databricks Intake volume → Auto Loader → bronze → silver (DQ exceptions → silver quarantine) → gold
→ PUSHBACK (per-file bronze stats → FastAPI → MongoDB)
→ RECONCILIATION (uploaded = routed = transferred = bronze processed)
→ resilience certificate → gates GridFS purge (archival)
```

## Sources & detection

| # | Source | Detection |
| --- | --- | --- |
| 1 | UI Portal (React) | Push — direct FastAPI upload API |
| 2 | SharePoint document libraries | Microsoft Graph change notifications (webhook); change-token poller fallback |
| 3 | External → Internal SFTP | File-system watcher; poller with watermark + "file stable" check |
| 4 | Databricks Volume | Scheduled watcher diffing listings vs `file_registry` |

## Architecture highlights

* **Event-driven** — all arrivals/transitions/outcomes are canonical events in `event_log` (unique `event_type` + `dedup_key` → idempotent, at-least-once safe). Transport: MongoDB change streams to start; RabbitMQ/Redis Streams if volume grows.
* **Physical store** — MongoDB GridFS holds the bytes; all framework metadata lives in `file_registry` (20-collection catalog: registry / eventing / execution / governance / resilience layers).
* **Metadata-driven & versioned** — file types, schemas, rules, routing, and retention are documents, not code. New rule = insert a `validation_rules` doc (parameter-only, no deploy); new rule *kind* = one `BaseValidator` subclass registered by decorator; onboard safely in `WARN` shadow mode.
* **Resilience proof** — reconciliation compares uploaded vs routed vs transferred vs bronze-processed per file (`file_id`/`sha256` carried into bronze `_metadata`); clean `MATCHED` = resilience certificate, and the **purge gate**: GridFS copies are deleted only after a `MATCHED` result.

## Design document

The consolidated v1.3 design (architecture, 20-collection data model, state machine, mongosh `$jsonSchema` validators + index specs, Databricks ingestion-results pushback job design, open decisions) lives in **`File Validation & Routing Framework — Design & Specs.ipynb`** (9 markdown cells, documentation-only).

## Repository layout (planned)

```
File_Validation_Dispatcher/
├── README.md                                  # this file
├── docs/
│   └── design/                                 # design documents (move the .ipynb here as docs grow)
├── backend/                                    # FastAPI application (Python)
│   ├── app/
│   │   ├── main.py                            # FastAPI entrypoint
│   │   ├── api/                                # routers
│   │   │   ├── files.py                        #   portal upload endpoints (push)
│   │   │   ├── events.py                       #   internal event webhooks (SharePoint Graph)
│   │   │   ├── ingestion_results.py            #   /internal/ingestion-results (Databricks pushback)
│   │   │   └── admin.py                        #   rules / schemas / routing / retention config CRUD
│   │   ├── core/                               # settings, security, logging, event bus
│   │   ├── models/                             # Pydantic models + Mongo access (motor/pymongo)
│   │   ├── services/
│   │   │   ├── staging.py                      # GridFS staging + sha256 checksums
│   │   │   ├── identification.py               # 4-step file-type identification
│   │   │   ├── validation/                     # rule engine
│   │   │   │   ├── base.py                     #   BaseValidator interface
│   │   │   │   ├── registry.py                 #   @register_validator plugin registry
│   │   │   │   └── rules/                      #   FILE / HEADER / CONTENT validators
│   │   │   ├── routing.py                      # policy evaluation (priority, first-match)
│   │   │   ├── transfer.py                    # GridFS → volume workers (checksum verify, idempotent)
│   │   │   ├── reconciliation.py               # resilience certification
│   │   │   └── archival.py                      # retention / purge (gated by MATCHED)
│   │   └── workers/
│   │       └── event_consumer.py               # change-stream worker (at-least-once, DEAD re-drive)
│   ├── connectors/                             # source detection adapters
│   │   ├── sharepoint_poller.py                #   Graph change-token poller (fallback)
│   │   ├── sftp_watcher.py                     #   internal SFTP watcher + "file stable" check
│   │   └── volume_watcher.py                   #   UC volume diff watcher
│   └── tests/
├── frontend/                                   # React portal
│   └── src/
│       ├── upload/                             # single/multi-file upload, resumable
│       ├── ops/                                # DEAD-event re-drive, discrepancies, resilience KPI
│       └── admin/                              # rules / schemas / routing config UI
├── databricks/                                 # Databricks-side components
│   ├── ingestion/                              # Auto Loader bronze/silver jobs
│   └── pushback/                               # ingestion-results pushback (foreachBatch → FastAPI)
└── deploy/                                     # CI/CD, docker-compose, infra-as-code
```

Directory skeletons are intentionally not pre-created — Git does not track empty folders; each directory materializes with its first committed file.

## Tech stack

React.js (portal) • FastAPI / Python (framework) • MongoDB + GridFS (files + 20-collection metadata model) • Databricks Unity Catalog volumes, Auto Loader, bronze/silver/gold • PII masking via Databricks governed tagging policies (silver/gold)
