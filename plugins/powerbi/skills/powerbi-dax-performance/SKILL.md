---
name: powerbi-dax-performance
description: Guide to optimize DAX query performance using the DAX Performance Tuner MCP Server. Use this skill when asked to optimize DAX queries, troubleshoot slow queries, analyze DAX performance, tune query execution, investigate SE/FE bottlenecks, analyze xmSQL, or reduce query execution time. Do NOT use for creating or editing measures (use powerbi-semantic-model), TMDL syntax (use powerbi-tmdl), report layout/visual authoring (use powerbi-pbir), or workspace administration (use fabric-cli).
---

# DAX Performance Optimization Skill

This skill provides guidance on optimizing DAX query performance using the `dax-performance-tuner-mcp` MCP server.

## IMPORTANT

- Check if the `dax-performance-tuner-mcp` MCP Server is available. If it is, always use it for performance optimization workflows.
- The MCP server provides automated baselining, trace analysis, model metadata extraction, and research article matching — do NOT attempt to replicate this manually.
- Connecting to a dataset always resets any prior optimization session. Each `connect_to_dataset` call starts fresh.

## Tool Selection Priority

1. **MCP Server available** → Use `dax-performance-tuner-mcp` tools for the full optimization workflow.
2. **MCP Server unavailable** → Guide the user through manual optimization: suggest Performance Analyzer in Power BI Desktop, manual DAX Studio traces, or best-practice DAX patterns.

## Relationship to Other Skills

- **powerbi-semantic-model**: Use for creating/editing measures and model structure. If the user asks to *optimize* a measure's performance, defer here. If they ask to *create* or *edit* a measure, use `powerbi-semantic-model`.
- **powerbi-tmdl**: Use for TMDL file syntax. Not relevant to performance tuning.
- **powerbi-pbir**: Use for report authoring. Not relevant to performance tuning.
- **fabric-cli**: Use for discovering workspaces and semantic models. Can help locate the dataset to connect to before starting optimization.

## The 2-Stage Optimization Workflow

### Stage 1: Connect to the Dataset

Use `connect_to_dataset` to establish a connection. The tool is smart — it connects, discovers, or auto-matches depending on what parameters you provide:

- **Direct connect** (has dataset name + location info): `connect_to_dataset(dataset_name='Sales Model', workspace_name='Sales Analytics')` or `connect_to_dataset(dataset_name='Sales Model', xmla_endpoint='powerbi://...')`
- **Auto-match** (has dataset name only): `connect_to_dataset(dataset_name='Sales')` — searches desktop instances for matches
- **Discover datasets** (has location, no name): `connect_to_dataset(workspace_name='Sales Analytics')` or `connect_to_dataset(desktop_port=57466)` — lists available datasets
- **Discover desktop instances** (no parameters): `connect_to_dataset()` — lists all open Power BI Desktop instances

### Stage 2: Prepare, Optimize, Iterate

1. **Prepare the query**: Call `prepare_query_for_optimization` with the original DAX query. This single call:
   - Inlines all measure and user-defined function definitions from the query
   - Executes a baseline performance measurement with comprehensive trace analysis
   - Extracts relevant model metadata for the query context
   - Retrieves targeted DAX research articles based on detected patterns
   - Returns the complete foundation for optimization work

2. **Analyze the baseline**: Study the output carefully:
   - **Performance**: Total ms, FE (formula engine) ms, SE (storage engine) ms, SE_CPU, SE_Par (parallelism), SE_Queries, SE_Cache
   - **EventDetails**: Execution waterfall showing xmSQL queries, rows returned, KB scanned, callbacks
   - **Research articles**: Pattern-matched articles with optimization guidance — cross-validate matches against EventDetails before applying (false positives will occur)
   - **Model metadata**: Relationships and columns referenced by the query

3. **Optimize measure definitions**: Using the research articles, performance data, and model metadata:
   - Focus on optimizing MEASURE and USER-DEFINED FUNCTION DEFINITIONS, not query structure
   - Keep the same SUMMARIZECOLUMNS grouping as baseline
   - Optimized query MUST return identical row count, column structure, and values (semantic equivalence)

4. **Test with `execute_dax_query`**: Submit the optimized query. The tool automatically compares performance to baseline and checks semantic equivalence.

5. **Iterate**: Analyze results after each attempt. If bottlenecks remain, try different optimization strategies. Continue until >=10% improvement is achieved with semantic equivalence.

6. **Re-baseline** (optional): After a successful optimization, the optimized query can be used as the new baseline via another `prepare_query_for_optimization` call for further iterative gains.

## Task: Optimize a DAX Query

1. Connect to the dataset using `connect_to_dataset`.
2. Call `prepare_query_for_optimization` with the user's DAX query.
3. Deeply analyze the baseline Performance and EventDetails.
4. Cross-reference detected patterns with the returned research articles.
5. Develop an optimization strategy targeting the identified bottlenecks.
6. Submit the optimized query via `execute_dax_query`.
7. Analyze the comparison results — check both performance improvement and semantic equivalence.
8. If improvement is insufficient, iterate with a different approach.
9. Present the final optimized query and performance comparison to the user.

## Task: Optimize from Power BI Desktop

1. Call `connect_to_dataset()` with no parameters to discover running Desktop instances.
2. If the user has a specific model open, connect to it by name or port.
3. Have the user capture the slow query from Performance Analyzer or provide it directly.
4. Follow the standard optimization workflow above.

## Task: Optimize from Fabric Workspace

1. Connect using workspace name: `connect_to_dataset(dataset_name='Model Name', workspace_name='Workspace Name')`.
2. Or connect using XMLA endpoint: `connect_to_dataset(dataset_name='Model Name', xmla_endpoint='powerbi://api.powerbi.com/v1.0/myorg/WorkspaceName')`.
3. Follow the standard optimization workflow above.

## Task: Check Session Status

Use `get_session_status` at any time to review:
- Current connection details
- Baseline performance metrics
- All optimization attempts and their results
- Best optimization found so far (improvement %, semantic equivalence)
- Recommendations for next steps

This is useful for staying oriented during long optimization sessions or resuming after context switches.

## Understanding the Output

### Performance Metrics

| Metric | Description |
|---|---|
| Total | Total query execution time in milliseconds |
| FE | Formula Engine time — DAX calculation overhead |
| SE | Storage Engine time — data retrieval from model |
| SE_CPU | Total CPU time across all SE queries |
| SE_Par | Parallelism ratio (SE_CPU / SE) — higher means better parallel utilization |
| SE_Queries | Number of storage engine queries generated |
| SE_Cache | Number of SE queries served from cache |

### Key Optimization Patterns

- **High FE, Low SE**: The bottleneck is in DAX calculations. Look for row-by-row iteration, nested CALCULATE with complex filters, or excessive context transitions.
- **High SE, Low FE**: The bottleneck is in data retrieval. Look for large table scans, missing aggregations, or overly broad filters in xmSQL.
- **Many SE queries**: The formula engine is generating too many individual storage queries. Look for callback patterns in EventDetails — these force row-by-row SE evaluation.
- **Low SE_Par**: Storage queries are not parallelizing well. May indicate dependency chains between SE queries.

## Error Handling

- **Connection failure**: Verify the dataset name, workspace name, or XMLA endpoint. For desktop, ensure Power BI Desktop is running. For service, ensure XMLA read/write is enabled on the capacity.
- **Warm-up failure**: The baseline runs the query 3 times for warm-up before measuring. If warm-up fails, the query itself has errors — fix the DAX syntax first.
- **Semantic equivalence failure**: The optimized query returns different results than the baseline. Review the measure logic — the optimization changed the calculation semantics, not just performance.
- **No improvement found**: Some queries are already well-optimized. Consider whether the bottleneck is in the data model itself (e.g., missing aggregations, poor star schema design) rather than the DAX.
