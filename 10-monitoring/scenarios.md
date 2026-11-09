# Monitoring — Scenario-Based Questions

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

> Real-world monitoring and observability situations and how a senior engineer would handle them · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Monitoring](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Alerting and On-Call Design](#alerting-and-on-call-design) — Q1–Q25
- [Prometheus and PromQL at Scale](#prometheus-and-promql-at-scale) — Q26–Q50
- [Logging Pipelines](#logging-pipelines) — Q51–Q75
- [Tracing and OpenTelemetry](#tracing-and-opentelemetry) — Q76–Q100
- [SLOs and Error Budgets](#slos-and-error-budgets) — Q101–Q125
- [Grafana, Dashboards and Kubernetes Monitoring](#grafana-dashboards-and-kubernetes-monitoring) — Q126–Q150
- [Long-Term Storage, Scale and Cost](#long-term-storage-scale-and-cost) — Q151–Q175
- [Cloud Monitoring, Incidents and Migrations](#cloud-monitoring-incidents-and-migrations) — Q176–Q200

---

## Alerting and On-Call Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your on-call engineers receive 300 alerts a night and have started ignoring the pager. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alert fatigue. The signal is buried, so a real incident will be missed. This is a process and design problem, not a tooling problem.

**What I'd do →**
1. Export a week of alerts from Alertmanager (or PagerDuty) and group by `alertname`. Usually 5 alerts produce 80% of the noise.
2. For each noisy alert ask: did a human take an action? If not, delete it or downgrade it to a ticket or dashboard.
3. Convert cause-based alerts (CPU high, pod restarted) to symptom-based alerts (error rate, latency) that page only on user impact.
4. Add `for:` durations, grouping (`group_by: [alertname, cluster]`) and inhibition.
5. Review alert quality weekly in the on-call handover.

**Why →** Every page must be urgent, actionable and need a human. Anything else trains people to ignore the pager.

**Interview tip →** Quote a rule: "if it doesn't need action now, it isn't a page."

</details>

### Q2. You are asked to design alerting for a brand-new HTTP service from scratch. What do you alert on? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No history, so start from the golden signals and keep the first set small.

**What I'd do →**
1. Page on **error ratio** (5xx / total) and **latency** (p99 over threshold) at the load balancer or service level.
2. Page on **availability** from an external blackbox probe.
3. Add **saturation** warnings (queue depth, connection pool, disk) as tickets, not pages.
4. Attach `severity`, `team`, `runbook_url` and a clear summary to each rule.
5. Tune after two weeks of real data.

```yaml
- alert: HighErrorRatio
  expr: sum(rate(http_requests_total{job="api",code=~"5.."}[5m])) / sum(rate(http_requests_total{job="api"}[5m])) > 0.05
  for: 10m
  labels: {severity: page, team: payments}
  annotations: {summary: "API 5xx ratio above 5%", runbook_url: "https://runbooks.example.com/api-5xx"}
```

**Why →** Golden signals map to what users feel. Infrastructure metrics are for diagnosis.

**Interview tip →** Name the four golden signals (latency, traffic, errors, saturation) and say which of them page.

</details>

### Q3. A "CPU above 90%" alert wakes someone at 3 a.m., but users saw no impact. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A cause-based alert without evidence of user impact. High CPU alone is often just efficient use.

**What I'd do →**
1. Remove the page. Keep CPU as a dashboard panel or a low-severity ticket alert with a long `for:` (30m+).
2. Replace it with a symptom alert: latency or error ratio on the service.
3. If CPU saturation does matter (throttling), alert on the effect: `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m]) > 0.25`.
4. Add the CPU panel to the service dashboard so it is a diagnosis step.

**Why →** Pages should reflect symptoms. Causes help you debug after you are already looking.

**Interview tip →** Say "I alert on symptoms, I dashboard causes."

</details>

### Q4. A new hire asks: "Should we page on symptoms or causes?" How do you answer with an example? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic Google SRE principle.

**What I'd do →** Explain with a database example:
1. Symptom: checkout API p99 latency > 2s for 10 minutes. Users are hurting right now, so page.
2. Cause: replica lag, connection pool at 95%, disk 85%. These explain the symptom but may never hurt users.
3. Page on the symptom. Use cause alerts as warnings or tickets, or surface them in the dashboard linked from the page.
4. Exception: a cause that is guaranteed to become a symptom soon (disk will be full in 4 hours) deserves a page.

**Why →** Symptom alerts catch problems you did not predict. Cause alerts only catch the failure modes you already imagined.

**Interview tip →** Mention predictive alerts (`predict_linear`) as the exception.

</details>

### Q5. Design burn-rate alerts for a 99.9% availability SLO over 30 days. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Error budget is 0.1%. A static error threshold either pages too late or too often. Use multi-window, multi-burn-rate alerts as in the Google SRE Workbook.

**What I'd do →**
1. Define the SLI as a recording rule per window: `sli:http_errors:ratio_rate5m`, `...1h`, `...6h`, `...3d`.
2. Page when burn rate is 14.4x over 1h AND 5m (2% of budget in 1 hour).
3. Page when 6x over 6h AND 30m (5% of budget in 6 hours).
4. Ticket when 1x over 3d AND 6h.

```yaml
- alert: ErrorBudgetFastBurn
  expr: sli:http_errors:ratio_rate1h > (14.4 * 0.001) and sli:http_errors:ratio_rate5m > (14.4 * 0.001)
  labels: {severity: page}
```

**Why →** The long window proves the burn is significant, the short window proves it is still happening so the alert resets quickly.

**Interview tip →** Know the 14.4 / 6 / 1 numbers and what fraction of budget each represents.

</details>

### Q6. Payments and platform teams share one Prometheus and one Alertmanager. How do you route alerts to the right team and severity? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Routing depends on consistent labels. Without a `team` label on every rule, routing becomes guesswork.

**What I'd do →**
1. Enforce `team` and `severity` labels on every alert rule (CI check with `promtool` plus a linter).
2. Build the route tree on those labels.

```yaml
route:
  receiver: default-slack
  group_by: [alertname, cluster, service]
  routes:
    - matchers: [team="payments", severity="page"]
      receiver: payments-pagerduty
    - matchers: [team="payments"]
      receiver: payments-slack
    - matchers: [team="platform", severity="page"]
      receiver: platform-pagerduty
```

3. Keep a catch-all receiver so unlabeled alerts are visible rather than lost.

**Why →** Label-based routing scales better than one route per alert name.

**Interview tip →** Mention `matchers` (the newer syntax) instead of the deprecated `match`/`match_re`.

</details>

### Q7. A database migration is scheduled for Saturday 02:00–04:00. How do you avoid pager noise without hiding real problems? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Expected noise on one scope, for a bounded time.

**What I'd do →**
1. Create a **silence** in Alertmanager scoped narrowly, e.g. `service="orders-db"` plus `env="prod"`, with start/end times and a comment with the change ticket.
2. Do not silence by `severity` alone, and never silence everything.
3. Keep end-user symptom alerts (checkout error ratio) active.

```bash
amtool silence add service="orders-db" env="prod" \
  --start="2026-11-14T02:00:00Z" --end="2026-11-14T04:00:00Z" \
  --author="aditya" --comment="CHG-2041 DB migration"
```

**Why →** A narrow, expiring silence is auditable. Inhibition is for alert-to-alert relationships, silences are for human-planned events.

**Interview tip →** Mention that silences expire, so nobody forgets to un-mute.

</details>

### Q8. One node dies and 40 alerts fire at once. How do you cut the noise structurally? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A single root cause is producing many dependent alerts (NodeDown, PodNotReady, TargetDown, HighLatency).

**What I'd do →**
1. Group by what matters so one notification carries many alerts: `group_by: [cluster, alertname]`.
2. Add inhibition so the root cause suppresses its symptoms.

```yaml
inhibit_rules:
  - source_matchers: [alertname="NodeDown"]
    target_matchers: [severity=~"warning|info"]
    equal: [node]
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: [alertname, cluster, service]
```

3. Tune `group_wait` (30s) and `group_interval` (5m) so correlated alerts land in one message.

**Why →** On-call needs the root cause on top, not 40 equal-looking messages.

**Interview tip →** `equal` must reference labels present on both source and target alerts.

</details>

### Q9. An alert flaps between firing and resolved every few minutes. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The expression hovers around the threshold, or the metric is noisy.

**What I'd do →**
1. Check the expression in the Prometheus graph view with a 6h range to see the oscillation.
2. Increase `for:` so a short blip does not fire (e.g. `for: 10m`).
3. Smooth the input: use `avg_over_time(x[10m])` or a longer `rate()` window.
4. Use `keep_firing_for: 15m` (Prometheus 2.42+) so a firing alert stays active briefly after the condition clears.
5. Consider hysteresis: different thresholds to fire and clear using two rules.

**Why →** `for:` filters short spikes, `keep_firing_for` prevents resolve/fire churn that spams receivers.

**Interview tip →** Distinguish `for` (delay before firing) from `keep_firing_for` (delay before resolving).

</details>

### Q10. How do you know if Prometheus or Alertmanager itself has silently died? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** If the alerting pipeline fails, no alerts fire, and silence looks like health. You need an external heartbeat.

**What I'd do →**
1. Add an always-firing **Watchdog** alert: `expr: vector(1)`.
2. Route it to a dead man's switch service (healthchecks.io, PagerDuty heartbeat, Opsgenie heartbeat, or a Grafana Cloud/Alertmanager heartbeat) with `repeat_interval: 1m`.
3. If the heartbeats stop, that external service pages you.
4. Also run a second, independent Prometheus that scrapes the first and alerts on `up{job="prometheus"} == 0`.

**Why →** The only reliable way to detect a dead alerter is something outside it that expects a regular signal.

**Interview tip →** The kube-prometheus-stack ships a Watchdog alert for exactly this purpose.

</details>

### Q11. A nightly backup job must complete every day, but when it fails to run there is no metric at all. How do you alert? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Absent data. A threshold alert on a missing series never fires.

**What I'd do →**
1. Have the job push `backup_last_success_timestamp_seconds` to the Pushgateway (or expose it from a CronJob exporter) on success.
2. Alert on staleness, not on the value.

```yaml
- alert: BackupNotRunIn26h
  expr: time() - backup_last_success_timestamp_seconds > 26 * 3600
  labels: {severity: page}
- alert: BackupMetricMissing
  expr: absent(backup_last_success_timestamp_seconds)
  for: 30m
```

3. For Kubernetes CronJobs with kube-state-metrics: `time() - kube_cronjob_status_last_successful_time > 93600`.

**Why →** "Last success timestamp" turns a missing event into a measurable age. `absent()` covers the case where the series itself disappears.

**Interview tip →** Say "alert on the absence of success, not the presence of failure."

</details>

### Q12. Nobody acknowledges a page for 10 minutes. How should escalation be configured? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Escalation policy is a safety net for a missed or slept-through page.

**What I'd do →**
1. Primary on-call gets the page, with push + SMS + phone call at increasing intervals.
2. After 5–10 minutes with no ack, escalate to the secondary on-call.
3. After another 10–15 minutes, escalate to the engineering manager or incident commander.
4. Keep escalation in PagerDuty/Opsgenie, not in Alertmanager. Alertmanager only decides who gets notified first.
5. Test the policy quarterly with a fire drill.

**Why →** Humans miss pages. A defined chain means a real outage is never waiting on one sleeping person.

**Interview tip →** Mention that ack timeouts are a team agreement tied to the SLO, not a global constant.

</details>

### Q13. What should every alert carry so the 3 a.m. engineer can act quickly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** An alert named `HighLatency` with no context costs ten minutes of orientation during an outage.

**What I'd do →** Standardize annotations:
1. `summary`: one line with the impacted service and value (`{{ $labels.service }} p99 is {{ $value | humanizeDuration }}`).
2. `description`: what it means for users, and likely causes.
3. `runbook_url`: step-by-step diagnosis and mitigation.
4. `dashboard_url`: pre-filtered Grafana link with `var-service={{ $labels.service }}`.
5. Labels: `severity`, `team`, `env`.

**Why →** The goal is to move straight from page to diagnosis. A good runbook cuts mean time to mitigate more than a clever alert.

**Interview tip →** Mention that an alert without a runbook should not be allowed to page.

</details>

### Q14. Prevent a disk-full outage on a log server before it happens. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A static "disk above 90%" alert gives too little time on fast-growing disks and too much noise on slow ones. Use rate-of-change.

**What I'd do →**
1. Use `predict_linear` to ask "will this fill within 4 hours?".

```yaml
- alert: DiskWillFillIn4h
  expr: |
    predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[1h], 4*3600) < 0
    and node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.4
  for: 15m
  labels: {severity: page}
```

2. Add a 24-hour prediction at warning severity (ticket).
3. Add a static 85% warning as a backstop.

**Why →** Time-to-full is what matters operationally. The second condition avoids paging on large, mostly-empty disks.

**Interview tip →** Mention inodes (`node_filesystem_files_free`) as the other way disks "fill".

</details>

### Q15. You wrote 15 new alert rules. How do you ship them without paging the team on day one? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** New alerts are untested guesses about thresholds. Roll them out like code.

**What I'd do →**
1. Unit test with `promtool test rules` using synthetic series.
2. Deploy with `severity: ticket` or route to a staging Slack channel for one to two weeks.
3. Backtest expressions on historical data in Grafana/Prometheus: how often would this have fired?
4. Review each firing: was it actionable? Adjust the threshold and `for:`.
5. Promote to `severity: page` only after the false-positive rate is acceptable.

**Why →** Shadow mode measures precision before you spend anyone's sleep.

**Interview tip →** Treat alerts as code: PR review, tests, staged rollout.

</details>

### Q16. How do you alert when an entire region fails, but not when a single pod restarts? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want to aggregate to the right level. Per-pod alerts are noise, aggregate alerts catch real loss.

**What I'd do →**
1. Alert on the aggregate success ratio per region: `sum by (region) (rate(http_requests_total{code!~"5.."}[5m])) / sum by (region) (rate(http_requests_total[5m])) < 0.95`.
2. Add a traffic-drop alert comparing the region to its own history, since a dead region may stop emitting errors altogether.
3. Use an external blackbox probe per region, so you do not depend on the region's own monitoring.
4. Pod restarts become a dashboard panel or a ticket with a rate threshold.

**Why →** A region outage can look like "no data", so aggregate and external views are both required.

**Interview tip →** Mention that probes must run from outside the failure domain.

</details>

### Q17. Design alerting for Kafka consumer lag without false positives during traffic bursts. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Absolute lag is misleading: 1M messages behind with a fast consumer clears in seconds, 1000 behind with a stuck consumer never clears.

**What I'd do →**
1. Alert on **lag that is not decreasing**, not lag size. With kafka_exporter or Burrow-style metrics:

```yaml
expr: |
  sum by (consumergroup, topic) (kafka_consumergroup_lag) > 10000
  and
  deriv(sum by (consumergroup, topic) (kafka_consumergroup_lag)[15m:1m]) >= 0
for: 15m
```

2. Add **time-lag** (seconds behind) if exposed, which maps better to SLOs.
3. Alert if consumption rate is zero while lag exists.

**Why →** Growth or stagnation means consumers cannot keep up. A shrinking backlog is a recovering system.

**Interview tip →** Say "I alert on lag trend and time-lag, not the raw count."

</details>

### Q18. Your Alertmanager is a single pod. How do you make it highly available without duplicate notifications? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One Alertmanager is a single point of failure for the entire alert path.

**What I'd do →**
1. Run 3 Alertmanager replicas in a gossip cluster (`--cluster.peer=alertmanager-0:9094`, etc.). On Kubernetes, a StatefulSet with a headless service, or the Prometheus Operator `replicas: 3`.
2. Configure **every** Prometheus to send alerts to **all** Alertmanagers, not through a load balancer.
3. The cluster deduplicates using gossip of notification log entries, so each notification is sent once.
4. Persist `/alertmanager` data so silences survive restarts.
5. Monitor `alertmanager_cluster_members` and `alertmanager_notifications_failed_total`.

**Why →** Alertmanager prefers availability: during a partition you may get duplicates, but never lost alerts.

**Interview tip →** A common mistake is putting a load balancer in front, which breaks the deduplication model.

</details>

### Q19. How do you measure whether your on-call rotation is healthy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** On-call health is measurable. Burnout shows up in numbers before it shows up in resignations.

**What I'd do →**
1. Track pages per shift, out-of-hours pages, and pages per week per person. Google's guideline is at most two incidents per 12-hour shift.
2. Track the **actionable ratio**: pages that led to human action versus those auto-resolved or ignored.
3. Track time-to-ack and time-to-mitigate.
4. Review the top five noisiest alerts every week and assign owners to fix or delete them.
5. Check balance: no single person absorbing most night pages.

**Why →** What you do not measure silently degrades. Noisy rotations lose people.

**Interview tip →** Mention PagerDuty analytics or Opsgenie reports as the data source.

</details>

### Q20. A platform with 60 teams needs alert ownership so every alert reaches an owner. How do you design it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** At scale, a central platform team cannot own every alert. You need self-service with guardrails.

**What I'd do →**
1. Derive ownership from a service catalog (Backstage, or a labels convention such as `team` on every Deployment and namespace).
2. Teams ship their own `PrometheusRule` objects (or Mimir/Loki ruler rules) in their repos.
3. CI enforces required labels and annotations, with a `promtool check rules` plus custom policy (OPA/conftest).
4. Alertmanager config is generated: one route per team from the catalog, or use per-tenant Alertmanager in Mimir.
5. A default catch-all route goes to the platform team with a "no owner" tag to drive cleanup.

**Why →** Ownership attached to the code means alerts follow the service, not a spreadsheet.

**Interview tip →** Say "you build the paved road, teams drive on it."

</details>

### Q21. How would you monitor a public API from the outside? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Internal metrics cannot see DNS, CDN, certificate or routing failures. External probes can.

**What I'd do →**
1. Deploy **blackbox_exporter** in at least two locations outside your cluster.
2. Define a module that checks status, TLS and latency.
3. Scrape it through a probe:

```yaml
scrape_configs:
  - job_name: blackbox-http
    metrics_path: /probe
    params: {module: [http_2xx]}
    static_configs: [{targets: ["https://api.example.com/health"]}]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

4. Alert on `probe_success == 0` for 2 minutes from multiple locations.

**Why →** The user's view of your service is what you must measure first.

**Interview tip →** Alert only when more than one probe location fails, to avoid a single bad prober paging you.

</details>

### Q22. How do you prevent a surprise TLS certificate expiry? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Expired certificates are a classic self-inflicted outage. They are entirely predictable.

**What I'd do →**
1. Probe endpoints with blackbox_exporter, which exposes `probe_ssl_earliest_cert_expiry`.
2. If you use cert-manager, scrape `certmanager_certificate_expiration_timestamp_seconds`.
3. Alert in tiers:

```yaml
- alert: CertExpiresIn21d
  expr: (probe_ssl_earliest_cert_expiry - time()) < 21 * 86400
  labels: {severity: ticket}
- alert: CertExpiresIn5d
  expr: (probe_ssl_earliest_cert_expiry - time()) < 5 * 86400
  labels: {severity: page}
```

4. Alert on renewal failure (cert-manager `Ready=False`) because that is the earlier signal.

**Why →** Early warnings leave time for a human to fix a broken renewal.

**Interview tip →** Alert on the renewal pipeline failing, not only on the date.

</details>

### Q23. The security team wants alerts on repeated failed logins found only in application logs. How do you do it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The signal exists in logs, but alerting is cheaper and faster on metrics.

**What I'd do →**
1. With Loki, use the **ruler** with a LogQL metric query:

```yaml
- alert: BruteForceLogin
  expr: sum by (client_ip) (count_over_time({app="auth"} |= "login failed" | logfmt [5m])) > 50
  for: 2m
  labels: {severity: page, team: security}
```

2. Alternatively expose a counter from the app (`auth_failures_total{reason}`), which is better long term.
3. For Elastic, use Watcher or Kibana alerting rules.
4. Route to the security receiver and include client IP and user in annotations.

**Why →** Rules over logs work, but a first-class metric is cheaper and does not depend on log parsing.

**Interview tip →** Mention cardinality: per-IP labels are fine in an alert expression but not as a stored metric label.

</details>

### Q24. During a cascading outage with 200 firing alerts, the incident commander says "silence everything." What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The instinct is understandable, but blanket silences hide new failures and recovery signals.

**What I'd do →**
1. Do not silence globally. Propose a targeted silence for alerts already acknowledged by the IC, using a matcher on `incident` or the known failing `service` set.
2. Mark one dashboard or war-room view as the source of truth, so no one reads raw alert streams.
3. Rely on grouping and inhibition rules to collapse noise (critical suppresses warning for the same cluster).
4. Keep paging only for new symptoms outside the incident scope.
5. Expire silences at a fixed time and review them in the post-incident review.

**Why →** You need recovery signals and any independent second failure to stay visible.

**Interview tip →** Say "silence by scope and time, never by 'everything'."

</details>

### Q25. A service normally handles 500 rps in daytime and 5 rps overnight. How do you detect a silent traffic loss? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A fixed floor (`rps < 100`) would page every night. Traffic drops are real incidents (broken DNS, routing, upstream auth) and produce no errors.

**What I'd do →** Compare current traffic with the same time last week and use a ratio:

```yaml
- alert: TrafficDropVsLastWeek
  expr: |
    sum(rate(http_requests_total{job="api"}[10m]))
      / sum(rate(http_requests_total{job="api"}[10m] offset 1w)) < 0.4
    and sum(rate(http_requests_total{job="api"}[10m] offset 1w)) > 20
  for: 10m
  labels: {severity: page}
```

1. The second condition avoids dividing tiny numbers overnight.
2. Average several previous weeks to avoid learning from a past outage.
3. Cross-check with the blackbox probe.

**Why →** Seasonality-aware comparisons detect "too quiet" without a static threshold.

**Interview tip →** Mention holidays and deploys as known false-positive sources, and add a silence process.

</details>

## Prometheus and PromQL at Scale

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Prometheus memory has grown to 60 GB and keeps climbing. How do you find the cause? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prometheus memory scales mainly with active series (head block cardinality). A sudden climb means a label explosion.

**What I'd do →**
1. Open `/tsdb-status` in the UI (or the API) to see top metrics by series count and top labels by value count.
2. Query the series growth: `prometheus_tsdb_head_series` and `rate(prometheus_tsdb_head_series_created_total[5m])`.
3. Find the top offenders:

```promql
topk(10, count by (__name__) ({__name__=~".+"}))
```

4. Drop or relabel the bad label using `metric_relabel_configs` and `labeldrop`/`drop`, or fix the app.
5. Set `sample_limit`/`label_limit` per scrape job as a guardrail.

**Why →** Memory is roughly proportional to active series, so you must find which metric or label has exploded.

**Interview tip →** Mention `promtool tsdb analyze` for offline inspection.

</details>

### Q27. A developer added `user_id` as a metric label. What is the problem and what do you advise? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unbounded label values create one time series per user. That is a cardinality explosion that raises memory, CPU and storage cost.

**What I'd do →**
1. Remove the label from the metric.
2. Use bounded labels only: `method`, `route` template (`/users/:id`, not `/users/42`), `status_class`.
3. For per-user analysis use logs or traces, which handle high cardinality, or use exemplars to link a metric to a trace ID.
4. Add `label_limit` and `sample_limit` on the scrape job and a CI review check on new metrics.

**Why →** Each unique label combination is a separate series. A million users times 10 routes is 10 million series.

**Interview tip →** Rule of thumb: metrics for aggregates, logs and traces for individuals.

</details>

### Q28. Your dashboard shows the average of per-pod p99 latencies. Why is it wrong, and how do you compute a correct fleet-wide p99? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Percentiles cannot be averaged. A p99 of 50 ms on one pod and 2 s on another does not average to anything meaningful.

**What I'd do →** Aggregate **histogram buckets** first, then compute the quantile:

```promql
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

1. Always keep `le` in the `by` clause.
2. Do not use `summary` quantiles for cross-instance aggregation, since they are pre-computed per instance.
3. Make a recording rule for the expression for faster dashboards.

**Why →** Histogram buckets are counters and are additive across instances, so summing them gives the correct combined distribution.

**Interview tip →** "Summaries can't be aggregated; histograms can." Say it clearly.

</details>

### Q29. Choose histogram bucket boundaries for a service with a 300 ms latency SLO. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `histogram_quantile` interpolates inside a bucket, so accuracy depends on bucket layout. Default buckets (0.005 to 10) waste resolution near your SLO.

**What I'd do →**
1. Put a bucket **exactly at 0.3** so the SLO ratio is exact: `rate(..._bucket{le="0.3"}) / rate(..._count)`.
2. Add dense buckets around the target: `0.05, 0.1, 0.2, 0.25, 0.3, 0.35, 0.4, 0.5, 0.75, 1, 2.5, 5`.
3. Keep the number of buckets modest (10–15), since each is a series multiplied by label combinations.
4. Consider **native histograms** (Prometheus 3.x) which remove manual bucket choices.

**Why →** SLO compliance should come from exact bucket counts, not interpolation guesses.

**Interview tip →** Mention that changing bucket boundaries later breaks aggregation across old and new series.

</details>

### Q30. You have 500 short-lived batch jobs that finish before Prometheus can scrape them. How do you capture their metrics? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pull-based scraping misses jobs that live for seconds. This is the intended use case for the Pushgateway.

**What I'd do →**
1. Run a Pushgateway and have each job push on completion with a grouping key such as `job` and `instance`.

```bash
echo "batch_last_success_timestamp_seconds $(date +%s)" | \
  curl --data-binary @- http://pushgateway:9091/metrics/job/nightly_etl/instance/etl-1
```

2. Scrape the gateway with `honor_labels: true`.
3. Push only job-level outcomes (duration, success timestamp, rows processed).
4. Delete stale groups when a job is retired, because the gateway keeps them forever.
5. For jobs with a lifetime longer than a scrape interval, expose a normal `/metrics` instead.

**Why →** The Pushgateway is a metric cache for batch results, not a general push path.

**Interview tip →** Know its drawbacks: single point of failure, no `up` health signal per job, stale data.

</details>

### Q31. Your EC2 fleet autoscales constantly. How do you keep Prometheus targets up to date? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static target lists go stale. Use service discovery plus relabeling to select and label targets.

**What I'd do →**
1. Use `ec2_sd_configs` with an IAM role allowing `ec2:DescribeInstances`.
2. Filter and label with relabeling:

```yaml
scrape_configs:
  - job_name: node
    ec2_sd_configs:
      - region: ap-south-1
        port: 9100
        filters:
          - name: tag:Monitoring
            values: ["enabled"]
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_availability_zone]
        target_label: az
```

3. Alert on `up == 0` for 5 minutes, since instances that are terminated drop out of discovery and do not alert.

**Why →** Discovery reflects the real fleet and removes manual edits.

**Interview tip →** Mention `file_sd` for custom sources and `kubernetes_sd` for pods and services.

</details>

### Q32. You run 3 Kubernetes clusters. Compare federation, remote_write and Thanos sidecar for a global view. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need cross-cluster queries and long retention. Each approach differs in cost, complexity and failure modes.

**What I'd do →** Compare:
1. **Federation**: a global Prometheus scrapes `/federate` from each cluster. Simple, but only works for a small, selected set of aggregated series. Not for raw data.
2. **remote_write** to Mimir/Cortex/Thanos Receive/VictoriaMetrics: central ingestion, global query, long retention, multi-tenancy. Needs a durable central system and network egress.
3. **Thanos Sidecar + Querier**: Prometheus stays local, blocks upload to object storage, Querier fans out. No central write path, but you operate Thanos components.
4. I would pick remote_write to Mimir for 3+ clusters needing HA and multi-tenancy, and sidecar when I want to keep local autonomy.

**Why →** Federation is a stopgap, while the others are real long-term architectures.

**Interview tip →** State trade-offs first, then your choice.

</details>

### Q33. What scrape interval would you choose, and how does it affect `rate()` windows? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shorter intervals give finer resolution but cost more series samples, CPU and storage.

**What I'd do →**
1. Default to **15s** for application and infrastructure metrics, and 30–60s for low-value exporters (cloud APIs, slow databases).
2. The `rate()` range must contain **at least two samples**. As a rule use **4x the scrape interval**: 15s scrape means `[1m]` minimum, `[5m]` for alerts.
3. Keep one global `scrape_interval` and override per job only with a reason.
4. Match Grafana's `$__rate_interval` variable, which accounts for the interval.

**Why →** If the window contains fewer than two samples, `rate()` returns nothing and graphs show gaps.

**Interview tip →** Mention that storage cost scales linearly with scrape frequency.

</details>

### Q34. You need CPU usage per team, but metrics only have namespace labels and teams are in pod labels. How do you join them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need to enrich one metric with labels from another using a many-to-one join.

**What I'd do →** Use `kube_pod_labels` from kube-state-metrics (value always 1) and `group_left`:

```promql
sum by (label_team) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
  * on (namespace, pod) group_left (label_team)
  kube_pod_labels
)
```

1. kube-state-metrics must allow the label: `--metric-labels-allowlist=pods=[team]`.
2. Consider a recording rule since this join is expensive.
3. Alternatively label at the namespace level and use `kube_namespace_labels`.

**Why →** `group_left` copies labels from the "one" side to the "many" side without changing the value.

**Interview tip →** Remember the multiplier trick: info metrics have value 1, so multiplication keeps the original value.

</details>

### Q35. When should you create recording rules, and how do you name them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Expensive or frequently reused queries slow dashboards and alert evaluation.

**What I'd do →**
1. Create recording rules for aggregations used by dashboards or alerts, especially `histogram_quantile` and cross-label sums over many series.
2. Use the convention `level:metric:operations`:

```yaml
groups:
  - name: api.rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

3. Keep labels the aggregation preserves in the `level` part. The name should say what was aggregated.
4. Do not record everything. Each rule adds series and evaluation cost.

**Why →** Pre-computation trades small storage for fast, consistent queries.

**Interview tip →** Alerts should use recording rules for SLI expressions, so dashboards and alerts agree.

</details>

### Q36. What do you monitor about Prometheus itself? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** If the monitor is unhealthy, everything downstream is suspect.

**What I'd do →** Watch these and alert on thresholds:
1. **Ingestion**: `rate(prometheus_tsdb_head_samples_appended_total[5m])`, series count `prometheus_tsdb_head_series`.
2. **Scrape health**: `up`, `scrape_duration_seconds`, `prometheus_target_scrapes_exceeded_sample_limit_total`.
3. **Rule evaluation**: `prometheus_rule_evaluation_failures_total`, `prometheus_rule_group_last_duration_seconds` versus the interval.
4. **Remote write**: `prometheus_remote_storage_samples_pending`, `..._failed_total`, shard lag.
5. **Storage**: `prometheus_tsdb_compactions_failed_total`, disk free for the data dir.
6. **Alert delivery**: `prometheus_notifications_dropped_total`.

**Why →** These cover the whole path: collect, store, evaluate, notify.

**Interview tip →** Rule groups that take longer than their interval silently skip evaluations.

</details>

### Q37. Prometheus takes 20 minutes to restart after a crash. Why, and how do you reduce it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** On start, Prometheus replays the WAL and loads the head block. A large head and a large WAL slow this down.

**What I'd do →**
1. Check logs: `msg="WAL replay completed"` with duration, `Replaying WAL, this may take a while`.
2. Reduce series count (see cardinality fixes). Less head data means a shorter replay.
3. Keep the WAL on fast disk (NVMe/gp3 with enough IOPS).
4. Enable `--storage.tsdb.wal-compression` (default on in recent versions) to shrink the WAL.
5. Run two replicas so one stays up while the other restarts, and use a readiness gate on `/-/ready`.
6. Shard if a single instance is too large.

**Why →** Replay time is proportional to the WAL size, which is proportional to ingested samples since the last head compaction.

**Interview tip →** Do not use `kill -9` or OOM-kill loops: they leave long WALs to replay.

</details>

### Q38. One Prometheus cannot scrape 5000 targets. How do you scale scraping horizontally? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A single Prometheus has CPU, memory and scrape-time limits. Scale by splitting the target set, not by adding replicas that scrape the same targets.

**What I'd do →**
1. **Functional sharding**: one Prometheus per team, environment or job type.
2. **Hash-based sharding** with relabeling:

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 4
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ^1$        # this instance keeps shard 1
    action: keep
```

3. The Prometheus Operator supports `shards: 4` natively.
4. Put a global query layer on top (Thanos Querier, Mimir, VictoriaMetrics) because each shard sees only part of the data.
5. Run HA pairs per shard.

**Why →** Hashing gives an even, stable split. Global queries then need a layer that merges shards.

**Interview tip →** Before sharding, check whether cardinality, not target count, is the real problem.

</details>

### Q39. A new app team wants to be monitored on your Kubernetes cluster using the Prometheus Operator. What do you give them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** With the Operator, teams self-serve using CRDs. They do not edit `prometheus.yml`.

**What I'd do →**
1. The app exposes `/metrics` on a named port in its Service.
2. The team creates a `ServiceMonitor` selecting that Service.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: {name: checkout, labels: {release: kube-prometheus-stack}}
spec:
  selector: {matchLabels: {app: checkout}}
  endpoints: [{port: http-metrics, interval: 30s}]
```

3. They add alerts as a `PrometheusRule`.
4. Confirm the target appears under Status → Targets.

**Why →** The Operator reconciles CRDs into Prometheus config, so teams get safe self-service.

**Interview tip →** The `release` label must match the Prometheus `serviceMonitorSelector`, which is the usual first mistake.

</details>

### Q40. Which metrics would you instrument in a new backend service? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use RED for request-driven services and USE for resources.

**What I'd do →**
1. **Rate**: `http_requests_total{method,route,status}` counter.
2. **Errors**: derive from the status label, plus business errors such as `payment_failures_total{reason}`.
3. **Duration**: `http_request_duration_seconds` histogram, using route templates.
4. **Saturation**: in-flight requests gauge, worker pool queue length, DB pool usage.
5. **Dependencies**: client-side metrics for each downstream call (latency, errors).
6. Runtime metrics (GC, goroutines or threads, memory) come free from the client library.

**Why →** Client-side dependency metrics tell you whether the problem is yours or downstream.

**Interview tip →** Mention that you instrument on the boundary: incoming requests and outgoing calls.

</details>

### Q41. Write the PromQL for the HTTP error ratio of a service, and explain the pitfalls. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ratios of rates are the standard SLI form. Pitfalls are in aggregation and zero traffic.

**What I'd do →**

```promql
sum(rate(http_requests_total{job="api",code=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="api"}[5m]))
```

1. Aggregate numerator and denominator **before** dividing, to avoid dividing per-instance series that do not match.
2. If there are no 5xx at all, the numerator is empty and the division returns nothing. Use `or vector(0)` on the numerator if you need a value: `(sum(rate(...{code=~"5.."}[5m])) or vector(0)) / sum(rate(...[5m]))`.
3. With zero traffic the result is NaN or empty, so guard with an `and` on a minimum rate.

**Why →** Empty vectors and NaN silently break alerts and dashboards.

**Interview tip →** Call out the empty-numerator case unprompted.

</details>

### Q42. A deploy restarts pods and counters reset. Will `rate()` show a drop? What about `increase()`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Counters reset to zero on process restart. Prometheus handles that for you.

**What I'd do →**
1. `rate()` and `increase()` detect a decrease as a reset and treat the post-reset value as an increase from zero. No negative spike.
2. Only apply them to counters. For gauges use `deriv()` or `delta()`.
3. `increase(x[1h])` is `rate(x[1h]) * 3600`, extrapolated at the window edges, so it may return non-integers like 9.97 for a count that is really 10.
4. Never graph a raw counter, as its value is meaningless on its own.

**Why →** Counters make restarts safe and make scraping losses recoverable, which is why they are preferred over gauges for totals.

**Interview tip →** Mention that `irate` is for volatile fast graphs only. Do not use it in alerts.

</details>

### Q43. A legacy app has no `/metrics` endpoint and cannot be changed. How do you get metrics out of it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Without code access you must observe the app from outside or translate its existing outputs.

**What I'd do →**
1. If it writes logs with useful lines, use **mtail** or **grok_exporter** to derive counters and histograms from the log stream.
2. If it emits StatsD or JMX, use **statsd_exporter** or the **JMX exporter** as a Java agent.
3. If it has a management API or DB, write a small exporter or use the matching community exporter.
4. Otherwise monitor from outside with blackbox probes and process metrics from `node_exporter` or `process-exporter`.
5. Document the gap and plan native instrumentation.

**Why →** Translating existing signals is faster than modifying legacy code and gives metrics right away.

**Interview tip →** Prefer the lowest-effort source of truth, and record the tech debt.

</details>

### Q44. How do you monitor a PostgreSQL primary and replicas? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use `postgres_exporter` and watch for saturation, replication health and query behavior.

**What I'd do →**
1. Run `postgres_exporter` with a dedicated read-only role in the `pg_monitor` group.
2. Key metrics and alerts:
   - Connections: `pg_stat_activity_count` versus `pg_settings_max_connections` (alert above 80%).
   - Replication lag: `pg_replication_lag_seconds` or `pg_stat_replication` LSN differences.
   - Transactions: `rate(pg_stat_database_xact_rollback[5m])`, deadlocks.
   - Cache hit ratio, long-running transactions, and `pg_stat_database_tup_*`.
   - Disk, and `pg_wal` growth (a stuck replication slot fills the disk).
3. Add `pg_stat_statements` for slow-query visibility.

**Why →** Connection exhaustion, replication lag and WAL retention are the common ways Postgres takes down a service.

**Interview tip →** Mention that an inactive replication slot retaining WAL is a classic disk-full cause.

</details>

### Q45. How do you secure Prometheus scraping across namespaces in a cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Metrics endpoints can reveal internals. Scraping must be authenticated and limited by network and RBAC.

**What I'd do →**
1. Give Prometheus a ServiceAccount with a ClusterRole limited to `get/list/watch` on pods, services, endpoints, and `nonResourceURLs: ["/metrics"]`.
2. Use **TLS and bearer tokens** for sensitive targets (kubelet, API server, kube-rbac-proxy in front of app metrics).
3. Use **NetworkPolicies**: allow ingress to metrics ports only from the monitoring namespace.
4. Restrict which namespaces the Operator watches (`serviceMonitorNamespaceSelector`).
5. Do not expose the Prometheus UI publicly. Put it behind an authenticated proxy (OAuth2 Proxy).

**Why →** Metrics can leak topology and business numbers, and Prometheus holds powerful discovery credentials.

**Interview tip →** Mention that Prometheus itself has no built-in user authentication on its UI, so you front it.

</details>

### Q46. A third-party exporter emits 40,000 series you never use. How do you drop them at ingest? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You are paying memory and storage for series nobody queries. Drop them after scrape, before storage.

**What I'd do →**
1. Use `metric_relabel_configs`, which runs on scraped samples.

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: "go_gc_.*|http_request_size_bytes_.*|unused_metric_.*"
    action: drop
  - regex: "pod_template_hash|controller_revision_hash"
    action: labeldrop
```

2. Verify with `prometheus_target_scrape_pool_...` and `scrape_samples_post_metric_relabeling`.
3. Prefer an allowlist (`action: keep`) for noisy exporters.
4. Confirm no dashboards or alerts use the dropped names first.

**Why →** Dropping in relabeling is cheaper than storing and ignoring.

**Interview tip →** `relabel_configs` is for targets, `metric_relabel_configs` is for samples, a favorite quiz question.

</details>

### Q47. How do exemplars help you jump from a slow-latency graph to a specific trace? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A p99 spike tells you "something is slow" but not which request. Exemplars attach a trace ID to a specific histogram observation.

**What I'd do →**
1. Instrument the app so histogram observations include the trace ID (OpenTelemetry or Prometheus client libraries with exemplar support). Expose in OpenMetrics format.
2. Start Prometheus with `--enable-feature=exemplar-storage` and scrape using OpenMetrics.
3. In Grafana, set the Prometheus data source's exemplar link to the Tempo data source using the `trace_id` label.
4. In a latency panel, enable **Exemplars**. Click a dot to open the trace.

**Why →** You move from aggregate to individual evidence in one click, shrinking the diagnosis time.

**Interview tip →** Exemplars are sparse samples, not full coverage. They are pointers, not a replacement for tracing.

</details>

### Q48. Your team wants to move to native histograms. What changes and what are the risks? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Native histograms store a full distribution in a single series with exponential buckets. They reduce cardinality and give better resolution, but are a different data model.

**What I'd do →**
1. Check client library support and enable it in the app (e.g. client_golang with native histogram options).
2. Enable ingestion on Prometheus (`--enable-feature=native-histograms`, version-dependent) and scrape using protobuf, or use the OpenTelemetry exponential histogram path.
3. Run **dual exposure** (classic plus native) during migration, then switch queries to `histogram_quantile(0.99, sum(rate(x[5m])))` without `le`.
4. Check that remote storage (Mimir, Thanos, Grafana Cloud) supports them.
5. Update recording rules, alerts, and dashboards.

**Why →** Gradual migration avoids blind spots in alerts that depend on `_bucket` series.

**Interview tip →** Mention the win: fewer series, no manual bucket tuning.

</details>

### Q49. A Grafana dashboard with a 30-day range takes 40 seconds to load. How do you speed it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long ranges over raw high-cardinality series force Prometheus to read huge numbers of samples.

**What I'd do →**
1. Replace heavy panel queries with **recording rules** that pre-aggregate (`job:http_requests:rate5m`).
2. Use `$__rate_interval` and let Grafana set a sensible `step`, or set Min step to 5m for long ranges.
3. Reduce the series each panel returns with `sum by` and `topk`.
4. For long retention, use downsampled data (Thanos Compactor 5m/1h resolutions, Mimir with recording rules).
5. Enable query-frontend caching and splitting by interval (Thanos Query Frontend, Mimir).

**Why →** Query cost scales with series x samples x range. Reduce each.

**Interview tip →** Mention the Query Inspector in Grafana to see the real query time and number of series.

</details>

### Q50. Workloads in private VPCs cannot be scraped from your central Prometheus. What architecture do you use? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pull from a central location is blocked by network boundaries. Move collection to the edge and push outward.

**What I'd do →**
1. Run a lightweight local collector per VPC or cluster: **Prometheus in agent mode** (`--enable-feature=agent`, or `--agent` in Prometheus 3.x), Grafana Alloy, or the OpenTelemetry Collector.
2. It scrapes locally and **remote_writes** to a central Mimir/Thanos Receive/VictoriaMetrics over outbound HTTPS only.
3. Authenticate with per-tenant tokens or mTLS, and add `external_labels` (`cluster`, `region`).
4. Size the WAL and queue so a network outage of a few hours does not drop data (`queue_config.capacity`, `max_shards`).
5. Alert on `prometheus_remote_storage_samples_failed_total`.

**Why →** Outbound-only egress satisfies strict firewall policies and the agent keeps local state minimal.

**Interview tip →** Agent mode has no local querying or alerting, so keep rule evaluation central (Mimir ruler).

</details>


---

## Logging Pipelines

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Developers SSH into servers and `grep` log files to debug. You are asked to centralise logging. Where do you start? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No central logs means slow debugging, lost evidence when instances die, and too many people with SSH access.

**What I'd do →**
1. Pick a backend that matches budget and query needs: Loki (cheap, label-indexed), Elasticsearch/OpenSearch (full-text), or a managed service (CloudWatch Logs, Grafana Cloud).
2. Deploy one shipper per node: Grafana Alloy, Fluent Bit or Vector, tailing `/var/log/*.log` and the journal.
3. Attach consistent labels: `env`, `service`, `host`, `cluster`.
4. Set a retention policy (e.g. 14 days hot, 90 days archived to S3).
5. Give developers Grafana Explore or Kibana and then remove routine SSH access.

**Why →** Logs survive instance termination, everyone searches in one place, and you can build alerts on top.

**Interview tip →** Say what you would *not* do first: shipping everything at DEBUG level on day one.

</details>

### Q52. Your app writes free-text logs like `User 42 paid 19.99 in 340ms`. Why push for structured logging, and how? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Free text forces regex parsing that breaks every time a developer edits a message. Queries become fragile.

**What I'd do →**
1. Agree a JSON log schema: `ts`, `level`, `msg`, `service`, `trace_id`, `span_id`, plus event-specific fields.
2. Switch to a structured logger (zap/slog in Go, structlog in Python, pino in Node, Logback JSON encoder in Java).
3. Inject `trace_id` automatically from the OpenTelemetry context.
4. Parse once in the shipper (`json` stage) and query fields directly.

```json
{"ts":"2026-03-02T10:15:01Z","level":"info","msg":"payment completed","user_id":42,"amount":19.99,"duration_ms":340,"trace_id":"4bf92f3577b34da6"}
```

**Why →** Fields are stable contracts; message text is not.

**Interview tip →** Mention that `user_id` belongs in the log body, never as a Loki label.

</details>

### Q53. Loki ingestion is failing with `stream limit exceeded` and the ingester memory is exploding. What happened? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A high-cardinality label was added to log streams. Every unique label set is a new stream, so Loki is creating millions of tiny streams.

**What I'd do →**
1. Find the offending label: `logcli series '{}' --analyze-labels` or check `loki_ingester_memory_streams` per tenant.
2. Remove labels like `request_id`, `user_id`, `pod_ip`, `status_code` from the shipper config; keep them in the log line.
3. Query them at read time with `| json | user_id="42"` or structured metadata (Loki 3.x).
4. Keep `max_global_streams_per_user` as a guardrail rather than raising it blindly.

**Why →** Loki indexes labels only. Few labels with high volume per stream is the design point.

**Interview tip →** Rule of thumb: labels describe *where* a log came from, not *what* it says.

</details>

### Q54. Fluent Bit pods on busy nodes are dropping logs during traffic spikes. How do you make the pipeline lossless? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The in-memory buffer fills when the output is slower than input, and Fluent Bit pauses or drops the input.

**What I'd do →**
1. Enable filesystem buffering: `storage.path /var/fluent-bit/state`, `storage.type filesystem` on inputs, and `storage.total_limit_size` on outputs.
2. Set `Mem_Buf_Limit` on tail inputs and keep a DB file (`DB /var/fluent-bit/state/tail.db`) so offsets survive restarts.
3. Increase output `Workers` and set `Retry_Limit` sensibly (or `no_limits` for critical streams).
4. Give the DaemonSet enough CPU; check `fluentbit_output_retries_failed_total` and `fluentbit_input_bytes_total`.
5. If the backend is the bottleneck, put Kafka between collectors and the backend.

**Why →** Disk buffering absorbs bursts and backend outages without data loss.

**Interview tip →** Distinguish "back-pressure" (slow down input) from "drop" (lose data) and say which you chose.

</details>

### Q55. The Elasticsearch cluster for logs goes red every Monday after the weekend batch jobs. How do you stabilise it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Likely too many small shards, unbalanced hot nodes, or disk watermarks being hit during a burst.

**What I'd do →**
1. Check `GET _cluster/health`, `GET _cat/shards?v&s=store`, and `GET _cluster/allocation/explain` for unassigned shards.
2. Look at disk: the default `cluster.routing.allocation.disk.watermark.high` is 90%; flood stage at 95% makes indices read-only.
3. Use ILM with rollover by size (`max_primary_shard_size: 50gb`) instead of daily indices for tiny services.
4. Move to hot/warm/cold tiers with node attributes, and force-merge then shrink warm indices.
5. Put a buffer (Kafka) in front so ingest spikes do not hit ES directly.

**Why →** Shard count and disk headroom drive most ES instability. Thousands of 1 GB shards waste heap.

**Interview tip →** Quote a target: keep shards in the 10–50 GB range and under ~20 shards per GB of heap.

</details>

### Q56. Finance says the log bill doubled in a quarter. How do you cut it without losing debugging ability? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Volume grew, retention is uniform, and nobody owns cost per service.

**What I'd do →**
1. Measure bytes per `service` and `level` for a week; usually a few services emit most of the volume.
2. Drop DEBUG in production by default; sample repeated INFO lines (e.g. health checks, access logs for 200s).
3. Filter at the shipper (`grep`/`drop` processors in Fluent Bit, Vector `filter`, Alloy `stage.drop`).
4. Tier retention: 7 days hot for most, 30 days for audit-relevant, then archive to S3 Glacier.
5. Publish a per-team log cost dashboard to create ownership.

**Why →** Most log spend is low-value noise; cutting at the source is cheapest.

**Interview tip →** Never drop errors or audit logs to save money; say so explicitly.

</details>

### Q57. A Java stack trace shows up in Kibana as 40 separate log events. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The shipper treats every newline as a new event, so multi-line exceptions are split.

**What I'd do →**
1. Best fix: log in JSON so the stack trace is a single escaped field.
2. If you cannot change the app, configure multi-line parsing in the shipper. Fluent Bit:

```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    multiline.parser  docker, cri, java
```

3. In Vector use a `reduce` transform or `multiline` on the file source with a start pattern like `^\d{4}-\d{2}-\d{2}`.
4. Verify with a test exception before rolling out.

**Why →** One exception = one event makes counting and alerting on errors possible.

**Interview tip →** Mention that container runtimes split long lines at 16 KB, which needs the `cri`/`docker` parser to rejoin.

</details>

### Q58. Developers log full request bodies and you find credit card numbers and emails in the log store. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a data-protection incident (PCI DSS / GDPR), not just a logging bug.

**What I'd do →**
1. Raise it with security, scope which indices, buckets and backups contain the data, and restrict access immediately.
2. Stop the source: remove body logging or add an allow-list of safe fields in the app.
3. Add redaction in the pipeline as a safety net (Vector `remap` with `redact()`, Fluent Bit Lua filter, OTel Collector `redaction` or `transform` processor).
4. Delete or reindex affected data, including archives and snapshots, per the retention and legal guidance.
5. Add a CI check or log linter and periodic scanning for PAN/email patterns.

**Why →** Logs are often less protected than databases and copied into many places.

**Interview tip →** Lead with containment and notification, then the technical fix.

</details>

### Q59. You run hundreds of Kubernetes nodes. Choose between sidecar shippers and a node-level DaemonSet. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Most apps log to stdout; a few legacy apps write to files inside the container.

**What I'd do →**
1. Default to a **DaemonSet** (Alloy, Fluent Bit, Vector) reading `/var/log/pods` and enriching with Kubernetes metadata.
2. Use a **sidecar** only for apps that write to files and cannot change, sharing an `emptyDir` volume.
3. Make apps log to stdout/stderr as the standard in the platform guidelines.
4. Set resource requests and a PriorityClass on the DaemonSet so it is not evicted first.

**Why →** One agent per node is cheaper and simpler than one per pod; sidecars multiply CPU/memory and config.

**Interview tip →** Note that kubelet rotates container logs (`containerLogMaxSize`), so a slow shipper can miss rotated files.

</details>

### Q60. An alert should fire when the application logs `OutOfMemoryError` more than 3 times in 5 minutes. How do you build it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A log-based alert. With Loki you can use the ruler with LogQL metric queries.

**What I'd do →**
1. Write a LogQL query and test it in Grafana Explore.
2. Put it in a Loki ruler rule group that sends to Alertmanager:

```yaml
groups:
  - name: app-logs
    rules:
      - alert: JavaOOM
        expr: sum by (service) (count_over_time({env="prod"} |= "OutOfMemoryError" [5m])) > 3
        for: 0m
        labels: {severity: page}
        annotations: {summary: "{{ $labels.service }} OOM errors in logs"}
```

3. Prefer also exposing a metric (`jvm` OOM counter or container restarts with `OOMKilled`) for long-term trends.

**Why →** Log alerts catch events with no metric, but metrics are cheaper to evaluate.

**Interview tip →** If you can turn a log pattern into a metric, do it; log alerts are a bridge.

</details>

### Q61. Loki queries over 7 days time out for everyone. How do you make them usable? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Queries scan too many chunks because label selectors are too broad, or the read path is under-provisioned.

**What I'd do →**
1. Narrow selectors: `{namespace="payments", app="api"}` instead of `{env="prod"}`; put line filters (`|= "error"`) before parsers.
2. Run Loki in simple scalable or microservices mode with a query-frontend: `split_queries_by_interval: 1h` and results caching.
3. Scale queriers and set `max_query_parallelism`.
4. Use bloom filters/structured metadata in Loki 3.x for needle-in-haystack searches by trace ID.
5. For recurring analytics, convert to recording rules on the Loki ruler.

**Why →** Loki is grep at scale; the less it reads, the faster it is.

**Interview tip →** Explain "filter early, parse late" in LogQL.

</details>

### Q62. You need to keep audit logs for 7 years for compliance, but only query them rarely. Design storage. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long retention, rare access, must be tamper-evident. Hot search clusters are the wrong place.

**What I'd do →**
1. Route audit events to a separate stream from application logs.
2. Write to object storage (S3) with **Object Lock in compliance mode** and a lifecycle to Glacier Deep Archive after 90 days.
3. Store in compressed, partitioned format (e.g. `s3://audit/year=2026/month=03/`) as JSON or Parquet.
4. Query on demand with Athena or load into a temporary cluster.
5. Restrict IAM access and log access with CloudTrail data events.

**Why →** Object storage is cheap and durable; Object Lock gives immutability auditors accept.

**Interview tip →** Separate "searchable now" from "retained for compliance"; they have different SLAs.

</details>

### Q63. Logs from one service suddenly stop arriving in Loki, but the service is healthy. How do you trace the gap? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Something broke between the container's stdout and the backend.

**What I'd do →**
1. Confirm the app still writes: `kubectl logs deploy/api --tail=20`.
2. Check the node's shipper: `kubectl -n monitoring logs ds/alloy` on that node for errors like `429` or `entry out of order`.
3. Check Loki for rate limiting: `loki_discarded_samples_total` by `reason` (e.g. `rate_limited`, `per_stream_rate_limit`, `too_far_behind`).
4. Verify labels did not change (a renamed label means a new stream, so the old query shows nothing).
5. Add an alert: absence of logs for a critical service, `absent_over_time({app="api"}[15m])`.

**Why →** Walk the pipeline hop by hop; each layer exposes its own metrics.

**Interview tip →** Monitor the logging pipeline like any other production service.

</details>

### Q64. Your log volume is 5 TB/day. Should you put Kafka between shippers and the log backend? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** At this scale, backend maintenance, upgrades and spikes cause data loss without a buffer.

**What I'd do →**
1. Shippers (Vector/Fluent Bit) write to Kafka topics partitioned by service or tenant.
2. Consumers (Vector, Logstash, OTel Collector with Kafka receiver) read, enrich and write to Loki/ES/S3.
3. Size retention on Kafka for at least 24–72 hours of replay.
4. Monitor consumer lag as the key health metric for the pipeline.
5. Use the same stream to fan out to SIEM and archive without double-shipping.

**Why →** Kafka decouples producers from consumers, absorbs spikes, and allows replay after an outage.

**Interview tip →** Mention the trade-off: one more stateful system to operate and secure. Below ~100 GB/day, disk buffering is usually enough.

</details>

### Q65. Logs from different services have mismatched timestamps and some appear "in the future". How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mix of local time zones, missing time zones in log formats, clock drift, or the shipper using ingest time.

**What I'd do →**
1. Standardise on UTC with RFC 3339 timestamps (`2026-03-02T10:15:01.123Z`) in all apps.
2. Parse the app timestamp in the shipper (Alloy `stage.timestamp`, Fluent Bit parser `Time_Key`/`Time_Format`).
3. Ensure NTP/chrony is running and alert on `node_timex_offset_seconds` above 50 ms.
4. Check Loki's `reject_old_samples_max_age` and `creation_grace_period` for rejected entries.

**Why →** Correlation across services is only as good as clock and format consistency.

**Interview tip →** Clock drift silently breaks trace and log correlation; monitor it.

</details>

### Q66. Each team wants its own log index, retention and access. How do you implement multi-tenancy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared backend, separate data boundaries, different retention needs.

**What I'd do →**
1. In Loki, enable `auth_enabled: true` and set the `X-Scope-OrgID` tenant header per team at the gateway or shipper.
2. Configure per-tenant limits and retention in `runtime_config` overrides (`retention_period`, `ingestion_rate_mb`).
3. In Grafana, create a data source per tenant and map it to a team with permissions.
4. In Elasticsearch/OpenSearch, use index patterns per team and role-based access with document-level security if needed.

**Why →** Tenant isolation prevents one noisy team from hurting others and keeps data access least-privilege.

**Interview tip →** Mention per-tenant rate limits as the main defence against noisy neighbours.

</details>

### Q67. A service logs 50,000 identical `connection refused` lines per minute during an outage. How do you handle that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Log storms during incidents can overload the pipeline exactly when you need it.

**What I'd do →**
1. In the app, rate-limit or deduplicate repeated errors (log once with a count every N seconds).
2. In the shipper, throttle: Vector `throttle` transform, Fluent Bit `throttle` filter.
3. Make sure per-stream limits in Loki protect other tenants.
4. Track error rate as a metric instead, so the signal is not lost when logs are throttled.

**Why →** One error per second tells you the same as 800 per second, at a fraction of the cost.

**Interview tip →** Log storms are a classic way the observability stack fails during incidents.

</details>

### Q68. Developers cannot connect a log line to the request that produced it across 6 microservices. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No shared correlation ID across services.

**What I'd do →**
1. Adopt W3C Trace Context (`traceparent` header) via OpenTelemetry SDKs and auto-instrumentation.
2. Configure loggers to add `trace_id` and `span_id` from the active context to every line.
3. In Grafana, configure Loki **derived fields** to turn `trace_id` into a Tempo link, and Tempo "trace to logs" back.
4. Make gateways generate a trace ID if the client did not send one.

**Why →** One ID lets you follow a request from the edge through every service and its logs.

**Interview tip →** Say "logs, metrics and traces linked by trace_id", not three separate tools.

</details>

### Q69. You need to collect logs from 2,000 Windows and Linux VMs in a hybrid environment. What agent and design do you pick? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mixed OS, mixed networks, need central management.

**What I'd do →**
1. Pick one cross-platform agent: Grafana Alloy, OpenTelemetry Collector (contrib has `windowseventlog` and `filelog` receivers), or Vector.
2. Manage config centrally (Ansible, Fleet management in Grafana, OpAMP for OTel).
3. Send to regional gateways that buffer and forward over TLS to the backend.
4. Collect Windows Event Logs (Security, System, Application) and Linux journal plus app files.
5. Monitor agent health: heartbeats and per-host log rate.

**Why →** One agent means one config language, one upgrade path, one set of dashboards.

**Interview tip →** Mention agent fleet management; at 2,000 hosts, config drift is the real problem.

</details>

### Q70. Should you log every HTTP request at the ingress controller? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Access logs are valuable for security and debugging but can be the largest log source.

**What I'd do →**
1. Enable structured (JSON) access logs at ingress with key fields: status, latency, upstream, bytes, client IP, request ID.
2. Keep all 4xx/5xx and slow requests; sample successful 2xx (e.g. 10%) if volume is a concern.
3. Derive request metrics from Prometheus (ingress exports them) instead of computing them from logs.
4. Set shorter retention for access logs than for app errors.

**Why →** Metrics give the rate; logs give the evidence for specific failing requests.

**Interview tip →** Mention security needs: sometimes full access logs are required for forensics.

</details>

### Q71. Logstash is the bottleneck in your ELK pipeline with 100% CPU. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Heavy grok parsing and a single pipeline on JVM Logstash nodes.

**What I'd do →**
1. Use the Logstash monitoring API (`GET _node/stats/pipelines`) to find the slowest filter.
2. Replace grok with `dissect` or JSON parsing where possible; anchor grok patterns (`^...$`).
3. Increase `pipeline.workers` and `pipeline.batch.size`, enable persistent queues.
4. Move parsing to the edge (structured logs from apps, or Fluent Bit/Vector parsing).
5. Consider replacing Logstash with Elasticsearch ingest pipelines or Vector for lower resource use.

**Why →** Most Logstash CPU goes to regex; avoiding regex is the biggest win.

**Interview tip →** Unanchored grok patterns that fail to match are the most expensive case.

</details>

### Q72. How do you make sure audit logs from the Kubernetes API server are captured and useful? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** API audit logs answer "who did what to which resource", but default levels can be huge.

**What I'd do →**
1. Write an audit policy: `Metadata` for most resources, `RequestResponse` for RBAC and secrets changes, `None` for noisy reads like `get` on events and leases.
2. On self-managed clusters set `--audit-policy-file` and `--audit-log-path` or a webhook backend. On EKS enable the `audit` control plane log type to CloudWatch.
3. Ship to the SIEM and alert on sensitive actions: `create clusterrolebinding`, `exec` into pods, secret reads by humans.

**Why →** Audit logs are your forensic record for cluster access.

**Interview tip →** Never log secret contents; use `Metadata` level for `secrets`.

</details>

### Q73. Should you choose Loki or Elasticsearch for a new logging platform? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** It depends on query patterns, scale and budget.

**What I'd do →**
1. List use cases: debugging by service and time (Loki fits), full-text search across all fields and analytics (Elasticsearch/OpenSearch fits), security/SIEM (ES or a dedicated SIEM).
2. Compare cost: Loki indexes labels only and stores chunks in S3, so storage is cheap; ES indexes everything, so it is faster for arbitrary search but needs much more disk and memory.
3. Consider the team: Grafana-native stack favours Loki; existing Kibana skills favour ES.
4. Run a two-week proof of concept on real traffic, measuring cost per GB and query latency.

**Why →** There is no universal winner; it is a cost versus query flexibility trade-off.

**Interview tip →** State the core trade-off in one line: "Loki is cheap to write, ES is cheap to search."

</details>

### Q74. The logging agent is consuming 1.5 CPU cores per node. How do you reduce it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Heavy parsing, regex, or excessive volume on the node.

**What I'd do →**
1. Profile which pipeline stages use CPU (agent metrics, or pprof for Alloy/Vector).
2. Replace regex parsing with JSON parsing or move parsing to a central aggregation tier.
3. Drop noisy logs as early as possible (before enrichment and parsing).
4. Batch and compress output (gzip/snappy).
5. Set CPU limits carefully: throttling causes lag and growing buffers.

**Why →** Agents run on every node, so wasted CPU is multiplied by fleet size.

**Interview tip →** Present the cost: 1.5 cores x 300 nodes is 450 cores spent on log shipping.

</details>

### Q75. A security incident needs logs from 6 months ago that were deleted after 30 days. How do you avoid this next time? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retention was set by cost, not by investigation or compliance needs.

**What I'd do →**
1. Agree retention with security and legal: e.g. 30 days hot, 1 year cold archive for security-relevant logs.
2. Archive raw logs to S3 with lifecycle rules (Standard → Glacier), alongside hot storage.
3. Document how to restore or query archives (Athena, rehydrating into a temporary index).
4. Test the restore process once a quarter.

**Why →** Archival storage is cheap; missing evidence is expensive.

**Interview tip →** Retention is a business decision; engineers implement it, not choose it alone.

</details>

---

## Tracing and OpenTelemetry

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Your company has 30 microservices and no tracing. How do you roll out distributed tracing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Debugging latency across services is guesswork. Roll out gradually with minimum code changes.

**What I'd do →**
1. Standardise on OpenTelemetry (SDKs, OTLP protocol, W3C Trace Context).
2. Deploy an OpenTelemetry Collector (agent DaemonSet plus gateway Deployment) exporting to Tempo, Jaeger or a vendor.
3. Start with zero-code auto-instrumentation: Java agent, Python `opentelemetry-instrument`, Node `@opentelemetry/auto-instrumentations-node`, or the OpenTelemetry Operator `Instrumentation` CRD.
4. Instrument the edge first (gateway, ingress), then the most critical request path.
5. Add manual spans for business operations later.

**Why →** Auto-instrumentation covers HTTP, gRPC, DB and messaging quickly; a central collector keeps exporters swappable.

**Interview tip →** Say "OTel for instrumentation, backend is a separate choice" to show vendor neutrality.

</details>

### Q77. Traces stop at service B: you see spans from A, but B's spans start new traces. What is wrong? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Context propagation is broken between A and B.

**What I'd do →**
1. Check whether A sends the `traceparent` header: capture the request or log headers in B.
2. Ensure both use the same propagators (`OTEL_PROPAGATORS=tracecontext,baggage`); a B3-only service will not read W3C headers.
3. Look for a proxy or gateway that strips unknown headers.
4. For async paths (Kafka, SQS), check that the producer injects context into message headers and the consumer extracts it.
5. Check for custom HTTP clients that bypass instrumentation.

**Why →** A trace is only as connected as its weakest propagation hop.

**Interview tip →** Mention mixed propagators (B3 and W3C) as a common cause in brownfield setups.

</details>

### Q78. Tracing every request costs too much. How do you design sampling? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** 100% tracing at high volume is expensive; random 1% misses the interesting traces.

**What I'd do →**
1. Head sampling in SDKs: `OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.1` as a base.
2. Tail sampling in the Collector gateway to keep all errors, slow requests and important routes:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - {name: errors, type: status_code, status_code: {status_codes: [ERROR]}}
      - {name: slow, type: latency, latency: {threshold_ms: 1000}}
      - {name: baseline, type: probabilistic, probabilistic: {sampling_percentage: 5}}
```

3. Keep RED metrics from 100% of spans (spanmetrics connector) so rates stay accurate.

**Why →** You keep the traces worth reading and still get accurate aggregate metrics.

**Interview tip →** Explain that tail sampling needs all spans of a trace on the same collector (load-balancing exporter by trace ID).

</details>

### Q79. You set up tail sampling with 4 Collector replicas, but traces are fragmented and error traces are missing. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Spans of one trace land on different replicas, so each makes a decision on partial data.

**What I'd do →**
1. Add a first tier of Collectors with the `loadbalancing` exporter, `routing_key: traceID`, resolving the second tier via a headless Service (`k8s` or `dns` resolver).
2. Run `tail_sampling` only on the second tier.
3. Size `decision_wait` to cover your longest normal trace and `num_traces` to fit memory.
4. Monitor `otelcol_processor_tail_sampling_*` metrics and dropped spans.

**Why →** Tail sampling needs the complete trace in one place before deciding.

**Interview tip →** Scaling the second tier reshuffles trace ownership briefly; mention that you accept a little fragmentation during scale events.

</details>

### Q80. A checkout request takes 4 seconds. Walk through how you use a trace to find the cause. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You have the trace; the job is to read the waterfall.

**What I'd do →**
1. Open the trace in Tempo/Jaeger and look at the critical path: which spans are sequential and long.
2. Check for gaps between spans (uninstrumented work, queueing, GC pauses).
3. Look for N+1 patterns: dozens of short repeated DB or HTTP spans.
4. Read span attributes: `db.statement`, `http.response.status_code`, retries, `error` events.
5. Jump to logs for that span via trace ID, and to metrics (pod CPU) at that time.

**Why →** The waterfall shows where time is actually spent, not where you guess it is.

**Interview tip →** Mention "N+1 queries" and "serial calls that could be parallel" as common findings.

</details>

### Q81. Your OpenTelemetry Collector is getting OOMKilled under load. How do you stabilise it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No memory limiter, large batches, or a slow exporter causing the queue to grow.

**What I'd do →**
1. Add `memory_limiter` as the first processor in each pipeline:

```yaml
processors:
  memory_limiter: {check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 20}
  batch: {send_batch_size: 8192, timeout: 5s}
service:
  pipelines:
    traces: {receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp]}
```

2. Set `GOMEMLIMIT` to about 80% of the container limit.
3. Configure `sending_queue` size and optional file storage for persistence.
4. Scale horizontally with an HPA on CPU/memory.
5. Monitor `otelcol_exporter_queue_size` and `otelcol_processor_refused_spans`.

**Why →** The memory limiter applies back-pressure instead of crashing and losing everything.

**Interview tip →** Processor order matters: memory_limiter first, batch last before export.

</details>

### Q82. How do you trace a request that goes through Kafka from a producer to three consumers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Async messaging breaks the simple parent/child chain.

**What I'd do →**
1. Use instrumented Kafka clients (OTel Java agent, `opentelemetry-instrumentation-kafka-python`, confluent-kafka instrumentations) so the producer injects `traceparent` into message headers.
2. Consumers extract context and create a span with a **link** to the producer span (or a child, depending on semantics).
3. Use messaging semantic conventions (`messaging.system`, `messaging.destination.name`).
4. Track consumer lag as a metric alongside traces.

**Why →** Links represent fan-out and batch processing better than strict parent/child.

**Interview tip →** Mention span links for batch consumers that process messages from many traces at once.

</details>

### Q83. A developer asks: "Should I add a span for every function?" What do you tell them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Over-instrumentation adds overhead and noise; under-instrumentation leaves gaps.

**What I'd do →**
1. Let auto-instrumentation cover network boundaries (HTTP, gRPC, DB, cache, queue).
2. Add manual spans for meaningful units of work: "calculate price", "reserve inventory", external calls without instrumentation.
3. Add attributes (`order.id`, `customer.tier`) rather than more spans.
4. Use span events for notable moments (retry, cache miss).

**Why →** Each span costs CPU and storage; aim for the spans that explain latency.

**Interview tip →** "Span boundaries at I/O and business steps, attributes for context."

</details>

### Q84. You want RED metrics for every service without asking teams to add Prometheus code. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Traces already contain rate, errors and duration for each operation.

**What I'd do →**
1. In the Collector, add the `spanmetrics` connector (traces in, metrics out), with histogram buckets and dimensions like `service.name`, `span.name`, `http.response.status_code`.
2. Export metrics to Prometheus via `prometheusremotewrite` or Prometheus exporter.
3. Or enable Tempo's metrics-generator (`span-metrics` and `service-graphs` processors).
4. Run spanmetrics **before** tail sampling so counts are accurate.

**Why →** One instrumentation source produces both traces and metrics consistently.

**Interview tip →** Watch cardinality: never include IDs or raw URLs as dimensions.

</details>

### Q85. Span names look like `GET /users/8812/orders/99812`. Why is that a problem? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Raw paths with IDs create unbounded span names, which explode cardinality in span metrics and make grouping useless.

**What I'd do →**
1. Use the route template: `GET /users/{id}/orders/{orderId}` (most framework instrumentations set `http.route`).
2. Where it is wrong, fix it in the Collector with the `transform` processor or in the SDK.
3. Keep the full URL in `url.full` or `url.path` as an attribute, not in the name.

**Why →** Span names should be low-cardinality groups; specifics belong in attributes.

**Interview tip →** Same principle as metric labels: names group, attributes describe.

</details>

### Q86. Tempo storage costs are growing fast. How do you control them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Volume, retention and span size all drive cost.

**What I'd do →**
1. Measure ingested bytes per service (`tempo_distributor_bytes_received_total` by tenant, spans per service).
2. Apply or tighten tail sampling and drop noisy spans (health checks, `/metrics`) with a `filter` processor.
3. Remove large attributes (SQL with full payloads, request bodies) via `attributes`/`transform` processors.
4. Set block retention in the compactor (`compaction.block_retention: 336h`).
5. Keep 100% data only in span metrics, not raw traces.

**Why →** Most trace data is never read; sample and trim it.

**Interview tip →** Health check spans are often 30%+ of volume; dropping them is a quick win.

</details>

### Q87. Your team uses Jaeger clients that are deprecated. How do you migrate to OpenTelemetry? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jaeger client libraries are retired; Jaeger itself accepts OTLP natively.

**What I'd do →**
1. Upgrade the Jaeger backend to a version that accepts OTLP (Jaeger v2 is built on the OTel Collector).
2. Deploy an OTel Collector that receives both `jaeger` and `otlp`, exporting to the backend, so old and new services coexist.
3. Configure propagators to accept both `jaeger` (uber-trace-id) and `tracecontext` during migration.
4. Migrate services one by one to OTel SDKs, starting at the edges.
5. Remove Jaeger propagation once all services are migrated.

**Why →** A Collector bridge and dual propagation keep traces connected during the transition.

**Interview tip →** The propagation format is the main risk; mention handling both headers.

</details>

### Q88. How do you instrument a Kubernetes cluster so every new pod gets tracing automatically? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want platform-level defaults, not per-team effort.

**What I'd do →**
1. Install the OpenTelemetry Operator (needs cert-manager).
2. Create an `Instrumentation` resource with exporter endpoint, propagators and sampler.
3. Teams opt in with an annotation such as `instrumentation.opentelemetry.io/inject-java: "true"` (also `-python`, `-nodejs`, `-dotnet`, `-go`).
4. Use the `k8sattributes` processor in the Collector to add pod, namespace and deployment metadata.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata: {name: default}
spec:
  exporter: {endpoint: http://otel-collector.observability:4317}
  propagators: [tracecontext, baggage]
  sampler: {type: parentbased_traceidratio, argument: "0.1"}
```

**Why →** Annotation-based injection gives tracing without image or code changes.

**Interview tip →** Mention that Go injection uses eBPF and needs elevated privileges.

</details>

### Q89. Can you get traces from services you cannot modify at all? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Third-party binaries or legacy apps with no SDK.

**What I'd do →**
1. Use eBPF-based instrumentation (Grafana Beyla/OpenTelemetry eBPF instrumentation, Pixie, Cilium Hubble) for HTTP/gRPC spans and RED metrics.
2. Use a service mesh (Istio, Linkerd) that emits spans from sidecars; the app still must forward trace headers for a connected trace.
3. Trace at the edges you control (gateway, DB proxy).

**Why →** eBPF observes syscalls and network traffic without code changes.

**Interview tip →** Mention the limitation: without header forwarding inside the app, mesh spans are disconnected.

</details>

### Q90. Your service graph shows a call from `orders` to `legacy-billing` that nobody knew existed. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hidden dependency, a reliability and ownership risk.

**What I'd do →**
1. Check call volume, latency and error rate of that edge in the service graph.
2. Find the code path via trace attributes (`code.function`, route).
3. Identify the owner of `legacy-billing` and add it to the dependency map and incident runbooks.
4. Decide with the team: add timeouts and circuit breakers, or remove the dependency.

**Why →** Unknown dependencies cause unexplained outages; traces make them visible.

**Interview tip →** Service graphs from traces are one of the quickest wins for architecture reviews.

</details>

### Q91. How do you pass a tenant ID from the gateway to every downstream span without changing every service? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want a business attribute on all spans for filtering by customer.

**What I'd do →**
1. At the gateway, set W3C **baggage** (`baggage: tenant.id=acme`).
2. Instrumented services propagate baggage automatically.
3. Add a span processor (e.g. baggage span processor in the SDK contrib packages) to copy baggage entries to span attributes.
4. Never put secrets or PII in baggage; it travels in headers, including to third parties if not stripped.

**Why →** Baggage propagates with the trace context, so one change at the edge reaches all services.

**Interview tip →** Strip baggage at egress to external APIs.

</details>

### Q92. You need traces from AWS Lambda functions linked to traces from your EKS services. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Serverless and containers must share context.

**What I'd do →**
1. Use the AWS Distro for OpenTelemetry (ADOT) Lambda layer or OTel Lambda layers for your runtime.
2. Export via OTLP to your central Collector (or to X-Ray if you use it).
3. Ensure W3C propagation on both sides; configure X-Ray propagator if API Gateway/X-Ray headers are involved.
4. Watch cold start overhead and flush spans before the function returns.

**Why →** Common OTLP and propagation format links both worlds into one trace.

**Interview tip →** Mention that unflushed spans are lost when the Lambda freezes.

</details>

### Q93. Traces show 800 ms of "missing time" between two spans in the same service. What could it be? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time spent in uninstrumented work or waiting.

**What I'd do →**
1. Check for uninstrumented code: CPU-heavy loops, serialization, local file I/O.
2. Check GC pauses (JVM `jvm.gc.duration`, Go GC metrics) at that timestamp.
3. Check CPU throttling: `container_cpu_cfs_throttled_periods_total`.
4. Check thread pool or connection pool waiting.
5. Use continuous profiling (Pyroscope, Parca) linked to the span to see the stack.

**Why →** Gaps are evidence of work you cannot see; profiling fills the gap.

**Interview tip →** Mention span profiles (trace-to-profile) as the modern answer to "missing time".

</details>

### Q94. How do you make sure the tracing backend failing never impacts the application? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Observability must not become a dependency that can take down production.

**What I'd do →**
1. Use the batch span processor (async export), never synchronous export.
2. Send to a local Collector agent (node-local) instead of directly to a remote backend.
3. Set bounded queues (`OTEL_BSP_MAX_QUEUE_SIZE`) so spans are dropped, not memory exhausted.
4. Set export timeouts (`OTEL_EXPORTER_OTLP_TIMEOUT`).
5. Test by shutting down the Collector in staging.

**Why →** Dropping telemetry is acceptable; dropping user requests is not.

**Interview tip →** "Telemetry is best effort" is the right principle to state.

</details>

### Q95. Logs, metrics and traces each live in different tools. How do you make correlation work in Grafana? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Engineers context-switch between tools and lose time.

**What I'd do →**
1. Use consistent resource attributes: `service.name`, `service.namespace`, `deployment.environment`, `k8s.pod.name`.
2. Tempo data source: configure **trace to logs** (Loki) and **trace to metrics** (Prometheus) with tag mappings.
3. Loki data source: **derived fields** to open trace IDs in Tempo.
4. Prometheus: **exemplars** linking to Tempo.
5. Test the full loop: metric spike → exemplar → trace → logs.

**Why →** Correlation works only with consistent labels and configured links.

**Interview tip →** The hard part is naming consistency, not the tooling.

</details>

### Q96. Product wants to know which customers experience the slowest checkouts. How do you answer with traces? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need customer-level latency analysis without high-cardinality metrics.

**What I'd do →**
1. Add `customer.id` or `customer.tier` as a span attribute on the checkout span.
2. Query with TraceQL in Tempo:

```
{ span.name = "checkout" && duration > 2s } | select(span.customer.tier)
```

3. Use TraceQL metrics (`| quantile_over_time(duration, 0.99) by (span.customer.tier)`) for aggregated views on recent versions.
4. Keep customer IDs out of Prometheus labels.

**Why →** Traces hold high-cardinality attributes cheaply; metrics do not.

**Interview tip →** Split which data goes where: tiers in metrics, IDs in traces.

</details>

### Q97. An OTel Collector config change broke trace ingestion for all services. How do you prevent this? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Collector is shared infrastructure; config changes need the same rigour as app deploys.

**What I'd do →**
1. Validate configs in CI: `otelcol-contrib validate --config=config.yaml`.
2. Use GitOps with canary rollout: update one agent group or one Collector replica first.
3. Monitor `otelcol_receiver_accepted_spans`, `otelcol_receiver_refused_spans`, `otelcol_exporter_send_failed_spans`.
4. Alert on a sudden drop in accepted spans per service.
5. Keep the previous config ready for instant rollback.

**Why →** One bad config affects every service; staged rollout limits the blast radius.

**Interview tip →** Mention `validate` command; many candidates do not know it exists.

</details>

### Q98. Your trace volume dropped 60% after a release, but traffic is unchanged. What do you check? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Instrumentation or sampling changed, or spans are being dropped.

**What I'd do →**
1. Check if the release changed the SDK version, sampler env vars or removed the agent.
2. Compare span counts per service to find which one dropped.
3. Check Collector drop metrics and exporter failures.
4. Check if the new version changed `service.name` (data is there under a different name).

**Why →** Telemetry regressions are often caused by the same deploy that changes code.

**Interview tip →** Alert on telemetry volume per service to detect instrumentation regressions.

</details>

### Q99. Tracing adds 15% latency overhead to a high-throughput service. What do you tune? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Overhead usually comes from too many spans, synchronous export, or heavy attribute processing.

**What I'd do →**
1. Profile with tracing on and off to confirm the source.
2. Ensure batch processor with async export.
3. Lower the head sampling ratio; unsampled spans skip most recording work.
4. Disable noisy auto-instrumentations (e.g. `OTEL_INSTRUMENTATION_<NAME>_ENABLED=false` in Java) for internal libraries.
5. Avoid expensive attribute computation (serializing payloads) in hot paths.

**Why →** Well-configured tracing typically costs low single-digit percent.

**Interview tip →** Give a measurement plan rather than guessing.

</details>

### Q100. Leadership asks whether to buy a commercial APM or build on open source (OTel + Grafana stack). How do you decide? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build versus buy trade-off: cost, staff time, features, lock-in.

**What I'd do →**
1. Estimate data volume (spans, series, logs per day) and price both options for 3 years.
2. Count the engineering time needed to run Tempo, Mimir and Loki at scale (on-call, upgrades).
3. Compare features: AI-assisted root cause, RUM, profiling, alert quality.
4. Instrument with OpenTelemetry either way so switching later is cheap.
5. Run a proof of concept with one critical service on both.

**Why →** OTel instrumentation decouples the decision from the code; the backend becomes swappable.

**Interview tip →** "Instrument with open standards, choose the backend on cost and capability."

</details>

---

## SLOs and Error Budgets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Your company has no SLOs. A VP asks you to introduce them. Where do you begin? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SLOs are a reliability contract between engineering and the business. Start small with one user journey.

**What I'd do →**
1. Pick one critical user journey (e.g. "user can log in", "checkout succeeds").
2. Define an SLI that measures it from the user's view: good events / valid events.
3. Look at 4 weeks of historical data to set a realistic target (e.g. 99.5%), not an aspirational 99.99%.
4. Agree the target and an error budget policy with product.
5. Build a dashboard and burn-rate alerts, then expand to more journeys.

**Why →** One well-adopted SLO beats 50 ignored ones.

**Interview tip →** Explain SLI (measurement), SLO (target), SLA (contract with penalties) in one line each.

</details>

### Q102. Choose an SLI for a REST API: availability measured at the pod, the load balancer, or the client? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The closer to the user, the more accurate, but the harder to measure.

**What I'd do →**
1. Use the **load balancer or ingress** as the primary source: it captures requests that never reached pods (5xx from LB, no healthy targets).
2. Add **synthetic probes** for coverage during low traffic.
3. Use client-side RUM data for web/mobile journeys where possible.
4. Avoid pod-level metrics alone; they miss failures before the app.

```promql
sum(rate(nginx_ingress_controller_requests{ingress="api",status!~"5.."}[5m]))
/
sum(rate(nginx_ingress_controller_requests{ingress="api"}[5m]))
```

**Why →** An SLI that cannot see failures cannot protect users.

**Interview tip →** Mention what counts as "valid": usually exclude 4xx caused by clients.

</details>

### Q103. The product owner wants a 99.99% SLO for a service that depends on a database with 99.95% availability. What do you say? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You cannot be more reliable than a hard dependency without redundancy.

**What I'd do →**
1. Show the maths: 99.99% is ~4.3 minutes/month of budget; the DB alone may spend ~21.9 minutes.
2. Offer options: accept 99.9%, or invest in redundancy (multi-AZ failover, caching, graceful degradation, read replicas).
3. Estimate the cost of each extra nine.
4. Agree a target based on what users actually need and what the business will pay for.

**Why →** Each nine roughly multiplies cost and complexity.

**Interview tip →** "Reliability of serial dependencies multiplies"; quote 0.9995 x 0.9999 ≈ 0.9994.

</details>

### Q104. The error budget for the month is exhausted on day 18. What happens now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The error budget policy should already define this; if it does not, write it now.

**What I'd do →**
1. Freeze risky feature launches for that service; allow fixes and reliability work.
2. Run a review of what burned the budget (top incidents, bad deploys, dependencies).
3. Prioritise the top reliability actions in the next sprint.
4. Lift the freeze when the rolling window recovers or when the fixes ship, as agreed.
5. Report to stakeholders with data.

**Why →** The budget turns reliability versus velocity into a data-driven decision, not an argument.

**Interview tip →** Mention that the policy must be signed by product leadership, or it will be ignored.

</details>

### Q105. Write an SLO in a declarative tool so teams do not hand-write burn-rate rules. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hand-written multi-window burn-rate rules are error-prone. Generate them.

**What I'd do →**
1. Use Sloth or Pyrra (or OpenSLO spec) to define SLOs as code.
2. Generate recording rules and alerts in CI and deploy via GitOps.

```yaml
version: "prometheus/v1"
service: "checkout"
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{job="checkout",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{job="checkout"}[{{.window}}]))
    alerting:
      name: CheckoutHighErrorRate
      page_alert: {labels: {severity: page}}
      ticket_alert: {labels: {severity: ticket}}
```

3. Run `sloth generate -i slo.yaml -o rules.yaml` and review the output.

**Why →** Consistent, tested rules across teams with one input per SLO.

**Interview tip →** Mention the generated multi-window, multi-burn-rate alerts by name.

</details>

### Q106. Latency SLO: "99% of requests under 300 ms." How do you measure it correctly in Prometheus? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A latency SLO is a ratio of fast requests to all requests, not a p99 gauge.

**What I'd do →**
1. Ensure the histogram has a bucket at exactly 0.3.
2. SLI = requests under 300 ms / all requests:

```promql
sum(rate(http_request_duration_seconds_bucket{job="api",le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{job="api"}[5m]))
```

3. Use this ratio in burn-rate alerts the same way as availability.

**Why →** Ratios aggregate correctly across pods and time; percentiles do not.

**Interview tip →** "Latency SLOs are counted as good events under a threshold."

</details>

### Q107. A service has 20 requests per hour overnight. One failure blows the SLO alert. How do you handle low-traffic services? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Small denominators make ratios noisy; one failure is 5% error rate.

**What I'd do →**
1. Add synthetic traffic (probes every 30s) to raise the baseline and detect outages.
2. Use longer windows or require a minimum event count in alerts: `and sum(increase(http_requests_total[1h])) > 100`.
3. Group small related services into one SLO.
4. Consider a time-based SLI ("minutes the probe succeeded") for very low-traffic endpoints.

**Why →** Alerts should fire on statistically meaningful impact.

**Interview tip →** Name the issue: "low request volume makes ratio SLIs unstable."

</details>

### Q108. How do you report SLO compliance to leadership monthly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Leadership needs a clear picture: are we meeting commitments, where are we spending budget?

**What I'd do →**
1. One table per service: SLO target, actual over 30 days, budget remaining.
2. Top 3 budget consumers (incidents, deploys, dependencies) with links to postmortems.
3. Trend over the last 6 months.
4. Actions planned and their expected impact.
5. Build it in Grafana with `avg_over_time` of recording rules, or use the SLO feature of your tool.

**Why →** A short, consistent report builds trust in SLOs as a decision tool.

**Interview tip →** Report budget consumed, not just "99.93%"; budgets are easier to understand.

</details>

### Q109. Developers want to deploy 20 times a day, ops wants stability. How do error budgets settle this? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic dev versus ops tension.

**What I'd do →**
1. Agree an SLO with both sides.
2. While budget remains, deploy freely; release velocity is encouraged.
3. When budget runs low, slow down and invest in reliability.
4. Make budget visible on the team dashboard.

**Why →** Both sides agree on the rules beforehand, so decisions become objective.

**Interview tip →** "100% is the wrong target; the error budget is permission to take risk."

</details>

### Q110. You have SLOs for 40 microservices but users still complain. What is missing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-service SLOs can all be green while the end-to-end journey fails.

**What I'd do →**
1. Define SLOs on user journeys (login, search, checkout) measured at the edge or with synthetic journeys.
2. Map which services contribute to each journey.
3. Use per-service SLOs as supporting indicators for teams, not as the main promise.
4. Validate with support tickets and RUM data.

**Why →** Users experience journeys, not services.

**Interview tip →** Mention that internal service SLOs must be stricter than the journey SLO they support.

</details>

### Q111. A dependency (third-party payment API) causes most of your SLO burns. Should it count against your budget? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Users do not care whose fault it is, but the team needs actionable data.

**What I'd do →**
1. Keep it in the user-facing SLO; users still felt it.
2. Track a separate SLI for the dependency (calls to the provider: success and latency).
3. Invest in mitigation: retries with backoff, timeouts, circuit breaker, fallback provider, queueing.
4. Use the data in vendor SLA discussions.

**Why →** Excluding dependencies hides real user pain and removes the incentive to design for failure.

**Interview tip →** "You own your users' experience, including your dependencies."

</details>

### Q112. How do you prevent planned maintenance from consuming the error budget? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teams argue whether planned downtime counts.

**What I'd do →**
1. Decide policy up front: users still see errors, so often it counts.
2. Prefer zero-downtime maintenance (rolling updates, blue-green, online schema migrations).
3. If excluded per agreement, record windows explicitly and exclude them in the SLI query (e.g. a `maintenance` metric multiplied in).
4. Communicate maintenance to users in advance.

**Why →** Explicit rules prevent quiet budget manipulation.

**Interview tip →** "If maintenance hurts users, that's a design problem to fix."

</details>

### Q113. Your SLO dashboard shows 99.95% while the outage last week lasted 2 hours. How can that be? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The SLI did not see the outage.

**What I'd do →**
1. Check if the metric source was down too (pods dead, no metrics, `rate()` of nothing = no data, not errors).
2. Check whether failures happened before the measurement point (DNS, LB, CDN).
3. Check for `absent` handling: missing data might be treated as good.
4. Fix: measure at the edge, add synthetic probes, and treat missing data as bad in the SLI.
5. Correct the record so the budget reflects reality.

**Why →** An SLI that goes silent during outages over-reports reliability.

**Interview tip →** "No data is not good data" is the takeaway.

</details>

### Q114. How do you set an SLO for a batch pipeline that runs once per hour? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Request-based SLIs do not fit; use freshness and correctness.

**What I'd do →**
1. Freshness SLI: data is no older than 2 hours, 99% of the time.
2. Correctness SLI: percentage of runs that pass validation checks.
3. Export `last_success_timestamp_seconds` from the job and compute freshness:

```promql
time() - max(pipeline_last_success_timestamp_seconds{pipeline="orders_etl"}) < 7200
```

4. Alert when freshness is about to breach, not after.

**Why →** For data pipelines, "late" and "wrong" are the user-facing failures.

**Interview tip →** Name the SLI types: freshness, coverage, correctness.

</details>

### Q115. Two teams argue whether 4xx responses should count as errors. How do you decide? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Most 4xx are client mistakes, but some are server bugs.

**What I'd do →**
1. Exclude 4xx by default from the error count; they are client errors.
2. Include specific codes caused by the service: 429 (if you throttle unfairly), 408, or 404 on routes that should exist.
3. Track 4xx rate separately and alert on sudden jumps (often a broken client release or API change).

**Why →** The SLI must reflect service failures, not user typos.

**Interview tip →** Mention 429 as the grey area worth discussing.

</details>

### Q116. How do you use SLOs to decide whether a deploy should be rolled back automatically? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Progressive delivery can use the SLI as its health signal.

**What I'd do →**
1. Use Argo Rollouts or Flagger with canary steps (5%, 25%, 50%).
2. Define an `AnalysisTemplate` querying Prometheus for the canary's error ratio and latency SLI.
3. Fail the analysis if the canary burns budget faster than baseline (e.g. error ratio > 1% for 2 consecutive checks).
4. Automatic rollback on failure; notify the team.

```yaml
metrics:
  - name: error-ratio
    interval: 1m
    failureLimit: 2
    successCondition: result[0] < 0.01
    provider:
      prometheus:
        address: http://prometheus.monitoring:9090
        query: sum(rate(http_requests_total{app="api",track="canary",code=~"5.."}[2m])) / sum(rate(http_requests_total{app="api",track="canary"}[2m]))
```

**Why →** Bad deploys are caught on a fraction of traffic before they burn the full budget.

**Interview tip →** Compare canary to baseline, not to a fixed number, when traffic varies.

</details>

### Q117. Your SLO window is calendar month. Should you use a rolling 28/30-day window instead? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Calendar windows reset on the 1st; rolling windows reflect recent reality continuously.

**What I'd do →**
1. Use a **rolling** window (28 or 30 days) for operations and alerting; behaviour on the 2nd is the same as on the 29th.
2. Use **calendar** windows for SLA reports and contracts.
3. A 28-day window has an equal number of weekdays and weekends, reducing weekly seasonality.

**Why →** Calendar resets create perverse incentives ("budget refills tomorrow, ship it").

**Interview tip →** Mention the 28-day choice and why.

</details>

### Q118. Executives want one "platform reliability" number. How do you aggregate SLOs? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Averaging percentages hides failures and mixes different traffic volumes.

**What I'd do →**
1. Avoid simple averages of SLO percentages.
2. Offer: percentage of SLOs met this month, or budget remaining per critical journey.
3. If one number is required, weight by traffic or business impact (e.g. revenue-weighted journey availability).
4. Always show the worst performer next to the aggregate.

**Why →** A single average lets one broken critical service look fine.

**Interview tip →** "Report count of SLOs breached and the worst one" is a strong answer.

</details>

### Q119. A team set an SLO of 99.999% and it is always breached. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The target is unrealistic, so the SLO is ignored.

**What I'd do →**
1. Review historical performance and user expectations.
2. Reset to a realistic target slightly above current performance (e.g. 99.9%).
3. Raise it gradually after reliability investments prove out.

**Why →** An SLO that is always red loses all meaning; teams stop looking.

**Interview tip →** "SLOs should be achievable and just tight enough to keep users happy."

</details>

### Q120. How do you validate that an SLO target actually matches user happiness? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An SLO can be met while users complain, or breached with no complaints.

**What I'd do →**
1. Compare SLO burn periods to support tickets, NPS drops, churn and social mentions.
2. Check conversion or abandonment rates against latency.
3. Adjust thresholds (e.g. latency 300 ms → 500 ms) if users do not notice, or tighten them if they do.
4. Review quarterly.

**Why →** The SLO is a proxy for happiness; validate the proxy.

**Interview tip →** Mention conversion rate versus latency as the strongest business signal.

</details>

### Q121. How would you monitor SLOs for a gRPC service? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** gRPC returns status codes in trailers, often with HTTP 200 at the transport level.

**What I'd do →**
1. Use gRPC interceptors (e.g. `grpc_server_handled_total` from go-grpc-prometheus or OTel `rpc.server.duration`).
2. Count error codes that indicate server failure: `UNAVAILABLE`, `INTERNAL`, `DEADLINE_EXCEEDED`, `UNKNOWN`, `DATA_LOSS`.
3. Exclude client errors such as `INVALID_ARGUMENT`, `NOT_FOUND`.

```promql
sum(rate(grpc_server_handled_total{grpc_service="orders.Orders",grpc_code=~"Unavailable|Internal|DeadlineExceeded|Unknown"}[5m]))
/ sum(rate(grpc_server_handled_total{grpc_service="orders.Orders"}[5m]))
```

**Why →** HTTP status codes alone miss gRPC failures.

**Interview tip →** Mention that gRPC errors often hide behind HTTP 200.

</details>

### Q122. A postmortem shows the incident burned 40% of the budget, but the alert fired only after 50 minutes. How do you improve detection? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Burn-rate thresholds or windows are too slow for this failure pattern.

**What I'd do →**
1. Replay the incident data against the rules to see when each window would trigger.
2. Check the fast-burn tier (e.g. 14.4x over 1h with a 5m short window) exists and is not muted.
3. Check `for:` durations and evaluation intervals stacking up delays.
4. Check whether the SLI itself lagged (e.g. data only from a 5-minute scrape of a batch exporter).
5. Add synthetic probes for faster outage detection.

**Why →** Detection time directly drives budget consumed per incident.

**Interview tip →** Track MTTD (time to detect) as a metric for your alerting quality.

</details>

### Q123. How do you create SLOs for an internal platform (e.g. CI system or Kubernetes platform) used by developers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Platform users are internal engineers; their journeys are builds, deploys and cluster API access.

**What I'd do →**
1. Define journeys: "CI pipeline starts within 2 minutes", "deploy succeeds", "kube-apiserver request success".
2. SLIs: queue time for CI runners, deploy success ratio, API server `apiserver_request_total` non-5xx ratio and latency.
3. Publish targets and dashboards to developers.
4. Use error budgets to prioritise platform work.

**Why →** Platform teams need the same contract with their customers as product teams.

**Interview tip →** Queue time is a great SLI for CI platforms.

</details>

### Q124. Developers ask why their SLO dashboard and the burn-rate alert disagree. How do you explain it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dashboards show long-window compliance; alerts look at recent burn rate.

**What I'd do →**
1. Explain: a 30-day compliance of 99.92% can coexist with a fast burn right now.
2. Show burn rate on the same dashboard (1h, 6h, 3d).
3. Add "budget remaining" and "time to exhaustion at current rate".

**Why →** Different windows answer different questions: "are we OK this month" versus "is something wrong now".

**Interview tip →** Put "time until budget exhaustion" on dashboards; it is intuitive.

</details>

### Q125. How do you get teams to actually care about SLOs after launch? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SLOs often become shelfware after the initial project.

**What I'd do →**
1. Make SLO review part of the weekly on-call handover and sprint planning.
2. Tie the error budget policy to release decisions.
3. Page only on SLO burn, so SLOs become part of daily work.
4. Celebrate wins: reduced incidents, faster releases.
5. Report SLO status in leadership reviews.

**Why →** Adoption comes from SLOs driving decisions, not from dashboards.

**Interview tip →** "If no decision changes because of an SLO, it is just a number."

</details>

---

## Grafana, Dashboards and Kubernetes Monitoring

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. Grafana has 900 dashboards and nobody can find anything. How do you clean it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dashboard sprawl from copy-paste and no ownership.

**What I'd do →**
1. Export usage data (Grafana usage insights in Enterprise/Cloud, or API listing plus `grafana_*` access logs) and find dashboards not viewed in 90 days.
2. Create a folder structure by team and tier: `Platform/`, `Payments/`, with a `Service Overview` per service.
3. Archive (move to `Archive/`) unused dashboards, then delete after 30 days.
4. Move key dashboards into Git (provisioning or Terraform) with owners.
5. Tag dashboards and link them from runbooks.

**Why →** Fewer, owned, linked dashboards are faster during incidents.

**Interview tip →** Mention a "dashboard hierarchy": overview → service → component.

</details>

### Q127. How do you manage Grafana dashboards as code? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual UI edits get lost and cannot be reviewed.

**What I'd do →**
1. Store dashboard JSON in Git; generate it with Grafonnet (Jsonnet) or the Grafana Foundation SDK for reuse.
2. Deploy via file provisioning (`/etc/grafana/provisioning/dashboards`), the Grafana Terraform provider (`grafana_dashboard`), or the Grafana Operator `GrafanaDashboard` CRD.
3. In Kubernetes with kube-prometheus-stack, ship dashboards as ConfigMaps labelled `grafana_dashboard: "1"` picked up by the sidecar.
4. Lock provisioned dashboards (`allowUiUpdates: false`) to force changes through PRs.

**Why →** Reviewable, reproducible and restorable dashboards.

**Interview tip →** Mention that you test dashboards in a staging Grafana before merging.

</details>

### Q128. Design a service overview dashboard that an on-call engineer can read in 10 seconds. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** During incidents, attention is limited. Lead with user impact.

**What I'd do →**
1. Top row: SLO status, error ratio, request rate, p50/p95/p99 latency (RED).
2. Second row: saturation (CPU, memory, pool usage, queue depth) and pod restarts.
3. Third row: dependencies (DB, cache, downstream error rates).
4. Deploy annotations overlaid on graphs.
5. Variables for `env`, `cluster`, `service`; links to logs, traces and runbook.

**Why →** Top-down layout answers "are users hurt?" before "why?".

**Interview tip →** Always mention deploy annotations; most incidents follow a change.

</details>

### Q129. You need to show deploy events on every Grafana graph. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Correlating changes with metrics speeds up root cause analysis.

**What I'd do →**
1. Post annotations from the CD pipeline to the Grafana API:

```bash
curl -s -X POST https://grafana.example.com/api/annotations \
  -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" \
  -d '{"tags":["deploy","checkout"],"text":"checkout v2.14.0 deployed"}'
```

2. Or query a metric such as `kube_deployment_status_observed_generation` changes or Argo CD sync events as an annotation query.
3. Enable the annotation on all service dashboards, filtered by tag.

**Why →** One glance shows if the spike started with a deploy.

**Interview tip →** Annotate feature flag changes and config changes too, not only deploys.

</details>

### Q130. Pods are being OOMKilled but the memory graph never reaches the limit. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scrape intervals miss short spikes, or the wrong memory metric is graphed.

**What I'd do →**
1. Graph `container_memory_working_set_bytes` (what the kubelet and OOM decisions relate to), not `container_memory_usage_bytes` which includes cache.
2. Confirm kills: `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` and `increase(kube_pod_container_status_restarts_total[1h])`.
3. Remember a 30s scrape cannot see a 2-second spike; check `container_oom_events_total` (cAdvisor).
4. Profile the app (heap dumps, JVM `-XX:MaxRAMPercentage`) and set limits based on peak, not average.

**Why →** OOM kills happen in milliseconds; averages and sparse scrapes hide them.

**Interview tip →** Know `working_set` vs `usage` vs `rss`; it comes up often.

</details>

### Q131. How do you detect CPU throttling hurting latency in Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CFS quotas throttle containers even when average CPU looks low.

**What I'd do →**
1. Compute the throttling ratio:

```promql
sum by (namespace, pod) (rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m]))
/
sum by (namespace, pod) (rate(container_cpu_cfs_periods_total{container!=""}[5m]))
```

2. Correlate with p99 latency of the service.
3. Raise or remove CPU limits for latency-sensitive services (keep requests accurate).
4. For multi-threaded runtimes, tune thread counts (`GOMAXPROCS` via automaxprocs, JVM `-XX:ActiveProcessorCount`).

**Why →** Throttling adds latency in 100 ms CFS periods; averages hide it.

**Interview tip →** Mention the debate on CPU limits and why many teams set requests only.

</details>

### Q132. You inherit a cluster with no monitoring. What do you install first? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need fast coverage with sane defaults.

**What I'd do →**
1. Install kube-prometheus-stack via Helm: Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, default rules and dashboards.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

2. Add persistent storage for Prometheus and Grafana.
3. Configure Alertmanager receivers (Slack, PagerDuty).
4. Add a log agent (Alloy/Fluent Bit) to Loki.
5. Review and tune default alerts within the first week.

**Why →** The stack gives node, pod and control plane visibility in an hour.

**Interview tip →** Mention that default alerts need tuning; do not page on all of them.

</details>

### Q133. kube-state-metrics vs metrics-server vs cAdvisor: a junior engineer is confused. Explain with a scenario. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** All three provide "Kubernetes metrics" but serve different purposes.

**What I'd do →**
1. **cAdvisor** (inside kubelet): actual resource usage per container, e.g. `container_cpu_usage_seconds_total`. Use it for "how much CPU is the pod using".
2. **kube-state-metrics**: object state from the API, e.g. `kube_deployment_status_replicas_available`, `kube_pod_status_phase`. Use it for "how many replicas are unavailable".
3. **metrics-server**: short-term CPU/memory for `kubectl top` and HPA; not for Prometheus history.

**Why →** Usage, state, and autoscaling input are separate concerns.

**Interview tip →** "metrics-server is for autoscaling, not monitoring."

</details>

### Q134. Alert when a Deployment has fewer available replicas than desired for 15 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Uses kube-state-metrics.

**What I'd do →**

```yaml
- alert: DeploymentReplicasMismatch
  expr: |
    kube_deployment_spec_replicas{namespace="prod"}
      != kube_deployment_status_replicas_available{namespace="prod"}
  for: 15m
  labels: {severity: ticket}
  annotations:
    summary: "{{ $labels.namespace }}/{{ $labels.deployment }} has unavailable replicas"
```

1. Use `for: 15m` to allow rollouts to finish.
2. Combine with an HPA check so scaling events do not trigger it falsely.

**Why →** Catches stuck rollouts, crash loops and scheduling failures.

**Interview tip →** Mention that kube-prometheus-stack already ships `KubeDeploymentReplicasMismatch`.

</details>

### Q135. Pods are stuck in `Pending` for minutes during peaks. What do you monitor and alert on? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scheduling capacity or autoscaler problems.

**What I'd do →**
1. Alert on `sum by (namespace) (kube_pod_status_phase{phase="Pending"}) > 0` for 10m.
2. Track cluster requests versus allocatable: `sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"})`.
3. Monitor Cluster Autoscaler or Karpenter metrics (e.g. unschedulable pods, provisioning errors).
4. Check events: `kubectl get events --field-selector reason=FailedScheduling`.

**Why →** Pending pods mean lost capacity during exactly the peak you scaled for.

**Interview tip →** Requests-based capacity, not actual usage, determines scheduling.

</details>

### Q136. How do you monitor the Kubernetes control plane on EKS where you cannot access the masters? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Managed control plane, limited metrics.

**What I'd do →**
1. Scrape `apiserver` metrics via the `kubernetes` Service endpoint (`/metrics`) with RBAC permission.
2. Watch `apiserver_request_total` (5xx ratio) and `apiserver_request_duration_seconds` by verb.
3. Enable EKS control plane logs (api, audit, authenticator, controllerManager, scheduler) to CloudWatch.
4. Recent EKS versions expose scheduler and controller manager metrics via `/apis/metrics.eks.amazonaws.com/...`; check your version.
5. Monitor CoreDNS and kubelet yourself.

**Why →** Even managed control planes slow down or throttle; your clients feel it.

**Interview tip →** Mention API server throttling (429) from misbehaving controllers.

</details>

### Q137. DNS lookups inside the cluster intermittently take 5 seconds. How do you monitor and catch this? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic conntrack race or CoreDNS overload, often with `ndots:5` causing many lookups.

**What I'd do →**
1. Monitor CoreDNS: `coredns_dns_request_duration_seconds`, `coredns_dns_responses_total{rcode="SERVFAIL"}`, `coredns_forward_requests_total`.
2. Add a blackbox DNS probe from inside the cluster.
3. Deploy NodeLocal DNSCache to reduce conntrack races and load.
4. Lower `ndots` in pod `dnsConfig` or use FQDNs with trailing dots for external names.
5. Scale CoreDNS (HPA or proportional autoscaler).

**Why →** 5-second delays match the default resolver timeout after a dropped UDP packet.

**Interview tip →** Mentioning the 5s timeout and NodeLocal DNSCache shows real experience.

</details>

### Q138. How do you monitor persistent volume usage for StatefulSets? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** PVs fill silently; databases crash when full.

**What I'd do →**
1. Use kubelet volume metrics: `kubelet_volume_stats_used_bytes` and `kubelet_volume_stats_capacity_bytes`.
2. Alert predictively:

```promql
predict_linear(kubelet_volume_stats_available_bytes{namespace="prod"}[6h], 24*3600) < 0
```

3. Also alert on inode exhaustion (`kubelet_volume_stats_inodes_free`).
4. Enable `allowVolumeExpansion` on the StorageClass for quick fixes.

**Why →** Predictive alerts give time to expand before the outage.

**Interview tip →** Mention inodes; many forget them.

</details>

### Q139. You run 15 clusters. How do you provide one Grafana view across all of them? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many Prometheus instances, need global queries and consistent labels.

**What I'd do →**
1. Add `external_labels: {cluster: "prod-eu-1"}` to every Prometheus.
2. Send to a central store via remote_write (Mimir/VictoriaMetrics) or query via Thanos Query across sidecars.
3. Use a `cluster` variable in all dashboards.
4. Keep alerting local per cluster for resilience, with global alerts for fleet-wide issues.
5. Standardise the stack per cluster with GitOps (Argo CD ApplicationSet).

**Why →** Consistent labels plus a global query layer give one pane of glass.

**Interview tip →** Local alerting keeps working when the central system is down.

</details>

### Q140. Teams complain they cannot see their own dashboards without seeing everyone else's data. How do you set up Grafana access? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need team isolation with shared platform views.

**What I'd do →**
1. Integrate Grafana with SSO (OIDC/SAML) and map IdP groups to Grafana teams.
2. Use folder permissions per team.
3. For data isolation, use tenant-specific data sources (Mimir/Loki `X-Scope-OrgID`) or label-based access control where supported.
4. Use separate Grafana organisations only when isolation must be strict.

**Why →** Folder permissions control dashboards; data source isolation controls data.

**Interview tip →** Note that folder permissions alone do not stop someone querying the data source in Explore.

</details>

### Q141. Nodes become `NotReady` occasionally. What metrics and alerts help you catch the cause? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Often memory pressure, disk pressure, kubelet issues or network.

**What I'd do →**
1. Alert on `kube_node_status_condition{condition="Ready",status="true"} == 0` for 5m.
2. Track `kube_node_status_condition` for `MemoryPressure`, `DiskPressure`, `PIDPressure`.
3. node-exporter: `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`, `node_load5`.
4. Check kubelet logs via journald in Loki: `{unit="kubelet.service"}`.
5. Reserve resources with `systemReserved` and `kubeReserved` in kubelet config.

**Why →** Most NotReady events are starved kubelets.

**Interview tip →** Mention eviction thresholds and system reservations.

</details>

### Q142. A Grafana variable dropdown takes 30 seconds to load. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The variable query scans all series.

**What I'd do →**
1. Replace `label_values(pod)` with a scoped query: `label_values(kube_pod_info{namespace="$namespace"}, pod)`.
2. Chain variables (cluster → namespace → pod).
3. Set refresh to "On dashboard load" instead of "On time range change".
4. Use a small metric (e.g. `kube_pod_info`) rather than a high-cardinality one.

**Why →** Variable queries run against the series index; scoping them reduces work.

**Interview tip →** Small, well-chosen "info" metrics make the best variable sources.

</details>

### Q143. HPA is not scaling a deployment even under load. How do you troubleshoot with monitoring? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** HPA depends on metrics pipelines that may be broken.

**What I'd do →**
1. `kubectl describe hpa api` to see `FailedGetResourceMetric` or similar events.
2. Check metrics-server: `kubectl top pods`.
3. For custom metrics, check prometheus-adapter or KEDA: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1`.
4. Check that pods have CPU requests (HPA percentages need requests).
5. Monitor `kube_horizontalpodautoscaler_status_current_replicas` vs `_spec_max_replicas` to see if it is capped.

**Why →** HPA silently does nothing when its metric source is broken.

**Interview tip →** Missing CPU requests is the most common cause.

</details>

### Q144. How do you monitor cost per namespace in a shared cluster? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Finance wants showback by team.

**What I'd do →**
1. Deploy OpenCost (CNCF) or Kubecost, which combine Prometheus usage data with cloud pricing.
2. Allocate by max(requests, usage) per namespace and label (`team`).
3. Build Grafana dashboards showing cost per team and idle cost.
4. Flag over-requested workloads: requests much higher than p95 usage.

**Why →** Visibility drives right-sizing; idle requests are a big part of Kubernetes waste.

**Interview tip →** Mention "requests drive cost, not usage" in shared clusters.

</details>

### Q145. A team wants a TV dashboard in the office. What do you put on it, and what do you avoid? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Wall dashboards should show health at a glance from across a room.

**What I'd do →**
1. A few big stat panels: SLO status, error rate, request rate, active alerts.
2. Use clear thresholds with colour (green/amber/red).
3. Use kiosk mode (`?kiosk`) and auto-refresh of 1m.
4. Avoid dense graphs, many small panels, and per-pod details.

**Why →** If it cannot be read from 5 metres, it is decoration.

**Interview tip →** Keep incident debugging on service dashboards, not the TV.

</details>

### Q146. How do you monitor Ingress NGINX for an SLO and troubleshooting? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ingress is the front door; it sees all requests.

**What I'd do →**
1. Enable metrics (`controller.metrics.enabled=true`, ServiceMonitor) in the Helm chart.
2. Use `nginx_ingress_controller_requests` (by `ingress`, `status`) and `nginx_ingress_controller_request_duration_seconds_bucket`.
3. Monitor controller health: config reload failures (`nginx_ingress_controller_config_last_reload_successful`), and upstream latency.
4. Ship JSON access logs to Loki.

**Why →** Ingress metrics give a per-route SLI without app changes.

**Interview tip →** Distinguish request duration from upstream response time to see if latency is in NGINX or the app.

</details>

### Q147. CrashLoopBackOff pods go unnoticed for hours in dev and staging. How do you surface them without paging? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-prod issues need visibility but not 3 a.m. pages.

**What I'd do →**
1. Alert on `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1` for 15m.
2. Route `env!="prod"` alerts to a team Slack channel, not PagerDuty.
3. Include namespace, pod and a link to logs in the message.

**Why →** Catching problems in staging prevents them reaching production.

**Interview tip →** Routing by environment is a basic Alertmanager pattern worth showing.

</details>

### Q148. Grafana itself goes down during an incident. How do you make it resilient? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The observability UI is a critical incident tool.

**What I'd do →**
1. Run Grafana with multiple replicas behind a load balancer.
2. Use an external database (PostgreSQL/MySQL) instead of SQLite so state is shared.
3. Provision dashboards and data sources from Git so a rebuild is quick.
4. Run monitoring in a separate cluster or account from the workloads it monitors.
5. Configure HA alerting (`unified_alerting.ha_peers`) if using Grafana Alerting.

**Why →** The tool you need during outages should not fail with the outage.

**Interview tip →** "Don't host your monitoring on the thing it monitors."

</details>

### Q149. How do you alert on Kubernetes Jobs and CronJobs failing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failed jobs leave no running pods to observe.

**What I'd do →**
1. Alert on failed jobs: `kube_job_status_failed > 0`.
2. Alert on CronJobs not succeeding recently:

```promql
time() - max by (namespace, cronjob) (kube_cronjob_status_last_successful_time) > 2 * 3600
```

3. Set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` to keep evidence.
4. Ship job logs before pods are cleaned up.

**Why →** "Last successful time" catches both failures and jobs that never ran.

**Interview tip →** Mention `concurrencyPolicy` and suspended CronJobs as hidden causes.

</details>

### Q150. You need to monitor a service mesh (Istio) without overwhelming Prometheus. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Istio sidecars expose many high-cardinality metrics per workload pair.

**What I'd do →**
1. Use Istio standard metrics (`istio_requests_total`, `istio_request_duration_milliseconds_bucket`) and drop unused ones with relabelling.
2. Remove high-cardinality labels via the Telemetry API (`tagOverrides`) or `metric_relabel_configs`.
3. Pre-aggregate workload-level series with recording rules and use federation or remote_write for only the aggregates.
4. Use mesh-wide dashboards (Kiali, Grafana Istio dashboards).

**Why →** Sidecar metrics can be the largest series source in the cluster.

**Interview tip →** Mention Istio's recommendation to aggregate at workload level for production.

</details>

---

## Long-Term Storage, Scale and Cost

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Prometheus keeps 15 days of data, but leadership wants 13 months of metrics for capacity planning. What do you build? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local Prometheus retention is the wrong tool for a year of data: big disks, slow restarts, no HA for history.

**What I'd do →**
1. Keep Prometheus local retention short (2–15 days) for fast queries and alerting.
2. Add object-storage-backed long-term storage: Thanos sidecar uploading blocks, or `remote_write` to Mimir / a managed service.
3. Enable downsampling (Thanos) and set retention per resolution:
   ```bash
   thanos compact --retention.resolution-raw=30d --retention.resolution-5m=120d --retention.resolution-1h=400d --wait
   ```
4. Create recording rules for the capacity metrics (CPU, memory, requests per service) so year-long queries hit a few pre-aggregated series.
5. Point Grafana's capacity dashboards at the Query Frontend.

**Why →** Object storage is cheap and durable; aggregates keep long-range queries fast.

**Interview tip →** Say clearly that only a small set of aggregated metrics needs 13 months, not every raw series.

</details>

### Q152. The Thanos compactor has halted and the bucket keeps growing. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `thanos_compact_halted == 1` usually means overlapping blocks (two uploaders with the same external labels) or a corrupted block. Without compaction, there is no downsampling or retention, and queries slow down.

**What I'd do →**
1. Read the compactor logs for the block ULIDs and the reason (e.g. `overlaps found while gathering blocks`).
2. Inspect the bucket:
   ```bash
   thanos tools bucket inspect --objstore.config-file=bucket.yml
   thanos tools bucket verify --objstore.config-file=bucket.yml --issues=overlapped_blocks
   ```
3. Find the cause: two Prometheus replicas with identical `external_labels` (missing `replica` label), or a Receive/sidecar misconfiguration.
4. Fix the labels, then mark the bad blocks for deletion or exclude them:
   ```bash
   thanos tools bucket mark --id=<ULID> --marker=deletion-mark.json --details="overlap fix" --objstore.config-file=bucket.yml
   ```
5. Restart the compactor and alert on `thanos_compact_halted` going forward.

**Why →** The compactor halts on purpose to avoid making data corruption worse.

**Interview tip →** Mention that exactly one compactor per bucket is a hard rule.

</details>

### Q153. Grafana shows every series twice after you added a second Prometheus replica with Thanos. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The HA pair scrapes the same targets; Thanos Querier is not deduplicating.

**What I'd do →**
1. Give each replica a distinguishing external label, everything else identical:
   ```yaml
   global:
     external_labels: { cluster: prod-eu, replica: prometheus-0 }
   ```
2. Start the Querier with the replica label:
   ```bash
   thanos query --query.replica-label=replica --endpoint=dnssrv+_grpc._tcp.thanos-store
   ```
3. Make sure Grafana's datasource does not disable deduplication (the `dedup=true` parameter is the default).
4. For the compactor, set `--deduplication.replica-label=replica` to deduplicate stored blocks too.

**Why →** Deduplication merges series that differ only by the replica label and fills gaps from the other replica.

**Interview tip →** With the Prometheus Operator, the `prometheus_replica` label is added automatically; use that as the replica label.

</details>

### Q154. Remote write to your central store is falling further behind every hour. How do you handle it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prometheus is ingesting faster than it can send. If lag exceeds the WAL window (~2 hours), data is lost for the remote store.

**What I'd do →**
1. Measure the lag and shard behaviour:
   ```promql
   prometheus_remote_storage_highest_timestamp_in_seconds
     - ignoring(remote_name, url) group_right
   prometheus_remote_storage_queue_highest_sent_timestamp_seconds
   prometheus_remote_storage_shards_desired / prometheus_remote_storage_shards_max
   rate(prometheus_remote_storage_samples_failed_total[5m])
   ```
2. If `shards_desired` exceeds `max_shards`, raise it and tune batches:
   ```yaml
   queue_config: { max_shards: 100, capacity: 10000, max_samples_per_send: 2000 }
   ```
3. Check the receiver: 429s mean tenant rate limits; 5xx mean the backend is overloaded.
4. Reduce what you send: `write_relabel_configs` to drop unused metrics.
5. Check network latency between regions; remote write is sensitive to round-trip time.

**Why →** Each shard sends one request at a time, so throughput = shards × batch size / latency.

**Interview tip →** Alert on remote write lag above a few minutes, not only on failures.

</details>

### Q155. Mimir rejects writes from a team with `err-mimir-max-series-per-user`. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The tenant has hit its active series limit. New series are dropped; existing ones keep working.

**What I'd do →**
1. Confirm the tenant and growth in Mimir's own metrics (`cortex_ingester_active_series` per user) or the tenant's cardinality page.
2. Find the cause with the cardinality API or `mimirtool`:
   ```bash
   curl -H 'X-Scope-OrgID: team-a' http://mimir/prometheus/api/v1/cardinality/label_names
   ```
3. If it is an accident (new label like `request_id`), drop it at the source or with `write_relabel_configs`.
4. If it is genuine growth, raise the limit in the runtime overrides:
   ```yaml
   overrides:
     team-a:
       max_global_series_per_user: 3000000
   ```
5. Add an alert at 80% of the limit so teams hear about it before rejections.

**Why →** Per-tenant limits protect the shared ingesters from one noisy team.

**Interview tip →** Limits are a feature; the fix is visibility plus a clear process to raise them.

</details>

### Q156. Your observability SaaS bill tripled in a quarter, driven by custom metrics. How do you bring it down? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SaaS vendors bill per unique series (or per host plus custom metrics). Some team added labels with high cardinality, or metrics are shipped that nobody uses.

**What I'd do →**
1. Get a per-metric and per-team cost report from the vendor's usage page.
2. Find unused metrics: compare what is ingested against what dashboards and alerts reference (`mimirtool analyze grafana` and `analyze prometheus`, or the vendor's usage tooling).
3. Drop unused metrics and high-cardinality labels at the agent/Collector, before they leave the cluster.
4. Aggregate where per-pod detail is not needed (drop `pod`, keep `deployment`).
5. Set per-team budgets and show cost in a monthly review.

**Why →** Cost scales with series, and most organisations use well under half of what they ingest.

**Interview tip →** Drop data at the edge; paying to ingest data and then filtering it in the vendor is the expensive way.

</details>

### Q157. You must choose between Thanos and Mimir for a new long-term metrics platform. How do you decide? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both store Prometheus data in object storage and speak PromQL; the difference is operating model.

**What I'd do →**
1. List requirements: number of clusters, tenants, write rate, query patterns, team size, managed options.
2. Choose **Thanos** if: you already run many Prometheus servers, want incremental adoption via sidecars, need downsampling for years of history, and have a small platform team.
3. Choose **Mimir** if: you want a central multi-tenant service with strict per-tenant limits, high write rates via `remote_write`, and fast queries across all clusters.
4. Run a proof of concept with real data: measure ingest cost, p99 query latency on 30-day queries, and operational effort.
5. Consider managed versions (Grafana Cloud, AMP) if the team cannot carry on-call for the store.

**Why →** The best choice is the one your team can operate reliably, not the one with the best benchmark.

**Interview tip →** Mention the hidden cost: Mimir has more components, Thanos has more moving parts at query time across many sidecars.

</details>

### Q158. Object storage for metrics has grown to 400 TB. How do you reduce the cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retention is probably "keep everything forever at raw resolution".

**What I'd do →**
1. Measure size by resolution and tenant (`thanos tools bucket inspect`).
2. Set retention per resolution: raw 30 days, 5m 6 months, 1h 2 years.
3. Check that the compactor actually runs and keeps up; a halted compactor means no retention at all.
4. Move older objects to a cheaper storage class only if the store gateway can still read them (avoid archive classes like Glacier that need restores).
5. Reduce cardinality at the source so fewer series are stored at all.

**Why →** Raw data is the bulk of storage, and almost nobody queries raw data older than a month.

**Interview tip →** Never use bucket lifecycle rules to delete blocks behind the compactor's back; it causes missing-block errors.

</details>

### Q159. Queries over 90 days in Grafana time out on Thanos. How do you make them fast? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The Querier is pulling too many raw chunks from the store gateways.

**What I'd do →**
1. Put the **Query Frontend** in front: split by day and cache results.
   ```bash
   thanos query-frontend --query-range.split-interval=24h --query-range.response-cache-config-file=memcached.yml
   ```
2. Enable automatic downsampling on the Querier (`--query.auto-downsampling`) so 90-day queries use 1h data.
3. Add memcached/Redis index and chunk caches to store gateways.
4. Shard store gateways by time or block hash so each holds a smaller index.
5. Replace heavy panel queries with recording rules.
6. Set a sensible Grafana min interval (e.g. `1h` on long-range dashboards).

**Why →** Long-range queries need fewer, pre-aggregated points, not more CPU.

**Interview tip →** Check `thanos_bucket_store_series_data_touched` to prove where the cost comes from.

</details>

### Q160. A small startup with one Kubernetes cluster asks if it needs Thanos or Mimir. What do you advise? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Probably not yet. Extra components add on-call burden without clear value.

**What I'd do →**
1. Run kube-prometheus-stack with 15–30 days of local retention on a persistent volume.
2. Use `--storage.tsdb.retention.size` so the disk never fills.
3. Run two Prometheus replicas if alerting reliability matters.
4. Set a trigger for revisiting: more than one cluster, a need for history beyond 30 days, or Prometheus above ~16–32 GB of memory.
5. Consider a managed service (Grafana Cloud free tier, AMP) if long retention is needed early.

**Why →** Simple systems are more reliable, and migrating to remote_write later is easy.

**Interview tip →** Interviewers like candidates who do not over-engineer.

</details>

### Q161. You must offer a shared metrics platform to 40 teams with isolation and fair use. How do you design it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs tenant isolation, per-tenant limits, self-service and cost visibility.

**What I'd do →**
1. Use a multi-tenant backend (Mimir or similar); each team is a tenant via `X-Scope-OrgID`.
2. Collect with Alloy/Prometheus Agent per cluster, setting the tenant header based on namespace ownership.
3. Set per-tenant limits: `max_global_series_per_user`, `ingestion_rate`, `max_fetched_series_per_query`.
4. Give each team a Grafana folder and datasource scoped to its tenant; a platform tenant can query across tenants for platform views.
5. Publish usage dashboards (series, samples/s, cost) per tenant and review monthly.
6. Document how to request higher limits.

**Why →** Limits stop one team from hurting all others; visibility makes teams fix their own cardinality.

**Interview tip →** Mention federated querying across tenants for shared services (`tenant-a|tenant-b`).

</details>

### Q162. You need to know which team and which metric generate the most series. How do you find out? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need cardinality attribution before you can manage cost.

**What I'd do →**
1. Prometheus TSDB status:
   ```bash
   curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10], .data.labelValueCountByLabelName[:10]'
   ```
2. Count by namespace and metric:
   ```promql
   topk(20, count by (namespace) ({__name__=~".+"}))
   topk(20, count by (__name__) ({__name__=~".+"}))
   ```
   Run these on a recording rule or off-peak; they are expensive.
3. In Mimir/Grafana Cloud, use the cardinality management dashboards and API.
4. Map namespaces to teams with an ownership label and publish a weekly report.

**Why →** You cannot reduce what you cannot attribute.

**Interview tip →** Mention churn too: `prometheus_tsdb_head_series_created_total` shows short-lived series that never appear in snapshots.

</details>

### Q163. Your team wants to replace Prometheus with VictoriaMetrics to reduce memory. How do you evaluate it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A storage change affects queries, alerts and dashboards. Measure before switching.

**What I'd do →**
1. Dual-write: add `remote_write` from Prometheus to a VictoriaMetrics instance (or run `vmagent` in parallel).
2. Compare memory, CPU and disk for the same ingestion over two weeks.
3. Run the alert rules with `vmalert` and compare firing alerts with Prometheus.
4. Point a copy of key dashboards to VictoriaMetrics and check differences. MetricsQL differs from PromQL in edge cases (e.g. `rate()` extrapolation and lookbehind windows).
5. Check operational fit: backups (`vmbackup`), HA, upgrades and support.

**Why →** Shadow running proves the savings and shows query differences before users notice.

**Interview tip →** Mention that compatibility is high but not 100%; alerts must be validated.

</details>

### Q164. Estimate resources for a Prometheus that will hold 2 million active series with 30-day retention. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Size memory from active series and disk from samples per second.

**What I'd do →**
1. **Samples/s**: 2M series / 30 s interval ≈ 66,700 samples/s.
2. **Disk**: 66,700 × 2,592,000 s (30 days) × ~1.5 bytes ≈ 260 GB, plus WAL and compaction headroom: provision ~400 GB.
3. **Memory**: a few KB per active series ≈ 8–12 GB for the head, plus queries and churn: plan 24–32 GB.
4. **CPU**: 4–8 cores; more for heavy rule evaluation.
5. Validate with a load test (`avalanche` generating synthetic series) and watch `process_resident_memory_bytes`.

**Why →** Estimates prevent OOMs and full disks after go-live.

**Interview tip →** At this size, discuss sharding or a remote store rather than a single huge Prometheus.

</details>

### Q165. Prometheus crashed because its disk filled up. How do you prevent a repeat? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time-based retention alone does not cap disk when series grow.

**What I'd do →**
1. Add size-based retention alongside time-based:
   ```bash
   --storage.tsdb.retention.time=15d --storage.tsdb.retention.size=180GB
   ```
   (keep it at ~80% of the volume size to leave room for WAL and compaction).
2. Alert on disk usage:
   ```promql
   predict_linear(node_filesystem_avail_bytes{mountpoint="/prometheus"}[6h], 24*3600) < 0
   ```
   or `kubelet_volume_stats_available_bytes` for a PVC.
3. Use an expandable StorageClass (`allowVolumeExpansion: true`).
4. Investigate why series grew (`/api/v1/status/tsdb`).

**Why →** The oldest blocks are deleted first when the size limit is reached, which is better than a crash.

**Interview tip →** Size retention is a safety net, not a plan; the real fix is controlling series growth.

</details>

### Q166. How do you back up Prometheus data before a risky migration? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prometheus data is usually re-creatable, but history may be important.

**What I'd do →**
1. Enable the admin API temporarily (`--web.enable-admin-api`).
2. Create a snapshot:
   ```bash
   curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot
   # data goes to <data-dir>/snapshots/<timestamp>-<id>
   ```
3. Copy the snapshot to object storage (`aws s3 sync`).
4. To restore, start Prometheus with the snapshot directory as `--storage.tsdb.path`.
5. Disable the admin API afterwards.

**Why →** Snapshots use hard links, so they are instant and consistent.

**Interview tip →** With Thanos or a remote store, object storage is already the backup.

</details>

### Q167. Cross-AZ data transfer from metrics and logs agents is a top line in your AWS bill. How do you reduce it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Agents in every AZ send uncompressed data to collectors or ingesters in other AZs; replication factor 3 multiplies it.

**What I'd do →**
1. Measure with VPC Flow Logs or Cost Explorer (`DataTransfer-Regional-Bytes`).
2. Enable compression everywhere: OTLP gRPC `compression: gzip` or `zstd`, Loki and remote write already use snappy.
3. Use topology-aware routing so agents send to collectors in the same zone (`service.kubernetes.io/topology-mode: Auto` / `trafficDistribution: PreferClose`).
4. Batch more data per request to improve compression.
5. Drop and aggregate at the edge before sending.
6. For Mimir/Loki, consider zone-aware replication settings and whether replication factor 3 is required for every tenant.

**Why →** Cross-AZ traffic costs money per GB in both directions, and observability traffic is high-volume.

**Interview tip →** Show you know where the bytes flow, not just how much storage costs.

</details>

### Q168. Dev and staging clusters cost as much to monitor as production. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same configuration everywhere means the same cost everywhere.

**What I'd do →**
1. Shorter retention in non-prod (3–7 days for metrics, a few days for logs).
2. Longer scrape intervals (60 s) in dev.
3. Lower trace sampling (1–5%).
4. Drop debug-heavy metrics and DEBUG logs except when a team opts in temporarily.
5. Keep the same dashboards and alert rules (routed to chat, not the pager) so problems are caught before prod.

**Why →** Non-prod needs signal for testing, not prod-level history and resolution.

**Interview tip →** Make settings Helm values per environment, not separate configs.

</details>

### Q169. Leadership wants a yearly availability report per service, but year-long queries time out. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Raw request counters over a year is too much data per query.

**What I'd do →**
1. Create recording rules for SLI ratios at coarse resolution:
   ```yaml
   - record: service:sli_availability:ratio_rate1h
     expr: |
       sum by (service) (rate(http_requests_total{code!~"5.."}[1h]))
       / sum by (service) (rate(http_requests_total[1h]))
   ```
2. Keep these series in long-term storage at full retention.
3. Compute the yearly figure with `avg_over_time(service:sli_availability:ratio_rate1h[365d])`, or better, sum good and total counts separately and divide.
4. Export monthly results to a table or warehouse for reporting.

**Why →** Pre-aggregation turns millions of series into one per service.

**Interview tip →** Note that averaging ratios is wrong when traffic varies; aggregate good/total events.

</details>

### Q170. You need alerts that span all clusters, such as "global checkout error rate". Where do you evaluate them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-cluster Prometheus only sees its own data.

**What I'd do →**
1. Keep most alerts local to each cluster Prometheus (most reliable).
2. Evaluate global rules in the central store: Thanos Ruler against the Querier, or Mimir's ruler.
3. Use recording rules locally to send small aggregates (per-cluster error rates) to the central store.
4. Add a meta-alert for the global ruler itself (rule evaluation failures, `thanos_rule_evaluation_with_warnings_total`).
5. Route global alerts with a `scope="global"` label.

**Why →** Global rules depend on more components; keep them for genuinely global questions.

**Interview tip →** Partial responses from missing clusters can hide problems, so configure the ruler to fail on partial data.

</details>

### Q171. Edge clusters in retail stores have unreliable internet. How do you collect their metrics centrally? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need local buffering and resumable shipping.

**What I'd do →**
1. Run Prometheus in **agent mode** (`--agent`) or Alloy on each edge cluster with `remote_write` to the central store.
2. The WAL buffers data during outages; size its disk for the longest expected outage.
3. Keep a few local alert rules for critical store-level checks (via a small local Prometheus or by syncing alerts).
4. Add `external_labels: {store: "store-142"}`.
5. Alert centrally on stale data per store: `time() - max by (store) (timestamp(up)) > 900`.

**Why →** Agent mode is light and designed for forwarding with WAL-based retries.

**Interview tip →** Mention that the central store must accept out-of-order or late samples (Prometheus/Mimir out-of-order ingestion settings).

</details>

### Q172. Histograms are 60% of all series in your Prometheus. How do you reduce them? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each classic histogram creates one series per bucket per label set; 20 buckets × many labels adds up fast.

**What I'd do →**
1. Find the largest: `topk(10, count by (__name__) ({__name__=~".+_bucket"}))`.
2. Reduce buckets to those around the SLO thresholds (8–12 buckets).
3. Remove high-cardinality labels from histograms (e.g. full path, pod).
4. Drop unused histograms at scrape with `metric_relabel_configs`.
5. Evaluate native histograms, which store all buckets in one series.

**Why →** Histogram cost = buckets × label combinations.

**Interview tip →** Keep the `_count` and `_sum` series even if you drop the buckets; they still give rate and average.

</details>

### Q173. You suspect half of your ingested metrics are never used. How do you prove it and act? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need usage data from dashboards, alerts and ad-hoc queries.

**What I'd do →**
1. Analyse dashboards and rules:
   ```bash
   mimirtool analyze grafana --address=https://grafana.example.com --key=$GRAFANA_TOKEN
   mimirtool analyze prometheus --address=https://mimir.example.com --grafana-metrics-file=metrics-in-grafana.json
   ```
2. Enable query logging to find metrics used in ad-hoc queries.
3. Produce a list of unused metrics with their series counts.
4. Drop them in stages: first in non-prod, then in prod with a communication to teams.
5. Repeat quarterly.

**Why →** Data-driven pruning avoids breaking someone's hidden dashboard.

**Interview tip →** Grafana Cloud's Adaptive Metrics automates the same idea by aggregating unused labels.

</details>

### Q174. Compliance says payment metrics must be kept 2 years, but everything else only 90 days. How do you set it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need different retention for different data.

**What I'd do →**
1. Put payment services in their own tenant (Mimir) and set retention per tenant:
   ```yaml
   overrides:
     payments:
       compactor_blocks_retention_period: 2y
   ```
2. Or send only the required metrics to a separate bucket using a second `remote_write` with `write_relabel_configs`.
3. Document which metrics are covered and why.
4. Test that old data is actually deleted for other tenants.

**Why →** Per-tenant retention avoids storing everything for 2 years.

**Interview tip →** Ask compliance which metrics are really needed; it is usually a small set.

</details>

### Q175. Black Friday traffic is expected to be 5x normal. How do you make sure the observability stack survives? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** More traffic means more logs, spans and possibly series; the monitoring system must not fail when you need it most.

**What I'd do →**
1. Load test the pipeline at 5–7x: `avalanche` for metrics, `k6` or telemetrygen (`telemetrygen traces --rate 5000`) for OTel, synthetic logs for Loki.
2. Scale ahead: more Collector gateway replicas, ingesters, distributors; pre-warm Elasticsearch shards.
3. Check limits: tenant ingestion rates, Loki `ingestion_rate_mb`, Collector `memory_limiter`.
4. Plan a degradation mode: lower trace sampling and drop DEBUG logs with one config change.
5. Freeze observability config changes in the week before.
6. Watch the monitoring system's own dashboards during the event.

**Why →** Observability load grows with traffic, and overload hides the incidents you are there to catch.

**Interview tip →** Mention priority: keep metrics and alerts flowing even if logs and traces must be reduced.

</details>

---

## Cloud Monitoring, Incidents and Migrations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. A new AWS workload uses EC2, an ALB and RDS, with only CloudWatch available. Which alarms do you create first? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start with user-facing symptoms at the load balancer, then key resource limits.

**What I'd do →**
1. ALB: `HTTPCode_Target_5XX_Count` and `HTTPCode_ELB_5XX_Count` (as a ratio of `RequestCount` with metric math), `TargetResponseTime` p99, `UnHealthyHostCount > 0`.
2. RDS: `FreeStorageSpace`, `CPUUtilization`, `DatabaseConnections` near `max_connections`, `ReplicaLag`, `FreeableMemory`.
3. EC2: `StatusCheckFailed`, and install the CloudWatch agent for memory and disk usage.
4. Send all alarms to an SNS topic integrated with the paging tool.
5. Use a composite alarm to page only when user-facing symptoms fire.

```bash
aws cloudwatch put-metric-alarm --alarm-name rds-storage-low \
  --namespace AWS/RDS --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-db \
  --statistic Minimum --period 300 --evaluation-periods 1 \
  --threshold 10737418240 --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:eu-west-1:111122223333:oncall
```

**Why →** The ALB sees what users see; resource alarms give early warning.

**Interview tip →** Mention that EC2 memory is not a default CloudWatch metric.

</details>

### Q177. The CloudWatch bill jumped from $2k to $15k a month. How do you find and cut the cost? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usual suspects: log ingestion, custom metrics with high-cardinality dimensions, and API calls from dashboards or exporters.

**What I'd do →**
1. Cost Explorer filtered on CloudWatch, grouped by usage type (`DataProcessing-Bytes`, `MetricMonitorUsage`, `GMD-Metrics`, `TimedStorage-ByteHrs`).
2. Logs: find the biggest log groups with the `IncomingBytes` metric; reduce DEBUG logging, set retention, use the Infrequent Access class for rarely searched logs.
3. Custom metrics: look for dimensions like `RequestId` or `UserId` that create thousands of metrics.
4. API calls: Grafana dashboards refreshing every 10 s or YACE polling every namespace every minute; reduce frequency and scope.
5. Delete unused alarms and dashboards; set budgets with alerts.

**Why →** CloudWatch bills on several small meters that add up quietly.

**Interview tip →** Set `--retention-in-days` on every log group by default through IaC; "never expire" is the default.

</details>

### Q178. Your company has 50 AWS accounts. How do you get one monitoring view? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each account has its own CloudWatch; switching roles per incident is slow.

**What I'd do →**
1. Set up **CloudWatch cross-account observability**: one monitoring account as the sink, source accounts linked through Observability Access Manager (OAM), ideally automated with AWS Organizations.
2. Share metrics, logs and traces (X-Ray/Application Signals) to the monitoring account.
3. Build cross-account dashboards and alarms there.
4. For an open-source stack, deploy collectors in each account that send to a central AMP/Mimir with an `account` label.
5. Grafana: one datasource per account via assumed roles, or a single central store.

**Why →** One pane of glass speeds up triage across services spread over accounts.

**Interview tip →** Mention least-privilege: the monitoring account reads telemetry, it does not get admin in source accounts.

</details>

### Q179. Your team uses Prometheus and Grafana, but RDS, SQS and ALB metrics live only in CloudWatch. How do you unify? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choose between querying CloudWatch live and copying metrics into Prometheus.

**What I'd do →**
1. For dashboards only: add the Grafana CloudWatch datasource; no copying.
2. For PromQL alerts and joins with app metrics: run YACE scoped by tag and namespace:
   ```yaml
   discovery:
     jobs:
       - type: AWS/SQS
         regions: [eu-west-1]
         searchTags: [{ key: team, value: payments }]
         metrics:
           - { name: ApproximateNumberOfMessagesVisible, statistics: [Maximum], period: 60, length: 300 }
   ```
3. For many accounts or high volume: CloudWatch Metric Streams to a collector.
4. Account for CloudWatch delay (metrics arrive minutes late) in alert `for:` durations.

**Why →** One query language and one alerting system reduce on-call confusion.

**Interview tip →** Mention cost: every polled metric is a billed `GetMetricData` request.

</details>

### Q180. How do you monitor a fleet of AWS Lambda functions? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No servers to watch; focus on invocations, errors, throttles and duration.

**What I'd do →**
1. Alarms on `Errors` / `Invocations` ratio, `Throttles > 0`, `Duration` p99 against the timeout, and `IteratorAge` for stream consumers.
2. Watch account-level `ConcurrentExecutions` against the concurrency quota.
3. Track cold starts from the `REPORT` log line (`Init Duration`) with Logs Insights:
   ```
   filter @type = "REPORT" and ispresent(@initDuration)
   | stats count() as coldStarts, avg(@initDuration) by bin(1h)
   ```
4. Add tracing with ADOT Lambda layer or X-Ray active tracing; structured JSON logs with request IDs.
5. Set log group retention.

**Why →** Most Lambda incidents are throttling, timeouts or downstream failures.

**Interview tip →** Mention DLQs or on-failure destinations and alarming on their depth.

</details>

### Q181. Your RDS PostgreSQL database gets slow every afternoon. What monitoring do you use to find out why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need database load by query and wait event, not only CPU.

**What I'd do →**
1. CloudWatch metrics: `CPUUtilization`, `ReadIOPS`/`WriteIOPS`, `ReadLatency`, `DiskQueueDepth`, `FreeableMemory`, `DatabaseConnections`, and `BurstBalance` or `CPUCreditBalance` on burstable or gp2 storage.
2. Performance Insights / CloudWatch Database Insights: top SQL by load and wait events (`IO:DataFileRead`, `Lock:transactionid`).
3. Enhanced Monitoring for OS-level detail at 1–60 s granularity.
4. Enable `pg_stat_statements` and the slow query log (`log_min_duration_statement = 500`).
5. Correlate with scheduled jobs or reports running in the afternoon.

**Why →** Afternoon slowness is often a batch job, credit exhaustion, or lock contention.

**Interview tip →** Burst credit exhaustion is a classic hidden cause; mention it.

</details>

### Q182. For a new EKS cluster, should you use CloudWatch Container Insights or the Prometheus stack? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both work; the choice depends on team skills, portability and cost.

**What I'd do →**
1. Choose **Container Insights** if: the team lives in AWS, wants minimal operations, and has few clusters. Install via the `amazon-cloudwatch-observability` add-on.
2. Choose **Prometheus** (self-managed or AMP + Managed Grafana) if: you want PromQL, community dashboards and alerts, custom app metrics, or multi-cloud portability.
3. Estimate cost: Container Insights charges per metric and log ingestion; AMP charges per sample ingested and queried.
4. Many teams combine: Prometheus for app and cluster metrics, CloudWatch for AWS service metrics and control-plane logs.

**Why →** Kubernetes ecosystems assume Prometheus; AWS-native options cut operations.

**Interview tip →** Say what you would measure in a two-week trial, not just your preference.

</details>

### Q183. Orders are processed through SQS. How do you detect that consumers are falling behind? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Queue depth alone can be misleading during bursts; message age shows real delay.

**What I'd do →**
1. Alarm on `ApproximateAgeOfOldestMessage` (Maximum) above your processing SLO, e.g. 300 s for 10 minutes.
2. Watch `ApproximateNumberOfMessagesVisible` for trends and capacity planning.
3. Alarm on the DLQ: `ApproximateNumberOfMessagesVisible > 0`.
4. Scale consumers on queue depth (KEDA `aws-sqs-queue` scaler or ECS target tracking).

```bash
aws cloudwatch put-metric-alarm --alarm-name orders-queue-age \
  --namespace AWS/SQS --metric-name ApproximateAgeOfOldestMessage \
  --dimensions Name=QueueName,Value=orders --statistic Maximum \
  --period 60 --evaluation-periods 10 --threshold 300 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:eu-west-1:111122223333:oncall
```

**Why →** Age expresses customer impact directly: "orders are 5 minutes late".

**Interview tip →** Mention that a poisoned message can block FIFO queues; DLQ alerts catch it.

</details>

### Q184. You must migrate 3,000 checks from Nagios/Zabbix to Prometheus. How do you plan it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nagios checks are host-and-threshold based; Prometheus is metrics and labels. A 1:1 port carries old noise over.

**What I'd do →**
1. Inventory checks by type (ping, disk, service, HTTP, custom scripts) and by how often they fired and were actioned last year.
2. Delete checks that never fired or were always ignored.
3. Map types to exporters: node_exporter for host checks, blackbox_exporter for ping/HTTP/TCP, database exporters, textfile collector for custom scripts.
4. Rewrite alerts as symptom-based PromQL rules with `for:` and severity labels; route via Alertmanager.
5. Run both systems in parallel per team; compare alerts for 2–4 weeks.
6. Cut over team by team, then decommission Nagios.

**Why →** Migration is a chance to delete noise, and parallel running proves coverage.

**Interview tip →** Mention the textfile collector as the bridge for legacy check scripts.

</details>

### Q185. You need to move from a self-managed ELK stack to Loki without losing logs or blocking developers. What is your plan? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Query language, retention and habits all change. Big-bang cutovers fail.

**What I'd do →**
1. Prepare Loki with sensible labels (`namespace`, `app`, `env`) and structured metadata for high-cardinality fields.
2. Dual-ship: configure Fluent Bit (or Alloy) with two outputs, Elasticsearch and Loki.
3. Translate key saved searches and alerts from KQL to LogQL; publish a cheat sheet.
4. Run in parallel for one retention period; compare volumes per service.
5. Move alerts first, then dashboards, then switch developers' default.
6. Keep ES read-only until old data expires, or export what compliance needs to object storage.

**Why →** Dual-shipping removes risk; ES ages out naturally.

**Interview tip →** Mention that full-text heavy use cases (security search) may still need an indexed store.

</details>

### Q186. Leadership wants to move from Datadog to OpenTelemetry plus the Grafana stack. How do you migrate without losing visibility? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Instrumentation, dashboards, monitors and on-call habits all depend on Datadog today.

**What I'd do →**
1. Inventory: agents, custom metrics, APM services, monitors, dashboards, SLOs and their owners.
2. Switch instrumentation to OTel SDKs/auto-instrumentation and route through an OTel Collector that exports to **both** Datadog (`datadog` exporter) and the new backends.
3. Build the new stack (Mimir/Prometheus, Loki, Tempo, Grafana); recreate top dashboards and monitors as PromQL/LogQL rules.
4. Run in parallel; compare alert firing and key graphs.
5. Move team by team; turn off Datadog ingestion per service as each team signs off.
6. Track cost and on-call experience before and after.

**Why →** OTel in the middle makes the backend switch reversible.

**Interview tip →** Be honest about cost of ownership: self-hosting replaces license fees with engineering time.

</details>

### Q187. The site is down and you are the first responder. How do you use observability in the first 15 minutes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The goal is to restore service, not to find the root cause.

**What I'd do →**
1. Confirm impact from user-facing signals: synthetic checks, ingress/ALB error rate, traffic drop.
2. Declare an incident, open a channel, assign an incident commander if it is big.
3. Check "what changed": deploy annotations, config changes, feature flags, infra events.
4. Narrow scope with RED dashboards: one service, one region, one dependency?
5. Follow a failing request through a trace to the failing hop; check that service's logs.
6. Mitigate fast: roll back, fail over, scale, disable the feature flag.
7. Post status updates every 15–30 minutes.

**Why →** Most outages are caused by changes, and rollback is faster than diagnosis.

**Interview tip →** Say "mitigate first, root-cause later" out loud.

</details>

### Q188. During a cluster outage, your monitoring went down too because it ran in the same cluster. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Monitoring shared the failure domain with what it monitors.

**What I'd do →**
1. Add out-of-band checks: external synthetic probes (blackbox exporter in another region, or a SaaS uptime checker).
2. Send a Watchdog/dead man's switch alert to an external service (e.g. healthchecks.io or the paging tool's heartbeat) that pages when it stops.
3. Run the central monitoring (Grafana, Alertmanager, long-term store) in a separate management cluster or account.
4. Use `remote_write` so data survives the cluster.
5. Keep runbooks accessible outside the cluster (wiki, not an in-cluster app).

**Why →** You need to see an outage most when everything else is down.

**Interview tip →** "Who monitors the monitoring?" is the question; the answer is an external heartbeat.

</details>

### Q189. A postmortem shows the alert fired correctly but went to a Slack channel nobody watched. What do you improve? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detection worked, notification failed: routing and ownership problem.

**What I'd do →**
1. Route critical alerts to the paging tool (PagerDuty, Opsgenie, Grafana OnCall), not only chat.
2. Require `team` and `severity` labels on every alert; CI rejects rules without them (`promtool` plus a lint check).
3. Add a default route that catches unowned alerts and sends them to the platform team.
4. Review routing quarterly; delete dead channels.
5. Test routing with `amtool config routes test`.

```bash
amtool config routes test --config.file=alertmanager.yml severity=critical team=payments
```

**Why →** An alert nobody sees is the same as no alert.

**Interview tip →** Keep the postmortem blameless: the system allowed the gap.

</details>

### Q190. Users report slowness, but every dashboard is green. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You are measuring the wrong place, the wrong percentile, or averaging away the problem.

**What I'd do →**
1. Ask for specifics: which users, region, page, time.
2. Check client-side signals: RUM data, Core Web Vitals, CDN logs.
3. Look at higher percentiles (p99, p99.9) and break down by region, endpoint and customer.
4. Search traces for slow requests matching the report (TraceQL `{ duration > 3s && resource.service.name = "frontend" }`).
5. Check dependencies outside your dashboards: DNS, CDN, third-party APIs, client networks.
6. Add the missing signal (RUM, per-region SLI) so it shows next time.

**Why →** Server-side averages often hide tail latency and edge-network problems.

**Interview tip →** "Green dashboards with unhappy users means our SLIs are wrong" is a strong answer.

</details>

### Q191. How do you design monitoring that still works if an entire cloud region fails? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A regional outage is exactly when you need monitoring for failover decisions.

**What I'd do →**
1. Run Prometheus/collectors in every region, each alerting locally.
2. Run Alertmanager as a cluster spread across at least two regions, or one cluster per region with deduplication in the paging tool.
3. Store long-term data in a multi-region or replicated bucket.
4. Run Grafana in a second region with a replicated database, or use a managed Grafana.
5. External synthetic probes from several regions and providers.
6. Test it: game days where you cut off one region and confirm alerts, dashboards and runbooks still work.

**Why →** If monitoring lives in the failed region, you are blind during the most critical incident.

**Interview tip →** Mention that the paging provider itself is an external dependency worth considering.

</details>

### Q192. You need to monitor AKS clusters and App Services on Azure. What do you set up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Azure has native options that map well to the open-source stack.

**What I'd do →**
1. Enable **Azure Monitor managed service for Prometheus** on AKS (metrics add-on) and **Azure Managed Grafana** for dashboards.
2. Enable **Container Insights** for container logs into a Log Analytics workspace.
3. Instrument apps with the Azure Monitor OpenTelemetry Distro, sending to **Application Insights**.
4. Create alert rules: Prometheus rule groups for cluster alerts, log search alerts with KQL, and action groups for paging.
5. Set Log Analytics retention and use the Basic/Auxiliary log tiers for high-volume, rarely queried logs.

```kusto
ContainerLogV2
| where LogLevel == "error"
| summarize count() by bin(TimeGenerated, 5m), ContainerName
```

**Why →** Managed Prometheus keeps PromQL skills portable; Log Analytics gives KQL for logs.

**Interview tip →** Mention cost of Log Analytics ingestion and how to control it with data collection rules.

</details>

### Q193. How do you monitor a Cloud Run service on GCP and alert on errors? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cloud Run exports request metrics to Cloud Monitoring automatically.

**What I'd do →**
1. Use built-in metrics: `run.googleapis.com/request_count` (by `response_code_class`), `request_latencies`, `container/instance_count`, `container/cpu/utilizations`.
2. Create an alerting policy on the 5xx ratio (in PromQL, which Cloud Monitoring supports):
   ```promql
   sum(rate(run_googleapis_com:request_count{monitored_resource="cloud_run_revision",response_code_class="5xx"}[5m]))
   / sum(rate(run_googleapis_com:request_count{monitored_resource="cloud_run_revision"}[5m])) > 0.02
   ```
3. Add an **uptime check** against the public URL from multiple regions.
4. Define an SLO in Cloud Monitoring's service monitoring for burn-rate alerts.
5. Use Cloud Trace via OTel for latency breakdown.

**Why →** Managed metrics plus uptime checks cover most serverless needs.

**Interview tip →** Mention that MQL is being phased out in favour of PromQL in Cloud Monitoring.

</details>

### Q194. You run Prometheus in an on-prem datacenter and on AWS. How do you unify metrics? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two environments, networks with firewalls, and one team that needs one view.

**What I'd do →**
1. Keep local Prometheus (or agents) in each environment for scraping and local alerts.
2. `remote_write` from both to one central store (AMP, Mimir or Thanos Receive), over VPN/Direct Connect or HTTPS with auth.
3. Standardise labels: `env`, `site`, `cluster`, `team` via `external_labels`.
4. Use the same recording rules and dashboards in both places.
5. Monitor the link: alert on remote write lag per site.

**Why →** A single store and consistent labels make cross-environment comparison possible.

**Interview tip →** Outbound-only `remote_write` is firewall-friendly compared with inbound scraping.

</details>

### Q195. AWS announced scheduled maintenance for some of your EC2 instances, and nobody noticed. How do you catch these? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS Health events are not part of normal metrics.

**What I'd do →**
1. Create an EventBridge rule for AWS Health events:
   ```json
   { "source": ["aws.health"], "detail-type": ["AWS Health Event"] }
   ```
2. Target an SNS topic, Lambda or chat integration that routes by service and account.
3. Use AWS Health organizational view to cover all accounts.
4. Create tickets automatically for scheduled events with a deadline.

**Why →** Scheduled retirements and maintenance can reboot instances you consider stable.

**Interview tip →** Mention that the same pattern works for GuardDuty findings and RDS events.

</details>

### Q196. A misconfigured autoscaler ran 200 extra instances over a weekend. How do you catch cost spikes early? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cost is a signal too and should be monitored like latency.

**What I'd do →**
1. Enable **AWS Cost Anomaly Detection** with daily alerts to the team channel.
2. Create **AWS Budgets** with alerts at 80% and 100% of forecast.
3. Add a CloudWatch billing alarm on `EstimatedCharges` (in `us-east-1`).
4. Alert on infrastructure metrics that predict cost: instance count per Auto Scaling group (`GroupInServiceInstances`), node count in Kubernetes.
5. Set max sizes on Auto Scaling groups and Karpenter NodePools (`limits`).

**Why →** Billing data is delayed; capacity metrics alert in minutes.

**Interview tip →** Mention tagging to attribute cost to teams.

</details>

### Q197. A team claims a noisy neighbour is slowing down their pods on a shared cluster. How do you prove or disprove it with data? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** "Noisy neighbour" can be CPU, memory bandwidth, disk I/O, or network. Look for contention signals, not just usage.

**What I'd do →**
1. Check CPU throttling on their pods:
   ```promql
   rate(container_cpu_cfs_throttled_periods_total{namespace="team-a"}[5m])
   / rate(container_cpu_cfs_periods_total{namespace="team-a"}[5m])
   ```
2. Check node pressure during slow periods: `node_pressure_cpu_waiting_seconds_total`, `node_pressure_io_waiting_seconds_total` (PSI), disk `node_disk_io_time_seconds_total`.
3. Identify other pods on the same nodes and their usage at the same time (`container_cpu_usage_seconds_total` by node).
4. Correlate the team's latency with those neighbours' usage over several incidents.
5. If proven: set requests and limits correctly, use dedicated node pools or taints, or Guaranteed QoS.

**Why →** PSI metrics show real waiting time caused by contention, which usage graphs do not.

**Interview tip →** Mention that often the "neighbour" turns out to be the team's own low requests.

</details>

### Q198. Leadership asks for MTTD and MTTR numbers after a rough quarter. How do you measure them reliably? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** These numbers need consistent incident timestamps, not memory.

**What I'd do →**
1. Define timestamps: impact start, detection (alert fired or reported), acknowledgement, mitigation, resolution.
2. Record them in the incident tool (PagerDuty, incident.io, Jira) for every incident; impact start comes from metrics, not chat.
3. Compute MTTD = detection − impact start; MTTR (time to restore) = mitigation − impact start.
4. Report median and p90, not just mean, and split by severity.
5. Pair with SLO burn and incident count so a low MTTR does not hide frequent incidents.

**Why →** Clear definitions make the trend meaningful and stop arguments about the numbers.

**Interview tip →** Mention that MTTD is the number observability work improves most directly.

</details>

### Q199. You are moving from self-managed Prometheus to Amazon Managed Service for Prometheus. What are the steps and gotchas? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scraping stays in the cluster; storage, querying and optionally rules move to AMP.

**What I'd do →**
1. Create the workspace and an IAM role with `aps:RemoteWrite` (IRSA or EKS Pod Identity).
2. Add `remote_write` with `sigv4` to the existing Prometheus; switch to agent mode later.
3. Point Grafana to AMP with SigV4 auth; compare dashboards against the old Prometheus.
4. Upload rules and Alertmanager config to AMP (`aws amp create-rule-groups-namespace`, `aws amp create-alert-manager-definition`); AMP Alertmanager sends to SNS, so route SNS to the pager.
5. Check quotas: active series per workspace, ingestion rate, labels per series.
6. Run in parallel, then remove local storage.

**Why →** Parallel writing allows a safe, reversible migration.

**Interview tip →** Mention cost model: per sample ingested and per query sample processed, so heavy dashboards cost money.

</details>

### Q200. You join a company using CloudWatch, Datadog, ELK and Prometheus in different teams. What is your 90-day observability plan? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tool sprawl means high cost, no correlation and inconsistent on-call. Do not start by ripping things out.

**What I'd do →**
1. **Days 1–30, discover**: map tools, cost, owners, critical services, incidents of the last quarter, and how they were detected.
2. Define standards: OpenTelemetry for instrumentation, required labels (`service`, `env`, `team`), alert requirements (severity, runbook, owner).
3. **Days 31–60, quick wins**: SLOs and burn-rate alerts for the top 5 user journeys, fix alert routing, reduce noisy alerts, set log retention.
4. Put an OTel Collector layer in front of all backends so data can be routed anywhere.
5. **Days 61–90, decide**: choose the target platform with cost and team input; start migrating one team; publish a roadmap.
6. Measure success: MTTD, alert volume per on-call shift, cost per service.

**Why →** Standards and a collector layer create leverage before any expensive migration.

**Interview tip →** Show you balance quick wins with long-term platform decisions.

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
