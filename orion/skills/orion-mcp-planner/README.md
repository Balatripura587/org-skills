# orion-mcp-planner

A query planner skill that translates natural language performance questions into [orion-mcp](https://github.com/cloud-bulldozer/orion-mcp) tool calls.

## What it does

Takes questions like *"has 4.22 regressed on ROSA-HCP?"* and:

1. **Parses** version, platform, workload, scale, and flags from the question
2. **Discovers** matching CI jobs and their Orion configs via `discover_jobs`
3. **Calls** the right orion-mcp tool (`has_openshift_regressed`, `openshift_report_on`, etc.)

## Prerequisites

The `orion-mcp` MCP server must be connected. This skill orchestrates MCP tools — it does not run Orion CLI directly.

Unlike the `orion-regression-analysis` skill (which teaches Orion CLI usage), this skill is purpose-built for environments where orion-mcp provides the tool layer (e.g., Claude Code with MCP, Chai-bot).

## Usage

```
/orion-mcp-planner has 4.22 regressed?
/orion-mcp-planner show podReadyLatency for 5.0 vs 4.22 on rosa-hcp
/orion-mcp-planner are there networking regressions on 4.22 fips?
```

## License

Apache-2.0
