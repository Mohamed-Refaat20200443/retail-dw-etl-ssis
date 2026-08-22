# Retail Data Warehouse ETL Pipeline (SSIS)

Enterprise-grade ETL pipeline built with **SQL Server Integration Services (SSIS)** that transforms the raw `ContosoRetailDW` source database into a production-ready **Star Schema Data Warehouse**, processing over **20 million records** across 4 fact tables and 10 dimension tables — with full validation, business transformation, auditing, and incremental load support.

---

## 📐 Architecture

```
Source (ContosoRetailDW)
        │
        ▼
   [ Staging ]              Raw 1:1 copy, no transformation — stable rollback baseline
        │
        ▼
   [ Validation ]           3-tier check: Mandatory Fields → Numeric Rules → Business Rules
        │           ┌──────────────┴──────────────┐
        ▼           ▼                              ▼
     Valid       InValid (business rejects)   Pipeline Errors (technical, logged separately)
        │
        ▼
   [ Transformation ]       Business metrics, classification, standardization, audit columns
        │
        ▼
   [ dw Schema ]            Star Schema — real Primary/Foreign Key relationships
        │
        ▼
   [ Incremental Load ]     Lookup (Dims) + MERGE (Facts) — detects new/changed records only
```

---

## 🗂️ Data Warehouse Schema

| Layer | Count | Tables |
|---|---|---|
| **Dimensions** | 10 | DimGeography, DimCurrency, DimDate, DimProductCategory, DimProductSubcategory, DimProduct, DimStore, DimCustomer, DimChannel, DimPromotion |
| **Facts** | 4 | FactOnlineSales (12.5M rows), FactSales (3.4M rows), FactInventory (8M rows), FactExchangeRate (~800 rows) |

All tables enforce real `PRIMARY KEY` / `FOREIGN KEY` constraints — dimensions are loaded before facts to satisfy referential integrity.

### Audit Layer

Every table in every layer (staging → validation → transformation → dw) carries four standard audit columns:

| Column | Purpose |
|---|---|
| `LoadDate` | Timestamp of the load |
| `PackageName` | SSIS package that loaded the row |
| `ETLUser` | Execution user/service account |
| `BatchID` | **Single GUID per package run** (not per row) — every record loaded in the same execution shares the same BatchID, enabling precise rollback and full traceability of *what* was loaded, *when*, and *by which run* |

---

## ✅ Validation Strategy

Instead of assuming rules theoretically, validation rules were **derived from real data profiling** (NULL% analysis across all 14 tables) rather than guesswork.

**3-Tier Validation (Conditional Split in SSIS):**

1. **Mandatory Fields** — required business keys/attributes must not be NULL or empty
2. **Numeric Rules** — no negative values where they are logically invalid (income, quantities, counts)
3. **Business Rules** — logical consistency checks (e.g. dates not in the future, part-values not exceeding their totals)

Rows failing validation are redirected to a dedicated `InValid` table (not silently dropped), so every rejection is auditable and reviewable — which is exactly how a data-quality bug in one of the business-rule conditions was caught and corrected during this project (see [Lessons Learned](#-real-challenges--lessons-learned)).

**Column retention decisions** were also data-driven:
- NULL % ≥ 95% → column dropped (no analytical value)
- NULL % 50–95% with clear business meaning → retained (e.g. `CloseDate`/`CloseReason` in DimStore)
- NULL % < 50% → retained as normal

---

## 🔄 Business Transformation

- **Financial metrics**: Gross Profit, Net Sales Amount, Profit Margin %, Average Unit Profit
- **Business classification**: Customer Income Segment, Customer Life Stage, Stock Status, Store Operational Status, Store Size Category, Product Price Tier, Promotion Status/Discount Tier
- **Standardization**: Proper Case text cleanup, phone number formatting, trimmed/cleaned labels (`_Clean` columns replace raw values in the final `dw` layer — the warehouse keeps only the clean, standardized version under the original column name)

---

## ⚡ Incremental Load

| Target | Technique | Why |
|---|---|---|
| **Dimensions** | Lookup Transformation (row-by-row) + Type 1 update | Dimension volumes are small; simplicity over raw throughput |
| **Facts** (12.5M+ rows) | T-SQL `MERGE` statement (set-based) | Row-by-row Lookup on large fact tables was prohibitively slow — MERGE processes all rows in a single set-based operation |

A `dw.ETL_Watermark` table tracks the last successful load timestamp per table, driving the `WHERE LoadDate > @LastWatermarkDate` filter on every incremental run.

---

## 🚧 Real Challenges & Lessons Learned

### 1. Row-by-Row vs. Set-Based Processing
Lookup Transformation (same pattern used for dimensions) was first applied to the large fact tables. It processes every row individually and took an extremely long time at 12.5M+ rows. Switching to a **T-SQL `MERGE`** statement (set-based) resolved the performance issue immediately.

### 2. Foreign Key Dependency Chains
`TRUNCATE TABLE` refuses to execute on any table referenced by a Foreign Key — **even if the referencing table is completely empty** — because SQL Server checks for the existence of the constraint itself, not the row count. The fix: either drop and re-add constraints around a Full Load, or truncate/delete in the exact reverse order of the build sequence (facts first, then dependent dimensions, then independent dimensions).

### 3. Out-of-Memory During Parallel Fact Loading
Running all 4 fact tables in parallel caused the SSIS Buffer Manager to fail allocating enough memory for the largest table. This lesson was carried forward and applied proactively when designing the Incremental Load package (sequential/controlled execution instead of unconstrained parallelism).

### 4. Hidden Data Quality Bug Caught by Referential Integrity
A Full Load run failed on `FK_FactOnlineSales_DimCustomer` — 1,431 `CustomerKey` values in the sales fact had no matching row in `DimCustomer`. Root-cause tracing (via the `InValid` validation table) showed these customers were being **incorrectly rejected** by a business-rule condition comparing `NumberChildrenAtHome > TotalChildren` — a logic flaw, not a real data quality issue. Correcting the condition and re-running the pipeline recovered all 1,431 customers with zero data loss. This is a direct example of why **Referential Integrity should be validated explicitly as part of the QA process**, not discovered only when Foreign Key constraints fail during the final load.

### 5. Transaction Log Growth on Bulk Delete
Large `DELETE` statements on multi-million row tables filled the transaction log and forced a rollback. Replacing `DELETE` with `TRUNCATE` (with constraints temporarily dropped/re-added) avoided row-by-row logging entirely.

---

## 🛠️ Tech Stack

- **SQL Server** — relational engine, T-SQL (Dynamic SQL, `MERGE`, `INFORMATION_SCHEMA` metadata queries)
- **SSIS (SQL Server Integration Services)** — Control Flow / Data Flow orchestration, Conditional Split, Lookup Transformation, OLE DB Source/Destination/Command
- **Star Schema Design** — Kimball-style dimensional modeling with enforced PK/FK constraints

---

## 📁 Repository Structure

```
├── sql/
│   ├── 01_dw_schema.sql              # Full CREATE TABLE DDL (14 tables, ordered by dependency)
│   ├── 02_incremental_load_merge.sql # MERGE statements for all 14 tables
│   └── 03_utility_queries.sql        # Validation / NULL% / referential integrity check queries
├── ssis/
│   └── Full_Load_DWH.dtsx            # Full Load package
│   └── Incremental_Load_DWH.dtsx     # Incremental Load package
├── docs/
│   └── architecture-diagram.png
└── README.md
```

---

## 🚀 Getting Started

1. Restore or attach the `ContosoRetailDW` source database in SQL Server.
2. Run `sql/01_dw_schema.sql` to create the `dw` schema and all 14 tables with constraints.
3. Open and execute `ssis/Full_Load_DWH.dtsx` for the initial full load (staging → validation → transformation → dw).
4. For subsequent runs, execute `ssis/Incremental_Load_DWH.dtsx`, which reads from `dw.ETL_Watermark` and loads only new/changed records.

---

## 📌 Notes

- All validation and column-retention decisions were derived from **actual data profiling**, not assumptions.
- The pipeline separates **technical pipeline errors** from **business-rule rejections**, so failures are diagnosable rather than opaque.
- Designed to be idempotent for Full Load (safe to re-run after truncation) and additive for Incremental Load (safe to run on a schedule).

---

## 🏷️ Topics

`ssis` `etl` `data-warehouse` `sql-server` `data-engineering` `star-schema` `t-sql`
