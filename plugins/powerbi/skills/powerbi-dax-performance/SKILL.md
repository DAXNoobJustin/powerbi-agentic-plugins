---
name: powerbi-dax-performance
description: Guide to optimize DAX query performance using the DAX Performance Tuner MCP Server. Use this skill when asked to optimize DAX queries, troubleshoot slow queries, analyze DAX performance, tune query execution, investigate SE/FE bottlenecks, analyze xmSQL, or reduce query execution time. Do NOT use for creating or editing measures (use powerbi-semantic-model), TMDL syntax (use powerbi-tmdl), report layout/visual authoring (use powerbi-pbir), or workspace administration (use fabric-cli).
---

# DAX Performance Optimization Skill

This skill provides guidance on optimizing DAX query performance using the `dax-performance-tuner-mcp` MCP server.

## IMPORTANT

- Check if the `dax-performance-tuner-mcp` MCP Server is available. If it is, always use it for performance optimization workflows. The MCP tools are self-documenting — follow their descriptions and output guidance.
- Connecting to a dataset always resets any prior optimization session. Each `connect_to_dataset` call starts fresh.

## Tool Selection Priority

1. **MCP Server available** → Use `dax-performance-tuner-mcp` tools. The workflow is: `connect_to_dataset` → `prepare_query_for_optimization` → iterate with `execute_dax_query` → use `get_session_status` to stay oriented.
2. **MCP Server unavailable** → Guide the user through manual optimization: suggest Performance Analyzer in Power BI Desktop, manual DAX Studio traces, or best-practice DAX patterns.

## Relationship to Other Skills

- **powerbi-semantic-model**: Use for creating/editing measures and model structure. If the user asks to *optimize* a measure's performance, defer here. If they ask to *create* or *edit* a measure, use `powerbi-semantic-model`.
- **powerbi-tmdl**: Use for TMDL file syntax. Not relevant to performance tuning.
- **powerbi-pbir**: Use for report authoring. Not relevant to performance tuning.
- **fabric-cli**: Use for discovering workspaces and semantic models. Can help locate the dataset to connect to before starting optimization.

## Task: Optimize a DAX Query

Use the `dax-performance-tuner-mcp` tools. The tools provide detailed guidance in their descriptions and output — follow their instructions at each step.

## Error Handling

- **Connection failure**: Verify the dataset name, workspace name, or XMLA endpoint. For desktop, ensure Power BI Desktop is running. For service, ensure XMLA read/write is enabled on the capacity.
- **Warm-up failure**: The query itself has errors — fix the DAX syntax first.
- **Semantic equivalence failure**: The optimized query returns different results. Review the measure logic — the optimization changed calculation semantics, not just performance.
- **No improvement found**: Some queries are already well-optimized. Consider whether the bottleneck is in the data model itself (e.g., missing aggregations, poor star schema design) rather than the DAX.
