# SRE & Reliability — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Most run on a laptop with kind, Prometheus and a small demo service (the one from the [hands-on lab](./hands-on-lab.md) works for all of them). Some are paper exercises; do them in writing, because in an interview you will have to say the numbers out loud.

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Turn SLO targets into downtime and request budgets |
| 2 | 🟢 | Write SLIs for three user journeys |
| 3 | 🟢 | Instrument a service with request metrics |
| 4 | 🟢 | Query availability and latency SLIs in PromQL |
| 5 | 🟢 | Write an error-budget policy |
| 6 | 🟢 | Page or ticket: audit an alert list |
| 7 | 🟢 | Measure and rank your toil |
| 8 | 🟢 | Write a runbook that someone else can follow |
| 9 | 🟢 | Define a severity matrix and run a tabletop incident |
| 10 | 🟢 | Rewrite a blameful postmortem and run the 5 whys |
| 11 | 🟡 | Multi-window SLI recording rules |
| 12 | 🟡 | Burn-rate alerts with promtool unit tests |
| 13 | 🟡 | Alertmanager routing, inhibition and silences |
| 14 | 🟡 | A Grafana SLO dashboard with budget burndown |
| 15 | 🟡 | Availability math for a real dependency graph |
| 16 | 🟡 | Load test with k6 and find the knee |
| 17 | 🟡 | Chaos Mesh experiments with written hypotheses |
| 18 | 🟡 | Timeouts, retries with jitter and a circuit breaker in code |
| 19 | 🟡 | Canary analysis gated on an SLI with Argo Rollouts |
| 20 | 🟡 | Compute DORA metrics from your own pipeline |
| 21 | 🔴 | SLOs as code with Sloth and Pyrra |
| 22 | 🔴 | A game day with stop conditions (AWS FIS or LitmusChaos) |
| 23 | 🔴 | Build a capacity model with N+2 headroom |
| 24 | 🔴 | Disaster recovery for a stateful service, proven by a timed restore |
| 25 | 🔴 | Lead a mock SEV-1 and run the postmortem review |

---

## 🟢 Beginner

### Assignment 1. Turn SLO targets into downtime and request budgets 🟢

**Objective:** Be able to convert any SLO into allowed downtime and allowed failed requests without a calculator app.

**Tasks**
1. For 99%, 99.5%, 99.9%, 99.95%, 99.99% and 99.999%, compute allowed downtime per day, per week, per 28 days, per 30 days and per year.
2. A service handles a steady 250 requests per second. For a 99.9% SLO over 30 days, compute the total requests and the number of failed requests allowed.
3. An outage returns 100% errors for 18 minutes. What percentage of a 99.9% 30-day budget did it consume? What about at 99.95%?
4. An incident fails 5% of requests for 3 hours. Compute the budget consumed at 99.9% over 30 days.
5. Write a 20-line script (bash, Python or Go) that takes an SLO and a window and prints the downtime budget in human units.

**Expected outcome / acceptance criteria**
- Your table matches: 99.9% = 43.2 min per 30 days, 99.99% = 4.32 min per 30 days, 99.999% = about 5.26 min per year.
- Task 2: 648,000,000 requests and 648,000 allowed failures.
- Task 3: about 41.7% at 99.9%, about 83.3% at 99.95%.
- Task 4: 0.05 × 180 min = 9 minutes of equivalent full downtime, about 20.8% of the budget.

<details>
<summary>Hints</summary>

Allowed downtime = (1 - SLO) × window. 30 days = 43,200 minutes; a year = 525,600 minutes. Partial outages convert to "equivalent full downtime" by multiplying duration by error ratio. Budget consumed = bad events / (total events × (1 - SLO)). In Python: `print(f"{(1-slo)*window_min:.2f} min")`.

</details>

**Bonus:** Add a column for "time to exhaust the budget at 100% errors" and at "10x burn rate", and explain why the two answers differ only by the error ratio.

---

### Assignment 2. Write SLIs for three user journeys 🟢

**Objective:** Choose SLIs that reflect user experience, expressed as good events over valid events.

**Tasks**
1. Pick three journeys from a real or imagined product: an HTTP API (for example "place order"), an asynchronous pipeline (for example "invoice email sent"), and a storage or data journey (for example "uploaded file can be downloaded").
2. For each, write one to three SLIs in the form *good events / valid events*, and state exactly where you measure (load balancer, service, client, synthetic probe, pipeline database).
3. For each SLI, list what is excluded from valid events and why (health checks, client 4xx, bots, internal traffic).
4. Propose an SLO target and window for each, and justify it from user expectations, not from current performance alone.
5. Write one sentence per SLI explaining what user pain it would miss.

**Expected outcome / acceptance criteria**
- At least one latency SLI uses a threshold ("served in under 400 ms") rather than an average.
- The pipeline journey uses a **freshness** or **end-to-end latency** SLI, not request availability.
- 429s caused by your own overload are counted as bad; client mistakes (400, 401, 404) are excluded.
- Each SLO has a stated window (for example rolling 28 days).

<details>
<summary>Hints</summary>

SLI types: availability, latency, quality (degraded responses), freshness, correctness, coverage, durability, throughput. For a queue-based pipeline: "proportion of invoices emailed within 10 minutes of the order". Measure as close to the user as practical; the load balancer catches crashed pods that the app's own metrics never see.

</details>

**Bonus:** Write the same SLIs in the OpenSLO YAML format (`apiVersion: openslo/v1`, `kind: SLO`).

---

### Assignment 3. Instrument a service with request metrics 🟢

**Objective:** Expose the two metrics every request-based SLI needs: a request counter by status and a latency histogram with sensible buckets.

**Tasks**
1. Write or reuse a small HTTP service (Python, Go or Node) with at least two routes.
2. Add a counter `http_requests_total{route, method, code}` and a histogram `http_request_duration_seconds{route, method}`.
3. Choose histogram buckets so that your intended latency SLO threshold is an exact bucket boundary.
4. Use the route **template** (`/users/{id}`), not the raw path, as the label value.
5. Run it in kind, add a ServiceMonitor (or a static scrape config), and confirm Prometheus scrapes it.

**Expected outcome / acceptance criteria**
- `curl :8080/metrics` shows `_bucket`, `_sum` and `_count` series and a `le` value equal to your SLO threshold.
- `count(http_requests_total)` in Prometheus stays small and does not grow with each new user ID.
- The target is `up` in Prometheus.

<details>
<summary>Hints</summary>

Python: `Histogram("http_request_duration_seconds", "...", ["route","method"], buckets=(0.05,0.1,0.25,0.5,1,2.5))`. Go: `promauto.NewHistogramVec(prometheus.HistogramOpts{Name: "http_request_duration_seconds", Buckets: []float64{...}}, []string{"route","method"})`. Do not add labels like `pod` or `namespace` in code; Prometheus adds them from the target. Raw paths as labels cause cardinality explosions.

</details>

**Bonus:** Enable Prometheus native histograms (`--enable-feature=native-histograms`, check the release notes of your Prometheus version), expose a native histogram from the client library, and compare the number of series with the classic histogram.

---

### Assignment 4. Query availability and latency SLIs in PromQL 🟢

**Objective:** Write the core SLI queries by hand and understand why each part is there.

**Tasks**
1. Write an availability error ratio over 5 minutes: 5xx requests / all requests for one job.
2. Write a latency SLI over 5 minutes: requests under your threshold / all requests, using the histogram bucket.
3. Write p50 and p99 latency with `histogram_quantile` and explain why p99 is not the same as the latency SLI.
4. Make the availability query return `0` instead of "no data" when no 5xx has ever occurred.
5. Compute the 30-day SLI and the remaining error budget as a percentage.

**Expected outcome / acceptance criteria**
- Queries return sane values during a short load test (for example `hey -z 2m -q 20 http://...` or a k6 script).
- You can explain why `sum(rate(...))` is applied before the division, not `rate()` of a ratio.
- Your budget query returns 1.0 (100%) on a service that has had no errors.

<details>
<summary>Hints</summary>

```promql
(sum(rate(http_requests_total{job="shop-api",code=~"5.."}[5m])) or vector(0))
/ sum(rate(http_requests_total{job="shop-api"}[5m]))

sum(rate(http_request_duration_seconds_bucket{job="shop-api",le="0.25"}[5m]))
/ sum(rate(http_request_duration_seconds_count{job="shop-api"}[5m]))
```

Budget remaining = `1 - (error_ratio_30d / (1 - 0.999))`. Averaging ratios across pods weights quiet pods the same as busy ones; summing numerators and denominators first does not.

</details>

**Bonus:** Write the latency SLI with native histograms using `histogram_fraction(0, 0.25, sum(rate(http_request_duration_seconds[5m])))`.

---

### Assignment 5. Write an error-budget policy 🟢

**Objective:** Produce the document that turns an SLO from a dashboard number into decisions.

**Tasks**
1. Pick one service and its SLO (from assignment 2).
2. Define what happens at budget levels: healthy, below 50%, below 25%, exhausted.
3. Define what counts against the budget (incidents, planned maintenance, dependency failures) and what does not (for example a cloud provider region outage you explicitly accepted).
4. Name who can grant an exception and how it is recorded.
5. Define the trigger for a mandatory postmortem (for example any incident consuming more than 20% of the budget).
6. Get feedback from one developer and one product-minded person and record what changed.

**Expected outcome / acceptance criteria**
- One page, signed off by service owner, SRE and product (or people role-playing them).
- The "exhausted" action is concrete: for example "feature releases frozen; only fixes for reliability and security ship; P1 postmortem actions jump the queue".
- The policy says how the freeze ends (budget recovers, or an approved exception).

<details>
<summary>Hints</summary>

The policy must be agreed **before** the budget runs out; negotiated during an incident it becomes a fight. Keep the consequences proportional and time-bound. Excluding dependency failures from your budget hides real user pain; prefer counting them and fixing the dependency contract.

</details>

**Bonus:** Add a "budget surplus" clause: what the team may do when consistently using less than 20% of the budget (ship faster, run chaos experiments, lower the SLO cost).

---

### Assignment 6. Page or ticket: audit an alert list 🟢

**Objective:** Learn to separate symptoms from causes and pages from tickets.

**Tasks**
1. Export the alert rules from a real Prometheus (kube-prometheus-stack works: `kubectl get prometheusrules -A -o yaml`) and pick 20 alerts.
2. For each alert, record: symptom or cause, urgent or not, actionable or not, needs human judgment or could be automated.
3. Classify each as page, ticket, dashboard only, or delete.
4. For every alert you keep as a page, check it has a runbook link and an owner.
5. Propose two SLO-based symptom alerts that would replace at least five of the cause-based alerts.

**Expected outcome / acceptance criteria**
- A table of 20 rows with a decision and one-line reason each.
- Fewer than a quarter of alerts remain pages.
- CPU, memory, single-pod restarts and disk usage at 70% are not pages.

<details>
<summary>Hints</summary>

Page test: urgent, actionable now, needs a human, and affects users (or soon will). "Disk will be full in 4 hours" (`predict_linear`) can be a page; "disk at 80%" is a ticket. `KubePodCrashLooping` is usually a ticket when replicas still serve traffic. `Watchdog` is a dead man's switch: it should always fire and route to an external heartbeat service.

</details>

**Bonus:** Track your own (or your team's) pages for two weeks and compute pages per shift and the percentage that required action.

---

### Assignment 7. Measure and rank your toil 🟢

**Objective:** Quantify toil so it can be prioritized like any other engineering work.

**Tasks**
1. For two weeks (or from your ticket history), list every recurring operational task: restarts, certificate renewals, access requests, manual deploy steps, data fixes.
2. For each, record frequency, minutes per occurrence, and whether it scales with traffic or number of services.
3. Mark each against the toil criteria: manual, repetitive, automatable, tactical, no enduring value, grows linearly.
4. Compute hours per month per task and rank them.
5. For the top three, write an automation proposal: approach, effort estimate, hours saved per month, payback period.

**Expected outcome / acceptance criteria**
- A ranked table with hours per month.
- Total toil as a percentage of working time, compared with the 50% SRE guideline.
- At least one proposal that removes the task entirely (fix the cause) rather than scripting it.

<details>
<summary>Hints</summary>

Payback = effort hours / hours saved per month. Examples: cert-manager replaces manual renewal; a self-service access workflow replaces tickets; fixing a memory leak beats a cron job that restarts pods. Incident response and postmortems are not toil when they produce learning.

</details>

**Bonus:** Tag toil tickets in your tracker (for example label `toil`) and build a monthly chart.

---

### Assignment 8. Write a runbook that someone else can follow 🟢

**Objective:** Write a runbook good enough that a teammate who has never seen the service can mitigate the alert.

**Tasks**
1. Pick one page-worthy alert (for example a burn-rate alert or "database connections exhausted").
2. Write the runbook: what the alert means, user impact, how to confirm (with exact queries and commands), mitigation steps in order of safety, escalation, and links to dashboards.
3. Link it from the alert's `runbook_url` annotation.
4. Give it to someone else, break the service on purpose, and let them follow the runbook while you watch silently.
5. Record every point where they hesitated and fix the runbook.

**Expected outcome / acceptance criteria**
- The tester mitigates the problem using only the runbook.
- Every command is copy-pasteable with placeholders clearly marked.
- The first section can be read in under 60 seconds.

<details>
<summary>Hints</summary>

Order: confirm impact, mitigate, then diagnose. Prefer reversible actions first (rollback, scale up, flag off). Put the "when to escalate and to whom" section near the top, not the bottom. A runbook that has not been followed by someone else is untested.

</details>

**Bonus:** Turn the first three steps of the runbook into a script or a ChatOps command, keeping the human decision points explicit.

---

### Assignment 9. Define a severity matrix and run a tabletop incident 🟢

**Objective:** Practise incident roles and communication without breaking anything.

**Tasks**
1. Write a severity matrix (SEV-1 to SEV-4) based on user impact, with examples and expected response for your product.
2. Write the role descriptions: Incident Commander, Ops lead, Comms lead, Scribe.
3. With two to four people, run a 30-minute tabletop: a facilitator reveals a scenario step by step ("checkout errors at 4%", "now 30%", "the database team says it's fine").
4. Rotate the IC role halfway through and practise a formal handover.
5. Write three status updates (investigating, mitigated, resolved) as the Comms lead.

**Expected outcome / acceptance criteria**
- The severity was declared within the first five minutes and changed at least once as facts arrived.
- The IC never ran a command.
- The handover included current status, hypothesis, actions in flight and next update time.
- Status updates contain impact, time of next update, and no speculation about root cause.

<details>
<summary>Hints</summary>

Severity is about impact, not difficulty. Declare early: downgrading costs nothing. Status update template: what users experience, since when, what we are doing, next update time. The IC's two most-used phrases: "who owns that?" and "when will we know?".

</details>

**Bonus:** Repeat with an incident that crosses teams (your service and a dependency owned by someone else) and practise pulling in their on-call.

---

### Assignment 10. Rewrite a blameful postmortem and run the 5 whys 🟢

**Objective:** Learn to write postmortems that find system causes instead of blaming people.

**Tasks**
1. Start from this summary: *"Ravi ran a database migration on Friday evening without testing it. It locked the orders table and checkout was down for 47 minutes. Ravi should have known better. Action: Ravi to be more careful."*
2. Rewrite it in blameless form using a standard template (impact, timeline, root cause, contributing factors, went well, went wrong, got lucky, action items).
3. Run the 5 whys from "checkout was down", and continue until you reach process or design gaps.
4. Produce at least five action items with types (prevent, detect, mitigate, process), owners and due dates.
5. Compute the budget impact for a 99.9% 30-day SLO.

**Expected outcome / acceptance criteria**
- No sentence implies a person is the cause; the engineer appears only as a neutral actor.
- Contributing factors include things like: no migration review or linter for locking operations, deploys allowed on Friday evening without a policy, no lock timeout, slow detection.
- 47 minutes exceeds the entire 43.2-minute budget; the policy consequence is stated.

<details>
<summary>Hints</summary>

Ask "why did this action make sense at the time?" and "why did the system allow it?". Concrete fixes: `SET lock_timeout = '5s'` in migrations, online schema change tools (gh-ost, pg-osc, `CREATE INDEX CONCURRENTLY`), a CI check for blocking DDL, staging with production-sized data, an alert that would have fired in two minutes.

</details>

**Bonus:** Draw the causal tree as a Mermaid flowchart instead of a linear 5 whys and show where two independent factors combined.

---

## 🟡 Intermediate

### Assignment 11. Multi-window SLI recording rules 🟡

**Objective:** Precompute SLIs for every alerting window with clean naming and correct edge-case handling.

**Tasks**
1. For one service, write recording rules for an availability error ratio over 5m, 30m, 1h, 2h, 6h, 1d and 3d, named `slo:sli_error:ratio_rate<window>` with an `slo` label.
2. Do the same for a latency SLI based on a histogram bucket.
3. Add `slo:objective:ratio` as a constant per SLO, and an `slo:error_budget_remaining:ratio30d` rule that joins on `slo`.
4. Deploy them as a PrometheusRule and confirm they evaluate with `health="ok"`.
5. Validate syntax with `promtool check rules` in a pre-commit hook or CI job.

**Expected outcome / acceptance criteria**
- `curl -s localhost:9090/api/v1/rules | jq` shows every rule healthy.
- The availability ratio is `0`, not empty, before the first 5xx.
- With zero traffic the ratio is NaN, and you can explain why that is acceptable.

<details>
<summary>Hints</summary>

Wrap the numerator: `(sum(rate(...{code=~"5.."}[1h])) or vector(0))`. Constants: `expr: vector(0.999)` with `labels: {slo: name}`. Join: `slo:sli_error:ratio_rate30d / on (slo) (1 - slo:objective:ratio)`. For CI, extract the spec from the CRD: `yq '.spec' rule.yaml > plain.yaml`.

</details>

**Bonus:** Replace the raw `rate(...[3d])` with a cheaper approximation built from the 1h recorded series (for example `avg_over_time(slo:sli_error:ratio_rate1h[3d])`) and explain the accuracy trade-off when traffic varies by time of day.

---

### Assignment 12. Burn-rate alerts with promtool unit tests 🟡

**Objective:** Write the four standard burn-rate alerts and prove their behaviour with tests before they reach production.

**Tasks**
1. Write the four alerts: 14.4x (1h and 5m) and 6x (6h and 30m) with `severity: page`; 3x (1d and 2h) and 1x (3d and 6h) with `severity: ticket`.
2. Write a `promtool test rules` file with at least four tests: a full outage pages fast; a 1% error ratio pages only through the 6x rule; a 0.2% error ratio opens only a ticket; a service exactly at its SLO alerts nobody.
3. Add a test that the fast alert **resolves** within 10 minutes after errors stop, even though the 1h window is still high.
4. Introduce a deliberate bug (wrong threshold, typo in a label) and confirm a test catches it.
5. Run the tests in CI on every change to the rules.

**Expected outcome / acceptance criteria**
- All tests pass; the deliberate bug fails a test with a clear `exp` vs `got` diff.
- You can derive each threshold from "budget spent" and window length.
- The resolve test passes because of the short window.

<details>
<summary>Hints</summary>

Threshold = budget fraction × window period / long window: 0.02 × 720 h / 1 h = 14.4; 0.05 × 720 / 6 = 6; 0.10 × 720 / 24 = 3; 0.10 × 720 / 72 = 1. In tests, chain segments to switch behaviour mid-test: `'0+60x60 3600+0x30'` is a 5xx counter that grows by 60 per minute for an hour, then stays flat (errors stopped). Each segment states its own start value, so keep it equal to where the previous one ended or you create a counter reset. Annotations in `exp_annotations` must match exactly, so keep them static.

</details>

**Bonus:** Add a minimum-traffic guard to the page alerts and a test showing one error out of five requests no longer pages.

---

### Assignment 13. Alertmanager routing, inhibition and silences 🟡

**Objective:** Make sure the right alert reaches the right place exactly once.

**Tasks**
1. Configure routes: `severity=page` to a pager receiver (a webhook sink is fine), `severity=ticket` to a ticket receiver, everything else to `null`.
2. Group by `alertname` and `slo`; set `group_wait`, `group_interval` and `repeat_interval` deliberately and write down why.
3. Add an inhibition rule: while a page alert fires for an SLO, suppress the ticket alert for the same SLO.
4. Use `amtool config routes test` to prove where five different label sets go.
5. Create a silence for a planned maintenance window with `amtool silence add`, verify the alert is suppressed, then expire it.

**Expected outcome / acceptance criteria**
- `amtool check-config` passes.
- During a fast burn only one notification arrives at the pager; the ticket alert is inhibited.
- The silence has an author, a comment and an end time.

<details>
<summary>Hints</summary>

```yaml
inhibit_rules:
  - source_matchers: ['severity = "page"']
    target_matchers: ['severity = "ticket"']
    equal: ['slo']
```

`amtool silence add slo=shop-api-availability --duration=2h --comment="DB maintenance CHG-123" --alertmanager.url=http://localhost:9093`. `amtool silence query`, `amtool silence expire <id>`. `repeat_interval` too short becomes noise; too long and a forgotten alert goes unnoticed.

</details>

**Bonus:** Add a dead man's switch: route `Watchdog` to an external heartbeat service (or a cron job that alerts when no webhook arrived in 10 minutes).

---

### Assignment 14. A Grafana SLO dashboard with budget burndown 🟡

**Objective:** Build a dashboard that answers "are we OK?" in five seconds and "where is it hurting?" in one minute.

**Tasks**
1. Top row: SLI over the SLO window, error budget remaining, current burn rate (1h), for each SLO.
2. A budget burndown panel: remaining budget over the last 30 days as a line falling from 100%.
3. Burn rate per window (5m, 1h, 6h, 1d) with the 14.4x and 6x page thresholds drawn as lines.
4. Diagnosis row: requests by route and code, latency p50/p99 with the SLO threshold, dependency latency.
5. Keep the dashboard as JSON in Git and deploy it through provisioning or the Grafana sidecar, with a `datasource` variable instead of a hard-coded UID.

**Expected outcome / acceptance criteria**
- A deleted dashboard reappears unchanged after re-applying from Git.
- A newcomer can tell you within five seconds whether the SLO is currently met.
- The burndown panel shows the step change caused by an injected outage.

<details>
<summary>Hints</summary>

Burndown: plot `slo:error_budget_remaining:ratio30d` as a time series with min 0, max 1. Thresholds as lines: `fieldConfig.defaults.custom.thresholdsStyle.mode = "dashed"`. Sidecar: ConfigMap with label `grafana_dashboard: "1"`. Datasource reference in panels: `{"type": "prometheus", "uid": "${datasource}"}`.

</details>

**Bonus:** Add Grafana annotations from deploy events (for example a Prometheus metric set by your CD system, or Grafana's annotation API called from CI) so you can see which release burned budget.

---

### Assignment 15. Availability math for a real dependency graph 🟡

**Objective:** Compute what availability a design can actually deliver and where to invest.

**Tasks**
1. Draw the request path of a checkout service: load balancer, API (3 replicas), payment provider (external, 99.95% SLA), database (primary with synchronous replica and automatic failover), cache (optional, with fallback).
2. Assign availability numbers to each component and state your assumptions.
3. Compute end-to-end availability, treating serial hard dependencies as multiplied and redundant components as `1 - (1 - a)^n`.
4. Identify the component that limits the result and propose two design changes, recomputing after each.
5. Explain why the cache does not appear in the product if the fallback works, and what happens if the fallback overloads the database.

**Expected outcome / acceptance criteria**
- A diagram plus a calculation table.
- You show that a 99.99% target is impossible with a single 99.95% hard dependency in series.
- At least one change removes a hard dependency (async processing, queueing, graceful degradation) rather than adding redundancy.

<details>
<summary>Hints</summary>

Serial: 0.9999 × 0.9995 × 0.9999 ≈ 0.9993. Parallel replicas only help when failures are independent and failover is automatic and fast; correlated failures (same AZ, same bad deploy) break the formula. Making payment asynchronous ("order accepted, payment pending") moves it out of the synchronous path.

</details>

**Bonus:** Include failover time: if the database fails twice a month and failover takes 45 seconds, compute the availability contribution from failovers alone.

---

### Assignment 16. Load test with k6 and find the knee 🟡

**Objective:** Measure real capacity and turn SLOs into release gates.

**Tasks**
1. Write a k6 script with a `ramping-arrival-rate` scenario from 10 to 500 requests per second over 10 minutes against one replica of your service.
2. Record p50, p99, error rate and CPU per replica at each stage (Grafana or `kubectl top`).
3. Identify the knee: the rate where p99 crosses your latency SLO threshold.
4. Write a second script with `constant-arrival-rate` at 70% of the knee and thresholds matching your SLOs, using `abortOnFail` for the error rate.
5. Run it in CI (or as a Kubernetes Job) and confirm the exit code is 99 when a threshold fails.

**Expected outcome / acceptance criteria**
- A chart or table of rate vs p99 with the knee marked.
- A per-replica capacity number you would use for capacity planning.
- The CI job fails on a build where you add an artificial 300 ms delay.

<details>
<summary>Hints</summary>

```javascript
export const options = {
  scenarios: { ramp: { executor: 'ramping-arrival-rate', startRate: 10, timeUnit: '1s',
    preAllocatedVUs: 50, maxVUs: 500, stages: [{ target: 500, duration: '10m' }] } },
  thresholds: { http_req_failed: [{ threshold: 'rate<0.001', abortOnFail: true, delayAbortEval: '30s' }],
                http_req_duration: ['p(99)<250'] },
};
```

Arrival-rate executors keep injecting load when the service slows (open model); `constant-vus` does not. Watch for `Insufficient VUs` warnings: the measurement is invalid if k6 cannot keep the rate.

</details>

**Bonus:** Stream k6 metrics to Prometheus with `k6 run -o experimental-prometheus-rw` (set `K6_PROMETHEUS_RW_SERVER_URL`) and overlay them on the SLO dashboard.

---

### Assignment 17. Chaos Mesh experiments with written hypotheses 🟡

**Objective:** Run chaos as a controlled experiment and learn something you did not know.

**Tasks**
1. Install Chaos Mesh on kind (containerd runtime settings).
2. For each of these, write steady state, hypothesis, blast radius and abort condition before running it: pod-kill of one replica, 300 ms network delay to a dependency, 10% packet loss, CPU stress on one pod.
3. Run each with background load and record SLI impact from Prometheus.
4. For any hypothesis that failed, write the weakness found and a fix, implement it, and rerun.
5. Combine two faults in a Chaos Mesh `Workflow` or run one on a `Schedule`.

**Expected outcome / acceptance criteria**
- Four experiment records with hypothesis, result (held or failed) and evidence (queries, screenshots).
- At least one weakness found and fixed, proven by a rerun.
- All experiments cleaned up: `kubectl get podchaos,networkchaos,stresschaos -A` is empty.

<details>
<summary>Hints</summary>

`helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock`. Kinds: `PodChaos` (`pod-kill`, `pod-failure`, `container-kill`), `NetworkChaos` (`delay`, `loss`, `partition`, `bandwidth`), `StressChaos` (`stressors.cpu.workers`). Pause a running experiment: `kubectl annotate networkchaos NAME experiment.chaos-mesh.org/pause=true`.

</details>

**Bonus:** Write a script that watches the burn rate and deletes all chaos objects automatically when it exceeds 14.4 (a home-made stop condition).

---

### Assignment 18. Timeouts, retries with jitter and a circuit breaker in code 🟡

**Objective:** Implement the core reliability patterns and see a retry storm happen.

**Tasks**
1. Build a three-service chain A → B → C (any language). C fails 30% of requests when overloaded.
2. Add naive retries (3 attempts, no backoff) at A and at B. Load-test and measure the requests C receives per user request.
3. Replace them with: a timeout budget (each hop shorter than its caller), retries only at one layer, exponential backoff with full jitter, and a retry budget (retries at most 10% of requests).
4. Add a circuit breaker in B around calls to C (closed, open, half-open) and a fallback response.
5. Compare C's load, user-visible error rate and p99 latency before and after.

**Expected outcome / acceptance criteria**
- Naive retries at two layers amplify load up to 16x (4 attempts × 4 attempts) on C in the worst case; you measure the real factor.
- After the fixes C's load stays near 1.1x and the system recovers once C recovers.
- The breaker opens under sustained failure and closes after successful half-open probes.

<details>
<summary>Hints</summary>

Full jitter: `sleep = random(0, min(cap, base * 2**attempt))`. Retry only idempotent requests, and only on retryable errors (connection refused, 503, timeouts), never on 400s. Python: `tenacity` (`wait_random_exponential`), Go: `cenkalti/backoff`, Java: Resilience4j, .NET: Polly. In a mesh, Envoy supports `retry_budget` in cluster circuit-breaker thresholds.

</details>

**Bonus:** Add a bulkhead (a separate connection pool or semaphore for C) and show that a slow C no longer exhausts B's threads for other routes.

---

### Assignment 19. Canary analysis gated on an SLI with Argo Rollouts 🟡

**Objective:** Release progressively and let an SLI decide whether a release continues.

**Tasks**
1. Install Argo Rollouts on kind and convert a Deployment into a `Rollout` with canary steps 10%, 30%, 60%, 100% with pauses.
2. Write an `AnalysisTemplate` that queries Prometheus for the canary's error ratio and p99 latency.
3. Run the analysis in the background from the first step; fail on error ratio above 1% or p99 above your SLO threshold.
4. Release a good version and watch it promote; release a version with a 5% error rate and watch it abort.
5. Explain when you would use a feature flag instead of a canary.

**Expected outcome / acceptance criteria**
- The bad release aborts automatically and traffic returns to stable without human action.
- `kubectl argo rollouts get rollout NAME` shows the failed `AnalysisRun`.
- The analysis query isolates canary pods (by a label such as `rollouts-pod-template-hash` or a canary Service).

<details>
<summary>Hints</summary>

```yaml
metrics:
  - name: error-ratio
    interval: 1m
    failureLimit: 1
    successCondition: result[0] < 0.01
    provider:
      prometheus:
        address: http://prometheus-operated.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{job="shop-api",code=~"5..",rollouts_pod_template_hash="{{args.canary-hash}}"}[2m]))
          / sum(rate(http_requests_total{job="shop-api",rollouts_pod_template_hash="{{args.canary-hash}}"}[2m]))
```

Pass `valueFrom: podTemplateHashValue: Latest` as the arg. The pod label must be copied onto the series (add it as a target label in the ServiceMonitor via `podTargetLabels`). Without traffic the query returns NaN; decide whether that is pass or inconclusive.

</details>

**Bonus:** Do the same with Flagger and compare the configuration model.

---

### Assignment 20. Compute DORA metrics from your own pipeline 🟡

**Objective:** Measure delivery performance from data you already have, without buying a tool.

**Tasks**
1. Pick a repository with a deploy workflow (GitHub Actions or GitLab CI).
2. Extract deployments for the last 90 days: time, commit SHA, result.
3. Compute deployment frequency, lead time for changes (first commit or merge to deploy), change failure rate (deployments that caused a rollback, hotfix or incident) and failed deployment recovery time.
4. Decide and document how you identify a "failed change" (rollback commits, `hotfix` labels, linked incidents).
5. Build a weekly chart and write three observations and one improvement experiment.

**Expected outcome / acceptance criteria**
- A script that recomputes all four numbers from raw data.
- Definitions written down; the numbers are reproducible by someone else.
- The improvement targets a constraint (for example slow reviews or manual approvals), not the metric itself.

<details>
<summary>Hints</summary>

`gh run list --workflow deploy.yml --limit 500 --json databaseId,conclusion,createdAt,updatedAt,headSha,event`. Lead time per deploy: `git log --format=%cI -1 <first commit in the range>` compared to the deploy time. Use medians, not means. DORA metrics are for teams and trends, never individual performance.

</details>

**Bonus:** Add MTTD and MTTR from your incident tracker and plot them next to change failure rate.

---

## 🔴 Advanced

### Assignment 21. SLOs as code with Sloth and Pyrra 🔴

**Objective:** Manage SLOs declaratively and understand what generators produce.

**Tasks**
1. Write a Sloth spec for availability and latency SLOs of one service and generate Prometheus rules with the CLI.
2. Write the same SLOs as Pyrra `ServiceLevelObjective` resources and deploy Pyrra on kind.
3. Diff the generated recording and alert rules against your hand-written ones from assignments 11 and 12: windows, naming, how long windows are computed, thresholds.
4. Put the specs in Git with CI that regenerates rules and runs `promtool check rules` and your unit tests.
5. Write a one-page recommendation for which approach your team should standardize on and why.

**Expected outcome / acceptance criteria**
- Both tools produce working alerts that fire on an injected outage.
- The diff explains at least three concrete differences.
- CI fails when someone edits generated rules by hand.

<details>
<summary>Hints</summary>

Sloth: `version: "prometheus/v1"`, `service`, `slos[].objective`, `slos[].sli.events.error_query` and `total_query` using `{{.window}}`, then `sloth generate -i slos.yaml -o rules.yaml`. Sloth also ships a Kubernetes controller with a `PrometheusServiceLevel` CRD. Pyrra: `apiVersion: pyrra.dev/v1alpha1`, `kind: ServiceLevelObjective`, `spec.target: "99.9"`, `spec.window: 4w`, `spec.indicator.ratio.errors.metric` and `total.metric`. The generic exchange format is OpenSLO.

</details>

**Bonus:** Write a small converter from OpenSLO YAML to a Sloth spec.

---

### Assignment 22. A game day with stop conditions (AWS FIS or LitmusChaos) 🔴

**Objective:** Run a planned failure exercise on real infrastructure with automated safety.

**Tasks**
1. Choose a platform: AWS (EKS or EC2 with an ALB) with AWS FIS, or Kubernetes with LitmusChaos.
2. Plan the game day: scope, participants and roles, hypothesis, expected alerts, communication plan, and a go/no-go checklist.
3. Create a stop condition tied to your SLO: a CloudWatch alarm on 5xx ratio or burn rate (FIS), or a probe in the Litmus experiment that fails the run.
4. Run at least two faults: instance or node termination in one AZ, and network latency or API errors to a dependency.
5. Hold a review: hypotheses held or not, time to detect, runbook gaps, and action items.

**Expected outcome / acceptance criteria**
- A written game-day plan approved before the exercise.
- Evidence that the stop condition works: deliberately trigger it once and show the experiment stopped.
- A report with timeline, findings and owned action items.

<details>
<summary>Hints</summary>

FIS: experiment template with actions such as `aws:ec2:terminate-instances`, `aws:eks:pod-delete` or `aws:network:disrupt-connectivity`, targets selected by tags and `selectionMode: PERCENT(30)`, and `stopConditions: [{source: aws:cloudwatch:alarm, value: <alarm ARN>}]`. CLI: `aws fis create-experiment-template --cli-input-json file://tpl.json`, `aws fis start-experiment --experiment-template-id <id>`, `aws fis stop-experiment --id <id>`. Litmus: `ChaosEngine` with `pod-delete` or `pod-network-latency` and `probe` entries (`httpProbe`, `promProbe`) in `mode: Continuous`.

</details>

**Bonus:** Run the same game day unannounced to the on-call (with management approval) to test detection and response, not only the system.

---

### Assignment 23. Build a capacity model with N+2 headroom 🔴

**Objective:** Turn load-test data and forecasts into a defensible provisioning plan.

**Tasks**
1. From assignment 16, take per-replica capacity at your latency SLO (the knee minus a safety margin).
2. Collect or simulate 90 days of peak traffic and fit a growth trend; add one known launch event (for example 2x for a sale).
3. Compute replicas needed for the 6-month forecast peak, with N+2 redundancy across three availability zones (survive the loss of one AZ during a deploy).
4. Check every shared limit along the path: database connections, connection pools, NAT gateway ports, rate limits of external APIs, node counts and IP addresses per subnet.
5. Present the plan: current vs required capacity, cost, the first resource that will run out, and the trigger to re-plan.

**Expected outcome / acceptance criteria**
- A spreadsheet or script with explicit assumptions.
- The plan survives the "lose one AZ at peak during a sale" test on paper.
- At least one non-compute bottleneck identified.

<details>
<summary>Hints</summary>

Required replicas = ceil(peak rps / per-replica rps / target utilization). With three AZs and one lost, the remaining two must carry the full peak: provision at least 1.5x the base requirement. PromQL forecast: `predict_linear(sum(rate(http_requests_total[1h]))[30d:1h], 86400 * 180)`. HPA does not help if nodes, quotas or the database cannot scale.

</details>

**Bonus:** Validate the model with a load test at the forecast peak while one third of the pods are cordoned or drained.

---

### Assignment 24. Disaster recovery for a stateful service, proven by a timed restore 🔴

**Objective:** Define RTO and RPO for a database-backed service and prove them with a real restore.

**Tasks**
1. Deploy PostgreSQL on kind (CloudNativePG or a StatefulSet) with an app writing a row every second.
2. Set targets: RPO 5 minutes, RTO 30 minutes. Choose the mechanism: logical dumps, WAL archiving with point-in-time recovery to object storage (MinIO), and Velero for Kubernetes objects and volumes.
3. Simulate three disasters: accidental `DROP TABLE`, namespace deleted, cluster lost (delete the kind cluster and rebuild).
4. For each, restore with a stopwatch running and measure actual RTO and data lost (last row ID before vs after).
5. Write the DR runbook and schedule a monthly automated restore test that alerts on failure.

**Expected outcome / acceptance criteria**
- Measured RTO and RPO for all three scenarios, compared with targets.
- The full-cluster scenario is restored from object storage only, with no access to the old cluster.
- The restore test is automated and its last success time is a Prometheus metric with an alert.

<details>
<summary>Hints</summary>

Velero with MinIO: `velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.10.0 --bucket velero --secret-file ./credentials-velero --use-node-agent --use-volume-snapshots=false --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000`, then `velero backup create db --include-namespaces db --default-volumes-to-fs-backup` and `velero restore create --from-backup db`. Volume-level backups of a running database are crash-consistent at best; use database-native backups (pg_basebackup plus WAL, or CloudNativePG's `Backup` with barman) for real RPO.

</details>

**Bonus:** Measure RTO again with a warm standby in a second kind cluster and compare cost vs RTO.

---

### Assignment 25. Lead a mock SEV-1 and run the postmortem review 🔴

**Objective:** Act as Incident Commander for a realistic, multi-team failure and turn it into lasting improvements.

**Tasks**
1. With a facilitator (who knows the scenario) and at least three participants, prepare an environment like the hands-on lab with two teams' services and a shared dependency.
2. The facilitator injects a compound failure, for example a bad config release in one service plus degraded network to the database, without telling anyone.
3. You act as IC: declare severity, assign Ops and Comms leads, run 15-minute check-ins, decide on mitigation, coordinate the second team, and declare resolution.
4. Within two days, write the postmortem and run a 45-minute blameless review meeting with all participants.
5. Track action items to completion and, a month later, rerun the scenario to verify the fixes.

**Expected outcome / acceptance criteria**
- The incident log shows declaration within 5 minutes of the first page and mitigation before root cause was known.
- Status updates went out on schedule with impact and next-update time.
- The postmortem identifies more than one contributing factor and has detect, mitigate and prevent actions.
- The rerun shows a shorter time to mitigate or no user impact.

<details>
<summary>Hints</summary>

The IC's loop: what is the impact, what are we doing, who owns it, when is the next check-in. Explicitly hand off command if you need to debug. In compound failures the first fix often reveals the second problem; keep the incident open until SLIs are back within SLO. In the review, ask "what made this hard to see?" before "what broke?".

</details>

**Bonus:** Compute MTTD, MTTA, time to mitigate and MTTR from the log, and add the scenario to your team's "wheel of misfortune" for on-call training.

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
