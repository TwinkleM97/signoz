
# Branch: fix-docker-stats-receiver-crash

## Hypothesis

The SigNoz app failed to start because the OpenTelemetry Collector ('signoz-otel-collector') was incorrectly set in place. The 'docker_stats' receiver was introduced in an attempt to gather Docker metrics, but it is not supported by the SigNoz collector program. As a result, the collector crashed instantly upon initialization.

##  What Was Attempted

To enable Docker metrics, the following receiver was added to `otel-collector-config.yaml`:

```yaml
receivers:
  docker_stats:
    endpoint: unix:///var/run/docker.sock
    collection_interval: 10s
```

This was also added to the `metrics` pipeline:

```yaml
service:
  pipelines:
    metrics:
      receivers: [otlp, docker_stats]
      processors: [batch]
      exporters: [clickhousemetricswrite]
```

The stack was restarted using Docker Compose.

## Problem Encountered

After the change, the `signoz-otel-collector` container crashed on startup. Logs showed:

```json
"error": "collector stopped unexpectedly"
```

This confirmed that the SigNoz collector image did not support the `docker_stats` receiver, and the service was shutting down due to invalid configuration.

## Resolution Steps

To resolve the issue:

1. I removed the 'docker_stats' block from the'receivers' section of 'otel-collector-config.yaml'.
2. I removed 'docker_stats' from the'metrics' pipeline in'service.pipelines'.
3. Rebuilt and restarted the Docker stack with the following command to ensure that all services were reset to the revised configuration.

```markdown
 **MANDATORY: Run this to apply the fixed config**
```bash
docker compose -f clickhouse-setup/docker-compose-minimal.yaml up -d --force-recreate

This command forcefully recreated all containers, ensuring no broken state persisted from the previous configuration.

## Test Result

After removing the unsupported `docker_stats` configuration and force-restarting the stack, the application started successfully. The UI was accessible and containers are up and running.

Logs showed no errors from the collector, and SigNoz operated normally with system metrics and dashboards functional.

## Conclusion

The test confirms that the SigNoz custom collector image does not include the `docker_stats` receiver by default. Including it in the configuration results in the entire application failing to start.

To collect Docker container metrics, one would need to:
- Build a custom OpenTelemetry Collector from the `otel-collector-contrib` repo including the `docker_stats` receiver, or
- Use an external metrics solution (e.g. cAdvisor + Prometheus) to push container metrics to SigNoz.

For now, removing the unsupported block allowed the platform to start correctly and verified the cause of failure.