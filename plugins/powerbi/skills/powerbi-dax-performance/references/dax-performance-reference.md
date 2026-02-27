# DAX Optimization Patterns Reference

A comprehensive catalog of DAX anti-patterns, optimization strategies, and trace analysis guidance.

## Table of Contents

- [Section 1: How the Engine Works](#section-1-how-the-engine-works)
  - [Query Processing Architecture](#query-processing-architecture)
  - [xmSQL: The Storage Engine Query Language](#xmsql-the-storage-engine-query-language)
  - [Compression, Segments, and Parallelism](#compression-segments-and-parallelism)
  - [SE Query Fusion](#se-query-fusion)
- [Section 2: Reading and Diagnosing Traces](#section-2-reading-and-diagnosing-traces)
  - [Understanding FE vs. SE Metrics](#understanding-formula-engine-fe-vs-storage-engine-se-metrics)
  - [Analyzing Trace Events](#analyzing-trace-events)
  - [What to Look For](#what-to-look-for)
  - [DAX vs. Data Layout: Reading the Signal](#dax-vs-data-layout-reading-the-signal)
- [Section 3: Tier 1–2: Query Optimization](#section-3-tier-12-query-optimization)
  - [Tier 1: DAX Optimization Patterns](#tier-1-dax-optimization-patterns)
  - [Tier 2: Query Structure Patterns](#tier-2-query-structure-patterns)
- [Section 4: Tier 3–4: Model and Data Layout](#section-4-tier-34-model-and-data-layout)
  - [General Data Layout Best Practices](#general-data-layout-best-practices)
  - [Tier 3: Model Optimization Patterns](#tier-3-model-optimization-patterns)
  - [Tier 4: Direct Lake Optimization Patterns](#tier-4-direct-lake-optimization-patterns)

---

## Section 1: How the Engine Works

### Query Processing Architecture

Every DAX query runs through two components: the **Formula Engine (FE)** and the **Storage Engine (SE)**.

The **FE** handles all DAX — branching logic, context transitions, complex arithmetic, measure evaluation. It is **single-threaded** and the bottleneck in most poorly written queries.

The **SE** reads compressed columnar data from VertiPaq. It is **multi-threaded** and very fast, but supports only a limited set of operations: the four basic arithmetic operators, GROUP BY, LEFT OUTER JOINs, and basic aggregations (SUM, COUNT, MIN, MAX, DISTINCTCOUNT). It cannot evaluate DAX functions, measure references, or context transitions.

For **Direct Query models**, the SE role is played by the underlying data source (SQL, Spark, etc.). The FE generates SQL and pushes it down. The trade-off is network and source latency instead of in-memory scan cost.

**How they interact:** The FE requests data from the SE in one or more scans — each result is a **datacache** (a set of columns and aggregated values). Complex queries may require multiple datacaches: one to build a filter set, another to aggregate the fact. When the SE cannot evaluate an expression natively, it **calls back** to the FE row-by-row — making that SE scan effectively single-threaded.

The core principle of DAX optimization: **push as much work as possible into the SE, minimize SE scans, and eliminate callbacks entirely.**

---

### xmSQL: The Storage Engine Query Language

xmSQL is the human-readable representation of SE scan activity in trace events — it shows which tables are scanned, which columns are aggregated, which filters apply, and how joins resolve. Syntax resembles SQL with key differences:

**Implicit GROUP BY:** Every column in the SELECT list is automatically a grouping column — no GROUP BY keyword.

**Computed expressions:** Row-level calculations use a `WITH` block with `:=`, referenced in aggregations via `@`:
```
WITH $Expr0 := ( Sales[UnitPrice] * Sales[OrderQuantity] )
SELECT Product[Category], SUM ( @$Expr0 )
FROM Sales
    LEFT OUTER JOIN Product ON Sales[ProductKey] = Product[ProductKey]
```

**Joins are always LEFT OUTER:** The many-side table is FROM, the one-side is joined in.

**Semi-join projections:** Appear as `DEFINE TABLE $Filter0 ... ININDEX` in xmSQL — an initial dimension scan builds a key index injected into the fact WHERE clause.

**Callbacks:** Occur whenever the SE must compute an expression that exceeds VertiPaq's native capabilities — forcing row-by-row evaluation back in the FE. Example: `IF(Sales[Amount] > 1000, 1, 0)` inside an iterator requires a callback because the SE cannot evaluate conditional logic. Replace with `INT(Sales[Amount] > 1000)` to keep the expression SE-native. See DAX001–DAX007 for callback elimination patterns.

---

### Compression, Segments, and Parallelism

**Compression** determines scan speed. VertiPaq uses run-length encoding (RLE) and dictionary encoding. **V-ordering** reorders rows within segments to maximize RLE compression. Import models are V-ordered automatically. Direct Lake models are **not** — enable V-ordering explicitly (see DL001).

**Segments** are fixed-size column chunks — the unit of both compression and parallel execution. The SE assigns one CPU thread per segment, so segment count determines how many cores a scan can utilize.

**Parallelism:** A 32M-row table in 2 segments uses 2 threads; in 32 segments it uses all 16 available threads — a 4–8× speedup with zero DAX changes.

**Segment skew matters equally:** if one segment has 15M rows and the rest have 1M, the scan bottlenecks on the oversized segment. Segments must be evenly sized for parallelism to be effective.

**Diagnosing low parallelism:** The **SE Parallelism Factor** (StorageEngineCpuTime ÷ StorageEngineDuration) shows thread utilization. Values near 1.0 mean single-threaded execution; values of 8–16 indicate strong multi-core use. When a trace shows few SE queries (1–4), high SE Duration, Parallelism Factor ≈ 1.0, and clean xmSQL — the bottleneck is too few segments or skewed segment sizes. This cannot be fixed with DAX; the fix is data layout (see General Data Layout Best Practices and DL001–DL002).

---

### SE Query Fusion

Fusion is the engine's ability to combine multiple SE scans into fewer scans. There are two types:

**Vertical fusion** merges multiple measure aggregations that share the same filter context into a single SE query. Three measures on the same fact table under the same filter = one scan instead of three. Gain scales with fact table size.

**What blocks vertical fusion:**
- **Time intelligence functions** (DATESYTD, DATEADD, SAMEPERIODLASTYEAR) — each TI-modified measure needs its own date-filtered SE scan
- **DAX variables building date ranges** — manual date predicates block fusion the same way as built-in TI
- **SWITCH/IF selecting between measures** — engine cannot determine at plan time which aggregation to include
- **Calculation group items** applying different filter modifications — each generates its own SE query

**Horizontal fusion** merges SE queries that differ only in which single value of a column they filter. N separate fact scans collapse to one; the FE partitions the result.

**What blocks horizontal fusion:**
- **Filtered column not in groupby** — engine cannot merge slices if the slicing column is absent from the groupby
- **Table-valued filter per measure** (e.g., time intelligence) — prevents slice merging even when column filters are identical
- **Filter value computed at runtime** (stored in a variable) — engine treats it as dynamic and will not fuse

**Trace diagnosis:** Multiple SE queries hitting the same fact table with same joins → vertical fusion blocked. N near-identical SE queries with only the WHERE filter differing → horizontal fusion blocked. See DAX patterns and Section 2 trace analysis.

---

## Section 2: Reading and Diagnosing Traces

### Understanding Formula Engine (FE) vs. Storage Engine (SE) Metrics

When server timings are returned as `CalculatedExecutionMetrics`, use these raw field names:

| Metric | Raw field | Description | Target |
|--------|-----------|-------------|--------|
| **TotalDuration** | `totalDuration` | End-to-end query time (ms) | Lower is better |
| **FormulaEngineDuration** | `formulaEngineDuration` | Single-threaded FE processing time (ms) | Lower is better |
| **StorageEngineDuration** | `storageEngineDuration` | Multi-threaded SE query time (ms) | Higher % of total is better |
| **StorageEngineQueryCount** | `storageEngineQueryCount` | Number of SE queries generated | Fewer is better |
| **StorageEngineCpuTime** | — | Total CPU across all SE threads | Higher ratio to SE Duration is better |
| **VertipaqCacheMatches** | `vertipaqCacheMatches` | Cache hits (SE queries answered from memory) | Only relevant on warm cache |
| **SE Parallelism Factor** | `storageEngineCpuFactor` | CpuTime ÷ Duration | Higher is better |
| **FE %** | `formulaEngineDurationPercentage` | FE Duration ÷ Total Duration | Lower is better |
| **SE %** | `storageEngineDurationPercentage` | SE Duration ÷ Total Duration | Higher is better |

> **Net wall-clock:** StorageEngineDuration is the *union* of overlapping SE intervals — not the sum of individual durations. Three concurrent 100ms scans = ~100ms wall clock, not 300ms.

**Parallelism — aggregate vs. per-scan:** `storageEngineCpuFactor` is the aggregate parallelism factor. When per-scan events are available, each scan has its own `cpuTime / duration`. A healthy aggregate factor can mask a single unparallelized scan where `cpuTime ≈ duration`.

**FE processing gaps:** `formulaEngineDuration` is the sum of all time intervals where no SE query was executing — gaps between SE events on the timeline.

### Analyzing Trace Events

**Pre-processed EventDetails waterfall (execute_dax_query):**
When trace events are returned as a pre-processed `EventDetails` array, the structure interleaves FE gap blocks between SE query events. Each SE entry includes: `type`, `subclass` (VertiPaqScan / BatchVertiPaqScan / DirectQuery), `duration_ms`, `cpu_ms`, `rows`, `kb`, `query` (xmSQL text), `par` (cpuTime/duration for that individual scan), and `timeline` (start/end offsets in ms from query start + relative percentages). FE entries have `type: "FE"` and `duration_ms` only. Find the SE entry with the highest `duration_ms` — that is the primary optimization target. Check its `par` value: low `par` means single-threaded despite healthy aggregate SE_Par. Check its `query` field first for `CallbackDataID` or `EncodeCallback` before drawing any other conclusion.

**Raw trace events (when EventDetails is not available):**
Fetch `VertiPaqSEQueryEnd` events directly. Key events to examine:

**Storage Engine Events (VertiPaqSEQueryEnd):**
- **TextData**: The xmSQL query sent to the storage engine. Look for:
  - **CallbackDataID**: FE callback — row-by-row evaluation
  - **EncodeCallback**: Grouping by calculated expressions instead of physical columns
  - **`[Estimated size (volume, marshalling bytes): X, Y]`** appended at the end of TextData — X is rows scanned, Y is marshalling bytes (divide by 1024 for KB). These are not separate event fields; parse them from TextData.
- **Duration**: How long this individual SE query took
- **CpuTime**: CPU time for this SE query

### What to Look For

Scan for these signals in priority order when analyzing a slow query:

1. **Callbacks** — `CallbackDataID` or `EncodeCallback` in SE TextData. Fix first (DAX001–DAX007).
2. **High FE %** — FE doing too much work; usually paired with many short SE queries.
3. **High SE query count / repeated fact scans** — multiple SE queries hitting the same fact table with same joins but different WHERE clauses or aggregations → blocked fusion. See SE Query Fusion.
4. **Large materializations** — SE rows far exceed final result, or SE queries with no WHERE clause → FE filtering post-materialization instead of pushing to SE. See DAX009.
5. **Low parallelism factor** — near 1.0 on slow scans → data layout problem, not DAX. See Compression, Segments, and Parallelism.
6. **High KB per SE event** — wide intermediate tables; reduce columns or aggregate earlier.
7. **Two-step dimension pre-scans** — dimension-only SELECT followed by `where predicate` on the fact. Restructure query to collapse into one scan.
8. **Large semi-join index tables** — `DEFINE TABLE` + `ININDEX` where the index contains thousands of rows. 

**Prioritization:** Callbacks → Large FE processing → SE query count (DAX) → parallelism and data volume (data layout). Target the highest-duration SE scan first — ignore 0ms cache-hit scans.

---

### DAX vs. Data Layout: Reading the Signal

**Many SE queries + high FE time + individually short SE scans → DAX problem**

Fusion is blocked, callbacks are present, or filters resolve iteratively. Fix the DAX — see Tier 1 and Tier 2 patterns. *Example:* 109 SE queries, 30% FE → after restructuring: 4 SE queries, 1% FE.

**Few SE queries + low FE time + high SE duration + low parallelism → Data layout problem**

The DAX is clean but SE scans are slow due to insufficient segments or poor compression. DAX changes will not help — see General Data Layout Best Practices and DL001–DL002.

---

## Section 3: Tier 1–2: Query Optimization

### Tier 1: DAX Optimization Patterns

### DAX001: Replace ADDCOLUMNS/SUMMARIZE with SUMMARIZECOLUMNS

SUMMARIZECOLUMNS defines grouping + calculation in one step, enabling better SE fusion. Replace all ADDCOLUMNS/SUMMARIZE patterns.

**Anti-patterns:**
```dax
SUMMARIZE ( Sales, Sales[ProductKey], "Total Profit", [Profit] )
ADDCOLUMNS ( SUMMARIZE ( Sales, Sales[ProductKey] ), "Total Profit", [Profit] )
ADDCOLUMNS ( Sales, "Total Profit", CALCULATE ( [Profit] ) )
ADDCOLUMNS ( VALUES(Sales[ProductKey]), "Total Profit", [Profit] )
```

**Preferred:**
```dax
SUMMARIZECOLUMNS ( Sales[ProductKey], "Total Profit", [Profit] )
```

---

### DAX002: Cache Repeated and Context-Independent Expressions in Variables

Evaluating the same measure multiple times or placing context-independent expressions inside iterators causes redundant SE queries. Cache in a variable.

**Anti-pattern — repeated measure reference:**
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

**Anti-pattern — context-independent expression inside iterator:**
```dax
SUMX( Sales, Sales[Quantity] * [Average Price] * 1.1 )
// [Average Price] doesn't change per row
```

**Preferred:**
```dax
VAR _AvgPrice = [Average Price]
RETURN SUMX( Sales, Sales[Quantity] * _AvgPrice * 1.1 )
```

---

### DAX003: Remove Duplicate / Redundant Filters

Applying the same filter condition twice — whether as duplicate CALCULATE arguments or as a variable that restates an existing predicate — causes redundant SE evaluation.

**Anti-pattern — same predicate in CALCULATE + FILTER:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    Sales[Year] = 2023,
    FILTER(Sales, Sales[Year] = 2023)
)
```

**Anti-pattern — redundant filter variable:**
```dax
VAR FilteredValues = CALCULATETABLE ( DISTINCT ( Table[Key1] ), Table[Amount] > 1000 )
VAR Result =
    CALCULATETABLE (
        SUMMARIZECOLUMNS ( Table[Key2], "TotalQty", SUM ( Table[Quantity] ) ),
        Table[Amount] > 1000,
        Table[Key1] IN FilteredValues  -- redundant: already filtered by Amount > 1000
    )
```

**Preferred — single filter, no duplication:**
```dax
CALCULATE( SUM(Sales[Amount]), Sales[Year] = 2023 )

VAR Result =
    CALCULATETABLE (
        SUMMARIZECOLUMNS ( Table[Key2], "TotalQty", SUM ( Table[Quantity] ) ),
        Table[Amount] > 1000
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

### DAX005: Pre-Materialize Context Transitions with SUMMARIZECOLUMNS

Materializing context transition results in SUMMARIZECOLUMNS and iterating over pre-calculated values often improves efficiency.

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

### DAX008: Wrap SUMMARIZECOLUMNS Filters with CALCULATETABLE

Filters passed as direct arguments to SUMMARIZECOLUMNS inside measures can produce unexpected results or block optimizations. Move filters to a wrapping CALCULATETABLE instead.

**Anti-pattern:**
```dax
SUMMARIZECOLUMNS (
    Table[Column],
    TREATAS ( { "Value" }, Table[FilterColumn] ),
    "@Calculation", [Measure]
)
```

**Preferred:**
```dax
CALCULATETABLE (
    SUMMARIZECOLUMNS (
        Table[Column],
        "@Calculation", [Measure]
    ),
    Table[FilterColumn] = "Value"
)
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

### DAX010: Use Simple Column Filter Predicates as CALCULATE Arguments

CALCULATE accepts simple boolean predicates directly — filter columns not tables. Split `&&` into separate filter arguments.

**Anti-pattern:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    FILTER(Product, Product[Category] = "Electronics")
)
```

**Preferred:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    KEEPFILTERS(Product[Category] = "Electronics")
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

### DAX011: Distinct Count Alternatives

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

### DAX012: Replace SELECTEDVALUE with MAX/MIN

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

### DAX013: Use ALLEXCEPT Instead of ALL + VALUES Restoration

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

### DAX014: Flatten Nested CALCULATE Calls

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

### DAX015: SWITCH/IF Branch Optimization in SUMMARIZECOLUMNS

SWITCH/IF inside SUMMARIZECOLUMNS enables branch optimization — the engine evaluates only the matching branch. When this fails, it materializes a full cartesian product. Three things break it:

1. **Multiple aggregations in one branch** — merge into single SUMX: `SUMX(Sales, Sales[SalesAmount] - Sales[TotalCost])`
2. **Mismatched data types across branches** — explicitly match: `CONVERT(SUM(Sales[OrderQuantity]), CURRENCY)`
3. **Measure references inside iterator branches** — cache before SWITCH: `VAR _UnitDiscount = [Unit Discount]`

---

### DAX016: Use COUNTROWS Instead of DISTINCTCOUNT on Key Columns

When a column is a primary key (one-side of a relationship), COUNTROWS avoids the distinct count algorithm entirely.

**Anti-pattern:**
```dax
DISTINCTCOUNT ( Product[ProductKey] )
```

**Preferred:**
```dax
COUNTROWS ( Product )
```

For non-key columns where DISTINCTCOUNT is a bottleneck, see DAX011 for alternatives.

---

### DAX017: Move Calculation to Lower Granularity

When an iterator scans a high-cardinality table but the calculation depends on a low-cardinality attribute, iterate over the attribute instead.

**Anti-pattern:**
```dax
-- 100K customers but only 5 distinct DiscountRate values → 100K context transitions
SUMX( Customer, CALCULATE(SUM(Sales[Amount])) * Customer[DiscountRate] )
```

**Preferred:**
```dax
-- 5 iterations instead of 100K
SUMX( VALUES(Customer[DiscountRate]), CALCULATE(SUM(Sales[Amount])) * Customer[DiscountRate] )
```

---

### DAX018: Experiment with Relationship Overrides via TREATAS and CROSSFILTER

Relationship direction and filter propagation directly affect SE query plans. Sometimes bidirectional is faster; sometimes unidirectional wins. Use TREATAS and CROSSFILTER to experiment without model changes.

**Example — replace bidirectional bridge with explicit filter:**
```dax
CALCULATE(
    SUM(Sales[Amount]),
    CROSSFILTER(Customer[CustomerKey], SportBridge[CustomerKey], NONE),
    TREATAS(VALUES(SportBridge[CustomerKey]), Customer[CustomerKey])
)
```

The only way to know which direction is faster for your model is to test both. TREATAS bypasses auto-exist behavior — always verify semantic equivalence.

---

### DAX019: 1-Column Fusion via MAXX for Same-Column Filter Variants

When multiple measures each filter the same column to different values, the engine issues separate SE queries per value. MAXX with a boolean multiplier forces them into a single SE scan.

**Anti-pattern — separate SE query per filter value:**
```dax
MEASURE Sales[Sales - Bag] = CALCULATE([Base Measure], Sales[Package] = "Bag")
MEASURE Sales[Sales - Box] = CALCULATE([Base Measure], Sales[Package] = "Box")
```

**Preferred — single SE scan via MAXX:**
```dax
MEASURE Sales[Sales - Bag] =
    MAXX(ALL(Sales[Package]), [Base Measure] * (Sales[Package] = "Bag"))
MEASURE Sales[Sales - Box] =
    MAXX(ALL(Sales[Package]), [Base Measure] * (Sales[Package] = "Box"))
```

MAXX iterates all distinct values; the boolean evaluates to 1 for the match and 0 for others. All variants fuse into one SE query. Only works for filters on the **same column**.

---

### DAX020: Replace DIVIDE() with / Operator in Iterators

DIVIDE() includes divide-by-zero protection that forces FE callbacks inside iterators. Use the native `/` operator to keep the expression SE-native.

**Anti-pattern:**
```dax
SUMX(Fact, Fact[BaseAmount] * DIVIDE(RELATED(Items[Discount]), RELATED(Items[LocationAdjustment])))
```

**Preferred:**
```dax
SUMX(Fact, Fact[BaseAmount] * (RELATED(Items[Discount]) / RELATED(Items[LocationAdjustment])))
```

Only use `/` when the denominator is guaranteed non-zero. If zero is possible, pre-filter: `CALCULATETABLE(Items, Items[LocationAdjustment] <> 0)`.

---

### DAX021: Lift Time Intelligence to Outer CALCULATE for Vertical Fusion

TI functions (DATESYTD, DATEADD, etc.) break vertical fusion — each TI-modified measure gets its own SE query. Keep base measures TI-free and apply TI once in an outer wrapper.

**Anti-pattern — each measure applies TI independently (no fusion):**
```dax
MEASURE Sales[Revenue YTD] = CALCULATE ( SUM(Sales[Amount]), DATESYTD(Date[Date]) )
MEASURE Sales[Cost YTD]    = CALCULATE ( SUM(Sales[Cost]),   DATESYTD(Date[Date]) )
```

**Preferred — base measures fuse, TI applied once:**
```dax
MEASURE Sales[Margin YTD] =
    CALCULATE ( [Revenue] - [Cost], DATESYTD ( Date[Date] ) )
```

---

### DAX022: Unblock Horizontal Fusion by Lifting Filters

Horizontal fusion merges SE queries that differ only by column-slice filter. It breaks when the filtered column is missing from groupby, or when table-valued / runtime-computed filters are applied per measure. Fix: keep only simple column-slice filters inside base measures; lift everything else (TI, dynamic variables) to an outer CALCULATE.

**Anti-pattern — TI inside each slice measure (no fusion):**
```dax
MEASURE Sales[Bikes YTD]       = CALCULATE ( SUM(Sales[Amount]), Product[Category] = "Bikes",       DATESYTD(Date[Date]) )
MEASURE Sales[Accessories YTD] = CALCULATE ( SUM(Sales[Amount]), Product[Category] = "Accessories", DATESYTD(Date[Date]) )
```

**Preferred — slice measures fuse, TI applied once:**
```dax
MEASURE Sales[Bikes]       = CALCULATE ( SUM(Sales[Amount]), Product[Category] = "Bikes" )
MEASURE Sales[Accessories] = CALCULATE ( SUM(Sales[Amount]), Product[Category] = "Accessories" )
MEASURE Sales[Combined YTD] = CALCULATE ( [Bikes] + [Accessories], DATESYTD(Date[Date]) )
```

Same principle applies to runtime variable filters — move them to the consuming measure. See DAX019 for the MAXX workaround when the filtered column is not in groupby.

---

### DAX023: Use WINDOW as CALCULATE Filter, Not Inside Iterators

WINDOW inside an iterator (SUMX) is O(n²). As a CALCULATE filter with additive measures, it optimizes to O(n log n).

**Anti-pattern:**
```dax
SUMX ( WINDOW ( 1, ABS, 0, REL, ORDERBY ( Product[ProductCode] ) ), [Sales Amount] )
```

**Preferred:**
```dax
CALCULATE ( [Sales Amount], WINDOW ( 1, ABS, 0, REL, ORDERBY ( Product[ProductCode] ) ) )
```

Only works with additive aggregations (SUM, COUNT, COUNTROWS). Non-additive expressions like DISTINCTCOUNT do not benefit.

---

### Tier 2: Query Structure Patterns

### QRY001: Remove Unneeded Filters

Every filter adds a `WHERE` clause in xmSQL and may force an extra SE join. Users often apply global slicer or visual-level filters that don't actually affect the calculation being optimized.

**Detection:** `WHERE` clauses on columns not used in the measure logic, or filter variables that restrict to a single value (e.g., `Currency[Code] = "USD"` in a USD-only model).

**Fix:** Experiment — remove filters one at a time and re-run. If the result doesn't change, the filter is unnecessary. Global filters that are needed across all visuals should be pushed to the data source or RLS (model-level change — see Tier 3).

```dax
-- Before: filter on Currency adds an SE join for no benefit
SUMMARIZECOLUMNS (
    Product[Category],
    KEEPFILTERS ( TREATAS ( {"USD"}, Currency[Code] ) ),
    "Revenue", [Total Revenue]
)

-- After: filter removed, same result, one fewer SE join
SUMMARIZECOLUMNS ( Product[Category], "Revenue", [Total Revenue] )
```

---

### QRY002: Eliminate Report Measure Filters (__ValueFilterDM)

When a visual filters on a measure value (e.g., "Revenue > 1M"), Power BI generates a `__ValueFilterDM` variable that evaluates the measure twice — once for the filter check, once for display. Roughly doubles execution time.

**Detection:** `__ValueFilterDM` in the generated query.

**Fix:** Move the threshold into the measure itself — return BLANK below the cutoff. SUMMARIZECOLUMNS auto-drops blank rows, achieving the same visual result in one pass:
```dax
MEASURE Sales[Total Revenue Filtered] =
    VAR __Rev = [Total Revenue]
    RETURN IF ( __Rev > 1000000, __Rev )
```

---

### QRY003: Reduce Query Grain

Grouping by a high-cardinality column (e.g., `Calendar[Date]` → 365 rows) when the user only needs monthly data (12 rows) inflates SE row count ~30×.

**Detection:** Groupby on a date or high-cardinality column producing far more rows than the visual needs.

**Option A — coarser groupby:**
```dax
-- Daily → monthly
SUMMARIZECOLUMNS ( 'Calendar'[YearMonth], "Revenue", [Total Revenue] )
```

**Option B — period-end dates only** (keep date grain, skip irrelevant dates):
```dax
SUMMARIZECOLUMNS (
    'Calendar'[Date],
    KEEPFILTERS ( TREATAS ( VALUES ( 'Calendar'[MonthEndDate] ), 'Calendar'[Date] ) ),
    "Revenue", [Total Revenue]
)
```

**Option C — return BLANK for non-boundary dates** (keeps all dates in groupby but only computes on end-of-month):
```dax
MEASURE Sales[Revenue EOM] =
    IF ( MAX('Calendar'[Date]) = EOMONTH(MAX('Calendar'[Date]), 0), [Total Revenue] )
```

**Option D — daily additive measure approximated at coarser grain** (divide monthly total by days in month):
```dax
MEASURE Sales[Daily Avg Revenue] =
    DIVIDE (
        [Total Revenue],
        DAY ( EOMONTH ( MAX('Calendar'[Date]), 0 ) )
    )
```

---

### QRY004: Remove BLANK Suppression (Changes Result Shape)

`+ 0`, `IF(ISBLANK([M]), 0, [M])`, or `COALESCE(..., 0)` force SUMMARIZECOLUMNS to evaluate every groupby combination — including rows with no data — inflating the result set.

**Detection:** `+ 0`, `IF(ISBLANK(...))`, or `COALESCE(..., 0)` appended to measures.

**Anti-pattern:**
```dax
MEASURE Sales[Revenue] = SUM ( Sales[SalesAmount] ) + 0
```

**Preferred:**
```dax
MEASURE Sales[Revenue] = SUM ( Sales[SalesAmount] )
```

**If zeros are required selectively**, conditionally add 0 where it makes sense:
```dax
MEASURE Sales[Revenue] =
    VAR _ForceZero = NOT ISEMPTY ( Sales )
    RETURN [Sales Amount] + IF ( _ForceZero, 0 )
```
---

## Section 4: Tier 3–4: Model and Data Layout

### General Data Layout Best Practices

Data layout decisions affect performance at the source level — before DAX, before the SE. Apply after exhausting DAX optimizations; changes here require ETL or pipeline modifications. Apply to both Import and Direct Lake.

1. **Remove unused columns and filter rows at the source.**
2. **Drop all-null/all-zero fact rows** that never contribute to results.
3. **Move low-cardinality string attributes off the fact table** into dimensions with integer keys.
4. **Partition on high-filter columns** (DateKey, TenantKey) so the engine skips entire files. Use **Z-order clustering** when partitioning creates too many small files.
5. **Presort on the most filtered/grouped column first** (e.g., DateKey, then ProductKey). RLE compression improves dramatically when values cluster into longer runs per segment.
6. **Use optimal data types.** See MDL003.

---

### Tier 3: Model Optimization Patterns

### MDL001: Many-to-Many Relationship Optimization

Bridge tables create expanded tables the engine materializes every query. The right layout depends on filter paths, bridge cardinality, and RLS. Test each option. Scenario: `User` (security), `Customer` (dimension), `UserCustomer` (bridge), `Fact`.

**A — Canonical (bidir bridge):** `User 1──* UserCustomer *──bidir──1 Customer 1──* Fact`
Customer filters Fact directly; bridge only traversed for User. Best when User is rarely a slicer alongside Customer. Bidir causes high FE cost when both filter together.

**B — M2M bridge to fact (no bidir):**
```
User 1──* UserCustomer *──1 Customer
                │
                *──M2M──* Fact
```
Both dims always filter through bridge M2M. Best when consistent query times matter more than peak Customer-only performance.

**C — Optimized hybrid:** `User 1──* UserCustomer *──M2M──* Fact *──1 Customer`
Customer filters Fact directly; User filters through bridge M2M. No bidir. Best general-purpose layout. Use inactive relationship + `USERELATIONSHIP` if you need Customer↔UserCustomer cross-queries.

**D — Pre-computed combination key:** `User 1──* UserCombinations *──M2M──* Fact *──1 Customer`
ETL assigns a surrogate key per unique set of customers a user can access — users with identical access share one key. Best when bridge is very large and many users share the same access patterns.

---

### MDL002: Star Schema Conformance

Snowflake schemas force multiple SE joins per query. Flatten dimension chains into a single wide dimension to reduce join depth and enable better fusion.

`Sales ──* Product ──* Subcategory ──* Category` → `Sales ──* Product [ProductKey, ProductName, Subcategory, Category]`

---

### MDL003: Column Cardinality and Data Type Optimization

High-cardinality columns inflate dictionary size and segment memory.

- **Integer keys over string keys:** Replace `"PROD-001234"` with integer surrogates.
- **Reduce timestamp precision:** `DateTime` → `Date` when queries only group by date.
- **Bin continuous values:** 50K distinct decimals → binned ranges if measure logic allows.
- **Split high-cardinality columns:** `FullAddress` (100K distinct) → `City`, `State`, `Zip`.

---

### MDL004: Aggregation Table Strategies

Pre-summarized Import tables intercept SE queries before they hit large DQ facts. Aggregate Awareness redirects automatically — no DAX changes.

**Setup:** `GROUP BY [FKs], SUM([Metrics])` → load as Import → connect to same dimensions → map in Manage Aggregations as `SUM OF [FactTable[Column]]`. Fact tables must be DQ.

**Filtered Aggs (hot/cold split):** Import only recent data (e.g., last 3 months). 95%+ queries served from Import.

---

### MDL005: Pre-Compute Period Comparison Columns

Period-over-period calcs (YoY, MoM) require two SE scans. Pre-computing prior-period values as physical columns on the fact row reduces it to one scan.

**Before (two scans):**
```dax
YoY = SUM ( Fact[Sales] ) - CALCULATE ( SUM ( Fact[Sales] ), SAMEPERIODLASTYEAR ( Date[Date] ) )
```
**After (one scan):**
```dax
YoY = SUM ( Fact[Sales] ) - SUM ( Fact[SalesLY] )
```

Wider fact table, but eliminates the TI scan entirely. Best for fixed period comparisons on large DQ tables.

---

### MDL006: Row-Based Time Intelligence Table

DAX TI functions break vertical fusion — each period measure gets its own SE query. A row-based TI table pre-materializes all periods as data rows so all period measures fuse into a single SE scan.

**Table:** `Period` (slicer label), `Date` (actual dates → relationship to fact), `AxisDate` (x-axis anchor). Relate via M2M to Fact or BiDir through Calendar.

**Benchmark (8.2B-row DQ, 5 measures × 6 periods):** 87s / 20+ SE events → **29s / 3 SE events**.

---

### MDL007: Eliminate Referential Integrity Violations

Fact FKs with no matching dimension row prevent inner-join rewriting for SWITCH/multi-measure patterns. Import models only.

**Detection:**
```dax
SELECT [Dimension_Name], [RIVIOLATION_COUNT]
FROM $SYSTEM.DISCOVER_STORAGE_TABLES
WHERE [RIVIOLATION_COUNT] > 0
```

**Fix:** Add an "Unknown" catch-all row to the dimension.

---

### MDL008: Replace SEARCH/FIND Filters with Pre-Computed Boolean Columns

`SEARCH()`/`FIND()` in filters forces row-by-row string scanning. Pre-compute the result as a boolean column (cardinality 2, ~1 bit/row) for pure columnar access. Generalizes to any fixed-value logical test — date flags, category indicators, prefix checks.

---

### MDL009: Cardinality Reduction via Historical Value Substitution

Replace old key values beyond a retention window with a single placeholder to collapse cardinality and shrink dictionaries.

```sql
CASE WHEN SaleDate >= DATEADD(year, -1, GETDATE()) THEN SalesKey ELSE 'Historical Key' END
```

---

### MDL010: Set IsAvailableInMDX on Disconnected Slicer Tables

When `IsAvailableInMDX = false` on a disconnected slicer column, SWITCH/IF branches can't be statically resolved — the engine evaluates both branches. Set to `true` so dead branches are eliminated at plan time.

---

### Tier 4: Direct Lake Optimization Patterns

Direct Lake reads from OneLake Delta Parquet files instead of importing. Import-like speed when data is memory-resident, but unique characteristics around cold cache and segment loading.

### DL001: V-Ordering for Optimal VertiPaq Compression

Import models are always V-ordered. Direct Lake models are **not** — enable it explicitly. V-ordering reorders rows within each rowgroup to maximize RLE compression (2–5× improvement).

Two approaches:
- **Spark:** `spark.conf.set("spark.microsoft.delta.vorder.enabled", "true")` then run `OPTIMIZE`.
- **Fabric resource profile:** Use the [`readHeavyForPBI` resource profile](https://learn.microsoft.com/en-us/fabric/data-engineering/configure-resource-profile-configurations) which enables V-ordering and optimized write settings automatically.

---

### DL002: Segment Size and Parallelism

Delta rowgroups map directly to VertiPaq segments — one segment per CPU core. More segments = better CPU saturation (see SE Parallelism Factor in Section 1).

**Target: 1–16M rows per rowgroup.** Too few rowgroups → single-threaded scans; too many tiny rowgroups → merge overhead. For small tables (< 1M rows) this rarely matters. Run `OPTIMIZE` regularly to consolidate small files into properly sized rowgroups.

Maximize available cores by choosing a capacity SKU that matches table size — a table with 2 segments on an F64 wastes most of its parallelism budget.
