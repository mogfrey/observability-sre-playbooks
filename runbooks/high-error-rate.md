# Runbook: High API error rate

## Alert meaning

The service is consuming its 99.9% availability error budget faster than the configured burn-rate threshold. This is a customer-impact signal, not merely an infrastructure warning.

## Immediate priorities

1. Confirm the alert is based on real traffic and not a telemetry or label change.
2. Measure customer impact by route, region, tenant and dependency.
3. Stop or roll back a change when the timing and evidence strongly correlate.
4. Protect the service from retry storms and overload.
5. Maintain an incident timeline while mitigation proceeds.

## Validate the signal

Check request volume, total errors and the error ratio together:

```promql
service:http_requests:rate5m
```

```promql
service:http_requests:error_ratio5m
```

```promql
sum by (service_namespace, service_name, http_response_status_code) (
  rate(http_server_request_duration_seconds_count[5m])
)
```

Confirm that the numerator and denominator use the same eligible request population. Exclude health checks, expected client failures and intentionally rejected traffic only when the SLI definition documents that choice.

## Determine scope

Break down errors by:

- route or RPC method;
- status code or exception type;
- deployment version;
- region, zone, cluster or node pool;
- tenant or customer segment, using privacy-safe labels;
- upstream dependency;
- newly enabled feature flag.

Useful questions:

- Did the error rate begin immediately after a deployment, configuration or infrastructure change?
- Are all instances affected or only a subset?
- Is latency rising before errors, suggesting saturation or dependency timeout?
- Is traffic materially above forecast?
- Are retries increasing load faster than useful throughput?

## Correlate telemetry

### Logs

Search for the dominant error signature and compare it with the last known-good period. Preserve correlation IDs and deployment metadata. Do not paste credentials, tokens or customer payloads into the incident channel.

### Traces

Inspect representative failed traces. Identify where time is spent and which span first reports an error. A downstream timeout reported at the API edge is not necessarily an API implementation fault.

### Infrastructure

Review saturation and health for:

- CPU throttling and memory pressure;
- pod restarts and unavailable replicas;
- connection-pool exhaustion;
- database locks, replication lag and storage latency;
- cache hit rate and evictions;
- queue depth and consumer lag;
- DNS, TLS and network errors;
- telemetry pipeline drops that might distort the signal.

## Mitigation options

Choose the fastest safe mitigation supported by evidence:

- roll back the latest release;
- disable a faulty feature flag;
- shift traffic away from an unhealthy zone or region;
- scale a saturated stateless tier within tested limits;
- reduce concurrency or apply load shedding;
- stop unbounded retries and enforce deadlines;
- fail over a dependency according to its runbook;
- restore a previous configuration.

Avoid scaling blindly when the bottleneck is a database, external dependency or lock. More callers can intensify the failure.

## Communication

State:

- the customer-visible symptom;
- affected scope;
- incident start time;
- current mitigation;
- next decision point;
- known data-integrity risk, if any.

Do not wait for root cause before communicating confirmed impact.

## Recovery verification

The incident is not resolved when pods turn green. Verify:

- error ratio returns below the alert threshold;
- latency and saturation return to expected ranges;
- queues and retries drain;
- dependency health is stable;
- synthetic and real-user paths succeed;
- no delayed data-integrity or reconciliation work remains.

## Follow-up

Document the trigger, contributing factors, why safeguards did not prevent impact and concrete actions. Prioritize work according to remaining error budget and recurrence risk rather than the length of the postmortem.
