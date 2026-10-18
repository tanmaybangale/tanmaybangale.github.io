---
title: "Why Your LLM Can’t Read Spreadsheets (And How an In-Memory DuckDB Agent Fixes It)"
description: "Stuffing a 50k-row sheet into a 1M-token prompt burns ~670k tokens and hallucinates sums. Sending a 90-token schema and running the generated SQL in DuckDB finishes in 8.7 ms with exact arithmetic."
tags: [agents, duckdb, llm, sql]
featured: false
---

> **TL;DR:** A 1M-token context window is not a database. Stuffing raw CSV rows into a prompt causes attention dilution, structural drift on multi-table sheets, and stochastic arithmetic. Split the job: let the LLM translate intent + schema (<150 tokens) into SQL, and let an in-process DuckDB instance execute the math in milliseconds. Full ADK implementation: [`google_sheets_analyst_agent`](https://github.com/tanmaybangale/google_sheets_analyst_agent).

---

## 1. The In-Context Tabular Anti-Pattern

When teams wire Google Sheets or Excel into an AI assistant, the default pipeline is to serialize the file into text and drop it into the prompt:

```text
[Google Sheet / CSV] ──► [Text Serializer] ──► [250k-Token Prompt] ──► [LLM] ──► [Guessed Answer]
```

This works on a 15-row demo and breaks on production sheets (50,000+ rows, multi-tab workbooks, side-by-side summary tables) for three reasons:

1. **Attention dilution.** In scaled dot-product attention,
   $$
   \text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{QK^T}{\sqrt{d_k}}\right)V
   $$
   spreadsheet cells are repetitive numbers, status flags, and IDs with almost no semantic redundancy. Across hundreds of thousands of tokens, softmax weights flatten and rows in the middle get skipped.
2. **Next-token prediction $$\neq$$ an ALU.** Sampling from $$P(w_t \mid w_{<t})$$ does not execute IEEE 754 floating-point addition. On a 10,000-row `SUM`, the model emits a number that *looks plausible*, not the sum.
3. **Messy layout.** Real sheets have title banners above the header row, multiple tables separated by blank rows/columns on a single tab, and mixed casing (`Powered Off`, `poweredOff`, `POWERED_OFF`).

---

## 2. The Decoupled Pattern: LLM for Intent, DuckDB for Math

Instead of sending the *data* to the model, send only the *schema* and let the model write DuckDB SQL:

```text
[User Question] + [Compact Schema (~100 tokens)]
                       │
                       ▼
           [LLM (Gemini 2.5 Flash)]
                       │  ANSI / DuckDB SQL
                       ▼
        [In-Memory DuckDB (:memory:)] ◄── [Sheet / CSV / Parquet]
                       │
                       ▼
     [Exact DataFrame / Summary / Chart Artifact]
```

In [`google_sheets_analyst_agent`](https://github.com/tanmaybangale/google_sheets_analyst_agent) (built with the Google Agent Development Kit), the runtime is a ~300-line `SheetDataEngine` around `duckdb.connect(':memory:')` with four production guardrails:

### Dual-path ingestion (`<=15 MB` vs. `>15 MB`)
Small sheets get parsed with `calamine` (Rust Excel reader) or Pandas so we can detect and split sub-tables; large CSVs skip Pandas entirely and bind a zero-copy DuckDB view straight over the file:

```python
class SheetDataEngine:
    def __init__(self):
        self.con = duckdb.connect(":memory:")
        self.con.execute("SET memory_limit = '256MB'")
        self.con.execute("SET threads = 2")

    def register_csv(self, view_name: str, csv_path: str, size_mb: float):
        if size_mb <= 15.0:
            raw = pd.read_csv(csv_path, header=None)
            for idx, sub_df in enumerate(self._detect_and_split_tables(raw)):
                clean = self._promote_header_row(sub_df)
                name = f"{view_name}_t{idx}" if idx else view_name
                self.con.register(name, clean)          # zero-copy DataFrame view
                self.con.execute(
                    f"COPY {name} TO '/tmp/rehydrate_{name}.parquet' (FORMAT PARQUET)"
                )
        else:
            # >15 MB: bypass Pandas, scan CSV directly in DuckDB
            self.con.execute(
                f"CREATE OR REPLACE VIEW {view_name} AS "
                f"SELECT * FROM read_csv_auto('{csv_path}', sample_size=1000)"
            )
```

Persisting each view to `/tmp/rehydrate_*.parquet` means that if a serverless container resets between turns, `_rehydrate_parquet_views()` rebuilds the catalog in milliseconds without re-downloading from Drive.

### Multi-table splitting on a single tab
Finance sheets frequently stack or place two summary tables on one tab separated by blank rows or columns. `_detect_and_split_tables()` finds contiguous non-null blocks and registers them as `file_xyz_Sheet1_t0`, `_t1`, etc. (with a fast-path exit above 2,000 rows so massive tables don't pay the scan cost).

### Gap-mapped `ILIKE` for dirty status strings
Enterprise sheets never have clean enums. A column like `Powerstate` will contain `"Powered Off"`, `"poweredOff"`, and `"POWERED_OFF"` in the same file. The agent prompt enforces gap-mapped wildcards:

```sql
-- Misses "poweredOff" and "POWERED_OFF":
SELECT * FROM sheet WHERE LOWER(Powerstate) = 'powered off';

-- Matches all three:
SELECT * FROM sheet WHERE Powerstate ILIKE '%power%off%';
```

### Bounded self-healing SQL loop
If DuckDB raises a binder or syntax error, `execute_sql` returns `"SQL Error: ..."` as the tool observation and the agent retries against the schema (capped at 3 attempts).

---

## 3. Benchmark: Context Stuffing vs. In-Memory DuckDB

You can reproduce the core tradeoff locally in 35 lines of Python. We generate synthetic VM inventory sheets from 1,000 to 100,000 rows with mixed-case power states (`Powered Off`, `poweredOff`) and ask: *"By department, how many VMs are powered off and what is the idle monthly spend?"*

```python
import os, random, tempfile, time, duckdb, pandas as pd

random.seed(42)
zones = ["asia-south1-a", "asia-southeast1-b", "us-central1-a", "europe-west1-b"]
states = ["Powered On", "poweredOn", "POWERED_ON", "Powered Off", "poweredOff", "TERMINATED"]
depts = ["payments", "logistics", "catalog", "ads", "core-infra"]

for n in [1_000, 10_000, 50_000, 100_000]:
    df = pd.DataFrame([{
        "vm_id": f"vm-{i:06d}",
        "zone": random.choice(zones),
        "dept": random.choice(depts),
        "vcpu": random.choice([2, 4, 8, 16, 32]),
        "monthly_cost_usd": round(random.uniform(18.5, 940.0), 2),
        "Powerstate": random.choices(states, weights=[35, 15, 10, 12, 8, 20])[0],
    } for i in range(n)])

    csv_path = os.path.join(tempfile.gettempdir(), f"vms_{n}.csv")
    df.to_csv(csv_path, index=False)
    raw_tokens = os.path.getsize(csv_path) // 4  # ~4 bytes/token

    con = duckdb.connect(":memory:")
    t0 = time.perf_counter()
    con.register("sheet", df)
    res = con.execute("""
        SELECT dept, COUNT(*) AS off_vms, ROUND(SUM(monthly_cost_usd), 2) AS idle_usd
        FROM sheet
        WHERE Powerstate ILIKE '%power%off%'
        GROUP BY dept ORDER BY idle_usd DESC
    """).df()
    duck_ms = (time.perf_counter() - t0) * 1000

    naive = con.execute(
        "SELECT COUNT(*) FROM sheet WHERE LOWER(Powerstate) = 'powered off'"
    ).fetchone()[0]
    print(f"{n:>7,} rows | raw≈{raw_tokens:>9,} tok | duckdb={duck_ms:>5.2f} ms "
          f"| exact={naive:,} vs gap_ILIKE={int(res['off_vms'].sum()):,}")
```

### Results (Apple M-series, DuckDB 1.4.4, Python 3.9)

| Rows | CSV Size | Raw Prompt Tokens (stuffed) | Schema Prompt Tokens (DuckDB agent) | DuckDB Register + SQL Time | Exact `='powered off'` | Gap-Mapped `ILIKE '%power%off%'` (ground truth) |
|---:|---:|---:|---:|---:|---:|---:|
| 1,000 | 52.7 KB | ~13,490 | ~90 | **5.5 ms** | 118 (misses 45%) | **216 (100%)** |
| 10,000 | 525.5 KB | ~134,520 | ~90 | **3.9 ms** | 1,189 (misses 42%) | **2,038 (100%)** |
| 50,000 | 2.63 MB | ~672,990 | ~90 | **8.7 ms** | 5,952 (misses 40%) | **9,926 (100%)** |
| 100,000 | 5.26 MB | ~1,345,780 *(exceeds 1M window)* | ~90 | **11.6 ms** | 12,184 (misses 40%) | **20,165 (100%)** |

Three takeaways from the numbers:

1. **Prompt size stays flat ($$O(1)$$) instead of $$O(N)$$.** Whether the sheet has 1,000 rows or 100,000 rows, the LLM only ever sees `Table: sheet | Columns: vm_id, zone, dept, vcpu, monthly_cost_usd, Powerstate` (~90 tokens). At 100k rows, raw context stuffing already overflows a 1M-token window.
2. **Execution takes `<12 ms`.** Even adding a ~600–900 ms Gemini Flash call to generate the SQL, end-to-end latency is ~1 second versus 20–35+ seconds of prefill on a 600k-token prompt—and the arithmetic is exact IEEE 754 floating point.
3. **Query normalization matters as much as the engine.** Naive `LOWER(col) = 'powered off'` silently drops 40% of matching rows (`poweredOff`). Teaching the prompt to emit `%word1%word2%` patterns recovers 100% recall without pre-cleaning every cell.

---

## 4. Checklist When You Build One

- **Stream tabs as CSV via the Sheets v4 export URL** (`/export?format=csv&gid=...`) instead of downloading whole multi-tab `.xlsx` workbooks when a user links a specific `#gid=`.
- **Promote the first mostly-string row as the header** when row 0 is a title banner or blank, and strip characters that force SQL quoting.
- **Cap memory and threads inside serverless containers** (`SET memory_limit = '256MB'; SET threads = 2`) so a single noisy query cannot OOM-kill the worker.
- **Route by result size:** inline `<10` rows into the chat response; for larger outputs, persist the result DataFrame as an artifact CSV and hand it to a code-execution sub-agent only when the user asks for a plot.

The full code—OAuth2 Drive fetching, multi-table splitter, Parquet rehydration, and ADK tool bindings—is in [`tanmaybangale/google_sheets_analyst_agent`](https://github.com/tanmaybangale/google_sheets_analyst_agent).
