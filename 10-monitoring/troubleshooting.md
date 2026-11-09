# Monitoring & Observability — Troubleshooting Drills

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

> Real monitoring, logging and tracing failures with exact error text, diagnosis commands and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Monitoring & Observability](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Prometheus scraping and targets](#prometheus-scraping-and-targets) — Q1–Q25
- [Prometheus storage, memory and cardinality](#prometheus-storage-memory-and-cardinality) — Q26–Q50
- [PromQL and recording rules](#promql-and-recording-rules) — Q51–Q75
- [Alerting and Alertmanager](#alerting-and-alertmanager) — Q76–Q100
- [Grafana dashboards and data sources](#grafana-dashboards-and-data-sources) — Q101–Q125
- [Kubernetes monitoring and the Prometheus Operator](#kubernetes-monitoring-and-the-prometheus-operator) — Q126–Q150
- [Logging: Loki, Fluent Bit and Elasticsearch](#logging-loki-fluent-bit-and-elasticsearch) — Q151–Q175
- [Tracing, OpenTelemetry and CloudWatch](#tracing-opentelemetry-and-cloudwatch) — Q176–Q200

---

## Prometheus scraping and targets

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A target shows `DOWN` on `/targets` with `Get "http://10.0.3.14:9100/metrics": context deadline exceeded` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The exporter is too slow to answer inside `scrape_timeout` (default 10s), a security group / NetworkPolicy drops packets, or the host is overloaded.

**Diagnose →**
```bash
time curl -s -o /dev/null -w '%{http_code}\n' http://10.0.3.14:9100/metrics
# from inside the Prometheus pod/host
kubectl exec -n monitoring prometheus-k8s-0 -c prometheus -- wget -qO- -T 5 http://10.0.3.14:9100/metrics | head
```
Check `scrape_duration_seconds{instance="10.0.3.14:9100"}` against the timeout.

**Fix →** Open the port (SG/NACL/NetworkPolicy), disable slow collectors (e.g. node_exporter `--no-collector.hwmon` on hosts where it hangs), or raise `scrape_timeout` (must stay ≤ `scrape_interval`).

**Prevent →** Alert on `up == 0` and on `scrape_duration_seconds > 0.8 * scrape_timeout`.

</details>

### Q2. Target is `DOWN` with `server returned HTTP status 401 Unauthorized` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The endpoint needs auth and the scrape config has none, the token/password rotated, or the wrong Secret is mounted.

**Diagnose →**
```bash
curl -i http://app:8080/metrics                     # 401 confirms auth required
curl -i -H "Authorization: Bearer $TOKEN" http://app:8080/metrics
```

**Fix →** Add credentials to the job:
```yaml
- job_name: app
  authorization:
    type: Bearer
    credentials_file: /etc/prometheus/secrets/app-token
  # or basic_auth: { username: prom, password_file: /etc/prometheus/secrets/pw }
```
Use `*_file` fields so rotation does not need a config edit, then reload.

**Prevent →** Alert on `up == 0` per job; keep scrape creds in a secret manager with rotation hooks.

</details>

### Q3. Target shows `server returned HTTP status 403 Forbidden` when scraping kubelet / cAdvisor 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Prometheus ServiceAccount lacks RBAC for `nodes/metrics` or `nodes/proxy`, or the kubelet has webhook authorization on and the token is not bound.

**Diagnose →**
```bash
kubectl auth can-i get nodes/metrics --as=system:serviceaccount:monitoring:prometheus-k8s
kubectl auth can-i get nodes/proxy  --as=system:serviceaccount:monitoring:prometheus-k8s
```

**Fix →** Grant a ClusterRole with `nodes`, `nodes/metrics`, `services`, `endpoints`, `pods` (get/list/watch) and `nonResourceURLs: ["/metrics"]`, bound to the SA. Use `bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token`.

**Prevent →** Install via kube-prometheus-stack, which ships correct RBAC, and test with `kubectl auth can-i` in CI.

</details>

### Q4. Target is `DOWN` with `connect: connection refused` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Nothing listens on that port, the app binds to `127.0.0.1` only, or the exporter crashed.

**Diagnose →**
```bash
ss -lntp | grep 9100
systemctl status node_exporter
curl -v http://<ip>:9100/metrics
```

**Fix →** Start the exporter, bind to `0.0.0.0:<port>` (e.g. `--web.listen-address=:9100`), or correct the port in the scrape config / Service.

**Prevent →** Run exporters under systemd with `Restart=always`; alert on `up == 0` for 5m.

</details>

### Q5. Target shows `x509: certificate signed by unknown authority` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The endpoint uses an internal CA that Prometheus does not trust.

**Diagnose →**
```bash
openssl s_client -connect app:8443 -showcerts </dev/null | openssl x509 -noout -issuer -subject -dates
```

**Fix →**
```yaml
scheme: https
tls_config:
  ca_file: /etc/prometheus/certs/internal-ca.pem
  server_name: app.internal
```
`insecure_skip_verify: true` only as a short-term workaround.

**Prevent →** Distribute the internal CA bundle via cert-manager `trust-manager` or config management.

</details>

### Q6. Target shows `x509: certificate is valid for app.svc, not 10.0.4.22` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service discovery gives Prometheus a pod IP, but the cert SANs only contain DNS names.

**Diagnose →** Compare the `__address__` on `/targets` with SANs: `openssl s_client -connect 10.0.4.22:8443 </dev/null | openssl x509 -noout -ext subjectAltName`.

**Fix →** Set `tls_config.server_name: app.svc` so verification uses the DNS name while connecting to the IP, or add IP SANs to the cert.

**Prevent →** Standardise on `server_name` for every TLS job scraped by IP.

</details>

### Q7. A target is `UP` but `/targets` shows `expected a valid start token, got "<" ("INVALID") while parsing: "<"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The path returns HTML (login page, 404 page, SPA index) instead of exposition format. Usually wrong `metrics_path` or an auth proxy redirect.

**Diagnose →**
```bash
curl -sL http://app:8080/metrics | head -5
curl -sI http://app:8080/metrics      # look for 302 to /login
```

**Fix →** Point `metrics_path` at the real endpoint (`/actuator/prometheus` for Spring Boot, `/metrics` for most others), or exempt it from the auth proxy.

**Prevent →** Add a smoke test that runs `promtool check metrics` against the endpoint in CI.

</details>

### Q8. A target disappeared from `/targets` entirely after a config change 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `relabel_configs` `keep`/`drop` rule filtered it, the reload failed silently, or service discovery no longer finds it.

**Diagnose →** Open `/service-discovery` in the UI: dropped targets are listed with their discovered labels. Then:
```bash
promtool check config /etc/prometheus/prometheus.yml
curl -s localhost:9090/api/v1/status/config | head
```
Check `prometheus_config_last_reload_successful`.

**Fix →** Correct the relabel regex (anchored by default: `regex: web` does not match `web-1`), fix the config and reload with `curl -X POST localhost:9090/-/reload` (needs `--web.enable-lifecycle`).

**Prevent →** Run `promtool check config` in CI and alert on `prometheus_config_last_reload_successful == 0`.

</details>

### Q9. Prometheus logs `Error reloading config ... "job_name" is duplicated` and keeps the old config 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two scrape jobs share a `job_name` (often from merged include files or Helm `additionalScrapeConfigs`).

**Diagnose →**
```bash
promtool check config prometheus.yml
grep -n 'job_name' prometheus.yml | sort -k3 | uniq -d -f2
```

**Fix →** Rename one job; job names must be unique across all `scrape_configs` and `scrape_config_files`.

**Prevent →** Gate merges with `promtool check config`; alert on failed reloads.

</details>

### Q10. Scrape errors show `body size limit exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `body_size_limit` is set on the job and the exporter response grew past it, usually due to a new high-cardinality metric.

**Diagnose →**
```bash
curl -s http://app:8080/metrics | wc -c
curl -s http://app:8080/metrics | grep -v '^#' | cut -d'{' -f1 | sort | uniq -c | sort -rn | head
```

**Fix →** Remove the offending labels in the app, or drop them via `metric_relabel_configs`. Only raise the limit once the growth is understood.

**Prevent →** Keep `body_size_limit` and `sample_limit` on every job as a guardrail.

</details>

### Q11. Scrape fails with `sample limit exceeded` and the whole target goes to `up == 0` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target exposes more series than `sample_limit`. When the limit is exceeded, the whole scrape is rejected, not just the excess.

**Diagnose →**
```promql
scrape_samples_scraped{job="app"}
scrape_samples_post_metric_relabeling{job="app"}
```
Inspect which metric names exploded with the `curl | cut | uniq -c` trick.

**Fix →** Drop or aggregate the high-cardinality series with `metric_relabel_configs` (`action: drop` or `labeldrop`), fix instrumentation, then adjust the limit deliberately.

**Prevent →** Alert when `scrape_samples_scraped / <limit> > 0.8`.

</details>

### Q12. Prometheus logs `Error on ingesting samples with different value but same timestamp` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two targets produce identical label sets after relabeling (e.g. you dropped `instance` or `pod`), or one exporter emits the same series twice with explicit timestamps.

**Diagnose →**
```promql
rate(prometheus_target_scrapes_sample_duplicate_timestamp_total[5m]) > 0
```
Check `/targets` for two endpoints with the same `job`/`instance` after relabeling.

**Fix →** Keep a unique label per target (`instance`, `pod`). Fix exporters that print a series twice.

**Prevent →** Never `labeldrop` identity labels; review relabel rules against `/service-discovery` output.

</details>

### Q13. Prometheus logs `Error on ingesting out-of-order samples` with `num_dropped=...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An exporter attaches its own timestamps that go backwards, two sources write the same series (duplicate targets or two remote-writers), or a pushed timestamp is older than the TSDB head allows.

**Diagnose →**
```promql
rate(prometheus_target_scrapes_sample_out_of_order_total[5m])
rate(prometheus_tsdb_out_of_order_samples_total[5m])
```
`curl` the target and look for explicit timestamps at the end of sample lines.

**Fix →** Remove explicit timestamps (or set `honor_timestamps: false`), dedupe targets, and give each writer a unique external label. If legitimate late data exists, enable `storage.tsdb.out_of_order_time_window` (e.g. `30m`).

**Prevent →** One series, one writer. Alert on the counters above.

</details>

### Q14. Remote write receiver (Mimir/Cortex/Thanos Receive) returns `out of order sample` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two Prometheus replicas in an HA pair write identical series without dedup, or a replica restarted and replays old WAL data after newer samples were accepted.

**Diagnose →** Check the sender: `prometheus_remote_storage_samples_failed_total` and logs `server returned HTTP status 400 Bad Request: ... out of order sample`. Confirm both replicas have the same `external_labels`.

**Fix →** Give HA replicas `external_labels: {cluster: prod, __replica__: a|b}` and enable HA dedup in the receiver (Mimir: `accept_ha_samples` with `ha_cluster_label`/`ha_replica_label`). Or enable out-of-order ingestion on the receiver for a bounded window.

**Prevent →** Treat 400s as non-retryable data loss: alert on `rate(prometheus_remote_storage_samples_failed_total[5m]) > 0`.

</details>

### Q15. EC2 service discovery finds no targets; logs show `could not describe instances: AccessDenied` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Prometheus IAM role lacks `ec2:DescribeInstances` / `ec2:DescribeAvailabilityZones`, or the wrong region is configured.

**Diagnose →**
```bash
aws sts get-caller-identity
aws ec2 describe-instances --region ap-south-1 --max-items 1
```

**Fix →** Attach a policy with `ec2:DescribeInstances` and `ec2:DescribeAvailabilityZones`; set `region:` in `ec2_sd_configs`; filter with `filters: [{name: "tag:monitor", values: ["true"]}]`.

**Prevent →** Alert on `prometheus_sd_discovered_targets` dropping to 0 for a job.

</details>

### Q16. EC2 targets are discovered but all scrape the public IP and time out 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Relabeling uses `__meta_ec2_public_ip`, or the default address is wrong for your network path.

**Diagnose →** Look at discovered labels on `/service-discovery` and compare `__address__` with `__meta_ec2_private_ip`.

**Fix →**
```yaml
relabel_configs:
  - source_labels: [__meta_ec2_private_ip]
    target_label: __address__
    replacement: '${1}:9100'
  - source_labels: [__meta_ec2_tag_Name]
    target_label: instance
```

**Prevent →** Template SD jobs and review relabeling against the SD page before merging.

</details>

### Q17. `file_sd_configs` targets never change even though the JSON file was updated 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file was written outside the watched glob, the JSON is invalid (Prometheus keeps the last good version), or a ConfigMap update has not propagated yet.

**Diagnose →**
```bash
jq . /etc/prometheus/file_sd/nodes.json
journalctl -u prometheus | grep -i 'file_sd\|discovery'
```
Check `prometheus_sd_file_read_errors_total`.

**Fix →** Write files atomically (write temp, then `mv`), fix JSON, and make sure the glob in `files:` matches.

**Prevent →** Alert on `increase(prometheus_sd_file_read_errors_total[10m]) > 0`.

</details>

### Q18. Blackbox exporter probe shows `probe_success 0` but the site loads in a browser 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module expects a different status code, follows no redirects, fails on IPv6 preference, or the probe target is passed as `__address__` instead of `__param_target`.

**Diagnose →**
```bash
curl 'http://blackbox:9115/probe?module=http_2xx&target=https://example.com&debug=true'
```
The debug output shows each step and the failing reason.

**Fix →** Use correct relabeling (`__address__ → __param_target`, then `__address__` = blackbox host), set `preferred_ip_protocol: ip4`, and `valid_status_codes: [200, 301, 302]` if needed.

**Prevent →** Keep a debug probe link in each alert's annotations.

</details>

### Q19. node_exporter scrape takes 25 seconds on some hosts 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A collector is stuck: `filesystem` on a hung NFS mount, `hwmon`, or a huge textfile directory.

**Diagnose →**
```promql
topk(5, node_scrape_collector_duration_seconds)
node_scrape_collector_success == 0
```
```bash
timeout 5 stat -f /mnt/nfs || echo "mount hung"
```

**Fix →** Exclude the mount: `--collector.filesystem.mount-points-exclude='^/(mnt/nfs|proc|sys)($|/)'`, or disable the slow collector.

**Prevent →** Alert on per-collector duration and failures.

</details>

### Q20. Pushgateway shows stale metrics from a batch job that stopped running a week ago 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pushgateway never expires groups; the last push persists until deleted.

**Diagnose →**
```bash
curl -s http://pushgateway:9091/metrics | grep push_time_seconds
```

**Fix →** Delete the group: `curl -X DELETE http://pushgateway:9091/metrics/job/nightly_backup/instance/db1`. Alert on freshness instead of the value:
```promql
time() - push_time_seconds{job="nightly_backup"} > 26*3600
```

**Prevent →** Use `honor_labels: true` on the Pushgateway job and only use it for batch jobs, never for long-running services.

</details>

### Q21. All Pushgateway metrics show `instance="pushgateway:9091"` and `job="pushgateway"` instead of the pushed values 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `honor_labels` is `false`, so the scraped `job`/`instance` labels are renamed to `exported_job`/`exported_instance`.

**Diagnose →** Query `{exported_job!=""}` and you will see the original labels.

**Fix →** Set `honor_labels: true` on the Pushgateway scrape job (or `honorLabels: true` in the ServiceMonitor endpoint).

**Prevent →** Template the Pushgateway job once and reuse it.

</details>

### Q22. Federation job returns `context deadline exceeded` from the global Prometheus 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `match[]` selector pulls too many series (e.g. `{job=~".+"}`), so `/federate` cannot respond in time.

**Diagnose →**
```bash
time curl -sG 'http://prom-a:9090/federate' --data-urlencode 'match[]={__name__=~"job:.*"}' | wc -l
```

**Fix →** Federate only aggregated recording rules (`match[]={__name__=~"job:.*|cluster:.*"}`), raise `scrape_timeout` modestly, and shard by job.

**Prevent →** For global views at scale use Thanos/Mimir remote write or sidecar instead of federating raw series.

</details>

### Q23. `up` is 1 but a specific metric is missing from the target 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app has not registered the metric yet (lazy init until first event), a `metric_relabel_configs` rule dropped it, or the name changed in a library upgrade (e.g. `_total` suffix added).

**Diagnose →**
```bash
curl -s http://app:8080/metrics | grep -i request
```
Compare with the `metric_relabel_configs` of the job.

**Fix →** Initialise counters at zero for known label values in code, fix the drop rule, update queries to the new name.

**Prevent →** Pre-initialise metrics and test exposition in CI.

</details>

### Q24. Prometheus logs `Scrape commit failed ... err="out of bounds"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Samples carry timestamps older than the head block's lower bound, usually from exporters with explicit old timestamps or a host clock far behind.

**Diagnose →**
```bash
timedatectl status          # on Prometheus and target hosts
chronyc tracking
```
```promql
rate(prometheus_target_scrapes_sample_out_of_bounds_total[5m])
```

**Fix →** Fix NTP/chrony on hosts, set `honor_timestamps: false` for exporters with bad timestamps.

**Prevent →** Alert on `node_timex_offset_seconds` and enforce chrony everywhere.

</details>

### Q25. `/targets` shows hundreds of targets for a job that should have 6 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kubernetes SD with `role: pod` scrapes every container port of every pod, or no `keep` rule on annotations/labels.

**Diagnose →** Look at `/service-discovery` labels: `__meta_kubernetes_pod_container_port_number`, `__meta_kubernetes_namespace`.

**Fix →**
```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: "true"
  - source_labels: [__meta_kubernetes_pod_container_port_name]
    action: keep
    regex: metrics
```

**Prevent →** Prefer `role: endpoints` or ServiceMonitors with named ports.

</details>

---

## Prometheus storage, memory and cardinality

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Prometheus pod restarts with `OOMKilled` every few hours 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many active series in the head block (cardinality growth), heavy queries loading long ranges, or memory limits sized for last year's load.

**Diagnose →**
```bash
kubectl describe pod prometheus-k8s-0 -n monitoring | grep -A3 'Last State'
```
```promql
prometheus_tsdb_head_series
topk(10, count by (__name__)({__name__=~".+"}))
```
Use **Status → TSDB Status** for top series by metric and label.

**Fix →** Drop the offending series with `metric_relabel_configs`, add query limits (`--query.max-samples`), and size memory at roughly a few KB per active series plus query headroom.

**Prevent →** Alert on `prometheus_tsdb_head_series` growth and per-job `scrape_series_added`.

</details>

### Q27. A query fails with `query processing would load too many samples into memory in query execution` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query touches more samples than `--query.max-samples` (default 50,000,000), usually a long range over a high-cardinality metric with a small step.

**Diagnose →** Re-run with a shorter range and check `count(<selector>)` to see the series count.

**Fix →** Add label filters, aggregate earlier (`sum by (job) (rate(...))`), use a recording rule, or increase the step. Raise the limit only if memory allows.

**Prevent →** Back expensive dashboard panels with recording rules.

</details>

### Q28. Head series jumped from 2M to 9M after a deploy 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A new label with unbounded values: `user_id`, `request_id`, raw URL path, `trace_id`, error message, or pod churn from a crash loop.

**Diagnose →**
```promql
topk(10, sum by (job) (scrape_series_added))
count by (__name__)({job="checkout"})
count(count by (path) (http_requests_total{job="checkout"}))
```
TSDB Status page lists "Top 10 label names with high value count".

**Fix →** Short term: drop the label at ingest:
```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: path
```
Long term: route templating in code (`/users/:id`, not `/users/42`) and roll back the instrumentation change.

**Prevent →** `sample_limit` per job, a cardinality budget per team, and review of new metrics in PRs.

</details>

### Q29. Prometheus logs `opening storage failed: ... unexpected EOF` / `corruption in segment` and won't start 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** WAL corruption after an unclean shutdown, node crash, full disk, or a storage layer that does not honour fsync (NFS).

**Diagnose →**
```bash
kubectl logs prometheus-k8s-0 -c prometheus --previous | grep -i 'wal\|corrupt'
ls -la /prometheus/wal/
df -h /prometheus
```

**Fix →** Recent versions attempt WAL repair automatically and truncate at the corrupt record. If it still fails, back up the directory, then move the bad segment(s) out of `wal/` (you lose recent un-compacted samples, up to ~2h) and restart.

**Prevent →** Use local SSD or block volumes (EBS), never NFS; keep disk headroom; give pods a long `terminationGracePeriodSeconds` so the WAL closes cleanly.

</details>

### Q30. Prometheus is in CrashLoopBackOff with `no space left on device` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Retention by time only (`--storage.tsdb.retention.time=30d`) while ingest grew, so disk filled before old blocks expired.

**Diagnose →**
```bash
kubectl exec prometheus-k8s-0 -c prometheus -- du -sh /prometheus/* | sort -h
```
```promql
prometheus_tsdb_storage_blocks_bytes
```

**Fix →** Expand the PVC (if the StorageClass allows `allowVolumeExpansion`), or delete the oldest block directories, then add `--storage.tsdb.retention.size=180GB` (about 80–85% of the volume).

**Prevent →** Set both retention flags and alert on `predict_linear(kubelet_volume_stats_available_bytes[6h], 24*3600) < 0`.

</details>

### Q31. Prometheus takes 15 minutes to become ready after a restart 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** WAL replay of a large head block. Time scales with active series and WAL size.

**Diagnose →** Logs show `Replaying WAL` progress; check `du -sh /prometheus/wal` and `prometheus_tsdb_head_series`.

**Fix →** Reduce cardinality, give more CPU during startup, enable `--enable-feature=memory-snapshot-on-shutdown` so clean restarts load a snapshot instead of replaying the full WAL.

**Prevent →** Raise readiness probe `failureThreshold` so Kubernetes does not kill it mid-replay, and run HA pairs.

</details>

### Q32. Prometheus is killed by the liveness probe during startup in a loop 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** WAL replay exceeds the probe budget, the kubelet restarts it, replay starts again from scratch.

**Diagnose →** `kubectl describe pod` shows `Liveness probe failed` followed by restarts; logs end during `Replaying WAL`.

**Fix →** Add a `startupProbe` with a generous `failureThreshold * periodSeconds` (e.g. 60 × 15s), or temporarily remove the liveness probe.

**Prevent →** Size probes for worst-case replay; the Prometheus Operator already sets a startupProbe in recent versions.

</details>

### Q33. Compaction fails with `compaction failed ... overlapping blocks` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Blocks with overlapping time ranges, often from backfilling with `promtool tsdb create-blocks-from` or copying blocks from another instance.

**Diagnose →**
```bash
promtool tsdb list /prometheus
```
Look for blocks whose MinTime/MaxTime ranges overlap.

**Fix →** Vertical compaction handles overlaps in Prometheus 2.39+ by default. On older versions, remove or move the overlapping backfilled blocks and re-backfill into non-overlapping windows.

**Prevent →** Backfill into a staging directory, validate with `promtool tsdb analyze`, then move blocks in.

</details>

### Q34. Remote write logs `Skipping resharding, last successful send was beyond threshold` and the queue keeps growing 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The receiver is slow or rejecting, network latency is high, or `max_shards` is too low for the ingest rate.

**Diagnose →**
```promql
prometheus_remote_storage_highest_timestamp_in_seconds - ignoring(remote_name, url) group_right prometheus_remote_storage_queue_highest_sent_timestamp_seconds
prometheus_remote_storage_shards_desired
rate(prometheus_remote_storage_samples_retried_total[5m])
```

**Fix →** Fix receiver health first; then tune `queue_config` (`max_shards`, `capacity`, `max_samples_per_send: 2000+`). Drop unneeded series with `write_relabel_configs`.

**Prevent →** Alert when remote write lag exceeds a few minutes; the WAL only covers ~2h of buffering.

</details>

### Q35. Remote write logs `server returned HTTP status 429 Too Many Requests: per-user series limit of 150000 exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tenant hit the receiver's per-tenant series or ingestion rate limit (Mimir/Cortex/Grafana Cloud).

**Diagnose →** Check `prometheus_remote_storage_samples_failed_total` and receiver per-tenant limit dashboards; compare with `prometheus_tsdb_head_series`.

**Fix →** Drop unneeded series via `write_relabel_configs`, or raise `max_global_series_per_user` / ingestion rate for the tenant.

**Prevent →** Track per-tenant usage vs limits and alert at 80%.

</details>

### Q36. Disk usage on Prometheus grows faster than ingestion suggests 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Compaction is failing so many small 2h blocks pile up, churn is high (new series every deploy), or retention is not applied.

**Diagnose →**
```promql
increase(prometheus_tsdb_compactions_failed_total[1d])
rate(prometheus_tsdb_head_series_created_total[1h])
```
```bash
promtool tsdb list -r /prometheus | head
```

**Fix →** Resolve compaction errors (check logs), reduce churn (drop `pod_template_hash`-style labels when not needed), verify retention flags.

**Prevent →** Alert on compaction failures and on series churn.

</details>

### Q37. `promtool tsdb analyze` shows a label with 1.2M unique values 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unbounded value used as a label: IPs, session IDs, full URLs, timestamps.

**Diagnose →**
```bash
promtool tsdb analyze /prometheus | sed -n '/Highest cardinality labels/,/^$/p'
```

**Fix →** Remove it from instrumentation; meanwhile `labeldrop` it or `drop` the metric. If you need the detail, put it in logs or traces and link via exemplars.

**Prevent →** Lint metrics in code review; labels should have bounded values (tens to hundreds).

</details>

### Q38. Histogram metrics alone account for 60% of all series 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Classic histograms with many buckets (×labels). 20 buckets × 50 routes × 10 status codes × 30 pods = 300k series for one metric.

**Diagnose →**
```promql
topk(10, count by (__name__)({__name__=~".+_bucket"}))
```

**Fix →** Cut buckets to what your SLOs need, drop the `le` buckets you never query, remove `status` from histograms (keep it on counters), or migrate to native histograms (`--enable-feature=native-histograms`, one series per histogram).

**Prevent →** Standard bucket presets in a shared instrumentation library.

</details>

### Q39. Prometheus CPU is pegged at 100% but ingestion is flat 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expensive queries: dashboards auto-refreshing every 5s over 7d, regex matchers like `{__name__=~".*"}`, or slow rule groups.

**Diagnose →** Enable `--query.active-query-tracker`/query log (`global.query_log_file`), then:
```promql
topk(10, prometheus_rule_group_last_duration_seconds)
rate(prometheus_engine_query_duration_seconds_sum[5m])
```

**Fix →** Increase dashboard refresh intervals, replace hot queries with recording rules, and kill bad ones.

**Prevent →** Put a query frontend (Thanos/Mimir) with caching in front for heavy dashboards.

</details>

### Q40. Thanos Compactor logs `critical error detected; halting` and stops compacting 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Overlapping blocks in the bucket (two sidecars with identical external labels), or a corrupt block.

**Diagnose →**
```bash
thanos tools bucket verify --objstore.config-file=bucket.yml
thanos tools bucket inspect --objstore.config-file=bucket.yml
```
Check `thanos_compact_halted == 1`.

**Fix →** Give each Prometheus unique `external_labels` (including `replica`), mark the bad block with `thanos tools bucket mark --marker=no-compact-mark.json` or delete it, then restart the compactor. Use `--deduplication.replica-label=replica`.

**Prevent →** Alert on `thanos_compact_halted`, and run exactly one compactor per bucket.

</details>

### Q41. Thanos Query shows duplicated series from HA Prometheus pairs 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The replica label name passed to Querier does not match the external label used by Prometheus.

**Diagnose →** Query the metric and look for two series differing only by `prometheus_replica` (or `replica`).

**Fix →** Start Querier with `--query.replica-label=prometheus_replica` (match exactly) and keep dedup enabled in the UI.

**Prevent →** Standardise external label names across clusters.

</details>

### Q42. Thanos Store Gateway is OOMKilled on startup 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It loads index headers for every block in a large bucket, or blocks were never downsampled/compacted.

**Diagnose →** Count blocks with `thanos tools bucket inspect`; check `thanos_bucket_store_blocks_loaded`.

**Fix →** Shard stores by time (`--min-time=-30d --max-time=-2d`) or by hash relabel, enable compactor downsampling, give the store fast local disk for index-header cache.

**Prevent →** Keep the compactor healthy; it is the main lever on store memory.

</details>

### Q43. Prometheus logs `too many open files` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process file descriptor limit is too low for the number of blocks, WAL segments and scrape connections.

**Diagnose →**
```bash
cat /proc/$(pgrep prometheus)/limits | grep 'open files'
ls /proc/$(pgrep prometheus)/fd | wc -l
```
```promql
process_open_fds / process_max_fds
```

**Fix →** Set `LimitNOFILE=65536` in the systemd unit or raise container ulimits.

**Prevent →** Alert on `process_open_fds / process_max_fds > 0.8`.

</details>

### Q44. After a Prometheus upgrade, queries for `http_requests_total` return nothing but `http_requests_total_total` exists 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app's client library or an OTel-to-Prometheus conversion appended `_total` to a counter that already ended in `_total`, or a library upgrade changed naming.

**Diagnose →**
```promql
{__name__=~"http_requests.*"}
```

**Fix →** Rename the counter in code to `http_requests` (library adds `_total`), or set the OTel exporter/translation option correctly. Update dashboards with a temporary `or` query during migration.

**Prevent →** Pin client libraries and review metric names after upgrades.

</details>

### Q45. Recording rule results disappear for 5 minutes after each Prometheus restart 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Normal behaviour: rule evaluation needs enough raw data in the range (`rate(x[5m])` needs 2 samples inside 5m), and gaps appear while Prometheus is down.

**Diagnose →** Compare the gap with the pod restart time.

**Fix →** Run Prometheus as an HA pair and query through Thanos/Mimir with dedup so one replica covers the other's gap.

**Prevent →** Use `for:` on alerts so a short gap does not resolve and re-fire alerts.

</details>

### Q46. `prometheus_tsdb_head_series` is stable, but `prometheus_tsdb_head_series_created_total` rate is very high 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Series churn: labels whose value changes over time (pod names in constant redeploys, `version` labels, Kubernetes Jobs/CronJobs). Each new series costs memory until the head is compacted.

**Diagnose →**
```promql
topk(10, sum by (job) (rate(scrape_series_added[1h])))
```

**Fix →** Drop churning labels you do not query, aggregate away `pod` in recording rules, stop CronJob pods from being scraped individually if not needed.

**Prevent →** Track churn per job as a capacity metric.

</details>

### Q47. VictoriaMetrics / Prometheus-compatible backend logs `cannot parse sample: too long label value` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A label carries a huge string (SQL query, stack trace, JSON blob).

**Diagnose →** Find the source: `curl -s app/metrics | awk 'length > 1000' | head`.

**Fix →** Remove that label in the app; drop it at scrape with `labeldrop`. Configure `label_value_length_limit` on the scrape job so Prometheus rejects it early.

**Prevent →** Set `label_limit`, `label_name_length_limit` and `label_value_length_limit` as global defaults.

</details>

### Q48. Snapshot via `/api/v1/admin/tsdb/snapshot` returns `404 page not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Admin APIs are disabled by default.

**Diagnose →** `curl -XPOST localhost:9090/api/v1/admin/tsdb/snapshot` returns 404 or "admin APIs disabled".

**Fix →** Start with `--web.enable-admin-api` (Operator: `enableAdminAPI: true`), take the snapshot, then disable again.

**Prevent →** Keep admin APIs off in normal operation and protect the endpoint.

</details>

### Q49. Deleting a series with the admin API did not free disk 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `delete_series` only writes tombstones; data is removed at the next compaction.

**Diagnose →** `promtool tsdb list` shows blocks unchanged in size.

**Fix →**
```bash
curl -XPOST 'localhost:9090/api/v1/admin/tsdb/clean_tombstones'
```

**Prevent →** Drop bad series at ingest instead of deleting after the fact.

</details>

### Q50. Prometheus on Kubernetes loses all data on every pod reschedule 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Storage is `emptyDir` because no `storageSpec`/PVC was configured.

**Diagnose →**
```bash
kubectl get pod prometheus-k8s-0 -n monitoring -o jsonpath='{.spec.volumes}' | jq
kubectl get pvc -n monitoring
```

**Fix →** In the Prometheus CR set:
```yaml
storage:
  volumeClaimTemplate:
    spec:
      storageClassName: gp3
      resources: { requests: { storage: 200Gi } }
```

**Prevent →** Make persistent storage a required value in your Helm values review.

</details>

---

## PromQL and recording rules

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. A PromQL query in the UI returns `Empty query result` for a metric you can see on `/metrics` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target is not actually scraped by this Prometheus, label values differ from what you typed (`job="api"` vs `job="api-svc"`), or the time range is before the target was added.

**Diagnose →**
```promql
up{job=~".*api.*"}
count by (job, instance) ({__name__="http_requests_total"})
```
Drop label matchers one at a time until results appear.

**Fix →** Use the real label values (autocomplete in Grafana Explore helps), or add the scrape job.

**Prevent →** Use label value variables in dashboards instead of hardcoded strings.

</details>

### Q52. `rate(http_requests_total[1m])` returns nothing but `http_requests_total` has data 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The range window holds fewer than 2 samples. With `scrape_interval: 60s`, a 1m window often has only one sample.

**Diagnose →** `count_over_time(http_requests_total[1m])` returns 1 or less.

**Fix →** Use a window of at least 4× the scrape interval (`[4m]` for 60s scrapes), or `$__rate_interval` in Grafana.

**Prevent →** Standardise on `$__rate_interval` in all dashboard panels.

</details>

### Q53. `rate()` applied to a gauge shows huge spikes 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `rate()` assumes a monotonically increasing counter; every decrease in a gauge is treated as a counter reset, producing big fake increases.

**Diagnose →** Check `# TYPE` on the metric's `/metrics` output.

**Fix →** For gauges use `deriv()`, `delta()`, `avg_over_time()` or the raw value. Use `rate()`/`increase()` only on counters (`_total`, `_count`, `_sum`, `_bucket`).

**Prevent →** Follow naming conventions so types are obvious.

</details>

### Q54. Binary operation fails: `many-to-many matching not allowed: matching labels must be unique on one side` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both sides have multiple series per match group, often when joining with `kube_pod_info` or `kube_pod_labels` which can have duplicates during pod restarts or from two kube-state-metrics replicas.

**Diagnose →**
```promql
count by (namespace, pod) (kube_pod_info) > 1
```

**Fix →** Aggregate the "one" side first and use explicit matching:
```promql
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
  * on (namespace, pod) group_left(node)
  max by (namespace, pod, node) (kube_pod_info)
```

**Prevent →** Run one kube-state-metrics per cluster (or shard properly) and always wrap join sides in `max by`.

</details>

### Q55. `found duplicate series for the match group {pod="api-1"} on the right hand-side of the operation` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `group_left` expects one series per match group on the right; there are two (e.g. different `instance` or `job` labels from two scrape paths).

**Diagnose →** Run the right-hand side alone and look for duplicates on the `on(...)` labels.

**Fix →** Reduce the right side with `max by (pod, <labels you need>)`, or add the differing label to `on(...)`.

**Prevent →** Avoid scraping the same exporter through two jobs.

</details>

### Q56. Error ratio query returns no data during quiet hours 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** When there are no errors, the `status=~"5.."` series do not exist yet, so the numerator is empty and the division returns nothing.

**Diagnose →** Run numerator and denominator separately.

**Fix →**
```promql
(sum(rate(http_requests_total{status=~"5.."}[5m])) or vector(0))
/
sum(rate(http_requests_total[5m]))
```
Or initialise counters for known status classes in code.

**Prevent →** Pre-initialise label combinations you alert on.

</details>

### Q57. `histogram_quantile` returns `NaN` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No requests in the window (all bucket rates are 0), so the quantile is undefined.

**Diagnose →** `sum(rate(http_request_duration_seconds_count[5m]))` is 0.

**Fix →** Accept it as "no traffic", or filter: `... and sum(rate(..._count[5m])) > 0`. In Grafana, set the panel to show NaN as null/gap.

**Prevent →** Alerts on latency should be combined with a minimum traffic condition.

</details>

### Q58. p99 latency is reported as exactly `10` seconds for hours 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Requests exceed the highest finite bucket; `histogram_quantile` returns the upper bound of the last finite bucket when the quantile lands in `+Inf`.

**Diagnose →**
```promql
sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
```
Look at how many observations sit above the top `le`.

**Fix →** Add higher buckets (e.g. 15, 30, 60) or move to native histograms.

**Prevent →** Design buckets around SLO thresholds plus headroom above the timeout.

</details>

### Q59. `histogram_quantile` result jumps wildly between refreshes 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `le` was dropped or aggregated incorrectly (`sum without(le)`), buckets are mixed across services with different boundaries, or the window is too short for low traffic.

**Diagnose →** Check the inner expression keeps `le`: `sum by (le, service) (rate(..._bucket[5m]))`.

**Fix →** Always aggregate with `by (le, ...)`, do not mix histograms with different bucket layouts, widen the window.

**Prevent →** Wrap quantile queries in recording rules such as `service:http_request_duration_seconds:p99_5m`.

</details>

### Q60. Recording rule never produces data; `/rules` shows health `err` with `vector contains metrics with the same labelset after applying rule labels` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The expression returns several series that become identical once `__name__` is replaced and rule `labels:` are applied (e.g. `rate(x[5m])` across two jobs then a `job` label is overwritten).

**Diagnose →** Run the expression in the UI; check which labels differ between series.

**Fix →** Aggregate explicitly: `sum by (service) (...)` so output labels are unique, or stop overwriting a distinguishing label.

**Prevent →** Lint rules with `promtool check rules` and unit-test them with `promtool test rules`.

</details>

### Q61. `promtool check rules` fails with `group "api": duplicate rule` or YAML errors 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Duplicate record/alert names within a group, tabs in YAML, or a multi-line `expr` without `|`.

**Diagnose →**
```bash
promtool check rules rules/*.yml
yamllint rules/
```

**Fix →** Rename duplicates, use spaces, and write long expressions as `expr: |` blocks.

**Prevent →** Run `promtool check rules` and `promtool test rules` in CI for every rules PR.

</details>

### Q62. Rule group evaluation is slower than its interval; `prometheus_rule_group_iterations_missed_total` is increasing 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rules within a group run sequentially; a few heavy queries exceed the 1m interval, so iterations are skipped.

**Diagnose →**
```promql
prometheus_rule_group_last_duration_seconds > prometheus_rule_group_interval_seconds
topk(10, prometheus_rule_evaluation_duration_seconds)
```

**Fix →** Split heavy rules into separate groups (groups evaluate concurrently), layer recording rules (5m base → 1h derived), increase group `interval` for slow-moving rules.

**Prevent →** Alert on missed iterations.

</details>

### Q63. `increase(errors_total[1h])` shows `2.97` even though errors are whole numbers 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Expected. `increase()` extrapolates to the window edges, so results are fractional.

**Diagnose →** Compare with `errors_total - errors_total offset 1h` for a rough raw difference.

**Fix →** Round for display (`round(increase(...[1h]))`) and do not treat it as an exact count.

**Prevent →** For exact counts use logs or a billing-grade system, not Prometheus.

</details>

### Q64. Query fails with `parse error: unexpected character inside braces: '-'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A label name contains characters PromQL does not allow unquoted, or quotes are missing around a value.

**Diagnose →** Check the selector: `{app-name=checkout}` is invalid.

**Fix →** Label values must be quoted: `{app="checkout"}`. Label names must match `[a-zA-Z_][a-zA-Z0-9_]*`; fix the label at relabel time (`labelmap` converts unsupported characters to `_` for Kubernetes labels).

**Prevent →** Use Grafana's query builder or autocomplete.

</details>

### Q65. `absent(up{job="payments"})` alert never fires even though the job is gone 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another series still matches (e.g. a second Prometheus federates it, or a stale `instance` still returns `up=0` which is not "absent").

**Diagnose →** Run `up{job="payments"}` and check what is returned.

**Fix →** Alert on both conditions:
```promql
absent(up{job="payments"} == 1)
```
This fires when no healthy target exists.

**Prevent →** Pair `up == 0` alerts with absent-style alerts for critical jobs.

</details>

### Q66. A Grafana panel and the Prometheus UI show different values for the same query 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different step: Grafana computes step from the panel width and `Min interval`, which changes how `rate()` windows align; or Grafana is querying a different data source / Thanos with downsampling.

**Diagnose →** In Grafana Query inspector check the actual `step`, `start`, `end`, and the data source URL.

**Fix →** Use `$__rate_interval`, set `Min interval` to the scrape interval, and confirm the data source.

**Prevent →** Document data sources clearly in dashboard descriptions.

</details>

### Q67. `topk(5, ...)` shows more than 5 series on a graph 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `topk` is evaluated per step in a range query, so different series win at different times.

**Diagnose →** Run as an instant query; you get exactly 5.

**Fix →** For a stable set, pick the top 5 over the whole range with a variable, or use `topk(5, avg_over_time(...[$__range]))` as an instant query in a table panel.

**Prevent →** Explain this in the panel description; it surprises everyone once.

</details>

### Q68. `sum(rate(container_cpu_usage_seconds_total[5m]))` is double what `kubectl top` shows 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cAdvisor exposes pod-level cgroup series (`container=""`) and the pause container (`container="POD"`) alongside each container.

**Diagnose →**
```promql
count by (container) (container_cpu_usage_seconds_total{namespace="shop"})
```

**Fix →**
```promql
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!="", container!="POD", image!=""}[5m]))
```

**Prevent →** Use the kube-prometheus recording rules (`node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m`).

</details>

### Q69. Memory dashboard uses `container_memory_usage_bytes` and shows pods near limit, yet no OOMKills happen 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `usage_bytes` includes page cache that the kernel can reclaim. OOM decisions are based on working set.

**Diagnose →** Compare `container_memory_working_set_bytes` and `container_memory_rss` for the same pod.

**Fix →** Use `container_memory_working_set_bytes` for limit-based alerts and dashboards.

**Prevent →** Standardise memory panels across teams.

</details>

### Q70. `predict_linear` disk-full alert fires constantly on log volumes 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Logrotate makes usage sawtooth; a short window extrapolates each growth spike as if it continues.

**Diagnose →** Plot `node_filesystem_avail_bytes` for the mount over 24h.

**Fix →** Use a longer window and a minimum free threshold together:
```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/var/log"}[6h], 4*3600) < 0
and node_filesystem_avail_bytes{mountpoint="/var/log"} / node_filesystem_size_bytes < 0.2
```

**Prevent →** Add `for: 30m`.

</details>

### Q71. Query with `offset 1w` returns empty while the current value exists 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Retention is shorter than one week, or the series did not exist then (label change, renamed metric).

**Diagnose →** Check `--storage.tsdb.retention.time` on `/flags` and `prometheus_tsdb_lowest_timestamp`.

**Fix →** Query long-term storage (Thanos/Mimir), or extend retention.

**Prevent →** Keep week-over-week comparisons on a long-term store.

</details>

### Q72. Subquery `max_over_time(rate(x[5m])[1d:])` is extremely slow 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without an explicit resolution, the subquery uses the global evaluation interval, computing `rate` at thousands of points per series.

**Diagnose →** Check query time in the Prometheus UI stats and the series count.

**Fix →** Set resolution `[1d:5m]`, or better, create a recording rule for `rate(x[5m])` and run `max_over_time` over that.

**Prevent →** Ban raw subqueries in alerts; use recording rules.

</details>

### Q73. Stale series keep showing on a graph for 5 minutes after a pod is deleted 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lookback delta (default 5m) or the target was removed without a stale marker (e.g. Prometheus restarted at the same time).

**Diagnose →** Check if the line flattens then ends ~5m later.

**Fix →** Usually normal. Prometheus writes stale markers when a target disappears from SD, so series end quickly; if not, reduce `--query.lookback-delta` carefully.

**Prevent →** Understand lookback before tuning it; shorter values break sparse metrics.

</details>

### Q74. `label_replace` does not change anything 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The regex does not match the full value (it is anchored), or source and destination arguments are swapped.

**Diagnose →** Signature: `label_replace(v, dst_label, replacement, src_label, regex)`.

**Fix →**
```promql
label_replace(up, "host", "$1", "instance", "([^:]+):.*")
```

**Prevent →** Test regexes on real values before saving recording rules.

</details>

### Q75. `promtool test rules` fails with `expected: ... got: nil` for an alert test 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `eval_time` is earlier than the alert's `for` duration allows, input series intervals are wrong, or labels in `exp_labels` miss rule-added labels like `severity`.

**Diagnose →** Run the alert expression in the test file as a `promql_expr_test` to see raw output at `eval_time`.

**Fix →** Set `eval_time` past `for` (e.g. `for: 5m` → `eval_time: 10m`), use series notation like `'0+10x20'`, and list every label including rule labels.

**Prevent →** Keep a test file per rule file and run it in CI.

</details>

---

## Alerting and Alertmanager

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. An alert shows `Pending` in Prometheus but never goes to `Firing` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The condition does not stay true for the full `for:` duration; a single evaluation dip resets the timer.

**Diagnose →** Graph the alert expression for the last hour and check for gaps or dips.

**Fix →** Smooth the expression (`avg_over_time`, longer `rate` window), or shorten `for`. Prometheus 2.42+ also supports `keep_firing_for` to stop flapping on resolve.

**Prevent →** Use windows long enough that one missed scrape does not reset the alert.

</details>

### Q77. An alert is `Firing` in Prometheus but nothing reaches Alertmanager 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `alerting.alertmanagers` configured, wrong address, or Operator `alerting` section points at the wrong Service/port.

**Diagnose →**
```promql
prometheus_notifications_alertmanagers_discovered
rate(prometheus_notifications_errors_total[5m])
rate(prometheus_notifications_dropped_total[5m])
```
Check **Status → Runtime & Build Info** for the Alertmanager list.

**Fix →** Point `alerting.alertmanagers` at the Alertmanager service (port `9093`, or the named port `web` in the Operator).

**Prevent →** Alert on `prometheus_notifications_alertmanagers_discovered < 1`, and run a Watchdog alert to a dead man's switch.

</details>

### Q78. Alertmanager shows the alert but Slack receives nothing 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The route matches a different receiver, the webhook URL is wrong or revoked, or the channel is not the one the webhook is bound to.

**Diagnose →**
```bash
amtool config routes test --config.file=alertmanager.yml severity=critical team=payments
kubectl logs alertmanager-main-0 -c alertmanager | grep -i 'notify\|slack'
```
```promql
rate(alertmanager_notifications_failed_total{integration="slack"}[5m])
```

**Fix →** Correct routing matchers and `api_url`/`api_url_file`. Test the webhook directly with `curl -X POST -H 'Content-type: application/json' --data '{"text":"test"}' $WEBHOOK`.

**Prevent →** Alert on `alertmanager_notifications_failed_total` via a second channel (email/PagerDuty).

</details>

### Q79. Alertmanager logs `notify retry canceled due to unrecoverable error after 1 attempts: channel_not_found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Using a bot token (`api_url: https://slack.com/api/chat.postMessage`) with a channel the app is not a member of, or a typo in `channel:`.

**Diagnose →** Check the `channel:` value and whether the Slack app is invited.

**Fix →** `/invite @alertmanager-bot` in the channel, or use an incoming webhook bound to that channel.

**Prevent →** Keep receivers in version control and test them after Slack workspace changes.

</details>

### Q80. Alertmanager logs `unexpected status code 404: no_service` / `invalid_token` for Slack 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The incoming webhook was deleted or rotated, or the Secret has a trailing newline.

**Diagnose →**
```bash
kubectl get secret alertmanager-slack -n monitoring -o jsonpath='{.data.url}' | base64 -d | od -c | tail -2
```

**Fix →** Create a new webhook, store it without newline (`printf '%s'`), reference it via `api_url_file`.

**Prevent →** Track webhook ownership; a departing admin can revoke it.

</details>

### Q81. Alertmanager fails to load config: `yaml: unmarshal errors: field matchers not found in type config.plain` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alertmanager version is older than the config syntax (`matchers:` arrived in 0.22), or indentation puts it at the wrong level.

**Diagnose →**
```bash
amtool check-config alertmanager.yml
alertmanager --version
```

**Fix →** Upgrade Alertmanager or use the old `match`/`match_re` form; fix indentation.

**Prevent →** Run `amtool check-config` in CI with the same version as production.

</details>

### Q82. Alerts fire constantly: the same alert re-notifies every 5 minutes 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `repeat_interval` is too low, or the alert flaps (resolves and re-fires), creating new notifications.

**Diagnose →** Check alert history in Prometheus (`ALERTS{alertname="X"}`) and Alertmanager route timings.

**Fix →** Set sensible timings:
```yaml
route:
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```
Add `keep_firing_for` on flapping rules.

**Prevent →** Review the top noisy alerts weekly.

</details>

### Q83. One incident sends 200 separate Slack messages 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `group_by` includes high-variety labels (`instance`, `pod`) or is missing, so each alert is its own group.

**Diagnose →** `amtool alert query` and look at the groups in the Alertmanager UI.

**Fix →** `group_by: [alertname, cluster, namespace]` and add inhibition rules (e.g. `NodeDown` inhibits pod alerts on that node).

**Prevent →** Design grouping per team route, not at the root only.

</details>

### Q84. Duplicate notifications arrive from an HA Alertmanager cluster 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Peers are not clustered (gossip port `9094` blocked or wrong `--cluster.peer`), so each instance notifies independently. Or Prometheus sends to a load-balanced VIP instead of all instances.

**Diagnose →**
```promql
alertmanager_cluster_members
alertmanager_cluster_failed_peers
```
UI **Status** page shows cluster peers.

**Fix →** Open TCP+UDP 9094 between peers, set `--cluster.peer` for each, and configure Prometheus to send to every Alertmanager individually.

**Prevent →** Alert when `alertmanager_cluster_members` is below the expected count.

</details>

### Q85. A silence was created but the alert still notifies 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Matchers do not match exactly (label value differs, regex not anchored as you think), or the silence was created on one Alertmanager that is not clustered.

**Diagnose →**
```bash
amtool silence query
amtool alert query alertname=HighLatency
```
Compare labels with the silence matchers.

**Fix →** Recreate with exact labels: `amtool silence add alertname=HighLatency service=checkout --duration=2h --comment="deploy"`. Fix clustering if needed.

**Prevent →** Create silences from the alert's "Silence" button so labels are pre-filled.

</details>

### Q86. Inhibition rule does not suppress target alerts 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `equal:` labels are missing on one side (e.g. `cluster` exists on the source but not the target), or the source alert is not firing in the same Alertmanager.

**Diagnose →** Compare label sets of the source and target alerts in the UI.

**Fix →**
```yaml
inhibit_rules:
  - source_matchers: [alertname="NodeDown"]
    target_matchers: [severity=~"warning|critical"]
    equal: [cluster, node]
```
Ensure both alerts carry `cluster` and `node`.

**Prevent →** Add consistent external labels and test inhibition with synthetic alerts via `amtool alert add`.

</details>

### Q87. Alerts arrive with an empty summary like `[FIRING:1]  ()` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The template references annotations/labels that the rule does not set (`.CommonAnnotations.summary`), or a typo in the template.

**Diagnose →** Check the rule's `annotations:` and the receiver template.

**Fix →** Add `summary` and `description` annotations to every rule; test templates with `amtool template render --template.glob=templates/*.tmpl --template.text='{{ template "slack.title" . }}'`.

**Prevent →** Enforce required annotations (`summary`, `runbook_url`) with a rule linter in CI.

</details>

### Q88. Alertmanager logs `template: ... executing "slack.default.text" ... map has no entry for key` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Custom template indexes a label that some alerts lack.

**Diagnose →** Find the alert missing the label in the UI.

**Fix →** Guard the template: `{{ if .Labels.service }}{{ .Labels.service }}{{ else }}unknown{{ end }}`, or use `{{ index .Labels "service" }}` which returns empty instead of failing.

**Prevent →** Render templates against sample alerts in CI.

</details>

### Q89. PagerDuty never receives alerts; Alertmanager logs `unexpected status code 400 ... Event object is invalid` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Using `service_key` (Events API v1) with an Events API v2 integration key, or the payload exceeds limits from huge annotations.

**Diagnose →** Check the integration type in PagerDuty and the receiver config.

**Fix →** For v2 integrations use `routing_key:` (or `routing_key_file`). Trim large annotation fields.

**Prevent →** Test receivers with `amtool alert add` in staging.

</details>

### Q90. Email receiver fails with `*smtp.plainAuth auth: unencrypted connection` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SMTP PLAIN auth over a non-TLS connection is refused by Go's client.

**Diagnose →** Check `smarthost` port and `require_tls`.

**Fix →** Use port 587 with STARTTLS (`require_tls: true`, default) or 465 with TLS. For an internal relay without auth, remove `auth_username`/`auth_password`.

**Prevent →** Prefer API-based receivers (SES via webhook, Slack, PagerDuty) over raw SMTP.

</details>

### Q91. Alert resolves in Slack and immediately fires again 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus misses an evaluation or a scrape (target restarting), the expression briefly becomes false, Alertmanager sends resolved, then the alert fires again.

**Diagnose →** Graph `ALERTS{alertname="X"}` and check scrape gaps around the flip.

**Fix →** Add `keep_firing_for: 10m` to the rule, widen the `rate` window, set `send_resolved: false` for noisy, low-value receivers.

**Prevent →** Track flap counts per alert.

</details>

### Q92. Alerts stop being sent after Alertmanager restarts and then all old ones re-fire 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alertmanager stores silences and notification log on disk (`--storage.path`); with `emptyDir` and a single replica, state is lost on restart.

**Diagnose →** Check pod volumes and whether silences disappeared.

**Fix →** Run 3 clustered replicas (state is gossiped) and/or use a PVC for storage.

**Prevent →** Treat Alertmanager as stateful.

</details>

### Q93. `Watchdog` alert stopped arriving at the dead man's switch 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The whole pipeline is broken somewhere: Prometheus down, rule not loaded, Alertmanager can't send, or the Watchdog route was swallowed by a `null` receiver.

**Diagnose →**
```bash
amtool config routes test --config.file=alertmanager.yml alertname=Watchdog
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].rules[] | select(.name=="Watchdog")'
```

**Fix →** Fix whichever stage failed; route Watchdog first with `continue: false` to the heartbeat receiver (healthchecks.io, PagerDuty heartbeat, Opsgenie heartbeat).

**Prevent →** Exactly this alert is the prevention; make sure its receiver pages when heartbeats stop.

</details>

### Q94. Alerts route to the default receiver even though a team route exists 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Route order: an earlier sibling route matches first and has `continue: false` (default), or the team label is missing on the alert.

**Diagnose →**
```bash
amtool config routes show --config.file=alertmanager.yml
amtool config routes test --config.file=alertmanager.yml --verify.receivers=payments-slack team=payments severity=critical
```

**Fix →** Reorder routes (specific first), add missing `team` labels to rules, use `continue: true` where an alert should reach multiple receivers.

**Prevent →** Keep route tests in CI.

</details>

### Q95. `AlertmanagerConfig` CR in a namespace has no effect 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Alertmanager CR's `alertmanagerConfigSelector` / `alertmanagerConfigNamespaceSelector` does not match the CR, or the Operator rejected it as invalid.

**Diagnose →**
```bash
kubectl get alertmanager -n monitoring -o yaml | grep -A5 alertmanagerConfig
kubectl logs -n monitoring deploy/prometheus-operator | grep -i alertmanagerconfig
kubectl get secret alertmanager-main-generated -n monitoring -o jsonpath='{.data.alertmanager\.yaml\.gz}' | base64 -d | gunzip | less
```

**Fix →** Label the CR to match the selector. Remember the Operator adds a `namespace` matcher to every route in that CR, so alerts must carry the matching `namespace` label.

**Prevent →** Document the required labels for team-owned AlertmanagerConfigs.

</details>

### Q96. Alert rule never loads; `/rules` does not list it 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is not in `rule_files` glob, Prometheus was not reloaded, or (Operator) the PrometheusRule lacks the label the `ruleSelector` expects.

**Diagnose →**
```bash
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].name'
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.ruleSelector}'
```

**Fix →** Add the file/glob, reload, or add the label (often `release: kube-prometheus-stack`).

**Prevent →** Lint that every PrometheusRule carries the selector label.

</details>

### Q97. Grafana-managed alert shows `Error` state with `[sse.dataQueryError] failed to execute query A: ... context deadline exceeded` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The alert query is too heavy or the data source is slow; Grafana's evaluation timeout is hit.

**Diagnose →** Run the query in Explore with the same time range and check duration.

**Fix →** Simplify or back it with a recording rule; set "Error state handling" to `Keep last state` or `Error` deliberately rather than `Alerting`.

**Prevent →** Prefer Prometheus-evaluated rules for critical alerts; Grafana alerting depends on Grafana being up.

</details>

### Q98. Grafana alerts fire as `DatasourceNoData` all night 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query returns no series during low traffic or after a metric rename; "No data" state is configured to alert.

**Diagnose →** Run the query for the night time range in Explore.

**Fix →** Make the query return `0` with `or vector(0)`, or set "Alert state if no data" to `Normal`/`OK`, and add a separate absent-style alert for real missing data.

**Prevent →** Decide no-data behaviour per rule, not by default.

</details>

### Q99. Alert fires for a cluster that was decommissioned months ago 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An `absent()`/`up == 0` rule still references the old cluster, or the old Prometheus still sends heartbeats from a forgotten VM.

**Diagnose →** Check `generatorURL` in the alert to find the sending Prometheus.

**Fix →** Remove the rule or shut down the stray Prometheus.

**Prevent →** Include cluster decommission in the runbook: remove monitoring, rules, routes.

</details>

### Q100. Alert storm: Alertmanager memory climbs and the UI becomes unresponsive 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A rule produces tens of thousands of alert instances (e.g. per-pod or per-path labels), each a separate alert.

**Diagnose →**
```promql
alertmanager_alerts
count by (alertname) (ALERTS{alertstate="firing"})
```

**Fix →** Aggregate the rule (`sum by (service)`), silence the offender, and restart if needed.

**Prevent →** Set a rule `limit:` on the group (Prometheus 2.31+) so a rule producing too many alerts fails instead of flooding.

</details>

---

## Grafana dashboards and data sources

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Every panel on a dashboard shows `No data` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The dashboard's data source variable points to the wrong (or a deleted) data source, the time range is wrong, or a template variable resolved to nothing.

**Diagnose →** Open one panel → **Query inspector** → see the actual request and response; check the `datasource` and `$variables` at the top.

**Fix →** Select the correct data source, fix variable queries (e.g. `label_values(up, job)`), or set the time range to cover data.

**Prevent →** Use a `${datasource}` variable in shared dashboards instead of hardcoded UIDs.

</details>

### Q102. Grafana data source test fails with `Bad Gateway` (HTTP 502) 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Grafana's backend cannot reach the URL: wrong host/port, DNS failure from the Grafana pod, NetworkPolicy, or the backend is down.

**Diagnose →**
```bash
kubectl exec -n monitoring deploy/grafana -- wget -qO- -T 5 http://prometheus-operated:9090/-/ready
kubectl logs -n monitoring deploy/grafana | grep -i 'proxy\|datasource'
```

**Fix →** Use the in-cluster DNS name (`http://prometheus-operated.monitoring.svc:9090`), set access mode to **Server** (not Browser), and open NetworkPolicy egress.

**Prevent →** Provision data sources as code and check them after cluster changes.

</details>

### Q103. Data source test returns `Post "http://prometheus:9090/api/v1/query": dial tcp: lookup prometheus on 10.96.0.10:53: no such host` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The short Service name only resolves in the same namespace; Grafana runs in another namespace.

**Diagnose →** `kubectl exec deploy/grafana -- nslookup prometheus.monitoring.svc.cluster.local`.

**Fix →** Use the FQDN `http://prometheus.monitoring.svc.cluster.local:9090`.

**Prevent →** Always use FQDNs in provisioned data sources.

</details>

### Q104. Grafana shows `504 Gateway Timeout` on long-range queries 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query outlasts the data source timeout, the Grafana `[dataproxy] timeout` (default 30s), or an ingress/load balancer idle timeout.

**Diagnose →** Query inspector shows request duration; check ingress annotations and Grafana logs.

**Fix →** Optimise the query (recording rules), raise the data source `Timeout` field and `[dataproxy] timeout`, and the ingress `proxy-read-timeout`.

**Prevent →** Keep heavy long-range views on downsampled data (Thanos/Mimir).

</details>

### Q105. Grafana behind a reverse proxy shows `If you're seeing this Grafana has failed to load its application files` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Served under a sub-path (`/grafana/`) without `root_url` and `serve_from_sub_path`.

**Diagnose →** Browser devtools show 404s for `/public/build/*.js`.

**Fix →**
```ini
[server]
root_url = https://ops.example.com/grafana/
serve_from_sub_path = true
```

**Prevent →** Set these via Helm `grafana.ini` values from day one.

</details>

### Q106. Grafana login loops back to the login page after successful OAuth 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `root_url` does not match the external URL so the redirect URI/cookies mismatch, or cookies are marked secure on HTTP, or multiple replicas without shared sessions/database.

**Diagnose →** Check Grafana logs for `oauth` errors and the browser's cookie for `grafana_session`.

**Fix →** Set `root_url` to the exact public URL; match the IdP redirect URI (`https://grafana.example.com/login/generic_oauth`); use a shared DB (PostgreSQL/MySQL) for multiple replicas.

**Prevent →** Never run multiple Grafana replicas on SQLite.

</details>

### Q107. Grafana pod logs `database is locked` and dashboards fail to save 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SQLite on a network or shared volume, or several replicas pointing at the same SQLite file.

**Diagnose →** Check `[database] type` in config and the replica count.

**Fix →** Move to PostgreSQL or MySQL (`[database] type = postgres`), or scale back to one replica with a local PVC.

**Prevent →** Use an external DB for any HA or production Grafana.

</details>

### Q108. Dashboard variable dropdown is empty 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The variable query returns nothing for the selected time range or references a parent variable that is empty; regex filter excludes everything.

**Diagnose →** Dashboard settings → Variables → **Run query** / preview of values.

**Fix →** Use `label_values(kube_pod_info{namespace="$namespace"}, pod)`, set refresh to "On time range change", and test the regex.

**Prevent →** Order variables so parents load before children.

</details>

### Q109. Panel shows `Data outside time range` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Data exists but not in the selected window, often due to timezone mismatch or a panel-level relative time override.

**Diagnose →** Check panel **Query options** for relative time/time shift and the dashboard timezone.

**Fix →** Remove the override or set timezone to the browser/UTC consistently; use "Zoom to data".

**Prevent →** Pick UTC for operational dashboards.

</details>

### Q110. Grafana panel shows `Too many datapoints` / browser freezes 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query returns thousands of series (no aggregation) or a tiny step over a long range.

**Diagnose →** Query inspector → **Data** shows series count.

**Fix →** Aggregate (`sum by (service)`), use `topk`, set `Max data points` and `Min interval`.

**Prevent →** Review new panels for series count before sharing.

</details>

### Q111. Provisioned dashboards revert after users edit them 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File-provisioned dashboards are overwritten from disk on each `updateIntervalSeconds`; UI changes are not persisted to the file.

**Diagnose →** Dashboard shows "Cannot save provisioned dashboard".

**Fix →** Edit the JSON in Git, or set `allowUiUpdates: true` in the provider (changes still lost on restart unless exported).

**Prevent →** Treat dashboards as code (Grafonnet, Terraform `grafana_dashboard`, or Git sync) with a clear workflow.

</details>

### Q112. Grafana sidecar does not load dashboards from ConfigMaps 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ConfigMap lacks the label the sidecar watches (`grafana_dashboard: "1"` by default), it's in a namespace the sidecar doesn't search, or JSON is invalid.

**Diagnose →**
```bash
kubectl logs -n monitoring deploy/grafana -c grafana-sc-dashboard | tail
kubectl get cm -A -l grafana_dashboard=1
```

**Fix →** Add the label, set `sidecar.dashboards.searchNamespace: ALL` in the Helm values, validate JSON with `jq`.

**Prevent →** CI-validate dashboard JSON and labels.

</details>

### Q113. Imported dashboard panels show `Datasource ${DS_PROMETHEUS} was not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The dashboard JSON came from grafana.com with an `__inputs` placeholder, and it was provisioned from file without resolving it.

**Diagnose →** Search the JSON for `${DS_`.

**Fix →** Replace the placeholder with your data source UID, or convert to a `datasource` template variable.

**Prevent →** Normalise imported dashboards before committing them.

</details>

### Q114. Loki data source returns `Data source connected, but no labels received` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No logs have been ingested in the queried range, the `X-Scope-OrgID` tenant header is wrong/missing in multi-tenant Loki, or the URL points at the wrong component.

**Diagnose →**
```bash
curl -s -H 'X-Scope-OrgID: prod' http://loki-gateway/loki/api/v1/labels
```

**Fix →** Add the `X-Scope-OrgID` custom header in the data source, point to the gateway/query-frontend, and confirm agents are shipping.

**Prevent →** Provision tenants explicitly per data source.

</details>

### Q115. Grafana Tempo data source `Trace not found` when clicking a trace ID from logs 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The derived-field regex captures the wrong string, the trace wasn't sampled, or it is outside the search window.

**Diagnose →** Copy the trace ID and search Tempo directly; check the Loki derived field regex (e.g. `trace_id=(\w+)`).

**Fix →** Fix the regex and target data source UID; widen Tempo's search time shift settings.

**Prevent →** Log trace IDs in a consistent field (`trace_id`) across services.

</details>

### Q116. Grafana CloudWatch data source returns `AccessDenied: ... not authorized to perform: cloudwatch:GetMetricData` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM role used by Grafana (IRSA, instance profile or keys) lacks permissions.

**Diagnose →** `aws sts get-caller-identity` from the Grafana pod; check the role policy.

**Fix →** Grant `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`, `cloudwatch:DescribeAlarms`, `logs:StartQuery`, `logs:GetQueryResults`, `ec2:DescribeRegions`, `tag:GetResources`.

**Prevent →** Use the AWS-documented Grafana policy as a managed policy.

</details>

### Q117. Grafana memory spikes and the pod is OOMKilled when a big dashboard opens 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Server-side expressions or transformations over huge result sets, image rendering, or many concurrent panel queries.

**Diagnose →** `grafana_` process metrics, logs around the time, and Query inspector sizes per panel.

**Fix →** Reduce series per panel, move transformations into PromQL, raise memory limits, run the image renderer as a separate service.

**Prevent →** Split mega-dashboards into overview + drill-down.

</details>

### Q118. Graph shows a flat line at the last value after the service died 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Grafana "Connect null values" set to Always, or the query uses `last_over_time`/`max_over_time` with a long window that hides gaps.

**Diagnose →** Check panel settings and the raw query output.

**Fix →** Set Connect null values to `Never` or a threshold, and use short windows.

**Prevent →** Pair key graphs with `up` panels.

</details>

### Q119. Grafana annotations from Prometheus alerts don't appear 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The annotation query uses the wrong metric or is disabled.

**Diagnose →** Dashboard settings → Annotations; run `ALERTS{alertstate="firing"}` in Explore.

**Fix →** Add an annotation with data source Prometheus and query `ALERTS{alertstate="firing", service="$service"}`, with "Series value as timestamp" off.

**Prevent →** Add deploy annotations too, so changes and alerts line up.

</details>

### Q120. Grafana image rendering in alerts fails: `Rendering failed: ... no image renderer available` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The renderer plugin/service is not installed or the URLs are wrong.

**Diagnose →** Check `[rendering] server_url` and `callback_url` and logs.

**Fix →** Deploy `grafana-image-renderer` as a service; set `server_url = http://renderer:8081/render` and `callback_url = http://grafana:3000/`.

**Prevent →** Include the renderer in the Helm release (`imageRenderer.enabled: true`).

</details>

### Q121. A user sees dashboards but gets `Permission denied` on Explore 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Viewer role cannot use Explore unless `viewers_can_edit` is enabled or the user has the Editor role / a data source query permission.

**Diagnose →** Check user role in the org and data source permissions.

**Fix →** Grant Editor or use RBAC data source query permission (Enterprise/Cloud), or team-based roles.

**Prevent →** Map IdP groups to roles in `[auth.generic_oauth] role_attribute_path`.

</details>

### Q122. Panels using `$__rate_interval` show nothing at short time ranges 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The data source `Scrape interval` setting is lower than the real scrape interval, so `$__rate_interval` becomes too small to hold 2 samples.

**Diagnose →** Query inspector shows the expanded interval (e.g. `[1m]` for a 60s scrape).

**Fix →** Set the Prometheus data source **Scrape interval** to the real value (e.g. `60s`), so `$__rate_interval` = max(4 × scrape, interval + scrape).

**Prevent →** Provision data sources with the correct `timeInterval`.

</details>

### Q123. After upgrading Grafana, old Angular panels show `Panel plugin not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AngularJS support was deprecated and removed in recent Grafana versions; old plugins (e.g. legacy graph, singlestat, worldmap) no longer load.

**Diagnose →** Dashboard shows a deprecation warning badge on panels before upgrade.

**Fix →** Migrate to core React panels (Time series, Stat, Geomap); Grafana auto-migrates many core ones when you open and save.

**Prevent →** Scan for Angular panels before major upgrades.

</details>

### Q124. Elasticsearch data source in Grafana returns `index_not_found_exception` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Index pattern does not match the real index names (`logs-*` vs `filebeat-*`) or uses daily pattern syntax wrongly.

**Diagnose →** `curl -s es:9200/_cat/indices?v | head`.

**Fix →** Set the index name to the right wildcard (or data stream), and the correct time field (`@timestamp`).

**Prevent →** Standardise index naming via index templates.

</details>

### Q125. Grafana shows timestamps 5h30m off from the logs 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Logs are parsed without timezone (treated as UTC) while the source writes local time (IST), or the dashboard timezone setting differs.

**Diagnose →** Compare raw log line time with `@timestamp`/Loki timestamp.

**Fix →** Log in UTC with an explicit offset (ISO 8601), or set the parser's timezone (Fluent Bit `Time_Offset`, Logstash `timezone`).

**Prevent →** Enforce UTC on servers and in app log formats.

</details>

---

## Kubernetes monitoring and the Prometheus Operator

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. A new ServiceMonitor is not picked up by the Prometheus Operator 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ServiceMonitor lacks the label matching `serviceMonitorSelector` (kube-prometheus-stack: `release: <helm-release>`), its namespace is not in `serviceMonitorNamespaceSelector`, or the CRD version differs.

**Diagnose →**
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}{"\n"}{.items[0].spec.serviceMonitorNamespaceSelector}'
kubectl get servicemonitor -n shop checkout -o yaml | grep -A3 labels
```

**Fix →** Add the label (e.g. `release: kube-prometheus-stack`), or set the Helm values `serviceMonitorSelectorNilUsesHelmValues: false` so all ServiceMonitors are selected.

**Prevent →** Provide a ServiceMonitor template with the right labels to app teams.

</details>

### Q127. ServiceMonitor is selected but its targets don't appear on `/targets` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `spec.selector` does not match the Service's labels (not the pod labels), `endpoints[].port` refers to a port name that the Service doesn't define, or `namespaceSelector` is missing for a cross-namespace Service.

**Diagnose →**
```bash
kubectl get svc -n shop checkout --show-labels
kubectl get svc -n shop checkout -o jsonpath='{.spec.ports[*].name}'
kubectl get endpoints -n shop checkout
```

**Fix →** Match Service labels, use the **named** port (`port: http-metrics`) or `targetPort`, and add `namespaceSelector: { matchNames: [shop] }` if needed.

**Prevent →** Name metrics ports consistently (`metrics`).

</details>

### Q128. ServiceMonitor target appears but Operator logs `skipping servicemonitor ... failed to get secret` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ServiceMonitor references a basic auth / bearer / TLS Secret that is in a different namespace or missing; the Operator only reads Secrets in the ServiceMonitor's namespace.

**Diagnose →**
```bash
kubectl logs -n monitoring deploy/prometheus-operator | grep -i 'servicemonitor\|secret'
kubectl get secret -n shop
```

**Fix →** Create the Secret in the same namespace as the ServiceMonitor and reference it via `basicAuth`/`authorization.credentials`.

**Prevent →** Keep secrets beside the ServiceMonitor in the app's Helm chart.

</details>

### Q129. PodMonitor targets show `DOWN` with `connection refused` on port 8080 while the app listens on 9090 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `podMetricsEndpoints[].port` names a container port that isn't the metrics port, or the container doesn't declare `ports` at all.

**Diagnose →** `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].ports}'`.

**Fix →** Declare `- name: metrics, containerPort: 9090` in the pod spec and use `port: metrics`.

**Prevent →** Lint deployments for a named metrics port.

</details>

### Q130. PrometheusRule created but rules don't show in Prometheus 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ruleSelector`/`ruleNamespaceSelector` mismatch, or the admission webhook rejected invalid PromQL.

**Diagnose →**
```bash
kubectl get prometheusrule -A --show-labels | grep myrules
kubectl get cm -n monitoring | grep rulefiles
kubectl logs -n monitoring deploy/prometheus-operator | grep -i rule
```

**Fix →** Add the selector label, fix the expression (`promtool check rules` on the extracted spec).

**Prevent →** Keep the Operator's admission webhook enabled so bad rules fail at `kubectl apply`.

</details>

### Q131. `KubeletDown` / `TargetDown` alerts fire for kube-scheduler and kube-controller-manager on EKS/GKE 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Managed control planes don't expose those components for scraping; kube-prometheus-stack defaults try anyway.

**Diagnose →** `/targets` shows `kube-scheduler` and `kube-controller-manager` jobs with no endpoints or DOWN.

**Fix →** Disable in Helm values: `kubeScheduler.enabled: false`, `kubeControllerManager.enabled: false`, `kubeEtcd.enabled: false`, `kubeProxy.enabled` as appropriate.

**Prevent →** Keep per-platform values files.

</details>

### Q132. On kubeadm clusters, `kube-controller-manager` and `kube-scheduler` targets are `DOWN` with `connection refused` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** They bind to `127.0.0.1` by default (`--bind-address=127.0.0.1`), so Prometheus can't reach them on the node IP.

**Diagnose →** `grep bind-address /etc/kubernetes/manifests/kube-scheduler.yaml`.

**Fix →** Set `--bind-address=0.0.0.0` in the static pod manifests (ports 10259 and 10257, HTTPS), restricting access with firewall rules. Same for etcd metrics with `--listen-metrics-urls=http://0.0.0.0:2381`.

**Prevent →** Bake these flags into the cluster bootstrap.

</details>

### Q133. `kube-proxy` target shows `connection refused` on port 10249 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** kube-proxy's `metricsBindAddress` defaults to `127.0.0.1:10249`.

**Diagnose →** `kubectl -n kube-system get cm kube-proxy -o yaml | grep metricsBindAddress`.

**Fix →** Set `metricsBindAddress: 0.0.0.0:10249` and restart the kube-proxy DaemonSet.

**Prevent →** Include it in cluster config.

</details>

### Q134. kube-state-metrics series are missing for a new CRD-based resource 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** kube-state-metrics only exposes built-in resources by default; custom resources need `--custom-resource-state-config`.

**Diagnose →** `curl ksm:8080/metrics | grep -i <crd>` returns nothing.

**Fix →** Provide a CustomResourceStateMetrics config describing which fields to expose, and grant RBAC to list/watch the CRD.

**Prevent →** Prefer the operator's own metrics endpoint for its CRDs where available.

</details>

### Q135. kube-state-metrics pod OOMKilled in a large cluster 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Thousands of objects × all resources × all labels (`--metric-labels-allowlist=pods=[*]`) in one replica.

**Diagnose →** `kubectl top pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics`; check series count by `job="kube-state-metrics"`.

**Fix →** Restrict `--resources`, narrow `--metric-labels-allowlist`, and shard horizontally (`--shard`/`--total-shards`, or StatefulSet auto-sharding).

**Prevent →** Track KSM series per resource type.

</details>

### Q136. Pods in a namespace have no cAdvisor metrics but node metrics are fine 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `metric_relabel_configs` drop on the cAdvisor job, or the container runtime changed and labels (`container`, `pod`) are empty.

**Diagnose →**
```promql
count by (namespace) (container_cpu_usage_seconds_total)
```
Curl `kubectl get --raw /api/v1/nodes/<node>/proxy/metrics/cadvisor | grep <ns>`.

**Fix →** Fix relabel rules; upgrade kubelet/containerd if labels are empty.

**Prevent →** Alert when a namespace with pods has zero container metrics.

</details>

### Q137. `kubectl top pods` returns `error: Metrics API not available` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** metrics-server is not installed or its APIService is unavailable (often TLS to kubelets).

**Diagnose →**
```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl logs -n kube-system deploy/metrics-server
```
Common log: `x509: cannot validate certificate ... doesn't contain any IP SANs`.

**Fix →** Install metrics-server; fix kubelet serving certs (enable `serverTLSBootstrap`), or as a lab-only workaround add `--kubelet-insecure-tls`.

**Prevent →** Remember HPA depends on it; alert on the APIService availability.

</details>

### Q138. HPA shows `unknown` targets: `failed to get cpu utilization: missing request for cpu` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CPU utilisation is computed as a percentage of requests; a container without `resources.requests.cpu` makes it impossible.

**Diagnose →** `kubectl describe hpa checkout` shows the event.

**Fix →** Set CPU requests on every container in the pod (including sidecars).

**Prevent →** Enforce requests with LimitRange or a policy engine (Kyverno/Gatekeeper).

</details>

### Q139. HPA on a custom metric reports `unable to get metric http_requests_per_second: no metrics returned from custom metrics API` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** prometheus-adapter rules don't expose the metric, the query returns no series for that namespace/pod, or the APIService `v1beta1.custom.metrics.k8s.io` is down.

**Diagnose →**
```bash
kubectl get apiservice v1beta1.custom.metrics.k8s.io
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/shop/pods/*/http_requests_per_second" | jq
```

**Fix →** Add an adapter rule with `seriesQuery`, `resources.overrides` mapping `namespace`/`pod` labels, and `metricsQuery`; or use KEDA's Prometheus scaler.

**Prevent →** Test custom metric queries before wiring them to HPAs.

</details>

### Q140. Prometheus Operator logs `sync "monitoring/k8s" failed: ... the server could not find the requested resource` after upgrade 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CRDs are older than the Operator version; Helm does not upgrade CRDs automatically.

**Diagnose →** `kubectl get crd prometheuses.monitoring.coreos.com -o jsonpath='{.metadata.annotations}'` and compare with the Operator release.

**Fix →** Apply the matching CRDs (server-side to avoid size limits):
```bash
kubectl apply --server-side -f https://github.com/prometheus-operator/prometheus-operator/releases/download/<version>/stripped-down-crds.yaml
```

**Prevent →** Make CRD upgrades an explicit step in your upgrade runbook.

</details>

### Q141. `kubectl apply` of the Prometheus CRDs fails with `metadata.annotations: Too long: must have at most 262144 bytes` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client-side apply stores the full object in the `last-applied-configuration` annotation; the CRDs are too big.

**Diagnose →** The error text is clear.

**Fix →** Use `kubectl apply --server-side --force-conflicts -f crds/` or `kubectl replace`/`create`.

**Prevent →** Use server-side apply for large CRDs in all pipelines.

</details>

### Q142. Prometheus StatefulSet pod stuck `Pending` with `0/6 nodes are available: 6 node(s) had volume node affinity conflict` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The EBS PV lives in one AZ, and there is no schedulable node in that AZ (node group scaled in or moved).

**Diagnose →**
```bash
kubectl get pv $(kubectl get pvc -n monitoring prometheus-k8s-db-prometheus-k8s-0 -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.nodeAffinity}'
kubectl get nodes -L topology.kubernetes.io/zone
```

**Fix →** Add capacity in that AZ, or restore the data into a new volume in another AZ.

**Prevent →** Use `volumeBindingMode: WaitForFirstConsumer` and keep node groups spread across all AZs.

</details>

### Q143. `KubePodCrashLooping` alert fires but the pod is in `Completed` state 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A Job/CronJob pod with `restartPolicy: OnFailure` retried and the alert expression counts container restarts without excluding Jobs.

**Diagnose →** `kube_pod_owner{owner_kind="Job"}` for the pod.

**Fix →** Exclude Jobs or alert on `kube_job_failed` instead.

**Prevent →** Tune default kube-prometheus rules for your workload mix.

</details>

### Q144. `CPUThrottlingHigh` fires for many pods even though node CPU is low 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CPU limits are low relative to bursty usage; CFS quota throttles per 100ms period regardless of node idle capacity.

**Diagnose →**
```promql
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m]))
/ sum by (pod) (rate(container_cpu_cfs_periods_total[5m]))
```

**Fix →** Raise or remove CPU limits for latency-sensitive services (keep requests).

**Prevent →** Raise the alert threshold or route it as a ticket, not a page.

</details>

### Q145. `KubeMemoryOvercommit` and `KubeCPUOvercommit` fire after adding a namespace 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The sum of requests exceeds what the cluster can absorb after losing a node (N-1).

**Diagnose →** Compare `sum(kube_pod_container_resource_requests{resource="cpu"})` with allocatable minus the largest node.

**Fix →** Add nodes, right-size requests, or let the autoscaler add capacity.

**Prevent →** Review requests with VPA recommendations.

</details>

### Q146. node-exporter DaemonSet does not run on tainted nodes 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No toleration for the taints (GPU, infra, control-plane).

**Diagnose →** `kubectl get ds -n monitoring node-exporter` shows DESIRED < node count; `kubectl describe node` for taints.

**Fix →** Add `tolerations: [{operator: Exists}]` to the DaemonSet.

**Prevent →** Alert when DaemonSet desired != node count.

</details>

### Q147. Prometheus pod logs `forbidden: User "system:serviceaccount:monitoring:prometheus-k8s" cannot list resource "endpoints" in API group "" in the namespace "shop"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Operator namespaced RBAC doesn't include the new namespace (kube-prometheus creates Roles per listed namespace).

**Diagnose →**
```bash
kubectl auth can-i list endpoints -n shop --as=system:serviceaccount:monitoring:prometheus-k8s
```

**Fix →** Add a Role/RoleBinding in `shop`, or use a ClusterRole for cluster-wide discovery.

**Prevent →** Include the monitoring Role in namespace onboarding.

</details>

### Q148. Two Prometheus instances (kube-prometheus and a team's own) both scrape the same targets and double cost 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both select all ServiceMonitors (empty selectors).

**Diagnose →** Compare `/targets` across instances; check each Prometheus CR's selectors.

**Fix →** Give each Prometheus a distinct `serviceMonitorSelector` (e.g. `prometheus: platform` vs `prometheus: team-a`), or use namespace selectors.

**Prevent →** One label convention for ownership of ServiceMonitors.

</details>

### Q149. Operator-generated config fails reload: `relabel configuration for replace action requires 'target_label' value` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A ServiceMonitor `relabelings`/`metricRelabelings` entry is missing `targetLabel`, breaking the whole generated config.

**Diagnose →**
```bash
kubectl logs prometheus-k8s-0 -n monitoring -c config-reloader
kubectl logs prometheus-k8s-0 -n monitoring -c prometheus | grep -i 'error loading config'
```

**Fix →** Add `targetLabel`; find the offending ServiceMonitor from the job name in the error.

**Prevent →** Keep the admission webhook on; newer Operator versions validate relabelings and reject the resource.

</details>

### Q150. Metrics from short-lived Kubernetes Job pods never appear 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The pod finishes before the next scrape (30s interval), or the PodMonitor is selected but the pod is gone by discovery time.

**Diagnose →** Job duration vs scrape interval.

**Fix →** Push final metrics to Pushgateway (or OTel collector) at job end, or have the job stay alive for one scrape window.

**Prevent →** Track job success via kube-state-metrics (`kube_job_status_succeeded`, `kube_job_status_failed`) instead of app metrics.

</details>

---

## Logging: Loki, Fluent Bit and Elasticsearch

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Promtail/Alloy logs `server returned HTTP status 400 Bad Request (400): entry with timestamp ... ignored, reason: 'entry out of order' for stream` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Older Loki (before 2.4) or `unordered_writes: false` rejects entries older than the latest one in the same stream; two agents ship the same file with the same labels; or a pipeline parses timestamps from the line that go backwards.

**Diagnose →**
```bash
kubectl logs -n logging ds/promtail | grep -c 'out of order'
logcli series '{app="checkout"}' --since=1h
```
Check whether two pods/agents produce identical label sets.

**Fix →** Upgrade Loki and enable `limits_config.unordered_writes: true` (default since 2.4), add a unique label per source (`pod`, `instance`), and drop the `timestamp` stage if app timestamps are unreliable.

**Prevent →** One writer per stream; monitor `loki_discarded_samples_total{reason="out_of_order"}`.

</details>

### Q152. Agents log `server returned HTTP status 429 Too Many Requests (429): Ingestion rate limit exceeded for user fake (limit: 4194304 bytes/sec)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The tenant exceeds `ingestion_rate_mb` / `ingestion_burst_size_mb` (defaults are low: 4 MB/s, 6 MB burst), often after a noisy service starts debug logging.

**Diagnose →**
```promql
sum by (tenant, reason) (rate(loki_discarded_samples_total[5m]))
topk(10, sum by (app) (rate(loki_distributor_bytes_received_total[5m])))
```

**Fix →** Find and quiet the noisy source (drop debug lines in the agent), then raise per-tenant limits:
```yaml
limits_config:
  ingestion_rate_mb: 20
  ingestion_burst_size_mb: 40
  per_stream_rate_limit: 5MB
```

**Prevent →** Alert on discarded samples and set per-app log budgets.

</details>

### Q153. Loki rejects pushes with `Per stream rate limit exceeded (limit: 3MB/sec) while attempting to ingest for stream` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One stream (one label set) carries too much volume, e.g. all pods of a deployment collapsed into one stream because `pod` was dropped.

**Diagnose →** `logcli series --analyze-labels '{namespace="shop"}'` and per-stream volume in the Loki dashboards.

**Fix →** Split streams by a bounded label (`pod` or `container`), or raise `per_stream_rate_limit`/`per_stream_rate_limit_burst`.

**Prevent →** Design labels for ~low thousands of streams per tenant, each modest in volume.

</details>

### Q154. Loki logs `maximum active stream limit exceeded, reduce the number of active streams` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** High-cardinality labels in logs: `trace_id`, `user_id`, `request_id`, or JSON-extracted fields promoted to labels.

**Diagnose →**
```bash
logcli series --analyze-labels '{}' --since=1h
```
Look for labels with thousands of unique values.

**Fix →** Remove those labels from the agent pipeline (keep them in the line or as structured metadata in Loki 3.x) and filter at query time with `| json | trace_id="..."`.

**Prevent →** Allowlist labels in agent config; `max_global_streams_per_user` as a guardrail.

</details>

### Q155. LogQL query fails with `max entries limit per query exceeded, limit > max_entries_limit (10000 > 5000)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Grafana line limit or `logcli --limit` is above Loki's `max_entries_limit_per_query`.

**Diagnose →** Check Grafana data source "Maximum lines" and Loki `limits_config`.

**Fix →** Lower the client limit, narrow the query, or raise `max_entries_limit_per_query`.

**Prevent →** Use metric queries (`count_over_time`, `rate`) for overviews instead of pulling raw lines.

</details>

### Q156. LogQL returns `too many outstanding requests` in Grafana 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query-frontend/scheduler queue is full: big time ranges split into many sub-queries with too few queriers.

**Diagnose →** `cortex_query_scheduler_queue_length` / `loki_query_scheduler_queue_length`, querier CPU.

**Fix →** Scale queriers, raise `max_outstanding_requests_per_tenant`, narrow label selectors (stream selector first, line filters before parsers).

**Prevent →** Put recording rules (Loki ruler) behind dashboards that query weeks of logs.

</details>

### Q157. Logs appear in Loki for most pods but not for one namespace 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A relabel/`drop` rule in the agent excludes it, the pods log to files instead of stdout, or the agent pod on those nodes is not running.

**Diagnose →**
```bash
kubectl get ds -n logging
kubectl logs -n logging <agent-pod-on-that-node> | grep -i error
```
Check agent targets UI (Promtail `:9080/targets`, Alloy UI).

**Fix →** Fix drop rules, log to stdout/stderr, add tolerations for tainted nodes.

**Prevent →** Alert on agent DaemonSet coverage and on namespaces with zero log volume.

</details>

### Q158. Loki compactor logs `failed to upload ... AccessDenied` and retention stops working 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM role (IRSA) lacks `s3:DeleteObject`/`s3:PutObject`, or retention isn't enabled on the compactor.

**Diagnose →** Compactor logs; `aws s3 ls s3://loki-chunks/index/` to see if index files update.

**Fix →** Grant `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`; set `compactor.retention_enabled: true` with `delete_request_store: s3` and `limits_config.retention_period: 744h`.

**Prevent →** Alert on `loki_compactor_last_successful_run_timestamp_seconds` staleness.

</details>

### Q159. Fluent Bit logs `[warn] [input] tail.0 paused (mem buf overlimit)` and logs go missing 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Mem_Buf_Limit` is reached because the output is slower than input (backpressure); while paused, new lines are not read and rotated files can be lost.

**Diagnose →**
```bash
curl -s http://127.0.0.1:2020/api/v1/metrics | jq
```
Check `fluentbit_input_paused`, output retries and errors.

**Fix →** Enable filesystem buffering so data spills to disk instead of pausing:
```ini
[SERVICE]
    storage.path  /var/fluent-bit/state/flb-storage/
[INPUT]
    Name          tail
    storage.type  filesystem
```
Fix the slow output (scale Elasticsearch/Loki, increase `Workers`).

**Prevent →** Alert on paused inputs and on `fluentbit_output_dropped_records_total`.

</details>

### Q160. Fluent Bit logs `[error] [output:es:es.0] HTTP status=400 URI=/_bulk` with `mapper_parsing_exception` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mapping conflict: one app sends `status` as a string, another as an object or number, and the index mapping rejects the document. Kubernetes labels with dots (`app.kubernetes.io/name`) also clash with object mappings.

**Diagnose →** Set `Trace_Error On` in the es output to print the ES response; check `GET logs-*/_mapping/field/status`.

**Fix →** Use `Replace_Dots On` in the es output, rename conflicting fields with a `modify` filter, or route the app to its own index.

**Prevent →** Index templates with explicit mappings and `dynamic: false` for unknown fields.

</details>

### Q161. Fluent Bit logs `[warn] [engine] failed to flush chunk ... retry in 10 seconds` repeatedly, then `chunk cannot be retried` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The destination is unreachable or rejecting; after `Retry_Limit` attempts the chunk is discarded.

**Diagnose →** Output errors in logs, `fluentbit_output_retries_failed_total`, connectivity test from the node.

**Fix →** Fix the destination; set `Retry_Limit no_limits` (or `False`) for critical outputs with filesystem buffering and `storage.total_limit_size` to cap disk.

**Prevent →** Alert on `rate(fluentbit_output_retries_failed_total[5m]) > 0`.

</details>

### Q162. Kubernetes metadata is missing from Fluent Bit records (no `kubernetes.pod_name`) 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Kube_Tag_Prefix` doesn't match the tail `Tag`, the ServiceAccount can't read pods, or the API server is unreachable.

**Diagnose →** Fluent Bit logs show `[filter:kubernetes] ... could not get meta for POD` or `403`.

**Fix →** Use `Tag kube.*` on the tail input with `Kube_Tag_Prefix kube.var.log.containers.`, grant `get/list/watch` on pods and namespaces, and consider `Use_Kubelet On` to reduce API load.

**Prevent →** Ship the official Helm chart config and test after edits.

</details>

### Q163. Multi-line Java stack traces arrive in Loki/ES as one event per line 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No multiline parser is configured, or the CRI parser runs after multiline grouping.

**Diagnose →** Search for lines starting with `\tat ` stored as separate entries.

**Fix →** In Fluent Bit:
```ini
[INPUT]
    Name              tail
    multiline.parser  cri, java
```
In Alloy/Promtail use a `multiline` stage with `firstline: '^\d{4}-\d{2}-\d{2}'`.

**Prevent →** Better: log JSON in the app so stack traces are one field.

</details>

### Q164. Fluent Bit CPU spikes and logs `[warn] [input:tail:tail.0] file=... has been rotated` thousands of times 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Extremely chatty containers rotate their log every few seconds (kubelet `containerLogMaxSize` small), plus regex parsers on every line.

**Diagnose →** Find the noisy pods: `du -sh /var/log/pods/* | sort -h | tail`.

**Fix →** Fix the noisy app, raise `containerLogMaxSize`, use `Refresh_Interval`, `Buffer_Max_Size` sensibly, and prefer JSON parser over regex.

**Prevent →** Per-namespace log volume dashboards and quotas.

</details>

### Q165. Elasticsearch cluster health is `red` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** At least one primary shard is unassigned: node lost with no replica, disk watermark reached, or corrupted shard.

**Diagnose →**
```bash
curl -s es:9200/_cluster/health?pretty
curl -s 'es:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason' | grep UNASSIGNED
curl -s es:9200/_cluster/allocation/explain?pretty
```

**Fix →** Act on the explain reason: bring back the node, free disk, or `POST _cluster/reroute?retry_failed=true`. As a last resort for lost primaries, `allocate_empty_primary` (data loss) or restore from snapshot.

**Prevent →** At least 1 replica, snapshot lifecycle policies, disk alerts.

</details>

### Q166. Elasticsearch logs `flood stage disk watermark [95%] exceeded ... all indices on this node will be marked read-only` and writes fail with `FORBIDDEN/12/index read-only / allow delete (api)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Disk on a data node passed the flood-stage watermark; ES set `index.blocks.read_only_allow_delete` on its indices.

**Diagnose →**
```bash
curl -s 'es:9200/_cat/allocation?v'
curl -s 'es:9200/_all/_settings/index.blocks*?pretty' | grep -B3 true
```

**Fix →** Free space (delete old indices, add nodes or disk). Since 7.4 the block is removed automatically once usage drops below the high watermark; on older versions:
```bash
curl -XPUT es:9200/_all/_settings -H 'Content-Type: application/json' \
  -d '{"index.blocks.read_only_allow_delete": null}'
```

**Prevent →** ILM delete phase and alerts at 75% disk.

</details>

### Q167. Elasticsearch cluster is `yellow` on a single-node setup 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Replicas can't be placed on the same node as their primary, so every index with `number_of_replicas: 1` stays yellow.

**Diagnose →** `_cat/shards` shows replicas `UNASSIGNED` with reason `INDEX_CREATED`.

**Fix →** For a single-node lab: `PUT _all/_settings {"index.number_of_replicas": 0}` and set it in the index template. In production, add nodes.

**Prevent →** Template replicas per environment.

</details>

### Q168. Elasticsearch shows `low disk watermark [85%] exceeded ... replicas will not be assigned to this node` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The node is above 85% disk; ES stops allocating new shards to it.

**Diagnose →** `_cat/allocation?v` per node disk.

**Fix →** Delete or shrink old indices, add capacity, rebalance. Adjust `cluster.routing.allocation.disk.watermark.*` only with care.

**Prevent →** ILM rollover by size (`max_primary_shard_size: 50gb`) and delete after N days.

</details>

### Q169. Bulk indexing fails with `es_rejected_execution_exception ... rejected execution ... queue capacity` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The write thread pool queue is full; the cluster can't keep up (hot shards, too many small shards, slow disks).

**Diagnose →**
```bash
curl -s 'es:9200/_cat/thread_pool/write?v&h=node_name,active,queue,rejected'
curl -s 'es:9200/_cat/shards?v&s=store:desc' | head
```

**Fix →** Let clients back off and retry, reduce shard count, increase `refresh_interval` to `30s` on log indices, scale hot nodes.

**Prevent →** Size shards to 10–50 GB and watch the `rejected` counter.

</details>

### Q170. Elasticsearch logs `this action would add [2] shards, but this cluster currently has [1000]/[1000] maximum normal shards open` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Daily indices per app with default shards exceeded `cluster.max_shards_per_node` (1000 per non-frozen data node).

**Diagnose →** `_cat/indices?v&s=index` and `_cluster/stats` shard count.

**Fix →** Delete or shrink old indices, merge per-app indices into data streams with ILM rollover by size.

**Prevent →** Fewer, bigger shards; do not raise the limit as a fix.

</details>

### Q171. Kibana shows `Kibana server is not ready yet` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kibana can't connect to Elasticsearch (wrong URL, credentials, TLS), versions mismatch, or a saved-objects migration is stuck.

**Diagnose →** `kubectl logs deploy/kibana` / `journalctl -u kibana`; look for `Unable to retrieve version information` or migration errors.

**Fix →** Correct `elasticsearch.hosts`, use a service account token, match versions; for stuck migrations follow the error (often a red `.kibana` index).

**Prevent →** Upgrade ES before Kibana and snapshot before upgrades.

</details>

### Q172. Logstash pipeline logs `Could not index event to Elasticsearch ... "type"=>"illegal_argument_exception", "reason"=>"Limit of total fields [1000] has been exceeded"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dynamic mapping creates a field per unique JSON key (e.g. keys containing IDs), hitting `index.mapping.total_fields.limit`.

**Diagnose →** `GET logs-*/_mapping` and count fields; find the app sending dynamic keys.

**Fix →** Fix the producer, map the dynamic object as `flattened`, or set `dynamic: false` on that object.

**Prevent →** Index templates with explicit mappings; avoid raising the limit blindly.

</details>

### Q173. Logs in Elasticsearch are 20 minutes behind real time 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Queue backlog somewhere: agent buffer, Kafka consumer lag, Logstash pipeline workers saturated, or ES indexing slow.

**Diagnose →** Compare `@timestamp` vs ingest time (add an ingest pipeline `set` of `event.ingested`); check Kafka lag and `_nodes/stats/indices/indexing`.

**Fix →** Scale the bottleneck stage (pipeline workers, partitions/consumers, ES hot nodes), batch larger bulk requests.

**Prevent →** Alert on end-to-end lag (`event.ingested - @timestamp`).

</details>

### Q174. Alloy logs `error sending batch, will retry ... status=401 ... no org id` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Loki runs with `auth_enabled: true` and the agent sends no tenant ID.

**Diagnose →** Check the `loki.write` endpoint config for `tenant_id`.

**Fix →**
```alloy
loki.write "default" {
  endpoint {
    url       = "http://loki-gateway/loki/api/v1/push"
    tenant_id = "prod"
  }
}
```

**Prevent →** Template tenant IDs per cluster.

</details>

### Q175. Loki queriers log `context canceled` and queries over 7 days time out 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Broad stream selectors (`{namespace=~".+"}`) force reading huge chunk volumes; no query splitting or caching; `query_timeout` short.

**Diagnose →** Query stats in Grafana (bytes processed) and querier logs.

**Fix →** Narrow the selector, put line filters (`|= "error"`) before parsers, enable `split_queries_by_interval: 1h`, results cache and chunks cache (memcached), raise `query_timeout` modestly.

**Prevent →** Teach teams the LogQL order: stream selector → line filter → parser → label filter.

</details>

---

## Tracing, OpenTelemetry and CloudWatch

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. Traces show only the frontend span; downstream services start new traces 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Broken context propagation: the `traceparent` header isn't forwarded (custom HTTP client, message queue, async thread pool), or services use different propagators (W3C vs B3).

**Diagnose →** Log incoming headers in the downstream service, or `curl -v` through the chain; check `OTEL_PROPAGATORS` on each service.

**Fix →** Use instrumented clients (auto-instrumentation), set `OTEL_PROPAGATORS=tracecontext,baggage` (add `b3` where legacy services need it), inject/extract context manually for queues (message headers).

**Prevent →** A contract test that asserts trace IDs are continuous across services.

</details>

### Q177. Traces break at the API gateway / ingress controller 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ingress strips or regenerates `traceparent`, or doesn't participate in tracing.

**Diagnose →** Compare `traceparent` before and after the ingress with request logs.

**Fix →** Enable OpenTelemetry in the ingress (ingress-nginx `enable-opentelemetry: "true"` with `otlp-collector-host`), or configure it to pass headers through untouched.

**Prevent →** Include proxies, gateways and service meshes in the propagation design.

</details>

### Q178. No traces at all arrive; the app logs `Failed to export spans. The request could not be executed. Full error message: Failed to connect to localhost/127.0.0.1:4317` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SDK defaults to `localhost:4317` but the collector runs elsewhere (DaemonSet on the node, or a Service).

**Diagnose →** Check `OTEL_EXPORTER_OTLP_ENDPOINT` and that the collector listens: `kubectl get svc -n observability`.

**Fix →** Set `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.observability:4317` (gRPC) or `:4318` with `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`. For a node agent use the downward API `status.hostIP`.

**Prevent →** Inject OTel env vars via the OpenTelemetry Operator `Instrumentation` CR.

</details>

### Q179. SDK error `Unimplemented` / `404 Not Found` when exporting to the collector 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Protocol/port mismatch: sending gRPC to the HTTP port 4318, or HTTP to 4317, or the HTTP path missing `/v1/traces` when setting a signal-specific endpoint.

**Diagnose →** Compare `OTEL_EXPORTER_OTLP_PROTOCOL` with the port.

**Fix →** gRPC → 4317; `http/protobuf` → 4318. With `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` include the full path `http://collector:4318/v1/traces`.

**Prevent →** One documented default for all services.

</details>

### Q180. OTel collector logs `Exporting failed. Will retry the request after interval. {"kind": "exporter", "data_type": "traces", "name": "otlp", "error": "rpc error: code = Unavailable desc = connection error"}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backend (Tempo, Jaeger, vendor) is down or unreachable, TLS mismatch (backend expects plaintext or vice versa), or DNS.

**Diagnose →**
```bash
kubectl logs -n observability deploy/otel-collector | grep -i export
kubectl exec -n observability deploy/otel-collector -- nc -zv tempo-distributor 4317
```

**Fix →** Fix endpoint/port; for plaintext in-cluster backends set `tls: { insecure: true }` on the exporter; otherwise provide `ca_file`.

**Prevent →** Alert on `otelcol_exporter_send_failed_spans` and queue size.

</details>

### Q181. Collector logs `Dropping data because sending_queue is full` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The exporter can't keep up (slow backend, rate limiting), so the in-memory queue overflows and data is dropped.

**Diagnose →**
```promql
otelcol_exporter_queue_size / otelcol_exporter_queue_capacity
rate(otelcol_exporter_enqueue_failed_spans[5m])
```

**Fix →** Fix backend throughput, increase `sending_queue.num_consumers` and `queue_size`, enable persistent queue with `file_storage`, scale collectors horizontally, and sample.

**Prevent →** Alert at 70% queue utilisation.

</details>

### Q182. Collector pod is OOMKilled under load 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `memory_limiter` processor, large batches, or tail sampling holding many traces in memory.

**Diagnose →** `kubectl describe pod` (OOMKilled), `otelcol_process_memory_rss`.

**Fix →** Put `memory_limiter` first in every pipeline:
```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20
  batch: {}
service:
  pipelines:
    traces:
      processors: [memory_limiter, batch]
```
Set `GOMEMLIMIT` to ~80% of the container limit.

**Prevent →** Load-test collectors and alert on refused spans.

</details>

### Q183. Collector fails to start: `error decoding 'exporters': unknown type: "loki"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The component isn't included in the distribution (core vs contrib), or it was removed/deprecated (the `loki` exporter was replaced by sending OTLP to Loki 3.x).

**Diagnose →** `otelcol-contrib components` lists available components.

**Fix →** Use the contrib image (`otel/opentelemetry-collector-contrib`), or switch to the supported path (`otlphttp` exporter to Loki's `/otlp` endpoint).

**Prevent →** Pin collector versions and read changelogs; build a custom distro with `ocb` for exactly what you need.

</details>

### Q184. Collector config loads but pipeline silently does nothing 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Components are defined but not referenced in `service.pipelines`, so they are never started.

**Diagnose →** Startup logs list started components; `otelcol validate --config=config.yaml`.

**Fix →** Add receivers, processors and exporters to the pipeline under `service.pipelines.traces/metrics/logs`.

**Prevent →** Validate configs in CI with `otelcol validate`.

</details>

### Q185. Collector `prometheusremotewrite` exporter logs `Permanent error: ... 400 Bad Request: out of order sample` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiple collector replicas send the same series (e.g. same `prometheus` receiver scrape config on every replica), or batching reorders samples.

**Diagnose →** Check whether series labels identify the collector replica; count collector pods running the scrape.

**Fix →** Shard scraping with the Target Allocator (OTel Operator) or run scrape receivers as a single StatefulSet; add a unique `collector` resource attribute; enable out-of-order on the backend if needed.

**Prevent →** Treat scraping in collectors like Prometheus: one scraper per target.

</details>

### Q186. Traces exist but spans have `service.name=unknown_service:java` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `OTEL_SERVICE_NAME` or `service.name` resource attribute not set.

**Diagnose →** Inspect a span's resource attributes in Tempo/Jaeger.

**Fix →** Set `OTEL_SERVICE_NAME=checkout` and `OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,service.version=1.4.2`.

**Prevent →** Inject via the OTel Operator or Helm templates; enforce with a collector `resource` processor default.

</details>

### Q187. Only ~1% of traces show up, and the error trace you need is missing 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Head sampling (`OTEL_TRACES_SAMPLER=parentbased_traceidratio`, `OTEL_TRACES_SAMPLER_ARG=0.01`) decides before the outcome is known.

**Diagnose →** Check sampler env vars and collector processors.

**Fix →** Use tail sampling in a collector tier: keep all errors and slow traces, sample the rest:
```yaml
tail_sampling:
  policies:
    - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
    - { name: slow, type: latency, latency: { threshold_ms: 1000 } }
    - { name: rest, type: probabilistic, probabilistic: { sampling_percentage: 5 } }
```
Route all spans of a trace to the same collector with the `loadbalancing` exporter (`routing_key: traceID`).

**Prevent →** Document sampling so engineers know what is kept.

</details>

### Q188. Tail sampling keeps partial traces with missing spans 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Spans of one trace land on different collector replicas, each deciding independently; or `decision_wait` is shorter than the trace duration.

**Diagnose →** Compare span counts for one trace across collectors; check `otelcol_processor_tail_sampling_sampling_late_span_age`.

**Fix →** Two-tier setup: agents → `loadbalancing` exporter by trace ID → sampling collectors. Raise `decision_wait` (e.g. 30s) and `num_traces` to fit memory.

**Prevent →** Size sampling tier for peak spans/sec × decision_wait.

</details>

### Q189. Async message consumers (Kafka/SQS) appear as separate, unlinked traces 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Context isn't injected into message headers/attributes, or the consumer doesn't extract it.

**Diagnose →** Inspect a message's headers for `traceparent`.

**Fix →** Use instrumented Kafka/SQS clients, or inject with the propagator API on produce and extract on consume; for batches, use span links instead of parent-child.

**Prevent →** Make propagation part of the messaging library your teams share.

</details>

### Q190. Tempo returns `trace not found` for traces that exist a few minutes later 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Querying before ingesters flush, or the querier isn't configured to query ingesters for recent data (`query_backend_after`/`query_ingesters_until`).

**Diagnose →** Retry after a minute; check Tempo querier config and ingester health.

**Fix →** Make sure the query frontend searches both ingesters and backend for recent windows; check ingester replication.

**Prevent →** Monitor `tempo_ingester_live_traces` and flush durations.

</details>

### Q191. Trace timing looks impossible: child span starts before its parent 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clock skew between hosts, since each service stamps its own spans.

**Diagnose →** Compare `node_timex_offset_seconds` across hosts; check chrony status.

**Fix →** Fix NTP (chrony with Amazon Time Sync `169.254.169.123` on EC2); Jaeger UI has clock skew adjustment but it only masks the issue.

**Prevent →** Alert on clock offset > 50ms.

</details>

### Q192. Exemplars don't show on Grafana latency graphs 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Prometheus isn't started with `--enable-feature=exemplar-storage`, the app doesn't attach trace IDs to observations, scraping doesn't use OpenMetrics, or the panel's "Exemplars" toggle is off.

**Diagnose →**
```bash
curl -s -H 'Accept: application/openmetrics-text' http://app:8080/metrics | grep '# {trace_id'
```

**Fix →** Enable exemplar storage, use a client library that attaches exemplars from the active span, toggle Exemplars in the query, and configure the data source's exemplar link to Tempo with `trace_id`.

**Prevent →** Include exemplars in the instrumentation library defaults.

</details>

### Q193. CloudWatch agent is running but no custom metrics appear in the `CWAgent` namespace 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IAM role lacks `cloudwatch:PutMetricData`, the agent runs with the default config (not yours), or you're looking in the wrong region.

**Diagnose →**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
sudo tail -50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

**Fix →** Attach `CloudWatchAgentServerPolicy` to the instance role, then load your config:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

**Prevent →** Bake agent + config into the AMI or SSM State Manager association.

</details>

### Q194. CloudWatch agent log shows `AccessDeniedException: User: arn:aws:sts::...:assumed-role/... is not authorized to perform: logs:PutLogEvents` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Role permissions cover metrics but not logs, or log group is encrypted with a KMS key the role can't use.

**Diagnose →** Read the agent log; check the role and the log group's `kmsKeyId`.

**Fix →** Grant `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`, `logs:DescribeLogStreams` (included in `CloudWatchAgentServerPolicy`) and KMS `kms:GenerateDataKey`/`kms:Encrypt` if needed.

**Prevent →** Use a single managed policy for all agent hosts.

</details>

### Q195. CloudWatch agent on a private subnet logs `RequestError: send request failed ... dial tcp ...:443: i/o timeout` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No NAT gateway and no VPC interface endpoints for `monitoring` and `logs`.

**Diagnose →**
```bash
curl -sv https://monitoring.ap-south-1.amazonaws.com 2>&1 | head
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-xxx
```

**Fix →** Create interface endpoints `com.amazonaws.<region>.monitoring` and `com.amazonaws.<region>.logs` with private DNS enabled and a security group allowing 443 from the instances.

**Prevent →** Include these endpoints in the VPC Terraform module.

</details>

### Q196. Memory and disk metrics are missing in CloudWatch for EC2, only CPU shows 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** EC2 basic monitoring has no in-guest metrics; memory and disk need the CloudWatch agent with `mem` and `disk` sections.

**Diagnose →** Look for `mem_used_percent` in the `CWAgent` namespace.

**Fix →** Add to config:
```json
"metrics": { "metrics_collected": {
  "mem":  { "measurement": ["mem_used_percent"] },
  "disk": { "measurement": ["used_percent"], "resources": ["/"] } } }
```
Then `fetch-config` and restart.

**Prevent →** Standard agent config for all instances.

</details>

### Q197. CloudWatch alarm stays `INSUFFICIENT_DATA` for an Auto Scaling group's instances 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The alarm's dimensions don't match published ones (e.g. alarm on `InstanceId` + `AutoScalingGroupName` but agent publishes only `InstanceId`), or `append_dimensions` / `aggregation_dimensions` weren't configured.

**Diagnose →**
```bash
aws cloudwatch list-metrics --namespace CWAgent --metric-name mem_used_percent
```

**Fix →** Add `"append_dimensions": {"AutoScalingGroupName": "${aws:AutoScalingGroupName}"}` and `"aggregation_dimensions": [["AutoScalingGroupName"]]`; alarm on the ASG dimension.

**Prevent →** Set `TreatMissingData` deliberately on every alarm.

</details>

### Q198. Container Insights on EKS shows no data after installing the add-on 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CloudWatch agent / Fluent Bit pods lack IAM permissions (no IRSA or Pod Identity), or pods are not scheduled on all nodes.

**Diagnose →**
```bash
kubectl get pods -n amazon-cloudwatch
kubectl logs -n amazon-cloudwatch ds/cloudwatch-agent | grep -i 'denied\|error'
```

**Fix →** Associate `CloudWatchAgentServerPolicy` with the `cloudwatch-agent` service account via IRSA or EKS Pod Identity, then restart the DaemonSet.

**Prevent →** Manage the add-on and its IAM in Terraform together.

</details>

### Q199. AWS X-Ray console shows segments but service map has disconnected nodes 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mixed propagation: some services emit W3C `traceparent` via OTel while others expect `X-Amzn-Trace-Id`; or the ADOT collector exports with non-X-Ray-compatible trace IDs from older SDKs.

**Diagnose →** Inspect inbound headers at each hop; check the ADOT collector `awsxray` exporter logs.

**Fix →** Configure the propagators to include X-Ray (`OTEL_PROPAGATORS=tracecontext,baggage,xray`) and use the X-Ray ID generator where the SDK requires it; send through ADOT with the `awsxray` exporter.

**Prevent →** Pick one propagation standard org-wide and bridge only at edges.

</details>

### Q200. Prometheus `up` is fine, logs flow and traces exist, but on-call still can't correlate them during an incident 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inconsistent identity across signals: `service` in metrics, `app` in logs, `service.name` in traces; no trace IDs in logs; different environment labels.

**Diagnose →** Pick one request and try to follow it metric → trace → log; note where the trail breaks.

**Fix →** Standardise on OTel resource attributes (`service.name`, `deployment.environment`, `k8s.namespace.name`) and map them to the same label names in Prometheus and Loki; log `trace_id`/`span_id` in every line; configure Grafana derived fields, exemplars, and Tempo's trace-to-logs/trace-to-metrics links.

**Prevent →** A shared instrumentation library and a correlation check in production readiness reviews.

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
