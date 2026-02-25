---
name: powerbi-dax-performance
description: Guide to optimize DAX query performance using the Power BI Modeling MCP Server. Use this skill when asked to optimize DAX queries, troubleshoot slow queries, analyze DAX performance, tune query execution, investigate SE/FE bottlenecks, analyze xmSQL, or reduce query execution time. Do NOT use for creating or editing measures (use powerbi-semantic-model), TMDL syntax (use powerbi-tmdl), report layout/visual authoring (use powerbi-pbir), or workspace administration (use fabric-cli).
---

# DAX Performance Optimization Skill

This skill provides a structured two-phase workflow for optimizing DAX query performance using exclusively the `powerbi-modeling-mcp` server tools. The approach: establish a baseline, analyze bottlenecks, apply targeted optimizations to measure definitions, and verify improvement with semantic equivalence.

## IMPORTANT

- Requires the `powerbi-modeling-mcp` MCP Server. All operations use its tools.
- You must be connected to a semantic model before starting. Use `connection_operations` to connect if not already connected.
- **Optimize MEASURE DEFINITIONS, not query structure.** Keep the EVALUATE / SUMMARIZECOLUMNS grouping identical between baseline and optimized queries. The bottleneck is almost always in the measure logic.
- **Success criteria**: >= 10% performance improvement AND semantic equivalence (identical row count, column count, and data values).
- Refer to the `references/dax-optimization-patterns.md` file for the complete catalog of anti-patterns and optimization strategies.

## Relationship to Other Skills

- **powerbi-semantic-model**: Use for creating/editing measures and model structure. If the user asks to *optimize* a measure's performance, defer here. If they ask to *create* or *edit* a measure, use `powerbi-semantic-model`.
- **powerbi-tmdl**: Use for TMDL file syntax. Not relevant to performance tuning.
- **powerbi-pbir**: Use for report authoring. Not relevant to performance tuning.
- **fabric-cli**: Use for discovering workspaces and semantic models. Can help locate the dataset to connect to before starting optimization.

---

## Phase 1: Establish Baseline

### Step 1: Resolve All Measure and Function Definitions

Before optimizing, you need full visibility into every DAX expression being evaluated. The user's query references measures that may themselves reference other measures, creating a dependency chain you must fully resolve.

1. **Identify measure references** in the user's query. Look for `[MeasureName]` patterns — these are implicit measure calls.
2. **Retrieve each measure's expression** using `measure_operations` → Get operation, specifying the measure name and its table.
3. **Check for nested dependencies**: Read each measure's expression and identify any `[OtherMeasure]` references within it. Recursively retrieve those definitions too.
4. **Retrieve user-defined function definitions** if the query or any measure references them. Use `function_operations` → Get or List.
5. **Build a DEFINE block** that explicitly defines ALL resolved measures and functions. This "inlines" the definitions so you can see and optimize them directly.

**Example**: If the user's query contains `[Profit Margin]`, and that measure's expression is `DIVIDE([Total Profit], [Total Revenue])`, you need to also retrieve `[Total Profit]` and `[Total Revenue]` definitions, then build:

```dax
DEFINE
    MEASURE Sales[Total Revenue] = SUM(Sales[Revenue])
    MEASURE Sales[Total Profit] = SUM(Sales[Revenue]) - SUM(Sales[Cost])
    MEASURE Sales[Profit Margin] = DIVIDE([Total Profit], [Total Revenue])

EVALUATE
SUMMARIZECOLUMNS(
    Product[Category],
    "Profit Margin", [Profit Margin]
)
```

### Step 2: Gather Model Context

Collect relevant model metadata to inform optimization decisions:

1. Use `table_operations` → List to understand the model's table structure.
2. Use `column_operations` → Get for tables involved in the query to understand column types, cardinalities.
3. Use `relationship_operations` → List to understand how tables are joined — this affects filter propagation.

This context helps you identify whether issues come from model design (e.g., missing star schema, high-cardinality columns in filters) versus DAX expression logic.

### Step 3: Execute Baseline (Multi-Run)

Run the enhanced query (with DEFINE block) multiple times to get a reliable baseline, taking the fastest cold-cache run.

**For each run (repeat 3 times):**

1. **Clear the cache** using `dax_query_operations` → ClearCache. This ensures cold-cache execution.
2. **Execute the query** using `dax_query_operations` → Execute with `GetExecutionMetrics` set to `true`. This automatically:
   - Creates/resumes a trace with required events (QueryBegin/End, VertiPaqSEQueryBegin/End, DirectQueryBegin/End, ExecutionMetrics, Error)
   - Clears any previously captured trace events
   - Executes the query
   - Returns `CalculatedExecutionMetrics` (TotalDuration, FormulaEngineDuration, StorageEngineDuration, StorageEngineQueryCount, StorageEngineCpuTime, VertipaqCacheMatches) and `ReportedExecutionMetrics` (server-reported JSON including Direct Lake diagnostics).
3. **Immediately fetch detailed trace events** using `trace_operations` → Fetch. This returns the raw Storage Engine events with xmSQL text, per-query duration, rows returned, and KB transferred. **You must fetch events after each execution because the next Execute call will clear them.**
4. **Record** the TotalDuration and all metrics for this run.

**After all runs:**
- Select the **fastest run** (lowest TotalDuration) as the baseline.
- Record its full metrics and trace events for analysis.

### Step 4: Analyze Baseline

Using the baseline metrics and trace events, identify bottlenecks:

**Metrics Analysis:**
- **FE % = FormulaEngineDuration / TotalDuration**: If > 50%, the formula engine is doing too much work. Look for context transitions, callbacks, complex iterators.
- **SE Query Count**: If > 5-10 SE queries, the engine can't fuse operations efficiently. Look for duplicate expressions, nested iterators.
- **SE Parallelism = StorageEngineCpuTime / StorageEngineDuration**: If < 1.5, the SE isn't parallelizing well.

**Trace Event Analysis (from Fetch results):**
- **CallbackDataID in xmSQL**: A major red flag. This means the Formula Engine is being called back for each row — forces single-threaded evaluation. Look for measure references in FILTER, context transitions in iterators.
- **EncodeCallback**: Grouping by calculated expressions rather than physical columns.
- **Large row counts**: SE queries returning >>100K rows when the final result is small indicate excessive materializations.
- **Large KB values**: Wide materializations (too many columns being pulled).
- **Many short SE queries**: FE/SE "ping-pong" suggests the engine can't find an efficient plan.

**Cross-reference with the optimization patterns reference** to identify which anti-patterns are present.

---

## Phase 2: Optimization Iterations

### Step 1: Identify Optimization Opportunities

Based on your baseline analysis, identify which optimizations to apply. Consult `references/dax-optimization-patterns.md` for the full catalog. Common high-impact optimizations:

1. **Callbacks detected** → Replace measure references in FILTER with direct column predicates. Replace context transitions with SUMMARIZECOLUMNS base tables.
2. **High FE %** → Move computation to SE by eliminating iterators, using SUMMARIZECOLUMNS virtual columns, caching with variables.
3. **Many SE queries** → Consolidate duplicate expressions into variables. Use SUMMARIZECOLUMNS for better fusion.
4. **Large materializations** → Reduce iterator table size with VALUES() on key columns. Apply filters earlier.
5. **IF/SWITCH with context transitions** → Replace with INT() or pre-cached variables.

### Step 2: Modify Measure Definitions

**CRITICAL**: Only modify the measure definitions in the DEFINE block. Do NOT change the EVALUATE clause or SUMMARIZECOLUMNS grouping columns. The query structure must remain identical to preserve semantic equivalence.

Apply one or more optimizations to the DEFINE block measures. For example:

```dax
-- BASELINE measure
DEFINE
    MEASURE Sales[HighValueCount] =
        SUMX(Sales, IF(Sales[Amount] > 1000, 1, 0))

-- OPTIMIZED measure (CUST006: IF→INT, CUST009: FILTER→CALCULATETABLE)
DEFINE
    MEASURE Sales[HighValueCount] =
        COUNTROWS(CALCULATETABLE(Sales, Sales[Amount] > 1000))
```

### Step 3: Execute and Compare

1. **Clear cache** → `dax_query_operations` ClearCache.
2. **Execute optimized query** → `dax_query_operations` Execute with `GetExecutionMetrics=true`.
3. **Fetch trace events** → `trace_operations` Fetch.
4. **Run 3 times** (same multi-run approach as baseline), take the fastest.

**Compare to baseline:**
- **Calculate improvement**: `(BaselineDuration - OptimizedDuration) / BaselineDuration * 100`
- **Verify semantic equivalence**: The optimized query must return the same number of rows, the same columns, and the same data values as the baseline. If results differ, the optimization changed calculation semantics — revert it.

### Step 4: Iterate

- If improvement is **>= 10%** and results are semantically equivalent → **Success**. Present the optimized query and the improvement to the user.
- If improvement is **< 10%** → Try a different optimization strategy. Revisit the trace analysis for other bottlenecks.
- If results **differ** → The optimization changed semantics. Revert and try a different approach.
- If **multiple opportunities** exist, apply incrementally. Test each change individually to understand its impact, then combine the successful ones.

After achieving a successful optimization, **offer to continue**: the optimized query can become the new baseline for further optimization rounds. This iterative approach can achieve compound improvements.

---

## Error Handling

- **Connection failure**: Verify the dataset name, workspace name, or XMLA endpoint. For desktop, ensure Power BI Desktop is running. For service, ensure XMLA read/write is enabled on the capacity.
- **Query syntax error**: Use `dax_query_operations` Validate to check DAX syntax before executing.
- **Semantic equivalence failure**: The optimized query returns different results. Review the measure logic — the optimization changed calculation semantics, not just performance. Common causes: changed filter context, modified aggregation granularity, altered CALCULATE filter arguments.
- **No improvement found**: Some queries are already well-optimized. Consider whether the bottleneck is in the data model itself (e.g., missing aggregations, poor star schema design, high-cardinality columns) rather than the DAX.
- **Trace events empty**: Ensure `GetExecutionMetrics=true` was set on the Execute call. The trace is automatically managed when this flag is set.
