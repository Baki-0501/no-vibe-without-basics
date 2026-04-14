# Observability

## What is it?

Observability is the practice of understanding what is happening inside your system from the outside. It means you can answer the question "why is this broken?" without having to reproduce the problem locally or guess at the cause.

The three pillars of observability are:

- **Logs**: A record of discrete events in your system, timestamped and annotated with context. Think of them as the black box in an airplane — a running transcript of what happened, in order.
- **Metrics**: Numerical measurements aggregated over time — counts, rates, and distributions. Where logs tell you *what happened*, metrics tell you *how the system is performing*.
- **Traces**: The path a single request takes through your system. If logs are a transcript of the whole system, traces are the story of one request from entry to exit.

These three signals are not independent. A metric spike tells you something is wrong. You look at logs to find the error messages. You look at a trace to see exactly where in the request path the error occurred. Together they give you the full picture.

## Why it matters for vibe coding

AI-generated code ships without instrumentation. You ask AI to build a payment endpoint, it builds one, it works on your machine, you deploy it, and then it fails in production — silently, or with an error you can't trace back to its cause. Without observability, you are flying blind.

Specific failure modes:

**Without logs, AI bugs in production are invisible.** AI-generated code often fails in ways that don't produce obvious errors — wrong data returned, slow responses, partial failures. Without structured logs, you don't know what the code did when it failed. You only know that the feature stopped working. The AI cannot debug what it cannot see.

**Without request IDs in logs, you cannot correlate events across services.** A request hits your API, calls a database, then calls a third-party service, and fails. Without a `request_id` that travels with the request through every log line, you cannot reconstruct what happened. AI almost never includes request ID propagation unless you explicitly ask for it.

**Without metrics, you don't know something is wrong until users complain.** AI introduces subtle performance degradation — a new query that wasn't optimized, a loop that wasn't paginated. Without metrics, the first signal is a user report. With metrics, you catch it before anyone notices.

**Without distributed tracing, debugging microservices is archaeology.** If your AI-generated service calls three other services and one of them is slow, you cannot figure out which one without trace IDs. Attaching a profiler to each service manually is not scalable. This is why OpenTelemetry exists.

**AI generates logs that look human-readable but are useless for machines.** AI loves `console.log("Processing order...")` and `logger.info("Order processed successfully")`. These look informative but cannot be queried, filtered by severity, or parsed by log aggregation tools. Structured JSON logs are what production systems actually need.

## The 20% you need to know

### Structured logging

Human-readable logs (`Order processed successfully`) are written for humans to read one at a time. Structured logs are written for machines to read in bulk. The difference:

```
# Human-readable (AI's default)
[2026-04-13 10:15:32] INFO: Processing order 12345

# Structured (what production needs)
{"timestamp":"2026-04-13T10:15:32.000Z","level":"INFO","request_id":"req-abc-123","order_id":"12345","event":"order_processing_started","duration_ms":null}
```

Structured logs use key-value pairs. Every log line is a JSON object. This means you can:
- Search logs for `level:ERROR` across a million lines in milliseconds
- Filter by `request_id` to get every log line for a specific user request
- Aggregate on `event` to count how many orders were processed per hour
- Alert on `duration_ms > 5000` automatically

Every log line should contain enough context to understand it in isolation — at minimum: timestamp, log level, a message, and a request ID. Context fields (user ID, order ID, session ID) go in additional fields.

**Log levels:**

- `DEBUG`: Detailed diagnostic information. Never in production normally, enabled when investigating a specific issue.
- `INFO`: Normal operational events — a request started, a job completed. Expected to happen routinely.
- `WARN`: Something unexpected happened but the system handled it. A retry succeeded, a fallback was used. This is your early warning signal.
- `ERROR`: Something failed. The request returned an error response, a database call timed out. Requires investigation.

Use `WARN` more than you think. Anything that required a workaround, a retry, or a fallback is a `WARN`. These often predict larger failures.

### Metrics

Metrics are numerical measurements over time. There are three fundamental types:

**Counter**: A value that only goes up. Number of requests received, number of orders placed, number of errors. Counters are reset when the service restarts, so the right way to use them is as rates: "requests per second" not "total requests ever."

**Gauge**: A value that can go up and down. Current memory usage, number of active connections, queue depth. Gauges are useful for monitoring resource utilization and capacity.

**Histogram**: A distribution of values. Request latency, response body size, database query duration. A histogram buckets values into ranges (e.g., `<10ms`, `10-50ms`, `50-100ms`, `>100ms`) and counts how many observations fall into each bucket. This lets you answer "what is the 95th percentile latency?" which is more useful than the average.

The histogram is the most powerful metric type and the one AI most commonly omits. When you only track average latency, you miss the tail. The users whose requests take 2 seconds are the ones who leave and tell others.

### Distributed tracing

In a monolithic application, a single log file tells the whole story. In a microservices architecture, one request might touch five services — your API, an auth service, a database, a cache, and a third-party payment processor. A trace tracks a single request's journey across all of them.

A **trace** is made of **spans**. Each span represents a unit of work — an HTTP request, a database query, a function call. Spans have a name, a start time, an end time, and optionally attributes (tags with extra context). Spans can be nested: the `handle_order` span contains the `validate_payment` span, which contains the `call_payment_gateway` span.

The trace ID is a unique identifier that travels with the request through every service. Service A generates a trace ID and includes it in its log output and in every outbound HTTP call's headers. Service B receives the request, extracts the trace ID, and uses it as its own trace ID. All spans across all services share the same trace ID, letting you reconstruct the full request path.

OpenTelemetry (OTel) is the open standard for collecting traces, metrics, and logs in a vendor-neutral way. Instead of coupling your code to Datadog or Grafana, you instrument with the OpenTelemetry SDK — it collects the data and forwards it to an **OpenTelemetry Collector**, which then exports to your backend of choice. The key insight: you write instrumentation once and can switch backends without rewriting your code.

### Alerting basics

Alerts are rules that trigger notifications when a metric crosses a threshold. An alert without a runbook — a documented response procedure — is not an alert, it's a smoke detector with no fire extinguisher.

Good alerts have three things:
1. **A clear threshold**: "Error rate > 5% for 5 minutes" — not "something might be wrong."
2. **A runbook**: When the alert fires, the on-call engineer should know exactly what to check first.
3. **A escalation path**: If the first response doesn't acknowledge and resolve within N minutes, escalate.

Alerts that fire constantly without resolution become ignored — this is called alert fatigue. If your alert fires more than a few times per week and nobody is actively fixing the underlying cause, either the threshold is wrong or the alert should not exist.

### SLIs, SLOs, and SLAs

These are the vocabulary of reliability:

**SLI (Service Level Indicator)**: The quantitative measure of a service's behavior. "Percentage of requests that return in under 200ms," "percentage of requests that return a 2xx status code." An SLI is what you actually measure.

**SLO (Service Level Objective)**: The target value for an SLI. "99% of requests return in under 200ms." An SLO is what you promise to your users (or your team).

**SLA (Service Level Agreement)**: The contractual version of an SLO — what happens if you miss the target. Often tied to financial penalties or service credits. SLAs are between you and your customers. SLOs are the internal target that keeps you within your SLA.

**Error budgets** are the practical tool. If your SLO is 99% availability and you have 1% error budget, and you are using 0.8% of that budget in a 30-day window, you know you are in good shape. When you are at 0.95%, you know you are one incident away from blowing the budget. Error budgets tell you when to slow down and fix reliability issues versus when you can ship faster.

For AI-generated code, you do not need to define formal SLIs/SLOs/SLAs from day one. But you do need to instrument enough to know when your AI-generated service is performing within acceptable bounds.

## Hands-on exercise

**Instrument a Python script with structured logging, metrics, and simulated distributed trace context — in 15 minutes.**

Uses the `logging` standard library and `prometheus_client`. No OpenTelemetry SDK install required.

1. Set up:
```bash
mkdir observability-lab && cd observability-lab
python -m venv venv && source venv/bin/activate
pip install prometheus_client
```

2. Create `app.py`:
```python
import logging
import json
import time
import random
from prometheus_client import Counter, Histogram, generate_latest

logging.basicConfig(level=logging.INFO, format="%(message)s")

def logger_with_context(log, request_id, **context):
    """Emit one structured JSON log line."""
    print(json.dumps({
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S.000Z", time.gmtime()),
        "level": log.levelname,
        "request_id": request_id,
        **context
    }))

request_counter = Counter("http_requests_total", "Total HTTP requests", ["method", "endpoint", "status"])
request_latency = Histogram("http_request_duration_seconds", "Request latency", ["endpoint"])
order_counter   = Counter("orders_total", "Total orders processed", ["status"])
order_latency   = Histogram("order_processing_seconds", "Order processing time")

def process_order(request_id, order_id, amount):
    logger_with_context(logging.INFO, request_id,
        event="order_processing_started", order_id=order_id, amount=amount)

    start = time.time()
    time.sleep(random.uniform(0.05, 0.2))

    if random.random() < 0.1:  # ~10% failure rate
        logger_with_context(logging.ERROR, request_id,
            event="order_processing_failed", order_id=order_id, error="payment_declined")
        order_counter.labels(status="failed").inc()
        return {"status": "error", "order_id": order_id}

    duration = time.time() - start
    logger_with_context(logging.INFO, request_id,
        event="order_processing_completed", order_id=order_id,
        duration_ms=round(duration * 1000, 2))
    order_counter.labels(status="success").inc()
    order_latency.observe(duration)
    return {"status": "success", "order_id": order_id}

def handle_request(request_id, endpoint):
    request_counter.labels(method="POST", endpoint=endpoint, status="200").inc()
    start = time.time()
    result = process_order(request_id, f"order-{random.randint(1000,9999)}", 99.99) if endpoint == "/orders" else {"status": "ok"}
    request_latency.labels(endpoint=endpoint).observe(time.time() - start)
    logger_with_context(logging.INFO, request_id,
        event="request_completed", endpoint=endpoint,
        duration_ms=round((time.time() - start) * 1000, 2))
    return result

if __name__ == "__main__":
    for i in range(5):
        handle_request(f"req-{i:04d}", "/orders")
    print("\n# Prometheus metrics:")
    print(generate_latest().decode("utf-8"))
```

3. Run and verify:
```bash
python app.py
```

**What to check:**
- Each output line is a JSON object with `timestamp`, `level`, `request_id`, `event`
- Filter output for `order_processing_failed` to see simulated errors
- The `/metrics` block at the end contains `orders_total{status=...}` counters and `order_processing_seconds_bucket` histogram data
- Each `request_id` appears on every log line for that request — this is the simulated trace ID

**Success criteria:**
- At least one `order_processing_started` and one `order_processing_completed` event visible
- `/metrics` output includes `orders_total` and `order_processing_seconds`

## Common mistakes

**Mistake 1: Logging without log levels.**

What happens: Everything is logged at `INFO` or `DEBUG`. When something breaks, you cannot filter to just errors. Your logs are flooded with routine events and the signal is buried.

Why it happens: AI generates logs like `logger.info("Request received")` for everything, treating `INFO` as the default level for all messages. It does not think about severity classification.

How to fix: Every log statement should have an intentional level. `WARN` for things that required workarounds. `ERROR` for things that failed. If it is routine operational data, it is `INFO`. If it is diagnostic detail for investigating a known issue, it is `DEBUG`.

**Mistake 2: Logs without correlation IDs.**

What happens: You have thousands of log lines but no way to group them by request. A request fails, you see ten ERROR lines from ten different components, but you cannot tell which ERROR lines belong to the same request.

Why it happens: AI generates logging calls without thinking about request context propagation. Each function logs its own messages but doesn't pass a request ID through the call chain.

How to fix: Establish a request ID at the entry point of every request and thread it through every function call. Every log line for that request includes it. In Python, use `logging.LoggerAdapter` or `contextvars` to make the request ID available without passing it as an argument to every function.

**Mistake 3: Metrics without histograms.**

What happens: You track average latency. Your service looks healthy. A small percentage of your users are experiencing 5-second latencies, but your average is 200ms, so nobody notices. Until they leave.

Why it happens: Averages hide outliers. AI generates counter and gauge metrics naturally because they are simple to explain. Histograms require bucketing decisions and are slightly more complex.

How to fix: Add latency histograms for every external call — HTTP requests, database queries, cache lookups. Define buckets that make sense for your SLA: at minimum `[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]` seconds. This lets you compute p50, p95, and p99 latencies.

**Mistake 4: Not logging failure paths.**

What happens: Your logs show that requests succeeded. Users report that requests are failing. The error is not in your code path because the code that handles errors never logged that it was handling an error.

Why it happens: AI generates logging for the happy path — "order processed" — but skips logging for the error path. When an exception is caught and handled, or a fallback is used, those events are invisible.

How to fix: Every catch block that handles an exception should log at `WARN` or `ERROR`. Every fallback path should log. Every retry should log. If you did not log it, you cannot see it.

**Mistake 5: Creating alerts without runbooks.**

What happens: An alert fires at 2am. The on-call engineer stares at a dashboard and doesn't know what to do. The alert keeps firing because nobody knows how to resolve it. Eventually the engineer guesses and either fixes the symptom or misses the real cause.

Why it happens: It is easy to set up a metric threshold. It is hard to write a clear runbook. AI can generate the alert rule. It cannot generate the institutional knowledge about what to do when it fires.

How to fix: For every alert, write the first-response steps before you ship the alert. At minimum: "When this fires, check these three dashboards. If you see X, do Y." Alerts without runbooks generate alert fatigue and slow down incident response.

## AI-specific pitfalls

**AI generates code with no instrumentation at all.** This is the most common AI observability failure. AI builds a working feature with zero logs, zero metrics, and zero traces. You deploy it, something breaks, and you have no visibility into what the code is doing. When you ask AI to build anything that will run in production, specify that you want instrumentation built in from the start.

**AI generates logs without request IDs.** AI will log at the INFO level within functions without including any correlation ID. When you search logs for an error, you cannot reconstruct the request that caused it. You must explicitly instruct AI: "Every log line must include the request ID passed as a parameter."

**AI does not understand why structured logs matter.** AI generates human-readable string logs because they are easier to write and look nicer in a terminal. AI does not understand that production log aggregation tools (Datadog, Splunk, Loki) cannot efficiently filter or search unstructured logs. When prompting AI to add logging, specify: "Use structured JSON logs with these fields: timestamp, level, request_id, event, and relevant context fields."

**AI generates dashboards with no actionable alerts.** AI can generate a Grafana dashboard definition. It will fill it with every metric it can think of, organized into panels, with pretty visualizations. What it will not generate is a focused set of alerts with clear thresholds and runbooks. Treat AI-generated dashboards as a starting point for triage, not a replacement for thoughtful alerting.

**AI does not instrument failure paths by default.** AI tends to log the happy path ("Order placed successfully"). It rarely logs the error path ("Order rejected: inventory unavailable"). When reviewing AI-generated code, check every error handler and ask: "Does this log what went wrong?" If there is a try/except, there should be a log line in the except block.

## Quick reference

### Structured log fields

| Field | Always present? | Notes |
|---|---|---|
| `timestamp` | Yes | ISO 8601 format (`2026-04-13T10:00:00.000Z`) |
| `level` | Yes | `DEBUG`, `INFO`, `WARN`, `ERROR` |
| `request_id` | Yes (in request context) | Propagated from entry point |
| `event` | Yes | Short past-tense identifier: `order_placed`, `payment_failed` |
| `message` | Recommended | Human-readable summary (can be derived from `event`) |
| Context fields | As needed | `user_id`, `order_id`, `duration_ms`, `error` |

### Log level decision tree

```
Should this event be logged?
  |
  +-- Is it an error that requires investigation?
  |     YES -> ERROR
  |
  +-- Is it something unexpected that the system handled gracefully?
  |     YES -> WARN  (fallback used, retry succeeded, degraded mode active)
  |
  +-- Is it routine operational data?
  |     YES -> INFO  (request started, job completed, config loaded)
  |
  +-- Is it diagnostic detail for a known issue?
  |     YES -> DEBUG  (only enabled when actively investigating)
```

### Metric types

| Type | Goes up and down? | Use case | Example |
|---|---|---|---|
| Counter | No (only increments) | Events that count | `requests_total`, `errors_total` |
| Gauge | Yes | Resource utilization | `memory_bytes`, `active_connections` |
| Histogram | N/A (distrib.) | Latency, sizes | `request_duration_seconds_bucket` |

### SLI / SLO / SLA at a glance

| Term | What it is | Who sets it |
|---|---|---|
| SLI | The metric you measure | You (based on what's measurable) |
| SLO | The target you promise | You (internal target) |
| SLA | The contractual obligation | Legal/Customer facing |
| Error budget | `1 - SLO` — how much failure you can afford | Derived from SLO |

### OpenTelemetry signal types

| Signal | What it measures | In OTel: |
|---|---|---|
| Traces | Request path through services | `trace-sdk` |
| Metrics | Counters, gauges, histograms | `metrics-sdk` |
| Logs | Structured event records | `logs-sdk` |

## Go deeper

- [OpenTelemetry Python Documentation](https://opentelemetry.io/docs/instrumentation/python/) (verified 2026-04) — Official instrumentation guide for Python. Covers traces, metrics, and logs with the OTel SDK.
- [Prometheus: Writing Exporters](https://prometheus.io/docs/instrumenting/writing_exporters/) (verified 2026-04) — Best practices for exposing custom metrics in Prometheus format. Useful even if you are not building an exporter.
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/) (verified 2026-04) — How to build dashboards that actually help during incidents, not just look informative.
- [Google SRE Workbook: SLO Engineering](https://sre.google/workbook/slo-engineering/) (verified 2026-04) — Free online book chapter on defining SLIs, SLOs, error budgets, and choosing the right targets.
- [Structured Logging: Why and How](https://www.datadoghq.com/blog/structured-logging/) (verified 2026-04) — Practical guide to structured logging with concrete examples in multiple languages.
