# Orion Troubleshooting Guide: No Data Returned

## Problem Summary

You have an Orion configuration targeting an Elasticsearch instance but are getting no data returned. This guide walks through systematic debugging steps.

## Environment Details

- **Elasticsearch URL:** https://es.example.com
- **Authentication:** Basic auth (user: `admin`, pass: `secret123`)
- **Benchmark Index:** `ripsaw-kube-burner-*`
- **Metadata Index:** `perf_scale_ci*`

---

## Step 1: Verify Elasticsearch Connectivity

Before debugging Orion itself, confirm that your Elasticsearch instance is reachable and your credentials work.

```bash
# Basic connectivity test
curl -u admin:secret123 https://es.example.com

# Check cluster health
curl -u admin:secret123 https://es.example.com/_cluster/health?pretty
```

If you get a connection error, check:
- Network access / VPN requirements
- Whether the ES instance uses a self-signed certificate (add `-k` or `--insecure` flag if so)
- Firewall or proxy rules blocking access

```bash
# With self-signed certificate
curl -k -u admin:secret123 https://es.example.com/_cluster/health?pretty
```

---

## Step 2: Verify the Indices Exist

Confirm that the indices you are referencing actually exist in Elasticsearch.

```bash
# List all indices matching the benchmark pattern
curl -u admin:secret123 https://es.example.com/_cat/indices/ripsaw-kube-burner-*?v

# List all indices matching the metadata pattern
curl -u admin:secret123 https://es.example.com/_cat/indices/perf_scale_ci*?v
```

If no indices are returned, the index names may be wrong. List all available indices to find the correct names:

```bash
# List all indices (can be large output)
curl -u admin:secret123 https://es.example.com/_cat/indices?v&s=index

# Search for indices containing keywords
curl -u admin:secret123 https://es.example.com/_cat/indices?v | grep -i "kube-burner"
curl -u admin:secret123 https://es.example.com/_cat/indices?v | grep -i "perf_scale"
```

Common issues:
- Index name typos or case sensitivity differences
- Indices may have been rolled over or use date-based naming (e.g., `ripsaw-kube-burner-2024.01.15`)
- The wildcard pattern may not match the actual index naming convention

---

## Step 3: Verify Data Exists in the Indices

Even if the indices exist, they may be empty or may not contain data in the expected time range.

```bash
# Check document count in benchmark index
curl -u admin:secret123 https://es.example.com/ripsaw-kube-burner-*/_count?pretty

# Check document count in metadata index
curl -u admin:secret123 https://es.example.com/perf_scale_ci*/_count?pretty

# Sample a few documents to inspect their structure
curl -u admin:secret123 https://es.example.com/ripsaw-kube-burner-*/_search?pretty&size=2

# Sample metadata documents
curl -u admin:secret123 https://es.example.com/perf_scale_ci*/_search?pretty&size=2
```

---

## Step 4: Verify Field Names and Mappings

Orion queries specific fields. If the field names in your index do not match what Orion expects, no data will be returned.

```bash
# Check the mapping (field schema) of the benchmark index
curl -u admin:secret123 https://es.example.com/ripsaw-kube-burner-*/_mapping?pretty

# Check the mapping of the metadata index
curl -u admin:secret123 https://es.example.com/perf_scale_ci*/_mapping?pretty
```

Look at the field names in the mapping output and compare them to what your Orion config references. Common fields Orion may look for include:
- `uuid` -- the unique run identifier
- `timestamp` -- the time the data was recorded
- `metricName` or `metric_name` -- the name of the metric
- `value` -- the metric value
- Workload-specific fields like `jobConfig.jobIterations`, `jobConfig.qps`, etc.

---

## Step 5: Check Your Orion Configuration

Review your Orion YAML config file for common issues.

### 5a: Verify ES connection settings in the config

Your Orion config should specify the Elasticsearch connection. Depending on how Orion is configured, you may set ES details via environment variables or in the config file itself.

```bash
# Set environment variables for Orion ES connection
export ES_SERVER=https://es.example.com
export ES_USERNAME=admin
export ES_PASSWORD=secret123
```

Or if using command-line flags:

```bash
orion cmd --es-url https://es.example.com \
          --es-username admin \
          --es-password secret123 \
          --config your-config.yaml
```

### 5b: Verify index names in the config

In your Orion YAML configuration, make sure the index names match exactly:

```yaml
# Example Orion config snippet -- verify these match your actual indices
benchmarkIndex: ripsaw-kube-burner-*
metadataIndex: perf_scale_ci*
```

### 5c: Check metric and filter definitions

The config defines which metrics to query and what filters to apply. If filters are too restrictive or reference non-existent field values, no data will be returned.

Verify that:
- Metric names referenced in the config actually exist in your data
- Filter values (e.g., cluster name, workload type, platform) match what is stored in ES
- Date ranges or time windows are not excluding all available data

---

## Step 6: Run Orion with Debug/Verbose Output

Orion may support verbose or debug logging that can show you the actual Elasticsearch queries being constructed.

```bash
# Try running with debug or verbose flags
orion cmd --config your-config.yaml --debug

# Or with log level
orion cmd --config your-config.yaml --log-level debug
```

Examine the debug output for:
- The actual Elasticsearch query being sent
- Any error messages from ES (auth failures, index not found, etc.)
- Whether the query is returning hits but post-processing is filtering them out

---

## Step 7: Manually Run the Elasticsearch Query

If you can see the query Orion generates (from debug output), run it manually to verify results:

```bash
# Example: manually query for data with a specific uuid or time range
curl -u admin:secret123 -X POST \
  "https://es.example.com/ripsaw-kube-burner-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "bool": {
        "must": [
          { "term": { "workload": "node-density" } }
        ]
      }
    }
  }'
```

Adjust the query fields and values to match what Orion would send based on your config.

---

## Step 8: Check for Time Range Issues

Orion typically filters data by a time range. If your data falls outside the configured range, nothing will be returned.

```bash
# Check the date range of available data
curl -u admin:secret123 -X POST \
  "https://es.example.com/ripsaw-kube-burner-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "aggs": {
      "min_date": { "min": { "field": "timestamp" } },
      "max_date": { "max": { "field": "timestamp" } }
    }
  }'
```

Compare the min and max dates with the time window your Orion config specifies. Adjust accordingly.

---

## Step 9: Verify UUID Linking Between Indices

Orion typically joins benchmark data with metadata using a shared identifier (commonly `uuid`). If the UUIDs in the benchmark index do not match any UUIDs in the metadata index, no joined results will be produced.

```bash
# Get sample UUIDs from the benchmark index
curl -u admin:secret123 -X POST \
  "https://es.example.com/ripsaw-kube-burner-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "aggs": {
      "uuids": {
        "terms": { "field": "uuid.keyword", "size": 10 }
      }
    }
  }'

# Check if those UUIDs exist in the metadata index
curl -u admin:secret123 -X POST \
  "https://es.example.com/perf_scale_ci*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "query": {
      "term": { "uuid.keyword": "PASTE_A_UUID_FROM_ABOVE" }
    }
  }'
```

If UUIDs do not match across indices, Orion will return no usable data.

---

## Step 10: Common Pitfalls Checklist

| Issue | What to Check |
|-------|---------------|
| SSL certificate errors | Use `--insecure` or configure the CA cert |
| Wrong ES port | Verify the URL includes the correct port (default 9200 for HTTP, 443 for HTTPS) |
| Index lifecycle management | Old indices may have been deleted or archived |
| Keyword vs text fields | Use `.keyword` suffix for exact match on text fields in filters |
| Empty metadata index | Metadata may be in a different index or not being ingested |
| Permissions | The ES user may not have read access to the specified indices |
| Orion version mismatch | Ensure your Orion version supports the config schema you are using |
| Python environment | Ensure Orion's dependencies are installed (`pip install orion-cli` or from source) |

---

## Quick Diagnostic Script

Run this all-in-one diagnostic to gather key information:

```bash
#!/bin/bash
ES_URL="https://es.example.com"
ES_USER="admin"
ES_PASS="secret123"
BENCH_INDEX="ripsaw-kube-burner-*"
META_INDEX="perf_scale_ci*"

echo "=== ES Cluster Health ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/_cluster/health?pretty

echo ""
echo "=== Benchmark Index Info ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/_cat/indices/${BENCH_INDEX}?v

echo ""
echo "=== Metadata Index Info ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/_cat/indices/${META_INDEX}?v

echo ""
echo "=== Benchmark Doc Count ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/${BENCH_INDEX}/_count?pretty

echo ""
echo "=== Metadata Doc Count ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/${META_INDEX}/_count?pretty

echo ""
echo "=== Sample Benchmark Document ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/${BENCH_INDEX}/_search?pretty&size=1

echo ""
echo "=== Sample Metadata Document ==="
curl -s -k -u ${ES_USER}:${ES_PASS} ${ES_URL}/${META_INDEX}/_search?pretty&size=1

echo ""
echo "=== Benchmark Date Range ==="
curl -s -k -u ${ES_USER}:${ES_PASS} -X POST \
  "${ES_URL}/${BENCH_INDEX}/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 0,
    "aggs": {
      "min_date": { "min": { "field": "timestamp" } },
      "max_date": { "max": { "field": "timestamp" } }
    }
  }'
```

Save this as `diagnose-orion-es.sh`, make it executable with `chmod +x diagnose-orion-es.sh`, and run it. The output will tell you immediately whether the problem is connectivity, missing indices, empty data, or field mismatch.

---

## Summary of Debugging Flow

1. Can you reach ES? (connectivity)
2. Do the indices exist? (index names)
3. Is there data in them? (document counts)
4. Do the fields match what Orion expects? (mappings)
5. Does the config reference the correct fields and values? (config review)
6. Is there a time range mismatch? (date range check)
7. Do UUIDs link across indices? (cross-index join)
8. What does Orion's debug output say? (verbose logging)

Work through these steps in order -- most "no data" issues are resolved by steps 2-4 (wrong index name, empty index, or mismatched field names).
