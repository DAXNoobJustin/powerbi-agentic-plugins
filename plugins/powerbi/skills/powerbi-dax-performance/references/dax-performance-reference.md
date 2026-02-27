# DAX Optimization Patterns Reference

A comprehensive catalog of DAX anti-patterns, optimization strategies, and trace analysis guidance.

## Table of Contents

- [Section 1: How the Engine Works](#section-1-how-the-engine-works)
  - [Query Processing Architecture](#query-processing-architecture)
  - [xmSQL: The Storage Engine Query Language](#xmsql-the-storage-engine-query-language)
  - [Segments and Parallelism](#segments-and-parallelism)
- [Section 2: Reading and Diagnosing Traces](#section-2-reading-and-diagnosing-traces)
  - [Understanding FE vs. SE Metrics](#understanding-formula-engine-fe-vs-storage-engine-se-metrics)
  - [Working with Raw Aggregate Metrics](#working-with-raw-aggregate-metrics)
  - [Analyzing Trace Events](#analyzing-trace-events)
  - [What to Look For](#what-to-look-for)
  - [xmSQL Pattern Searching](#xmsql-pattern-searching)
  - [DAX vs. Data Layout: Reading the Signal](#dax-vs-data-layout-reading-the-signal)
- [Section 3: Tier 1–2: Query Optimization](#section-3-tier-12-query-optimization)
  - [Tier 1: Core DAX Guidelines + DAX001–DAX026](#tier-1-dax-optimization-patterns)
  - [Vertical Fusion](#vertical-fusion)
  - [Horizontal Fusion](#horizontal-fusion)
  - [Tier 2: Query Structure Patterns — QRY001–QRY003](#tier-2-query-structure-patterns)
- [Section 4: Tier 3–4: Model and Data Layout](#section-4-tier-34-model-and-data-layout)
  - [General Data Layout Best Practices](#general-data-layout-best-practices)
  - [Tier 3: Model Optimization Patterns — MDL001–MDL012](#tier-3-model-optimization-patterns)
  - [Tier 4: Direct Lake Optimization — DL001–DL004](#tier-4-direct-lake-optimization-patterns)
- [Section 5: Real-World Optimization Examples](#section-5-real-world-optimization-examples)

---

## Section 1: How the Engine Works

### Query Processing Architecture

When a client (Power BI, Excel, a third-party tool, or a composite/proxy model) submits a query, it is routed to the **Analysis Services engine**, which contains the **Formula Engine (FE)**. The FE understands all supported query languages — DAX, MDX, and SQL — and is responsible for translating the query into a physical execution plan.

**The Formula Engine** is the general-purpose component:
- Handles any query, including branching logic, complex arithmetic, context transitions, and all DAX functions
- Single-threaded — it processes work serially
- May need to decompress or materialize data before operating on it
- Slow relative to the SE; it is the bottleneck in most poorly written queries

**The Storage Engine** is the high-performance data retrieval component:
- Multi-threaded and very fast — it operates directly on compressed columnar data in VertiPaq
- Supports only a limited set of operations: the four basic arithmetic operators, group by, joins, and basic aggregations (SUM, COUNT, MIN, MAX, DISTINCTCOUNT)
- Cannot evaluate DAX functions, branching logic, measure references, or context transitions

**For Direct Query models**, the SE role is played by the underlying data source (SQL, Spark, etc.). The FE generates SQL and pushes it to the source. Modern DQ sources can absorb more logic into the pushed SQL — reducing how much the FE must process locally. The trade-off is network and source latency instead of in-memory scan cost.

**The FE↔SE interaction and why it matters:**

To execute a query, the FE builds a plan and requests data from the SE in one or more scans (called **datacaches**). Each datacache is the result of one SE scan — a set of columns and aggregated values returned to the FE. The FE then uses these datacaches to evaluate the remaining logic.

Some queries require multiple datacaches: the FE may need one scan to build a filter set (e.g., which customers match a condition), then a second scan to aggregate the fact table using that filter. When the SE cannot satisfy a filter or expression natively, it **calls back** to the FE row-by-row to evaluate each value — effectively making the SE single-threaded for that operation.

This back-and-forth is expensive. The core principle of DAX optimization is: **push as much work as possible into the SE, minimize the number of SE scans, and eliminate FE callbacks entirely.**

---

### xmSQL: The Storage Engine Query Language

The Storage Engine processes data from the VertiPaq columnar store and exposes its scan activity as a human-readable representation in trace events — commonly called xmSQL. These representations show exactly what the engine is scanning: which tables, which columns are aggregated, which filters are applied, and how joins are resolved. Reading xmSQL is the primary tool for understanding and optimizing SE behavior. The syntax resembles SQL but has several critical differences.

**Implicit GROUP BY:** Every column in the SELECT list is automatically a grouping column. There is no explicit GROUP BY keyword.
```
-- xmSQL: Category and CalendarYear both group the result
SELECT Product[Category], Date[CalendarYear], SUM ( Sales[SalesAmount] )
FROM Sales
    LEFT OUTER JOIN Product ON Sales[ProductKey]   = Product[ProductKey]
    LEFT OUTER JOIN Date    ON Sales[OrderDateKey] = Date[DateKey]
```

**Computed expressions:** Row-level calculations are declared in a `WITH` block using `:=` and referenced inside aggregations with `@`:
```
WITH $Expr0 := ( Sales[UnitPrice] * Sales[OrderQuantity] )
SELECT Product[Category], SUM ( @$Expr0 )
FROM Sales
    LEFT OUTER JOIN Product ON Sales[ProductKey] = Product[ProductKey]
```

**Joins always LEFT OUTER:** Every model relationship is rendered as a LEFT OUTER JOIN in xmSQL — the many-side table is the FROM table and the one-side is joined in:
```
SELECT Customer[Country], SUM ( Sales[SalesAmount] )
FROM Sales
    LEFT OUTER JOIN Customer ON Sales[CustomerKey] = Customer[CustomerKey]
```

**Multi-step batches via DEFINE TABLE:** When the FE cannot resolve a filter in a single SE scan, it uses temporary tables. The most common case is a two-step semi-join: one query builds an index of matching keys, and a second query filters the fact against it:
```
DEFINE TABLE $Filter0 :=
    SELECT SIMPLEINDEXN ( Customer[CustomerKey] )
    FROM Customer
    WHERE Customer[Country] = 'Canada'

SELECT SUM ( Sales[SalesAmount] )
FROM Sales
WHERE Sales[CustomerKey] ININDEX $Filter0[$SemijoinProjection]
```
This two-step pattern adds latency. When you see it in traces, check whether the relationship or DAX structure can be reorganized to collapse the two queries into one.

**Callbacks (performance red flags):** A callback occurs when the SE cannot evaluate an expression natively and must return each row to the (single-threaded) FE for evaluation — effectively making the SE single-threaded for that operation. The most common type is **CallbackDataID**, triggered by measure references, IF/SWITCH logic, or context transitions inside SE scans. **EncodeCallback** appears when grouping by a computed expression rather than a physical column.

**Eliminating callbacks:** Replace measure references with column expressions, use `INT()` instead of `IF(..., 1, 0)`, cache context-independent values in variables before the iterator, or pre-materialize with SUMMARIZECOLUMNS.

---

### Segments and Parallelism

VertiPaq stores each column in fixed-size chunks called **segments**. Segments are the fundamental unit of both compression and parallel execution — the SE assigns one CPU thread per segment when scanning. More segments on a table means more threads can work simultaneously.

**Default segment sizes by engine:**
- Power BI Import: ~1 million rows per segment (automatic)
- Analysis Services: ~8 million rows per segment (automatic)
- Direct Lake: determined by Delta rowgroup size — one Delta rowgroup maps to one VertiPaq segment

**Why segment count drives parallelism:**

On a machine with 16 available threads, a 32M-row table stored in 2 segments runs with a maximum degree of parallelism of 2 — 14 threads sit idle. The same table stored in 32 segments (1M rows each) can use all 16 threads simultaneously. This difference can be a 4–8× speedup on large scans with zero DAX changes.

**For Direct Lake models**, Delta rowgroups map directly to VertiPaq segments, and you control both via how the Delta table is written. See DL001 for V-ordering and DL002 for rowgroup sizing guidance.

**Diagnosing low parallelism in traces:**

The **SE Parallelism Factor** (StorageEngineCpuTime ÷ StorageEngineDuration) shows effective thread utilization during SE scans. Values near 1.0 mean single-threaded execution; values of 8–16 indicate strong multi-core utilization.

When a trace shows very few SE queries (1–4), high SE Duration on those queries, and a Parallelism Factor close to 1.0 — with clean xmSQL (no callbacks, no ININDEX, simple aggregation) — the bottleneck is almost certainly **too few segments** in the data. This cannot be fixed with DAX. The path forward is data layout: increase rowgroup count, run OPTIMIZE with V-ORDER, or adjust Spark write settings before the next model refresh.

---

## Section 2: Reading and Diagnosing Traces

### Understanding Formula Engine (FE) vs. Storage Engine (SE) Metrics

When you execute a DAX query with execution metrics, the trace captures detailed timing data:

| Metric | Description | Target |
|--------|-------------|--------|
| **TotalDuration** | End-to-end query execution time (ms) | Lower is better |
| **FormulaEngineDuration** | Single-threaded FE processing time | Should be < 30% of total |
| **StorageEngineDuration** | Multi-threaded SE query time | Should be > 70% of total |
| **StorageEngineQueryCount** | Number of SE queries generated | Fewer is better (1-3 ideal) |
| **StorageEngineCpuTime** | Total CPU across SE threads | Higher ratio to SE Duration = good parallelism |
| **VertipaqCacheMatches** | Cache hits (SE queries answered from memory) | Only relevant on warm cache |

**Key ratios:**
- **SE Parallelism Factor** (`storageEngineCpuFactor` in raw metrics) = StorageEngineCpuTime / StorageEngineDuration (net wall-clock). Values > 1.5 indicate good parallel utilization.
- **FE Percentage** = FormulaEngineDuration / TotalDuration. High FE% indicates optimization opportunities.
- **SE Percentage** = StorageEngineDuration / TotalDuration. Ideally > 70%.

> **Net wall-clock:** StorageEngineDuration is the *union* of overlapping SE intervals — not the sum of individual query durations. If three 100ms SE scans run concurrently, the wall clock is ~100ms, not 300ms. This makes SE_Par reflect true thread utilization.

### Working with Raw Aggregate Metrics

When server timings are returned as `CalculatedExecutionMetrics` (raw aggregate fields rather than a pre-processed Performance summary), use these field name mappings to apply the same analysis:

| Conceptual metric | Raw field name |
|---|---|
| SE Parallelism Factor | `storageEngineCpuFactor` |
| FE % | `formulaEngineDurationPercentage` |
| SE % | `storageEngineDurationPercentage` |
| SE Query Count | `storageEngineQueryCount` |
| Total Duration | `totalDuration` (ms) |
| SE Duration | `storageEngineDuration` (ms) |
| FE Duration | `formulaEngineDuration` (ms) |
| Cache Hits | `vertipaqCacheMatches` |

**Parallelism — aggregate vs. per-scan:**

`storageEngineCpuFactor` = `storageEngineCpuTime / storageEngineDuration` — arithmetically identical to SE_Par, apply the same >1.5 threshold. The denominator (`storageEngineDuration`) is computed as the union of overlapping SE intervals (same algorithm as SE_Par), so the factor correctly reflects thread utilization. This is the **aggregate** across all SE queries in the run.

When per-scan events are available (e.g., individual `VertiPaqSEQueryEnd` entries), each scan has its own `cpuTime` and `duration`. Per-scan parallelism = `cpuTime / duration` for that event. A query with a high aggregate `storageEngineCpuFactor` but one outlier scan with `cpuTime ≈ duration` has a single unparallelized scan hiding inside an otherwise healthy profile.

**FE processing gaps:**

`formulaEngineDuration` is computed as the sum of all time intervals where no SE query was executing — the gaps between SE events on the timeline. A large `formulaEngineDuration` means the FE spent significant time between SE scans doing post-processing (iterating over datacaches, evaluating callbacks, applying branching logic). When per-scan trace events are available, these gaps are the intervals between consecutive `VertiPaqSEQueryEnd` and the next `VertiPaqSEQueryBegin`.

### Analyzing Trace Events

**Pre-processed EventDetails waterfall (execute_dax_query):**
When trace events are returned as a pre-processed `EventDetails` array, the structure interleaves FE gap blocks between SE query events — the same view as DAX Studio's Server Timings timeline. Each SE entry includes: `type`, `subclass` (VertiPaqScan / BatchVertiPaqScan / DirectQuery), `duration_ms`, `cpu_ms`, `rows`, `kb`, `query` (xmSQL text), `par` (cpuTime/duration for that individual scan), and `timeline` (start/end offsets in ms from query start + relative percentages). FE entries have `type: "FE"` and `duration_ms` only. Read the waterfall by finding the SE entry with the highest `duration_ms` — that is the primary optimization target. Check its `par` value: a scan with `par ≈ 1.0` is single-threaded despite healthy aggregate SE_Par. Check its `query` field first for `CallbackDataID` or `EncodeCallback` before drawing any other conclusion.

**Raw trace events (when EventDetails is not available):**
Fetch `VertiPaqSEQueryEnd` events directly. Key events to examine:

**Storage Engine Events (VertiPaqSEQueryEnd):**
- **TextData**: The xmSQL query sent to the storage engine. Look for:
  - **CallbackDataID**: Indicates FE callbacks forcing row-by-row evaluation — a major performance bottleneck
  - **EncodeCallback**: Grouping by calculated expressions instead of physical columns
  - Column references (may appear as IDs like `$Column3` — these map to physical columns)
  - **`[Estimated size (volume, marshalling bytes): X, Y]`** appended at the end of TextData — X is rows scanned, Y is marshalling bytes (divide by 1024 for KB). These are not separate event fields; parse them from TextData.
- **Duration**: How long this individual SE query took
- **CpuTime**: CPU time for this SE query

**VertiPaqSEQueryCacheMatch:** Fires when an SE query is answered from cache. Duration, CpuTime, and timestamps are null. The matching query is in TextData. The `VertiPaqSEQueryEnd` still fires with Duration=0 and CpuTime=0. When `vertipaqCacheMatchesPercentage` is high, `storageEngineDuration` and `storageEngineCpuFactor` will be near 0 — not meaningful for parallelism analysis. Always clear cache before benchmarking.

### What to Look For

When analyzing a slow query, scan for these signals in order of impact:

1. **Callbacks in SE queries**: Search for `CallbackDataID` or `EncodeCallback` in SE event TextData — forces single-threaded row-by-row evaluation; the highest-impact bottleneck
2. **High FE percentage**: > 30–50% FE indicates the formula engine is doing too much work; usually paired with many short SE queries
3. **Many SE queries**: More than 5–10 SE queries suggests poor fusion — the FE is making unnecessary round-trips rather than letting the SE handle operations in bulk
4. **Large materializations**: SE queries returning >>100K rows when the final result is much smaller — the FE is filtering after full materialization instead of pushing filters to the SE
5. **Low degree of parallelism**: SE Parallelism Factor close to 1.0 on large, slow scans — the SE is running single-threaded; often caused by too few data segments
6. **Whole table scans**: SE queries with no WHERE clause and very high row counts — typically caused by `FILTER(Table, ...)` patterns that should be `CALCULATETABLE`
7. **High KB in SE events**: Large data volumes (>10MB+) per SE event mean the FE is materializing wide intermediate tables; look for opportunities to reduce columns or aggregate earlier

**Prioritization:** Fix callbacks first. Then reduce SE query count (DAX structure). Then investigate parallelism and data volume (data layout).

**Target the highest-duration SE scan first.** The single longest individual SE query gives the greatest return per optimization effort — ignore the 0ms cache-hit scans until the slow one is resolved.

---

### xmSQL Pattern Searching

Trace event TextData contains raw xmSQL strings. Because xmSQL is structured text, simple substring matching is sufficient to identify all major patterns. Check these in order of priority after collecting SE trace events.

**1. Callback detection — fix first:**
Search all SE TextData for `CallbackDataID` or `EncodeCallback`. Either string means the SE is handing scalar evaluation back to the FE one row at a time — single-threaded and slow regardless of fact table size. Fix these before investigating any other bottleneck. See DAX001–DAX007 for specific elimination patterns.

**2. Two-step dimension pre-scans:**
Look for a pair of SE queries where the first scans only a dimension table (e.g., `SELECT ... FROM Customer` with no LEFT OUTER JOIN to a fact) and the second uses `ININDEX` against the result. This means the FE materialized dimension keys as an intermediate filter set rather than letting the SE handle the join directly. When the dimension filter is simple (a single column predicate), restructuring the DAX to use the model relationship will collapse both queries into one.

**3. Repeated fact table scans:**
Find SE queries that hit the same fact table with the same LEFT OUTER JOINs but different WHERE clauses or different aggregated expressions. This is the signature of blocked vertical or horizontal fusion — the engine is making N trips to the fact when one trip should suffice. See the Vertical Fusion and Horizontal Fusion sections below.

**4. Unfiltered full-table scans:**
An SE query with no WHERE clause scans every row in the table. If the final query result is small, the filter is being applied by the FE after full materialization rather than being pushed to the SE. This typically comes from `FILTER(Table, condition)` being used instead of `CALCULATETABLE` — see DAX009.

**5. Large semi-join index tables:**
`DEFINE TABLE` + `ININDEX` patterns are semi-joins. They are efficient when the index table is small and selective. If `$Filter0` contains thousands of rows and the fact table is also large, the semi-join may add more overhead than it removes. Consider whether TREATAS or CROSSFILTER (DAX022) can replace the filter propagation with a cleaner single-step query.

---

### DAX vs. Data Layout: Reading the Signal

Two fundamentally different problems can both appear as a slow visual. The server timing trace tells you which path to take:

**Many SE queries + high FE time + individually short SE scans → DAX problem**

The FE is issuing many small SE scans because fusion is blocked, callbacks are forcing row-by-row evaluation, or filters are being resolved iteratively. Fix the DAX — consolidate measures, eliminate callbacks, restructure FILTER patterns. Tier 1 and Tier 2 patterns apply here.

*Example profile:* 109 SE queries, 30% FE / 70% SE, most scans under 10ms, same join pattern repeated. After DAX restructuring: 4 SE queries, 1% FE / 99% SE.

**Few SE queries + low FE time + high SE duration + low parallelism → Data layout problem**

The DAX is clean and the FE has little work to do, but the SE scans are slow because the data is not structured for parallel execution. Further DAX changes will not help. The fix requires restructuring the data: increase rowgroup count (more segments = more parallelism), V-order Delta files for better compression, partition on highly filtered columns, or presort data on most-used filter columns.

For Import models, these are fixed at refresh time and require upstream ETL changes. For Direct Lake, the Spark pipeline that writes the Delta table controls all of them.

---

## Section 3: Tier 1–2: Query Optimization

### Tier 1: DAX Optimization Patterns

### Core DAX Query Guidelines

These are baseline rules that apply to all DAX query optimization. Apply these first before looking for specific numbered patterns.

**Prefer SUMMARIZECOLUMNS over ADDCOLUMNS + SUMMARIZE:**

SUMMARIZECOLUMNS is fully supported in measures and enables better fusion optimization opportunities. The engine can push more work to the SE and process grouping + calculation in a single step.

```dax
-- Anti-pattern
ADDCOLUMNS (
    SUMMARIZE ( Table, Table[Column] ),
    "@Calculation", [Measure]
)

-- Preferred
SUMMARIZECOLUMNS (
    Table[Column],
    "@Calculation", [Measure]
)
```

**CRITICAL: Filter context with SUMMARIZECOLUMNS — always wrap with CALCULATETABLE:**

Filters cannot be applied as direct arguments to SUMMARIZECOLUMNS (other than via TREATAS). Wrap with CALCULATETABLE instead:

```dax
-- Wrong (invalid syntax)
SUMMARIZECOLUMNS (
    Table[Column],
    Table[FilterColumn] = "Value",
    "@Calculation", [Measure]
)

-- Correct
CALCULATETABLE (
    SUMMARIZECOLUMNS (
        Table[Column],
        "@Calculation", [Measure]
    ),
    Table[FilterColumn] = "Value"
)
```

**Remove redundant filter predicates:**

Intermediate filter variables that duplicate an existing filter argument should be eliminated:

```dax
-- Anti-pattern: FilteredValues duplicates the Amount > 1000 filter already applied
VAR FilteredValues = CALCULATETABLE ( DISTINCT ( Table[Key1] ), Table[Amount] > 1000 )
VAR Result =
    CALCULATETABLE (
        SUMMARIZECOLUMNS ( Table[Key2], "TotalQty", SUM ( Table[Quantity] ) ),
        Table[Amount] > 1000,
        Table[Key1] IN FilteredValues
    )

-- Optimized
VAR Result =
    CALCULATETABLE (
        SUMMARIZECOLUMNS ( Table[Key2], "TotalQty", SUM ( Table[Quantity] ) ),
        Table[Amount] > 1000
    )
```

**Avoid forcing measures to never return BLANK:**

Adding `+ 0` to a measure, or wrapping with `IF(ISBLANK([M]), 0, [M])`, prevents SUMMARIZECOLUMNS from auto-removing rows where no data exists. This forces the engine to evaluate and return every groupby combination — including those with no data — inflating the result set and increasing SE scan cost.

```dax
-- Anti-pattern: forces evaluation on all dimension rows
MEASURE Sales[Revenue] = SUM ( Sales[SalesAmount] ) + 0

-- Preferred: SUMMARIZECOLUMNS removes blank rows automatically
MEASURE Sales[Revenue] = SUM ( Sales[SalesAmount] )
```

Use BLANK suppression only at the visual layer ("Show items with no data") or inside explicit conditional logic where BLANK has a specific meaning — not as a blanket `+ 0` on base measures.

### DAX001: Use SUMMARIZECOLUMNS to Create Virtual Columns

SUMMARIZECOLUMNS directly defines grouping + calculation allowing better optimization. ADDCOLUMNS and SUMMARIZE with aggregation columns should be replaced.

**Anti-patterns:**
```dax
SUMMARIZE ( Sales, Sales[ProductKey], "Total Profit", [Profit] )
ADDCOLUMNS ( Sales, "Total Profit", CALCULATE ( [Profit] ) )
ADDCOLUMNS ( VALUES(Sales[ProductKey]), "Total Profit", [Profit] )
```

**Preferred:**
```dax
SUMMARIZECOLUMNS ( Sales[ProductKey], "Total Profit", [Profit] )
```

---

### DAX002: Variable Caching for Repeated Evaluations

Repeating the same expression — whether an expensive measure inside iterators or a scalar value used multiple times — causes duplicate SE queries. Cache in a variable.

**Anti-pattern — same measure iterated twice:**
```dax
VAR A = SUMX ( Sales, [Total Sales] )
VAR B = AVERAGEX ( Sales, [Total Sales] )
```

**Preferred — materialize once:**
```dax
VAR Base = SUMMARIZECOLUMNS ( Sales[ProductKey], "@TotalSales", [Total Sales] )
VAR A = SUMX ( Base, [@TotalSales] )
VAR B = AVERAGEX ( Base, [@TotalSales] )
```

**Anti-pattern — same measure referenced multiple times:**
```dax
VAR TotalA = [Sales Amount] * 1.1
VAR TotalB = [Sales Amount] * 0.9
VAR TotalC = [Sales Amount] + 1000
```

**Preferred:**
```dax
VAR _SalesAmount = [Sales Amount]
VAR TotalA = _SalesAmount * 1.1
VAR TotalB = _SalesAmount * 0.9
VAR TotalC = _SalesAmount + 1000
```

**Also applies to inline conditionals:**
```dax
// Anti-pattern:
IF([Sales Amount] > 1000, [Sales Amount] * 2, [Sales Amount] / 2)

// Preferred:
VAR _SalesAmount = [Sales Amount]
RETURN IF(_SalesAmount > 1000, _SalesAmount * 2, _SalesAmount / 2)
```

---

### DAX003: Duplicate Filter Conditions

Consolidating filters into a single context modifier avoids redundant evaluation.

**Anti-pattern:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    Sales[Year] = 2023,
    FILTER(Sales, Sales[Year] = 2023)
)
```

**Preferred:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    Sales[Year] = 2023
)
```

---

### DAX004: SUMMARIZE with Complex Table Expression

Using SUMMARIZE with complex table expressions as the first argument prevents optimization. Wrap with CALCULATETABLE instead.

**Anti-pattern:**
```dax
SUMMARIZE(
    CALCULATETABLE(Sales, Sales[Year] = 2023, Sales[CustomerKey] IN SellingPOCs),
    Sales[CustomerKey],
    "DistinctSKUs", DISTINCTCOUNT(Sales[StoreKey])
)
```

**Preferred:**
```dax
CALCULATETABLE(
    SUMMARIZECOLUMNS(
        Sales[CustomerKey],
        "DistinctSKUs", DISTINCTCOUNT(Sales[StoreKey])
    ),
    Sales[Year] = 2023,
    Sales[CustomerKey] IN SellingPOCs
)
```

---

### DAX005: Replace Iterator with Context Transition Using SUMMARIZECOLUMNS Base Table

Materializing context transition results once in SUMMARIZECOLUMNS and then iterating over pre-calculated values improves efficiency.

**Anti-pattern:**
```dax
SUMX(
    VALUES(Sales[CustomerKey]),
    CALCULATE(SUM(Sales[Amount]))
)
```

**Preferred:**
```dax
SUMX(
    SUMMARIZECOLUMNS(
        Sales[CustomerKey],
        "@Amount", SUM(Sales[Amount])
    ),
    [@Amount]
)
```

**Anti-pattern:**
```dax
AVERAGEX(
    Products,
    CALCULATE([Total Sales], Sales[Year] = 2023)
)
```

**Preferred:**
```dax
AVERAGEX(
    SUMMARIZECOLUMNS(
        Products[ProductKey],
        "@TotalSales", CALCULATE([Total Sales], Sales[Year] = 2023)
    ),
    [@TotalSales]
)
```

---

### DAX006: Replace IF with INT for Boolean Conversion

INT with boolean expressions avoids conditional logic callbacks that IF statements trigger.

**Anti-pattern:**
```dax
SUMX( Sales, IF(Sales[Amount] > 1000, 1, 0) )
```

**Preferred:**
```dax
SUMX( Sales, INT(Sales[Amount] > 1000) )
```

---

### DAX007: Context Transition in Iterator

Context transition is powerful but expensive. Optimize by:

1. **Remove it completely:**
```dax
// Instead of: SUMX( Sales, [Sales Amount] )
// Use: SUMX( Sales, Sales[Unit Price] * Sales[Quantity] )
```

2. **Reduce number of columns:**
```dax
// Instead of: SUMX( Account, [Total Sales] )
// Use: SUMX( VALUES ( Account[Account Key] ), [Total Sales] )
```

3. **Reduce cardinality before iteration:**
```dax
// Instead of: SUMX( Account, [Total Sales] * Account[Corporate Discount] )
// Use: SUMX( VALUES ( Account[Corporate Discount] ), [Total Sales] * Account[Corporate Discount] )
```

---

### DAX009: Apply Filters Using CALCULATETABLE Instead of FILTER

CALCULATETABLE modifies filter context directly for better query plans.

**Anti-pattern:**
```dax
FILTER( 'Sales', 'Sales'[Year] = 2023 )
```

**Preferred:**
```dax
CALCULATETABLE( 'Sales', 'Sales'[Year] = 2023 )
```

---

### DAX010: Move Context-Independent Expressions Outside Iterators

Expressions that don't depend on the iteration context should be cached in variables.

**Anti-pattern:**
```dax
SUMX(
    Sales,
    Sales[Quantity] * [Average Price] * 1.1
)
// [Average Price] doesn't change per Sales row
```

**Preferred:**
```dax
VAR _AvgPrice = [Average Price]
VAR _Markup = 1.1
RETURN
SUMX( Sales, Sales[Quantity] * _AvgPrice * _Markup )
```

---

### DAX011: Use Simple Column Filter Predicates as CALCULATE Arguments

CALCULATE accepts simple boolean predicates directly — more efficient than wrapping in FILTER or CALCULATETABLE. Split `&&` into separate filter arguments.

**Anti-pattern:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    FILTER(ALL(Product[Category]), Product[Category] = "Electronics")
)
```

**Preferred:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    Product[Category] = "Electronics"
)
```

**Anti-pattern:**
```dax
CALCULATETABLE( Sales, Sales[Region] = "West" && Sales[Amount] > 1000 )
```

**Preferred:**
```dax
CALCULATETABLE( Sales, Sales[Region] = "West", Sales[Amount] > 1000 )
```

---

### DAX012: Distinct Count Alternatives

Depending on cardinality and data layout, moving DISTINCTCOUNT to SUMX(VALUES(),1) can improve performance by forcing FE evaluation.

**Storage Engine Bound:**
```dax
DISTINCTCOUNT(Sales[CustomerKey])
```

**Formula Engine Bound (sometimes faster):**
```dax
SUMX(VALUES(Sales[CustomerKey]), 1)
```

---

### DAX013: Force Formula Engine Evaluation with CROSSJOIN

Context transitions in FILTER over VALUES can cause additional SE scans. Forcing FE evaluation with CROSSJOIN can eliminate redundant SE queries.

**Anti-pattern:**
```dax
CALCULATE(
    [Total Sales],
    FILTER(VALUES(Product[ProductKey]), [Total Sales])
)
```

**Preferred:**
```dax
CALCULATE(
    [Total Sales],
    FILTER(CROSSJOIN(VALUES(Product[ProductKey]), {1}), [Total Sales])
)
```

---

### DAX014: Replace SELECTEDVALUE with MAX/MIN

When context guarantees a single value (inside iterator over VALUES/DISTINCT), MAX/MIN avoids the unnecessary cardinality check SELECTEDVALUE performs.

**Anti-pattern:**
```dax
SUMX(
    VALUES(Product[Category]),
    SELECTEDVALUE(Product[Category]) & ": " & FORMAT([Total Sales], "#,0")
)
```

**Preferred:**
```dax
SUMX(
    VALUES(Product[Category]),
    MAX(Product[Category]) & ": " & FORMAT([Total Sales], "#,0")
)
```

---

### DAX015: Use ALLEXCEPT Instead of ALL + VALUES Restoration

When clearing filter context with ALL() and then restoring specific columns via VALUES(), ALLEXCEPT achieves the same in one operation.

**Anti-pattern:**
```dax
CALCULATE( [Total Sales], ALL(Sales), VALUES(Sales[Region]) )
```

**Preferred:**
```dax
CALCULATE( [Total Sales], ALLEXCEPT(Sales, Sales[Region]) )
```

---

### DAX016: Flatten Nested CALCULATE Calls

Nested CALCULATE calls create multiple context transitions. Flatten by merging filter arguments.

**Anti-pattern:**
```dax
CALCULATE(
    CALCULATE( [Total Sales], Sales[Region] = "West" ),
    Date[Year] = 2023
)
```

**Preferred:**
```dax
CALCULATE(
    [Total Sales],
    Sales[Region] = "West",
    Date[Year] = 2023
)
```

---

### DAX017: SWITCH/IF Branch Optimization in SUMMARIZECOLUMNS

When a SWITCH or IF is used inside SUMMARIZECOLUMNS, the engine attempts to evaluate only the branch matching the current context and skip all others. When this optimization succeeds, the query is fast. When it fails, the engine materializes a full cartesian product of all groupby column combinations before applying the branch logic — catastrophically slow on visuals with many row/column fields.

**Three things that break branch optimization:**

**1. Multiple aggregations combined within one branch:**
The engine can only optimize a branch containing a single root aggregation. Combining two aggregations with arithmetic disables it:
```dax
-- Slow: "Margin" branch has two separate SUM calls — branch cannot be short-circuited
MEASURE Sales[Selected Metric] =
    SWITCH (
        SELECTEDVALUE ( Metric[Name] ),
        "Revenue", SUM ( Sales[SalesAmount] ),
        "Margin",  SUM ( Sales[SalesAmount] ) - SUM ( Sales[TotalCost] )
    )
```

**Fix:** Merge the arithmetic into a single row-level expression inside one SUMX:
```dax
MEASURE Sales[Selected Metric] =
    SWITCH (
        SELECTEDVALUE ( Metric[Name] ),
        "Revenue", SUM ( Sales[SalesAmount] ),
        "Margin",  SUMX ( Sales, Sales[SalesAmount] - Sales[TotalCost] )
    )
```

**2. Mismatched data types across branches:**
All SWITCH branches must return the same type. When they differ, the engine inserts an implicit conversion that prevents the optimization:
```dax
-- Slow: SalesAmount is Currency, OrderQuantity is Integer — implicit cast disables optimization
SWITCH (
    SELECTEDVALUE ( Metric[Name] ),
    "Revenue",  SUM ( Sales[SalesAmount] ),
    "Quantity", SUM ( Sales[OrderQuantity] )
)

-- Fix: explicitly match types with CONVERT
SWITCH (
    SELECTEDVALUE ( Metric[Name] ),
    "Revenue",  SUM ( Sales[SalesAmount] ),
    "Quantity", CONVERT ( SUM ( Sales[OrderQuantity] ), CURRENCY )
)
```

**3. Measure references (context transitions) inside iterator branches:**
A measure called inside SUMX/AVERAGEX within a SWITCH branch forces a CallbackDataID — the SE cannot evaluate the measure natively:
```dax
-- Slow: [Unit Discount] is a measure → CallbackDataID in SE trace
SWITCH (
    SELECTEDVALUE ( Metric[Name] ),
    "Discount Total", SUMX ( Sales, Sales[OrderQuantity] * [Unit Discount] )
)

-- Fix: cache the measure value in a variable before the SWITCH
VAR _UnitDiscount = [Unit Discount]
RETURN
    SWITCH (
        SELECTEDVALUE ( Metric[Name] ),
        "Discount Total", SUMX ( Sales, Sales[OrderQuantity] * _UnitDiscount )
    )
```

**Detection:** Server Timings shows very high FE duration with little SE work. The broken pattern produces a large intermediate cartesian result — visible as a VertiScan row count far exceeding the final output row count.

---

### DAX018: Avoid Dimension Key Columns in Groupby

When a query groups by the key column of a dimension table (the one-side of a relationship), the engine splits execution into two SE queries — one for the fact aggregation (with only the key), one for the dimension attributes. This is usually fine, but can add overhead for large dimensions.

**Anti-pattern (key column in groupby — two SE queries):**
```dax
SUMMARIZECOLUMNS (
    Product[ProductKey],      -- Key column triggers split
    Product[ProductName],
    Product[Category],
    "Revenue", SUM ( Sales[Amount] )
)
```

SE Query 1:
```
SELECT Product[ProductKey], SUM ( Sales[Amount] )
FROM Sales LEFT OUTER JOIN Product ON Sales[ProductKey] = Product[ProductKey]
```

SE Query 2:
```
SELECT Product[ProductKey], Product[ProductName], Product[Category]
FROM Product
```

**Preferred (non-key column — single SE query):**
```dax
SUMMARIZECOLUMNS (
    Product[ProductCode],     -- Not the relationship key
    Product[ProductName],
    Product[Category],
    "Revenue", SUM ( Sales[Amount] )
)
```

This produces one SE query with all columns. The tradeoff is dimension values repeat in the result, but for most models this is faster than the extra SE round-trip.

**When to apply:** If you see paired SE queries (fact + unfiltered dimension scan) in traces and the dimension scan is slow, check whether the key column can be swapped for another unique identifier.

---

### DAX019: Use COUNTROWS Instead of DISTINCTCOUNT on Key Columns

The engine automatically converts `DISTINCTCOUNT(Table[KeyColumn])` to `COUNTROWS(Table)` when the column is on the one-side of a relationship or marked as a key. COUNTROWS avoids the distinct count algorithm entirely. When you know a column is a primary key, write COUNTROWS directly for clarity and to guarantee the optimization.

**Anti-pattern:**
```dax
DISTINCTCOUNT ( Product[ProductKey] )
```

**Preferred:**
```dax
COUNTROWS ( Product )
```

For non-key columns where DISTINCTCOUNT is a bottleneck, see DAX012 for alternatives.

---

### DAX020: Move Calculation to Lower Granularity

When an iterator scans a high-cardinality column but the calculation only depends on a low-cardinality attribute, restructure to iterate over the attribute instead. This drastically reduces the number of iterations and context transitions.

**Anti-pattern:** Iterating over every customer to check their discount tier:
```dax
MEASURE Sales[WeightedDiscount] =
    SUMX (
        Customer,
        CALCULATE ( SUM ( Sales[Amount] ) ) * Customer[DiscountRate]
    )
```
If there are 100K customers but only 5 distinct DiscountRate values, the engine performs 100K context transitions.

**Optimized:** Iterate over the distinct discount rates instead:
```dax
MEASURE Sales[WeightedDiscount] =
    SUMX (
        VALUES ( Customer[DiscountRate] ),
        CALCULATE ( SUM ( Sales[Amount] ) ) * Customer[DiscountRate]
    )
```
5 iterations instead of 100K. The CALCULATE inside resolves the filter for each rate value across all customers with that rate.

**Detection:** Look for SUMX/AVERAGEX over a large table where the expression only references one or two columns from that table. Check the cardinality of those columns — if much lower than the table row count, this pattern applies.

---

### DAX021: Iterate Over Fact Keys vs. Dimension Keys

When an iterator needs to scan unique combinations, choosing to iterate over fact-side keys vs. dimension-side keys affects performance. Fact tables are typically wider (more columns in the datacache), while dimension tables are narrower.

**Consider:** If your measure iterates over a dimension to compute per-entity results, the engine creates a datacache from the dimension. If it iterates over the fact table, it creates a datacache from the fact. The smaller datacache wins.

**Pattern A — Iterate dimension (fewer rows, narrow scan):**
```dax
MEASURE Sales[AvgCustomerRevenue] =
    AVERAGEX (
        VALUES ( Customer[CustomerKey] ),
        CALCULATE ( SUM ( Sales[Amount] ) )
    )
```

**Pattern B — Iterate fact (when fact is pre-filtered to few rows):**
```dax
MEASURE Sales[AvgTransactionRevenue] =
    AVERAGEX (
        Sales,
        Sales[Amount]
    )
```

**Decision criteria:**
- If the dimension has far fewer rows than the filtered fact → iterate dimension
- If the fact is heavily filtered (e.g., single day, single product) and smaller than the dimension → iterate fact
- Check SE query row counts in traces to validate which produces a smaller datacache

---

### DAX022: Override Relationships with TREATAS and CROSSFILTER

Instead of modifying the physical model, you can override relationship behavior directly in DAX. Use `TREATAS` to pass filters along a virtual relationship and `CROSSFILTER` to disable an existing one. This lets you experiment with relationship changes without model modifications.

**Scenario:** A bidirectional many-to-many through a bridge table is causing expensive expanded tables. You want to test passing filters directly instead.

**Anti-pattern:** Physical bidirectional relationship causing excess SE queries:
```dax
MEASURE Sales[CustomerSportRevenue] =
    SUM ( Sales[Amount] )
    -- Relies on bidirectional filter from SportBridge through Customer
```

**Optimized — disable physical relationship, pass filter via TREATAS:**
```dax
MEASURE Sales[CustomerSportRevenue] =
    CALCULATE (
        SUM ( Sales[Amount] ),
        CROSSFILTER ( Customer[CustomerKey], SportBridge[CustomerKey], NONE ),
        TREATAS (
            VALUES ( SportBridge[CustomerKey] ),
            Customer[CustomerKey]
        )
    )
```

**When to use:**
- Bidirectional relationships causing high SE query counts
- Many-to-many bridge tables creating large expanded tables
- Testing whether a different filter propagation path is faster without committing to model changes
- Large dimensions where switching from 1→* to *→* with selective filters may reduce SE work

**Trade-offs:** TREATAS bypasses auto-exist behavior. Results may differ from the physical relationship version if the bridge table contains combinations not present in the fact. Always verify semantic equivalence.

**Experiment with relationship direction and cardinality:**

Beyond TREATAS/CROSSFILTER, the physical relationship itself — specifically its filter direction and which table is on each side — directly controls whether the engine generates a **standard LEFT OUTER join** or a **semi-join batch plan**. These have very different SE query counts and durations.

- A **standard left join** issues one SE query that retrieves the fact aggregation with the dimension joined in. Fast, few queries.
- A **semi-join batch plan** scans the dimension first to collect matching keys, then uses `ININDEX` to filter the fact. This adds an extra SE event per filter and can multiply total SE query count significantly.

Changing from bidirectional to unidirectional filtering, or swapping which table owns the "one" side, can shift the engine from one plan to the other. The only way to know which is faster for your specific model is to test both. Use TREATAS/CROSSFILTER in DAX to prototype before committing to a physical model change.

**Example:** Switching a relationship from bidirectional (Many↔Many) to unidirectional (Many→One) reduced total from 17,031ms / 105 SE queries to 3,847ms / 98 SE queries on the same underlying data — a 4.4× improvement with no DAX changes.

---

### DAX023: 1-Column Fusion via MAXX for Same-Column Filter Variants

When multiple measures each filter the same column to different values (e.g., Sales by Package = "Bag", Sales by Package = "Box"), the engine issues separate SE queries — one per filter value. The 1-column fusion pattern forces all variants into a **single SE scan** by using `MAXX` over all distinct values with a boolean multiplier.

**Anti-pattern — separate SE query per filter value:**
```dax
MEASURE Sales[Sales - Bag] = CALCULATE ( [Base Measure], Sales[Package] = "Bag" )
MEASURE Sales[Sales - Box] = CALCULATE ( [Base Measure], Sales[Package] = "Box" )
```
Each measure triggers its own SE query with a different `WHERE` clause. On DQ sources, this means multiple round-trips to the data source.

**Optimized — single SE scan via MAXX:**
```dax
MEASURE Sales[Sales - Bag] =
    MAXX (
        ALL ( Sales[Package] ),
        [Base Measure] * ( Sales[Package] = "Bag" )
    )

MEASURE Sales[Sales - Box] =
    MAXX (
        ALL ( Sales[Package] ),
        [Base Measure] * ( Sales[Package] = "Box" )
    )
```

**How it works:** `MAXX` iterates over all distinct values of `Sales[Package]` (e.g., Bag, Box, Bulk, Each). For each iteration, the boolean `(Sales[Package] = "Bag")` evaluates to 1 for the matching value and 0 for all others. `MAXX` returns the one non-zero result. Because all variants share the same `ALL(Sales[Package])` scan, the engine fuses them into one SE query.

**Performance impact:** Up to 10x speedup on Direct Query when multiple same-column filtered measures appear in the same visual. Also effective on Import models with large fact tables.

**Limitation:** Only works when variants filter the **same column** to different values. For filters on different columns, standard vertical fusion (DAX engine grouping shared-context measures) applies instead.

---

### DAX024: SPARSE vs DENSE SUMMARIZECOLUMNS — Avoid Dimension PK in Groupby

Including a **primary key (PK) column** from a dimension table in `SUMMARIZECOLUMNS` groupby forces the engine into a **DENSE query plan** that iterates all PK × other-groupby combinations — including empty intersections. This is fundamentally different from the two-SE-query concern in DAX018. DENSE queries can be 10–100× slower than SPARSE queries on large dimensions.

**Root cause:** The engine treats PK columns as "dense" (one-row-per-entity semantics). When a PK appears in the groupby alongside other columns (or relationships), the engine generates an outer join over all PK values × filter values, even when most combinations produce no data.

**Anti-pattern — PK column in groupby (DENSE plan):**
```dax
-- ProductKey is the PK of the Product dimension (one-side of a relationship)
SUMMARIZECOLUMNS (
    Product[ProductKey],    -- ← PK column → DENSE plan
    Date[CalendarYear],
    "Revenue", SUM ( Sales[Amount] )
)
```
On a 500K-product dimension with 3 years of data, the engine materializes up to 1.5M intermediate rows (500K × 3), even if only 10K product/year combinations have actual sales.

**Preferred — use a non-PK attribute (SPARSE plan):**
```dax
SUMMARIZECOLUMNS (
    Product[ProductCode],   -- ← Not the PK → SPARSE plan
    Date[CalendarYear],
    "Revenue", SUM ( Sales[Amount] )
)
```
The engine only materializes rows where actual sales exist, scanning the fact table naturally.

**Additional guidance:**
- **Hide all relationship columns** from report authors. A hidden PK cannot be added to a visual groupby by mistake.
- If you need PK in the output, add it as a non-groupby column via `ADDCOLUMNS(SUMMARIZECOLUMNS(...), "PK", RELATED(Product[ProductKey]))`.
- The same issue applies when a PK is included in a visual's "Group By" or "Row" field well.

**Detection:** In Server Timings, look for an unexpectedly large number of `VertiScan` rows in the intermediate result compared to the final row count. A ratio > 10:1 often signals DENSE plan inflation.

---

### DAX025: Replace DIVIDE() with / Operator in Iterators

The `DIVIDE()` function includes built-in divide-by-zero protection, which requires the SE to call back to the FE for each row to handle the alternate-result logic. Inside a `SUMX` or any row iterator, this produces a `CallbackDataID` in the SE trace and forces single-threaded row-by-row evaluation — even though the underlying arithmetic is simple.

**Anti-pattern — DIVIDE() inside SUMX triggers CallbackDataID:**
```dax
MEASURE Fact[WeightedValue] =
    SUMX (
        Fact,
        Fact[BaseAmount] *
            DIVIDE ( RELATED ( Items[Discount] ), RELATED ( Items[LocationAdjustment] ) )
    )
```
The SE cannot evaluate `DIVIDE()` natively. Every row triggers an FE callback.

**Preferred — native / operator, no callback:**
```dax
MEASURE Fact[WeightedValue] =
    SUMX (
        Fact,
        Fact[BaseAmount] *
            ( RELATED ( Items[Discount] ) / RELATED ( Items[LocationAdjustment] ) )
    )
```
The SE evaluates this entirely natively. No callbacks, full parallelism.

**When to apply:** Use `/` instead of `DIVIDE()` only when you can guarantee the denominator is never zero. If zero denominators are possible, handle them explicitly before the iterator:
```dax
MEASURE Fact[WeightedValue] =
    VAR _SafeItems =
        CALCULATETABLE ( Items, Items[LocationAdjustment] <> 0 )
    RETURN
        SUMX (
            Fact,
            Fact[BaseAmount] *
                ( RELATED ( Items[Discount] ) / RELATED ( Items[LocationAdjustment] ) )
        )
```

**Detection:** Look for `CallbackDataID` in SE trace events on SUMX scans where the expression contains `DIVIDE()`. Removing it eliminates the callback entirely.

---

### Vertical Fusion

Vertical fusion is the engine behavior where multiple measure aggregations that share the same filter context are combined into a single SE query. Instead of scanning the fact table once per measure, all aggregations are computed in one pass. The performance gain scales directly with fact table size — on a 1B-row table, three measures fusing into one scan is a 3× improvement at the SE level.

**How it works:**

When multiple measures aggregate from the same table under the same filter, the engine batches their expressions into one xmSQL query:

```dax
DEFINE
    MEASURE Sales[Total Amount]  = SUM ( Sales[SalesAmount] )
    MEASURE Sales[Total Tax]     = SUM ( Sales[TaxAmount] )
    MEASURE Sales[Order Count]   = COUNTROWS ( Sales )

EVALUATE
SUMMARIZECOLUMNS (
    Product[Category],
    Date[CalendarYear],
    "Amount",  [Total Amount],
    "Tax",     [Total Tax],
    "Orders",  [Order Count]
)
```

The SE receives one query covering all three measures:
```
SELECT
    Product[Category],
    Date[CalendarYear],
    SUM ( Sales[SalesAmount] ),
    SUM ( Sales[TaxAmount] ),
    COUNT ( Sales[SalesKey] )
FROM Sales
    LEFT OUTER JOIN Product ON Sales[ProductKey]   = Product[ProductKey]
    LEFT OUTER JOIN Date    ON Sales[OrderDateKey] = Date[DateKey]
```

Three measures, one table scan. Subtotals (via ROLLUPADDISSUBTOTAL) also fuse — the same SE query handles totals, group-level, and detail-level rows when measures are additive.

**What prevents vertical fusion:**

1. **Time intelligence functions** — DATESYTD, DATEADD, SAMEPERIODLASTYEAR, and similar functions require the FE to produce a dynamic date set before the SE executes. Each TI-modified measure requires its own SE query because its filter context is determined at runtime:
```dax
-- These two will NOT fuse — each requires a separate date-filtered SE scan
MEASURE Sales[This Year]  = SUM ( Sales[SalesAmount] )
MEASURE Sales[Last Year]  = CALCULATE ( SUM ( Sales[SalesAmount] ), SAMEPERIODLASTYEAR ( Date[Date] ) )
```

2. **DAX variables building date ranges** — Manually constructed date predicates block fusion the same way as built-in TI functions:
```dax
MEASURE Sales[YTD] =
    VAR _Start = DATE ( YEAR ( MAX ( Date[Date] ) ), 1, 1 )
    VAR _End   = MAX ( Date[Date] )
    RETURN CALCULATE ( SUM ( Sales[SalesAmount] ), Date[Date] >= _Start && Date[Date] <= _End )
```

3. **SWITCH/IF selecting between measures** — When branch selection determines which measure to return, the engine cannot determine at plan time which aggregation to include in the fused query:
```dax
-- Prevents fusion for all co-located measures in the same visual
MEASURE Sales[Selected] =
    SWITCH (
        SELECTEDVALUE ( Metric[Name] ),
        "Amount",  [Total Amount],
        "Orders",  [Order Count]
    )
```

4. **Calculation group items applying different filters** — Each item that modifies the filter context differently generates its own SE query.

**Takeaway:** When traces show multiple SE queries hitting the same fact table with the same joins, vertical fusion was blocked. Time intelligence functions and SWITCH selectors are the most common causes.

---

### Horizontal Fusion

Horizontal fusion occurs when the engine recognizes that multiple SE queries differ only in which slice of a column they filter, and merges them into a single SE query that retrieves all slices at once. The FE partitions the result. This reduces what would be N separate fact table scans to one.

**How it works:**

When measures filter the same column to different single values AND that column is already in the groupby list, the engine scans once and the FE reads each slice from the result:

```dax
DEFINE
    MEASURE Sales[Direct Sales]   = CALCULATE ( SUM ( Sales[SalesAmount] ), Sales[SalesChannel] = "Direct" )
    MEASURE Sales[Reseller Sales] = CALCULATE ( SUM ( Sales[SalesAmount] ), Sales[SalesChannel] = "Reseller" )

EVALUATE
SUMMARIZECOLUMNS (
    Sales[SalesChannel],
    Date[CalendarYear],
    "Direct",   [Direct Sales],
    "Reseller", [Reseller Sales]
)
```

Because `Sales[SalesChannel]` is already a groupby column, both measures share one SE query. No separate scan per channel.

**What prevents horizontal fusion:**

1. **Filtered column not in groupby, or multi-value filter:**
When the filtered column is absent from the groupby and the filter has multiple values, the engine cannot merge the scans:
```dax
-- Won't fuse: Product[Category] is not in the groupby and each measure filters different values
MEASURE Sales[Bikes Revenue]       = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Bikes" )
MEASURE Sales[Accessories Revenue] = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Accessories" )
```

**Workaround:** Add the filtered column to the groupby and re-aggregate in an outer GROUPBY:
```dax
EVALUATE
GROUPBY (
    SUMMARIZECOLUMNS (
        Product[Category],
        Date[CalendarYear],
        "Amount", SUM ( Sales[SalesAmount] )
    ),
    Date[CalendarYear],
    "Amount", SUMX ( CURRENTGROUP (), [Amount] )
)
```

2. **Table-valued filter (e.g., time intelligence) applied per-measure:**
A filter that produces a table rather than a scalar prevents slice merging even when the column-level filter is identical across measures:
```dax
-- Won't fuse: DATESYTD returns a table filter, blocking horizontal merging
MEASURE Sales[Bikes YTD]       = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Bikes",       DATESYTD ( Date[Date] ) )
MEASURE Sales[Accessories YTD] = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Accessories", DATESYTD ( Date[Date] ) )
```

**Workaround:** Move the table-valued filter to an outer CALCULATE and keep only the column-slice filter inside each base measure:
```dax
MEASURE Sales[Bikes]        = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Bikes" )
MEASURE Sales[Accessories]  = CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = "Accessories" )
MEASURE Sales[Combined YTD] = CALCULATE ( [Bikes] + [Accessories], DATESYTD ( Date[Date] ) )
```

3. **Filter value computed at runtime:**
If the filter value is stored in a variable, the engine treats the filter as dynamic and will not fuse, even if the value is predictable:
```dax
-- Won't fuse: _Category is a runtime variable
MEASURE Sales[Bikes Latest] =
    VAR _Category = "Bikes"
    RETURN CALCULATE ( SUM ( Sales[SalesAmount] ), Product[Category] = _Category )
```

**Workaround:** Move the dynamic filter to an outer CALCULATE so the inner measure is filter-free.

**Takeaway:** When traces show N near-identical SE queries with the same joins but only the WHERE filter differing, horizontal fusion failed. Either add the filtered column to the groupby, or lift dynamic/table filters to an outer wrapper measure.

---

### Tier 2: Query Structure Patterns

> **Tier 2 — User permission required.** These recommendations change what the query returns (grain, grouping, time periods). The agent must present the trade-off and get explicit user approval before applying. Only suggest these after Tier 1 (DAX) optimizations have been exhausted or are insufficient.

### QRY001: Remove Unneeded or Global Filters

Visual-generated queries often include filters that don't affect the calculation or that apply globally and could be pushed to the data source. These filters appear as extra `WHERE` clauses in xmSQL and force additional SE joins.

**Detection:** In the generated query, look for filter variables that restrict to a single value or apply a condition that is always true for the dataset. In xmSQL, look for `WHERE` clauses on columns that don't participate in the measure logic.

**Example — unnecessary filter on a single-value column:**
```dax
-- Visual applies a filter on Currency = "USD" but the entire model is USD-only
EVALUATE
SUMMARIZECOLUMNS (
    Product[Category],
    KEEPFILTERS ( TREATAS ( {"USD"}, Currency[Code] ) ),
    "Revenue", [Total Revenue]
)
```

**Recommendation to user:** "This filter on Currency = 'USD' is redundant because your model only contains USD data. Removing it eliminates an SE join. If this filter comes from a slicer or visual filter, consider removing it from the report."

**Alternatively**, if the filter is not redundant but is global (applied to every visual), suggest pushing it to the data source or applying it via RLS, which avoids per-query filter overhead.

---

### QRY002: Eliminate Report Measure Filters (__ValueFilterDM)

When a visual applies a filter on a measure value (e.g., "show only rows where Revenue > 1M"), Power BI generates a `__ValueFilterDM` pattern that effectively computes the measure twice: once to evaluate the filter condition and once for the display value.

**Detection:** Look for `__ValueFilterDM` variables in the generated query:
```dax
VAR __ValueFilterDM1 = 
    FILTER (
        KEEPFILTERS (
            SUMMARIZECOLUMNS (
                'Calendar'[Date],
                "Revenue", [Total Revenue]
            )
        ),
        [Revenue] > 1000000
    )

VAR __DS0Core = 
    SUMMARIZECOLUMNS (
        ROLLUPADDISSUBTOTAL ( 'Calendar'[Date], "IsGrandTotalRowTotal" ),
        __ValueFilterDM1,
        "Revenue", [Total Revenue]
    )
```

**Impact:** The measure is computed in both the filter evaluation and the final result — roughly doubling execution time.

**Recommendation to user:** "This visual has a measure filter that causes the calculation to run twice. Options:
1. Move the filter logic into the measure itself (e.g., return BLANK() when Revenue <= 1M) so the visual's auto-remove-blank handles it
2. Create a pre-filtered calculated table or use a slicer on a dimension attribute instead
3. Accept the performance cost if the filter is essential"

**Simulated optimization** — the agent can test option 1 by rewriting:
```dax
MEASURE Sales[Total Revenue Filtered] =
    VAR __Rev = [Total Revenue]
    RETURN IF ( __Rev > 1000000, __Rev )
```
This returns BLANK for rows below the threshold. SUMMARIZECOLUMNS auto-removes blank rows, achieving the same visual result without the double evaluation.

---

### QRY003: Reduce Query Grain

If the visual displays data at a granularity finer than what the user needs, reducing the grain can dramatically cut row count and SE work.

**Detection:** The query groups by a high-cardinality date column (`Calendar[Date]`) producing hundreds or thousands of rows, but the user may only need weekly or monthly aggregation.

**Recommendation to user:** Present the trade-off:
- "This query returns 365 rows (daily). Switching to monthly would return 12 rows — approximately 30x fewer SE rows to process."
- "Alternatively, we can filter to only show end-of-period values (last day of each week/month) to reduce volume while keeping a date-level grain."

**Option A — aggregate to month:**
```dax
-- Change groupby from Calendar[Date] to Calendar[YearMonth]
EVALUATE
SUMMARIZECOLUMNS (
    'Calendar'[YearMonth],
    "Revenue", [Total Revenue]
)
```

**Option B — filter to period-end dates only:**
```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Calendar'[Date],
    KEEPFILTERS ( TREATAS ( VALUES ( 'Calendar'[MonthEndDate] ), 'Calendar'[Date] ) ),
    "Revenue", [Total Revenue]
)
```

---

## Section 4: Tier 3–4: Model and Data Layout

### General Data Layout Best Practices

Data layout decisions affect query performance at the source level — before DAX and before the SE can optimize. These apply to both Import and Direct Lake models, though Direct Lake gives you the most direct control since you own the Delta table structure. Apply these after exhausting DAX optimizations; changes here typically require ETL or pipeline modifications.

**Include only the data you need.** Remove unused columns and filter rows at the source. Every unused column consumes dictionary memory and adds to cold-cache load time. Every extra row increases scan cost.

**Drop rows with null or zero values where logically safe.** If a fact row has all-zero or null metric values and will never contribute to any query result, remove it in ETL. This reduces row count (faster scans) and improves segment compression.

**Group sparse attributes into proper dimension tables.** A fact table with many low-cardinality string columns (region, category, status flags) creates large dictionaries on the fact. Move these to dimension tables related via integer keys. Integer foreign keys are tiny compared to repeated string values stored on billions of fact rows.

**Choose the right artifact type for Delta writes.** Different Fabric artifact types produce Delta files with varying quality:
- **Dataflows Gen2**: convenient but may produce many small files with suboptimal rowgroup sizes
- **Pipelines (Copy Activity)**: solid for structured sources but limited control over row layout
- **Notebooks (PySpark)**: full control over rowgroup size, sort order, partition layout, and V-ordering — use notebooks when query performance is critical

**Partition on highly filtered columns.** If a column appears in nearly every query's WHERE clause (e.g., a date key, tenant key), partitioning the Delta table on that column allows the engine to skip entire data files at scan time. Fewer files touched = faster scans and faster Direct Lake cold-cache loads.

**Presort data on most-used filter and groupby columns.** VertiPaq's run-length encoding (RLE) compression improves dramatically when column values are sorted. Sorting by the most commonly used filter/groupby column before writing reduces segment size and improves scan speed:
```python
df.sort("DateKey", "ProductKey").write.format("delta").option("vorder", "true").saveAsTable("FactSales")
```

**Cast to optimal data types.** Prefer integers over decimals for keys and codes, use fixed decimal for currency, avoid high-cardinality `Text` columns on fact tables — string dictionaries are expensive in both memory and scan time. See MDL003 for the full cardinality and type optimization guidance.

---

### Tier 3: Model Optimization Patterns

> **Tier 3 — User permission required. High-risk changes.** Model changes can break downstream reports and require coordination with CI/CD processes. The agent should explain the trade-offs, suggest working on a copy of the model, and note that source-level changes (Lakehouse, Warehouse, Power Query) may be needed. Implementation may require other skills (powerbi-semantic-model, fabric-cli) and should follow the user's deployment process.

### MDL001: Many-to-Many Relationship Optimization

Many-to-many relationships through bridge tables are common but expensive. The bridge table creates an expanded table that the engine must materialize for every query. Different physical patterns trade off model complexity vs. query performance.

**Pattern A — Standard bridge table (baseline):**
```
Customer  *──  CustomerSport  ──*  Sport
                  (bridge)
```
The engine joins Customer → CustomerSport → Sport, expanding the intermediate table. Bidirectional filters on the bridge compound the cost.

**Pattern B — Pre-computed combination table:**
Instead of a bridge, create a table of distinct combinations with a surrogate key:
```
CustomerSportCombo [ComboKey, CustomerKey, SportKey]
```
Relate the fact directly to the combo table via ComboKey. The engine resolves a single 1→* join instead of traversing a bridge. Requires upstream ETL to assign combo keys.

**Pattern C — Denormalize into the fact:**
If the many-to-many attribute is small (e.g., 5-10 sports), add it directly as columns on the fact table. Eliminates the join entirely at the cost of wider fact rows.

**When to suggest each pattern:**
- **A**: Default, simplest. Use when bridge table is small relative to the fact.
- **B**: When traces show high SE query counts from bridge traversal and the combination space is bounded.
- **C**: When the many-to-many attribute has very low cardinality and query performance is critical.

---

### MDL002: Star Schema Conformance

Snowflake schemas (dimension → sub-dimension chains) force multiple SE joins per query. Converting to a flat star schema reduces join depth and enables better fusion.

**Anti-pattern — snowflake:**
```
Sales ──* Product ──* ProductSubcategory ──* ProductCategory
```
A query grouping by ProductCategory requires 3 joins.

**Preferred — flat star:**
```
Sales ──* Product [ProductKey, ProductName, Subcategory, Category]
```
Category and Subcategory are denormalized into the Product dimension. Single join.

**Detection:** Use `relationship_operations` → List to identify chains of 1→* relationships between dimension tables. If a dimension references another dimension, it's a snowflake candidate.

**Impact:** Each eliminated join removes one SE scan and enables better vertical fusion for measures that reference multiple levels of the hierarchy.

---

### MDL003: Column Cardinality and Data Type Optimization

High-cardinality columns inflate dictionary size and segment memory, slowing SE scans. String columns are especially expensive compared to integer equivalents.

**Patterns:**

- **Integer keys over string keys**: Replace `ProductCode = "PROD-001234"` with integer surrogate keys. Integer dictionaries compress ~10x better and compare faster.
- **Reduce timestamp precision**: If queries only group by date, change `DateTime` columns to `Date` type. This drops cardinality from potentially millions (with time components) to thousands.
- **Bin continuous values**: A `Price` column with 50K distinct decimal values can be binned into ranges (e.g., 0-10, 10-50, 50-100) if the measure logic allows it.
- **Split high-cardinality columns**: A `FullAddress` column with 100K distinct values can be split into `City`, `State`, `Zip` — each with much lower cardinality and independently compressible.

**Detection:** Use `COLUMNSTATISTICS()` via `dax_query_operations` → Execute to identify columns with high distinct counts. Focus on columns that appear in xmSQL `WHERE` clauses or `GROUP BY` — those are the ones actively used in queries.

**Note:** These changes require upstream modifications (Power Query, Lakehouse, Warehouse) and model refresh. Always suggest working on a copy.

---

### MDL004: Composite Model Storage Mode Rules (DUAL Dimensions)

In composite models with user-defined aggregation tables, the storage mode of dimension tables critically affects query efficiency. Using **Import** for dimension tables (instead of DUAL) causes the engine to generate semi-join batch plans with thousands of `UNION ALL` statements in the underlying SQL.

**Correct storage mode configuration:**

| Table Type | Storage Mode |
|---|---|
| Large Fact Table | Direct Query |
| Dimension Tables | **DUAL** (NOT Import!) |
| Aggregation Tables | Import (preferred) or Direct Query |

**Why DUAL and not Import for dimensions:**

A DUAL-mode dimension has two internal versions — Import and Direct Query. The engine automatically picks the right one:

- **Slicer / filter pattern (no joins needed):** Engine uses the Import version — fast, no data source round-trip.
- **Fact table grouping (join required):** Engine uses the DQ version — generates a clean SQL join with the fact table at the data source, returning only the rows needed.

If the dimension is set to **Import only**, the engine cannot use the DQ version for joins. Instead, it must first scan the Import dimension to get all key values, then pass those keys to the fact source as a massive `UNION ALL` derived table (one `UNION ALL` per unique dimension key). For a date table with 3,652 days, this generates a 3,652-row `UNION ALL` block — verbose, slow, and non-scalable.

**Detection:** In Server Timings, look for DQ SQL statements containing patterns like:
```sql
(SELECT 'T' AS [V] WHERE [DateKey] = 20200101)
UNION ALL
(SELECT 'T' AS [V] WHERE [DateKey] = 20200102)
-- ... thousands more
```
This is the semi-join batch plan produced by Import-mode dimensions.

**Trade-off:** DUAL tables are larger in memory (they store the Import copy), but the query performance improvement is almost always worth it. Fact tables must be DQ for user-defined aggregation awareness to function.

---

### MDL005: Aggregation Table Strategies

Pre-summarized aggregation tables intercept SE queries before they reach large DQ fact tables. With Power BI Aggregate Awareness, these redirections happen automatically — no DAX changes needed in existing measures. Target: every visual on an important report page hits an agg table on first load.

**Setup checklist:**
1. Create the agg table via SQL/Power Query: `GROUP BY [ForeignKey1], [ForeignKey2], SUM([Metric])`.
2. Load as Import storage mode.
3. Connect to dimension tables using the same relationship columns as the fact table.
4. In the Manage Aggregations dialog, map each agg column as `SUM OF [FactTable[Column]]`.
5. Set Precedence: higher-precedence tables are checked first (set smaller/faster tables higher).

**Advanced aggregation patterns:**

**Filtered Aggs (hot/cold split):** Import only recent data (e.g., last 3 months) as the "hot" Import table; leave older data in the DQ source. 95%+ of queries are served from the fast Import layer. Use a Dynamic Time Barrier measure to efficiently skip cold-period DQ scans:
```dax
Sum of Quantity =
    SUM ( 'Fact Sale (hot)'[Quantity] )
    + IF (
        MIN ( 'Date'[Date] ) < [Dynamic Time Barrier],
        SUM ( 'Fact Sale (cold)'[Quantity] )
    )
```

**Accordion Aggs (mixed grain):** Combine weekly and monthly summarization boundaries into a single agg table. The boundary dates serve as both week-start and month-start markers. Reduces an 8.3B-row table to 2.5M rows while handling both weekly and monthly period queries. Generate via SQL:
```sql
SELECT B.DateKey AS StartDate, SUM([Metric])
FROM FactTable
    INNER JOIN (SELECT DateKey FROM DimDate WHERE WEEKDAY = 1 OR DAY(DateKey) = 1) AS Boundaries
        ON FactTable.DateKey >= B.DateKey
GROUP BY B.DateKey
```

**Shadow Models (Import tables with agg awareness):** Aggregate Awareness requires the target fact table to use DQ mode. To use agg awareness over an Import fact table, create a 1:1 Import copy of the DQ table (the "shadow") with the same rows and relationships. Configure the shadow as `SUM OF` the DQ table. The agg table can then reference the shadow — indirectly enabling agg awareness over Import.

**Distinct Count via Agg Tables:** Standard agg table DISTINCTCOUNT is not supported in Manage Aggregations. Instead, create an agg table grouped by the key column you want to distinct-count (e.g., `CustomerKey + OrderDateKey`). Each unique CustomerKey now appears in exactly one row per date — making `DISTINCTCOUNT([CustomerKey])` equivalent to `COUNTROWS()` of the agg table when the date filter is applied. Reduces a 3.4-second query to 21 milliseconds.

**Key rule:** Fact tables must be in DQ mode for Aggregate Awareness to work. Use the Shadow Model pattern if you need agg awareness over a large Import fact table.

---

### MDL006: Pre-Compute Period Comparison Columns in the Fact Table

Period-over-period calculations (Year-over-Year, Month-over-Month) typically require **two SE scans** of the fact table: one for the current period and one for the prior period. On billion-row DQ tables, this doubles query cost. Pre-computing the prior-period value as a **physical column** in the fact table reduces two scans to one.

**Pattern:** In the ETL layer, pivot prior-period values into new columns on the same fact row:

| OrderDate | Sales | SalesLY | SalesLM |
|---|---|---|---|
| 2025-01-15 | 500 | 480 | 490 |

The column `SalesLY` stores the same product's sales value from exactly one year ago on the same row.

**DAX — before (two scans):**
```dax
YoY Difference =
    SUM ( Fact[Sales] )
    - CALCULATE ( SUM ( Fact[Sales] ), SAMEPERIODLASTYEAR ( Date[Date] ) )
```

**DAX — after (one scan):**
```dax
YoY Difference =
    SUM ( Fact[Sales] ) - SUM ( Fact[SalesLY] )
```

Both columns are scanned in a single SE event because they reside in the same table with the same filter context.

**Performance impact:** 35% improvement on an 8.3B-row DQ table; from 4 SE scans to 1. Scales with table size and concurrent user load.

**Tradeoffs:** Model becomes physically larger (one extra column per comparison type). ETL must handle "no prior row" cases — generate placeholder rows with null/0 for products that existed last year but not this year. Not practical for fully dynamic period comparisons.

**When to apply:** High-traffic reports with fixed period comparisons (YoY, MoM), especially on DQ tables > 1B rows.

---

### MDL007: Row-Based Time Intelligence Table

Standard DAX time intelligence functions (DATESYTD, SAMEPERIODLASTYEAR, DATEADD) break vertical fusion — each period measure generates its own SE query. On large DQ tables with multiple period columns in a visual, this can multiply SE queries 5–10×. A **row-based Time Intelligence table** pre-materializes all periods as data rows, allowing all period measures to fuse into a single SE scan.

**Data model:** Create a `TimeIntelligence` table with three columns:

| Period | Date | AxisDate |
|---|---|---|
| Current Year | 2025-01-01 | 2025-01-01 |
| Current Year | 2025-01-02 | 2025-01-02 |
| ... (all days in 2025) | | |
| Prior Year | 2024-01-01 | 2025-01-01 |
| Prior Year | 2024-01-02 | 2025-01-02 |
| Last 28 Days | 2025-01-15 | 2025-02-12 |
| YTD | 2025-01-01 | 2025-02-12 |

- `Period` — the label shown to end users (slicer value)
- `Date` — actual dates used to filter the fact table (drive the relationship)
- `AxisDate` — the "anchor" date for visual x-axis alignment

**Relationship options:**
- BiDirectional from TI table to Calendar → Calendar to Fact
- Direct many-to-many from TI table to Fact (slightly faster in testing — 25s vs 29s on 8.2B rows)

**DAX — simple measures work unchanged:**
```dax
-- No time intelligence functions needed in measures
Revenue = SUM ( Sales[Amount] )
Revenue LY = SUM ( Sales[AmountLY] )  -- Or use same measure via relationship
```

The `Period` and `AxisDate` columns replace the date hierarchy in visuals. Filtering by `Period = "Prior Year"` automatically propagates all matching `Date` rows to the fact.

**Performance benchmark (8.2B-row DQ table, 5 measures × 6 periods):**
- Row-based TI: **29 seconds**, 3 SE events
- Standard measure-based TI: **87 seconds**, 20+ SE events

**Key advantage:** All periods share the same relationship and filter context, enabling full horizontal fusion. Adding more period types does not proportionally increase query time.

**When to apply:** Reports that show multiple measures alongside multiple rolling periods. Especially effective on DQ or large Import fact tables where additional SE scans are expensive.

---

### MDL008: Eliminate Referential Integrity Violations

Analysis Services tracks **referential integrity (RI) violations** — fact table rows whose foreign key values have no matching row in the dimension table. When RI violations exist in an Import model, the engine cannot apply certain query plan optimizations (such as inner-join rewriting for SWITCH and multi-measure patterns). Eliminating violations can halve query time with no other changes.

**What causes RI violations:** A fact row references `ProductKey = 4` but no row with that key exists in the Product dimension. This is common when dimensions are filtered to active/recent records while the fact is unfiltered.

**Detection in DAX Studio:**
```dax
SELECT [Dimension_Name], [RIVIOLATION_COUNT]
FROM $SYSTEM.DISCOVER_STORAGE_TABLES
WHERE [RIVIOLATION_COUNT] > 0
```

**Find the missing keys:**
```dax
EVALUATE
    CALCULATETABLE (
        VALUES ( 'Fact Sales'[Product ID] ),
        ISBLANK ( 'Dim Product'[Product ID] )
    )
```

**Fix:** Add the missing key values to the dimension table (add an "Unknown" catch-all row for unresolvable keys). Do not remove fact rows — they represent real transactions.

**Impact:** In benchmarks, adding a few rows to resolve RI violations reduced query time from ~4 seconds to ~2 seconds for complex SWITCH/multi-measure expressions.

**Note:** This rule applies **only to Import models**. Direct Query models are not affected.

---

### MDL009: Replace SEARCH/FIND Filters with Pre-Computed Boolean Columns

Visual-level "contains" filters and DAX measures using `SEARCH()` or `FIND()` force the engine to scan string values row-by-row at query time — bypassing VertiPaq's compressed columnar access. Pre-computing the boolean result as a **calculated (or physical) column** pushes the work to refresh time and reduces query-time cost by 100×+.

**Anti-pattern — SEARCH filter in report or DAX:**
```dax
// Generated by "Advanced Filter > contains 'DBA'" on Fact Sale[Description]
FILTER (
    VALUES ( 'Fact Sale'[Description] ),
    NOT ( SEARCH ( "DBA", 'Fact Sale'[Description], 1, 0 ) >= 1 )
)
```
On 228K rows: **1,108 ms**

**Optimized — pre-computed boolean column:**
```dax
// Calculated column added to Fact Sale, computed once at refresh
IsExcluded =
    OR (
        SEARCH ( "DBA", 'Fact Sale'[Description], 1, 0 ) >= 1,
        SEARCH ( "Joke", 'Fact Sale'[Description], 1, 0 ) >= 1
    )
```
Apply a simple `= TRUE` filter on `[IsExcluded]` in the visual filter panel instead.

On 228K rows: **10 ms** — **110× faster**.

**Why so fast:** The boolean column has cardinality 2 (TRUE/FALSE), compresses to 1 bit per row (~7 KB total), and is accessed via VertiPaq's optimized columnar path rather than row-by-row string scanning.

**Generalization:** The same technique applies to any fixed-value logical test that repeats across many rows: relative date flags (`IsLast28Days`), category indicators, regex-like prefix/suffix checks. Move the computation to the column; keep the query-time filter simple.

**Tradeoff:** The filter is static (computed at refresh). End users cannot change the text being searched. Use only when the filter value is authored and fixed, not user-driven.

---

### MDL010: Promote Calculated Columns to Physical (Source) Columns

DAX calculated columns are evaluated **during refresh** (the recalc phase), which requires cross-partition data access and HASH dictionary management. This consumes more memory than loading pre-computed physical columns from the source. Converting calculated columns to physical columns reduces refresh memory pressure and can also improve query performance via better VertiPaq compression.

**Anti-pattern — calculated column in model:**
```dax
// Calculated column
Total = 'Fact Sales'[Price] * 'Fact Sales'[Quantity]
```
The recalc phase for this column must reference the Price and Quantity dictionaries across all partitions, consuming excess working memory.

**Preferred — pre-compute in source:**
```sql
-- SQL / Power Query
SELECT Price * Quantity AS Total FROM FactSales
```
The value arrives already computed; AS just compresses it.

**When to apply:**
- Calculated columns with simple arithmetic (`Price * Qty`, `YEAR([Date])`, `LEFT([Code], 3)`)
- Large tables with many partitions where recalc memory spikes during refresh
- Columns that do not require DAX context transitions (i.e., they only reference columns in the same row)

**When NOT to apply:** Columns that reference related tables (`RELATED()`), use DAX aggregations across rows, or rely on measures — these cannot be simply replicated in SQL.

**Memory impact:** On large models (billion+ rows, multiple partitions), this change can halve peak refresh memory, reducing OOM errors and capacity unit consumption.

---

### MDL011: Calendar Window Bridge Table for Rolling Distinct Counts

For rolling distinct count measures evaluated over many date points (e.g., 28-day rolling unique entity count displayed for every day over 6 months), the standard approach generates one SE scan per date point — typically 150–200 SE queries. Even with the FE materialization technique (DAX026), the single datacache can be very large when the fact table is wide.

A **Calendar Window bridge table** pre-computes the date-window membership at the data layer, replacing the rolling window logic entirely with a relationship join. Instead of the engine computing which dates fall within a 28-day window at query time, it looks that up from a pre-built many-to-many bridge.

**How it works:**

Create a bridge table with two columns:
- `WindowDate` — the "anchor" date (the date shown on the visual x-axis)
- `MemberDate` — each of the N dates that fall within that window

For a 28-day window, each WindowDate has 28 MemberDate rows. Additional columns like `WindowDaySize` (7, 28, etc.) and `WindowDayShift` (0, -365 for prior year) allow reuse across multiple window types.

```python
# PySpark — generate the CalendarWindow bridge table
spark.sql("""
    CREATE OR REPLACE TEMPORARY VIEW vw28DayWindowBase AS
    SELECT
        CalendarDate AS WindowDate,
        DATE_ADD(CalendarDate, Number) AS MemberDate
    FROM DIM_Calendar
    CROSS JOIN ( SELECT EXPLODE(SEQUENCE(0, 27)) AS Number )
""")

df = spark.sql("SELECT *, 28 AS WindowDaySize, 0 AS WindowDayShift FROM vw28DayWindowBase")
df.write.mode("overwrite").format("delta").option("vorder", "true").saveAsTable("BRIDGE_CalendarWindow")
```

Relate the bridge to the fact table via a many-to-many relationship on the date key (`MemberDate → Fact[DateKey]`).

**DAX measure using the bridge:**
```dax
MEASURE Fact[Rolling28DayCount] =
    VAR _MaxDate =
        CALCULATE ( MAX ( Fact[DateKey] ), ALLEXCEPT ( Fact, Calendar ), ALLSELECTED ( CalendarWindow ) )
    VAR _ValidWindows =
        FILTER (
            ALLSELECTED ( CalendarWindow[WindowDate] ),
            CalendarWindow[WindowDate] <= _MaxDate
        )
    RETURN
        CALCULATE (
            SUMX ( DISTINCT ( Fact[EntityKey] ), 1 ),
            CalendarWindow[WindowDaySize] = 28,
            CalendarWindow[WindowDayShift] = 0,
            KEEPFILTERS ( CalendarWindow[WindowDate] IN _ValidWindows ),
            REMOVEFILTERS ( Calendar )
        )
```

**Result:** The engine generates a semi-join batch plan — one SE batch query instead of 150+ individual daily scans. On a 250M-row fact table: 508 seconds (initial) → 49 seconds (FE materialization, DAX026) → **2 seconds** (Calendar Window bridge).

**When to apply:** Use when rolling distinct counts are a high-traffic measure on large fact tables, and DAX026 alone is not fast enough. Requires ETL to generate and maintain the bridge table.

**Tip:** Add `WindowDayShift` column to support prior-period comparisons without additional bridge tables — filter `WindowDayShift = -365` for prior year, `-28` for prior month, etc.

---

### MDL012: Cardinality Reduction via Historical Value Substitution

High-cardinality key columns (natural keys, surrogate keys with millions of distinct values) are among the largest contributors to model memory size and scan cost. In many cases, stakeholders need *recent* key values for drill-through or research, but do not need granular keys for historical records — they just need the historical aggregates.

When this is true, **replacing historical key values with a single placeholder** dramatically reduces column cardinality, shrinks the dictionary, and improves both compression and scan speed.

**Pattern:**

For any period older than the retention window, replace the key column value with a generic default:

| SalesKey | SaleDate | Amount |
|---|---|---|
| SK-00001 | 2025-03-15 | 500 | ← keep actual key (recent)
| SK-00002 | 2025-02-01 | 300 | ← keep actual key (recent)
| *Historical Key* | 2022-06-10 | 200 | ← replaced with placeholder
| *Historical Key* | 2019-11-03 | 150 | ← replaced with placeholder

In Power Query or SQL ETL:
```sql
SELECT
    CASE
        WHEN SaleDate >= DATEADD(year, -1, GETDATE()) THEN SalesKey
        ELSE 'Historical Key'
    END AS SalesKey,
    SaleDate,
    Amount
FROM FactSales
```

**Impact:** A SalesKey column with 10M distinct values becomes a column with ~1M recent values + 1 placeholder. Cardinality drops by 90%+, dictionary size collapses, and memory footprint follows.

**Example results:** SalesKey column: 94.94 MB → 6.74 MB. Total model: 168.42 MB → 85.39 MB (49% reduction from key column alone).

**Extend to dimension attributes:** The same technique applies to dimension attributes outside the retention window. Customer address data older than 1 year can be replaced with "Historical Address"; customer rows with no recent sales can be consolidated into a single "Historical Customer" member:

```sql
-- Remap old fact rows to a single historical customer
UPDATE FactSales
SET CustomerKey = 'HIST'
WHERE SaleDate < DATEADD(year, -1, GETDATE())

-- Remove dimension rows for customers with no recent sales  
DELETE FROM DimCustomer
WHERE CustomerKey NOT IN (SELECT DISTINCT CustomerKey FROM FactSales)
    AND CustomerKey <> 'HIST'
```

**When to apply:**
- Model is approaching capacity memory limits
- A single key column is > 20% of total model size
- Stakeholders confirm they only need recent granular keys; historical records just need aggregate totals
- Not applicable when granular historical keys are required for reporting or drill-through (talk to stakeholders — requirements often have flexibility)

**Detection:** Use DAX Studio's VertiPaq Analyzer or `COLUMNSTATISTICS()` to identify columns with highest cardinality and memory consumption. Key columns near the top of that list are the primary targets.

---

### Tier 4: Direct Lake Optimization Patterns

These patterns apply specifically to **Direct Lake** semantic models in Microsoft Fabric, where data is read directly from OneLake Delta Parquet files rather than imported. Direct Lake provides Import-like query speed when data is resident in memory, but has unique performance characteristics around cold cache, segment loading, and DQ fallback.

### DL001: V-Ordering for Optimal VertiPaq Compression

Delta Parquet files written without V-ordering do not compress as well in Direct Lake as imported data. **V-ordering** is a Fabric Spark write optimization that reorders rows within each rowgroup to maximize VertiPaq's RLE compression — the same row shuffling that Import models apply at refresh time.

**Key fact:** Import models are always V-ordered. Direct Lake models are **NOT automatically V-ordered**. You must explicitly enable it.

**Enable V-ordering in Spark:**
```python
# In a Fabric Notebook (PySpark)
spark.conf.set("spark.microsoft.delta.optimizeWrite.enabled", "true")
spark.conf.set("spark.microsoft.delta.vorder.enabled", "true")

df.write.format("delta").option("vorder", "true").saveAsTable("MyLakehouseTable")
```

**Or run OPTIMIZE with V-ORDER on an existing table:**
```python
spark.sql("OPTIMIZE MySchema.MyTable VORDER")
```

**Impact:** V-ordering on a fact table with a sorted key column (e.g., DateKey) can improve segment compression by 2–5× and reduce cold-cache load time because smaller segments load faster.

---

### DL002: Segment Size and Parallelism

VertiPaq stores data in **segments** — the unit of both compression and parallel processing. One segment processes per CPU core. Larger Fabric capacities have more cores (F64: 10×, F256: 40× parallelism), so models with more segments saturate the available CPU better.

**Optimal segment size:** Target **1–16 million rows per segment** depending on table size. Too few rows per segment → too many tiny segments, excess merge overhead. Too many rows → too few segments, CPU underutilized.

**Direct Lake segment sizing:** In Direct Lake, Delta rowgroups map directly to VertiPaq segments. Spark defaults to 1M rows per rowgroup; this can be adjusted:
```python
spark.conf.set("spark.databricks.delta.optimizeWrite.targetFileSizeBytes", "134217728")  # 128MB
```

**What determines segment row count:**
- Delta rowgroup size (Spark write config)
- `OPTIMIZE` compaction — run regularly to merge small files into target sizes
- V-ordering — affects compression ratio, indirectly affects rows/MB

**Columnar parallelism:** Because segments are the parallel unit, queries on tables split into 16 segments can use 16 cores simultaneously on an F64 capacity. Tables with only 1–2 segments are effectively single-threaded.

---

### DL003: Prevent Direct Query Fallback

Direct Lake can fall back to Direct Query (20–30% slower) when it cannot serve the query from the in-memory columnar store. DQ fallback should be avoided for hot paths.

**Conditions that trigger DQ fallback:**

| Cause | Fix |
|---|---|
| Capacity guardrails hit (row limit, memory) | Increase capacity or reduce model size |
| RLS/OLS defined as SQL on Warehouse/SQL endpoint | Define RLS in the model metadata, not the source SQL |
| Model not reframed (pointing to specific Delta version) | Re-frame the model after major data loads |
| Table points to a View instead of a Lakehouse table | Always use native Lakehouse tables, not SQL views |
| `DirectLakeOnly` mode disabled (default) | Enable DirectLakeOnly to prevent fallback (WARNING: queries fail instead of falling back) |

**Check whether your model is falling back:**
In DAX Studio Server Timings, DQ fallback events appear as SQL subclass events similar to standard Direct Query traces. If you see SQL-subclass SE events in a Direct Lake model, the model fell back.

---

### DL004: Cold vs Warm vs Hot Cache Testing

Direct Lake query performance depends heavily on cache state. Benchmark under the correct cache condition for your use case.

**Four cache states:**

| State | Description | How to achieve |
|---|---|---|
| **Cold** | Columns not loaded; dictionary and join indexes not built | Refresh model → run `EVALUATE {1}` → run query |
| **Warm** | Columns in memory, VertiScan query cache empty | Refresh → run query (loads columns) → Clear Cache → run `EVALUATE {1}` → run query |
| **Mixed** | Some columns cold, some warm | Target specific columns with pre-queries before clearing |
| **Hot** | Columns in memory + VertiScan query results cached | Run same query twice; 2nd run is hot |

**Cold cache is the worst case** — every column referenced must be loaded from OneLake on first access. The entire column is loaded (no partition pruning), and the dictionary + join indexes are rebuilt from scratch. This is what users experience on the first query after a model refresh.

**Check which segments are loaded:**
```dax
SELECT * FROM $SYSTEM.DISCOVER_STORAGE_TABLE_COLUMN_SEGMENTS
```

**Pre-warming strategy:** After refresh, run queries touching the most-used columns to populate them in memory before users arrive. This converts cold-start penalties into warm-speed queries for high-traffic models.

---

## Section 5: Real-World Optimization Examples

### Example 1: Context Transition in FILTER

**Original (Memory Overflow):**
```dax
DEFINE MEASURE 'Fabric Capacity Units NRT'[MyMeasure] =
    VAR _ActiveSeconds = [NRT Thirty Second Windows]
    VAR _TotalSeconds =
        CALCULATE (
            COUNTROWS (
                SUMMARIZE (
                    'Fabric Capacity Units NRT',
                    'Fabric Capacity Units NRT'[DIM_CalendarKey],
                    'Fabric Capacity Units NRT'[Capacity Id]
                )
            ),
            FILTER (
                'Fabric Capacity Units NRT',
                [NRT Thirty Second Windows] > 0  -- Measure reference causes context transition
            )
        ) * 2880
    VAR _Result = DIVIDE ( _ActiveSeconds, _TotalSeconds )
    RETURN _Result
```

**Analysis:** The `[NRT Thirty Second Windows]` measure reference in FILTER triggers context transition, causing 7.7M+ row materialization and memory overflow.

**Optimized (1.8 seconds):**
```dax
DEFINE MEASURE 'Fabric Capacity Units NRT'[MyMeasure] =
    VAR _ActiveSeconds = [NRT Thirty Second Windows]
    VAR _TotalSeconds =
        CALCULATE (
            COUNTROWS (
                SUMMARIZE (
                    'Fabric Capacity Units NRT',
                    'Fabric Capacity Units NRT'[DIM_CalendarKey],
                    'Fabric Capacity Units NRT'[Capacity Id]
                )
            ),
            'Fabric Capacity Units NRT'[ThirtySecondWindow] > 0  -- Direct column filter
        ) * 2880
    VAR _Result = DIVIDE ( _ActiveSeconds, _TotalSeconds )
    RETURN _Result
```

**Result:** Memory overflow → 1.8 seconds. Replace measure reference with direct column filter.

### Example 2: Cross-Join Materializations

**Problem:** 91.7 seconds execution (98% FE / 2% SE) with 5+ million row materialization

**Solution Strategy:**
- Replace ALL() + manual filter restoration with ALLEXCEPT()
- Combine logic to reduce table scans from 2 to 1
- Use MAX() instead of SELECTEDVALUE()
- Eliminate unnecessary DISTINCT() operations

**Result:** 91.7 seconds → 4.7 seconds (95% improvement)

### Example 3: Nested SUMMARIZE → ADDCOLUMNS → FILTER

**Problem:** 39.9 seconds with CALCULATE(DIVIDE()) in iterators causing CallbackDataID

**Solution:**
- Replace SUMMARIZE with SUMMARIZECOLUMNS
- Use direct column arithmetic instead of DIVIDE() function
- Eliminate nested iterator patterns
- Use INT() for boolean conversion

**Result:** 39.9 seconds → 0.76 seconds (98% improvement)

### Example 4: Variable Caching for Repeated Measures

**Problem:** Repeated measure evaluation causing duplicate SE scans

**Solution:**
```dax
SUMX (
    Capacity,
    VAR _CUHours = [CU Hours]
    RETURN IF ( _CUHours > 1000000, _CUHours )
)
```

**Advanced — SUMMARIZECOLUMNS base table:**
```dax
SUMX (
    SUMMARIZECOLUMNS (
        Capacity[DIM_CapacityId],
        "@Calculation", VAR _CUHours = [CU Hours] RETURN IF ( _CUHours > 1000000, _CUHours )
    ),
    [@Calculation]
)
```

**Result:** 3.8s → 2.5s → 1.5s (60% cumulative improvement)
