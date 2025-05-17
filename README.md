# Branch: fix-hostmetrics-process-filter

## Hypothesis

The SigNoz Host Metrics dashboard was showing **no values** for Host Name and OS Type because the `process` scraper in the `hostmetrics` receiver was misconfigured.

The default config was:

```yaml
process:
  include:
    match_type: regexp
    processes: {}
```

The key `processes` is **not valid** — the correct key is `names`. This caused the `otel-collector` to crash or silently ignore host-level metrics from the `process` scraper.

## What Was Attempted

The `otel-collector-config.yaml` file had the following configuration under the `hostmetrics` receiver:

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    root_path: /hostfs
    scrapers:
      cpu: {}
      load: {}
      memory: {}
      disk: {}
      filesystem: {}
      network: {}
      process:
        include:
          match_type: regexp
          processes: {}
```

After investigation, this was replaced with:

```yaml
process:
  include:
    match_type: regexp
    names: [".*"]
```

In addition, the container was named 'otel-collector' in the YAML, but the actual image used is'signoz/signoz-otel-collector', therefore logs and documentation should use the name'signoz-otel-collector'.

## Resolution Steps

1. Updated `otel-collector-config.yaml` to replace `processes` with `names`.
2. Verified file permissions and placement at:

```
/workspaces/signoz/clickhouse-setup/otel-collector-config.yaml
```
3. Restarted the stack with:

```bash
docker compose -f /workspaces/signoz/clickhouse-setup/docker-compose.yaml down
docker compose -f /workspaces/signoz/clickhouse-setup/docker-compose.yaml up -d --force-recreate
```

4. Waited for all containers to show `healthy` status.
5. Visited the Hosts tab in the SigNoz UI to verify correct metrics population.

## Test Result

After restart:

- The SigNoz UI showed `signoz-host` under **Host Name**
- OS type showed as `linux`
- Host-level metrics including CPU, Memory, and Load Avg were populated.

![hostmetrics](./hostmetrics-populated.png)

In the APM section, services such as `driver`, `frontend`, `customer`, and `redis` were also visible with latency and throughput values, confirming that metrics and traces were correctly ingested.

## Conclusion

The crash or empty display in the Hosts dashboard was caused by an invalid key under the `process` scraper. Updating the config and restarting the stack restored correct functionality.

Going forward:
- Always validate scraper configuration keys against the [OpenTelemetry Collector manual].(https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver)
- Be note that the SigNoz collector image is tagged'signoz-otel-collector', so use that name in logs, debugging, and documentation.
