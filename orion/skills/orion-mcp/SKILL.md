---
name: orion-mcp
description: "Translates performance questions into orion-mcp tool calls. Use when the user asks about OpenShift performance regressions, benchmark results, version comparisons, nightly/PR analysis, networking regressions, or Orion config metrics."
disable-model-invocation: false
user-invocable: true
allowed-tools: mcp__orion-mcp__discover_jobs mcp__orion-mcp__has_openshift_regressed mcp__orion-mcp__has_networking_regressed mcp__orion-mcp__has_nightly_regressed mcp__orion-mcp__openshift_report_on mcp__orion-mcp__openshift_report_on_pr mcp__orion-mcp__metrics_correlation mcp__orion-mcp__get_orion_performance_data mcp__orion-mcp__get_performance_summary mcp__orion-mcp__get_orion_metrics mcp__orion-mcp__get_orion_metrics_with_meta mcp__orion-mcp__get_orion_configs mcp__orion-mcp__get_release_date
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
- `platform` — `AWS`, `GCP`, `Azure`, `BareMetal`, `IBMCloud`
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
| ibm, ibmcloud | IBMCloud | (any) |

**Defaults** when user is vague:
- No filters at all → `workload="payload"`, `scale=6`, `platform="AWS"`, `cluster_type="self-managed"`
- fips/ipsec/encrypted mentioned without workload → `workload="control-plane"`
- Platform specified without workload → don't filter workload (show all)
**Benchmark reference** (for user-facing descriptions):

| Config | Benchmark | What it measures |
|---|---|---|
| cluster-density.yaml | cluster-density-v2 | General OpenShift object density — namespaces, pods, services, routes, configmaps; measures pod scheduling latency and control-plane resource usage |
| node-density.yaml | node-density | Worker node pod saturation — container startup latency (P99 ContainersStarted) under maximum pod density per node |
| node-density-cni.yaml | node-density-cni | CNI service readiness under node density load — P99 serviceReadyLatency (time for services to become ready after pod creation) |
| udn-density-pods.yaml | udn-density-pods | User Defined Network pod density — pod scheduling latency and OVN-K component CPU/memory (northd, nbdb, sbdb, ovn-controller) under UDN load |
| crd-scale.yaml | crd-scale | CRD and CR scaling — API server CPU/memory, etcd latency, and API call latency as CRDs and CRs are scaled |
| small-scale-udn-l2.yaml | udn-density-l2 | Layer 2 UDN pod density (24-node AWS) — OVN-K CPU/memory under L2 UDN network topology |
| small-scale-udn-l3.yaml | udn-density-l3 | Layer 3 (routed) UDN pod density (24-node AWS) — same as L2 but for routed UDN topology |
| netpol-24nodes.yaml | network-policy | NetworkPolicy enforcement at 24 workers — policy enforcement latency and OVN-K control-plane resource usage |
| metal-perfscale-cpt-node-density.yaml | node-density | Node density on bare metal — pod startup latency and CPU/memory across apiserver, OVN, etcd, kubelet |
| metal-perfscale-cpt-virt-density.yaml | virt-density | VM density on bare metal — VM readiness latency (P99 VMReady) and resource consumption under OpenShift Virtualization |
| rosa-hcp-cluster-density.yaml | cluster-density-v2 | Cluster density on ROSA HCP (hosted control plane) — same workload as cluster-density adapted for managed control plane |

**If unsure about a mapping**, call `orion-mcp:discover_jobs` with just the `version` and no other filters to see what jobs/platforms/scales exist. The response shows all available combinations.

## Step 2: Discover Jobs

**Always call `orion-mcp:discover_jobs`** — even when the user names a specific config file. It returns the config's required `input_vars` (platform, workerNodesCount, clusterType, etc.) from real ES metadata. Without this, Jinja-template configs like `cluster-density.yaml` render with empty values and return wrong or no results.

Call `orion-mcp:discover_jobs` with the params from Step 1. It returns jobs with **configs already resolved** from prow build logs:
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

**If the user specifies a config name explicitly** (e.g. "use cluster-density.yaml"): still call `orion-mcp:discover_jobs` to get the `metadata`/`input_vars` for that config. Pass the user-specified config as `config_name` and the discovered `metadata` as `input_vars`.

**If `configs` is empty** for a job (prow artifacts expired or unavailable): use `orion-mcp:get_orion_configs` to list available configs, then match by benchmark name (e.g. benchmark `cluster-density-v2` → config `cluster-density.yaml`). If no obvious match, ask the user which config to use.

## Step 3: Call orion-mcp

Build `input_vars` from the job's `metadata` as a JSON string. Comma-join config files for multi-config tools.

**Always run Steps 1-2 (`orion-mcp:discover_jobs`) before calling any tool — EXCEPT these three which need no discovery:**
- `orion-mcp:get_orion_configs` — lists configs, no ES needed
- `orion-mcp:get_release_date` — date lookup only
- `orion-mcp:get_orion_metrics_with_meta` — reads config YAML locally, pass `config_name` directly, no `input_vars` needed

**For PR analysis** (`orion-mcp:openshift_report_on_pr`): run `orion-mcp:discover_jobs` with `job_type="pull"` and **no workload filter** — a PR can trigger payload, control-plane, and networking jobs. Pass all returned configs to `openshift_report_on_pr`.

| Intent | Tool |
|---|---|
| "has X regressed" | `orion-mcp:has_openshift_regressed` |
| "networking regressions" | `orion-mcp:has_networking_regressed` |
| "inspect nightly" | `orion-mcp:has_nightly_regressed` |
| "show metric" / "compare versions" | `orion-mcp:openshift_report_on` |
| "get raw values for a metric" | `orion-mcp:get_orion_performance_data` |
| "health check" / "overall performance summary" | `orion-mcp:get_performance_summary` |
| "correlate X with Y" | `orion-mcp:metrics_correlation` |
| "what metrics does X track" | `orion-mcp:get_orion_metrics` |
| "thresholds / directions for metrics" | `orion-mcp:get_orion_metrics_with_meta` |
| "analyze PR" / "check PR impact" | `orion-mcp:openshift_report_on_pr` |
| "list configs" / "what benchmarks exist" | `orion-mcp:get_orion_configs` |
| "release date for X" | `orion-mcp:get_release_date` |

Multiple configs: pass comma-separated. All share same input_vars. Different input_vars → separate calls.

**Networking intent**: Networking configs (`node-density-cni.yaml`, `udn-density-pods.yaml`, `udn-*`) are run inside **payload jobs**. Call `orion-mcp:discover_jobs` with `workload="payload"`, then filter the returned `configs` list to keep only those matching `*cni*`, `*udn*`, `*cudn*`. Pass only those to `orion-mcp:has_networking_regressed`.

## When discover_jobs Returns Unexpected Results

If `configs` is empty (prow artifacts expired or job hasn't run recently):
1. Call `orion-mcp:get_orion_configs` to list all available configs
2. Match by benchmark name — e.g. benchmark `cluster-density-v2` → config `cluster-density.yaml`
3. If no clear match, ask the user which config to use

If `orion-mcp:discover_jobs` returns no jobs at all, the version/platform/workload combination may not exist. Relax filters (drop `scale`, then `workload`) or ask the user to confirm the job exists.

## Example

**"has 4.22 regressed?"**
1. Parse: version=4.22, no other filters → defaults: workload=payload, scale=6, platform=AWS, cluster_type=self-managed
2. `orion-mcp:discover_jobs(version="4.22", platform="AWS", cluster_type="self-managed", workload="payload", scale=6)` → returns jobs with configs already resolved
3. `orion-mcp:has_openshift_regressed(config_name="cluster-density.yaml,node-density.yaml,node-density-cni.yaml,crd-scale.yaml,udn-density-pods.yaml", input_vars='{"platform":"AWS","workerNodesCount":"6",...}', version="4.22")`
