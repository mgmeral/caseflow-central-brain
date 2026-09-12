# Observability

## Purpose
This document defines shared observability expectations across CaseFlow repositories.

## What belongs here
Cross-repository logging, metrics, tracing expectations, and operational signal ownership.

## Who should use this
Engineers and AI agents implementing monitoring or incident diagnostics spanning repositories.

## What should NOT be stored here
Vendor-specific dashboard exports or temporary troubleshooting notes.

## Logging expectations
- `caseflow-be` sets an MDC `correlationId` per request (`CorrelationIdFilter`) and echoes it as the `X-Correlation-Id` response header on every response, including errors. Any client (FE, mobile) reporting an issue should capture and pass along this header.
- Credentials (SMTP/IMAP passwords, OAuth2 client secrets, JWT tokens) must never appear in logs on any repository — this is a hard rule, not a preference.

## Metrics expectations
- `caseflow-be` exposes Micrometer counters for inbound/outbound email (`EmailMetrics`) and standard Spring Boot Actuator metrics.
- `caseflow-ai-service` exposes `/actuator/health` and `/actuator/info`; model-specific readiness via `/api/ai/health/ready` and `/api/ai/health/models`.
- No shared metrics backend (Prometheus/Grafana) topology is documented yet across repositories.

## Tracing expectations
- No distributed tracing (OpenTelemetry/Zipkin/Jaeger) is implemented yet. Cross-repo correlation today is limited to the `X-Correlation-Id` header — treat this as the minimum bar, not the end state.

## Alerting expectations
TODO: Define this decision. *(No alerting rules or on-call ownership have been recorded yet.)*
