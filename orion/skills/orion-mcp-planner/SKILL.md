---
name: orion-mcp-planner
description: "Translates performance questions into orion-mcp tool calls. Parses intent/filters, calls discover_jobs for configs, then calls the right orion tool."
disable-model-invocation: false
user-invocable: true
allowed-tools: Read Bash
argument-hint: "[user-query]"
---

# Orion MCP Query Planner

You translate user performance questions into orion-mcp tool calls. You parse, discover_jobs resolves, orion-mcp executes.

**Prerequisite:** The `orion-mcp` MCP server must be connected. This skill orchestrates MCP tools — it does not run Orion CLI directly.

## Knowledge: How CI Perf Jobs Work

**ES index `perf_scale_ci*`** stores metadata for every CI perf job run. Key fields:
- `upstreamJob` — full prow job name
- `benchmark` — workload name (e.g. `cluster-density-v2`, `node-density`)
- `ocpVersion` — full nightly string (e.g. `4.22.0-0.nightly-2026-08-10-215205`)
- `platform` — `AWS`, `GCP`, `Azure`, `BareMetal`
- `clusterType` — `self-managed`, `rosa-hcp`, `rosa`
- `workerNodesCount` — integer (6, 24, 120, etc.)
- `networkType` — `OVNKubernetes`
- `fips`, `ipsec`, `encrypted` — `"true"` or `"false"` strings
- `jobType` — `periodic` or `pull`
- `buildUrl` — prow job URL with artifacts

**Orion config files** live in `cloud-bulldozer/orion/examples/`. Each config defines which ES queries and metrics Orion runs. The config filename is what orion-mcp tools need as `config_name`.

## Step 1: Parse Query

Extract from user question:

| Field | How to identify | discover_jobs param |
|---|---|---|
| Version | "4.22", "5.0" | `version` |
| Platform | see mapping below | `platform` + `cluster_type` |
| Workload | "payload", "control-plane", "udn", "router" | `workload` |
| Scale | "6-node", "24-node", "120 workers" | `scale` (integer) |
| FIPS | "fips" mentioned | `fips="true"` |
| IPsec | "ipsec" mentioned | `ipsec="true"` |
| Encrypted | "encrypted", "etcd encryption" | `encrypted="true"` |
| Metric | "podReadyLatency", "ovnCPU", etc. | (passed to execution tool) |
| Intent | regression/compare/health/etc. | (determines which tool) |

**Platform mapping** (user term → discover_jobs params):

| User says | platform | cluster_type |
|---|---|---|
| aws (default) | AWS | self-managed |
| rosa-hcp | AWS | rosa-hcp |
| rosa | AWS | rosa |
| gcp | GCP | (any) |
| azure | Azure | (any) |
| metal, baremetal | BareMetal | (any) |

**Defaults** when user is vague:
- No filters at all → `workload="payload"`, `scale=6`, `platform="AWS"`, `cluster_type="self-managed"`
- fips/ipsec/encrypted mentioned without workload → `workload="control-plane"`
- Platform specified without workload → don't filter workload (show all)

**If unsure about a mapping**, call `discover_jobs` with just the `version` and no other filters to see what jobs/platforms/scales exist. The response shows all available combinations.

## Step 2: Discover Jobs

Call `discover_jobs` with the params from Step 1. It returns jobs with **configs already resolved** from prow build logs:
```json
{
  "jobs": {
    "periodic-ci-...-payload-control-plane-6nodes": {
      "benchmarks": ["cluster-density-v2", "node-density", ...],
      "configs": ["cluster-density.yaml", "node-density.yaml", ...],
      "metadata": {"platform": "AWS", "workerNodesCount": "6", ...},
      "buildUrl": "https://prow.ci.openshift.org/view/gs/..."
    }
  },
  "total": 4
}
```

Use `configs` directly as comma-separated `config_name`. Build `input_vars` JSON from `metadata`.

**If `configs` is empty** for a job (prow artifacts expired or unavailable): use `get_orion_configs` to list available configs, then match by benchmark name (e.g. benchmark `cluster-density-v2` → config `cluster-density.yaml`). If no obvious match, ask the user which config to use.

## Step 3: Call orion-mcp

Build `input_vars` from the job's `metadata` as a JSON string. Comma-join config files for multi-config tools.

| Intent | Tool | Key params |
|---|---|---|
| "has X regressed" | `has_openshift_regressed` | config_name, input_vars, version, lookback |
| "networking regressions" | `has_networking_regressed` | config_name, input_vars, version, lookback |
| "inspect nightly" | `has_nightly_regressed` | config_name, input_vars, nightly_version, lookback |
| "show metric" / "compare" | `openshift_report_on` | config_name, input_vars, versions, metric, lookback |
| "correlate X with Y" | `metrics_correlation` | config_name, input_vars, metric1, metric2, version |
| "health check" | `get_performance_summary` | config_name, input_vars, version, lookback |
| "what metrics" | `get_orion_metrics` | config_name, input_vars, version |
| "analyze PR" | `openshift_report_on_pr` | config_name, input_vars, version, org, repo, pull_request |
| "list configs" | `get_orion_configs` | (none — skip Steps 1-2) |
| "release date" | `get_release_date` | version (skip Steps 1-2) |

Multiple configs: pass comma-separated. All share same input_vars. Different input_vars → separate calls.

## Debugging: Where Things Live

Only use these when `discover_jobs` returns unexpected results or `configs` is empty:

| What | How to access |
|---|---|
| Job definitions | `gh api repos/openshift/release/contents/ci-operator/config/openshift-eng/ocp-perfscale/` |
| Prow step registry | `gh api repos/openshift/release/contents/ci-operator/step-registry/openshift-qe/orion/` |
| Orion config files | `get_orion_configs()` tool |
| Env var overrides | `gh api` to read step ref YAML or ci-operator config YAML |

## Example

**"has 4.22 regressed?"**
1. Parse: version=4.22, no other filters → defaults: workload=payload, scale=6, platform=AWS, cluster_type=self-managed
2. `discover_jobs(version="4.22", platform="AWS", cluster_type="self-managed", workload="payload", scale=6)` → returns jobs with configs already resolved
3. `has_openshift_regressed(config_name="cluster-density.yaml,node-density.yaml,node-density-cni.yaml,crd-scale.yaml,udn-density-pods.yaml", input_vars='{"platform":"AWS","workerNodesCount":"6",...}', version="4.22")`
