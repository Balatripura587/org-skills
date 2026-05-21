# Orion "No Data Returned" Troubleshooting Guide

## Environment Details

- **ES Server**: `https://es.example.com`
- **Authentication**: Basic auth (user: `admin`, pass: `secret123`)
- **Benchmark Index**: `ripsaw-kube-burner-*`
- **Metadata Index**: `perf_scale_ci*`

## Debugging Strategy

When Orion returns no data, the root cause is almost always one of these:

1. Elasticsearch is unreachable or authentication is failing
2. The index patterns do not match any actual indices in the cluster
3. The metadata filters in your Orion YAML config are too restrictive or use incorrect field names
4. The lookback period does not cover the time range when data was ingested
5. The metric field names or values in your config do not match the actual ES schema

Work through the steps below in order. Each step builds on the previous one, so do not skip ahead.

---

## Step 1: Verify Elasticsearch Connectivity

Before anything else, confirm that you can reach the ES cluster and authenticate successfully.

```bash
curl -s -u "admin:secret123" \
  "https://es.example.com/_cluster/health" | python3 -m json.tool
```

**What to look for:**
- A `200 OK` response with `cluster_name` and `status` fields proves connectivity and authentication
- `status: green` or `status: yellow` means the cluster is healthy enough to query
- `status: red` means some shards are unavailable; data may be partially missing
- A `401` or `403` response means credentials are wrong or the user lacks permissions
- A connection error or timeout means the URL is wrong, a firewall is blocking access, or the cluster is down

**If authentication fails**, double-check the username/password. Also verify whether the cluster requires HTTPS vs HTTP, and whether SSL certificate verification is an issue:

```bash
# Try with SSL verification disabled (for self-signed certs)
curl -s -k -u "admin:secret123" \
  "https://es.example.com/_cluster/health" | python3 -m json.tool
```

---

## Step 2: Verify That Your Indices Exist

Check whether the benchmark and metadata index patterns actually match indices in the cluster.

```bash
# Check benchmark indices
curl -s -u "admin:secret123" \
  "https://es.example.com/_cat/indices/ripsaw-kube-burner-*?v&h=index,docs.count,store.size"

# Check metadata indices
curl -s -u "admin:secret123" \
  "https://es.example.com/_cat/indices/perf_scale_ci*?v&h=index,docs.count,store.size"
```

**What to look for:**
- A list of index names with document counts confirms the pattern is correct
- If you see `0` documents, data may have been deleted or never ingested
- If you see no output or a `404` error, the index pattern does not match anything

**If no indices are found**, list all available indices and look for the right pattern:

```bash
# List all indices in the cluster
curl -s -u "admin:secret123" \
  "https://es.example.com/_cat/indices?v&h=index,docs.count&s=index" | head -50
```

Common mismatches include:
- `ripsaw-kube-burner-*` vs `ripsaw-*` vs `kube-burner-*`
- `perf_scale_ci*` vs `perf-scale-ci*` vs `perf_scale_ci-*`

---

## Step 3: Run a Basic Query to Confirm Data Access

Test that you can actually retrieve documents from each index.

```bash
# Test benchmark index
curl -s -u "admin:secret123" \
  "https://es.example.com/ripsaw-kube-burner-*/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"match_all": {}}, "size": 1}' | python3 -m json.tool
```

```bash
# Test metadata index
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"match_all": {}}, "size": 1}' | python3 -m json.tool
```

**What to look for:**
- `hits.total.value` should be greater than 0
- Examine the `_source` of the returned document to understand the schema (field names, nesting structure, etc.)
- Pay attention to whether fields like `benchmark`, `platform`, `ocpVersion` are at the top level or nested under a `metadata` object

---

## Step 4: Discover Available Benchmarks

Find what benchmark names actually exist in your metadata index.

```bash
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "benchmarks": {
        "terms": {
          "field": "benchmark.keyword",
          "size": 100
        }
      }
    }
  }' | python3 -c "
import sys, json
data = json.load(sys.stdin)
benchmarks = data['aggregations']['benchmarks']['buckets']
print('\nAvailable Benchmarks:')
print('='*60)
for b in benchmarks:
    print(f'{b[\"key\"]:40} {b[\"doc_count\"]:>6} runs')
"
```

**If this returns 0 benchmarks**, the field name might be different. Try these alternatives:

```bash
# Try jobName.keyword instead of benchmark.keyword
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "benchmarks": {
        "terms": {
          "field": "jobName.keyword",
          "size": 100
        }
      }
    }
  }' | python3 -c "
import sys, json
data = json.load(sys.stdin)
benchmarks = data['aggregations']['benchmarks']['buckets']
print('\nAvailable Benchmarks (via jobName):')
print('='*60)
for b in benchmarks:
    print(f'{b[\"key\"]:40} {b[\"doc_count\"]:>6} runs')
"
```

```bash
# Also try looking in the benchmark index directly
curl -s -u "admin:secret123" \
  "https://es.example.com/ripsaw-kube-burner-*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "benchmarks": {
        "terms": {
          "field": "jobName.keyword",
          "size": 100
        }
      }
    }
  }' | python3 -c "
import sys, json
data = json.load(sys.stdin)
benchmarks = data['aggregations']['benchmarks']['buckets']
print('\nAvailable Benchmarks in benchmark index (via jobName):')
print('='*60)
for b in benchmarks:
    print(f'{b[\"key\"]:40} {b[\"doc_count\"]:>6} runs')
"
```

---

## Step 5: Discover Available Platforms and OCP Versions

Verify what platform and version values exist, since these are common metadata filters.

```bash
# Find platforms
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "platforms": {
        "terms": {
          "field": "platform.keyword",
          "size": 20
        }
      }
    }
  }' | python3 -c "
import sys, json
data = json.load(sys.stdin)
platforms = data['aggregations']['platforms']['buckets']
print('\nAvailable Platforms:')
print('='*60)
for p in platforms:
    print(f'{p[\"key\"]:30} {p[\"doc_count\"]:>8} runs')
"
```

```bash
# Find OCP versions (optionally filter by benchmark)
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "versions": {
        "terms": {
          "field": "ocpVersion.keyword",
          "size": 50
        }
      }
    }
  }' | python3 -c "
import sys, json
data = json.load(sys.stdin)
versions = data['aggregations']['versions']['buckets']
print('\nOCP Versions with data:')
print('='*60)
for v in sorted(versions, key=lambda x: x['key'], reverse=True)[:20]:
    print(f'{v[\"key\"]:50} {v[\"doc_count\"]:>6} runs')
"
```

---

## Step 6: Examine the Document Schema

Get a sample document from each index to understand exactly what fields are available and how they are structured.

```bash
# Sample from benchmark index
curl -s -u "admin:secret123" \
  "https://es.example.com/ripsaw-kube-burner-*/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 1}' | python3 -m json.tool | head -80
```

```bash
# Sample from metadata index
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{"size": 1}' | python3 -m json.tool | head -80
```

```bash
# Get full index mapping to see all available fields and types
curl -s -u "admin:secret123" \
  "https://es.example.com/ripsaw-kube-burner-*/_mapping" | python3 -m json.tool | head -100
```

**What to look for:**
- Is `benchmark` a top-level field, or nested under `metadata`?
- Is the UUID field called `uuid`, `run_id`, or something else?
- Is the timestamp field `@timestamp` or `timestamp`?
- For kube-burner data: is there a `metricName` field? What are its values?
- Are metadata fields at the top level or nested (e.g., `metadata.platform` vs `platform`)?

---

## Step 7: Use the Orion Discovery Script

If you have the Orion skill installed, use the built-in discovery script for a more structured exploration. First, create (or update) your ES config file.

### 7a. Create/Update the Elasticsearch Config Asset

Create the file at `~/.orion/elasticsearch-config.yaml`:

```yaml
name: "My Performance ES Cluster"

connection:
  server_url: "https://es.example.com"
  benchmark_index: "ripsaw-kube-burner-*"
  metadata_index: "perf_scale_ci*"

authentication:
  type: "basic"
  username: "admin"
  password: "secret123"

settings:
  timeout: 30
  max_results: 10000
  verify_ssl: true

data:
  default_lookback: "15d"
  timestamp_field: "@timestamp"
  uuid_field: "uuid"
```

### 7b. Validate the Config

```bash
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/validate-es-asset.py ~/.orion/elasticsearch-config.yaml
```

This script will check:
- YAML syntax
- ES connectivity
- Index existence
- Data accessibility

### 7c. Run Discovery Commands

```bash
# Discover benchmarks
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml benchmarks

# Discover platforms
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml platforms

# Discover metrics for a specific benchmark (replace with your benchmark name)
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml metrics --benchmark cluster-density-v2

# Discover OCP versions for a benchmark
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml versions --benchmark cluster-density-v2

# Discover node configurations
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml node-config --benchmark cluster-density-v2

# Get a sample document
cd ~/.claude/skills/orion-regression-analysis && \
python3 scripts/discover-es-data.py --config ~/.orion/elasticsearch-config.yaml sample --benchmark cluster-density-v2
```

**Important**: The `--config` flag must come BEFORE the subcommand.

---

## Step 8: Check Your Orion Config for Common Mistakes

Once you know what data is actually in ES, compare it against your Orion YAML config. Here are the most common mistakes that lead to "no data":

### 8a. Wrong `benchmark.keyword` Value

The benchmark name in your config must exactly match what is in ES. This field is case-sensitive.

```yaml
# If ES has "cluster-density-v2", this will fail:
metadata:
  benchmark.keyword: cluster-density   # WRONG - missing "-v2"

# Correct:
metadata:
  benchmark.keyword: cluster-density-v2
```

### 8b. Wrong OCP Version Format

```yaml
# WRONG - unquoted number gets parsed differently
metadata:
  ocpVersion: 4.22

# CORRECT - quoted string
metadata:
  ocpVersion: "4.22"
```

Also verify the version format. Some clusters store the full version (e.g., `4.22.0-0.nightly-2026-05-01`) while Orion may use a prefix match.

### 8c. Metadata Filters Too Restrictive

Start by removing all metadata filters except `benchmark.keyword` and see if data comes back:

```yaml
tests:
  - name: debug-test
    metadata:
      benchmark.keyword: cluster-density-v2
      # Comment out ALL other filters for now:
      # platform: AWS
      # ocpVersion: "4.22"
      # masterNodesType: m6a.xlarge
      # workerNodesCount: 6
    metrics:
      - name: simple-test
        metricName: podLatencyQuantilesMeasurement
        metric_of_interest: P99
```

Then add filters back one at a time, running Orion after each addition, to find which filter is causing the mismatch.

### 8d. Schema Mismatch (Separate vs Unified Index)

If your metadata is embedded in the benchmark index (not a separate metadata index), you may need to adjust:

```yaml
# Pattern 1: Separate metadata index (perf_scale_ci*)
# Metadata fields are top-level: platform, ocpVersion, benchmark
metadata:
  benchmark.keyword: cluster-density-v2
  platform: AWS

# Pattern 2: Unified index (metadata embedded in ripsaw-kube-burner-*)
# Metadata may be nested: metadata.platform, jobName
metadata:
  jobName.keyword: cluster-density-v2    # NOT benchmark.keyword
  metadata.platform: AWS                 # Nested path
```

Use the sample documents from Step 6 to determine which pattern your cluster uses.

### 8e. Wrong Metric Field Names

```yaml
# Verify metricName values match what ES actually has
metrics:
  - name: apiserver-cpu
    metricName: containerCPU          # Must match metricName.keyword in ES
    metric_of_interest: value
    agg:
      value: cpu
      agg_type: avg
    labels.namespace.keyword: openshift-kube-apiserver
```

### 8f. Lookback Period Too Short

If data was ingested more than 15 days ago, the default lookback will miss it:

```bash
# Try a longer lookback
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --lookback=30d
```

---

## Step 9: Run Orion with Debug Mode

Once you have verified ES connectivity and data availability, run Orion with the `--debug` flag to see exactly what queries it generates:

```bash
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --lookback=15d \
  --debug
```

**What to look for in debug output:**
- The generated Elasticsearch queries (copy and run them manually via curl to verify)
- Whether UUIDs are being found in the metadata index
- Whether metric data is being retrieved for those UUIDs
- Any field name mismatches or query errors

---

## Step 10: Test the Generated Query Manually

Take the query from Orion's debug output and run it directly against ES:

```bash
# Example: test UUID lookup in metadata index
curl -s -u "admin:secret123" \
  "https://es.example.com/perf_scale_ci*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 5,
    "query": {
      "bool": {
        "must": [
          {"term": {"benchmark.keyword": "cluster-density-v2"}},
          {"range": {"@timestamp": {"gte": "now-15d"}}}
        ]
      }
    },
    "_source": ["uuid", "benchmark", "platform", "ocpVersion", "@timestamp"]
  }' | python3 -m json.tool
```

If this returns results, Orion should be able to find them too. If not, adjust the field names or date range until you get results, then update your Orion config to match.

---

## Summary: Debugging Checklist

Run through these checks in order, stopping when you find the problem:

| Step | Command / Check | What It Verifies |
|------|----------------|------------------|
| 1 | `curl _cluster/health` | ES is reachable and auth works |
| 2 | `curl _cat/indices/<pattern>` | Index patterns match real indices |
| 3 | `curl <index>/_search` (match_all) | Can retrieve documents from indices |
| 4 | Aggregation on `benchmark.keyword` | Benchmark names available in data |
| 5 | Aggregation on `platform.keyword`, `ocpVersion.keyword` | Metadata filter values exist |
| 6 | Sample document inspection | Schema matches your config assumptions |
| 7 | `validate-es-asset.py` + `discover-es-data.py` | Automated validation and discovery |
| 8 | Review Orion config YAML | No typos, correct field names, correct quoting |
| 9 | `orion --debug` | See generated queries and pinpoint failure |
| 10 | Manual query from debug output | Isolate exactly which query returns no data |

---

## Quick-Reference: Correct Orion Run Commands

Once you have fixed the issue, use one of these command patterns:

```bash
# Basic regression analysis with credentials in the URL
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --lookback=15d

# With input variables for OCP version templating
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --input-vars='{"version": "4.22"}' \
  --lookback=15d

# With debug output for continued troubleshooting
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --lookback=15d \
  --debug

# With visualization and text output
orion --config config.yaml --hunter-analyze \
  --es-server='https://admin:secret123@es.example.com' \
  --benchmark-index='ripsaw-kube-burner-*' \
  --metadata-index='perf_scale_ci*' \
  --lookback=15d \
  --output-format=text --viz \
  --save-output-path="results.txt"
```
