# Warehouse Text-to-SQL (T_S_T_Data)

A Jupyter notebook that builds a complete **Text-to-SQL pipeline** on a synthetic warehouse management system (WMS) database — entirely from scratch, no external data files needed.

---

## What This Project Does

1. **Creates a SQLite warehouse database** with 13 tables (orders, invoices, cases, inventory, work assignments, etc.)
2. **Seeds it with realistic synthetic data** (~20,000+ rows across all tables)
3. **Embeds the schema** using `sentence-transformers` and stores it in a **ChromaDB** vector store
4. **Loads a local LLM** (`Qwen2.5-3B-Instruct` in 4-bit quantization) as the SQL generator
5. **Runs a full RAG-based Text-to-SQL chain** — retrieves relevant schema chunks, builds a prompt, generates SQL, validates it, executes it, and returns a DataFrame

---

## Database Schema (Tables)

| Table | Description |
|---|---|
| `location_file` | Warehouse locations (PICK, RESERVE, DOCK, STAGE) |
| `item_file` | Product/SKU master |
| `dealer_file` | Customer/dealer master |
| `warehouse_user_file` | WMS users and roles |
| `inventory_file` | On-hand, allocated, and available quantities |
| `order_header` | Order headers with status and priority |
| `order_detail` | Order line items |
| `work_assignment_header` | Pick/putaway work assignment headers |
| `work_assignment_detail` | Individual task lines within a work assignment |
| `case_header` | Inbound case/shipment headers |
| `case_detail` | Line items within a case |
| `invoice_header` | Invoice headers linked to shipped orders |
| `invoice_detail` | Invoice line items with pricing |

---

## Requirements

### Hardware
- GPU recommended (the notebook uses `device_map="auto"` with 4-bit quantization)
- Tested on Google Colab with a T4 GPU

### Python Packages

```
sentence-transformers
chromadb
faiss-cpu
transformers>=4.40
accelerate
bitsandbytes
pandas
sqlite3  # built-in
```

Install all at once:

```bash
pip install sentence-transformers chromadb faiss-cpu "transformers>=4.40" accelerate bitsandbytes pandas
```

---

## How to Run

### Option A — Google Colab (Recommended)

1. Upload `T_S_T_Data.ipynb` to [colab.research.google.com](https://colab.research.google.com)
2. Set runtime to **GPU** → Runtime → Change runtime type → T4 GPU
3. Run all cells top to bottom

### Option B — Local Jupyter

```bash
# 1. Clone the repo
git clone https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
cd YOUR_REPO_NAME

# 2. Install dependencies
pip install sentence-transformers chromadb faiss-cpu "transformers>=4.40" accelerate bitsandbytes pandas jupyter

# 3. Launch Jupyter
jupyter notebook T_S_T_Data.ipynb
```

---

## Files in This Repository

```
T_S_T_Data.ipynb   ← The only file you need
README.md          ← This file
```

> **No input data files are required.** The notebook generates the SQLite database (`warehouse.db`), all synthetic data, and the ChromaDB vector index (`./chroma_wms_free/`) at runtime.

---

## Generated Files (at Runtime)

These are created when you run the notebook — do **not** need to be committed to Git:

| File / Folder | Description |
|---|---|
| `warehouse.db` | SQLite database with all tables and data |
| `./chroma_wms_free/` | ChromaDB vector store with schema embeddings |

Add these to your `.gitignore`:

```
warehouse.db
chroma_wms_free/
```

---

## Example Query

```python
result = text_to_sql("How is CASE-0001 reconciled?")
print(result["sql"])
result["df"]
```

**Generated SQL:**
```sql
SELECT * FROM case_header
WHERE case_id = 'CASE-0001'
AND reconciliation_date IS NOT NULL
```

---

## Notes

- The LLM (`Qwen2.5-3B-Instruct`) is downloaded automatically from Hugging Face on first run (~3B parameters, ~2GB in 4-bit)
- Only `SELECT` queries are allowed — the pipeline validates and rejects any destructive SQL
- A one-retry repair loop is included: if the generated SQL fails, the error is fed back to the LLM for a fix
- `random.seed(42)` is used, so synthetic data is fully reproducible

---

## License

MIT
