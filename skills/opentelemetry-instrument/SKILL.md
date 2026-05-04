---
name: opentelemetry-instrument
description: Instrument a service with OpenTelemetry — traces, metrics, and logs unified, with one OTLP collector and one vendor (Grafana Cloud, Honeycomb, Datadog). Covers Node, Go, Python, and the cardinality/sampling discipline that keeps bills sane. Use when adding observability to a service for the first time, or replacing fragmented Prometheus/Jaeger/loki setups.
---

# OpenTelemetry Instrumentation

OTel is the right answer for new services: vendor-agnostic, three signals (traces/metrics/logs) unified, growing ecosystem. The trap is footguns — easy to ship cardinality bombs, easy to drop spans on shutdown, easy to instrument so heavily that the trace becomes noise.

## When to use

- Greenfield service — instrument before you go to prod.
- Replacing a fragmented setup (Prometheus alone for metrics + Jaeger alone for traces + raw stdout for logs).
- Standardizing across many services so one tool/dashboard works for all.

## When NOT to use

- One existing service with deep Datadog/New Relic agent investment that already works — don't rip out unless cost forces it.
- Hyper-high-throughput data plane (>500k RPS per service) — sampling becomes the whole problem; use eBPF profiling alongside.

## Three signals, one collector

| Signal | What it tells you |
|---|---|
| **Traces** | Why this *one* request was slow. Distributed across services. |
| **Metrics** | What % of requests were slow over time. Aggregate. |
| **Logs** | The detail you forgot to put in a span attribute. |

Send all three to one **OTel Collector** running near your service (sidecar or daemonset on K8s). Collector batches, samples, and forwards to your backend. This insulates you from vendor changes — switch from Honeycomb to Grafana by changing the collector config, not the app.

## Node.js instrumentation (Express/Fastify/Next API routes)

```ts
// instrumentation.ts — load BEFORE any other code
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-http";
import { PeriodicExportingMetricReader } from "@opentelemetry/sdk-metrics";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: process.env.OTEL_SERVICE_NAME ?? "myservice",
    [ATTR_SERVICE_VERSION]: process.env.SERVICE_VERSION ?? "dev",
    "deployment.environment": process.env.NODE_ENV ?? "dev",
  }),
  traceExporter: new OTLPTraceExporter({ url: "http://localhost:4318/v1/traces" }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: "http://localhost:4318/v1/metrics" }),
    exportIntervalMillis: 30_000,
  }),
  instrumentations: [getNodeAutoInstrumentations({
    "@opentelemetry/instrumentation-fs": { enabled: false },  // noisy, low value
  })],
});

sdk.start();

process.on("SIGTERM", async () => {
  await sdk.shutdown();
  process.exit(0);
});
```

```bash
# Run the app with instrumentation loaded first
node --import ./instrumentation.js dist/server.js
```

`auto-instrumentations-node` covers HTTP, Express, Fastify, pg, mysql2, redis, ioredis, mongodb, aws-sdk, gRPC. You get spans for free; add custom spans only for business operations.

## Custom spans (the only ones worth writing manually)

```ts
import { trace } from "@opentelemetry/api";
const tracer = trace.getTracer("myservice");

export async function chargeCustomer(customerId: string, amountInPaise: number) {
  return tracer.startActiveSpan("charge_customer", async (span) => {
    span.setAttributes({
      "customer.id": customerId,
      "charge.amount_paise": amountInPaise,
      "charge.provider": "razorpay",
    });
    try {
      const result = await razorpay.charge(customerId, amountInPaise);
      span.setStatus({ code: 1 });
      return result;
    } catch (err) {
      span.recordException(err as Error);
      span.setStatus({ code: 2, message: (err as Error).message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

Wrap *business* operations, not framework operations. HTTP routing is auto-instrumented; "charge_customer" is the unit you care about.

## Cardinality: the bill killer

Every distinct combination of label values on a metric = one time series. Watch for these traps:

```ts
// BAD — every customer becomes a series
counter.add(1, { customer_id: "cus_xyz", endpoint: "/charge" });

// BAD — user-agent has thousands of variants
histogram.record(latency, { user_agent: req.headers["user-agent"] });

// BAD — error_message is unbounded
counter.add(1, { error_message: err.message });
```

Rule: **labels on metrics are bounded enums, not free-form strings.** Customer IDs and error messages belong in *trace attributes* (high cardinality is fine on traces — they're sampled, not aggregated).

```ts
// GOOD — bounded
counter.add(1, { endpoint: "/charge", status_class: "5xx" });

// Customer detail goes on the trace
span.setAttribute("customer.id", customerId);
```

If unsure: if the cardinality of a label is > 100, it should be a span attribute, not a metric label.

## Sampling (or how to afford this)

100% trace sampling at scale = bankrupt. Two strategies:

| Strategy | Where | Use when |
|---|---|---|
| **Head sampling** | At the SDK | Simple, cheap, drops detail before it leaves |
| **Tail sampling** | At the collector | Smart — keep all errors, slow requests, sample fast successes |

For most services start with **tail sampling at the collector**:

```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,   type: latency,     latency: { threshold_ms: 1000 } }
      - { name: rate,   type: probabilistic, probabilistic: { sampling_percentage: 5 } }
```

Result: 100% of errors, 100% of slow requests, 5% of normal traffic.

## Logs: connect them to traces

Use `slog` (Go), `pino` (Node), `structlog` (Python). Inject the active trace ID into every log line so a log → trace jump is one click in your backend.

```ts
import { context, trace } from "@opentelemetry/api";
import pino from "pino";

const logger = pino({
  mixin: () => {
    const span = trace.getSpan(context.active());
    if (!span) return {};
    const sc = span.spanContext();
    return { trace_id: sc.traceId, span_id: sc.spanId };
  },
});
```

## Anti-patterns

- **Custom spans for HTTP routing** — auto-instrumentation already does this. Wrapping it manually creates double spans.
- **High-cardinality metric labels** — customer IDs, error messages, full URLs. Bills explode, dashboards die.
- **Forgetting `await sdk.shutdown()` on SIGTERM** — last batch of spans is dropped.
- **Sampling at 100% in prod** — works until it doesn't; usually one viral endpoint takes down the collector.
- **Three vendors, three SDKs** — Datadog APM + Prometheus + Loki = three agents, three configs, three bills. OTel is the unifier.
- **Metric naming inconsistent** — agree once: `<service>_<unit>_<verb>` (e.g. `payments_charges_total`, `payments_charge_duration_seconds`). Drift makes dashboards fragile.
- **No `service.version` resource attribute** — can't tell if a perf regression started with a deploy. Always set it from CI build vars.
- **Disabling instrumentations and forgetting** — half-broken signal looks worse than no signal. Document what's off.
- **Trace attributes that include PII** — emails, phone numbers, plaintext tokens. Redact at SDK or collector.

## Verify it worked

- [ ] A request through the service produces one trace with parent-child spans across HTTP → DB → outbound HTTP.
- [ ] The trace ID appears in every log line for that request.
- [ ] Killing the service with `SIGTERM` flushes pending spans (no missing tail).
- [ ] Metrics dashboard shows: requests/sec, p50/p95/p99 latency, error rate — all queryable by `service.name` and `deployment.environment`.
- [ ] No metric has a label with > 100 values.
- [ ] Tail sampling: 100% of error traces are present; ~5% of success traces are sampled.
- [ ] OTel Collector running as sidecar/daemonset, app sends to localhost:4318 only.
- [ ] Switching backends (e.g. Honeycomb → Grafana) requires changes only to collector config, not app code.
- [ ] `service.version` is set from CI; visible on every span.
- [ ] No PII in attributes (spot-check 20 random traces).
