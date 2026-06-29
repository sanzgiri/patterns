# Log Aggregation & Application Metrics

**Aliases:** Centralized Logging, Log Pipeline, Log Shipping, Application Metrics, Metric Aggregation, Observability Stack
**Category:** Operations / Observability
**Sources:**
[Microsoft Azure — Application Insights / monitoring guidance](https://learn.microsoft.com/en-us/azure/architecture/microservices/logging-monitoring) ·
[microservices.io — Log Aggregation, Application Metrics](https://microservices.io/patterns/observability/application-logging.html) ·
[Google SRE Book — *Monitoring Distributed Systems*](https://sre.google/sre-book/monitoring-distributed-systems/) — Four Golden Signals ·
[Tom Wilkie — *RED Method*](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/) ·
[Brendan Gregg — *USE Method*](https://www.brendangregg.com/usemethod.html) ·
[Prometheus documentation](https://prometheus.io/docs/) ·
[Grafana Loki documentation](https://grafana.com/oss/loki/) ·
[OpenTelemetry documentation](https://opentelemetry.io/docs/)

---

## Problem

> [!TIP]
> **ELI5.** Imagine 50 services running on 200 containers, each writing logs to its own files and emitting its own counters. To diagnose anything, you'd have to SSH into 200 boxes. You can't see error trends across services. You can't alert on "errors spiked." You have no way to know your p99 latency. **Log aggregation** collects every log line from every container to a central searchable store. **Application metrics** does the same for numeric measurements (request rate, error rate, latency percentiles). Together with [distributed tracing](distributed-tracing.md) they form the "three pillars of observability" that make modern distributed systems debuggable. Without them, microservices are blind.

The motivating context is identical to that for [distributed tracing](distributed-tracing.md): in a microservices system, system state is scattered across hundreds of processes on hundreds of hosts. Without aggregation, you cannot:

- See what's happening *now* (live errors, hot endpoints, traffic patterns).
- Investigate what happened *yesterday* (logs from a recently-terminated pod).
- Alert proactively ("error rate >1%", "p99 latency >2s").
- Audit security events (failed logins, sensitive data access).
- Show dashboards that summarize service health.
- Comply with regulations that require log retention.

**Logs** are discrete events with messages and context (`2024-01-15 10:23:45 ERROR [req-abc] payment: stripe timeout`). They're high-cardinality, voluminous, expensive to store at full fidelity, but invaluable for forensic detail.

**Metrics** are numeric time-series (`requests_total{service="orders",status="500"} = 47`). They're aggregated, fixed-cardinality, cheap to store, perfect for alerting and dashboards — but they hide individual events.

Together they're complementary. Logs answer "what happened?" with detail; metrics answer "is the system healthy?" with cheap continuous monitoring. Modern systems need both. Tracing (covered separately) is the third pillar — bringing per-request structure to the picture.

The architecture of each — a pipeline from app to central store — looks similar but the details differ a lot. Log pipelines deal with volume (GB-TB per day) and unstructured data; metric pipelines deal with cardinality and time-series math.

## How it works (Log Aggregation)

> [!TIP]
> **ELI5.** Every application writes logs (often to stdout). A small agent on each host reads them, adds metadata (which host, which pod, which service), and ships them to a central log storage service. The storage service indexes them so you can search across all services at once. You can search by correlation ID to find one request's full activity, by error keyword to find all instances of a bug, or aggregate to count errors per minute.

### The pipeline

A typical log-aggregation stack:

![Log aggregation pipeline](../diagrams/svg/log-aggregation.svg)

**Sources** — applications write logs. In modern systems, this is **stdout/stderr** (12-factor app principle "Logs as event streams") rather than files. Containers capture stdout; Kubernetes does so too. Older systems write files (`/var/log/app.log`) which agents tail.

**Collection agents** — one per host (DaemonSet in K8s). They:
- Read from stdout/files/journald/Docker logs.
- Parse structured logs (JSON, key=value, regex for unstructured).
- Enrich with metadata (hostname, pod name, namespace, environment).
- Buffer locally if downstream is unreachable.
- Route to one or many destinations.

Common agents: **Fluent Bit** (lightweight, C, very fast), **Fluentd** (richer, Ruby), **Vector** (Rust, modern, very efficient), **Filebeat / Logstash** (Elastic stack), **Promtail** (Loki's agent), **OpenTelemetry Collector** (unified for logs+metrics+traces).

**Transport (optional)** — a message bus (Kafka, Kinesis, NATS) between collectors and storage. Decouples producers from sinks, absorbs bursts, enables fanout (same logs to long-term archive + real-time analytics + alerting).

**Storage + search** — the central destination. Major options:

| Tool | Notes |
|---|---|
| **Elasticsearch / OpenSearch** | Full-text inverted index; most flexible queries; expensive at scale. |
| **Grafana Loki** | Index labels only; logs stored cheap (S3-backed); query with LogQL. |
| **ClickHouse / VictoriaLogs** | Columnar; very cheap; great for high-volume aggregation. |
| **Splunk** | Commercial; powerful but expensive. |
| **Datadog Logs / Sumo Logic / New Relic Logs** | Commercial SaaS. |
| **AWS CloudWatch Logs / Azure Log Analytics / GCP Cloud Logging** | Cloud-native. |
| **S3 / GCS + Athena / Presto** | Cheap long-term archive; query on demand. |

The big architectural decision: **how much do you index?** Elasticsearch indexes every field (expensive but flexible queries); Loki indexes only labels (cheap but slower full-text). Most large orgs run multiple tiers — hot logs in Elasticsearch for 1 week, archived to S3 for 1 year.

**Query, alert, dashboard** — Kibana / Grafana / Splunk / vendor UI. Engineers query logs to investigate; alerts fire on error patterns (logs-as-metrics); dashboards summarize trends.

### Structured logging — non-negotiable

The single most-impactful improvement: **log JSON, not free-form strings**.

```python
# Bad
logger.info(f"User {user_id} placed order {order_id} for {amount}")

# Good
logger.info("order_placed", user_id=user_id, order_id=order_id, amount=amount)
# emits: {"timestamp": "...", "level": "INFO", "msg": "order_placed",
#         "user_id": "alice", "order_id": "ord-7", "amount": 49.99, ...}
```

Why:
- **Parseable**: agent extracts fields automatically.
- **Searchable**: query by field (`user_id:alice`) not regex.
- **Aggregatable**: count by user, sum amounts.
- **Stable**: code changes don't break message parsing.

Libraries: **structlog** (Python), **logback** + Logstash encoder (Java), **zerolog/zap** (Go), **pino/winston** (Node), **tracing** (Rust).

### Correlation, tenancy, and PII

Every log line should carry:
- `correlation_id` / `trace_id` — see [Distributed Tracing](distributed-tracing.md).
- `user_id` and/or `tenant_id` (carefully — see PII below).
- `service`, `version`, `env`, `region`.
- `severity`, `timestamp` (always UTC, RFC3339 or epoch).

**PII** is a constant concern: logs are written everywhere, kept for a long time, and accessed by many engineers. Best practices:
- **Never log full credit cards, SSNs, passwords, API keys** — period.
- **Hash or pseudonymize** PII where possible.
- **Tag fields as sensitive** — agent redacts on the way in.
- **Tiered access** — production logs restricted; dev logs broader.
- **Retention limits** — auto-delete after N days/years.
- **Audit log access** — who queried what.

GDPR explicitly classifies log access as data processing; treat it accordingly.

### Cost is the real challenge

At any reasonable scale, log volume is enormous:
- 1000 QPS × 5 log lines/req × 500 bytes/line = 2.5 MB/s = 215 GB/day per service.
- Across hundreds of services, easily 10-100 TB/day.

Cost-control techniques:
- **Sampling**: log 1% of successful requests, 100% of errors.
- **Log levels**: WARN and above to expensive store; DEBUG only to cheap tier.
- **Tiered storage**: hot (1 week, Elasticsearch) → warm (3 months, OpenSearch frozen tier) → cold (1+ year, S3/Glacier).
- **Index sparingly**: only fields you'll actually filter on.
- **Aggressive deduplication and compression**.
- **Quota enforcement** per service — prevent runaway logging.
- **Logs as metrics**: aggregate error counts to a metric instead of storing each line.

Without budgets, log spend becomes a top-3 infrastructure cost line item. Mature orgs have a "logging contract" per service: max bytes per request, max retention, max cardinality.

## How it works (Application Metrics)

> [!TIP]
> **ELI5.** Instead of logging every event, your app keeps **counters** ("400 requests since startup"), **gauges** ("currently 12 connections to DB"), and **histograms** ("distribution of request latencies"). These are aggregated continuously and shipped (or scraped) to a time-series database. Because they're aggregated, a million requests becomes one number. That makes them cheap, fast, and perfect for alerting and dashboards — at the cost of hiding individual events (you need logs/traces for that).

### Metric types

Four standard types:

![Application metrics types and pyramid](../diagrams/svg/application-metrics.svg)

- **Counter** — monotonically increasing. `requests_total`, `errors_total`. Use `rate()` in queries to get per-second rates.
- **Gauge** — current value, up or down. `memory_in_use`, `queue_depth`, `connections_open`.
- **Histogram** — buckets of observations plus count and sum. `request_duration_seconds_bucket{le="0.005"}=10`. Lets you compute quantiles (`histogram_quantile(0.99, ...)`).
- **Summary** — client-side quantile computation. Less flexible than histograms (can't aggregate across instances). Mostly historical now.

The choice between types is significant. Most "what's happening?" needs are counters and histograms; gauges are for instantaneous state.

### The frameworks: golden signals, RED, USE

Three influential frameworks for *what* to monitor:

**Google SRE — Four Golden Signals** (from the SRE book):
- **Latency** — how long requests take (especially p99).
- **Traffic** — request rate, throughput.
- **Errors** — error rate.
- **Saturation** — how full / busy the system is (queue depth, CPU%, memory).

For any user-facing service, these four are the must-monitor baseline.

**RED Method** (Tom Wilkie, ex-Weaveworks, now Grafana Labs):
- **Rate** — requests per second.
- **Errors** — failing requests per second.
- **Duration** — distribution of request latencies.

Simplification of golden signals for *services*; per-endpoint or per-service.

**USE Method** (Brendan Gregg, Netflix performance engineer):
- **Utilization** — what % busy is the resource?
- **Saturation** — how much work queued / waiting?
- **Errors** — error count.

For *resources* (CPU, memory, disk, network) rather than services.

Both RED and USE are subsets of the golden signals viewed from different angles. Use both: RED per service, USE per resource, business metrics per product.

### Push vs pull

Two delivery models:

**Pull (Prometheus model)**: TSDB scrapes `/metrics` endpoints periodically. Endpoint exposes current values; TSDB stores time-series. Advantages: service discovery integrated; easy to detect a down service (it stops being scrapeable); pull rate decoupled from app.

**Push (StatsD, Graphite, OTel push model)**: app pushes metrics to a collector. Advantages: works for ephemeral jobs that don't live long enough to be scraped; works through NAT/firewall; lower latency.

Most modern stacks use pull (Prometheus dominant) for long-lived services and push (StatsD, OTel push) for batch jobs and Lambda functions. OpenTelemetry supports both via the OTel Collector.

### Cardinality — the silent killer

A metric has *labels*: `http_requests_total{service="orders",method="GET",status="200",endpoint="/users/:id"}`. Each unique combination is a separate time-series stored in the TSDB.

Cardinality explodes if labels include high-variance values:
- `user_id` label → millions of series.
- `request_id` label → unbounded series.
- `tenant_id` label → thousands of series (manageable if you cap).

This is the #1 way to blow up a TSDB. Rules:
- **Never label with unbounded values** (request_id, full URL with params).
- **Label with bounded enumerations** (status code, method, endpoint *template*).
- **Aggregate before tagging** if you need per-user breakdown — use logs/traces for per-user.
- **Monitor cardinality**: Prometheus has a `prometheus_tsdb_head_series` metric — watch it.

Honeycomb's "observability 2.0" critique is essentially: pre-aggregated metrics force you to predict your queries (which labels matter) and lose data fidelity. Their answer: store wide events (like traces) and aggregate at query time. The trade-off is cost.

### Alerting on metrics

Metrics enable alerting because they're continuously evaluated:

```yaml
# Prometheus alert rule
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
  for: 5m
  annotations:
    summary: "Error rate >1% for 5 minutes"
```

Best practices (mostly from SRE book):
- **Alert on symptoms** (user-facing latency / errors), not causes (CPU high).
- **Alert on SLOs**: define an SLO (99.9% requests <1s), alert when burning error budget.
- **Tune for actionability**: every page should require a human action.
- **Avoid alert fatigue**: silence rules during deploys, group related alerts.
- **Runbooks**: every alert links to a runbook.

## Health Endpoint Monitoring

> [!TIP]
> **ELI5.** Every service exposes a tiny HTTP endpoint that says "I'm alive and ready for traffic" (or 5xx if not). Load balancers, Kubernetes, and monitors poll this endpoint constantly. There are 2-3 distinct check kinds with different semantics: **liveness** ("am I deadlocked?" → restart me), **readiness** ("can I take traffic right now?" → route around me), **startup** ("am I warmed up yet?" → wait longer). Conflating these is a top source of cascading failures.

The pattern looks trivial — `GET /healthz` returns 200 — but the semantics are subtle:

![Health endpoint kinds](../diagrams/svg/health-endpoint.svg)

**Liveness** (`/healthz/live`) — *Am I broken so badly the process should be restarted?* Should almost always return OK. Only fails on genuine deadlock, corruption, unrecoverable state. Failing → orchestrator (Kubernetes) **kills and restarts** the pod. **Must not check dependencies** — if DB blip flips liveness, you get restart storms.

**Readiness** (`/healthz/ready`) — *Am I ready to accept traffic?* Checks critical dependencies (DB connection alive, config loaded, warmup done). Failing → orchestrator removes from load-balancer rotation; **does not restart**; existing in-flight requests drain. Can flap during normal operation (dep blips, startup, shutdown).

**Startup** (`/healthz/startup`) — *Have I finished warming up?* For slow-start services (JVM, large ML model load). Once OK, liveness/readiness take over. Until OK, the pod gets extended time to start without being killed.

**Deep health** (`/healthz/deep` or `/status`) — detailed dependency report for humans / external monitors. Returns a JSON tree of each dependency's state. **Never use as a probe** — too easy for one flaky downstream to cascade pod restarts.

### Kubernetes probe configuration

Kubernetes pioneered the three-probe model:

```yaml
livenessProbe:
  httpGet: { path: /healthz/live, port: 8080 }
  initialDelaySeconds: 0     # rely on startupProbe instead
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet: { path: /healthz/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2

startupProbe:
  httpGet: { path: /healthz/startup, port: 8080 }
  failureThreshold: 30        # 30 * 10s = 5 min to start
  periodSeconds: 10
```

The classic mistake: setting `livenessProbe` with `initialDelaySeconds: 60`, getting restart-killed because DB is slow at startup, then making the probe check every dep and have flapping. Use **startup** probe for warmup, **readiness** for "I'm having dep issues but don't restart me," **liveness** only for genuine corruption.

### Health checks ↔ load shedding

Readiness is also the natural **load-shedding** signal: if a service is overloaded, returning unhealthy makes the LB route around it temporarily. Combined with [bulkhead](../res/bulkhead.md) and [backpressure](../res/backpressure.md), this gives a coherent overload-response strategy.

### Anti-patterns (health endpoints)

- **One endpoint conflating liveness + readiness + deep** → restart storms on dep blips.
- **Deep-checking dependencies in liveness** → cascading failures.
- **No timeout on the health check itself** → probes themselves cause load.
- **Returning OK without actually checking anything** → false confidence.
- **Returning unhealthy because of a non-critical dep** → reduces capacity unnecessarily.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Centralized logging (ELK, Loki)** | Standard log-aggregation stack. |
| **Logs-as-metrics** | Aggregate log counts into TSDB. |
| **Push metrics (StatsD)** | Older, push-based. |
| **Pull metrics (Prometheus)** | Modern dominant. |
| **OpenTelemetry** | Unified logs + metrics + traces. |
| **Golden Signals / RED / USE** | Frameworks for *what* to monitor. |
| **Liveness / Readiness / Startup probes** | K8s probe types. |
| **Deep health endpoint** | For human/external monitors only. |
| **[Distributed Tracing](distributed-tracing.md)** | The third pillar. |
| **[Audit Logging](audit-log.md)** | Specialized logging for security/compliance. |
| **[Chaos Engineering](chaos-engineering.md)** | Stresses the observability stack. |

## When NOT to use

- **A single-process app with no operational concerns** — overhead exceeds benefit (rare in practice).
- **Without correlation IDs / structured logs** — aggregation has limited value.
- **Without retention/cost discipline** — logs become a runaway expense.
- **For PII-sensitive data** without redaction — compliance liability.
- **Liveness probes that check dependencies** — guaranteed cascading failure mode.

---

## Real-world implementations

| Tool | Category |
|---|---|
| **Prometheus** | Metrics; CNCF; dominant pull-based TSDB |
| **Grafana** | Visualization for Prometheus, Loki, others |
| **VictoriaMetrics** | Prometheus-compatible; very fast |
| **M3** | Uber's; large-scale metrics |
| **InfluxDB** | TSDB; push or pull |
| **Datadog, New Relic, Dynatrace, Splunk** | Commercial APM/observability |
| **Elasticsearch + Logstash + Kibana (ELK)** | Logs |
| **OpenSearch** | AWS fork of Elasticsearch |
| **Grafana Loki** | Cheap log store; label-indexed |
| **Fluentd / Fluent Bit / Vector / Filebeat** | Log collection agents |
| **OpenTelemetry Collector** | Unified collector |
| **AWS CloudWatch, Azure Monitor, GCP Cloud Logging** | Cloud-native |
| **ClickHouse, VictoriaLogs** | High-volume log column stores |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | Borgmon (Prometheus's inspiration); SRE practices defined the field. | ✅ Verified — SRE book |
| **SoundCloud** | Created Prometheus (open-sourced 2015). | ✅ Verified — Prometheus history |
| **Uber** | M3 metrics platform (open-sourced). | ✅ Verified — Uber Engineering blog |
| **LinkedIn** | Kafka + extensive log aggregation pipeline. | ✅ Verified — LinkedIn Engineering blog |
| **Netflix** | Atlas (metrics, open-sourced), Mantis (stream processing for telemetry). | ✅ Verified — Netflix Tech Blog |
| **Facebook / Meta** | Scuba (logs/events at scale). | ✅ Verified — Scuba paper |
| **Twitter / X** | Heron + extensive metrics pipelines. | ✅ Verified — Twitter Engineering blog |
| **Cloudflare** | ClickHouse for log analytics at huge scale. | ✅ Verified — Cloudflare blog |
| **Stripe** | Heavy use of metrics, structured logs, custom dashboards. | ✅ Verified — Stripe Engineering blog |
| **Most Kubernetes shops** | Prometheus + Grafana + Loki / Elasticsearch are de facto. | ✅ Industry standard |

---

## Further reading

- *Site Reliability Engineering* (Google SRE book) — Monitoring chapter (Four Golden Signals).
- *The Site Reliability Workbook* — practical SRE.
- Tom Wilkie — *RED Method* blog post.
- Brendan Gregg — *USE Method* page.
- *Observability Engineering* (Majors, Fong-Jones, Miranda) — observability 2.0.
- *Distributed Systems Observability* (Sridharan, O'Reilly) — concise overview.
- *Prometheus: Up & Running* (Brazil, O'Reilly).
- *Logging in Action* (Vlasenko, Manning).
- OpenTelemetry, Prometheus, Loki documentation.
- Charity Majors's blog — observability 2.0 perspective.

---

*Diagram sources: [`../diagrams/src/log-aggregation.d2`](../diagrams/src/log-aggregation.d2), [`../diagrams/src/application-metrics.d2`](../diagrams/src/application-metrics.d2), [`../diagrams/src/health-endpoint.d2`](../diagrams/src/health-endpoint.d2).*
