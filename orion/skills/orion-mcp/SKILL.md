---
name: orion-mcp
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

**Always call `discover_jobs`** — even when the user names a specific config file. It returns the config's required `input_vars` (platform, workerNodesCount, clusterType, etc.) from real ES metadata. Without this, Jinja-template configs like `cluster-density.yaml` render with empty values and return wrong or no results.

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

**If the user specifies a config name explicitly** (e.g. "use cluster-density.yaml"): still call `discover_jobs` to get the `metadata`/`input_vars` for that config. Pass the user-specified config as `config_name` and the discovered `metadata` as `input_vars`.

**If `configs` is empty** for a job (prow artifacts expired or unavailable): use `get_orion_configs` to list available configs, then match by benchmark name (e.g. benchmark `cluster-density-v2` → config `cluster-density.yaml`). If no obvious match, ask the user which config to use.

## Step 3: Call orion-mcp

Build `input_vars` from the job's `metadata` as a JSON string. Comma-join config files for multi-config tools.

**Always run Steps 1-2 (discover_jobs) before calling any tool — EXCEPT these three which need no discovery:**
- `get_orion_configs` — lists configs, no ES needed
- `get_release_date` — date lookup only
- `get_orion_metrics_with_meta` — reads config YAML locally, pass `config_name` directly, no `input_vars` needed

**For PR analysis** (`openshift_report_on_pr`): run `discover_jobs` with `job_type="pull"` instead of `"periodic"`.

| Intent | Tool |
|---|---|
| "has X regressed" | `has_openshift_regressed` |
| "networking regressions" | `has_networking_regressed` |
| "inspect nightly" | `has_nightly_regressed` |
| "show metric" / "compare versions" | `openshift_report_on` |
| "correlate X with Y" | `metrics_correlation` |
| "health check" / "overall performance" | `get_performance_summary` |
| "what metrics does X track" | `get_orion_metrics` |
| "thresholds / directions for metrics" | `get_orion_metrics_with_meta` |
| "analyze PR" / "check PR impact" | `openshift_report_on_pr` |
| "list configs" / "what benchmarks exist" | `get_orion_configs` |
| "release date for X" | `get_release_date` |

Multiple configs: pass comma-separated. All share same input_vars. Different input_vars → separate calls.

**Networking intent**: Networking configs (`node-density-cni.yaml`, `udn-density-pods.yaml`, `udn-*`) are run inside **payload jobs** . Call `discover_jobs` with `workload="payload"`, then filter the returned `configs` list to keep only those matching `*cni*`, `*udn*`, `*cudn*`. Pass only those to `has_networking_regressed`.

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
