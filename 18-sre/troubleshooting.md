# SRE & Reliability — Troubleshooting Drills

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

> Real SRE & Reliability errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Prometheus SLIs & Recording Rules](#prometheus-slis--recording-rules) — Q1–Q20
- [Burn-Rate Alerts & Alertmanager](#burn-rate-alerts--alertmanager) — Q21–Q37
- [Grafana SLO Dashboards](#grafana-slo-dashboards) — Q38–Q46
- [k6 Load Testing](#k6-load-testing) — Q47–Q61
- [Chaos Mesh, Litmus & AWS FIS](#chaos-mesh-litmus--aws-fis) — Q62–Q76
- [Reliability Patterns in Production](#reliability-patterns-in-production) — Q77–Q90
- [Backups, DR & Runbook Automation](#backups-dr--runbook-automation) — Q91–Q100

---

## Prometheus SLIs & Recording Rules

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You applied a `PrometheusRule` for your SLO, but `slo:sli_error:ratio_rate5m` returns nothing in Prometheus. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Prometheus Operator is not selecting the rule (label mismatch with `ruleSelector`), the rule is in a namespace outside `ruleNamespaceSelector`, or the expression itself returns no data.

**Diagnose →**
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}'
kubectl get prometheusrule -A --show-labels | grep checkout
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].name'
```

**Fix →** Add the label the selector expects (kube-prometheus-stack defaults to `release: <helm-release-name>`), or install the chart with `prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false` so all rules are picked up. If the group is loaded but empty, run the raw expression in the Prometheus UI and fix the selectors.

**Prevent →** Keep a CI check that the rule group appears in `/api/v1/rules` after deploy, and standardize rule labels in a template.

</details>

### Q2. `promtool check rules` fails with `could not parse expression: 1:72: parse error: unexpected right parenthesis ')'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unbalanced parentheses or braces in a multi-line PromQL expression, often after editing a long burn-rate `and`/`or` expression, or YAML folding (`>`) joining lines in an unexpected way.

**Diagnose →**
```bash
promtool check rules slo-rules.yaml
promtool query instant http://localhost:9090 '<paste the expression>'
```
The `1:72` is line:column inside the expression, not the YAML file.

**Fix →** Use the literal block scalar `|` for multi-line expressions, align parentheses one level per line, and test the expression in the Prometheus UI first.

**Prevent →** Run `promtool check rules` and `promtool test rules` in CI on every change; generate SLO rules with Sloth or Pyrra instead of hand-editing.

</details>

### Q3. The availability SLI panel shows `NaN` every night between 02:00 and 05:00. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Zero traffic in the window: `0 / 0` is `NaN` in PromQL. Alerts comparing `NaN > threshold` are silently false.

**Diagnose →**
```promql
sum(rate(http_requests_total{job="checkout"}[5m]))
```
If it is 0 during those hours, the ratio is undefined.

**Fix →** For dashboards, display "no traffic" by using longer windows or wrapping with `or vector(0)` only for the error side. For alerting, keep `NaN` (no traffic means no failed users) but add a traffic-drop alert and synthetic probes so a total outage at night is still detected.

**Prevent →** Every SLO gets a companion traffic-drop or `absent()` alert, and synthetic checks for low-traffic periods.

</details>

### Q4. Your per-route SLI query fails with `found duplicate series for the match group {route="/api/checkout"} on the right hand-side of the operation`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Division with one-to-one matching where the right side has several series per `route` (for example still split by `instance` or `code`) because it was not aggregated to the same labels as the left side.

**Diagnose →** Run each side alone and compare the label sets:
```promql
sum by (route) (rate(http_requests_total{job="checkout",code=~"5.."}[5m]))
rate(http_requests_total{job="checkout"}[5m])
```

**Fix →** Aggregate both sides to identical labels:
```promql
sum by (route) (rate(http_requests_total{job="checkout",code=~"5.."}[5m]))
/
sum by (route) (rate(http_requests_total{job="checkout"}[5m]))
```

**Prevent →** In recording rules always use explicit `sum by (...)` on numerator and denominator with the same label list.

</details>

### Q5. Prometheus logs `vector contains metrics with the same labelset after applying rule labels` for your SLI recording rule. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The rule's `labels:` block overwrites a label that distinguishes series in the result. For example, the expression returns one series per `code`, and the rule sets `code: "all"`, so all results collapse to the same labelset.

**Diagnose →**
```bash
kubectl logs -n monitoring prometheus-monitoring-kube-prometheus-prometheus-0 -c prometheus | grep "same labelset"
```
Run the expression without the rule labels and inspect which labels vary.

**Fix →** Aggregate away the varying label in the expression (`sum without (code) (...)`) or choose a rule label name that does not clash.

**Prevent →** Make SLI recording rules return exactly one series per SLO (or per explicit dimension) and unit test them with `promtool test rules`.

</details>

### Q6. Your latency SLI using `le="0.3"` returns an empty result, but the histogram exists. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** 0.3 seconds is not a bucket boundary. The histogram may only have `0.25` and `0.5` buckets, or the `le` value is formatted differently (`"0.30"`, or `"1.0"` vs `"1"` for whole numbers in some client libraries).

**Diagnose →**
```promql
count by (le) (http_request_duration_seconds_bucket{job="checkout"})
```

**Fix →** Either add a 0.3 bucket in the instrumentation and redeploy, or align the SLO threshold with an existing bucket (for example 250 ms). Match the exact `le` string.

**Prevent →** Define histogram buckets together with SLO thresholds; document the bucket layout in the SLO document. Native histograms in recent Prometheus versions remove most of this pain.

</details>

### Q7. The SLI ratio is above 1.0 (or the latency "good" ratio is 1.07). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Numerator and denominator select different series: the good-events query includes a job, route, or instance that the total does not, or they come from different metrics with different label filters.

**Diagnose →** Compare label selectors character by character, and check counts:
```promql
count(http_request_duration_seconds_bucket{le="0.3"})
count(http_request_duration_seconds_count)
```

**Fix →** Use identical selectors on both sides, and for latency use `_bucket{le=X}` and `_count` of the same histogram.

**Prevent →** Generate rules from one SLO spec (Sloth/Pyrra) and add a `promtool` unit test asserting the ratio stays within 0 to 1.

</details>

### Q8. Errors clearly appear in the logs, but the SLI shows zero 5xx for low-traffic routes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Labeled counters are created lazily: the `code="500"` series appears only at the first error. `rate()` needs at least two samples, so the first increment of a new series is invisible, and on low-traffic routes most errors are "first errors" after each pod restart.

**Diagnose →**
```promql
http_requests_total{job="checkout",code="500"}
```
Check whether the series appears only after errors and with value 1.

**Fix →** Pre-initialize expected label combinations at startup (for example `REQS.labels(route, "500")` in Python, or `.WithLabelValues(...)` in Go), so the series exists at 0 before any error.

**Prevent →** Make counter initialization part of the instrumentation library and the service template.

</details>

### Q9. A Grafana panel using `rate(http_requests_total[30d])` fails with `query processing would load too many samples into memory in query execution`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A raw 30-day range over many series exceeds Prometheus' `--query.max-samples` limit (default 50,000,000).

**Diagnose →** Count series behind the selector and estimate samples: series x (30d / scrape interval).
```promql
count(http_requests_total{job="checkout"})
```

**Fix →** Pre-aggregate with recording rules (record 5m rates per SLO, then compute longer windows from those, in a slow-evaluated group), drop unneeded labels, or use a long-term store (Thanos, Mimir) with downsampling.

**Prevent →** Dashboards and alerts for SLOs read only recording rules, never raw high-cardinality metrics over long ranges.

</details>

### Q10. `prometheus_rule_group_iterations_missed_total` keeps increasing for the SLO rule group. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The group takes longer to evaluate than its interval, usually because long-window rules (`[1d]`, `[3d]`, `[30d]`) over raw metrics run every 30 seconds.

**Diagnose →**
```promql
prometheus_rule_group_last_duration_seconds{rule_group=~".*slo.*"}
prometheus_rule_group_interval_seconds{rule_group=~".*slo.*"}
```

**Fix →** Split rules into groups by window: short windows (5m to 1h) at 30s, long windows (6h to 30d) in a separate group with `interval: 5m`. Build long windows from pre-recorded short-window sums where possible.

**Prevent →** Alert on missed iterations and on group duration above 50% of interval.

</details>

### Q11. Your new ServiceMonitor is applied, but the service never appears under Status → Targets. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ServiceMonitor not selected by the Prometheus resource (labels or namespace selector), its `selector.matchLabels` does not match the Service labels, or `endpoints.port` refers to a port name the Service does not define.

**Diagnose →**
```bash
kubectl get servicemonitor -n shop checkout -o yaml | yq '.spec'
kubectl get svc -n shop checkout -o yaml | yq '.metadata.labels, .spec.ports'
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
```

**Fix →** Match labels exactly, use the named Service port (`port: http`), and add the release label or set `serviceMonitorSelectorNilUsesHelmValues=false`.

**Prevent →** Ship a ServiceMonitor in the service Helm chart template, tested in CI against a kind cluster.

</details>

### Q12. The target shows DOWN with `server returned HTTP status 404 Not Found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong metrics path (the app serves `/actuator/prometheus` or `/stats/prometheus`, not `/metrics`), wrong port (scraping the app port instead of a separate metrics port), or metrics disabled in this build.

**Diagnose →**
```bash
kubectl -n shop port-forward svc/checkout 8080:8080 &
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/metrics
```

**Fix →** Set `path:` and `port:` correctly in the ServiceMonitor endpoint.

**Prevent →** Standardize the metrics path and port across services in the platform template.

</details>

### Q13. Scrapes fail intermittently with `context deadline exceeded`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `/metrics` endpoint takes longer than `scrapeTimeout` (default 10s): huge cardinality, expensive collectors, or a CPU-throttled pod.

**Diagnose →**
```bash
time curl -s localhost:8080/metrics | wc -l
```
```promql
scrape_duration_seconds{job="checkout"}
scrape_samples_scraped{job="checkout"}
```

**Fix →** Reduce cardinality (drop per-user or per-URL labels), raise CPU limits, or increase `scrapeTimeout` modestly (must stay below the interval).

**Prevent →** Alert on `scrape_duration_seconds` approaching the timeout and on sample count growth.

</details>

### Q14. Prometheus got OOMKilled after a developer added a `user_id` label to `http_requests_total`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cardinality explosion: one series per user x route x code x pod.

**Diagnose →**
```promql
topk(10, count by (__name__) ({__name__=~".+"}))
count(count by (user_id) (http_requests_total))
```
Also check `prometheus_tsdb_head_series` before and after the deploy.

**Fix →** Drop the label at scrape time while the code is fixed:
```yaml
metricRelabelings:
  - action: labeldrop
    regex: user_id
```
Then remove it from the instrumentation; per-user data belongs in logs or traces.

**Prevent →** Set `sampleLimit` on ServiceMonitors, alert on head series growth, and review metric labels in code review.

</details>

### Q15. After migrating from ingress-nginx to Gateway API with Envoy, the SLO dashboard is blank and no burn alerts fire. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The recording rules reference `nginx_ingress_controller_requests`, which no longer exists. Empty results mean `NaN`/no data, so alerts silently never fire.

**Diagnose →**
```promql
absent(nginx_ingress_controller_requests)
count by (__name__) ({__name__=~"envoy_.*rq.*"})
```

**Fix →** Rewrite the SLI recording rules for the new metric names (for example Envoy's upstream request counters by response code class), validate against a week of overlap data, then switch.

**Prevent →** Add `absent()` alerts on every metric an SLO depends on, so a renamed or missing metric pages as "SLO blind".

</details>

### Q16. A 1m-window recording rule returns empty results intermittently. Scrape interval is 30s. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `rate()` needs at least two samples in the window. With a 30s scrape interval and a 1m window, any missed or delayed scrape leaves only one sample.

**Diagnose →**
```promql
count_over_time(http_requests_total{job="checkout"}[1m])
```

**Fix →** Use windows of at least 4x the scrape interval (2m for 30s scrapes); that is why the shortest SLO window is usually 5m. In Grafana use `$__rate_interval`.

**Prevent →** Lint recording rules for windows shorter than 4x scrape interval.

</details>

### Q17. Joining SLI series with `kube_pod_labels` to add a `team` label fails with `many-to-many matching not allowed: matching labels must be unique on one side`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both sides have multiple series per matching key: `kube_pod_labels` can have duplicates (several kube-state-metrics replicas, or pods recreated with the same name), and the SLI side is not unique either.

**Diagnose →**
```promql
count by (namespace, pod) (kube_pod_labels) > 1
```

**Fix →** Make one side unique with aggregation and use `group_left`:
```promql
sum by (namespace, pod) (rate(http_requests_total[5m]))
* on (namespace, pod) group_left(label_team)
max by (namespace, pod, label_team) (kube_pod_labels)
```

**Prevent →** Prefer static `team` labels via ServiceMonitor relabeling or `podTargetLabels` instead of runtime joins in SLO rules.

</details>

### Q18. In Thanos, the request counts used for error-budget events are exactly double what the app logs show. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two Prometheus HA replicas scrape the same targets, and the query runs without deduplication, so `sum()` counts each series twice. Ratios look right (both sides doubled), but absolute budget numbers are wrong.

**Diagnose →**
```promql
count by (prometheus_replica) (http_requests_total{job="checkout"})
```

**Fix →** Enable deduplication in Thanos Query with `--query.replica-label=prometheus_replica` (and the dedup toggle in Grafana's data source), or aggregate `max by` replica before summing.

**Prevent →** Configure the replica label consistently and verify event counts against a second source when building SLO reports.

</details>

### Q19. `promtool test rules` reports `FAILED: alertname: CheckoutErrorBudgetBurnFast, time: 10m, exp:[0: Labels:{alertname="CheckoutErrorBudgetBurnFast", severity="page"}], got:[]`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** At `eval_time: 10m` the alert is not yet firing: the input series do not have enough history for the `[1h]` window to exceed the threshold, the `for:` duration has not elapsed, or the expected labels miss labels the rule adds (like `slo`).

**Diagnose →** Evaluate the recording rules at that time in the test with `promql_expr_test`, and print what the alert produces at later times.

**Fix →** Move `eval_time` later (after the long window is above threshold plus `for:`), increase the error ratio in `input_series`, and list every label in `exp_labels`, including ones from recording rules.

**Prevent →** Keep a positive and a negative test case per alert tier, and document expected detection time in the test.

</details>

### Q20. Prometheus refuses to load the rules file: `invalid recording rule name: slo-checkout-error-ratio-5m`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Recording rule names must be valid metric names: letters, digits, underscores and colons only. Hyphens are not allowed.

**Diagnose →**
```bash
promtool check rules slo-rules.yaml
```

**Fix →** Rename following the `level:metric:operations` convention: `slo:sli_error:ratio_rate5m`.

**Prevent →** Lint in CI with `promtool check rules`, and document the naming convention.

</details>

## Burn-Rate Alerts & Alertmanager

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. The burn-rate alert sits in `Pending` for the whole incident and never reaches `Firing`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `for:` duration is longer than the condition stays true continuously. If the short-window ratio dips below threshold for even one evaluation, the pending timer resets. A long `for:` (15m) on a 5m short window makes this likely during spiky errors.

**Diagnose →**
```promql
ALERTS{alertname="CheckoutErrorBudgetBurnFast"}
slo:sli_error:ratio_rate5m{slo="checkout-availability"}
```
Graph the short-window ratio against the threshold.

**Fix →** Use `for: 2m` (or less) on paging burn alerts; the long window already provides the noise filtering. Optionally use `keep_firing_for` (Prometheus 2.42+) to avoid flapping once firing.

**Prevent →** Unit test alerts with spiky input series in `promtool test rules`.

</details>

### Q22. After a rollback fixed the incident, the page kept firing for another 50 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The alert uses only the 1h window. The 1h ratio stays above threshold long after recovery because it still contains the bad minutes.

**Diagnose →** Read the alert expression: if there is no short-window `and` condition, this is expected behaviour.

**Fix →** Add the short window (1/12 of the long one):
```promql
slo:sli_error:ratio_rate1h > 14.4 * 0.001
and
slo:sli_error:ratio_rate5m > 14.4 * 0.001
```

**Prevent →** Use the full multi-window, multi-burn-rate pattern, ideally generated by Sloth or Pyrra.

</details>

### Q23. Checkout was completely down for 25 minutes and no burn-rate alert fired. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Requests never reached the service, so app metrics recorded zero traffic and the ratio was `NaN` (pods crashlooping, ingress misrouted, DNS broken). Or the failures were load balancer 502/503s that the app SLI does not see.

**Diagnose →**
```promql
sum(rate(http_requests_total{job="checkout"}[5m]))
sum by (status) (rate(nginx_ingress_controller_requests{service="checkout"}[5m]))
up{job="checkout"}
```

**Fix →** Measure the SLI at the ingress or load balancer, add a traffic-drop alert, `absent(up{job="checkout"} == 1)`, and external synthetic probes.

**Prevent →** For every SLO, review "what failure would make this SLI go blind?" and cover each case.

</details>

### Q24. The alert shows `Firing` in Prometheus, but Alertmanager's UI shows nothing. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus is not configured to send to this Alertmanager (wrong `alerting.alertmanagers` config or service discovery), network policy blocks it, or Alertmanager is down.

**Diagnose →**
```bash
curl -s localhost:9090/api/v1/alertmanagers | jq '.data.activeAlertmanagers'
```
```promql
rate(prometheus_notifications_errors_total[5m])
prometheus_notifications_dropped_total
```

**Fix →** Fix the Alertmanager endpoints in the Prometheus config (in the operator: `spec.alerting.alertmanagers`), and allow traffic on port 9093.

**Prevent →** Alert on `prometheus_notifications_errors_total` and use a Watchdog heartbeat to an external dead man's switch.

</details>

### Q25. Alertmanager won't load the new config: `undefined receiver "pager" used in route`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A route references a receiver name that is not defined in `receivers:` (typo or the receiver block was removed).

**Diagnose →**
```bash
amtool check-config alertmanager.yaml
kubectl logs -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 -c alertmanager | tail
```

**Fix →** Add the receiver or fix the name:
```yaml
receivers:
  - name: "null"
  - name: "pager"
```

**Prevent →** Run `amtool check-config` in CI before deploying Alertmanager config.

</details>

### Q26. Alertmanager logs `Notify for alerts failed` with `unexpected status code 404: no_service` for the Slack receiver. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Slack incoming webhook URL was revoked or the app was removed from the workspace.

**Diagnose →**
```bash
kubectl logs -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 -c alertmanager | grep -i slack
curl -s -X POST -H 'Content-type: application/json' --data '{"text":"test"}' "$SLACK_WEBHOOK_URL"
```
```promql
rate(alertmanager_notifications_failed_total{integration="slack"}[5m])
```

**Fix →** Create a new webhook, store it in a Secret, and reference it with `api_url_file` or the operator's `AlertmanagerConfig` secret reference.

**Prevent →** Alert on `alertmanager_notifications_failed_total` through a different channel than the one being monitored.

</details>

### Q27. The same burn-rate incident re-pages the on-call every 5 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `repeat_interval` too short, or the alert's labels change (for example `{{ $value }}` placed in labels instead of annotations), creating a "new" alert each evaluation. Two tier alerts with different names but no grouping also produce separate pages.

**Diagnose →**
```bash
amtool --alertmanager.url=http://localhost:9093 alert query alertname=~"Checkout.*" -o extended
```
Check whether label values change between notifications.

**Fix →** Keep dynamic values in annotations only, group by `[alertname, slo]` or just `[slo]`, set `group_interval: 5m` and `repeat_interval: 4h` for pages.

**Prevent →** Lint rules to forbid template expressions in labels.

</details>

### Q28. A burn-rate alert flaps between firing and resolved several times per hour during a partial outage. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The error ratio hovers right around the threshold, so the short-window condition toggles. Each resolve-then-fire sends new notifications.

**Diagnose →** Graph both window ratios with the threshold line during the incident.

**Fix →** Add `keep_firing_for: 10m` to the rule (Prometheus 2.42+), which keeps the alert firing for a period after the condition clears. Make sure Alertmanager `group_interval` absorbs short resolve/fire cycles.

**Prevent →** Include hovering-around-threshold input series in `promtool` tests.

</details>

### Q29. You created a silence for a maintenance window, but the alert still pages. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Matchers do not match the alert's actual labels (typo, wrong label name, exact match where a regex was needed), the silence expired, or it was created on one Alertmanager that is not clustered with the one sending.

**Diagnose →**
```bash
amtool --alertmanager.url=http://localhost:9093 silence query
amtool --alertmanager.url=http://localhost:9093 alert query --silenced
```
Compare the silence matchers with the alert labels.

**Fix →** Recreate with correct matchers, for example `amtool silence add 'alertname=~"Checkout.*"' namespace=shop --duration=2h --comment="maintenance"`.

**Prevent →** Ensure Alertmanager replicas are clustered and use maintenance tooling that creates silences via the API with tested matchers.

</details>

### Q30. An inhibition rule should mute service alerts when the database is down, but both alerts arrive. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `equal` labels are not present (or differ) on both the source and target alert. If `equal: [cluster]` is set and the service alerts lack a `cluster` label, inhibition never applies. Matchers may also not match.

**Diagnose →**
```bash
amtool --alertmanager.url=http://localhost:9093 alert query -o extended
```
Compare labels on `PostgresDown` and the service alert.

**Fix →** Ensure both have the `equal` labels (add `external_labels: {cluster: prod}` in Prometheus, or relabel), and verify the matchers.

**Prevent →** Test inhibition with synthetic alerts posted to Alertmanager's API in staging.

</details>

### Q31. Every alert is delivered twice to PagerDuty from your 3-replica Alertmanager. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The replicas are not clustered (gossip not working), so each sends independently; or Prometheus sends to a load-balanced Service instead of all instances, which is fine only when clustering works.

**Diagnose →**
```promql
alertmanager_cluster_members
alertmanager_cluster_failed_peers
```
Each replica should report 3 members.

**Fix →** Fix `--cluster.peer` settings and open port 9094 TCP/UDP between replicas (network policies). With the Prometheus Operator, ensure the Alertmanager pods can reach each other's mesh port.

**Prevent →** Alert when `alertmanager_cluster_members` is lower than the replica count.

</details>

### Q32. Your external dead man's switch fires: no Watchdog heartbeat for 10 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something in the alerting pipeline broke: Prometheus down or not evaluating rules, Alertmanager down, the route for `Watchdog` changed, or egress to the heartbeat service is blocked.

**Diagnose →**
```bash
kubectl get pods -n monitoring
curl -s localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname=="Watchdog")'
amtool --alertmanager.url=http://localhost:9093 alert query alertname=Watchdog
```

**Fix →** Restore the failing component; until then, treat the cluster as unmonitored and watch dashboards manually or declare an incident.

**Prevent →** Keep the Watchdog route at the top of the routing tree with `continue: false`, and monitor the monitoring stack from a separate system.

</details>

### Q33. The fast-burn alert expression `slo:sli_error:ratio_rate1h > X and slo:sli_error:ratio_rate5m > X` never returns results, even when both series exceed X. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `and` matches on all labels. The 1h and 5m recording rules have different label sets (one has an extra `route` label or a different `slo` value), so no series pair matches.

**Diagnose →** Query both series and compare full label sets.

**Fix →** Make all window recording rules produce the same labels, or match explicitly:
```promql
slo:sli_error:ratio_rate1h > 0.0144 and on (slo) slo:sli_error:ratio_rate5m > 0.0144
```

**Prevent →** Generate all windows from one template so labels are identical; unit test the alert.

</details>

### Q34. Applying your `PrometheusRule` fails with `admission webhook "prometheusrulevalidate.monitoring.coreos.com" denied the request: Rules are not valid`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Prometheus Operator's admission webhook found invalid PromQL or invalid rule fields (wrong indentation, `expr` missing, invalid duration like `for: 2min`).

**Diagnose →**
```bash
yq '.spec' checkout-slo.yaml > /tmp/rules.yaml
promtool check rules /tmp/rules.yaml
```

**Fix →** Fix the reported expression or field (durations use `2m`, not `2min`) and reapply.

**Prevent →** Keep rules as plain Prometheus rule files tested with `promtool`, and wrap them into the CRD in CI.

</details>

### Q35. The 3-day ticket alert never fires, even during a slow multi-day burn. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus retention is shorter than the window (for example `--storage.tsdb.retention.time=2d`), so `[3d]` only sees 2 days of data, or a recent restart without persistent storage wiped history.

**Diagnose →**
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.retention}'
```
```promql
prometheus_tsdb_lowest_timestamp_seconds
```

**Fix →** Increase retention beyond the longest SLO window (at least 30d for the budget) with persistent volumes, or evaluate long-window rules in Thanos Ruler or Mimir ruler against long-term storage.

**Prevent →** Document retention requirements in the SLO platform and alert if the lowest timestamp is newer than 30 days.

</details>

### Q36. The fast-burn page fired, yet the monthly SLO dashboard shows 99.96% against a 99.9% target. Is the alert wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not wrong. Burn-rate alerts measure how fast the budget is being consumed right now, not whether the SLO is currently met. A 14.4x burn for an hour costs 2% of the monthly budget; left unchecked it would exhaust the budget in about 2 days.

**Diagnose →** Check the 1h error ratio and budget remaining side by side.

**Fix →** Nothing to fix in the alert; respond to the incident. Explain on the dashboard by showing burn rate next to budget remaining.

**Prevent →** Train on-call on burn rate vs budget; include both on the SLO dashboard.

</details>

### Q37. `amtool config routes test severity=page team=payments` returns `default-receiver` instead of `payments-pager`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An earlier sibling route matches first without `continue: true`, or the payments route sits under a parent route whose matchers do not match these labels, or matchers use the wrong label name.

**Diagnose →**
```bash
amtool config routes show --config.file=alertmanager.yaml
amtool config routes test --config.file=alertmanager.yaml --verify.receivers=payments-pager severity=page team=payments
```

**Fix →** Reorder routes so more specific ones come first, fix the matchers, or add `continue: true` where an alert should reach multiple receivers.

**Prevent →** Add `amtool config routes test --verify.receivers` cases to CI for each team.

</details>

## Grafana SLO Dashboards

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q38. Your SLO dashboard ConfigMap is applied, but the dashboard never shows up in Grafana. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ConfigMap is missing the label the Grafana sidecar watches (`grafana_dashboard: "1"` by default in kube-prometheus-stack), it is in a namespace the sidecar does not search, or the JSON is invalid.

**Diagnose →**
```bash
kubectl get cm -A -l grafana_dashboard=1
kubectl logs -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard | tail -20
```

**Fix →** Add the label, put the ConfigMap in the monitoring namespace (or set `grafana.sidecar.dashboards.searchNamespace: ALL`), and validate the JSON with `jq . dashboard.json`.

**Prevent →** Provision dashboards from Git with a CI JSON lint step.

</details>

### Q39. An imported SLO dashboard shows `Datasource ${DS_PROMETHEUS} was not found` on every panel. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The JSON was exported with "Export for sharing externally", which replaces data source references with an input variable that is only resolved through the UI import flow, not through file provisioning.

**Diagnose →** Search the JSON for `${DS_PROMETHEUS}` and check the `__inputs` block.

**Fix →** Replace references with the real data source UID (`{"type":"prometheus","uid":"prometheus"}`) or define a `datasource` template variable and use `${datasource}`.

**Prevent →** Export provisioned dashboards without the external-sharing option and use a data source variable.

</details>

### Q40. The "error budget remaining" stat changes when you change the dashboard time picker from 7 days to 30 days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query uses `$__range` or the panel's "reduce" over the selected range instead of a fixed SLO window. The budget must always be computed over the SLO window regardless of what range the viewer picks.

**Diagnose →** Inspect the panel query and the stat panel's calculation setting.

**Fix →** Query the fixed-window recording rule (`slo:sli_error:ratio_rate30d`) and set the stat to "Last" (not "Mean"). Optionally add a "relative time" override of `now-30d`.

**Prevent →** Document which panels are window-bound and which follow the time picker.

</details>

### Q41. You just created the SLO recording rules and the 30-day budget panel shows only a few hours of data. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus evaluates recording rules only going forward; it does not backfill history for newly added rules.

**Diagnose →** Check the earliest timestamp of the recorded series vs the raw metric.

**Fix →** Backfill with promtool:
```bash
promtool tsdb create-blocks-from rules \
  --start=$(date -d '-30 days' +%s) --end=$(date +%s) \
  --url=http://localhost:9090 slo-rules.yaml
```
Then copy the generated blocks into Prometheus' data directory. Alternatively, wait for the window to fill, and label the panel accordingly.

**Prevent →** Add SLO rules early in a service's life, or include backfill in the rollout procedure.

</details>

### Q42. The budget remaining panel shows `9,850%` instead of `98.5%`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unit mismatch. The query already multiplies by 100 (returns `98.5`), but the panel unit is "Percent (0.0-1.0)", which multiplies by 100 again. The reverse mistake (ratio `0.985` with unit "Percent (0-100)") shows `0.985%`.

**Diagnose →** Run the query in Explore and check the raw value.

**Fix →** If the value is 0.985, use unit "Percent (0.0-1.0)" (`percentunit`). If you multiply by 100 in the query, use "Percent (0-100)".

**Prevent →** Keep ratios as 0 to 1 in recording rules and format in Grafana with `percentunit`.

</details>

### Q43. SLO panels time out with `504 Gateway Timeout` when viewing 30 days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Panels run raw `rate()` queries over many series and a long range with a small step, which is slow; a proxy (Grafana or ingress) times out first.

**Diagnose →** Use Query inspector to see query time, and check Prometheus query logs or `prometheus_engine_query_duration_seconds`.

**Fix →** Switch panels to recording rules, set a sensible `Min interval` (for example 5m for 30d views), and avoid regex selectors on high-cardinality labels.

**Prevent →** Review dashboards for raw-metric queries; SLO dashboards should read only `slo:*` series.

</details>

### Q44. Deploy annotations stopped appearing on the SLO dashboard after the CI system was migrated. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new pipeline no longer posts to Grafana's annotation API, the service account token expired, or the dashboard's annotation query filters on a tag the new pipeline does not set.

**Diagnose →**
```bash
curl -s -H "Authorization: Bearer $GRAFANA_TOKEN" "$GRAFANA_URL/api/annotations?tags=deploy&limit=5" | jq .
```

**Fix →** Add a pipeline step:
```bash
curl -s -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" \
  "$GRAFANA_URL/api/annotations" \
  -d '{"tags":["deploy","checkout"],"text":"checkout 2.14.0 deployed"}'
```

**Prevent →** Treat deploy annotations as part of the deployment contract and test them in the pipeline.

</details>

### Q45. The SLO dashboard's 1h burn-rate panel shows 9x, but the alert evaluated 15x at the same moment. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The panel computes `rate()` itself with `$__rate_interval` or a different window and step, while the alert uses the recorded 1h ratio. Panel step alignment and min interval also change what is displayed.

**Diagnose →** Compare the panel query with the recording rule expression, and query the recorded series directly in Explore.

**Fix →** Make the panel query the same recorded series the alert uses (`slo:sli_error:ratio_rate1h / 0.001`).

**Prevent →** Single source of truth: dashboards and alerts both read recording rules.

</details>

### Q46. After a Helm upgrade of kube-prometheus-stack, all SLO dashboards created in the Grafana UI are gone. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Grafana ran without persistence; UI-created dashboards were stored in SQLite inside an ephemeral pod, lost when the pod was recreated.

**Diagnose →**
```bash
helm get values monitoring -n monitoring | grep -A3 persistence
```

**Fix →** Recreate from any exported JSON, then provision dashboards as code via ConfigMaps (sidecar) or the Grafana Terraform provider. Enable persistence or an external database for Grafana state.

**Prevent →** Dashboards as code in Git; the UI is for exploration, not the source of truth.

</details>

## k6 Load Testing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q47. k6 prints `WARN[0030] Insufficient VUs, reached 100 active VUs and cannot initialize more  executor=constant-arrival-rate scenario=steady`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With an arrival-rate executor, each in-flight request needs a VU. If the system slows down, more VUs are needed (Little's Law: rate x latency). When `maxVUs` is reached, k6 cannot start iterations on schedule and the target rate is not achieved, which hides how bad the slowdown is.

**Diagnose →** Check `dropped_iterations` in the summary, and compute required VUs: `rate x p99 latency (s)`. At 200 RPS and 2 s latency that is 400 VUs.

**Fix →** Raise `maxVUs` (and `preAllocatedVUs` to avoid allocation cost mid-test), or treat the warning as the test result: the system cannot sustain the rate within acceptable latency.

**Prevent →** Add a threshold on `dropped_iterations` (for example `'dropped_iterations': ['count<1']`) so the test fails visibly.

</details>

### Q48. CI fails with `ERRO[0602] thresholds on metrics 'http_req_duration' have been crossed` and exit code 99. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The release candidate is slower than the latency threshold, or the test environment was degraded (noisy neighbour, cold caches, undersized staging), or the load generator itself was saturated.

**Diagnose →** Read the end-of-test summary (`p(95)`, `p(99)`, `http_req_waiting` vs `http_req_connecting`), compare with the previous successful run, and check server-side latency and CPU throttling during the test window.

**Fix →** If server-side latency regressed, block the release and profile the change. If the environment was at fault, fix it and re-run, rather than loosening the threshold.

**Prevent →** Trend k6 results per build (JSON output or Prometheus remote write) and keep test environments sized consistently.

</details>

### Q49. Every request fails with `dial tcp: lookup checkout on 10.96.0.10:53: no such host`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hostname does not resolve from where k6 runs: running locally against an in-cluster Service name, wrong namespace (needs `checkout.shop.svc.cluster.local` from another namespace), or the Service does not exist.

**Diagnose →**
```bash
kubectl get svc -n shop checkout
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup checkout.shop.svc.cluster.local
```

**Fix →** Use the fully qualified Service name when running in-cluster, or port-forward / use an ingress URL when running outside.

**Prevent →** Pass `BASE_URL` via `-e` and validate it with a smoke test stage before the main scenario.

</details>

### Q50. Many requests fail with `request timeout` after exactly 60 seconds. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** k6's default per-request timeout is 60s. The service is hanging (thread pool exhausted, waiting on a dependency without timeouts), so requests never complete.

**Diagnose →** Check server-side: active connections, thread pools, DB pool waits, downstream latency. Check if the ingress has a longer timeout than k6.

**Fix →** Fix the hang on the server (timeouts on dependencies, bulkheads). In the test, set a realistic timeout matching real clients: `http.get(url, { timeout: '10s' })`.

**Prevent →** Every dependency call in the service has an explicit timeout; test with slow dependencies using chaos.

</details>

### Q51. The test starts fine, then errors switch to `dial tcp 10.96.41.12:8080: connect: connection refused`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pods crashed or restarted under load (OOMKilled, liveness probe failures), leaving no ready endpoints, or the Service points to a port nobody listens on after a restart.

**Diagnose →**
```bash
kubectl get pods -n shop -w
kubectl describe pod -n shop -l app=checkout | grep -A5 "Last State"
kubectl get endpointslices -n shop -l kubernetes.io/service-name=checkout
```

**Fix →** Fix the crash cause: memory limits, liveness probe too aggressive under load (use a cheap endpoint, longer timeouts), or startup time.

**Prevent →** Load tests should also assert pod restarts are zero; liveness probes should never depend on downstream services.

</details>

### Q52. The k6 load generator logs `socket: too many open files`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file descriptor limit on the load generator is too low for the number of concurrent connections.

**Diagnose →**
```bash
ulimit -n
cat /proc/$(pgrep k6)/limits | grep "open files"
```

**Fix →** Raise the limit: `ulimit -n 250000` before running, or set `LimitNOFILE` in systemd or the container runtime. Reuse connections (k6 does keep-alive by default; avoid `noConnectionReuse: true` unless intended).

**Prevent →** Use a tuned load-generator image or the k6 Operator with known limits, and document OS tuning for large tests.

</details>

### Q53. At high rates, k6 errors with `dial tcp 10.0.3.15:443: connect: cannot assign requested address`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ephemeral port exhaustion on the load generator: too many new connections to one destination IP:port, with old sockets stuck in `TIME_WAIT`. Often caused by `noConnectionReuse` or `noVUConnectionReuse`, or the server closing connections after each request.

**Diagnose →**
```bash
ss -tan state time-wait | wc -l
sysctl net.ipv4.ip_local_port_range
```

**Fix →** Enable connection reuse, widen the port range (`sysctl -w net.ipv4.ip_local_port_range="1024 65000"`), and spread load across multiple generators or target IPs.

**Prevent →** Run large tests distributed with the k6 Operator and check the generator's network metrics as part of the test.

</details>

### Q54. Latency results look terrible, but the server's histograms show p99 of 80 ms. Load generator CPU is at 100%. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The load generator is the bottleneck: k6 cannot process responses fast enough, so measured times include client-side scheduling delay.

**Diagnose →** Watch generator CPU and memory during the test (`top`, node metrics); compare `http_req_waiting` against server-side latency.

**Fix →** Give the generator more CPU, simplify the script (avoid heavy JSON parsing or large `check` functions per response, use `discardResponseBodies: true` when bodies are not needed), or distribute the test.

**Prevent →** Keep generator CPU below roughly 80% during tests and record it alongside results.

</details>

### Q55. k6 warns: `The test has generated metrics with 100001 unique time series, which is higher than the suggested limit of 100000 and could cause high memory usage`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** URLs with unique IDs (`/orders/8f2c...`) become the `name` and `url` tags, so each request creates new metric series.

**Diagnose →** Look for dynamic values in URLs and custom tags in the script.

**Fix →** Group URLs with a static name tag:
```javascript
http.get(`${BASE}/orders/${id}`, { tags: { name: 'GET /orders/:id' } });
```
or use `http.url` template literals, which set the name tag automatically.

**Prevent →** Review scripts for high-cardinality tags, same as Prometheus label hygiene.

</details>

### Q56. Streaming k6 results to Prometheus fails with `remote write receiver needs to be enabled with --web.enable-remote-write-receiver`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus does not accept remote write by default; the `/api/v1/write` endpoint returns 404 with that message.

**Diagnose →**
```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://localhost:9090/api/v1/write
```

**Fix →** Enable the receiver. In kube-prometheus-stack: `prometheus.prometheusSpec.enableRemoteWriteReceiver: true`. Then run:
```bash
K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write k6 run -o experimental-prometheus-rw test.js
```

**Prevent →** Document the integration in the platform's load-testing guide and enable it in the shared test Prometheus.

</details>

### Q57. The script fails at init with `ReferenceError: BASE_URL is not defined`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Environment variables are not globals in k6; they are read from `__ENV`.

**Diagnose →** Search the script for bare `BASE_URL`.

**Fix →**
```javascript
const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
```
and run with `k6 run -e BASE_URL=http://checkout.shop:8080 test.js`.

**Prevent →** Validate required env vars at the start of `setup()` and fail fast with a clear message.

</details>

### Q58. 20% of checks failed, but k6 still exited with code 0 and CI passed. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `check()` only records results; it does not fail the test. Only thresholds affect the exit code.

**Diagnose →** Look at the summary: `checks.........: 80.00%` with no threshold listed.

**Fix →** Add a threshold on checks:
```javascript
thresholds: { checks: ['rate>0.99'] }
```

**Prevent →** Template scripts with SLO thresholds on `http_req_failed`, `http_req_duration` and `checks`.

</details>

### Q59. The k6 Kubernetes Job fails with `The moduleSpecifier "/scripts/test.js" couldn't be found on local disk`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ConfigMap with the script is not mounted, mounted at a different path, or the key name differs from `test.js`.

**Diagnose →**
```bash
kubectl get cm -n loadtest k6-script -o yaml | head -20
kubectl get job -n loadtest k6-run -o yaml | yq '.spec.template.spec.volumes, .spec.template.spec.containers[0].volumeMounts'
```

**Fix →** Create the ConfigMap with the expected key (`kubectl create configmap k6-script --from-file=test.js`) and mount it at `/scripts`.

**Prevent →** Use the k6 Operator `TestRun` resource, which handles script ConfigMaps consistently.

</details>

### Q60. With `constant-vus`, the test reports p99 of 400 ms, but production users see multi-second latency at the same RPS. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Coordinated omission: in a closed model, each VU waits for its response before sending the next, so when the server slows down, k6 sends fewer requests and never measures the queueing real users experience.

**Diagnose →** Compare `http_reqs` rate during slow periods: it drops when latency rises.

**Fix →** Use an open model (`constant-arrival-rate` or `ramping-arrival-rate`) with enough `maxVUs` so the request rate is independent of response time.

**Prevent →** Make arrival-rate executors the default in the load-testing template for user-facing services.

</details>

### Q61. A Grafana k6 dashboard shows no data, although `experimental-prometheus-rw` output reports no errors. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Metric names differ from what the dashboard expects (k6 remote-write metrics are prefixed `k6_`, and trend metrics are exported as summaries or native histograms depending on `K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM` and `K6_PROMETHEUS_RW_TREND_STATS`), or the dashboard filters on a `testid` tag that was not set.

**Diagnose →**
```promql
count by (__name__) ({__name__=~"k6_.*"})
```

**Fix →** Align the dashboard queries with the exported names, set the trend stats you need (for example `K6_PROMETHEUS_RW_TREND_STATS="p(95),p(99),avg"`), and tag runs with `--tag testid=release-2.14`.

**Prevent →** Use the dashboard version that matches your k6 output settings and pin both.

</details>

## Chaos Mesh, Litmus & AWS FIS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q62. A `NetworkChaos` delay experiment does nothing, and chaos-daemon logs show `RTNETLINK answers: No such file or directory` / `Specified qdisc kind is unknown`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host kernel lacks the `sch_netem` module, which `tc` needs for delay and loss. Common on minimal VM images and some Docker Desktop or WSL kernels.

**Diagnose →**
```bash
lsmod | grep sch_netem
kubectl logs -n chaos-mesh ds/chaos-daemon --tail=50 | grep -i rtnetlink
```

**Fix →** On the node: `sudo modprobe sch_netem` (on kind, the node containers share the host kernel, so load it on the host). On cloud nodes, install the kernel extras package (for example `linux-modules-extra-$(uname -r)` on Ubuntu).

**Prevent →** Add a pre-flight check to the chaos platform that verifies kernel modules on all nodes.

</details>

### Q63. Chaos experiments fail with `dial unix /var/run/docker.sock: connect: no such file or directory` on a containerd cluster. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Chaos Mesh was installed with the default Docker runtime settings, but the nodes run containerd (kind, EKS, GKE, most modern clusters).

**Diagnose →**
```bash
kubectl get nodes -o jsonpath='{.items[0].status.nodeInfo.containerRuntimeVersion}'
helm get values chaos-mesh -n chaos-mesh
```

**Fix →**
```bash
helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --reuse-values \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```
(k3s uses `/run/k3s/containerd/containerd.sock`.)

**Prevent →** Keep Chaos Mesh values per cluster type in Git.

</details>

### Q64. `kubectl apply -f podchaos.yaml` fails with `Internal error occurred: failed calling webhook ... context deadline exceeded`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Chaos Mesh admission webhook (served by the controller manager) is unreachable: controller pods not ready, a NetworkPolicy blocks the API server, or on private clusters the control plane cannot reach the webhook port.

**Diagnose →**
```bash
kubectl get pods -n chaos-mesh
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep -i chaos
kubectl logs -n chaos-mesh deploy/chaos-controller-manager --tail=50
```

**Fix →** Restore the controller manager, allow API server traffic to the webhook port, and on private GKE/EKS clusters open the firewall or security group rule for that port.

**Prevent →** Monitor the chaos controller like any other platform component.

</details>

### Q65. A `PodChaos` resource is created, but no pod is killed. The status shows no records. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The selector matches no pods: wrong namespace, wrong label key or value, or namespace filtering is enabled and the namespace is not annotated.

**Diagnose →**
```bash
kubectl describe podchaos checkout-pod-kill -n shop
kubectl get pods -n shop -l app=checkout
```

**Fix →** Correct `selector.namespaces` and `labelSelectors`. If Chaos Mesh was installed with `controllerManager.enableFilterNamespace=true`, annotate the target namespace: `kubectl annotate ns shop chaos-mesh.org/inject=enabled`.

**Prevent →** Dry-run selectors with `kubectl get pods -l ...` before every experiment and keep selectors in reviewed manifests.

</details>

### Q66. You deleted the `NetworkChaos`, but the 200 ms delay is still there and the resource is stuck in `Terminating`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The controller could not recover the fault (chaos-daemon on that node restarted or the target pod's network namespace changed), so the finalizer is never removed and the `tc` rule may still exist.

**Diagnose →**
```bash
kubectl get networkchaos -n shop inventory-latency -o yaml | yq '.metadata.finalizers, .status'
kubectl logs -n chaos-mesh ds/chaos-daemon --tail=100 | grep -i recover
```

**Fix →** Restart the affected target pods (removes the qdisc with the network namespace), then, as a last resort, force-clean the finalizer:
```bash
kubectl annotate networkchaos inventory-latency -n shop chaos-mesh.chaos-mesh.org/cleanFinalizer=forced
```

**Prevent →** Always set `duration` on experiments and ensure chaos-daemon has PriorityClass and resources so it is not evicted mid-experiment.

</details>

### Q67. `NetworkChaos` added 300 ms delay to the checkout pods, but the checkout latency SLI did not move. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SLI comes from the app's own histogram, which measures time inside the handler. Delay on the pod's egress packets happens after the handler finishes, so only clients see it.

**Diagnose →** Compare k6 or ingress latency (client side) with the app histogram during the experiment.

**Fix →** Measure the latency SLI at the ingress or load balancer, or inject the delay on a downstream dependency so the caller's handler waits for it.

**Prevent →** Validate that every SLI reacts to a fault you inject; an SLI that does not move during a known fault is blind to that failure mode.

</details>

### Q68. A Litmus experiment pod never starts: `error looking up service account shop/pod-delete-sa: serviceaccount "pod-delete-sa" not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `ChaosEngine` references `chaosServiceAccount: pod-delete-sa`, but the ServiceAccount and its RBAC were not created in the target namespace.

**Diagnose →**
```bash
kubectl get sa -n shop
kubectl get chaosengine -n shop -o yaml | grep chaosServiceAccount
```

**Fix →** Apply the experiment's RBAC manifest (ServiceAccount, Role, RoleBinding) in the namespace, or point the engine to an existing chaos ServiceAccount.

**Prevent →** Template RBAC together with the `ChaosEngine` in the same manifest bundle.

</details>

### Q69. Litmus `pod-delete` fails: `pods is forbidden: User "system:serviceaccount:shop:pod-delete-sa" cannot delete resource "pods" in API group "" in the namespace "shop"`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ServiceAccount exists, but its Role lacks the `delete` verb on pods (or the RoleBinding points to another namespace).

**Diagnose →**
```bash
kubectl auth can-i delete pods -n shop --as=system:serviceaccount:shop:pod-delete-sa
kubectl describe rolebinding -n shop | grep -A3 pod-delete
```

**Fix →** Grant the minimum needed verbs (`get`, `list`, `delete` on pods; `create` on events; access to `chaosengines` and `chaosresults`).

**Prevent →** Use the RBAC manifests that ship with each ChaosHub experiment and test them in staging.

</details>

### Q70. The Litmus `ChaosResult` verdict is `Fail`, with the `promProbe` marked failed, but the app looked healthy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The probe's query, comparator or criteria is wrong: comparing a ratio to a percentage (`0.999` vs `99.9`), wrong operator (`<=` vs `>=`), the query returning no data (counts as failure), or the Prometheus endpoint unreachable from the experiment pod.

**Diagnose →**
```bash
kubectl describe chaosresult -n shop checkout-pod-delete
```
Run the probe's query manually against the same endpoint.

**Fix →** Correct the query and comparator (for example `criteria: ">="`, `value: "0.995"` on a success ratio), use the in-cluster Prometheus Service URL, and set a reasonable `probeTimeout` and `interval`.

**Prevent →** Test probes in a no-fault run first: the verdict must be Pass when nothing is injected.

</details>

### Q71. Litmus experiments stay `Awaited` forever in an air-gapped cluster. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chaos-runner or experiment image cannot be pulled (`ImagePullBackOff`) because images come from a public registry.

**Diagnose →**
```bash
kubectl get pods -n shop | grep -E "runner|pod-delete"
kubectl describe pod -n shop <runner-pod> | grep -A5 Events
```

**Fix →** Mirror the Litmus images to your private registry and set the image references in the `ChaosEngine` / `ChaosExperiment` (and the image registry setting in ChaosCenter), plus an `imagePullSecret` if needed.

**Prevent →** Include chaos tooling images in your standard image mirroring pipeline.

</details>

### Q72. An AWS FIS experiment fails immediately with a target resolution error saying the target set is empty. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No resources match the target's tags, filters or ARNs in that region (tag key typo, instances in another region, filters like `State.Name` excluding everything).

**Diagnose →**
```bash
aws fis get-experiment --id EXP123 --query 'experiment.state'
aws ec2 describe-instances --filters "Name=tag:chaos-target,Values=checkout" \
  --query 'Reservations[].Instances[].InstanceId'
```

**Fix →** Fix the tags or filters in the template and check the region.

**Prevent →** Validate target selection with a describe call in the same automation that starts the experiment.

</details>

### Q73. FIS fails with `is not authorized to perform: ec2:TerminateInstances on resource: arn:aws:ec2:ap-south-1:123456789012:instance/i-0abc...`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM role in the experiment template lacks permissions for the action, or an SCP or permission boundary denies it.

**Diagnose →**
```bash
aws fis get-experiment-template --id EXT123 --query 'experimentTemplate.roleArn'
aws iam simulate-principal-policy --policy-source-arn <role-arn> \
  --action-names ec2:TerminateInstances
```

**Fix →** Attach the AWS managed policy for the relevant FIS action category (for example the EC2 access policy for FIS) or a scoped custom policy, conditioned on the chaos tag.

**Prevent →** Keep FIS roles in IaC with least privilege scoped by tags, and test templates in a sandbox account.

</details>

### Q74. FIS `aws:eks:pod-delete` fails because FIS cannot create its pod in the cluster (`forbidden`). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The FIS experiment role is not mapped into Kubernetes RBAC (no EKS access entry or `aws-auth` mapping), or the Kubernetes ServiceAccount named in the action lacks permissions in the target namespace.

**Diagnose →**
```bash
aws eks list-access-entries --cluster-name prod
kubectl auth can-i create pods -n shop --as=system:serviceaccount:shop:fis-experiment
```

**Fix →** Add an access entry (or `aws-auth` mapping) for the FIS role to a Kubernetes user or group, and create the ServiceAccount, Role and RoleBinding that the FIS pod actions require in the target namespace.

**Prevent →** Provision FIS EKS integration with the cluster's IaC, and run a smoke experiment after cluster upgrades.

</details>

### Q75. A FIS experiment with a stop condition on the SLO alarm kept running while errors rose. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CloudWatch alarm was in `INSUFFICIENT_DATA` (metric missing or delayed), its evaluation periods were too long to react within the experiment, or it watched the wrong metric or dimension.

**Diagnose →**
```bash
aws cloudwatch describe-alarm-history --alarm-name checkout-slo-burn --max-items 10
aws cloudwatch describe-alarms --alarm-names checkout-slo-burn \
  --query 'MetricAlarms[].{state:StateValue,periods:EvaluationPeriods,period:Period}'
```

**Fix →** Use a 1-minute period with 1 to 3 datapoints, `TreatMissingData: breaching` for stop-condition alarms, and the correct dimensions. Verify the alarm fires by forcing it with `aws cloudwatch set-alarm-state` in a test.

**Prevent →** Test stop conditions before every new experiment template.

</details>

### Q76. A Chaos Mesh `StressChaos` CPU experiment pegged the node and evicted unrelated pods. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Stress workers ran without effective limits relative to the target container, so the injected load competed with other pods on the node, or the target pods had no CPU limits and the node's allocatable was exceeded.

**Diagnose →**
```bash
kubectl describe stresschaos -n shop checkout-cpu
kubectl get events -A --field-selector reason=Evicted
kubectl top node
```

**Fix →** Set `stressors.cpu.workers` and `load` conservatively, target containers with CPU limits, and use `mode: one` and short `duration`.

**Prevent →** Run resource stress only on dedicated or well-limited workloads, and review blast radius in the experiment design.

</details>

## Reliability Patterns in Production

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q77. Clients get 503s with body `upstream connect error or disconnect/reset before headers. reset reason: overflow`, and Envoy logs show flag `UO`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Istio/Envoy circuit breaker tripped: the `DestinationRule` `connectionPool` limits (`maxConnections`, `http1MaxPendingRequests`, `http2MaxRequests`) are lower than the current concurrency. `UO` means upstream overflow.

**Diagnose →**
```bash
kubectl get destinationrule -n shop inventory -o yaml
kubectl exec -n shop deploy/checkout -c istio-proxy -- \
  pilot-agent request GET stats | grep -E "upstream_rq_pending_overflow|upstream_cx_overflow"
```

**Fix →** If the limit is protecting a genuinely saturated service, it is working; fix capacity. If the limits were set too low, raise them based on Little's Law (RPS x latency) with headroom.

**Prevent →** Size connection pool limits from load tests and alert on overflow counters.

</details>

### Q78. NGINX ingress returns 504 and logs `upstream timed out (110: Connection timed out) while reading response header from upstream`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backend did not respond within `proxy-read-timeout` (default 60s): slow queries, a hung dependency, or a long-running endpoint that should be async.

**Diagnose →**
```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep "upstream timed out" | tail
kubectl get ingress -n shop checkout -o yaml | grep -i timeout
```
Check the backend's latency and dependency traces for those requests.

**Fix →** Fix the slow backend path (timeouts on its dependencies, indexes, async jobs). Only raise `nginx.ingress.kubernetes.io/proxy-read-timeout` for endpoints that legitimately take long.

**Prevent →** Nest timeouts: backend dependency timeouts < backend request timeout < ingress timeout < client timeout.

</details>

### Q79. After a 30-second database blip, Postgres keeps rejecting connections with `FATAL: sorry, too many clients already` for 20 minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A retry storm and connection storm: every pod retried immediately, opened new connections to replace broken ones, and the pools' total maximum exceeds `max_connections`. The DB is now saturated by reconnect attempts (a metastable failure).

**Diagnose →**
```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
SHOW max_connections;
```
Compare `pods x pool max` with `max_connections`.

**Fix →** Shed load: scale callers down or disable retries via flag, terminate idle connections, then bring traffic back gradually. Longer term, put PgBouncer or RDS Proxy in front, cap pool sizes so their sum fits, and add backoff with jitter on reconnects.

**Prevent →** Capacity check in reviews: `replicas x maxPoolSize` must fit `max_connections` at max HPA scale.

</details>

### Q80. Customers were charged twice during an incident where the payment API timed out. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client retried a non-idempotent `POST` after a timeout, but the first request had actually succeeded on the server.

**Diagnose →** Match duplicate charges by customer and timestamp in payment logs; check retry configuration in the client library or mesh (`VirtualService` `retries` with `retryOn: 5xx,reset`).

**Fix →** Refund duplicates. Add idempotency keys: the client generates a key per logical payment, the server stores the result per key and returns it on duplicates. Remove automatic retries for non-idempotent calls at the mesh layer.

**Prevent →** Idempotency keys mandatory for all write APIs; retry policies reviewed per route.

</details>

### Q81. Every rolling deploy produces a short burst of 502s on the AWS ALB. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pods terminate before the ALB stops sending traffic: no `preStop` delay, app exits immediately on SIGTERM, or deregistration delay is longer than `terminationGracePeriodSeconds`. ALB access logs show `elb_status_code 502` with `target_status_code -`.

**Diagnose →**
```bash
kubectl get deploy checkout -n shop -o yaml | yq '.spec.template.spec | {grace: .terminationGracePeriodSeconds, lifecycle: .containers[0].lifecycle}'
kubectl get ingress -n shop checkout -o yaml | grep -i deregistration
```

**Fix →** Add `preStop: sleep 15`, handle SIGTERM gracefully, set `terminationGracePeriodSeconds` above preStop plus drain time, and align the target group `deregistration_delay.timeout_seconds`. With the AWS Load Balancer Controller, enable pod readiness gates.

**Prevent →** Run a pod-kill chaos experiment under load after any change to deployment settings.

</details>

### Q82. Redis restarted and immediately the database CPU hit 100% and the site went down. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cache stampede: all cache misses at once sent the full read load to the database, which was sized only for the cache-miss rate.

**Diagnose →** Correlate cache hit ratio, DB QPS and DB CPU around the restart.

**Fix →** Shed load at the edge, then warm the cache gradually. Add request coalescing (single-flight) so concurrent misses for the same key trigger one DB query, use stale-while-revalidate, and add jitter to TTLs.

**Prevent →** Treat the cache as a hard dependency in capacity planning, or size the DB for cold-cache load; run a cache-flush chaos experiment.

</details>

### Q83. Logs fill with `CallNotPermittedException: CircuitBreaker 'inventory' is OPEN and does not permit further calls` long after inventory recovered. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The breaker cannot close: half-open trial calls keep failing (they time out because the timeout is too short for a cold dependency), `permittedNumberOfCallsInHalfOpenState` is tiny, or `waitDurationInOpenState` is very long.

**Diagnose →** Check breaker state metrics (`resilience4j_circuitbreaker_state`) and the result of half-open calls; call inventory manually from the pod.

**Fix →** Tune `waitDurationInOpenState` (for example 30s), allow several half-open calls, and make sure the call timeout is realistic for a recovering dependency.

**Prevent →** Export breaker state as metrics, alert on breakers open longer than N minutes, and test recovery in chaos experiments.

</details>

### Q84. Your NGINX rate limit returns 503 to throttled clients, and the SLO counts them as errors. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `limit_req` returns 503 by default, so intentional throttling looks like server failure in the SLI and to clients (who may retry aggressively).

**Diagnose →**
```bash
grep -n "limit_req" /etc/nginx/nginx.conf
```
Correlate 503 spikes with `limiting requests` entries in the NGINX error log.

**Fix →** Set `limit_req_status 429;` (ingress-nginx: `limit-req-status-code: "429"` in the ConfigMap). Decide explicitly whether 429s for fair-use throttling count against the SLO; overload-driven 429s usually should.

**Prevent →** Document status codes for throttling and shedding, and include them in the SLI definition.

</details>

### Q85. The product page fails with `BulkheadFullException: Bulkhead 'recommendations' is full and does not permit further calls`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bulkhead is working as designed (recommendations is slow and its concurrency slots are full), but the caller does not have a fallback, so the exception bubbles up and fails the whole page.

**Diagnose →** Check recommendations latency and bulkhead metrics (`resilience4j_bulkhead_available_concurrent_calls`).

**Fix →** Add a fallback (empty or cached recommendations) so the page renders without that section; fix the slowness in the recommendations service.

**Prevent →** For every soft dependency, define and test the fallback, not just the isolation.

</details>

### Q86. gRPC clients see `rpc error: code = DeadlineExceeded desc = context deadline exceeded` even though the server logs show requests finishing in 200 ms. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deadline is consumed before the request reaches this server: an upstream caller set a short overall deadline and earlier hops used most of it, or queueing (client-side connection limits, server max concurrent streams) delays the start. Server logs measure only processing time.

**Diagnose →** Use distributed traces to see where time is spent across hops; check the `grpc-timeout` header received by the server and client-side queueing metrics.

**Fix →** Rebalance deadlines along the chain, parallelize calls, remove queueing (raise `MaxConcurrentStreams`, add connections), and fail fast when remaining deadline is too small.

**Prevent →** Define latency budgets per hop from the top-level SLO, and propagate deadlines everywhere.

</details>

### Q87. A node drain for maintenance hangs with `Cannot evict pod as it would violate the pod's disruption budget.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A PodDisruptionBudget allows zero disruptions right now: `minAvailable` equals replica count, a single-replica deployment has `minAvailable: 1`, or some replicas are already unhealthy.

**Diagnose →**
```bash
kubectl get pdb -A
kubectl describe pdb -n shop checkout
```
Look at `ALLOWED DISRUPTIONS`.

**Fix →** Scale up so the PDB permits one disruption, fix unhealthy pods, or adjust the PDB (`maxUnavailable: 1`). Do not force-delete pods unless you accept the impact.

**Prevent →** At least 2 replicas for anything with a PDB, and use `maxUnavailable` rather than `minAvailable` equal to replicas.

</details>

### Q88. The HPA does not scale during a load test: `failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** metrics-server is not installed or not healthy (on kind it often fails TLS to kubelets).

**Diagnose →**
```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl -n kube-system logs deploy/metrics-server | tail
kubectl top pods -n shop
```

**Fix →** Install metrics-server; on kind add `--kubelet-insecure-tls` to its args (lab only). Also make sure pods have CPU requests, or the HPA cannot compute utilization.

**Prevent →** Include metrics-server in the cluster baseline and alert if the APIService is unavailable.

</details>

### Q89. Under load, pods start failing readiness with `Readiness probe failed: Get "http://10.244.1.7:8080/healthz": context deadline exceeded (Client.Timeout exceeded while awaiting headers)`, and the remaining pods collapse. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The probe endpoint competes with real traffic for the same threads, or checks downstream dependencies. Under load it times out (default `timeoutSeconds: 1`), pods are removed from endpoints, the remaining pods get more traffic and fail too: a cascading failure.

**Diagnose →**
```bash
kubectl describe pod -n shop -l app=checkout | grep -B2 -A2 "Readiness probe failed"
kubectl get endpointslices -n shop -l kubernetes.io/service-name=checkout -w
```

**Fix →** Make the readiness endpoint cheap and local (no dependency calls), raise `timeoutSeconds` and `failureThreshold`, and serve probes on a separate lightweight listener if possible. Add load shedding so overloaded pods reject excess work instead of becoming unready.

**Prevent →** Load test with probe behaviour in mind, and never use dependency checks in liveness probes.

</details>

### Q90. p99 latency spikes every few seconds although CPU usage averages only 40% of the limit. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CFS throttling: the container bursts above its CPU limit within a 100 ms period and is paused for the rest of the period. Averages hide it, especially for multi-threaded runtimes (JVM, Go with high GOMAXPROCS).

**Diagnose →**
```promql
rate(container_cpu_cfs_throttled_periods_total{namespace="shop",container="checkout"}[5m])
/ rate(container_cpu_cfs_periods_total{namespace="shop",container="checkout"}[5m])
```

**Fix →** Raise or remove CPU limits (keep requests), set GOMAXPROCS to match the limit (automaxprocs) or JVM `-XX:ActiveProcessorCount`, and re-test.

**Prevent →** Alert on throttling ratio above 25% for latency-sensitive services.

</details>

## Backups, DR & Runbook Automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. `velero backup describe shop-nightly` shows `Phase: PartiallyFailed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Some items failed: volume snapshots failed (missing snapshot class, cloud permissions), file-level backup of a volume failed, or a resource could not be read.

**Diagnose →**
```bash
velero backup describe shop-nightly --details
velero backup logs shop-nightly | grep -iE "error|fail"
kubectl get volumesnapshotclass
```

**Fix →** Fix the specific failures: add a `VolumeSnapshotClass` labeled for Velero, grant the plugin's IAM permissions, or enable file-system backup for volumes that cannot be snapshotted.

**Prevent →** Alert on Velero metrics (`velero_backup_partial_failure_total`, `velero_backup_last_successful_timestamp`) and run scheduled restore tests.

</details>

### Q92. After restoring a namespace into the DR cluster with Velero, PVCs stay `Pending`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The StorageClass referenced by the PVCs does not exist in the DR cluster (different name or CSI driver), or snapshots cannot be restored across regions without copying them first.

**Diagnose →**
```bash
kubectl get pvc -n shop
kubectl describe pvc -n shop data-postgres-0 | grep -A5 Events
kubectl get storageclass
```

**Fix →** Use Velero's storage class mapping (a ConfigMap for the `change-storage-class` restore item action) to map old to new classes, and copy snapshots to the DR region, or use file-system backups stored in object storage replicated across regions.

**Prevent →** Keep StorageClass names consistent across clusters and include cross-region restore in DR drills.

</details>

### Q93. `etcdctl snapshot save` fails with `Error: context deadline exceeded`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** etcdctl cannot reach or authenticate to etcd: missing TLS flags, wrong endpoint, or etcd is unhealthy.

**Diagnose →**
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health
```

**Fix →** Pass the CA, cert and key flags and the right endpoint to `snapshot save`, then verify with `etcdutl snapshot status <file> -w table`.

**Prevent →** Wrap etcd backups in a tested script or CronJob with alerting on failure and age.

</details>

### Q94. A restore test fails with `pg_restore: error: could not execute query: ERROR:  role "app_rw" does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pg_dump` captures objects and ownership but not cluster-wide roles. The target server lacks the roles referenced by `OWNER TO` and `GRANT` statements.

**Diagnose →** Check roles on the target with `\du` in `psql`.

**Fix →** Restore roles first (`pg_dumpall --roles-only` from the source, or create them via IaC), or restore with `--no-owner --no-privileges` and reapply grants.

**Prevent →** Include roles in the backup set and run automated restore tests into an empty instance, which catch exactly this.

</details>

### Q95. You restored an RDS instance to a point in time, but the application cannot connect. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A point-in-time restore creates a new instance with a new endpoint, and it may get the default security group, parameter group or subnet group rather than the original ones.

**Diagnose →**
```bash
aws rds describe-db-instances --db-instance-identifier orders-pitr \
  --query 'DBInstances[0].{endpoint:Endpoint.Address,sgs:VpcSecurityGroups,pg:DBParameterGroups}'
```

**Fix →** Attach the correct security groups and parameter group, then point the application to the new endpoint (update a Route 53 CNAME or the secret holding the host), or rename instances to swap endpoints.

**Prevent →** Script PITR in the runbook with all parameters, and use a DNS name the application resolves instead of raw endpoints.

</details>

### Q96. The nightly backup job reports success, but the backup files in S3 are 0 bytes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script pipes `pg_dump | gzip | aws s3 cp -`, and without `pipefail` only the last command's exit code counts; `pg_dump` failed (auth, connection) but the upload succeeded with empty input.

**Diagnose →** Check job logs for `pg_dump` errors and list object sizes:
```bash
aws s3 ls s3://backups/orders/ --human-readable | tail
```

**Fix →**
```bash
set -euo pipefail
pg_dump -Fc "$DATABASE_URL" | aws s3 cp - "s3://backups/orders/$(date +%F).dump"
```

**Prevent →** Alert on backup size anomalies and age, and verify each backup with a restore or at least `pg_restore --list`.

</details>

### Q97. The backup CronJob silently stopped running. Events show `Cannot determine if job needs to be started: too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CronJob was suspended or the controller could not schedule it for a long time; after more than 100 missed schedules the controller stops trying without a `startingDeadlineSeconds`.

**Diagnose →**
```bash
kubectl get cronjob -n ops db-backup -o yaml | yq '.spec.suspend, .spec.startingDeadlineSeconds, .status.lastScheduleTime'
kubectl get events -n ops --field-selector involvedObject.name=db-backup
```

**Fix →** Set `startingDeadlineSeconds` (for example 3600), unsuspend, and trigger a manual run: `kubectl create job --from=cronjob/db-backup db-backup-manual -n ops`.

**Prevent →** Alert on backup age, not on job status:
```promql
time() - kube_cronjob_status_last_successful_time{cronjob="db-backup"} > 26 * 3600
```

</details>

### Q98. An auto-remediation job restarted the checkout deployment 412 times this week, and nobody noticed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Automation turned a recurring failure (memory leak, deadlock) into silent toil. Each restart hid the symptom, and nobody alerted on how often the remediation ran.

**Diagnose →** Count remediation runs from automation logs or metrics, and correlate with versions and memory growth.

**Fix →** Open an incident or high-priority ticket to fix the underlying bug; add a rate limit to the automation (for example at most 3 restarts per hour, then page a human).

**Prevent →** Every auto-remediation emits a metric, alerts when it runs more than expected, and creates a ticket for each run.

</details>

### Q99. A Route 53 failover to the DR region took 25 minutes against an RTO of 5 minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Health checks were slow to mark the primary unhealthy (long interval, high failure threshold), DNS TTLs were high, and clients cached DNS beyond the TTL (JVM `networkaddress.cache.ttl`, connection pools holding old connections).

**Diagnose →**
```bash
aws route53 get-health-check-status --health-check-id <id>
dig +noall +answer shop.example.com
```
Review client DNS caching settings and connection reuse.

**Fix →** Use 10-second health checks with a low failure threshold, TTL of 60s or less on failover records, configure clients to respect TTLs and recycle connections, or use a global load balancer or AWS Global Accelerator instead of DNS alone.

**Prevent →** Measure failover time in every DR drill and include client behaviour in the test.

</details>

### Q100. The runbook linked from a page returns 404, and the on-call engineer improvised for 30 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The wiki moved, the runbook was renamed, or it was never written; alert annotations hard-code URLs that nobody validates.

**Diagnose →**
```bash
curl -s localhost:9090/api/v1/rules | jq -r '.data.groups[].rules[] | select(.type=="alerting") | .annotations.runbook_url' \
  | sort -u | while read u; do printf '%s %s\n' "$(curl -s -o /dev/null -w '%{http_code}' "$u")" "$u"; done
```

**Fix →** Restore or write the runbook, update the annotation.

**Prevent →** Keep runbooks in Git next to the alert rules, require `runbook_url` in CI linting, and run the link checker above on a schedule.

</details>

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
