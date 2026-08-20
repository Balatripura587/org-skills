# orion-mcp-planner

A query planner skill that translates natural language performance questions into [orion-mcp](https://github.com/cloud-bulldozer/orion-mcp) tool calls.

## What it does

Takes questions like *"has 4.22 regressed on ROSA-HCP?"* and:

1. **Parses** version, platform, workload, scale, and flags from the question
2. **Discovers** matching CI jobs and their Orion configs via `discover_jobs`
3. **Calls** the right orion-mcp tool (`has_openshift_regressed`, `openshift_report_on`, etc.)

## Prerequisites

The `orion-mcp` MCP server must be connected. This skill orchestrates MCP tools — it does not run Orion CLI directly.


## Usage

Requires the [orion-mcp](https://github.com/cloud-bulldozer/orion-mcp) MCP server connected. Works in any environment with MCP tool access (Claude Code plugin, Slack bot, etc.).

## License

Apache-2.0
