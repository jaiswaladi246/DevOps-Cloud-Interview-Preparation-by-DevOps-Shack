# Monitoring & Observability — Assignments

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

> 25 assignments · 🟢 Beginner (10) · 🟡 Intermediate (10) · 🔴 Advanced (5)

[⬅ Back to Monitoring & Observability](./README.md) · [🏠 Home](../README.md)

Beginner assignments run on Docker Compose on your laptop. Intermediate and advanced ones use a local `kind` cluster with `kube-prometheus-stack` (the same setup as the [hands-on lab](./hands-on-lab.md), release name `kps`, namespace `observability`). The CloudWatch assignments need an AWS account; delete what you create afterwards. Try each one before opening the hints.

## Table of contents
- [🟢 Beginner](#-beginner) — A1–A10
- [🟡 Intermediate](#-intermediate) — A11–A20
- [🔴 Advanced](#-advanced) — A21–A25

---

## 🟢 Beginner

### A1. Prometheus and node_exporter with Docker Compose 🟢

**Objective:** Run Prometheus locally, scrape a host exporter and understand the scrape model.

**Tasks**
1. Write a `docker-compose.yml` with `prom/prometheus` and `prom/node-exporter` on one network.
2. Write `prometheus.yml` with `scrape_interval: 15s` and two jobs: `prometheus` (itself on `:9090`) and `node` (`node-exporter:9100`).
3. Open `http://localhost:9090/targets` and confirm both targets are `UP`.
4. Stop node-exporter and watch `up{job="node"}` go to 0 within one scrape interval.
5. Open `http://localhost:9100/metrics` and find one counter, one gauge and one metric with labels.

**Expected outcome:** Both targets are `UP`, you can explain what the `up` metric is and where the `job` and `instance` labels come from.

<details>
<summary>Hints</summary>

Mount the config with `./prometheus.yml:/etc/prometheus/prometheus.yml:ro`. Inside Compose, use service names as hostnames, not `localhost`. `node_cpu_seconds_total` is a counter, `node_memory_MemAvailable_bytes` is a gauge. Reload config without a restart by starting Prometheus with `--web.enable-lifecycle` and running `curl -X POST localhost:9090/-/reload`.
</details>

**Bonus:** Add `scrape_timeout: 5s` and a `relabel_configs` rule that sets a static `env="lab"` label on the node job.

### A2. Your first ten PromQL queries 🟢

**Objective:** Get fluent with selectors, `rate`, `increase` and aggregation.

**Tasks**
Using the setup from A1, write and save queries that return:
1. CPU utilisation per instance as a percentage (from `node_cpu_seconds_total`).
2. Available memory as a percentage of total.
3. Per-second network receive bytes per interface, excluding `lo`.
4. Total bytes received in the last hour.
5. Root filesystem free space percentage.
6. The number of HTTP requests Prometheus served per second, by handler.
7. The top 3 handlers by request rate.
8. The 5-minute load average divided by the CPU count.
9. Whether any target is down.
10. The same query as 1, but one hour ago (using `offset`).

**Expected outcome:** Each query returns a sensible value in the graph view, and you can explain why you used `rate` versus `increase` versus a raw gauge.

<details>
<summary>Hints</summary>

CPU: `100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))`. CPU count: `count by (instance) (node_cpu_seconds_total{mode="idle"})`. Prometheus' own HTTP metric is `prometheus_http_requests_total`. `increase(x[1h])` is `rate(x[1h]) * 3600`.
</details>

**Bonus:** Turn query 1 into a recording rule named `instance:node_cpu_utilisation:rate5m` and query the recorded series.

### A3. Instrument a small app with a client library 🟢

**Objective:** Expose custom application metrics with correct types and labels.

**Tasks**
1. Write a tiny HTTP service (Python `prometheus_client`, Go `client_golang` or Node `prom-client`) with routes `/` and `/checkout`.
2. Add a counter `http_requests_total` with labels `route`, `method`, `status`.
3. Add a histogram `http_request_duration_seconds` with buckets suited to 10 ms–2 s.
4. Add a gauge `app_inflight_requests`.
5. Make `/checkout` fail with a 500 about 5% of the time and sleep a random 50–400 ms.
6. Add the app as a scrape target in Prometheus and generate traffic with a loop of `curl`.

**Expected outcome:** `/metrics` shows the three metrics; in Prometheus you can graph request rate, error ratio and p95 latency for `/checkout`.

<details>
<summary>Hints</summary>

Use the route template as the label value, never the raw path with IDs. Python: `Histogram("http_request_duration_seconds", "...", ["route"], buckets=(.01,.025,.05,.1,.25,.5,1,2))`. Traffic loop: `while true; do curl -s localhost:8080/checkout >/dev/null; sleep 0.1; done`.
</details>

**Bonus:** Expose a `build_info` gauge with value 1 and labels `version` and `commit`, and use it in a query that shows error ratio by version.

### A4. Grafana with provisioned data source and a golden-signals dashboard 🟢

**Objective:** Run Grafana with configuration as files and build a first service dashboard.

**Tasks**
1. Add `grafana/grafana` to the Compose file from A1–A3.
2. Provision the Prometheus data source with a file in `provisioning/datasources/`.
3. Build a dashboard for the A3 app with four panels: traffic, errors (%), p50/p95/p99 latency, in-flight requests (saturation).
4. Add a `route` template variable populated by `label_values(http_requests_total, route)`.
5. Export the dashboard JSON and provision it from `provisioning/dashboards/` so it survives `docker compose down -v`.

**Expected outcome:** After a full restart with volumes deleted, Grafana comes back with the data source and dashboard already present.

<details>
<summary>Hints</summary>

The dashboards provider file needs `providers: [{ name: default, type: file, options: { path: /var/lib/grafana/dashboards } }]`. Use `$__rate_interval` instead of a hard-coded `[5m]`. Set percentage units on the error panel and seconds on latency.
</details>

**Bonus:** Add a stat panel showing the current error ratio with green/amber/red thresholds at 0.1% and 1%.

### A5. Write and validate your first alert rule 🟢

**Objective:** Create alerting rules with `for:`, labels and annotations, and validate them.

**Tasks**
1. Create `rules/app.yml` with an alert `AppHighErrorRate` firing when the 5xx ratio for `/checkout` is above 5% for 5 minutes.
2. Add `severity: warning`, a `summary`, a `description` that prints the value, and a `runbook_url`.
3. Add an `InstanceDown` alert on `up == 0` for 2 minutes.
4. Validate with `promtool check rules rules/app.yml` and load the file via `rule_files`.
5. Raise the failure rate in the app to 20% and watch the alert go inactive → pending → firing on `/alerts`.

**Expected outcome:** Both rules load without errors, and you can show the pending state and explain why `for:` exists.

<details>
<summary>Hints</summary>

Run `promtool` from the image: `docker run --rm -v $PWD/rules:/rules --entrypoint promtool prom/prometheus check rules /rules/app.yml`. In annotations, `{{ $value | humanizePercentage }}` formats a ratio nicely.
</details>

**Bonus:** Add `keep_firing_for: 5m` and explain which kind of flapping it prevents.

### A6. Alertmanager routing to Slack (or a webhook) 🟢

**Objective:** Deliver alerts to a receiver with grouping.

**Tasks**
1. Add `prom/alertmanager` to Compose and point Prometheus at it with the `alerting` block.
2. Configure a Slack receiver using `api_url_file` (or a local webhook receiver if you have no Slack).
3. Group by `alertname` and `job`, with `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`.
4. Validate with `amtool check-config`.
5. Trigger `InstanceDown` by stopping node-exporter, then start it again and confirm a resolved notification.
6. Create a 15-minute silence from the UI or `amtool` and confirm it suppresses the alert.

**Expected outcome:** One firing and one resolved message arrive, and the silence demonstrably mutes the alert.

<details>
<summary>Hints</summary>

Add `send_resolved: true` to the Slack config. `amtool --alertmanager.url=http://localhost:9093 silence add alertname=InstanceDown --duration=15m --comment="A6 test"`. A simple local webhook receiver: `docker run -p 8000:8080 mendhak/http-https-echo` and look at its logs.
</details>

**Bonus:** Write a custom Slack `title` and `text` template that includes `runbook_url` and the number of firing alerts in the group.

### A7. Ship container logs to Loki and query them with LogQL 🟢

**Objective:** Run a minimal log pipeline and learn the basic LogQL shapes.

**Tasks**
1. Add `grafana/loki` (single binary) and `grafana/alloy` to Compose.
2. Configure Alloy to discover Docker containers and send their logs to Loki with labels `container` and `compose_service`.
3. Make the A3 app log one JSON line per request with `level`, `route`, `status`, `duration_ms`.
4. Add Loki as a Grafana data source and in Explore run: all logs for the app, only errors (`|= "error"`), JSON-parsed lines with `status >= 500`.
5. Build a panel with `sum by (status) (count_over_time({compose_service="app"} | json [1m]))`.

**Expected outcome:** Logs appear in Explore within seconds; you can filter by content and by parsed fields, and you can explain why `status` is not a Loki label.

<details>
<summary>Hints</summary>

Alloy components: `discovery.docker`, `discovery.relabel` (map `__meta_docker_container_name`), `loki.source.docker`, `loki.write`. Mount `/var/run/docker.sock` read-only. Promtail reached end of life in 2026, so do not start new setups with it.
</details>

**Bonus:** Write a Loki alerting rule (ruler) that fires when the app logs more than 10 errors per minute.

### A8. Synthetic checks with the blackbox exporter 🟢

**Objective:** Monitor endpoints from the outside, including TLS expiry.

**Tasks**
1. Add `prom/blackbox-exporter` to Compose with the default `http_2xx` module.
2. Configure a Prometheus job that probes three URLs (your app, `https://example.com`, a URL that returns 404) using the standard relabeling pattern.
3. Query `probe_success`, `probe_duration_seconds` and `probe_http_status_code`.
4. Write alerts for `probe_success == 0` for 3m and certificate expiry under 21 days.

**Expected outcome:** Two probes succeed, the 404 probe fails, and the TLS expiry query returns days remaining for the HTTPS site.

<details>
<summary>Hints</summary>

The job uses `metrics_path: /probe`, `params: { module: [http_2xx] }`, and relabels `__address__` → `__param_target`, `__param_target` → `instance`, then sets `__address__` to `blackbox-exporter:9115`. Days to expiry: `(probe_ssl_earliest_cert_expiry - time()) / 86400`.
</details>

**Bonus:** Add a `dns_udp` module that checks resolution of your domain against `1.1.1.1`.

### A9. Install kube-prometheus-stack and find your way around 🟢

**Objective:** Install the standard Kubernetes monitoring stack and learn what each component does.

**Tasks**
1. Create a `kind` cluster and install `prometheus-community/kube-prometheus-stack` as release `kps` in namespace `observability`.
2. List the pods and name the role of each (operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter).
3. Port-forward Prometheus and Grafana, log in to Grafana with the admin secret.
4. List the CRDs the chart installed and the `ServiceMonitor`s it created.
5. Find which default targets are `DOWN` on kind and explain why.
6. Open the "Kubernetes / Compute Resources / Namespace (Pods)" dashboard.

**Expected outcome:** You can explain the operator pattern in two sentences and map each `ServiceMonitor` to a target on `/targets`.

<details>
<summary>Hints</summary>

`kubectl -n observability get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d`. `kubectl get crd | grep monitoring.coreos.com`. On kind, kube-scheduler, controller-manager and etcd bind to `127.0.0.1` on the control-plane node, so their targets are down unless you change the bind addresses.
</details>

**Bonus:** Use `helm get values kps -a` and find the values that control `serviceMonitorSelector`.

### A10. CloudWatch alarm on EC2 CPU with SNS 🟢

**Objective:** Use native AWS monitoring for a basic alert.

**Tasks**
1. Launch a `t3.micro` instance (or use an existing test one).
2. Create an SNS topic and subscribe your email.
3. Create a CloudWatch alarm on `CPUUtilization` > 70% for 3 consecutive 1-minute periods (enable detailed monitoring) using the AWS CLI.
4. SSH in and generate load (`stress-ng --cpu 2 --timeout 600`), then confirm the alarm goes to `ALARM` and the email arrives.
5. Stop the load and confirm the `OK` notification.

**Expected outcome:** You receive both ALARM and OK emails, and you can explain periods, evaluation periods and datapoints to alarm.

<details>
<summary>Hints</summary>

`aws cloudwatch put-metric-alarm --alarm-name a10-cpu --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<id> --statistic Average --period 60 --evaluation-periods 3 --threshold 70 --comparison-operator GreaterThanThreshold --alarm-actions <topic-arn> --ok-actions <topic-arn>`. Detailed monitoring: `aws ec2 monitor-instances --instance-ids <id>`.
</details>

**Bonus:** Set `--treat-missing-data breaching` on a second alarm, stop the instance, and explain what happens and when that setting is right.

---

## 🟡 Intermediate

### A11. ServiceMonitor and PrometheusRule for your own app 🟡

**Objective:** Onboard an application to `kube-prometheus-stack` the way platform teams expect.

**Tasks**
1. Containerise the A3 app, load it into kind and deploy it to namespace `a11` with 2 replicas and a Service with a named port `http-metrics`.
2. Write a `ServiceMonitor` that selects the Service and scrapes `/metrics` every 30s.
3. Confirm both pods appear on `/targets`. If they do not, debug the selector chain.
4. Write a `PrometheusRule` with one recording rule (request rate per route) and one alert (error ratio > 5% for 5m).
5. Verify the rule appears on Prometheus' `/rules` page.

**Expected outcome:** Two `UP` targets with `namespace`, `pod` and `service` labels, and your rule group loaded and evaluating.

<details>
<summary>Hints</summary>

The chain is: Prometheus `serviceMonitorSelector` (default `release: kps`) → `ServiceMonitor` labels → `spec.selector` → Service labels → Service `ports[].name` → `endpoints[].port`. Load the image with `kind load docker-image app:dev --name obs`. Rules need the same `release: kps` label unless `ruleSelectorNilUsesHelmValues: false`.
</details>

**Bonus:** Switch to a `PodMonitor` instead and explain when you would choose it over a `ServiceMonitor`.

### A12. Recording rules and correct latency percentiles 🟡

**Objective:** Prove why you aggregate histogram buckets, not quantiles, and speed up dashboards with recording rules.

**Tasks**
1. Run three replicas of the A3 app and make one replica much slower than the others.
2. Compute p99 latency the right way: `histogram_quantile` over `sum by (le)` of bucket rates.
3. Compute it the wrong way: `avg` of per-pod p99 values. Compare the two numbers.
4. Create recording rules following the `level:metric:operations` convention for request rate, error ratio and bucket rates aggregated by `route`.
5. Rebuild the latency panel on the recorded series and compare query time in the query inspector.

**Expected outcome:** A short written explanation of why the averaged quantile is wrong, with both numbers, and a dashboard that queries only recorded series.

<details>
<summary>Hints</summary>

Record `route:http_request_duration_seconds_bucket:rate5m` as `sum by (route, le) (rate(http_request_duration_seconds_bucket[5m]))`. Then `histogram_quantile(0.99, route:http_request_duration_seconds_bucket:rate5m)`. Never drop `le` in the recording rule.
</details>

**Bonus:** Enable native histograms in Prometheus and the client library and compare the accuracy and series count.

### A13. Alertmanager routing tree, inhibition and route tests 🟡

**Objective:** Design routing for multiple teams and severities and prove it with tests.

**Tasks**
1. Write an `alertmanager.yaml` with a default receiver, a `critical` route to a pager receiver, and team routes (`team="payments"`, `team="search"`) to separate Slack channels.
2. Use `continue: true` so critical alerts also go to the team channel.
3. Add an inhibition rule: when `NodeDown` fires, suppress `warning` alerts on the same `instance`.
4. Add a weekday-only time interval for `severity="ticket"` alerts.
5. Test with `amtool config routes test` for at least six label sets and record the expected receiver for each.
6. Deploy it through the chart's `alertmanager.config` values and fire test alerts with `amtool alert add`.

**Expected outcome:** Every test label set reaches exactly the receivers you predicted, and inhibition visibly suppresses the child alert.

<details>
<summary>Hints</summary>

`amtool config routes test --config.file=alertmanager.yaml severity=critical team=payments`. Inhibition: `inhibit_rules: [{ source_matchers: [alertname="NodeDown"], target_matchers: [severity="warning"], equal: [instance] }]`. Time intervals are referenced from routes with `active_time_intervals`.
</details>

**Bonus:** Write a small CI script that runs all route tests from a YAML table and fails on any mismatch.

### A14. Dashboards as code with variables and the Grafana sidecar 🟡

**Objective:** Ship a reusable service dashboard with the application, not by clicking.

**Tasks**
1. Build a RED dashboard with `namespace`, `service` and `route` variables (chained so each one filters the next).
2. Use `$__rate_interval` in every rate and set a stable `uid`.
3. Export the JSON, remove `id`, and wrap it in a ConfigMap labelled `grafana_dashboard: "1"` with a `grafana_folder` annotation.
4. Deploy the ConfigMap with the app's manifests and confirm the dashboard appears without touching the UI.
5. Add a deployment annotation query that marks new versions from `kube_deployment_status_observed_generation` changes or `changes(build_info[5m])`.

**Expected outcome:** Deleting and recreating the ConfigMap removes and restores the dashboard, and the variables work for any onboarded service.

<details>
<summary>Hints</summary>

Variable query: `label_values(http_requests_total{namespace="$namespace"}, service)`. In `kube-prometheus-stack` values, set `grafana.sidecar.dashboards.searchNamespace: ALL` and `folderAnnotation: grafana_folder`.
</details>

**Bonus:** Generate the same dashboard with the Grafana Foundation SDK or Grafonnet and diff the output with your exported JSON.

### A15. Fluent Bit to Loki on Kubernetes with LogQL metrics 🟡

**Objective:** Build a cluster log pipeline with sane labels and turn logs into alerts.

**Tasks**
1. Install Loki (single binary) and Fluent Bit (DaemonSet) with Helm.
2. Configure Fluent Bit with the `kubernetes` filter and the `loki` output, using only `namespace`, `app`, `container` and `level` as labels.
3. Make sure JSON app logs are parsed so `level` is available.
4. Write LogQL for: error rate per app, top 5 noisiest pods by bytes, and p95 `duration_ms` per route using `unwrap`.
5. Create a Loki ruler alert for error rate and route it to Alertmanager.
6. Check the number of active streams and explain what would happen if you added `pod` or `trace_id` as a label.

**Expected outcome:** Logs flow with exactly the planned labels, the LogQL metrics work in a panel, and the ruler alert reaches Alertmanager.

<details>
<summary>Hints</summary>

Fluent Bit `loki` output: `labels job=fluent-bit, namespace=$kubernetes['namespace_name'], app=$kubernetes['labels']['app']`. Use `Merge_Log On` in the kubernetes filter to lift JSON fields. Loki ruler needs `ruler.alertmanager_url`. Streams: `loki_ingester_memory_streams`.
</details>

**Bonus:** Drop health-check log lines in Fluent Bit with a `grep` filter and measure the reduction with `bytes_over_time`.

### A16. Distributed tracing with OpenTelemetry and Tempo 🟡

**Objective:** Trace a request across two services through an OTel Collector.

**Tasks**
1. Create a second service (`inventory`) that `checkout` calls over HTTP.
2. Instrument both with the OpenTelemetry SDK (auto-instrumentation for HTTP plus one manual span named `reserve-stock`).
3. Deploy an OpenTelemetry Collector with `otlp` receiver, `memory_limiter` and `batch` processors, and an exporter to Tempo.
4. Set `service.name` and `deployment.environment` resource attributes.
5. Find a trace in Grafana that shows both services and the manual span; identify the slowest span.
6. Break propagation (stop forwarding headers) and observe two disconnected traces.

**Expected outcome:** One trace spans both services with correct parent-child links; you can explain the role of the `traceparent` header.

<details>
<summary>Hints</summary>

SDK env vars: `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317`, `OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=1.0`. Collector order: `memory_limiter` first, `batch` last.
</details>

**Bonus:** Install the OpenTelemetry Operator and use an `Instrumentation` resource with the pod annotation to inject auto-instrumentation without code changes.

### A17. Correlate metrics, logs and traces 🟡

**Objective:** Make one click take you from a latency spike to the trace and its logs.

**Tasks**
1. Add the current `trace_id` to every JSON log line in both services from A16.
2. Configure a Loki derived field in Grafana that turns `trace_id` into a link to Tempo.
3. Configure the Tempo data source "trace to logs" to query Loki by `service.name` and trace ID.
4. Record exemplars on the latency histogram, enable `exemplar-storage` in Prometheus, and show exemplars on the p99 panel.
5. Walk the path: p99 spike → exemplar → trace → logs for that trace.

**Expected outcome:** The full metric → trace → log path works with clicks only, and you can screenshot each hop.

<details>
<summary>Hints</summary>

Derived field regex: `"trace_id":"(\w+)"`. Prometheus feature flag: `prometheus.prometheusSpec.enableFeatures: [exemplar-storage]`. Exemplars require the OpenMetrics exposition format, which client libraries serve when Prometheus asks for it.
</details>

**Bonus:** Generate RED metrics from spans with the Collector's `spanmetrics` connector and compare them with the app's own metrics.

### A18. Unit-test alert rules with promtool 🟡

**Objective:** Test alert logic the same way you test code.

**Tasks**
1. Take the A11 error-ratio alert and the A5 `InstanceDown` alert into a plain rules file.
2. Write `tests.yml` with `input_series` that cross the threshold after 10 minutes.
3. Assert the alert is not firing at 4 minutes (pending), and is firing at 15 minutes with the expected labels and annotations.
4. Add a test proving the alert does not fire when traffic is zero (no division-by-zero false positives).
5. Run `promtool test rules tests.yml` and make a test fail on purpose to read the output.

**Expected outcome:** All tests pass, including the edge case, and a deliberate threshold change makes the suite fail.

<details>
<summary>Hints</summary>

Series notation: `'0+60x20'` grows by 60 per minute for 20 steps; `'0x20'` stays flat. Use `exp_alerts: []` to assert nothing fires. Annotations must match exactly, including templated values.
</details>

**Bonus:** Run `pint lint` on the same rules and fix every finding.

### A19. Find and fix a cardinality explosion 🟡

**Objective:** Experience a cardinality problem and fix it at the right layer.

**Tasks**
1. Add a label `user_id` with random values to a counter in the A3 app and run load for 10 minutes.
2. Watch `prometheus_tsdb_head_series` and Prometheus memory grow.
3. Find the offender with the `/tsdb-status` page and `topk(10, count by (__name__)({__name__=~".+"}))`.
4. Drop the label at scrape time with `metricRelabelings` (`labeldrop`) in the `ServiceMonitor` and confirm series stop growing.
5. Fix the code properly, and set `sampleLimit` on the `ServiceMonitor` as a guard rail.
6. Calculate how many series one high-cardinality label would add at 10,000 users × 20 routes × 5 statuses.

**Expected outcome:** A before/after graph of head series and a short explanation of why the fix belongs in code, with relabeling as a safety net.

<details>
<summary>Hints</summary>

`metricRelabelings: [{ action: labeldrop, regex: user_id }]`. Dropping a label can create duplicate series if it was the only differentiator; check for `duplicate sample for timestamp` warnings. `scrape_samples_scraped` per target shows which target got bigger.
</details>

**Bonus:** Alert on `scrape_samples_scraped` growing more than 50% compared with a day ago using `offset 1d`.

### A20. CloudWatch Agent, Logs Insights and metric filters 🟡

**Objective:** Collect OS metrics and logs on EC2 and alert from logs.

**Tasks**
1. Attach an IAM role with `CloudWatchAgentServerPolicy` to an instance and install the CloudWatch agent.
2. Configure it to send memory and disk usage metrics and `/var/log/nginx/access.log` to a log group with 7-day retention.
3. Write a Logs Insights query for the count of 5xx responses per 5 minutes.
4. Create a metric filter that counts 5xx lines into a custom metric and an alarm on it.
5. Add the CloudWatch data source to Grafana and put the custom metric next to a Prometheus panel.

**Expected outcome:** Memory and disk metrics appear under the `CWAgent` namespace, the Logs Insights query returns counts, and the alarm fires when you generate 5xx responses.

<details>
<summary>Hints</summary>

Agent config: `metrics.metrics_collected.mem.measurement: ["mem_used_percent"]`, `logs.logs_collected.files.collect_list` with `log_group_name` and `retention_in_days: 7`. Logs Insights: `fields @timestamp, @message | filter @message like / 5\d\d / | stats count() by bin(5m)`. Put the metric filter with `aws logs put-metric-filter`.
</details>

**Bonus:** Replace the metric filter with Embedded Metric Format (EMF) output from a small script and compare cost and flexibility.

---

## 🔴 Advanced

### A21. SLOs with multi-window multi-burn-rate alerts 🔴

**Objective:** Define an SLO and alert on error budget burn, then prove the alerts behave.

**Tasks**
1. Define two SLOs for `checkout`: 99.5% availability (non-5xx) and 99% of requests under 300 ms, over 30 days.
2. Write recording rules for error ratios over 5m, 30m, 1h, 2h, 6h, 1d and 3d.
3. Write the two standard page alerts (14.4× over 1h and 5m, 6× over 6h and 30m) and the ticket alerts (3× over 1d and 2h, 1× over 3d and 6h).
4. Build an error budget dashboard: SLI, budget remaining, burn rate per window.
5. Inject failures at 2%, then 10%, and record how long each alert takes to fire and to reset.
6. Generate the same rules with Sloth or Pyrra and diff them against yours.

**Expected outcome:** A written table of detection time and reset time per failure level, matching the theory within a few minutes.

<details>
<summary>Hints</summary>

Burn rate = error ratio / (1 − SLO). For 99.5%, 14.4× means an error ratio above 7.2%. Latency SLI from histograms: `1 - (rate of bucket{le="0.3"} / rate of count)`; make sure 0.3 is an actual bucket boundary. Use `promtool test rules` to check the windows before running live.
</details>

**Bonus:** Write an error budget policy (one page) describing what the team does at 50%, 25% and 0% budget remaining.

### A22. Highly available Prometheus with long-term storage 🔴

**Objective:** Run an HA pair with deduplicated, long-term queries.

**Tasks**
1. Set `replicas: 2` on Prometheus and confirm both scrape the same targets.
2. Run MinIO in the cluster as object storage.
3. Option A: enable the Thanos sidecar (`thanos.objectStorageConfig`), deploy Thanos Query and Store Gateway, and set `--query.replica-label=prometheus_replica`. Option B: deploy Grafana Mimir and `remote_write` from both replicas with HA deduplication on `cluster` and `__replica__`.
4. Point Grafana at the global query layer and confirm results are not doubled.
5. Delete one Prometheus pod during load and show there is no gap in the global view.
6. Query data older than the local retention to prove blocks come from object storage.

**Expected outcome:** No duplicate series and no gap during a replica restart, and queries beyond local retention succeed.

<details>
<summary>Hints</summary>

The operator adds a `prometheus_replica` external label automatically; that is what deduplication keys on. For Mimir HA tracking, set `accept_ha_samples: true` and configure `ha_cluster_label`/`ha_replica_label` in limits. Shorten local retention to 2h to make step 6 quick.
</details>

**Bonus:** Turn on compaction with downsampling (Thanos) or per-tenant retention (Mimir) and explain the cost impact.

### A23. Tail sampling with a load-balanced Collector tier 🔴

**Objective:** Keep every error and slow trace while sampling the rest, at scale.

**Tasks**
1. Deploy two Collector tiers: an agent tier with the `loadbalancing` exporter routing by `traceID`, and a gateway tier (3 replicas, headless Service) with the `tail_sampling` processor.
2. Configure policies: keep all traces with an error status, keep traces slower than 500 ms, keep 5% of the rest.
3. Generate traffic with a mix of fast, slow and failing requests.
4. Prove that every error trace is complete in Tempo, while normal traces are sampled near 5%.
5. Scale the gateway to 1 replica, repeat, and show the broken traces you get without load balancing by trace ID.
6. Monitor the Collectors' own metrics (`otelcol_processor_tail_sampling_*`, refused spans, queue size).

**Expected outcome:** Measured keep rates per category that match the policies, and an explanation of why consistent routing by trace ID is mandatory.

<details>
<summary>Hints</summary>

`loadbalancing` exporter: `routing_key: traceID`, `resolver: { dns: { hostname: otel-gateway-headless.observability.svc.cluster.local } }`. Tail sampling needs `decision_wait` long enough for the slowest trace (for example 10s) and enough memory for `num_traces`.
</details>

**Bonus:** Generate span metrics before sampling with the `spanmetrics` connector so RED metrics stay accurate at 100%.

### A24. Multi-cluster observability with a central store 🔴

**Objective:** Give one Grafana a global view across clusters while keeping alerting local.

**Tasks**
1. Create three kind clusters: `central`, `east`, `west`.
2. In `central`, install Mimir (or Thanos Receive) and Grafana; expose the write endpoint to the other clusters.
3. In `east` and `west`, install `kube-prometheus-stack` with `externalLabels.cluster` set and `remote_write` to central; keep local Alertmanager and rules.
4. Build a global dashboard with a `cluster` variable and a panel comparing request rates across clusters.
5. Write one global alert in the central ruler (for example "a cluster stopped sending data") using `absent` per cluster.
6. Cut the network from `west` to central and describe what still works locally and what the central view shows.

**Expected outcome:** A working global view, local alerts that keep firing during a central outage, and a central alert that detects the missing cluster.

<details>
<summary>Hints</summary>

kind clusters can reach each other via the shared `kind` Docker network using container IPs and a NodePort. `remote_write` queue metrics (`prometheus_remote_storage_samples_pending`) show backlog during the outage. Prometheus retries and catches up from the WAL if the outage is shorter than about 2 hours.
</details>

**Bonus:** Switch `east` to Prometheus agent mode and list what you lose locally.

### A25. Observability-as-code pipeline and a game day 🔴

**Objective:** Put the whole monitoring configuration behind CI and prove it works under a simulated incident.

**Tasks**
1. Create a repo with folders for `rules/`, `alertmanager/`, `dashboards/`, `servicemonitors/` and `otel/`.
2. Add a CI pipeline (GitHub Actions or GitLab CI) that runs `promtool check rules`, `promtool test rules`, `amtool check-config`, route tests, `pint`, a JSON lint for dashboards, and `otelcol validate --config` for the Collector config.
3. Deploy the repo to the cluster with Argo CD or Flux.
4. Plan a game day with three injected failures: a dependency returning 500s, a slow database (latency), and Prometheus losing a target.
5. Run it with a teammate as on-call; measure time to detect, time to find the cause, and whether each alert had a working runbook.
6. Write a short postmortem and open pull requests for every alert or dashboard gap found.

**Expected outcome:** A green pipeline that blocks a broken rule, a GitOps-deployed stack, and a postmortem with concrete fixes merged through the same pipeline.

<details>
<summary>Hints</summary>

Extract rules from `PrometheusRule` YAML for promtool with `yq '.spec' rule.yaml > /tmp/rule.yml`. Use the `prom/prometheus` and `prom/alertmanager` images in CI to get `promtool` and `amtool`. Fault injection: a feature flag in the app, `tc qdisc add dev eth0 root netem delay 300ms` in a debug container, or Chaos Mesh.
</details>

**Bonus:** Add a weekly job that reports alerts that fired but were never acknowledged, and alerts with no firing in 90 days, as candidates for deletion.

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
