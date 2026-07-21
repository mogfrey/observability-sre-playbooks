# Observability architecture decisions

## Telemetry ownership

The platform team operates shared collection, storage, access and standards. Service teams remain responsible for instrumenting their code, defining meaningful SLIs, controlling telemetry cardinality and responding to service alerts.

This avoids a common anti-pattern where an observability team becomes the owner of every application’s reliability simply because it owns Grafana.

## Collection model

Applications and agents send OTLP to a gateway tier. The gateway applies memory limits, resource metadata and batching before exporting signals to their backends.

A production design should normally use:

- local or node-level collectors where host, file or Kubernetes metadata is required;
- gateway collectors for centralized processing and backend export;
- multiple gateway replicas spread across failure domains;
- bounded sending queues and retry intervals;
- load balancing that preserves any signal-specific requirements;
- health, queue and drop metrics for the telemetry pipeline itself.

## Signal backends

| Signal | Example backend | Primary use |
|---|---|---|
| Metrics | Prometheus or Mimir | SLIs, alerts, trends and capacity |
| Logs | Loki | Event detail and contextual investigation |
| Traces | Tempo | Request path, dependency latency and causal investigation |
| Visualization | Grafana | Correlation and operational workflows |

Backends should not be selected only because they integrate visually. Retention, tenancy, query isolation, failure recovery, object-store dependency, cardinality and cost need explicit design decisions.

## Reliability objectives

The example uses request success as an occurrence-based availability SLI. A real service may need multiple objectives, including:

- successful request ratio;
- latency below a customer-relevant threshold;
- data freshness;
- job completion correctness;
- durable ingestion or delivery.

An infrastructure metric is an SLI only when it directly represents a customer outcome. CPU and pod readiness remain important diagnostic signals but are usually poor availability objectives.

## Alerting

The repository separates:

- **page alerts** for rapid error-budget consumption requiring immediate human action;
- **ticket alerts** for persistent but slower budget consumption;
- **diagnostic alerts** for the telemetry pipeline itself.

Every enabled alert should identify an owner, severity, expected action and runbook. Alerts without an operational response should be dashboards, reports or removed.

## Cardinality and cost

High-cardinality dimensions such as raw customer IDs, request IDs, unbounded URLs and exception messages should not become metric labels. They increase cost and can destabilize metrics systems.

Use traces and logs for high-detail dimensions, with privacy controls and appropriate retention. Apply sampling and filtering according to service criticality and investigation value.

## Failure modes

Important failure scenarios include:

- Collector memory pressure and queue saturation;
- backend throttling or unavailability;
- DNS, TLS, proxy or private-endpoint failure;
- object-store latency or permission loss;
- label or schema changes silently breaking dashboards and alerts;
- telemetry loops and retry amplification;
- one tenant exhausting shared query or ingestion capacity;
- loss of observability during the same incident affecting the application.

The observability platform needs its own SLOs and independent monitoring path for critical failures.

## Security

- Use workload identity or injected short-lived credentials.
- Encrypt every signal in transit and at rest.
- Separate tenants and data classifications.
- Redact credentials, tokens, personal data and sensitive payloads before export.
- Audit administrative and query access.
- Treat trace attributes and logs as potentially sensitive even when metrics are aggregated.
