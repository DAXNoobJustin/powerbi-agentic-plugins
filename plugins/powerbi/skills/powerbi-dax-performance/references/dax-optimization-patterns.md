# DAX Optimization Patterns Reference

A comprehensive catalog of DAX anti-patterns, optimization strategies, and trace analysis guidance.

## Table of Contents

- [Performance Analysis Framework](#performance-analysis-framework)
- [Trace Analysis Guide](#trace-analysis-guide)
- [Anti-Patterns and Optimizations](#anti-patterns-and-optimizations)
- [Real-World Optimization Examples](#real-world-optimization-examples)
- [Optimization Strategy Framework](#optimization-strategy-framework)
- [SQLBI Documentation References](#sqlbi-documentation-references)

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

## Anti-Patterns and Optimizations

### CUST001: Use SUMMARIZECOLUMNS to Create Virtual Columns

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

### CUST002: Variable Caching for Repeated Evaluations

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

### CUST003: Duplicate Filter Conditions

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

### CUST004: SUMMARIZE with Complex Table Expression

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

### CUST005: Replace Iterator with Context Transition Using SUMMARIZECOLUMNS Base Table

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

### CUST006: Replace IF with INT for Boolean Conversion

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

### CUST007: Context Transition in Iterator

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

### CUST008: Duplicate Measure or Expression

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

### CUST009: Apply Filters Using CALCULATETABLE Instead of FILTER

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

### CUST010: Move Context-Independent Expressions Outside Iterators

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

### CUST011: Use Simple Column Filter Predicates as CALCULATE Arguments

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

### CUST012: Distinct Count Alternatives

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

### CUST013: Force Formula Engine Evaluation with CROSSJOIN

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

### CUST014: Replace SELECTEDVALUE with MAX/MIN

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

### CUST015: Use ALLEXCEPT Instead of ALL + VALUES Restoration

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

### CUST016: Flatten Nested CALCULATE Calls

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

## Optimization Strategy Framework

When approaching DAX optimization, choose the strategy that best matches the identified bottlenecks.

### 1. Anti-Pattern Detection Approach
- **Scan for known issues**: Look for FILTER over tables, context transitions in iterators, duplicated measures, nested CALCULATE
- **Identify callbacks**: Search trace events for CallbackDataID, EncodeCallback
- **Detect materializations**: Find large row counts in SE queries
- **Apply targeted fixes**: Use the anti-pattern catalog above

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

## SQLBI Documentation References

For deeper understanding of DAX internals, consult these SQLBI resources:

- **VertiPaq xmSQL**: <https://docs.sqlbi.com/dax-internals/vertipaq/xmSQL> — Understanding the xmSQL syntax that appears in SE trace events
- **Vertical Fusion**: <https://docs.sqlbi.com/dax-internals/optimization-notes/vertical-fusion> — How the engine combines measure calculations
- **Horizontal Fusion**: <https://docs.sqlbi.com/dax-internals/optimization-notes/horizontal-fusion> — How the engine combines similar filter contexts
- **SWITCH Optimization**: <https://docs.sqlbi.com/dax-internals/optimization-notes/switch-optimization> — Performance implications of SWITCH with aggregations or context transitions
