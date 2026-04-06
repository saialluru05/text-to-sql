# Warehouse Text-to-SQL Pipeline

A fully self-contained Jupyter notebook that builds a **RAG-based (Retrieval-Augmented Generation) Text-to-SQL system** on top of a synthetic Warehouse Management System (WMS) database.

Ask questions in plain English — the pipeline retrieves the relevant schema, generates a safe SQLite `SELECT` query using a local LLM, validates it, and returns results as a Pandas DataFrame.

> **No input files needed.** The database, synthetic data, and vector index are all created at runtime.

---

## How It Works

```
Natural language question
        │
        ▼
 [1] Embed question  ──▶  ChromaDB semantic search  ──▶  Top-k schema chunks
        │
        ▼
 [2] Build prompt  (schema context + question + rules)
        │
        ▼
 [3] Qwen2.5-3B-Instruct  ──▶  Raw LLM output  ──▶  Extract SQL
        │
        ▼
 [4] Validate SQL  (SELECT/WITH only, no destructive keywords)
        │
        ▼
 [5] Execute on SQLite  ──▶  Pandas DataFrame
        │
        └──▶  On error: one auto-repair attempt (error fed back to LLM)
```

---

## Notebook Structure

| Cell | Purpose |
|------|---------|
| **Cell 0** | Install and pin all dependencies |
| **Cell 1** | Global configuration — paths, models, data volumes, pipeline knobs |
| **Cell 2** | Define WMS schema DDL + `create_schema()` — builds `warehouse.db` |
| **Cell 3** | Synthetic data generators + `seed_database()` — populates all 13 tables |
| **Cell 4** | `preview_table()` — quick DataFrame preview of any table |
| **Cell 5** | `build_vector_store()` — parses DDL, embeds with `sentence-transformers`, stores in ChromaDB |
| **Cell 6** | `load_llm()` — loads Qwen2.5-3B-Instruct in 4-bit (falls back to float16) |
| **Cell 7** | Full Text-to-SQL pipeline: `retrieve_schema`, `build_prompt`, `generate_sql`, `validate_sql`, `run_sql`, `text_to_sql` |
| **Cells 8–10** | Sample queries — valid lookups and a blocked destructive query demo |

---

## Database Schema

The notebook creates a SQLite WMS database (`warehouse.db`) with 13 tables:

| Table | Description |
|-------|-------------|
| `location_file` | Warehouse locations — PICK, RESERVE, DOCK, STAGE zones |
| `item_file` | Product / SKU master with UOM and case pack details |
| `dealer_file` | Customer / dealer master with address info |
| `warehouse_user_file` | WMS users with roles: OPERATOR, SUPERVISOR, ADMIN |
| `inventory_file` | On-hand, allocated, and available quantities per location/item |
| `order_header` | Order header — status: OPEN, ALLOCATED, PICKED, SHIPPED, CANCELLED |
| `order_detail` | Order line items with ordered, allocated, picked, shipped quantities |
| `work_assignment_header` | Pick / putaway work assignment headers |
| `work_assignment_detail` | Individual task lines within a work assignment |
| `case_header` | Inbound case / shipment headers with reconciliation tracking |
| `case_detail` | Line items within a case with receive quantities |
| `invoice_header` | Invoice headers linked to shipped orders |
| `invoice_detail` | Invoice line items with unit price and extended amount |

### Approximate Row Counts (default config)

| Table | Rows |
|-------|------|
| `location_file` | 120 |
| `item_file` | 250 |
| `dealer_file` | 80 |
| `warehouse_user_file` | 30 |
| `inventory_file` | ~20,000 |
| `case_header` | 300 |
| `case_detail` | ~1,000 |
| `order_header` | 600 |
| `order_detail` | ~2,600 |
| `work_assignment_header` | ~585 |
| `work_assignment_detail` | ~2,500 |
| `invoice_header` | ~100 |
| `invoice_detail` | ~450 |

---

## Requirements

### Hardware
- **GPU strongly recommended** — the notebook runs `Qwen2.5-3B-Instruct` in 4-bit quantization (~2 GB VRAM)
- Tested on **Google Colab T4 GPU**
- CPU-only will work but inference will be very slow

### Python Packages

All installed automatically in Cell 0 with pinned versions:

```
transformers==4.47.1
huggingface_hub==0.26.5
tokenizers>=0.20
sentence-transformers
chromadb
accelerate
bitsandbytes
pandas
```

> **Why pinned?** `huggingface_hub >= 0.27` introduced a `@strict` dataclass decorator that breaks `Qwen2Config` initialisation in older `transformers` builds. The pins in Cell 0 prevent this `StrictDataclassDefinitionError`.

---

## Quickstart

### Option A — Google Colab (recommended)

1. Upload `Text_SQL.ipynb` to [colab.research.google.com](https://colab.research.google.com)
2. Set runtime to **GPU**: Runtime → Change runtime type → T4 GPU
3. Run All: Runtime → Run all

### Option B — Local Jupyter

```bash
# 1. Clone the repo
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO

# 2. Install dependencies (Python 3.10+ required)
pip install "transformers==4.47.1" "huggingface_hub==0.26.5" \
    "tokenizers>=0.20" sentence-transformers chromadb \
    accelerate bitsandbytes pandas jupyter

# 3. Launch
jupyter notebook Text_SQL.ipynb
```

---

## Configuration

All settings are in **Cell 1** — the single place to change anything:

```python
# Paths
DB_PATH         = "warehouse.db"
CHROMA_PATH     = "./chroma_wms"
COLLECTION_NAME = "wms_schema"

# Models
EMBED_MODEL = "sentence-transformers/all-MiniLM-L6-v2"
LLM_NAME    = "Qwen/Qwen2.5-3B-Instruct"

# Pipeline
SCHEMA_RETRIEVE_K = 4    # schema chunks retrieved per query
MAX_NEW_TOKENS    = 160  # max LLM output tokens
```

### Changing Data Volume

Edit `DATA_CFG` in Cell 1:

```python
DATA_CFG = {
    "n_locations": 120,   # total warehouse locations
    "n_items":     250,   # total SKUs
    "n_dealers":   80,    # total dealers
    "n_users":     30,    # total WMS users
    "n_cases":     300,   # inbound cases
    "n_orders":    600,   # outbound orders
    "seed":        42,    # change for different random data
}
```

---

## Running Queries

Use `text_to_sql()` from Cell 7 onwards:

```python
result = text_to_sql("How is CASE-0001 reconciled?")

print(result["status"])   # OK | OK_AFTER_RETRY | REJECTED | ERROR
print(result["sql"])      # the generated SQL
result["df"]              # Pandas DataFrame with results
```

### Example Queries

```python
# Case reconciliation lookup
text_to_sql("How is CASE-0001 reconciled?")

# Inventory analysis
text_to_sql("Which items have the most inventory on hand?")

# Order reporting
text_to_sql("Show me all shipped orders with their dealer names.")

# Invoice totals
text_to_sql("What is the total invoice amount per dealer?")
```

### Safety Validator (demonstrated in Cell 10)

The pipeline rejects any question that would generate destructive SQL:

```python
# This is blocked automatically
text_to_sql("Delete reconciled cases from case_header")
# → Status: REJECTED | Reason: Forbidden keyword: 'DELETE'.
```

Blocked keywords: `DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`, `TRUNCATE`, `ATTACH`, `DETACH`, `PRAGMA`, `REINDEX`, `VACUUM`, `CREATE`

---

## Pipeline Return Object

`text_to_sql()` always returns a dictionary:

| Key | Type | Description |
|-----|------|-------------|
| `question` | `str` | The original natural language question |
| `sql` | `str` | The SQL query that was generated |
| `status` | `str` | `OK`, `OK_AFTER_RETRY`, `REJECTED`, or `ERROR` |
| `reason` | `str` | Human-readable explanation of the status |
| `df` | `DataFrame` or `None` | Query results (None if REJECTED or ERROR) |

---

## Files in This Repository

```
Text_SQL.ipynb    ← The notebook (only file you need)
README.md         ← This file
.gitignore        ← Excludes generated runtime files
```

### Generated at Runtime (not committed to Git)

```
warehouse.db      ← SQLite database (~5–10 MB)
chroma_wms/       ← ChromaDB vector store
```

Recommended `.gitignore`:

```
warehouse.db
chroma_wms/
__pycache__/
.ipynb_checkpoints/
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `StrictDataclassDefinitionError` on model load | Re-run Cell 0 and restart the runtime — the pinned versions fix this |
| `CUDA out of memory` | Reduce `MAX_NEW_TOKENS` in Cell 1, or use a smaller model |
| `collection already exists` error in ChromaDB | Cell 5 deletes and recreates the collection automatically on each run |
| SQL status is `ERROR` | The pipeline auto-retries once; check `result["reason"]` for the SQLite error |
| Model download is slow | Qwen2.5-3B-Instruct is ~3 GB — first run requires a stable internet connection |
| `bitsandbytes` import error locally | `load_llm()` automatically falls back to float16 if 4-bit loading fails |

---


