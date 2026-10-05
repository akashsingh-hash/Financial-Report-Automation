# Financial Report Automation & Intelligence Studio

> 📖 **Complete Documentation & Interview Prep Guide**: See [docs/00_index.md](docs/00_index.md) or the compiled [PROJECT_DOCUMENTATION.md](docs/PROJECT_DOCUMENTATION.md) for full architectural guides, 70+ interview questions, and beginner walkthroughs.

An end-to-end automated month-end financial reporting and RAG-powered analytics system built with **Python, pandas, openpyxl, SQLAlchemy, FastAPI, LangChain, Hugging Face, and React + Vite**.

---

## 🎨 Design System & Color Palette

The interface is styled in a custom pine/sage forest theme:

| Token | Hex | Usage |
|---|---|---|
| `--bg-deep` | `#051F20` | Full-page background |
| `--bg-card` | `#0B2B26` | Cards, chat panel |
| `--surface-border` | `#163832` | Borders, table headers |
| `--btn-primary` | `#235347` | Buttons, badges, chart fill |
| `--accent-sage` | `#8EB69B` | Muted text, labels |
| `--accent-mint` | `#DAF1DE` | High-contrast headings, KPI values |

---

## 🏗️ System Architecture

```
fin-report/
├── .env                  # Secrets & model config  ← git-ignored
├── .env.example          # Environment template
├── .gitignore
├── README.md
├── backend/
│   ├── requirements.txt
│   ├── models.py         # Pydantic schemas (TransactionItem, BudgetItem, MonthlyFinancialReport)
│   ├── seed.py           # Sample data generator (CSVs + SQLite)
│   ├── report.py         # Ingestion, summarize(), build_excel(), CLI
│   ├── rag.py            # FAISS + BM25 hybrid retrieval, cross-encoder reranking, LLM generation
│   ├── main.py           # FastAPI server — /api/summary, /api/report, /api/ask
│   ├── validate.py       # Legacy acceptance validation script (Phases 1–4)
│   ├── tests/
│   │   ├── __init__.py
│   │   ├── test_report.py   # Original Phase 2 unit tests
│   │   └── test_all.py      # Phase 7 full suite (31 tests)
│   ├── data/             # transactions.csv, budgets.csv, finance.db
│   └── output/           # Generated Excel reports
└── frontend/
    ├── package.json
    ├── vite.config.js    # Proxies /api → http://localhost:8000
    ├── index.html
    └── src/
        ├── index.css     # Complete pine/sage design system
        ├── main.jsx
        └── App.jsx       # Month picker, KPI cards, tables, Excel download, RAG chat
```

### Data Flow

```
CSV / SQLite ──► load_transactions() ──► summarize()
                                              │
                                    ┌─────────┴────────┐
                                    ▼                  ▼
                             build_excel()    chunk_financial_data()
                                 .xlsx               │
                                                 build_index()
                                               (FAISS + BM25)
                                                      │
                                            hybrid_retrieve() + reranker
                                                      │
                                           Qwen/Qwen2.5-72B-Instruct
                                                      │
                                              answer + [sources]
                                                      │
                        React UI ◄─── FastAPI /api/ask, /api/summary, /api/report
```

---

## 🚀 Getting Started

### Prerequisites

```bash
pip install -r backend/requirements.txt
```

### 1. Configure Environment

Copy `.env.example` to `.env` and fill in your key:

```bash
cp .env.example .env
# set HF_TOKEN=hf_your_huggingface_token_here
```

### 2. Seed Sample Data

```bash
python backend/seed.py
# Creates: backend/data/transactions.csv
#          backend/data/budgets.csv
#          backend/data/finance.db
```

### 3. Start the API Server

```bash
uvicorn backend.main:app --reload --port 8000
```

Interactive docs at **http://127.0.0.1:8000/docs**

### 4. Run the React Dashboard

```bash
cd frontend
npm install
npm run dev
# Opens at http://localhost:3000
```

> Alternatively, build the frontend (`npm run build`) and browse `http://127.0.0.1:8000/` — FastAPI serves the compiled SPA directly.

### 5. Generate Excel Reports via CLI

```bash
python backend/report.py --month 2026-09
# Output: backend/output/report_2026-09.xlsx
```

---

## 🔌 API Reference

All endpoints are fully documented at `/docs` (Swagger UI).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/summary?month=YYYY-MM` | Category-level budget vs actual breakdown |
| `GET` | `/api/report?month=YYYY-MM` | Download formatted `.xlsx` report |
| `POST` | `/api/ask` | RAG-powered natural language Q&A |
| `GET` | `/api/months` | List all distinct months in the dataset |
| `GET` | `/api/transactions?month=YYYY-MM` | Raw transaction list |

### POST /api/ask — Request body

```json
{
  "question": "Which categories exceeded their budget?",
  "month": "2026-09"
}
```

### Response

```json
{
  "answer": "Groceries ($650 actual vs $600 budget) and Dining ($450 vs $400) exceeded their budgets. [1][2]",
  "sources": [
    "[1] (Type: summary, Category: Groceries)\nBudget: $600 | Actual: $650 | Status: Over budget",
    "[2] (Type: summary, Category: Dining)\nBudget: $400 | Actual: $450 | Status: Over budget"
  ],
  "month": "2026-09"
}
```

### Error responses

| Code | When |
|------|------|
| 404 | Month has no transaction data |
| 422 | Invalid request shape / bad month format |
| 500 | `HF_TOKEN` missing or unexpected server error |

---

## 🧪 Running Tests

```bash
pytest backend/tests/ -v
```

**31 tests** across 6 classes:

| Class | Coverage |
|---|---|
| `TestValidateColumns` | Column validation guard |
| `TestSummarize` | All edge cases: over-budget, zero-budget, empty month, None tx |
| `TestBuildExcel` | Sheets exist, live formulas (D4, E4, F4), conditional-formatting rules ≥ 2, frozen pane, SUM TOTAL row |
| `TestHybridRetrieve` | FAISS+BM25 results, relevant doc retrieval, chunk metadata/headers |
| `TestAPIEndpoints` | 200/404/422/500 for all three spec endpoints + root HTML |
| `TestDataLoading` | CSV month filter, SQL month filter, missing file, bad columns |

---

## ✨ Key Features

- **Multi-source Ingestion**: CSV files or SQLite/PostgreSQL databases with schema validation and `YYYY-MM` month filtering.
- **Dynamic Excel Engine**: Live formulas (`=B-C`, `IF(B>0,…,…)`, `SUM`), conditional formatting (red for overruns, green for on-track), embedded clustered bar chart, frozen header row.
- **Hybrid RAG Pipeline**: FAISS semantic search + BM25Okapi keyword search, fused via Reciprocal Rank Fusion, reranked with a Cross-Encoder, answered by `Qwen/Qwen2.5-72B-Instruct`.
- **Smart Caching**: RAG index is rebuilt only when the underlying data file changes (SHA-256 hash comparison).
- **React Dashboard**: Month picker, 4 KPI cards, category summary table with red/green rows and spend progress bars, transactions tab with live search, one-click Excel download, AI chat with collapsible source citations and suggested question chips.
- **Robust API**: 404 for empty months, 500 with clear message for missing token, 422 for bad request format, structured `logging` throughout.

---

## 💬 Sample Questions for the RAG Chat

- *Which categories went over budget this month?*
- *What is the total variance between budget and actual?*
- *Give me an executive summary of September 2026 finances.*
- *Where can I cut spending to stay under budget next month?*
- *What was the highest single transaction in Dining?*
- *How does actual spending compare to the budget for each category?*

---

## 🔧 Configuration (.env)

```dotenv
HF_TOKEN=hf_your_token_here
LLM_MODEL=Qwen/Qwen2.5-72B-Instruct      # change if model is unavailable
EMBED_MODEL=BAAI/bge-small-en-v1.5
RERANK_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
DATA_SOURCE=backend/data/transactions.csv  # or sqlite:///backend/data/finance.db
BUDGET_FILE=backend/data/budgets.csv
```

> **Note:** `Qwen/Qwen2.5-7B-Instruct` is **not** available on the free HF serverless inference tier. Use `Qwen/Qwen2.5-72B-Instruct` (confirmed working).
