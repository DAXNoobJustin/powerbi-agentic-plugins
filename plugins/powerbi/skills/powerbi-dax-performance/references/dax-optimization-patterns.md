# DAX Optimization Patterns Reference

A comprehensive catalog of DAX anti-patterns, optimization strategies, and trace analysis guidance.

## Table of Contents

- [Performance Analysis Framework](#performance-analysis-framework)
- [Trace Analysis Guide](#trace-analysis-guide)
- [DAX Engine Fundamentals](#dax-engine-fundamentals)
- [Optimization Strategy Framework](#optimization-strategy-framework)
- [Anti-Patterns and Optimizations](#anti-patterns-and-optimizations)
- [Real-World Optimization Examples](#real-world-optimization-examples)
- [Query Structure Recommendations](#query-structure-recommendations)
- [Model Optimization Patterns](#model-optimization-patterns)

---

## Performance Analysis Framework

### SUMMARIZECOLUMNS Guidance

SUMMARIZECOLUMNS is now fully supported in measures and can leverage significant performance improvements in many scenarios by providing better fusion optimization opportunities.

**Old Pattern (inefficient):**
```dax
ADDCOLUMNS (
    SUMMARIZE (
        Table,
        Table[Column]
    ),
    "@Calculation",
    [Measure] // Or CALCULATE ( SUM () )
)
```

**New Pattern (optimized):**
```dax
SUMMARIZECOLUMNS (
    Table[Column],
    "@Calculation", [Measure] // Or CALCULATE ( SUM () ) Or SUM ()
)
```

### CRITICAL: Filter Context Application with SUMMARIZECOLUMNS

**WRONG — Filters as direct arguments:**
```dax
// Invalid syntax
SUMMARIZECOLUMNS (
    Table[Column],
    Table[FilterColumn] = "Value",  -- Invalid syntax
    "@Calculation", [Measure]
)
```

```dax
// Valid, but complex syntax
SUMMARIZECOLUMNS (
    Table[Column],
    TREATAS ( {"Value"}, Table[FilterColumn] ),  -- Filter as direct argument
    "@Calculation", [Measure]
)
```

**CORRECT — Wrap SUMMARIZECOLUMNS with CALCULATETABLE:**
```dax
CALCULATETABLE (
    SUMMARIZECOLUMNS (
        Table[Column],
        "@Calculation", [Measure]
    ),
    Table[FilterColumn] = "Value"  -- Filter applied outside
)
```

### Remove Redundant Filter Predicates

Often, complex DAX contains unnecessary intermediate filtering steps that can be eliminated while maintaining semantic equivalence.

**Anti-pattern (Redundant Filtering):**
```dax
VAR FilteredValues =
    CALCULATETABLE(DISTINCT(Table[Key1]), Table[Amount] > 1000)
VAR Result =
    CALCULATETABLE(
        SUMMARIZECOLUMNS(
            Table[Key2],
            "TotalQuantity", SUM(Table[Quantity])
        ),
        Table[Amount] > 1000,
        Table[Key1] IN FilteredValues
    )
```

**Optimized Pattern (Direct Filtering):**
```dax
VAR Result =
    CALCULATETABLE(
        SUMMARIZECOLUMNS(
            Table[Key2],
            "TotalQuantity", SUM(Table[Quantity])
        ),
        Table[Amount] > 1000
    )
```

The intermediate FilteredValues variable is redundant — the `Amount > 1000` filter already restricts the same rows. Applying the filter once is more efficient.

---

## Trace Analysis Guide

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
- **SE Parallelism Factor** = StorageEngineCpuTime / StorageEngineDuration. Values > 1.5 indicate good parallel utilization.
- **FE Percentage** = FormulaEngineDuration / TotalDuration. High FE% indicates optimization opportunities.
- **SE Percentage** = StorageEngineDuration / TotalDuration. Ideally > 70%.

### Analyzing Trace Events

After executing with metrics, fetch the raw trace events to see the execution waterfall. Key events to examine:

**Storage Engine Events (VertiPaqSEQueryEnd):**
- **TextData**: The xmSQL query sent to the storage engine. Look for:
  - **CallbackDataID**: Indicates FE callbacks forcing row-by-row evaluation — this is a major performance bottleneck
  - **EncodeCallback**: Grouping by calculated expressions instead of physical columns
  - Column references (may appear as IDs like `$Column3` — these map to physical columns)
- **Duration**: How long this individual SE query took
- **CpuTime**: CPU time for this SE query
- **Rows returned**: High row counts (>>100K) when the final result is small indicate excessive materializations
- **Size (KB)**: Large values (>1MB) indicate wide materializations (too many columns)

**Formula Engine Events (gaps between SE events):**
- Long gaps between SE queries indicate FE processing.
- Excessive FE/SE alternation (many short SE queries interspersed with FE work) suggests inefficient query plan.

### What to Look For

1. **Callbacks in SE queries**: Search for "CallbackDataID" in SE event TextData — this forces single-threaded FE evaluation
2. **Large materializations**: SE queries returning >>100K rows when final result is much smaller
3. **Many SE queries**: More than 5-10 SE queries suggests poor fusion — the engine can't combine operations
4. **High FE percentage**: > 50% FE indicates the formula engine is doing too much work
5. **Whole table scans**: FILTER(Table, ...) patterns often materialize entire tables

---

## DAX Engine Fundamentals

The DAX engine has two processing components: the **Formula Engine (FE)** and the **Storage Engine (SE)**. Understanding how they interact — and what internal query language the SE uses — is essential for interpreting trace events and identifying optimization opportunities.

### xmSQL: The Storage Engine Query Language

When the FE needs data from the in-memory columnar store, it generates xmSQL queries. These appear in SE trace events and look similar to SQL but have important differences:

**Implicit GROUP BY:** Any column in SELECT automatically groups the result — there is no explicit GROUP BY clause.
```
-- xmSQL (implicit grouping)
SELECT Sales[Region], SUM ( Sales[Amount] )
FROM Sales

-- Equivalent SQL
SELECT Region, SUM(Amount) FROM Sales GROUP BY Region
```

**Expressions:** Row-level arithmetic is declared with `WITH` and referenced via `@`:
```
WITH $Expr0 := ( Sales[Quantity] * Sales[UnitPrice] )
SELECT Sales[Region], SUM ( @$Expr0 )
FROM Sales
```

**Joins:** Model relationships produce `LEFT OUTER JOIN`:
```
SELECT Product[Category], SUM ( Sales[Amount] )
FROM Sales
    LEFT OUTER JOIN Product
        ON Sales[ProductKey] = Product[ProductKey]
```

**Batches:** Multi-step calculations use temporary tables (`DEFINE TABLE`). A common pattern is semi-join filtering where one query builds an index and the next filters against it:
```
DEFINE TABLE $TTable2 :=
SELECT SIMPLEINDEXN ( Product[ListPrice] )
FROM Product
WHERE Product[ListPrice] >= 100

DEFINE TABLE $TTable1 :=
SELECT SUM ( Sales[Amount] )
FROM Sales
    LEFT OUTER JOIN Product
        ON Sales[ProductKey] = Product[ProductKey]
WHERE Product[ListPrice] ININDEX $TTable2[$SemijoinProjection];
```

**Callbacks (performance red flags):**
- **CallbackDataID** — The SE calls back to the FE for row-by-row evaluation. Forces single-threaded processing. Caused by measure references inside iterators, IF/SWITCH inside aggregations, and context transitions.
- **EncodeCallback** — The SE asks the FE to encode a calculated value as a grouping key. Occurs when grouping by expressions that don't map to physical columns.
- **LogAbsValueCallback** — Used internally for PRODUCT/PRODUCTX via logarithmic identity.
- **RoundValueCallback** — Data type conversions the SE cannot perform (e.g., integer to currency).

**When you see callbacks:** Restructure the DAX to eliminate the source — replace measure references with column references, use INT() instead of IF(), cache context-independent values in variables, or pre-materialize with SUMMARIZECOLUMNS.

### xmSQL Pattern Searching

After fetching trace events, systematically search the TextData of SE events for these patterns. Since xmSQL is structured text, plain string/substring matching is sufficient.

**1. Callback detection — highest priority:**
Search for `CallbackDataID` or `EncodeCallback` in any SE event TextData. These force single-threaded FE evaluation and are the most common source of poor performance. Map back to the DAX construct that caused it (see DAX001-DAX006 for elimination patterns).

**2. Isolated dimension scans (filter push indicators):**
Look for SE queries that scan only a dimension table with no fact join — e.g., `SELECT ... FROM Customer` with no `LEFT OUTER JOIN`. This means the FE is building a datacache of dimension keys to use as a filter in a subsequent fact scan, rather than the SE resolving the join directly. Multiple queries where one feeds into another indicate the engine couldn't push the filter down into a single scan.

Pattern: An SE query like:
```
SELECT Customer[CustomerKey] FROM Customer WHERE Customer[Region] = 'West'
```
followed by a second query that uses those keys:
```
SELECT SUM(Sales[Amount]) FROM Sales WHERE Sales[CustomerKey] ININDEX $TTable1[...]
```
This two-step pattern is sometimes unavoidable, but if the dimension filter is simple, restructuring the DAX to let the SE resolve both in one query (via proper relationship traversal) is faster.

**3. Missing fusion opportunities:**
Look for multiple SE queries that scan the same fact table with the same joins and similar `WHERE` clauses but compute different aggregations. This indicates vertical fusion was blocked — see the Vertical Fusion section below for causes and fixes.

Also look for the same measure logic appearing in SE queries with different filter contexts (e.g., same `SUM(Sales[Amount])` but different `WHERE` date ranges). This indicates time intelligence or conditional logic is preventing the engine from batching the scans.

**4. Full table scans:**
SE queries with no `WHERE` clause at all indicate unfiltered scans. If the final result is small (few rows), the filter should be pushed into the SE. This often results from `FILTER(Table, ...)` patterns that materialize the entire table before applying the condition — see DAX009.

**5. Semi-join batches — evaluate selectivity:**
`DEFINE TABLE` / `ININDEX` patterns (semi-joins) can be efficient or wasteful depending on selectivity. If the semi-join table is large relative to the fact (low selectivity), the engine is doing significant work to build an index that barely filters. Consider whether the underlying relationship or filter propagation path could be restructured — see DAX022 for TREATAS/CROSSFILTER approaches.

---

### Vertical Fusion

Vertical fusion occurs when the engine combines multiple measure aggregations that share the same filter context into a single SE query. This means scanning a large fact table once instead of multiple times — one of the most impactful engine optimizations.

**When it works — shared filter context:**

Multiple measures aggregating from the same table with the same filters get fused:
```dax
DEFINE
    MEASURE Sales[Revenue] = SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] )
    MEASURE Sales[Cost] = SUMX ( Sales, Sales[Quantity] * Sales[UnitCost] )
    MEASURE Sales[Margin] = [Revenue] - [Cost]

EVALUATE
SUMMARIZECOLUMNS (
    Store[Region],
    Date[MonthNumber],
    "Revenue", [Revenue],
    "Cost", [Cost],
    "Margin", [Margin]
)
```

The engine generates a single SE query with both expressions:
```
WITH
    $Expr0 := ( Sales[Quantity] * Sales[UnitPrice] ),
    $Expr1 := ( Sales[Quantity] * Sales[UnitCost] )
SELECT
    Date[MonthNumber],
    Store[Region],
    SUM ( @$Expr0 ),
    SUM ( @$Expr1 )
FROM Sales
    LEFT OUTER JOIN Date ON Sales[OrderDate] = Date[Date]
    LEFT OUTER JOIN Store ON Sales[StoreKey] = Store[StoreKey]
```

Three measures, one table scan. The FE computes Margin from the fused results.

**Subtotals also fuse:** When SUMMARIZECOLUMNS computes subtotals (via ROLLUPADDISSUBTOTAL), the engine uses the same SE query for different aggregation granularities — total, group-level, and detail-level — as long as the measures are additive.

**What breaks vertical fusion:**

1. **Time intelligence functions** — DATESYTD, DATEADD, SAMEPERIODLASTYEAR, etc. prevent fusion because the FE must compute dynamic date ranges:
```dax
-- These two measures will NOT fuse into one SE query
MEASURE Sales[Current] = [Revenue]
MEASURE Sales[YTD] = CALCULATE ( [Revenue], DATESYTD ( Date[Date] ) )
```

2. **Custom time intelligence with variables** — Manual date range calculations also block fusion:
```dax
MEASURE Sales[YTD Custom] =
    VAR _LastDay = MAX ( Date[Date] )
    VAR _FirstDay = DATE ( YEAR ( _LastDay ), 1, 1 )
    RETURN CALCULATE ( [Revenue], Date[Date] >= _FirstDay && Date[Date] <= _LastDay )
```

3. **SWITCH/IF returning different measures** — Conditional measure selection prevents the engine from knowing which branch to fuse at plan time:
```dax
-- Breaks fusion for all measures in the query
SWITCH (
    SELECTEDVALUE ( MetricSelector[Name] ),
    "Revenue", [Revenue],
    "Cost", [Cost]
)
```

4. **Calculation groups with different items** — Multiple calculation items that invoke different measures or apply different filters break fusion.

**Optimization takeaway:** When you see multiple SE queries scanning the same fact table with the same joins and filters, vertical fusion was likely blocked. If time intelligence is the blocker, consider whether a date bridge table could make the calculation additive.

---

### Horizontal Fusion

Horizontal fusion occurs when the engine combines multiple SE queries that differ only in the filter applied to one or more columns into a single SE query. The engine adds the filtered column to the grouping and lets the FE split the slices afterward.

**When it works — single-value column filters on groupby columns:**

```dax
DEFINE
    MEASURE Sales[Online Revenue] =
        CALCULATE ( [Revenue], Sales[Channel] = "Online" )
    MEASURE Sales[Store Revenue] =
        CALCULATE ( [Revenue], Sales[Channel] = "Store" )
    MEASURE Sales[Combined] = [Online Revenue] + [Store Revenue]

EVALUATE
SUMMARIZECOLUMNS (
    Sales[Channel],
    Date[Year],
    "Combined", [Combined]
)
```

Because `Sales[Channel]` is already a groupby column, the engine recognizes the CALCULATE filters as slices on an existing dimension and fuses them into one SE query.

**What breaks horizontal fusion:**

1. **Multi-value filters on non-groupby columns** — If the filtered column is not in the groupby list and the filter has multiple values, fusion fails:
```dax
-- Won't fuse: Brand is not a groupby column and has multi-value filter
MEASURE Sales[Premium] =
    CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] IN { "Contoso", "Fabrikam" } ) )
MEASURE Sales[Budget] =
    CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] IN { "Litware" } ) )
```

   **Workaround:** Add the filtered column to groupby and re-aggregate:
```dax
EVALUATE
GROUPBY (
    SUMMARIZECOLUMNS (
        Product[Color],
        Product[Brand],
        "Val", [Combined]
    ),
    Product[Color],
    "Val", SUMX ( CURRENTGROUP (), [Val] )
)
```

2. **Table-valued filters (time intelligence, range predicates)** — Any filter that applies a table to the filter context blocks fusion, even if identical across measures:
```dax
-- Won't fuse: DATESYTD produces a table filter
MEASURE Sales[Contoso YTD] =
    CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] = "Contoso" ), DATESYTD ( Date[Date] ) )
MEASURE Sales[Fabrikam YTD] =
    CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] = "Fabrikam" ), DATESYTD ( Date[Date] ) )
```

   **Workaround:** Move the table filter to the consuming measure:
```dax
MEASURE Sales[Contoso] = CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] = "Contoso" ) )
MEASURE Sales[Fabrikam] = CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] = "Fabrikam" ) )
MEASURE Sales[Combined YTD] = CALCULATE ( [Contoso] + [Fabrikam], DATESYTD ( Date[Date] ) )
```

3. **Dynamic column filters** — If the filter value is computed at runtime (even if identical across measures), fusion is blocked:
```dax
-- Won't fuse: _LastDate is dynamically computed
MEASURE Sales[Contoso Latest] =
    VAR _LastDate = MAX ( Date[Date] )
    RETURN CALCULATE ( [Revenue], KEEPFILTERS ( Product[Brand] = "Contoso" ), Date[Date] = _LastDate )
```

   **Workaround:** Move the dynamic filter to the consuming measure.

**Optimization takeaway:** When you see N nearly-identical SE queries that differ only in their WHERE clause filters, horizontal fusion has failed. Check whether the filtered column can be added to the groupby, or whether dynamic/table filters can be moved to an outer CALCULATE.

---

## Optimization Strategy Framework

When approaching DAX optimization, choose the strategy that best matches the identified bottlenecks.

### 1. Anti-Pattern Detection Approach
- **Scan for known issues**: Look for FILTER over tables, context transitions in iterators, duplicated measures, nested CALCULATE
- **Identify callbacks**: Search trace events for CallbackDataID, EncodeCallback
- **Detect materializations**: Find large row counts in SE queries
- **Apply targeted fixes**: Use the anti-pattern catalog below

### 2. Set-Based Optimization Approach

Think in terms of mathematical set operations rather than iterative processing:

- **Cardinality operations**: Use COUNTROWS instead of SUMX with IF/INT conditions
- **Set filtering**: Apply predicates to entire sets rather than element-by-element
- **Set intersection**: Create qualifying sets first, then operate on the intersection
- **Boolean arithmetic**: Convert set membership tests to efficient numeric operations (INT)

**Set-based patterns:**
```dax
// Replace iterative counting with set cardinality
// Instead of: SUMX(table, IF(condition, 1, 0))
COUNTROWS(FILTER(table, condition))

// Pre-materialize qualifying sets
// Instead of: Multiple SUMX over same base with different conditions
// Use: Single SUMMARIZECOLUMNS with multiple calculated columns

// Set intersection approach
// Instead of: FILTER(CROSSJOIN(...), complex_condition)
CALCULATETABLE(SUMMARIZECOLUMNS(...), filter_conditions)
```

### 3. Fusion-Oriented Approach
- **Vertical Fusion**: Combine measures with same filter context using SUMMARIZECOLUMNS
- **Horizontal Fusion**: Consolidate similar queries differing only by column filters
- **Expression Consolidation**: Move repeated calculations into variables or base materializations

### 4. Storage Engine Optimization Approach
- **Minimize SE queries**: Fewer distinct SE queries = better (aim for 1-3)
- **Eliminate callbacks**: Remove CallbackDataID patterns — replace with native SE operations
- **Optimize materializations**: Reduce row/column counts in SE queries
- **Maximize parallelism**: Structure queries so SE can parallelize (SE_Par > 1.5)

---

## Anti-Patterns and Optimizations

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

Repeating the same expensive measure/expression inside multiple iterators over the same table causes duplicate scans. Eliminate through variable caching or base table materialization.

**Anti-pattern:**
```dax
VAR A = SUMX ( Sales, [Total Sales] )
VAR B = AVERAGEX ( Sales, [Total Sales] )
```

**Preferred:**
```dax
VAR Base = SUMMARIZECOLUMNS ( Sales[ProductKey], "@TotalSales", [Total Sales] )
VAR A = SUMX ( Base, [@TotalSales] )
VAR B = AVERAGEX ( Base, [@TotalSales] )
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

### DAX008: Duplicate Measure or Expression

Duplicate expression evaluations trigger duplicate SE queries. Cache in a variable.

**Anti-pattern:**
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

**Also applies to:**
```dax
// Anti-pattern:
IF([Sales Amount] > 1000, [Sales Amount] * 2, [Sales Amount] / 2)

// Preferred:
VAR _SalesAmount = [Sales Amount]
RETURN IF(_SalesAmount > 1000, _SalesAmount * 2, _SalesAmount / 2)
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

SWITCH translates to nested IF internally. The engine can evaluate only the matching branch — but when this optimization fails, it falls back to a full cartesian product of all groupby columns, which is catastrophically slow.

**Anti-pattern (multiple aggregations in one branch):**
```dax
MEASURE Sales[Selected Metric] =
    SWITCH (
        SELECTEDVALUE ( MetricSelector[Name] ),
        "Margin", SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] )
              - SUMX ( Sales, Sales[Quantity] * Sales[UnitCost] ),
        "Revenue", SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] )
    )
```

The "Margin" branch has two SUMX calls combined with `-`. This breaks the optimization.

**Preferred (single aggregation per branch):**
```dax
MEASURE Sales[Selected Metric] =
    SWITCH (
        SELECTEDVALUE ( MetricSelector[Name] ),
        "Margin", SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] - Sales[Quantity] * Sales[UnitCost] ),
        "Revenue", SUMX ( Sales, Sales[Quantity] * Sales[UnitPrice] )
    )
```

**Also breaks with implicit data type casts:**
```dax
-- SLOW: INTEGER branch gets implicit cast to CURRENCY
SWITCH (
    SELECTEDVALUE ( MetricSelector[Name] ),
    "Revenue", SUM ( Sales[Amount] ),          -- Returns CURRENCY
    "Count", SUM ( Sales[Quantity] )            -- Returns INTEGER
)

-- FIX: Explicit conversion so all branches match
"Count", CONVERT ( SUM ( Sales[Quantity] ), CURRENCY )
```

**Also breaks with context transitions in branches:**
```dax
-- Anti-pattern: measure reference inside SUMX branch
"Discounted", SUMX ( Sales, Sales[Quantity] * [Discount] )

-- Preferred: cache outside SWITCH
VAR _Discount = [Discount]
RETURN
    SWITCH (
        ...,
        "Discounted", SUMX ( Sales, Sales[Quantity] * _Discount )
    )
```

**How to detect:** Trace shows massive FE duration with minimal SE work on a SWITCH/IF measure in a visual with many groupby columns.

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

---

## Real-World Optimization Examples

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

---

## Query Structure Recommendations

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

## Model Optimization Patterns

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

