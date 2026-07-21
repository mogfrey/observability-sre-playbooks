# Contributing

This public repository demonstrates observability and SRE patterns. Contributions must use synthetic service names and telemetry and must never include production logs, traces, dashboards, endpoints, tenant labels or credentials.

## Engineering expectations

- Define customer-centred SLIs before adding alerts.
- Keep metric labels bounded and privacy-safe.
- Link actionable alerts to an owner and runbook.
- Protect telemetry pipelines with memory limits, queues and measured retry behaviour.
- Explain retention, sampling, reliability and cost trade-offs.

## Validation

```bash
otelcol-contrib validate --config otel/collector.yaml
promtool check rules prometheus/recording-rules.yaml prometheus/alert-rules.yaml
```

Use environment variables or a secret manager for exporter endpoints and authentication. Never add real backend credentials to configuration examples.
