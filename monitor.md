# Monitor — Observability, Logs, Metrics, Traces, Alerts

Set up or audit the observability stack for a service. Decides what to log, what to measure, what to trace, and what to alert on — so the next incident is detected in seconds, not hours.

The rule: **you can't fix what you can't see. Monitor the user's experience, not just the server's heartbeat.**

This skill is proactive: plan + install observability BEFORE the incident. For reacting to an active incident, use `/investigate` and `/incident`.

## Triggers
Use when: "monitor", "observability", "set up alerts", "logging", "metrics", "tracing", "dashboards", "SLO", "SLI", "add telemetry"
Proactively suggest when: the user ships a new service, promotes something to production for the first time, or asks "how would we know if this broke?"

---

## Step 0: Detect the Stack

```bash
# Tracing / metrics client libraries
grep -rE "sentry|datadog|new-relic|opentelemetry|prometheus|honeycomb" package.json go.mod Gemfile requirements.txt 2>/dev/null | head -5

# Log infra
grep -rE "pino|winston|zap|logrus|structlog|semantic_logger" package.json go.mod Gemfile requirements.txt 2>/dev/null | head -5

# Crash reporters
grep -rE "crashlytics|sentry|bugsnag|metrickit" **/*.{swift,kt,ts,js,py,rb} 2>/dev/null | head -5

# Dashboard tools
ls grafana/ dashboards/ 2>/dev/null
```

Answer:
- **Logging sink** — Datadog? Cloudwatch? Loki? Local file + Fluentbit?
- **Metrics** — Prometheus? Datadog? StatsD?
- **Tracing** — OpenTelemetry? Jaeger? Zipkin? Datadog APM? Sentry?
- **Crash reporting (mobile)** — Crashlytics? Sentry? MetricKit?
- **Alerting** — PagerDuty? Opsgenie? Slack only?

---

## Step 1: Start With SLIs, Not Infrastructure

Before instrumenting anything, ask: **what does "working" mean for the user?**

For each critical user flow, define a Service Level Indicator (SLI):
- **Availability** — `successful_requests / total_requests`
- **Latency** — `p99 < 500ms` for the checkout endpoint
- **Correctness** — `orders_with_correct_price / orders` (if the bug is business-logic)
- **Freshness** — `max_age_of_cached_data < 60s`

Then set a Service Level Objective (SLO): "99.9% of checkouts complete in < 2s over any 28-day window."

If you can't state SLOs, you can't alert on them. Start here, not with "log everything."

---

## Step 2: The Four Golden Signals (per service)

For every service, measure:

1. **Latency** — how long requests take (p50, p95, p99). Split by success vs error.
2. **Traffic** — requests per second.
3. **Errors** — rate of failed requests (by status code, by error type).
4. **Saturation** — how "full" the service is (CPU %, memory %, queue depth, thread pool utilization).

Build a dashboard that shows these four for every critical service. Nothing else on the first dashboard — it's the one on-call checks first.

---

## Step 3: Logging

### What to log
- **Request start + end** — with method, path, status, duration, user-id-hash
- **Every error** — with stack, context, request id
- **Every external call** — with upstream, duration, status
- **Every state transition** — auth granted/denied, order placed, payment captured

### What NOT to log
- Full request/response bodies (PII risk + volume)
- Tokens, passwords, API keys (ever)
- User emails, phone numbers, full names (hash them if you need correlation)
- `console.log` leftovers from development

### Structured, not string-formatted
```
❌  logger.info(`User ${userId} ordered ${itemId} for $${price}`)
✅  logger.info({ event: "order.placed", userId, itemId, priceUsd: price })
```

Structured logs are queryable. String logs are regex goo.

### Correlation
Every log line for a single request must share a correlation ID (request ID, trace ID). Without it, you can't reconstruct what happened.

---

## Step 4: Metrics

### Types — pick the right one
- **Counter** — monotonically increasing (requests_total, errors_total)
- **Gauge** — goes up and down (connection_pool_in_use, queue_depth)
- **Histogram** — distribution (request_duration_seconds) — gives you percentiles
- **Summary** — like histogram but client-side quantiles (rarely what you want)

### Labels — low cardinality
Label by: endpoint, status code, service. Do NOT label by: user id, request id, full URL with path params.

High cardinality labels explode metric storage costs and slow queries.

```
✅  http_requests_total{method="POST", route="/api/orders", status="201"}
❌  http_requests_total{method="POST", route="/api/orders/12345/items/67890", user="user_42"}
```

---

## Step 5: Tracing

Distributed traces show the path of a single request through every service.

Instrument:
- Every HTTP handler
- Every outbound HTTP / gRPC / DB call
- Every queue enqueue + dequeue
- Background jobs with `span.set_attribute('job.name', ...)`

Sampling:
- 100% of errors (tail-based sampling if your vendor supports it)
- 1-10% of successes (enough for perf analysis, cheap enough)

OpenTelemetry is the default — avoids vendor lock-in.

---

## Step 6: Alerts

### Alert on symptoms, not causes
```
❌  ALERT: CPU > 80%         (high CPU might be fine; the question is "are users affected?")
✅  ALERT: p99 latency > 2s   (directly visible to users)
✅  ALERT: error rate > 1%    (directly visible to users)
```

### Every alert has a runbook link
If the on-call can't figure out what to do in 30 seconds, the alert is broken. Write the runbook with `/docs`.

### Alert severity
- **Page (SEV 1/2)** — wake someone at 3 AM. Reserved for actual user impact.
- **Ticket / Slack (SEV 3)** — notify but don't wake. Fix during business hours.
- **Dashboard only** — awareness, no automatic notification.

### Multi-window + burn rate for SLO alerts
```
Page if: error budget burn rate > 14.4× for 5 minutes AND > 6× for 1 hour
  (This means you'd exhaust the month's error budget in 2 days)
```

Single-window alerts cause flapping. Multi-window balances speed vs noise.

### Don't have too many
If you get > 5 pages per week per person that aren't real incidents, alerts are too noisy. Tune or delete.

---

## Step 7: Mobile-Specific

For iOS and Android, the "server" is the user's device. You can't SSH in. Observability is:

- **Crash reporter** — Crashlytics, Sentry, MetricKit. Must capture non-fatal errors too, not just crashes.
- **Session telemetry** — app launch time, screen transitions, ANR / main-thread stalls.
- **Network telemetry** — request timing + error rates measured client-side.
- **Release health** — crash-free sessions %, crash-free users % per release.

Alerts:
- Crash-free users % drops below 99.5 in any 24h window
- ANR rate rises > 0.5%
- Any new crash affecting > 0.1% of sessions

---

## Step 8: Verify the Instrumentation

Instrumentation that never fired in a test never fires in production either.

```bash
# Generate synthetic traffic
curl ... && curl ... && curl ...

# Confirm:
# - Logs show up in the sink
# - Metrics appear in the metric system  
# - Traces span the expected services
# - An intentional error triggers the error alert
```

Never ship a new alert without confirming it fires when the condition is true. Fire a synthetic error in staging and watch the alert land.

---

## Step 9: Report

```
OBSERVABILITY AUDIT
══════════════════════════════════════
Service:       {name}
Stack:         Logs: {sink}, Metrics: {sink}, Traces: {sink}

SLOs DEFINED: {n}
  - {flow}: {SLI} at {target} / {window}

FOUR GOLDEN SIGNALS DASHBOARD: PRESENT / MISSING

LOGGING
  Structured:      YES / NO
  PII scrubbing:   CONFIRMED / UNVERIFIED
  Correlation ID:  PRESENT / MISSING

METRICS
  High-cardinality labels: {n found}  (list)

TRACING
  Coverage:        {% of routes instrumented}

ALERTS
  Total:           {n}
  Paging:          {n}
  With runbook:    {n}/{total}
  Tested:          {n}/{total}

GAPS
  - {gap 1}
  - {gap 2}

VERDICT: HEALTHY / NEEDS WORK / CRITICAL
```

---

## Hard rules

- **Never alert without a runbook.**
- **Never log secrets or PII.** Ever.
- **Never ship a metric with high-cardinality labels.** Cost + performance disaster.
- **Never declare an alert "tested" without actually firing it.**
- **Never add noise.** Every alert must be actionable, or it trains the team to ignore.

---

## Completion

- **HEALTHY** — SLOs defined, golden signals dashboarded, alerts tested, runbooks linked
- **NEEDS WORK** — gap list produced, fix plan drafted
- **CRITICAL** — service shipped without observability; block further deploys until remediated


---

## CRITICAL LESSON LEARNED — Build Verification

An alert that's never fired in anger is an unverified alert. Trigger it synthetically before trusting it.

## CRITICAL LESSON LEARNED — Acceptance Criteria Verification

The AC for observability is "when X breaks, we know within Y seconds." Prove that by breaking X in staging and measuring Y.

## CRITICAL LESSON LEARNED — Branch Scope Discipline

Dashboards + alerts are tied to specific deployed services. Don't set up dashboards for code that lives on a feature branch — instrument shipped code only.
