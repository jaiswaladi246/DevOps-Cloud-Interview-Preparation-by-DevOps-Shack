# Monitoring & Observability — Cheatsheet

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

[⬅ Back to Monitoring & Observability](./README.md) · [🏠 Home](../README.md)

Names follow the [hands-on lab](./hands-on-lab.md): `kube-prometheus-stack` release `kps` with `fullnameOverride: kps`, namespace `observability`.

## Contents
- [Golden signals, RED and USE](#golden-signals-red-and-use)
- [PromQL patterns](#promql-patterns)
- [Recording and alert rules](#recording-and-alert-rules)
- [SLO burn-rate reference](#slo-burn-rate-reference)
- [Alertmanager](#alertmanager)
- [LogQL](#logql)
- [OpenTelemetry Collector skeleton](#opentelemetry-collector-skeleton)
- [Exporters and ports](#exporters-and-ports)
- [Prometheus server, API and promtool](#prometheus-server-api-and-promtool)
- [kube-prometheus-stack commands](#kube-prometheus-stack-commands)
- [Grafana quick reference](#grafana-quick-reference)
- [CloudWatch quick reference](#cloudwatch-quick-reference)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

## Golden signals, RED and USE

| Method | Applies to | Signals | Example PromQL |
|---|---|---|---|
| Four Golden Signals (Google SRE) | Any user-facing system | Latency, Traffic, Errors, Saturation | see RED plus a saturation gauge |
| RED (Tom Wilkie) | Request-driven services | **R**ate | `sum by (service) (rate(http_requests_total[5m]))` |
| | | **E**rrors | `sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total[5m]))` |
| | | **D**uration | `histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))` |
| USE (Brendan Gregg) | Resources: CPU, memory, disk, network, queues | **U**tilisation | `1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))` |
| | | **S**aturation | `node_load5 / count by (instance) (node_cpu_seconds_total{mode="idle"})` |
| | | **E**rrors | `rate(node_network_receive_errs_total[5m])` |

Rule of thumb: RED for services, USE for infrastructure, golden signals on the top row of every service dashboard. Page on symptoms (RED, SLO burn), ticket on causes (USE).

## PromQL patterns

### Selectors and metric types

| Pattern | Example |
|---|---|
| Exact / not equal | `up{job="api"}` / `up{job!="api"}` |
| Regex / negative regex | `{status=~"5.."}` / `{container!~"POD\|"}` |
| Any metric by name regex | `{__name__=~"http_.*"}` |
| Range vector | `http_requests_total[5m]` |
| Offset / fixed time | `rate(x[5m] offset 1d)` / `x @ 1735689600` |
| Counter | always `rate()` / `increase()`, never graph raw |
| Gauge | use directly, `avg_over_time`, `max_over_time`, `deriv` |
| Histogram | `histogram_quantile()` over `sum by (le)` of `_bucket` rates |
| Summary | per-instance quantiles only; cannot be aggregated |

### Functions you use daily

| Need | Query |
|---|---|
| Per-second rate | `rate(http_requests_total[5m])` |
| Total over window | `increase(http_requests_total[1h])` |
| Fast-moving spikes (graphs only) | `irate(http_requests_total[1m])` |
| Aggregate, keep labels | `sum by (service, status) (rate(...[5m]))` |
| Aggregate, drop labels | `sum without (pod, instance) (rate(...[5m]))` |
| Top N | `topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))` |
| Quantile from histogram | `histogram_quantile(0.95, sum by (le) (rate(x_bucket[5m])))` |
| Average latency | `sum(rate(x_sum[5m])) / sum(rate(x_count[5m]))` |
| Missing series | `absent(up{job="api"} == 1)` / `absent_over_time(x[15m])` |
| Trend forecast | `predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0` |
| Change count | `changes(kube_pod_container_status_restarts_total[1h])` |
| Compare with last week | `sum(rate(x[5m])) / sum(rate(x[5m] offset 7d))` |
| Subquery | `max_over_time(sum(rate(x[5m]))[1d:5m])` |
| Rename / build label | `label_replace(up, "host", "$1", "instance", "(.*):.*")` |
| Clamp | `clamp_min(x, 0)` / `clamp_max(x, 1)` |
| Series count per metric | `topk(10, count by (__name__)({__name__=~".+"}))` |

### Binary operators and joins

```promql
# Ratio with matching label sets
sum by (service) (rate(errors_total[5m])) / sum by (service) (rate(requests_total[5m]))

# Only these labels must match
a / on (namespace, pod) b

# Copy a label from the "one" side (kube_pod_info has value 1)
sum by (namespace, pod) (rate(http_requests_total[5m]))
  * on (namespace, pod) group_left (node) max by (namespace, pod, node) (kube_pod_info)

# Set operations
up{job="api"} == 0 and on (instance) node_boot_time_seconds > 0
rate(x[5m]) unless on (pod) kube_pod_status_phase{phase="Succeeded"} == 1
vector(0) or sum(rate(x[5m]))          # default to 0 when empty
```

### Kubernetes queries

```promql
# CPU cores used per pod
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Memory working set vs limit (%)
100 * sum by (namespace, pod, container) (container_memory_working_set_bytes{container!=""})
    / sum by (namespace, pod, container) (kube_pod_container_resource_limits{resource="memory"})

# CPU throttling ratio
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m]))
  / sum by (pod) (rate(container_cpu_cfs_periods_total[5m]))

# Restarts in last hour, OOM kills
increase(kube_pod_container_status_restarts_total[1h]) > 0
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# Replicas mismatch, pending pods, not-ready nodes
kube_deployment_spec_replicas != kube_deployment_status_replicas_available
sum by (namespace) (kube_pod_status_phase{phase="Pending"}) > 0
kube_node_status_condition{condition="Ready", status="true"} == 0

# Cluster CPU requests vs allocatable
sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"})

# PVC almost full
kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
```

## Recording and alert rules

### Recording rules (`level:metric:operations`)

```yaml
groups:
  - name: http-recording
    interval: 30s
    rules:
      - record: service:http_requests:rate5m
        expr: sum by (service) (rate(http_requests_total[5m]))
      - record: service:http_errors:ratio_rate5m
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
          / sum by (service) (rate(http_requests_total[5m]))
      - record: service_le:http_request_duration_seconds_bucket:rate5m
        expr: sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
```

### Alert rule

```yaml
groups:
  - name: http-alerts
    rules:
      - alert: HighErrorRate
        expr: service:http_errors:ratio_rate5m > 0.05
        for: 5m
        keep_firing_for: 5m
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "{{ $labels.service }} 5xx ratio is {{ $value | humanizePercentage }}"
          description: "Error ratio above 5% for 5 minutes."
          runbook_url: "https://runbooks.example.com/high-error-rate"
          dashboard_url: "https://grafana.example.com/d/red?var-service={{ $labels.service }}"
      - alert: Watchdog                       # dead man's switch, always firing
        expr: vector(1)
        labels: { severity: none }
```

### As a `PrometheusRule` (operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: checkout-rules
  namespace: shop
  labels: { release: kps }        # must match ruleSelector unless ruleSelectorNilUsesHelmValues=false
spec:
  groups:
    - name: checkout
      rules:
        - alert: CheckoutDown
          expr: absent(up{job="checkout"} == 1)
          for: 2m
          labels: { severity: critical }
```

### Rule unit test

```yaml
# promtool test rules tests.yml
rule_files: [rules.yml]
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'service:http_errors:ratio_rate5m{service="checkout"}'
        values: '0.01x5 0.2x15'
    alert_rule_test:
      - eval_time: 8m
        alertname: HighErrorRate
        exp_alerts: []                 # still pending
      - eval_time: 15m
        alertname: HighErrorRate
        exp_alerts:
          - exp_labels: { severity: critical, team: payments, service: checkout }
```

## SLO burn-rate reference

Error budget = `1 - SLO`. Burn rate = `error ratio / (1 - SLO)`. For a 30-day window:

| SLO | Budget per 30 days | Error ratio at 14.4× | Error ratio at 6× |
|---|---|---|---|
| 99% | 7 h 12 m | 14.4% | 6% |
| 99.5% | 3 h 36 m | 7.2% | 3% |
| 99.9% | 43 m 12 s | 1.44% | 0.6% |
| 99.95% | 21 m 36 s | 0.72% | 0.3% |
| 99.99% | 4 m 19 s | 0.144% | 0.06% |

| Severity | Long window | Short window | Burn rate | Budget spent when it fires |
|---|---|---|---|---|
| Page | 1h | 5m | 14.4 | 2% |
| Page | 6h | 30m | 6 | 5% |
| Ticket | 1d | 2h | 3 | 10% |
| Ticket | 3d | 6h | 1 | 10% |

```promql
# Page: fast burn for a 99.9% SLO
(
  job:slo_errors:ratio_rate1h{job="checkout"} > (14.4 * 0.001)
  and
  job:slo_errors:ratio_rate5m{job="checkout"} > (14.4 * 0.001)
)
or
(
  job:slo_errors:ratio_rate6h{job="checkout"} > (6 * 0.001)
  and
  job:slo_errors:ratio_rate30m{job="checkout"} > (6 * 0.001)
)
```

Generators: Sloth, Pyrra, Grafana SLO, OpenSLO specs.

## Alertmanager

### Config skeleton

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default-slack
  group_by: [alertname, cluster, namespace]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="critical"]
      receiver: pagerduty
      continue: true                  # also notify the team channel below
    - matchers: [team="payments"]
      receiver: payments-slack
    - matchers: [severity="ticket"]
      receiver: jira-webhook
      active_time_intervals: [business-hours]
    - matchers: [alertname="Watchdog"]
      receiver: deadmans-snitch
      repeat_interval: 1m

inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: [alertname, namespace]
  - source_matchers: [alertname="NodeDown"]
    target_matchers: [severity=~"warning|info"]
    equal: [instance]

time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: ["monday:friday"]
        times: [{ start_time: "09:00", end_time: "18:00" }]
        location: Asia/Kolkata

receivers:
  - name: default-slack
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack/url
        channel: "#alerts"
        send_resolved: true
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }} <{{ .Annotations.runbook_url }}|runbook>{{ "\n" }}{{ end }}'
  - name: pagerduty
    pagerduty_configs:
      - routing_key_file: /etc/alertmanager/secrets/pd/key
  - name: payments-slack
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack/url
        channel: "#payments-alerts"
  - name: jira-webhook
    webhook_configs:
      - url: http://jira-bridge.observability.svc:8080/alert
  - name: deadmans-snitch
    webhook_configs:
      - url: https://nosnch.in/REPLACE_ME
```

### Timing knobs

| Setting | Meaning | Typical |
|---|---|---|
| `group_wait` | Wait before first notification of a new group | 30s |
| `group_interval` | Wait before notifying about new alerts in an existing group | 5m |
| `repeat_interval` | Re-send an unchanged, still-firing group | 4h (critical), 12–24h (warning) |
| `resolve_timeout` | Mark resolved if Prometheus stops sending (alerts without `endsAt`) | 5m |

### amtool

```bash
export AMURL=http://localhost:9093
amtool check-config alertmanager.yml
amtool config routes show --config.file=alertmanager.yml
amtool config routes test --config.file=alertmanager.yml severity=critical team=payments
amtool --alertmanager.url=$AMURL alert query
amtool --alertmanager.url=$AMURL alert add TestAlert severity=warning team=payments --annotation=summary="test"
amtool --alertmanager.url=$AMURL silence add alertname=HighErrorRate service=checkout --duration=2h --comment="deploy INC-123"
amtool --alertmanager.url=$AMURL silence query
amtool --alertmanager.url=$AMURL silence expire <silence-id>
```

## LogQL

### Shape of a query

```
{stream selector}  | line filter  | parser  | label filter  | formatting
{app="checkout"}   |= "error"     | json    | status >= 500 | line_format "{{.msg}}"
```

| Part | Examples |
|---|---|
| Stream selector (indexed) | `{namespace="prod", app="checkout"}`, `{app=~"api\|web"}` |
| Line filters (fast, use first) | `\|= "error"`, `!= "healthz"`, `\|~ "timeout\|refused"`, `!~ "debug"` |
| Parsers | `\| json`, `\| logfmt`, `\| pattern "<ip> - <_> [<_>] \"<method> <path> <_>\" <status> <_>"`, `\| regexp "(?P<code>\\d{3})"` |
| Label filters | `\| level="error"`, `\| status >= 500`, `\| duration > 1s`, `\| __error__=""` |
| Formatting | `\| line_format "{{.level}} {{.msg}}"`, `\| label_format svc=app` |
| Drop / keep labels | `\| drop pod, filename`, `\| keep app, level` |

### Metric queries

```logql
# Error lines per app per minute
sum by (app) (count_over_time({namespace="prod"} |= "error" [1m]))

# Log lines per second by level (JSON logs)
sum by (level) (rate({app="checkout"} | json | __error__="" [5m]))

# 5xx ratio from access logs
sum(rate({app="ingress-nginx"} | pattern "<_> <_> <_> [<_>] \"<_> <_> <_>\" <status> <_>" | status >= 500 [5m]))
/ sum(rate({app="ingress-nginx"} [5m]))

# p95 duration per route from a numeric field
quantile_over_time(0.95, {app="checkout"} | json | unwrap duration_ms [5m]) by (route)

# Top 10 pods by log volume
topk(10, sum by (namespace, pod) (bytes_over_time({namespace=~".+"}[1h])))

# Everything for a trace ID
{namespace=~"prod|payments"} |= "4bf92f3577b34da6a3ce929d0e0e4736"
```

### Loki ruler alert

```yaml
groups:
  - name: checkout-logs
    rules:
      - alert: CheckoutErrorLogsHigh
        expr: sum(rate({app="checkout"} | json | level="error" [5m])) > 2
        for: 10m
        labels: { severity: warning }
        annotations: { summary: "checkout logs more than 2 errors/s" }
```

Label rules: low-cardinality only (`namespace`, `app`, `container`, `level`, `cluster`). Never `pod` at high churn, `user_id`, `trace_id` or `status`; put those in the line or structured metadata.

```bash
# logcli
export LOKI_ADDR=http://localhost:3100
logcli labels
logcli labels app
logcli query '{app="checkout"} |= "error"' --since=1h --limit=50
logcli series '{namespace="prod"}' --analyze-labels
```

## OpenTelemetry Collector skeleton

```yaml
extensions:
  health_check: {}                       # :13133
  file_storage: { directory: /var/lib/otelcol/queue }

receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }
  prometheus:
    config:
      scrape_configs:
        - job_name: otelcol
          static_configs: [{ targets: ["localhost:8888"] }]

processors:
  memory_limiter: { check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 20 }
  k8sattributes: {}
  resourcedetection: { detectors: [env, system] }
  resource:
    attributes:
      - { key: deployment.environment, value: prod, action: upsert }
  filter/healthz:
    traces:
      span:
        - 'attributes["http.route"] == "/healthz"'
  tail_sampling:
    decision_wait: 10s
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,   type: latency,     latency: { threshold_ms: 500 } }
      - { name: sample, type: probabilistic, probabilistic: { sampling_percentage: 5 } }
  batch: { send_batch_size: 8192, timeout: 5s }

connectors:
  spanmetrics: {}                        # RED metrics from spans, before sampling

exporters:
  otlp/tempo:
    endpoint: tempo.observability.svc:4317
    tls: { insecure: true }
    sending_queue: { enabled: true, storage: file_storage }
  prometheusremotewrite:
    endpoint: http://mimir-gateway.observability.svc/api/v1/push
  otlphttp/loki:
    endpoint: http://loki.observability.svc:3100/otlp
  debug: { verbosity: basic }

service:
  extensions: [health_check, file_storage]
  pipelines:
    traces/in:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resourcedetection, resource, filter/healthz]
      exporters: [spanmetrics, otlp/tempo]
    metrics:
      receivers: [otlp, prometheus, spanmetrics]
      processors: [memory_limiter, k8sattributes, batch]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, batch]
      exporters: [otlphttp/loki]
```

In a real deployment `tail_sampling` runs on a gateway tier behind the `loadbalancing` exporter (routing by `traceID`); add it to the gateway's traces pipeline before `batch`. Order rule: `memory_limiter` first, `batch` last.

```bash
otelcol-contrib validate --config=config.yaml
curl -s localhost:13133            # health
curl -s localhost:8888/metrics | grep -E 'otelcol_(exporter_send_failed|processor_refused|exporter_queue_size)'
```

SDK environment variables: `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod`, `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317`, `OTEL_EXPORTER_OTLP_PROTOCOL=grpc`, `OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.1`, `OTEL_PROPAGATORS=tracecontext,baggage`.

## Exporters and ports

| Component | Default port | Key metrics / notes |
|---|---|---|
| Prometheus | 9090 | `prometheus_tsdb_head_series`, `/targets`, `/rules`, `/tsdb-status` |
| Alertmanager | 9093 (cluster gossip 9094) | `alertmanager_notifications_failed_total` |
| Pushgateway | 9091 | batch jobs only, scrape with `honor_labels: true` |
| Grafana | 3000 | `/api/health` |
| node_exporter | 9100 | `node_cpu_seconds_total`, `node_filesystem_avail_bytes`, `node_load5` |
| blackbox_exporter | 9115 | `probe_success`, `probe_ssl_earliest_cert_expiry`, `probe_duration_seconds` |
| kube-state-metrics | 8080 (self metrics 8081) | `kube_pod_status_phase`, `kube_deployment_status_replicas_available` |
| kubelet / cAdvisor | 10250 (`/metrics/cadvisor`) | `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes` |
| mysqld_exporter | 9104 | `mysql_up`, `mysql_global_status_threads_connected` |
| postgres_exporter | 9187 | `pg_up`, `pg_stat_database_numbackends` |
| redis_exporter | 9121 | `redis_up`, `redis_memory_used_bytes` |
| nginx-prometheus-exporter | 9113 | `nginx_connections_active` |
| kafka_exporter | 9308 | `kafka_consumergroup_lag` |
| elasticsearch_exporter | 9114 | `elasticsearch_cluster_health_status` |
| windows_exporter | 9182 | `windows_cpu_time_total` |
| statsd_exporter | 9102 (StatsD in 9125/udp) | mapping config turns dotted names into labels |
| cloudwatch_exporter | 9106 | Java, per-metric config |
| YACE (CloudWatch) | 5000 | tag-based discovery of AWS resources |
| NVIDIA DCGM exporter | 9400 | `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_XID_ERRORS` |
| Loki | 3100 (gRPC 9095) | `/ready`, `/loki/api/v1/push`, `/otlp` |
| Tempo | 3200 (OTLP 4317/4318) | `/ready` |
| Jaeger UI | 16686 | OTLP 4317/4318 for ingest |
| OTel Collector | OTLP 4317 gRPC, 4318 HTTP; self metrics 8888; health 13133 | `otelcol_receiver_accepted_spans` |
| Fluent Bit | 2020 (HTTP metrics) | `fluentbit_output_retries_failed_total` |
| Grafana Alloy | 12345 (UI and metrics) | component graph in the UI |
| Thanos sidecar | 10901 gRPC, 10902 HTTP | `thanos_objstore_bucket_operation_failures_total` |

## Prometheus server, API and promtool

```bash
# Useful server flags
prometheus --config.file=prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=90GB \
  --web.enable-lifecycle \
  --enable-feature=exemplar-storage,native-histograms
prometheus --agent --config.file=agent.yml          # agent mode: scrape and remote_write only

# HTTP API
curl -s localhost:9090/-/healthy ; curl -s localhost:9090/-/ready
curl -s -X POST localhost:9090/-/reload                       # needs --web.enable-lifecycle
curl -s 'localhost:9090/api/v1/query' --data-urlencode 'query=up == 0' | jq '.data.result'
curl -s 'localhost:9090/api/v1/query_range' --data-urlencode 'query=rate(http_requests_total[5m])' \
  -d start=$(date -d '-1 hour' +%s) -d end=$(date +%s) -d step=60
curl -s 'localhost:9090/api/v1/targets?state=active' | jq '.data.activeTargets[] | select(.health!="up") | {job: .labels.job, err: .lastError}'
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].name'
curl -s localhost:9090/api/v1/alerts | jq '.data.alerts[] | {a: .labels.alertname, s: .state}'
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'
curl -s localhost:9090/api/v1/status/config | jq -r .data.yaml
curl -s 'localhost:9090/api/v1/label/job/values'

# promtool
promtool check config prometheus.yml
promtool check rules rules/*.yml
promtool test rules tests/*.yml
promtool query instant http://localhost:9090 'sum(up)'
promtool tsdb analyze /prometheus                              # cardinality report from blocks
promtool tsdb create-blocks-from openmetrics data.om /prometheus   # backfill
```

### Scrape config essentials

```yaml
global: { scrape_interval: 30s, evaluation_interval: 30s, external_labels: { cluster: prod-eu-1 } }
scrape_configs:
  - job_name: app
    kubernetes_sd_configs: [{ role: pod }]
    relabel_configs:                         # before scrape: choose targets, set labels
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
    metric_relabel_configs:                  # after scrape: drop series and labels
      - source_labels: [__name__]
        regex: "go_gc_.*|http_request_size_bytes_bucket"
        action: drop
      - regex: user_id
        action: labeldrop
    sample_limit: 50000
remote_write:
  - url: https://mimir.example.com/api/v1/push
    queue_config: { max_samples_per_send: 2000, capacity: 10000, max_shards: 50 }
```

## kube-prometheus-stack commands

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions | head
helm show values prometheus-community/kube-prometheus-stack > default-values.yaml
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n observability --create-namespace -f kps-values.yaml --version <pinned>
helm get values kps -n observability -a | less               # effective values
helm diff upgrade kps prometheus-community/kube-prometheus-stack -f kps-values.yaml   # helm-diff plugin

kubectl get crd | grep monitoring.coreos.com
kubectl get servicemonitors,podmonitors,probes,prometheusrules -A
kubectl -n observability get prometheus kps-prometheus -o jsonpath='{.spec.serviceMonitorSelector}'
kubectl -n observability logs deploy/kps-operator | grep -i -E 'error|skipping'
kubectl -n observability logs prometheus-kps-prometheus-0 -c prometheus | tail
kubectl -n observability logs alertmanager-kps-alertmanager-0 -c alertmanager | grep -i notify

kubectl -n observability port-forward svc/kps-prometheus 9090:9090
kubectl -n observability port-forward svc/kps-alertmanager 9093:9093
kubectl -n observability port-forward svc/kps-grafana 3000:80
kubectl -n observability get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d

# CRDs are not upgraded by helm upgrade: apply the matching operator CRDs first
kubectl apply --server-side --force-conflicts -f <prometheus-operator-crds-dir>/
```

### Values you will touch

| Value | Why |
|---|---|
| `fullnameOverride: kps` | short, predictable resource names |
| `prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues: false` | pick up every `ServiceMonitor`, not only `release: kps` |
| `...podMonitorSelectorNilUsesHelmValues`, `...ruleSelectorNilUsesHelmValues`, `...probeSelectorNilUsesHelmValues` | same for PodMonitor, PrometheusRule, Probe |
| `prometheus.prometheusSpec.retention` / `retentionSize` | time and disk-based retention |
| `prometheus.prometheusSpec.storageSpec.volumeClaimTemplate` | persistent storage |
| `prometheus.prometheusSpec.externalLabels` | `cluster`, `env` on every series sent out |
| `prometheus.prometheusSpec.remoteWrite` | ship to Mimir, Thanos Receive, AMP |
| `prometheus.prometheusSpec.enableFeatures` | `exemplar-storage`, `native-histograms` |
| `alertmanager.config` / `alertmanager.alertmanagerSpec.alertmanagerConfigSelector` | global routing / team `AlertmanagerConfig`s |
| `grafana.sidecar.dashboards.searchNamespace: ALL` | dashboards from any namespace |
| `kubeEtcd/kubeScheduler/kubeControllerManager.enabled: false` | managed control planes |
| `defaultRules.disabled.<AlertName>: true` | turn off a single noisy default alert |

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: shopfront
  namespace: shop
  labels: { release: kps }
spec:
  selector: { matchLabels: { app: shopfront } }
  namespaceSelector: { matchNames: [shop] }
  endpoints:
    - port: http-metrics          # Service port NAME, not number
      path: /metrics
      interval: 30s
      metricRelabelings:
        - { action: drop, sourceLabels: [__name__], regex: "go_gc_.*" }
  sampleLimit: 20000
```

## Grafana quick reference

| Task | How |
|---|---|
| Rate window in panels | `rate(x[$__rate_interval])` |
| Variable from labels | `label_values(http_requests_total{namespace="$namespace"}, service)` |
| Multi-value variable in regex | `{service=~"$service"}` |
| Variable from query result | `query_result(topk(10, sum by (pod) (rate(x[5m]))))` with regex `/pod="([^"]+)"/` |
| Ad hoc time comparison | panel "Query options" → time shift `1w` |
| Deploy annotations | annotation query `changes(build_info{service="$service"}[$__interval]) > 0` |
| Provision data source | `/etc/grafana/provisioning/datasources/*.yaml` |
| Provision dashboards | `/etc/grafana/provisioning/dashboards/*.yaml` provider pointing at a JSON folder |
| Dashboards on Kubernetes | ConfigMap labelled `grafana_dashboard: "1"` (kube-prometheus-stack sidecar) |
| As code | Terraform `grafana_dashboard`, Grafonnet, Grafana Foundation SDK, Grafana Operator |
| Loki → Tempo link | Loki data source derived field: regex `"trace_id":"(\w+)"`, internal link to Tempo |
| Health check | `curl -s localhost:3000/api/health` |

## CloudWatch quick reference

```bash
# Alarm on EC2 CPU
aws cloudwatch put-metric-alarm --alarm-name api-cpu-high \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average --period 60 --evaluation-periods 5 --datapoints-to-alarm 3 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching --alarm-actions arn:aws:sns:ap-south-1:111122223333:oncall

aws cloudwatch describe-alarms --state-value ALARM --query 'MetricAlarms[].AlarmName'
aws cloudwatch set-alarm-state --alarm-name api-cpu-high --state-value ALARM --state-reason "test"
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/0123456789abcdef \
  --start-time $(date -u -d '-1 hour' +%FT%TZ) --end-time $(date -u +%FT%TZ) --period 300 --statistics Sum

# Logs
aws logs put-retention-policy --log-group-name /app/checkout --retention-in-days 14
aws logs tail /app/checkout --follow --since 10m --filter-pattern ERROR
aws logs put-metric-filter --log-group-name /app/checkout --filter-name 5xx \
  --filter-pattern '{ $.status >= 500 }' \
  --metric-transformations metricName=Checkout5xx,metricNamespace=App,metricValue=1
```

```
# Logs Insights
fields @timestamp, status, route, duration_ms
| filter status >= 500
| stats count() as errors, pct(duration_ms, 95) as p95 by bin(5m), route
| sort errors desc
```

| AWS service | Metric to alert on |
|---|---|
| ALB | `HTTPCode_Target_5XX_Count`, `TargetResponseTime` (p99), `UnHealthyHostCount` |
| RDS | `CPUUtilization`, `FreeStorageSpace`, `DatabaseConnections`, `ReplicaLag` |
| Lambda | `Errors`, `Throttles`, `Duration` (p99), `ConcurrentExecutions` |
| SQS | `ApproximateAgeOfOldestMessage`, `ApproximateNumberOfMessagesVisible` |
| DynamoDB | `ThrottledRequests`, `SystemErrors` |

## Top 15 interview one-liners

1. **Monitoring vs observability:** monitoring answers questions you predicted; observability lets you ask new questions of the system without shipping new code.
2. **Pull vs push:** Prometheus pulls metrics on its schedule (and knows when a target is down via `up`); logs and traces are pushed by agents and SDKs.
3. **Counter rule:** always `rate()` a counter before you `sum()` it, never the reverse, because `rate()` must see the per-series resets.
4. **Histogram vs summary:** histograms can be aggregated across instances with `histogram_quantile()`; summary quantiles cannot be averaged.
5. **Cardinality:** every unique label combination is a new series; never use unbounded values like user IDs, request IDs or raw URLs as labels.
6. **`for:` clause:** keeps an alert pending until the condition holds for the full duration, filtering out short blips.
7. **Alertmanager's job:** it never evaluates queries; it groups, deduplicates, inhibits, silences and routes what Prometheus sends.
8. **Page on symptoms:** page when users hurt (SLO burn, error rate, latency), ticket on causes (CPU, disk, a single pod restart).
9. **Error budget:** `1 - SLO`; 99.9% over 30 days is 43.2 minutes, and burn rate 14.4 spends 2% of it in one hour.
10. **Multi-window burn rate:** a long window proves the problem is significant, a short window proves it is still happening, so alerts fire fast and reset fast.
11. **RED vs USE:** RED (rate, errors, duration) for services, USE (utilisation, saturation, errors) for resources.
12. **Why Loki is cheap:** it indexes only a few labels and stores compressed log chunks in object storage, filtering content at query time.
13. **Head vs tail sampling:** head decides at the start (cheap, blind to outcome), tail decides after the trace ends (keeps errors and slow traces, needs a stateful tier routed by trace ID).
14. **ServiceMonitor not picked up:** its labels must match the Prometheus `serviceMonitorSelector` (by default the Helm `release` label) and its `port` must be the Service port name.
15. **Scaling Prometheus:** one HA pair per cluster for scraping and alerting, `remote_write` or Thanos sidecar to Mimir or Thanos for long-term, global, deduplicated queries.

[⬅ Back to Monitoring & Observability](./README.md) · [🏠 Home](../README.md)

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
