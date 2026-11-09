# SRE & Reliability — Cheatsheet

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Dense reference for on-call, SLO design and interviews: nines and downtime, error-budget and burn-rate math, SLI PromQL, alert windows, incident roles and severity, postmortem outline, DORA, reliability patterns, k6, Chaos Mesh, LitmusChaos and AWS FIS. Prometheus 2.5x / 3.x, Alertmanager 0.27+, k6 1.x, Chaos Mesh 2.7.

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Contents

[Vocabulary](#vocabulary) · [Nines and downtime](#availability-nines-and-downtime) · [Composite availability](#composite-availability) · [Error budget math](#error-budget-math) · [Burn rate math](#burn-rate-math) · [Alerting windows](#multi-window-multi-burn-rate-alerting-windows) · [SLI formulas in PromQL](#sli-formulas-in-promql) · [Recording and alert rules](#recording-and-alert-rule-templates) · [promtool and amtool](#promtool-and-amtool) · [Incident roles](#incident-roles) · [Severity matrix](#severity-matrix) · [Incident checklist](#incident-checklist-and-status-updates) · [Postmortem outline](#postmortem-template-outline) · [Incident and DORA metrics](#incident-metrics-and-dora-metrics) · [Reliability patterns](#reliability-patterns) · [Toil and on-call](#toil-and-on-call-rules-of-thumb) · [DR](#backup-and-disaster-recovery) · [k6](#k6) · [Chaos Mesh](#chaos-mesh) · [LitmusChaos and AWS FIS](#litmuschaos-and-aws-fis) · [Top 15](#top-15-interview-one-liners)

---

## Vocabulary

| Term | One-line definition |
|------|---------------------|
| **SLI** | A measurement of user experience, ideally `good events / valid events` |
| **SLO** | Target for an SLI over a window: "99.9% of checkout requests succeed over 28 days" |
| **SLA** | Contract with customers with penalties; always looser than the SLO |
| **Error budget** | `1 - SLO`: the failure you are allowed to spend |
| **Error-budget policy** | Pre-agreed consequences when the budget runs low or out (release freeze, reliability work) |
| **Burn rate** | Speed of budget consumption: `error ratio / (1 - SLO)`; 1 = exactly on budget |
| **Toil** | Manual, repetitive, automatable, tactical work without lasting value that grows with scale |
| **MTTD / MTTA / MTTR** | Time to detect / acknowledge / restore (or recover) |
| **RTO / RPO** | Maximum tolerable downtime / maximum tolerable data loss (measured in time) |
| **Golden signals** | Latency, traffic, errors, saturation |
| **RED / USE** | Rate, Errors, Duration (services) / Utilization, Saturation, Errors (resources) |

## Availability: nines and downtime

| SLO | Per day | Per week | Per 28 days | Per 30 days | Per year (365 d) |
|-----|---------|----------|-------------|-------------|------------------|
| 90% | 2.4 h | 16.8 h | 2.8 d | 3 d | 36.5 d |
| 95% | 72 min | 8.4 h | 33.6 h | 36 h | 18.25 d |
| 99% | 14.4 min | 1.68 h | 6.72 h | 7.2 h | 3.65 d |
| 99.5% | 7.2 min | 50.4 min | 3.36 h | 3.6 h | 1.83 d |
| 99.9% | 1.44 min | 10.08 min | 40.32 min | 43.2 min | 8.76 h |
| 99.95% | 43.2 s | 5.04 min | 20.16 min | 21.6 min | 4.38 h |
| 99.99% | 8.64 s | 60.5 s | 4.03 min | 4.32 min | 52.56 min |
| 99.999% | 0.86 s | 6.05 s | 24.2 s | 25.9 s | 5.26 min |

Formula: `allowed downtime = (1 - SLO) × window`. 30 days = 43,200 min; 28 days = 40,320 min; 365 days = 525,600 min.

Each extra nine costs roughly 10x less downtime and usually much more than 10x the money and engineering. Humans cannot respond inside a 99.99% monthly budget (4.3 min), so four nines needs automated detection and mitigation.

## Composite availability

| Topology | Formula | Example |
|----------|---------|---------|
| Serial (all required) | `A = a1 × a2 × ... × an` | 3 × 99.9% → 99.70% |
| Parallel (any one is enough, independent, automatic failover) | `A = 1 - (1 - a)^n` | 2 × 99% → 99.99%; 3 × 99% → 99.9999% |
| Serial chain with a redundant tier | compute the parallel tier first, then multiply | LB 99.99% × (2 × 99.5% → 99.9975%) × DB 99.95% ≈ 99.94% |
| Soft dependency with working fallback | not in the product | cache with DB fallback, recommendations |

Rules: your SLO cannot exceed the product of your hard dependencies; correlated failures (same AZ, same bad deploy, same config) break the parallel formula; failover time counts as downtime.

## Error budget math

| Quantity | Formula | Example (99.9%, 30 d, 200 req/s) |
|----------|---------|----------------------------------|
| Budget fraction | `1 - SLO` | 0.001 |
| Total events in window | `rate × window seconds` | 200 × 2,592,000 = 518.4 M |
| Budget in events | `total × (1 - SLO)` | 518,400 failed requests |
| Budget in time | `(1 - SLO) × window` | 43.2 min |
| Budget consumed by incident | `bad events / budget events` | 60,000 / 518,400 = 11.6% |
| Equivalent full downtime of a partial outage | `duration × error ratio` | 40 min × 10% = 4 min = 9.3% |
| Budget remaining | `1 - (error ratio over window / (1 - SLO))` | ratio 0.0004 → 60% left |
| Time-based SLI | `good minutes / total minutes` | a minute is good if its error ratio < 1% |

Request-based SLIs weight busy hours more than quiet ones (users feel that); time-based SLIs treat 3 a.m. and peak the same and are simpler to explain in SLAs.

## Burn rate math

```text
burn rate             = error ratio / (1 - SLO)
time to exhaust       = SLO window / burn rate
budget spent in t     = burn rate × t / SLO window
alert threshold       = (budget fraction to spend) × SLO window / alert window
detection time (full outage, alert window W, threshold B, SLO S)
                      = W × B × (1 - S)         (100% errors)
```

| Burn rate | 30-day budget lasts | Error ratio at 99.9% | Error ratio at 99% |
|-----------|---------------------|----------------------|--------------------|
| 1 | 30 days | 0.1% | 1% |
| 2 | 15 days | 0.2% | 2% |
| 3 | 10 days | 0.3% | 3% |
| 6 | 5 days | 0.6% | 6% |
| 10 | 3 days | 1% | 10% |
| 14.4 | 50 hours | 1.44% | 14.4% |
| 36 | 20 hours | 3.6% | 36% |
| 100 | 7.2 hours | 10% | not reachable (max 100) |
| 1000 | 43.2 min | 100% | not reachable |

Where 14.4 comes from: 2% of budget in 1 hour → `0.02 × 720 h / 1 h = 14.4`. Detection time for a full outage at 99.9%: `60 min × 14.4 × 0.001 ≈ 52 s`.

## Multi-window, multi-burn-rate alerting windows

| Severity | Long window | Short window | Burn rate | Budget spent at trigger | Error ratio at 99.9% | Error ratio at 99.5% | Error ratio at 99% |
|----------|-------------|--------------|-----------|-------------------------|----------------------|----------------------|--------------------|
| Page | 1h | 5m | 14.4 | 2% | 1.44% | 7.2% | 14.4% |
| Page | 6h | 30m | 6 | 5% | 0.6% | 3% | 6% |
| Ticket | 1d | 2h | 3 | 10% | 0.3% | 1.5% | 3% |
| Ticket | 3d | 6h | 1 | 10% | 0.1% | 0.5% | 1% |

- Long window: is it significant? Short window (≈ 1/12 of long): is it still happening? Both must be above the threshold.
- Short window makes the alert reset within minutes of the fix.
- Low-traffic services: add a minimum-request guard, synthetic probes, or longer windows; one failure in ten requests is a 10% ratio.
- For 28-day windows the same multipliers are commonly reused; recompute if you want exact budget percentages.

## SLI formulas in PromQL

**Availability (request-based, error ratio)**

```promql
(sum(rate(http_requests_total{job="shop-api",code=~"5.."}[5m])) or vector(0))
/
sum(rate(http_requests_total{job="shop-api"}[5m]))
```

**Availability, excluding health checks and counting overload 429 as bad**

```promql
sum(rate(http_requests_total{job="api",route!~"/healthz|/metrics",code=~"5..|429"}[5m]))
/
sum(rate(http_requests_total{job="api",route!~"/healthz|/metrics"}[5m]))
```

**Latency SLI (classic histogram, threshold must be a bucket boundary)**

```promql
sum(rate(http_request_duration_seconds_bucket{job="shop-api",le="0.25"}[5m]))
/
sum(rate(http_request_duration_seconds_count{job="shop-api"}[5m]))
```

**Latency SLI (native histogram)**

```promql
histogram_fraction(0, 0.25, sum(rate(http_request_duration_seconds{job="shop-api"}[5m])))
```

**Latency percentile (for dashboards, not for budgets)**

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="shop-api"}[5m])))
```

**Edge SLIs**

```promql
# ingress-nginx
sum(rate(nginx_ingress_controller_requests{ingress="shop",status=~"5.."}[5m]))
/ sum(rate(nginx_ingress_controller_requests{ingress="shop"}[5m]))

# Istio (server side)
sum(rate(istio_requests_total{reporter="destination",destination_service_name="shop-api",response_code=~"5.."}[5m]))
/ sum(rate(istio_requests_total{reporter="destination",destination_service_name="shop-api"}[5m]))

# AWS ALB via CloudWatch exporter / YACE (names depend on exporter config)
# HTTPCode_Target_5XX_Count + HTTPCode_ELB_5XX_Count  /  RequestCount
```

**gRPC availability**

```promql
sum(rate(grpc_server_handled_total{grpc_service="shop.Checkout",grpc_code=~"Unknown|DeadlineExceeded|Unavailable|Internal|DataLoss|ResourceExhausted"}[5m]))
/ sum(rate(grpc_server_handled_total{grpc_service="shop.Checkout"}[5m]))
```

**Synthetic probe (blackbox exporter)**

```promql
avg_over_time(probe_success{job="blackbox",instance="https://shop.example.com/healthz"}[5m])
```

**Freshness (pipelines): seconds since last successful run, and SLI as good minutes**

```promql
time() - max(pipeline_last_success_timestamp_seconds{pipeline="invoices"})
avg_over_time(((time() - max(pipeline_last_success_timestamp_seconds{pipeline="invoices"})) < bool 600)[1d:1m])
```

**Time-based SLI: fraction of good 5-minute slices over 30 days**

```promql
avg_over_time((slo:sli_error:ratio_rate5m{slo="shop-api-availability"} < bool 0.01)[30d:5m])
```

**Budget, burn rate, forecast**

```promql
# Error budget remaining (30d)
1 - (slo:sli_error:ratio_rate30d{slo="shop-api-availability"} / (1 - 0.999))
# Burn rate over 1h
slo:sli_error:ratio_rate1h{slo="shop-api-availability"} / (1 - 0.999)
# Bad requests in the last 30 days
sum(increase(http_requests_total{job="shop-api",code=~"5.."}[30d]))
# Days until the budget is gone at the current 6h burn rate (with 1 = a full 30-day budget)
(slo:error_budget_remaining:ratio30d * 30) / (slo:sli_error:ratio_rate6h / (1 - 0.999))
```

## Recording and alert rule templates

```yaml
groups:
  - name: slo-shop-api-availability
    rules:
      - record: slo:sli_error:ratio_rate5m      # repeat for 30m 1h 2h 6h 1d 3d 30d
        expr: |
          (sum(rate(http_requests_total{job="shop-api",code=~"5.."}[5m])) or vector(0))
          / sum(rate(http_requests_total{job="shop-api"}[5m]))
        labels:
          slo: shop-api-availability
      - record: slo:objective:ratio
        expr: vector(0.999)
        labels:
          slo: shop-api-availability

  - name: slo-shop-api-alerts
    rules:
      - alert: ShopApiAvailabilityBurnFast
        expr: |
          slo:sli_error:ratio_rate1h{slo="shop-api-availability"} > (14.4 * 0.001)
          and
          slo:sli_error:ratio_rate5m{slo="shop-api-availability"} > (14.4 * 0.001)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: shop-api burning availability budget at 14x or more
          runbook_url: https://runbooks.example.com/shop-api-slo
      - alert: ShopApiAvailabilityBurnSlow
        expr: |
          slo:sli_error:ratio_rate6h{slo="shop-api-availability"} > (6 * 0.001)
          and
          slo:sli_error:ratio_rate30m{slo="shop-api-availability"} > (6 * 0.001)
        labels:
          severity: page
      # tickets: 1d and 2h > 3x, 3d and 6h > 1x, severity: ticket
```

SLO tooling that generates these: **Sloth** (`sloth generate -i slos.yaml -o rules.yaml`), **Pyrra** (`ServiceLevelObjective` CRD + UI), **OpenSLO** (vendor-neutral spec), Google Cloud / Datadog / Grafana Cloud / New Relic native SLO features.

## promtool and amtool

| Task | Command |
|------|---------|
| Validate rule file | `promtool check rules rules.yaml` |
| Validate Prometheus config | `promtool check config prometheus.yml` |
| Unit test rules | `promtool test rules tests.yaml` |
| Extract rules from a PrometheusRule CRD | `yq '.spec' prometheusrule.yaml > rules.yaml` |
| Run promtool with Docker | `docker run --rm -v "$PWD:/w" -w /w --entrypoint promtool prom/prometheus:v3.5.0 test rules tests.yaml` |
| Instant query from CLI | `promtool query instant http://localhost:9090 'slo:sli_error:ratio_rate5m'` |
| Range query from CLI | `promtool query range --start=$(date -d -1h +%s) --end=$(date +%s) --step=1m http://localhost:9090 'up'` |
| Check TSDB cardinality | `promtool tsdb analyze /prometheus` |
| Validate Alertmanager config | `amtool check-config alertmanager.yml` |
| Which receiver gets these labels? | `amtool config routes test --config.file=alertmanager.yml severity=page slo=checkout` |
| Show routing tree | `amtool config routes show --config.file=alertmanager.yml` |
| List firing alerts | `amtool alert query --alertmanager.url=http://localhost:9093 severity=page` |
| Silence for maintenance | `amtool silence add service=checkout --duration=2h --comment="CHG-123 DB upgrade" --alertmanager.url=http://localhost:9093` |
| List / expire silences | `amtool silence query` · `amtool silence expire <id>` |
| Prometheus API: firing alerts | `curl -s localhost:9090/api/v1/alerts \| jq '.data.alerts[] \| {n:.labels.alertname,s:.state}'` |
| Prometheus API: rule health | `curl -s localhost:9090/api/v1/rules \| jq '.data.groups[].rules[] \| {name, health, lastError}'` |
| Hot reload (with `--web.enable-lifecycle`) | `curl -X POST localhost:9090/-/reload` |

Unit test skeleton:

```yaml
rule_files: [rules.yaml]
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{job="shop-api",code="200"}'
        values: '0+540x90'
      - series: 'http_requests_total{job="shop-api",code="503"}'
        values: '0+60x90'
    alert_rule_test:
      - eval_time: 70m
        alertname: ShopApiAvailabilityBurnFast
        exp_alerts:
          - exp_labels: {severity: page, slo: shop-api-availability}
            exp_annotations: {summary: "...", runbook_url: "..."}
```

## Incident roles

| Role | Responsibilities | Anti-pattern |
|------|------------------|--------------|
| **Incident Commander (IC)** | Declares severity, owns decisions, delegates, runs check-ins, keeps the big picture, declares resolved, hands off explicitly | IC debugging on a terminal |
| **Ops / Technical lead** | Diagnosis and mitigation, coordinates responders, reports to IC on a fixed cadence | Making large changes without telling the IC |
| **Comms lead** | Internal and external updates on a schedule, status page, stakeholder questions | Guessing root cause in public |
| **Scribe** | Timestamped log of facts, decisions, commands | Recording only "important" things |
| **Subject-matter experts** | Pulled in by IC or Ops lead for specific systems | Joining uninvited and adding noise |
| **Customer liaison / exec sponsor** (SEV-1) | Customer-facing decisions, credits, legal and PR | Directing the technical response |

## Severity matrix

| Severity | User impact | Examples | Response | Updates | Postmortem |
|----------|-------------|----------|----------|---------|------------|
| **SEV-1** | Critical journey down for most users, data loss or corruption, security breach | Checkout down, login down, data leak | Page IC and team immediately, all hands, exec informed | Every 30 min, status page | Mandatory |
| **SEV-2** | Critical journey degraded for many users, or fast budget burn | 20% of checkouts fail, p99 10x normal | Page on-call, IC assigned | Every 30-60 min | Mandatory |
| **SEV-3** | Non-critical feature degraded, workaround exists, slow burn | Search slow, emails delayed 30 min | Working hours, ticket | Daily or at milestones | If budget impact > threshold |
| **SEV-4** | No current user impact | Redundancy lost, one replica crashing | Ticket | None | No |

Severity is set by impact, not difficulty or cause. Declare early and high; downgrading is free.

## Incident checklist and status updates

```text
DETECT     ack the page within 5 min, look at the SLO dashboard, confirm user impact
DECLARE    severity, incident channel (#inc-YYYYMMDD-short-name), IC named, roles assigned
STABILIZE  mitigate first: rollback, flag off, failover, scale out, shed load, block abusive traffic
           change one thing at a time, announce each change in the channel
COMMUNICATE first update within 10-15 min, then on schedule even with no news
RESOLVE    SLIs back within SLO and stable for an agreed period, state known
FOLLOW UP  postmortem owner and due date, preserve logs and dashboards, thank people
```

```text
[Investigating] <SEV> <service>: Since <time UTC>, <who> are seeing <symptom> (<scope, e.g. ~25% of checkouts>).
We are investigating. Next update at <time UTC>.

[Identified] We have identified <high-level cause> and are applying <mitigation>. Next update at <time>.

[Mitigated] Since <time>, <service> is working again. <Any residual effect>. We are monitoring. Next update at <time>.

[Resolved] The incident is resolved as of <time>. Impact: <duration and scope>. A postmortem will follow.
```

IC handover script: current severity and impact, what has been tried, current hypothesis, actions in flight and owners, next comms time, "you have command".

## Postmortem template outline

| Section | What goes in it |
|---------|-----------------|
| Header | Title, date, authors, status (draft / review / final), severity, IC, impact window, detection method |
| Summary | 2-3 sentences: what users saw, how long, how it was mitigated |
| Impact | Users, requests, orders, revenue, SLA, support tickets, **error budget consumed** |
| Timeline (UTC) | Trigger, impact start, detection, declaration, mitigation, impact end, resolution |
| Root cause and trigger | The technical chain; trigger vs underlying cause |
| Contributing factors | 5 whys or causal tree; systems and processes, not people |
| What went well | Keep doing these |
| What went wrong | Gaps in detection, response, tooling, docs |
| Where we got lucky | Near misses that will not be lucky next time |
| Action items | Action, type (prevent / detect / mitigate / process), owner, priority, due date, ticket |
| Lessons learned | One paragraph a new engineer should read |
| Supporting data | Dashboards with time range, logs, chat log, alert history |

Blameless rules: assume good intent and reasonable decisions given the information at the time; ask "why did the system allow it?"; no names as causes; review within 5 working days; track action items to done.

## Incident metrics and DORA metrics

| Metric | Definition | Notes |
|--------|------------|-------|
| **MTTD** | Impact start → detection | Improved by SLO burn alerts, synthetic probes |
| **MTTA** | Alert → acknowledgement | On-call health, paging config |
| **Time to mitigate** | Impact start → user impact stops | The number users feel |
| **MTTR** | Impact start → restored (definitions vary: repair, recover, resolve) | Always state which definition |
| **MTBF** | Mean time between failures | Less useful for complex systems than SLO attainment |
| **Deployment frequency** | How often you deploy to production | Elite: on demand, many per day |
| **Lead time for changes** | Commit → running in production | Elite: less than one day |
| **Change failure rate** | % of deployments causing a failure needing remediation (rollback, hotfix, incident) | Elite: around 5% |
| **Failed deployment recovery time** | Time to recover from a failed deployment (formerly "time to restore service") | Elite: less than one hour |
| **Rework rate** | Share of deployments that are unplanned fixes for user-facing problems | Tracked in recent DORA research as a fifth stability signal |

DORA findings worth quoting: throughput and stability correlate positively; measure teams and trends, never individuals; use medians, not means.

## Reliability patterns

| Pattern | What it does | Practical settings | Where to implement |
|---------|--------------|--------------------|--------------------|
| **Timeout** | Bounds waiting on a dependency | Every network call; shorter than the caller's timeout; derived from the latency SLO and the dependency's p99 | HTTP client, Envoy/Istio `timeout`, DB driver |
| **Deadline propagation** | Passes remaining time budget downstream | gRPC deadlines, `X-Request-Timeout`-style headers | gRPC, service framework |
| **Retry with backoff + jitter** | Rides out transient failures | 1-3 attempts, only idempotent operations, only retryable errors; `sleep = rand(0, min(cap, base × 2^n))` | One layer only (usually the edge or the client) |
| **Retry budget** | Caps retries as a share of traffic | Retries ≤ 10-20% of requests | Envoy `retry_budget`, client libraries, gRPC retry throttling |
| **Circuit breaker** | Stops calling a failing dependency, fails fast | Open at e.g. 50% failures over 20+ calls, half-open probe after 30 s | Resilience4j, Polly, Envoy outlier detection |
| **Bulkhead** | Isolates resources per dependency or tenant | Separate pools / semaphores / queues | Thread pools, connection pools, separate deployments |
| **Rate limiting** | Protects from too many requests per client | Token bucket per API key or IP; return 429 with `Retry-After` | API gateway, ingress, Envoy local/global rate limit |
| **Load shedding** | Rejects work early when saturated | Shed by priority; reject at queue length or CPU threshold, return 503 fast | Server middleware, Envoy overload manager |
| **Graceful degradation** | Serves a reduced result instead of an error | Cached data, default values, hide non-critical features | Application, feature flags |
| **Idempotency keys** | Makes retries of writes safe | Client sends a unique key; server deduplicates | API design, payment flows |
| **Queue-based load leveling** | Absorbs bursts, decouples availability | Async processing with a backlog SLO | SQS, Kafka, RabbitMQ |
| **Health checks** | Remove bad instances from rotation | Readiness checks own dependencies lightly; liveness never checks dependencies | Kubernetes probes, LB health checks |
| **Progressive delivery** | Limits blast radius of changes | Canary 1-5-25-100% with SLI analysis; feature flags for features | Argo Rollouts, Flagger, LaunchDarkly, OpenFeature |
| **Cell / shard architecture** | Limits blast radius of failures | Independent cells, each serving a subset of customers | Architecture |

Istio snippets:

```yaml
# VirtualService: timeout and retries
http:
  - route:
      - destination: {host: inventory}
    timeout: 300ms
    retries:
      attempts: 2
      perTryTimeout: 120ms
      retryOn: connect-failure,refused-stream,unavailable,cancelled,5xx
---
# DestinationRule: connection pool (bulkhead) and outlier detection (circuit breaking)
trafficPolicy:
  connectionPool:
    tcp: {maxConnections: 100}
    http: {http1MaxPendingRequests: 50, maxRequestsPerConnection: 10}
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 10s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```

Retry amplification: 3 layers each making up to 3 attempts can send up to 27 requests to the bottom service per user request. Retry at one layer, with a budget.

## Toil and on-call rules of thumb

| Rule | Value |
|------|-------|
| Toil cap for SRE (Google guideline) | ≤ 50% of time; the rest is engineering |
| Incidents per on-call shift (Google guideline) | ≤ 2 per 12-hour shift |
| Rotation size for 24/7 single site | 6 to 8 engineers (fewer burns people out) |
| Follow-the-sun | 2 sites × 12 h, each with ≥ 6 engineers |
| Page criteria | Urgent, actionable, user-affecting, needs human judgment |
| Page on | Symptoms (SLO burn, probe failures) |
| Ticket on | Causes and slow burns (disk 80%, cert expiring in 14 days, replica crashlooping) |
| Every page has | A runbook link, an owner, a dashboard |
| Alert review | Weekly: delete or downgrade alerts that were not actionable |

## Backup and disaster recovery

| Strategy | RTO | RPO | Cost | Idea |
|----------|-----|-----|------|------|
| Backup and restore | Hours to a day | Hours (backup interval) | $ | Restore backups into new infrastructure from IaC |
| Pilot light | Tens of minutes to hours | Minutes | $$ | Data replicated, core services minimal or off, scale up on failover |
| Warm standby | Minutes | Seconds to minutes | $$$ | Scaled-down full copy running, scale up and switch traffic |
| Multi-site active-active | Near zero | Near zero | $$$$ | Both sites serve traffic, conflict handling required |

Rules: a backup is only real after a successful timed restore; store copies in another account and region, immutable (S3 Object Lock, vault lock); RPO comes from backup or replication frequency, RTO from restore plus DNS plus warm-up; DR runbooks are tested on a schedule.

## k6

| Task | Command |
|------|---------|
| Run a script | `k6 run load.js` |
| Quick ad-hoc test | `k6 run --vus 20 --duration 1m load.js` |
| Pass variables | `k6 run -e BASE_URL=https://staging.example.com load.js` (read with `__ENV.BASE_URL`) |
| Export summary | `k6 run --summary-export=summary.json load.js` |
| Raw results | `k6 run --out json=results.json load.js` · `--out csv=results.csv` |
| Stream to Prometheus | `K6_PROMETHEUS_RW_SERVER_URL=http://prom:9090/api/v1/write k6 run -o experimental-prometheus-rw load.js` (Prometheus needs `--web.enable-remote-write-receiver`) |
| Inspect options | `k6 inspect load.js` |
| Bundle for remote runs | `k6 archive load.js` |
| Run in Docker | `docker run --rm -i grafana/k6:1.0.0 run - <load.js` |
| Kubernetes, distributed | k6 Operator with a `TestRun` CR (`parallelism: 4`) |
| Exit code when a threshold fails | `99` (use it to fail CI) |

| Executor | Model | Use for |
|----------|-------|---------|
| `constant-vus` | Closed | Simple soak with fixed concurrency |
| `ramping-vus` | Closed | Ramp concurrency |
| `constant-arrival-rate` | Open | SLO gates: fixed requests per second regardless of latency |
| `ramping-arrival-rate` | Open | Find the knee, stress tests |
| `per-vu-iterations` / `shared-iterations` | Closed | Fixed amount of work |
| `externally-controlled` | Closed | Control VUs at runtime via the REST API |

```javascript
export const options = {
  scenarios: {
    checkout: { executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '10m',
                preAllocatedVUs: 50, maxVUs: 300 },
  },
  thresholds: {
    http_req_failed: [{ threshold: 'rate<0.001', abortOnFail: true, delayAbortEval: '1m' }],
    http_req_duration: ['p(99)<250', 'p(95)<150'],
    'http_req_duration{scenario:checkout}': ['p(99)<300'],
    checks: ['rate>0.999'],
  },
};
```

Test types: smoke (minimal load, script works), average load, stress (beyond normal), spike (sudden burst), soak (hours, finds leaks), breakpoint (ramp until failure).

## Chaos Mesh

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
# k3s: socketPath=/run/k3s/containerd/containerd.sock
```

| Task | Command |
|------|---------|
| List experiments | `kubectl get podchaos,networkchaos,stresschaos,iochaos,httpchaos,timechaos -A` |
| Status and conditions | `kubectl describe networkchaos inventory-latency -n shop` (look for `AllInjected` / `AllRecovered`) |
| Pause | `kubectl annotate networkchaos inventory-latency -n shop experiment.chaos-mesh.org/pause=true` |
| Resume | `kubectl annotate networkchaos inventory-latency -n shop experiment.chaos-mesh.org/pause-` |
| Stop (recover) | `kubectl delete networkchaos inventory-latency -n shop` |
| Emergency stop everything | `kubectl delete podchaos,networkchaos,stresschaos,iochaos,httpchaos,timechaos --all -A` |
| Daemon logs (tc / injection errors) | `kubectl -n chaos-mesh logs ds/chaos-daemon` |
| Restrict to opted-in namespaces | install with `--set controllerManager.enableFilterNamespace=true`, then `kubectl annotate ns shop chaos-mesh.org/inject=enabled` |

| Kind | Actions / purpose |
|------|-------------------|
| `PodChaos` | `pod-kill`, `pod-failure`, `container-kill` |
| `NetworkChaos` | `delay`, `loss`, `duplicate`, `corrupt`, `partition`, `bandwidth` (with `direction` and `target`) |
| `StressChaos` | CPU and memory stress (`stressors.cpu.workers`, `stressors.memory.size`) |
| `IOChaos` | Disk latency, faults, attribute changes |
| `HTTPChaos` | Abort, delay, replace or patch HTTP requests/responses on a port |
| `DNSChaos` | DNS errors or random answers |
| `TimeChaos` | Clock skew |
| `KernelChaos`, `JVMChaos` | Kernel fault injection, JVM exceptions and latency |
| `AWSChaos`, `GCPChaos`, `AzureChaos` | Stop instances, detach volumes |
| `Schedule`, `Workflow` | Recurring experiments; serial / parallel multi-step experiments |

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata: {name: inventory-latency, namespace: shop}
spec:
  action: delay
  mode: all                    # one | all | fixed | fixed-percent | random-max-percent
  selector:
    namespaces: [shop]
    labelSelectors: {app: inventory}
  delay: {latency: "300ms", jitter: "50ms", correlation: "0"}
  direction: to
  duration: "10m"
```

Experiment record: steady state (SLIs), hypothesis, blast radius, abort condition, rollback, result, findings.

## LitmusChaos and AWS FIS

| Tool | Key commands / objects |
|------|------------------------|
| LitmusChaos | `ChaosExperiment` (what), `ChaosEngine` (run it against an app: `appinfo`, `experiments`, `chaosServiceAccount`), `ChaosResult` (verdict) |
| Litmus experiments | `pod-delete`, `container-kill`, `pod-network-latency`, `pod-network-loss`, `pod-cpu-hog`, `node-drain` |
| Litmus probes | `httpProbe`, `cmdProbe`, `k8sProbe`, `promProbe`; modes `SOT`, `EOT`, `Edge`, `Continuous`, `OnChaos` |
| Litmus result | `kubectl get chaosresult -n app -o jsonpath='{.items[*].status.experimentStatus.verdict}'` |
| FIS: create template | `aws fis create-experiment-template --cli-input-json file://template.json` |
| FIS: run / stop | `aws fis start-experiment --experiment-template-id EXT123` · `aws fis stop-experiment --id EXP456` |
| FIS: status | `aws fis get-experiment --id EXP456 --query 'experiment.state'` |
| FIS actions | `aws:ec2:stop-instances`, `aws:ec2:terminate-instances`, `aws:eks:pod-delete`, `aws:eks:pod-network-latency`, `aws:eks:terminate-nodegroup-instances`, `aws:network:disrupt-connectivity`, `aws:rds:failover-db-cluster`, `aws:fis:inject-api-internal-error`, `aws:ssm:send-command` |
| FIS safety | `stopConditions: [{"source": "aws:cloudwatch:alarm", "value": "<alarm ARN>"}]`, targets by tag with `selectionMode` (`ALL`, `COUNT(1)`, `PERCENT(25)`) |

---

## Top 15 interview one-liners

1. **SRE vs DevOps:** DevOps is the culture of shared ownership; SRE is a concrete implementation with SLOs, error budgets, a toil cap and blameless postmortems.
2. **SLI vs SLO vs SLA:** the SLI measures user experience, the SLO is our internal target for it, the SLA is the contract with penalties and is always looser than the SLO.
3. **Good SLI:** a ratio of good events to valid events, measured as close to the user as possible, such as at the load balancer.
4. **Error budget:** `1 - SLO`. It turns "is it safe to ship?" into a number, and the error-budget policy says what happens when it runs out.
5. **Why not 100%:** users cannot tell 99.99% from 100% through their own networks, each nine costs roughly 10x more, and 100% means you can never change anything.
6. **Nines to remember:** 99.9% is 43 minutes a month, 99.99% is 4.3 minutes, 99.999% is about 5 minutes a year.
7. **Burn rate:** error ratio divided by the budget fraction; 14.4x for an hour spends 2% of a 30-day budget.
8. **Multi-window alerting:** page when both a long and a short window exceed the burn threshold (1h/5m at 14.4x, 6h/30m at 6x), ticket at 3x and 1x; precise, fast to detect, fast to reset.
9. **Page vs ticket:** page only for urgent, actionable, user-impacting symptoms; everything else is a ticket or a dashboard.
10. **Serial vs parallel availability:** serial dependencies multiply (three 99.9% → 99.7%); independent redundant copies combine as `1 - (1 - a)^n`.
11. **Incident response:** declare early, split IC, Ops and Comms, mitigate before root cause, and communicate on a fixed schedule.
12. **Blameless postmortem:** assume people acted reasonably; fix the system that let a reasonable action cause harm; every action item has an owner and a date.
13. **Toil:** manual, repetitive, automatable work without lasting value that scales with the service; measure it and keep it under 50%.
14. **Retries:** only for idempotent operations, with exponential backoff, jitter and a retry budget, at one layer only, or a blip becomes a retry storm.
15. **Chaos engineering:** an experiment with steady state, hypothesis, limited blast radius and automatic stop conditions, not random breakage; it proves your failure handling works before an outage does.

---

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
