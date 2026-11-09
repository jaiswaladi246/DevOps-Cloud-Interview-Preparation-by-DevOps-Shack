# SRE & Reliability — Scenarios & Troubleshooting

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

> 100 real-world "what would you do" scenarios and 100 troubleshooting drills with real error text, from SLO negotiations and SEV-1s to broken burn-rate alerts, k6 runs and chaos experiments · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Part 1: Scenarios (Q1–Q100)**
- [SLO Design & Error Budget Decisions](#slo-design--error-budget-decisions) — Q1–Q16
- [Alerting & On-Call](#alerting--on-call) — Q17–Q30
- [Running Incidents](#running-incidents) — Q31–Q50
- [Postmortems & Reliability Culture](#postmortems--reliability-culture) — Q51–Q60
- [Capacity & Load Testing](#capacity--load-testing) — Q61–Q71
- [Chaos Engineering & Game Days](#chaos-engineering--game-days) — Q72–Q81
- [Releases & Reliability Patterns](#releases--reliability-patterns) — Q82–Q93
- [Backup & Disaster Recovery](#backup--disaster-recovery) — Q94–Q100

**Part 2: Troubleshooting (Q101–Q200)**
- [Prometheus SLIs & Recording Rules](#prometheus-slis--recording-rules) — Q101–Q120
- [Burn-Rate Alerts & Alertmanager](#burn-rate-alerts--alertmanager) — Q121–Q137
- [Grafana SLO Dashboards](#grafana-slo-dashboards) — Q138–Q146
- [k6 Load Testing](#k6-load-testing) — Q147–Q161
- [Chaos Mesh, Litmus & AWS FIS](#chaos-mesh-litmus--aws-fis) — Q162–Q176
- [Reliability Patterns in Production](#reliability-patterns-in-production) — Q177–Q190
- [Backups, DR & Runbook Automation](#backups-dr--runbook-automation) — Q191–Q200

---

# Part 1: Scenarios

## SLO Design & Error Budget Decisions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your manager asks you to "set up SLOs" for a checkout API that has none. Where do you start? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No SLOs means no shared definition of "reliable enough". Start from user journeys and existing data, not from a number someone likes.

**What I'd do →**
1. Identify the critical user journey: add to cart, submit order, payment confirmation.
2. Find where requests are already measured: ingress controller metrics or app metrics.
   ```promql
   sum by (code) (rate(nginx_ingress_controller_requests{ingress="checkout"}[5m]))
   ```
3. Pull 4 weeks of history for availability (non-5xx ratio) and latency (fraction under 300 ms).
4. Propose an SLO slightly below current performance, for example 99.9% availability and 99% under 300 ms over 28 days.
5. Write the SLO document, error-budget policy, and get product sign-off.
6. Implement recording rules and burn-rate alerts; review after one month.

**Why →** An SLO based on measured reality is achievable and meaningful; one picked from thin air is either always red or meaningless.

**Interview tip →** Say "I start with the user journey and four weeks of data", not "I set 99.99%".

</details>

### Q2. Product insists on a 99.99% SLO for a service that has run at 99.7% for the last quarter. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The ask is a 30x reduction in failures (from about 2.2 hours down to 4.3 minutes per month). It is a roadmap decision, not a config change.

**What I'd do →**
1. Show the data: last quarter's SLI, the main sources of budget burn (deploys, a flaky dependency, one big incident).
2. Translate into time: 99.99% allows 4.32 minutes of failure per month; a single bad deploy usually takes longer than that to roll back.
3. List what 99.99% would require: multi-AZ everything, zero-downtime deploys, automated canary rollback, dependencies at 99.995%, maybe multi-region.
4. Propose a staged plan: commit to 99.9% now, fund the top three reliability fixes, re-evaluate in a quarter.
5. Ask what user problem drives the request; maybe only payment confirmation needs the higher target.

**Why →** SLOs must be achievable or they get ignored. Turning a number into concrete engineering cost lets leadership make an informed trade-off.

**Interview tip →** Convert nines into minutes and engineering cost; that ends most SLO arguments.

</details>

### Q3. The error budget for the month is exhausted on day 12, and a big feature launch is planned for day 15. What happens? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is exactly what the error-budget policy exists for. The decision should follow the policy, not whoever argues loudest.

**What I'd do →**
1. Confirm the numbers: rolling SLI, which incidents burned the budget, and whether they are fixed.
2. Apply the policy: non-critical releases freeze; reliability fixes and security patches continue.
3. If the business insists on the launch, escalate to the person named in the policy (usually VP Eng plus VP Product) for an explicit, documented exception.
4. If an exception is granted, reduce launch risk: dark launch behind a feature flag, ramp 1% to 10% to 50% with SLO checks, extra on-call coverage, rollback rehearsed.
5. Prioritize the action items from the incidents that burned the budget.

**Why →** The policy only works if it is applied consistently. Exceptions are fine when explicit and owned by leadership.

**Interview tip →** Mention the "explicit exception signed by leadership" path; it shows you understand policy is a tool, not a religion.

</details>

### Q4. You have one SLO for the whole API, and it's green, but the mobile team says login is broken for 10% of users. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An aggregate SLO is hiding a failing journey. Login is a small fraction of total requests, so even 100% login failure barely moves the global ratio.

**What I'd do →**
1. Slice the SLI by route and client:
   ```promql
   sum by (route) (rate(http_requests_total{code=~"5..",job="api"}[30m]))
   / sum by (route) (rate(http_requests_total{job="api"}[30m]))
   ```
2. Check whether login failures are 4xx (excluded from the SLI) such as 401 from a token bug.
3. Mitigate the login issue as an incident.
4. Split the SLO per critical journey: login, browse, checkout, each with its own target and alert.
5. Revisit whether specific 4xx codes on login should count as bad.

**Why →** SLOs must reflect user journeys; high-volume endpoints drown out low-volume, critical ones.

**Interview tip →** "One SLO per critical user journey" is the phrase interviewers want.

</details>

### Q5. Your service's latency SLO is "99% of requests under 500 ms", measured at the app. Users still complain about slowness. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Server-side latency excludes queueing in the load balancer, TLS, network, CDN, and client rendering. The SLI is measured too far from the user.

**What I'd do →**
1. Compare app latency with load balancer latency (ALB `TargetResponseTime` vs client-side timings).
2. Add real user monitoring (RUM) or synthetic checks from user regions.
3. Check for queueing before the app: ingress controller upstream latency, pod CPU throttling, connection limits.
4. Move the latency SLI to the edge (ingress or LB) and add a separate frontend performance SLO if needed.
5. Re-check the threshold with product: 500 ms may be too loose for the interaction.

**Why →** The closer you measure to the user, the better the SLI reflects their experience.

**Interview tip →** Mention that network delay injected with chaos tools never shows in app-side histograms; it is a nice proof that measurement point matters.

</details>

### Q6. A team wants to exclude all 5xx responses caused by a third-party payment provider from their SLO. Do you agree? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Users do not care whose fault it is; checkout failed. But the team also cannot fix the provider's code. It is a question of what the SLO should drive.

**What I'd do →**
1. Keep provider failures in the user-facing SLO, because it measures user experience and drives design decisions.
2. Add a separate dependency SLI to track the provider's own performance from our side, for vendor conversations.
3. Push design work: retries with idempotency keys, a second provider, async payment confirmation, clearer user messaging.
4. If the provider failures dominate budget burn, document it and let leadership decide on investment (a second provider) vs a lower SLO.

**Why →** Excluding failures makes the SLO look good while users suffer, and removes the incentive to design around the dependency.

**Interview tip →** "The SLO measures user happiness, not blame" is the key line.

</details>

### Q7. A low-traffic internal admin API (200 requests a day) keeps firing burn-rate pages because one failed request is 0.5% of the hour. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Request-based SLIs on tiny volumes are statistically noisy. One error can equal a 14x burn on a short window.

**What I'd do →**
1. Add synthetic probes (Blackbox Exporter every 30s) to provide a steady event stream.
2. Require a minimum number of events before alerting:
   ```promql
   (slo:sli_error:ratio_rate1h > 14.4 * 0.01) and on() (sum(increase(http_requests_total{job="admin"}[1h])) > 30)
   ```
3. Consider a time-based SLI (good minutes / total minutes) and route it to tickets, not pages.
4. Ask whether an internal admin tool needs paging at all outside business hours.

**Why →** Alerts must be statistically meaningful; paging on single requests creates fatigue without protecting users.

**Interview tip →** Show you know low-traffic SLOs need synthetic traffic or minimum-count guards.

</details>

### Q8. Leadership wants a single "company reliability score" on a dashboard. How do you build it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Averaging SLIs across services is meaningless (a 99.99% search and a 98% reporting job average to a fake number). Leadership needs a summary that still means something.

**What I'd do →**
1. Define the company's critical user journeys (5 to 10) with their SLOs.
2. Show each journey's status: SLO met or not, budget remaining, trend.
3. Summarize as "N of M journeys within SLO this month" plus the worst offenders.
4. Add incident counts by severity and customer-impacting minutes.
5. Review it monthly with service owners present, focusing on decisions (investment, freezes).

**Why →** A count of journeys meeting SLO is honest and actionable; an averaged percentage hides failures.

**Interview tip →** Refuse to average percentages across unrelated services, and offer a better summary instead.

</details>

### Q9. Two SLOs conflict: the checkout team wants faster releases; the payments team (a dependency) wants a release freeze because their budget is gone. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each team's budget governs its own changes. Checkout's releases only matter to payments if checkout changes affect payments.

**What I'd do →**
1. Check whether checkout's releases are actually burning payments' budget (new call patterns, more load, retries).
2. If not, checkout keeps shipping under its own healthy budget; payments freezes its own changes.
3. If yes, the checkout changes touching payments go behind flags with joint canary analysis.
4. Help payments with their burn: postmortem action items, capacity, timeouts.

**Why →** Error budgets are per service; coupling them blindly freezes the whole company whenever one team has a bad month.

**Interview tip →** Distinguish "who burned the budget" from "whose budget it is".

</details>

### Q10. Your SLO window is 30 days rolling, and a 3-hour outage happened 25 days ago. The team asks why the freeze is still on. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** In a rolling window, the outage counts until it ages out, 30 days after it happened.

**What I'd do →**
1. Show the burn-down chart: budget is still negative because the outage is inside the window.
2. Calculate when it ages out (5 more days) and what the SLI will be after that.
3. Check whether the policy allows lifting the freeze early when the action items are done; many policies do.
4. Use the remaining days to finish the postmortem action items.

**Why →** Rolling windows deliberately make big incidents have lasting consequences, which drives the fixes.

**Interview tip →** Explain rolling vs calendar windows with this exact example.

</details>

### Q11. A data pipeline team says "SLOs don't apply to us, we're batch". How do you help them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Batch systems have users too (analysts, downstream services); they just care about freshness and completeness, not request latency.

**What I'd do →**
1. Ask the downstream users what "late" or "wrong" means for them.
2. Define a freshness SLI: data younger than 2 hours, 99% of the time.
3. Define a completeness SLI: the daily job finishes by 06:00, 99% of days.
4. Have jobs push `pipeline_last_success_timestamp_seconds` to Pushgateway or expose it, and alert on age:
   ```promql
   time() - pipeline_last_success_timestamp_seconds{pipeline="orders-etl"} > 7200
   ```

**Why →** Every system with users can have an SLO; you pick SLI types that match what users experience.

**Interview tip →** Name freshness, completeness and correctness as batch SLI types.

</details>

### Q12. Your team has 47 SLOs across 6 services and nobody looks at them. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Too many SLOs means none are important. They have become dashboards with targets.

**What I'd do →**
1. List the critical user journeys (maybe 5) and map existing SLOs to them.
2. Keep one availability and one latency SLO per journey; demote the rest to dashboard metrics.
3. Rebuild alerting around the remaining SLOs with burn-rate rules and delete the old ones.
4. Put remaining budget for each journey in the weekly team meeting.
5. Tie the error-budget policy to these few SLOs.

**Why →** SLOs are useful when they drive decisions; a handful that everyone understands beats dozens nobody reads.

**Interview tip →** "If missing it would not change a decision, it is not an SLO."

</details>

### Q13. A customer's contract has an SLA of 99.95%, but your internal SLO is 99.9%. Is that a problem? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Yes. The SLA is stricter than the SLO, so you could be within internal targets and still owe credits.

**What I'd do →**
1. Raise the internal SLO for the affected journey to at least 99.95% (ideally 99.97% for margin), if the system can deliver it.
2. If it cannot, involve sales and legal: the contract promised more than engineering can deliver.
3. Add a process: sales cannot sign SLAs stricter than published SLOs without engineering review.

**Why →** The SLO must be stricter than the SLA so internal alerts trigger before contractual penalties.

**Interview tip →** Always state the order: SLA < SLO < actual performance.

</details>

### Q14. You need to set SLOs for a service that depends on a database with a 99.95% SLA, a cache, and two internal APIs at 99.9%. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** If all are hard dependencies, the composite ceiling is about `0.9995 x 0.999 x 0.999 ≈ 99.75%` before your own failures. The cache, if handled well, should be soft.

**What I'd do →**
1. Classify each dependency as hard or soft for each journey.
2. Make the cache soft: on cache failure, read from DB with a short timeout and degrade if needed.
3. For the internal APIs, check whether calls can be async, cached or have fallbacks.
4. Compute the ceiling for what remains hard, and set the SLO below it (for example 99.5% now).
5. Plan work to remove hard dependencies and raise the SLO later.

**Why →** You cannot promise more than your critical path delivers; architecture changes raise the ceiling, not wishful targets.

**Interview tip →** Do the multiplication out loud; then show how turning a hard dependency soft changes it.

</details>

### Q15. The error budget is always 90%+ unspent. A developer asks if they can skip the canary for their release to go faster. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Healthy budget does allow more risk, but the canary is cheap insurance; the budget should buy speed in ways that do not remove safety nets entirely.

**What I'd do →**
1. Keep the canary but shorten it: fewer steps, shorter pauses, since the budget can absorb more risk.
2. Allow more frequent releases, bigger experiments, and planned infrastructure work.
3. Consider using some budget deliberately: chaos experiments, a database failover test.
4. Check whether the SLO is too loose; if users are happy at current performance, it may be fine.

**Why →** Error budget lets you trade reliability for velocity, but removing automated rollback raises incident severity without adding much speed.

**Interview tip →** Show you would spend the budget, not hoard it.

</details>

### Q16. You inherit SLO recording rules where the 30-day availability is `avg_over_time(sli_ratio_5m[30d])`. Is anything wrong? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Averaging 5-minute ratios gives each window equal weight regardless of traffic: it is a time-based SLI disguised as a request-based one. Night-time outages and peak outages count the same.

**What I'd do →**
1. Replace it with a ratio of sums over the window:
   ```promql
   sum(increase(http_requests_total{job="checkout",code=~"5.."}[30d]))
   / sum(increase(http_requests_total{job="checkout"}[30d]))
   ```
2. Put it in a rule group with a 5-minute interval to keep the cost down.
3. Compare old and new values for the last month and explain the difference to stakeholders.
4. Generate future rules with Sloth or Pyrra to avoid hand-written mistakes.

**Why →** The SLI must match the SLO definition; a request-based SLO needs request-weighted math.

**Interview tip →** Explaining why averages of ratios are wrong is a strong senior signal.

</details>

---

## Alerting & On-Call

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q17. On-call got 140 pages last week, most auto-resolved within 5 minutes. What's your plan? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flapping, short-lived pages are noise. People will start ignoring the pager, which is how real incidents get missed.

**What I'd do →**
1. Export the pages and group by alert name; usually 3 to 5 alerts cause 80% of the volume.
2. For each, ask: was anyone able to act? Did users notice?
3. Delete or demote cause-based alerts (CPU spikes, single pod restarts) to tickets or dashboards.
4. Add `for:` durations and Alertmanager `group_by`, `group_wait`, `group_interval`.
5. Replace remaining threshold alerts with SLO burn-rate alerts.
6. Set a target (under 2 pages per shift) and review weekly.

**Why →** Every page should be urgent and actionable; auto-resolving pages are neither.

**Interview tip →** Lead with "I would look at the data first" and end with a measurable target.

</details>

### Q18. A critical outage went unnoticed for 40 minutes until customers called support, yet you have 300 alert rules. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Many alerts but none on the symptom users felt. Probably cause-based alerting with gaps, or the alerts fired but were lost in noise.

**What I'd do →**
1. Check whether any alert fired; if yes, why nobody acted (routing, noise, wrong receiver).
2. If none fired, find what users experienced (errors, timeouts, blank page) and why no rule matched: maybe traffic dropped to zero (ratio becomes NaN) or failures happened before the app (LB, DNS).
3. Add SLO burn-rate alerts on the journey, a traffic-drop alert and external synthetic probes.
4. Include MTTD in the postmortem and track "detected by customer" as a metric.
5. Start pruning the 300 rules.

**Why →** Detection must be symptom-based and measured from outside as well as inside.

**Interview tip →** Mention the NaN blind spot and synthetic probes; it shows depth.

</details>

### Q19. Your team is asked to join the on-call rotation for a service built by another team who won't share it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Taking a pager without context or influence creates unhappy on-call and slow incidents. SRE takeover should follow a readiness review.

**What I'd do →**
1. Propose a production readiness review: SLOs, dashboards, runbooks, alerts, deploy safety, known issues.
2. Agree the gaps to fix before handover, with owners and dates.
3. Shadow the owning team's on-call for a few weeks.
4. Agree on hand-back conditions: if pages exceed a threshold or action items stall, the pager returns.
5. Keep the owning team as escalation for code-level issues.

**Why →** On-call ownership without knowledge or authority leads to long MTTR and burnout.

**Interview tip →** Say "PRR first, with a hand-back clause"; it is the Google SRE engagement model.

</details>

### Q20. An engineer silenced a noisy alert for 30 days and an incident it would have caught happened during the silence. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The engineer acted reasonably against a noisy alert. The system allowed long silences without review and kept a noisy alert alive.

**What I'd do →**
1. Treat it blamelessly in the postmortem: the noise was the real problem.
2. Fix or replace the alert so it is not noisy (burn rate, better threshold).
3. Policy: silences longer than 24 hours need a ticket link and team-lead approval; post active silences in the handoff.
4. Alert on long-running silences, using Alertmanager's API in a small script or the paging tool.

**Why →** Silences are a symptom of alert quality problems; the fix is better alerts plus visibility of silences.

**Interview tip →** Never blame the person; blame the alert and the missing guardrail.

</details>

### Q21. How would you design alerting for a new microservice from scratch? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start minimal and symptom-based. It is easier to add alerts than to remove noisy ones.

**What I'd do →**
1. Define SLOs (availability, latency) per journey with the product owner.
2. Write recording rules for all windows and four burn-rate alerts (two page, two ticket).
3. Add a small set of predictive cause alerts with clear deadlines: certificate expiry, disk full in 4 hours, DLQ growing.
4. Add a traffic-drop alert and an `absent()` alert for the metrics.
5. Write a runbook for each alert before enabling it.
6. Route via Alertmanager by severity, and review alert quality after the first month.

**Why →** A small number of meaningful alerts protects users and on-call health.

**Interview tip →** Mention "runbook before alert"; it is a strong quality gate.

</details>

### Q22. The secondary on-call is never paged, and when the primary missed a page last month, nobody responded for 50 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The escalation policy is broken or misconfigured, and nobody tested it.

**What I'd do →**
1. Check the escalation policy in the paging tool: escalation timeout, secondary schedule, notification rules.
2. Verify each person's contact methods (phone, push, SMS) and that do-not-disturb overrides are set.
3. Run a monthly test page through the full escalation chain.
4. Add MTTA tracking and alert if a page is unacknowledged for more than 10 minutes.

**Why →** Escalation is a reliability system too; untested, it fails silently.

**Interview tip →** "Test the pager like you test backups" is a good line.

</details>

### Q23. Developers complain that SRE alerts are written in PromQL they don't understand, so they ignore them. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alerts are not useful if responders cannot interpret them.

**What I'd do →**
1. Improve annotations: plain summary, current value, impact, runbook and dashboard links.
2. Make the SLO dashboard the first click from every alert.
3. Run a short training: what burn rate means, how to read the dashboard.
4. Use consistent alert names (`<Service><SLO>BurnFast`).
5. Let developers own and edit alert rules in their repo with templates.

**Why →** Alerts are user interfaces for humans under stress; clarity reduces MTTR.

**Interview tip →** Talk about alert annotations as UX.

</details>

### Q24. A Kubernetes node goes NotReady every few nights, it pages, and by the time someone logs in it has recovered. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A cause-based page for something the platform self-heals. It needs fixing, but it should not wake people.

**What I'd do →**
1. Check whether users were affected (SLO burn during those windows). If not, downgrade to a ticket.
2. Investigate the node: kubelet logs, kernel messages, memory pressure, disk pressure.
   ```bash
   kubectl describe node <node> | grep -A8 Conditions
   journalctl -u kubelet --since "2 days ago" | grep -i -E "pleg|eviction|oom"
   ```
3. Ensure workloads tolerate node loss: PDBs, topology spread, enough replicas.
4. Fix the root cause or replace the node group (AMI, instance type).

**Why →** Page on user impact; infrastructure self-healing events are tickets.

**Interview tip →** Separate "fix the node" (ticket) from "page a human" (only if users hurt).

</details>

### Q25. You're asked to add an alert for "CPU above 80% on any pod". How do you respond? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** High CPU is a cause, not a symptom. A pod at 85% CPU serving requests fast is fine.

**What I'd do →**
1. Ask what problem they want to catch: slowness, saturation, capacity.
2. If slowness: that is covered by the latency SLO burn alert.
3. If capacity: create a ticket-level alert on sustained saturation across the deployment or HPA at `maxReplicas`.
   ```promql
   kube_horizontalpodautoscaler_status_current_replicas == kube_horizontalpodautoscaler_spec_max_replicas
   ```
4. Add CPU to the dashboard for diagnosis, and consider CPU throttling metrics.

**Why →** Cause-based alerts multiply noise; symptom and capacity alerts are actionable.

**Interview tip →** Ask "what decision would this alert trigger?"

</details>

### Q26. Your Alertmanager sends 30 separate Slack messages when one database goes down, one per dependent service. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The real cause triggers a cascade of symptom alerts. Grouping and inhibition can collapse them.

**What I'd do →**
1. Add an inhibition rule: when `DatabaseDown` fires for a cluster, inhibit dependent service alerts with the same `cluster` or `db` label.
   ```yaml
   inhibit_rules:
     - source_matchers: ['alertname="PostgresDown"']
       target_matchers: ['severity=~"page|ticket"', 'depends_on="orders-db"']
       equal: [cluster]
   ```
2. Add a `depends_on` label to service alerts.
3. Group notifications by `cluster` and `alertname` with a reasonable `group_wait` (30s).
4. Still keep user-facing SLO alerts visible to the incident channel; inhibit only the duplicates.

**Why →** On-call should see the cause first, not hunt for it among 30 notifications.

**Interview tip →** Know the three tools: grouping, inhibition, silencing.

</details>

### Q27. An on-call engineer is burning out: four night pages a week for months. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sustained night pages are a reliability and people problem; nobody does good work after weeks of broken sleep.

**What I'd do →**
1. Immediately: give recovery time, rebalance the rotation, add people or follow-the-sun if possible.
2. Analyze the night pages: which alerts, whether actionable, whether automatable.
3. Fix or demote non-urgent ones to business-hours tickets.
4. Make reliability work from those pages top priority, using the toil cap or error-budget policy.
5. Track out-of-hours pages per person as an on-call health metric reviewed by management.

**Why →** On-call load is a leading indicator of attrition and of incidents caused by tired responders.

**Interview tip →** Show you care about humans and data at the same time.

</details>

### Q28. You need to page different teams for the same alert depending on which namespace the failing service runs in. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ownership-based routing. Labels on alerts must carry the owner.

**What I'd do →**
1. Add a `team` label to workloads and propagate it into metrics (relabeling in ServiceMonitors, or `kube_namespace_labels` join).
2. Route in Alertmanager by `team`:
   ```yaml
   routes:
     - matchers: ['team="payments"']
       receiver: payments-pagerduty
     - matchers: ['team="search"']
       receiver: search-pagerduty
   ```
3. Add a catch-all route that notifies platform SRE for alerts missing a `team` label.
4. Validate with `amtool config routes test team=payments severity=page`.

**Why →** Pages must reach the owner directly; routing through a central team adds delay.

**Interview tip →** Mention the catch-all route for unlabeled alerts.

</details>

### Q29. You want to reduce MTTD for a payment service from ~20 minutes to under 5 minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detection time comes from alert windows, `for:` durations, scrape and eval intervals, and coverage gaps.

**What I'd do →**
1. Measure the parts: scrape interval, rule evaluation interval, `for:` duration, Alertmanager `group_wait`, paging delay.
2. Ensure the fast-burn alert (1h/5m at 14.4x) exists; for large outages it triggers in about a minute.
3. Reduce `for:` to 1 to 2 minutes on burn alerts and `group_wait` to 10 to 30 seconds for pages.
4. Add synthetic checks every 30 seconds from multiple regions for the payment journey.
5. Add a traffic-drop alert and business-metric alerts (payments per minute vs last week).
6. Replay past incidents against the new rules with `promtool test rules` to verify detection time.

**Why →** MTTD is a pipeline; you shorten it by measuring each stage and closing coverage gaps.

**Interview tip →** Break MTTD into its components; it shows you have actually tuned alerting.

</details>

### Q30. Product wants an alert every time a single customer gets a 500 error. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-error alerts do not scale and are not actionable; some error rate is expected and budgeted.

**What I'd do →**
1. Explain the error budget: a 99.9% SLO expects around 1 failure per 1,000 requests.
2. Offer alternatives: SLO burn alerts for the service, per-tier alerts for key customers if contractual, and error tracking (Sentry) for developers to triage individual errors.
3. Provide a dashboard of errors by customer for support.

**Why →** Alerts are for patterns that threaten users at scale; individual errors belong in error tracking.

**Interview tip →** Redirect, do not refuse: offer the right tool for the real need.

</details>

---

## Running Incidents

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q31. You're paged at 2 AM: CheckoutErrorBudgetBurnFast. Walk through your first 10 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A fast burn means significant user impact right now. Priority: confirm, assess scope, mitigate.

**What I'd do →**
1. Acknowledge the page so it does not escalate.
2. Open the SLO dashboard from the alert: current error ratio, when it started, which routes and regions.
3. Check recent changes: deploys, config, flags, infrastructure.
   ```bash
   kubectl rollout history deploy/checkout -n shop
   kubectl get events -n shop --sort-by=.lastTimestamp | tail -20
   ```
4. If impact is significant, declare an incident (SEV-2), open the channel, post the first update.
5. If a deploy lines up with the start, roll it back immediately.
6. If not obvious, call in help and assign an IC so you can focus on debugging.

**Why →** Acknowledge, assess, communicate, mitigate: in that order, fast.

**Interview tip →** Say "recent changes first" and "rollback before root cause".

</details>

### Q32. During an incident, three senior engineers are all debugging in different directions and nobody is updating stakeholders. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No incident command. Effort is duplicated and communication has stopped.

**What I'd do →**
1. Step in as IC (or ask someone to), and say it explicitly in the channel: "I'm IC."
2. Assign an Ops lead to coordinate debugging, a Comms lead for updates.
3. Ask for hypotheses, assign one per person, set a check-in time (10 minutes).
4. Start a timeline and post the first stakeholder update.
5. Keep side conversations out of the main channel (threads or a separate debug channel).

**Why →** Structure makes parallel work effective; without it, smart people collide.

**Interview tip →** Show you would take command calmly and delegate rather than jump into the terminal.

</details>

### Q33. Error rate spikes 5 minutes after a deploy, but the developer insists their change can't be the cause. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Correlation with a deploy is the strongest signal you usually have. Rollback is cheap; arguing is expensive.

**What I'd do →**
1. Roll back now: `kubectl rollout undo deploy/checkout -n shop` or `kubectl argo rollouts abort checkout`.
2. Watch the error ratio for 5 minutes.
3. If errors drop, the deploy was involved; if not, you lost a few minutes and removed one hypothesis.
4. Investigate the change afterwards without pressure.

**Why →** Mitigation is about restoring service, not proving guilt.

**Interview tip →** "Roll back first, debate later" is the expected answer.

</details>

### Q34. Latency doubled across all services at the same time, with no deploys. Where do you look? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Everything slowing at once points to a shared dependency: database, DNS, network, a node pool, the service mesh, or a cloud provider issue.

**What I'd do →**
1. Check shared components: DB latency and connections, CoreDNS latency and errors, node CPU steal and throttling.
   ```bash
   kubectl -n kube-system top pods -l k8s-app=kube-dns
   kubectl top nodes
   ```
2. Look at traces for a common slow span.
3. Check the cloud provider status page and AWS Health Dashboard.
4. Check traffic: a sudden spike or a bot.
5. Mitigate by scaling the bottleneck, failing over, or shedding non-critical traffic.

**Why →** Correlated symptoms usually mean a common cause; look at what is shared before what is different.

**Interview tip →** Mention DNS; it is a classic hidden shared dependency.

</details>

### Q35. A SEV-1 has been running for 3 hours. People are tired and making mistakes. What do you do as IC? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long incidents degrade decision-making. Fatigue causes secondary incidents.

**What I'd do →**
1. Plan a handover: bring in a fresh IC and fresh responders, with a written status summary.
2. Enforce breaks: rotate people off for food and rest.
3. Slow down risky actions: require a second person to review commands in production.
4. Re-evaluate the strategy: is there a bigger mitigation (failover, full rollback, maintenance mode) we have been avoiding?
5. Keep updates going to stakeholders on cadence.

**Why →** People are the most important component in long incidents; managing their capacity is the IC's job.

**Interview tip →** Mention formal handover with a written summary.

</details>

### Q36. Halfway through an incident, you discover the "fix" someone applied made it worse. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Uncoordinated changes are a common incident amplifier.

**What I'd do →**
1. Revert the change immediately if it is safe to.
2. As IC, declare a rule: no production changes without announcing them in the channel and getting IC acknowledgement.
3. Record the change and revert in the timeline.
4. In the postmortem, treat it blamelessly: why did it seem like a good idea, what information was missing?

**Why →** Change control during incidents prevents compounding failures.

**Interview tip →** "All changes announced in the channel before executing" is a concrete practice to name.

</details>

### Q37. Customers report errors, but all your dashboards are green. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Either the failures happen outside what you measure (DNS, CDN, client, a region), or your SLIs exclude them.

**What I'd do →**
1. Get specifics from support: region, error message, time, customer, client version.
2. Reproduce from outside: `curl -v` from another network, check DNS resolution and TLS.
3. Check CDN and WAF logs (blocked requests), load balancer 5xx (ELB-generated, never reached targets), and 4xx spikes.
4. Treat it as an incident if it is real; don't wait for dashboards.
5. Afterwards, add synthetic checks and edge-level SLIs to close the gap.

**Why →** Customer reports are a signal too; missing telemetry is a finding.

**Interview tip →** Never answer "our dashboards say it's fine"; investigate from the user's side.

</details>

### Q38. The incident is mitigated by a rollback. The developer wants to redeploy the fix in 20 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rushing a fix right after an incident is a classic way to cause a second one.

**What I'd do →**
1. Confirm the SLI is stable and the incident is resolved.
2. Require the fix to go through the normal pipeline with tests that reproduce the bug.
3. Deploy with a slower canary and close monitoring.
4. Only if the rolled-back version has its own serious problem (security, data issue) should the fix be expedited, with extra review.

**Why →** After mitigation, urgency drops; normal safety processes should come back.

**Interview tip →** "Once mitigated, we're no longer in a hurry" is a good line.

</details>

### Q39. A database failover happened automatically and writes failed for 4 minutes. Is this an incident? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Users saw failed writes, so yes, at least a minor incident, even though automation worked.

**What I'd do →**
1. Check the impact: error budget consumed, number of failed writes, any data loss.
2. Check why failover took 4 minutes: detection, DNS TTL, connection pool refresh in the apps.
3. Look for improvements: lower DNS TTL, apps using RDS Proxy or a driver that handles failover, retries with idempotency on writes.
4. Write a light postmortem if it crosses your threshold.

**Why →** Successful automation can still hurt users; improving it shrinks future impact.

**Interview tip →** Separate "the system recovered" from "users were unaffected".

</details>

### Q40. You need to communicate an ongoing SEV-1 to executives who keep asking for an ETA. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Executives need impact and confidence, not technical detail. Fake ETAs destroy trust.

**What I'd do →**
1. The Comms lead sends updates on a fixed cadence (every 30 minutes).
2. Each update: impact (who, how many), what we know, what we are doing, next update time.
3. Give ETAs only when a mitigation is in progress with a known duration ("failover is running, expected 15 minutes").
4. Keep executives out of the technical channel; give them a separate one.

**Why →** Predictable communication reduces interruptions to responders and builds trust.

**Interview tip →** "Next update time instead of a fake ETA."

</details>

### Q41. An incident involves your service and two other teams' services. Each team thinks it's the other's fault. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-team incidents need one IC and shared data.

**What I'd do →**
1. Establish one IC for the whole incident, not one per team.
2. Bring all teams into one channel.
3. Use traces and dependency dashboards to see where errors originate.
4. Focus on mitigation options each team can take, regardless of fault.
5. Write one joint postmortem.

**Why →** Blame slows mitigation; one IC with shared data speeds it up.

**Interview tip →** "One incident, one IC, one postmortem."

</details>

### Q42. Your status page still says "All systems operational" 45 minutes into a major outage. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** External communication lagging is a common failure that damages trust more than the outage itself.

**What I'd do →**
1. Have the Comms lead post an update immediately: impact, what we are doing, next update time.
2. In the postmortem, add an action item: status page update within 10 to 15 minutes of declaring SEV-1/2, owned by the Comms role.
3. Pre-write templates for common incidents.
4. Consider automation that drafts status updates when a SEV-1 is declared.

**Why →** Customers forgive outages; they don't forgive silence.

**Interview tip →** Put status page updates inside the Comms role's checklist.

</details>

### Q43. You suspect a memory leak is causing pods to OOM every few hours, triggering errors. Mitigate and investigate. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Recurring OOMs cause request failures during restarts. Mitigation buys time; investigation finds the leak.

**What I'd do →**
1. Confirm: `kubectl get pods -n shop` shows restarts; `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled`.
2. Mitigate: increase memory limit temporarily, add replicas so restarts hurt less.
3. Correlate memory growth with the version that introduced it (deploy annotations on the memory graph).
4. Capture a heap dump or profile before the next OOM (pprof, jmap, py-spy) on a canary pod.
5. Roll back to the previous version if the leak started with a deploy.

**Why →** Mitigate user impact first, then collect evidence while it is still reproducible.

**Interview tip →** Mention capturing evidence before it disappears.

</details>

### Q44. A large customer reports data inconsistency after an incident. What's your approach? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data integrity problems can be worse than downtime, and may need careful repair and communication.

**What I'd do →**
1. Declare or extend the incident; data issues need an IC and a scribe.
2. Stop the bleeding: disable the code path or job causing inconsistent writes.
3. Scope it: which records, time range, customers. Use logs, audit tables, and backups to compare.
4. Plan the repair carefully: scripts reviewed by two people, tested on a copy, run in batches, with backups taken first.
5. Communicate with the customer through the account team with facts and timeline.
6. Postmortem focused on why invariants were not enforced or detected.

**Why →** Rushed data fixes create more corruption; scope, back up, test, then repair.

**Interview tip →** "Snapshot before repair" is the line interviewers look for.

</details>

### Q45. An incident is caused by an expired TLS certificate. How do you mitigate and prevent recurrence? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cert expiry is preventable and very common. Mitigation is quick if you have access.

**What I'd do →**
1. Mitigate: renew the certificate (cert-manager `cmctl renew`, ACM, or manual) and reload.
2. Check every service using the same cert or CA.
3. Prevent: automate issuance and renewal (cert-manager, ACM), and alert well before expiry:
   ```promql
   probe_ssl_earliest_cert_expiry - time() < 86400 * 14
   ```
4. Inventory certificates and their owners.

**Why →** Expiry is predictable; it should never be a surprise.

**Interview tip →** Name both automation (cert-manager/ACM) and expiry alerting.

</details>

### Q46. A cloud provider region has an outage. Your service is single-region. What do you do now and later? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You cannot fix the provider; you can communicate, reduce impact and learn.

**What I'd do →**
1. Declare an incident; confirm via the provider's health dashboard.
2. Communicate early: status page, customer support scripts.
3. Mitigate what you can: serve static content from the CDN, disable features that depend on the affected services, queue writes.
4. If backups exist in another region and RTO justifies it, consider restore elsewhere (usually not if the outage is expected to be short).
5. Later: postmortem with a DR decision: does the business need warm standby or multi-region? What RTO/RPO, at what cost?

**Why →** Single-region is a valid business choice until it isn't; the incident gives data to revisit it.

**Interview tip →** Do not promise multi-region by reflex; tie it to RTO/RPO and cost.

</details>

### Q47. During an incident, someone wants to restart all pods "to see if it helps". 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Restarts can mitigate some issues (stuck connections, leaks) but also destroy evidence and can make things worse (cold caches, thundering herd on dependencies).

**What I'd do →**
1. Capture evidence first: logs, `kubectl describe`, a thread dump from one pod.
2. Restart a single pod and observe whether it helps.
3. If it does, do a rolling restart, not all at once: `kubectl rollout restart deploy/checkout -n shop`.
4. Record it in the timeline.

**Why →** Controlled, evidence-preserving actions beat blind ones.

**Interview tip →** "One pod first, then rolling" shows discipline.

</details>

### Q48. A config change pushed globally broke every region at once. What does this tell you and what do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config is code and needs the same progressive rollout. Global push removed every blast-radius limit.

**What I'd do →**
1. Revert the config globally.
2. In the postmortem, identify why config bypassed progressive delivery.
3. Route config through a pipeline: validation, schema checks, canary to one region or cell, bake time, then the rest.
4. Add automated rollback based on SLO checks during config rollouts.
5. Keep the ability to make urgent global changes, but behind an explicit break-glass process.

**Why →** Many of the largest public outages came from global config pushes.

**Interview tip →** "Treat config like code: staged, validated, reversible."

</details>

### Q49. You're the only on-call engineer and the incident needs database expertise you don't have. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Escalating is part of the job, not a failure.

**What I'd do →**
1. Use the escalation path: page the database on-call or the secondary.
2. While waiting, gather context: error messages, DB metrics (connections, CPU, locks, replication lag), recent changes.
3. Apply safe mitigations from the runbook (scale read replicas, kill a runaway query only if the runbook says so).
4. Keep the timeline.

**Why →** Early escalation shortens MTTR; heroics extend it.

**Interview tip →** Say you would escalate early and bring data with you.

</details>

### Q50. How do you decide between failing over to another region and continuing to debug in the primary? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failover has costs (possible data loss up to RPO, risk of failover issues, failback later), but extended outages cost more.

**What I'd do →**
1. Pre-define criteria in the DR runbook: "if SEV-1 with no mitigation path in 30 minutes, fail over."
2. Consider: how confident are we in the current hypothesis? Is the failover tested recently? What data loss is expected?
3. The IC makes the call, with the Ops lead's input, and records the decision and reasoning.
4. After failover, keep debugging the primary without user pressure.

**Why →** Pre-agreed decision criteria remove hesitation during high-stress moments.

**Interview tip →** "Decide criteria in peacetime, execute in wartime."

</details>

---

## Postmortems & Reliability Culture

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. An engineer ran a script that deleted a production table. Leadership wants to know "who did it". 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blame will make people hide future mistakes. The question to answer is how the system allowed a single command to delete production data.

**What I'd do →**
1. Run a blameless postmortem focused on conditions: why could the script reach production? Were there confirmations, dry runs, separate credentials?
2. Explain to leadership that naming and blaming reduces reporting and future learning.
3. Action items: separate prod credentials, require review for destructive operations, backups with tested restore, deletion protection, point-in-time recovery.
4. Thank the engineer publicly for reporting quickly.

**Why →** Systems that rely on nobody ever making a mistake will fail; guardrails are the fix.

**Interview tip →** Talk about "how did this make sense at the time" rather than "who".

</details>

### Q52. Postmortem action items pile up and never get done. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Learning without follow-through repeats incidents.

**What I'd do →**
1. Put action items in the normal backlog with labels, owners, priorities and due dates.
2. Limit to a few high-impact items per postmortem.
3. Review open items weekly in a reliability meeting; report completion rate to leadership.
4. Use the error-budget policy to prioritize reliability items when budget is low.
5. Close items that are no longer relevant explicitly, rather than letting them rot.

**Why →** Visibility and ownership drive completion.

**Interview tip →** Mention a completion-rate metric reviewed by leadership.

</details>

### Q53. The same type of incident (connection pool exhaustion) has happened four times this year. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeat incidents mean previous action items treated symptoms (raising the pool size) instead of underlying causes.

**What I'd do →**
1. Review the four postmortems together to find common contributing factors.
2. Look deeper: slow queries holding connections, no timeouts, retry storms, unbounded concurrency.
3. Fix systemically: query timeouts, connection pooling proxy (PgBouncer, RDS Proxy), bulkheads per dependency, saturation alerts.
4. Track repeat incidents as a metric.

**Why →** Cross-incident analysis reveals patterns that single postmortems miss.

**Interview tip →** Say you would analyze incidents in aggregate, not one at a time.

</details>

### Q54. How would you write the timeline for a postmortem when the incident spanned 6 hours and multiple Slack channels? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timelines need to be reconstructed from many sources and normalized.

**What I'd do →**
1. Collect sources: incident channel, alert history (Alertmanager, PagerDuty), deploy logs, Grafana annotations, kubectl audit logs.
2. Convert everything to UTC.
3. Include key events only: start of impact, detection, acknowledgement, declaration, hypotheses, mitigations, resolution.
4. Mark durations: time to detect, time to mitigate, total impact.
5. Ask participants to review for accuracy.

**Why →** An accurate timeline reveals where time was lost and what to improve.

**Interview tip →** Mention UTC and multiple sources.

</details>

### Q55. Your company has no postmortem culture. How do you introduce it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Culture changes by example and by making it easy.

**What I'd do →**
1. Create a simple template and write the first few postmortems yourself, blameless and useful.
2. Share them widely and hold short review meetings.
3. Get a senior leader to endorse blamelessness publicly.
4. Define triggers (SEV-1/2) and track action items.
5. Celebrate good postmortems and the improvements they produce.

**Why →** People adopt practices they see working and are safe to try.

**Interview tip →** "Lead by example, then make it policy."

</details>

### Q56. A postmortem's root cause section says "human error". What do you push back on? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** "Human error" is where the analysis stopped, not a root cause.

**What I'd do →**
1. Ask why the error was possible: no validation, confusing UI, missing guardrail, time pressure, lack of training.
2. Ask why it was not caught: no review, no canary, no alert.
3. Rewrite contributing factors around systems and processes.
4. Turn them into action items (validation, automation, safer defaults).

**Why →** You cannot fix humans; you can fix the system they work in.

**Interview tip →** "Human error is the start of the investigation, not the end."

</details>

### Q57. Leadership asks you to reduce incidents by 50% next quarter. How do you respond? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** An incident count target encourages people to stop declaring incidents. Better goals focus on impact and learning.

**What I'd do →**
1. Agree on the intent: fewer user-impacting failures.
2. Propose better measures: SLOs met per journey, customer-impacting minutes, MTTD/MTTR trends, repeat incident rate, action item completion.
3. Analyze last quarter's incidents for top causes (deploys, dependencies, capacity) and fund the biggest fixes.
4. Report progress monthly.

**Why →** Goodhart's law: when a measure becomes a target, it stops being a good measure.

**Interview tip →** Redirect to impact-based metrics without dismissing the goal.

</details>

### Q58. Developers say SRE "slows them down" with reviews and gates. How do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Friction without visible value breeds resentment. SRE should make the safe path the fast path.

**What I'd do →**
1. Listen: which gates cause the most delay?
2. Automate gates: canary analysis, policy checks, SLO checks in pipelines instead of manual approvals.
3. Tie gates to error budget: lighter process when budget is healthy.
4. Provide paved paths (templates with SLOs, alerts, dashboards) so teams get reliability for free.
5. Share data showing incidents prevented.

**Why →** SRE succeeds when reliability becomes the default, not an extra step.

**Interview tip →** "Make the safe way the easy way."

</details>

### Q59. An incident was resolved but nobody knows why it stopped. What now? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** "It went away by itself" means it will come back. The cause is still present.

**What I'd do →**
1. Keep the incident in "monitoring" state and gather data while it is fresh: logs, metrics, traces around the recovery time.
2. Look for what changed at recovery: traffic drop, a cron job finishing, autoscaling, a dependency recovering.
3. Write a postmortem with open questions and add telemetry where data was missing.
4. Set up specific alerts to catch it earlier next time, and consider reproducing it in staging.

**Why →** Unexplained recoveries are learning opportunities and future risks.

**Interview tip →** Say you would treat "self-resolved" as an open investigation.

</details>

### Q60. How do you measure whether your SRE practices are working after a year? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use outcomes for users and teams, not activity counts.

**What I'd do →**
1. SLOs: percentage of journeys meeting SLO, budget burn trends.
2. Incidents: customer-impacting minutes, median MTTD and MTTR, repeat incidents.
3. Delivery: DORA metrics (deploy frequency up, change failure rate down).
4. On-call health: pages per shift, out-of-hours pages, survey scores.
5. Toil: hours per month and trend.
6. Present them with stories: which investments moved which numbers.

**Why →** Balanced metrics show whether reliability improved without sacrificing velocity or people.

**Interview tip →** Combine user, delivery and people metrics in your answer.

</details>

---

## Capacity & Load Testing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Marketing announces a campaign that will bring 5x normal traffic in three weeks. Prepare. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A known, dated spike is the best kind of capacity problem: there is time to test and prepare.

**What I'd do →**
1. Get details: expected peak RPS, which journeys, duration, start time.
2. Load test at 5x (plus margin, say 7x) with k6 arrival-rate scenarios against production-like staging or production off-peak.
3. Find bottlenecks: DB connections, third-party rate limits, autoscaler `maxReplicas`, cloud quotas, node capacity.
4. Pre-scale: raise HPA minimums before the campaign, warm caches, request quota increases, pre-warm load balancers if needed.
5. Prepare degradation: feature flags to disable expensive features, rate limits for bots.
6. Plan on-call coverage and a war room for launch time; freeze risky changes for the day.

**Why →** Autoscaling alone will not handle a 5x step change; testing and pre-scaling will.

**Interview tip →** Mention quotas and third-party limits; they are the usual surprises.

</details>

### Q62. A load test shows p99 latency jumps from 120 ms to 2 s at 800 RPS, while CPU is only 45%. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Not CPU-bound. Something else saturates: a connection pool, thread pool, locks, a downstream service, or network limits.

**What I'd do →**
1. Check pool metrics: DB connections in use vs max, HTTP client pool, worker threads.
2. Check downstream latency in traces at the same load.
3. Check CPU throttling (`container_cpu_cfs_throttled_periods_total`), not just usage.
4. Look at DB locks and slow queries during the test.
5. Fix the bottleneck (pool size, query, concurrency) and re-test.

**Why →** Utilization of one resource does not tell you capacity; the first saturated resource does.

**Interview tip →** Apply USE: check saturation of every resource on the request path.

</details>

### Q63. You're asked how many pods are needed for Black Friday. Walk through the calculation. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use measured per-pod capacity, forecast peak, headroom and failure tolerance.

**What I'd do →**
1. Forecast peak: last year's peak x growth (for example 3,000 RPS x 1.4 = 4,200 RPS).
2. Measure per-pod capacity at SLO latency with k6 (say 200 RPS at p99 < 300 ms).
3. Apply target utilization of 70%: `4200 / (200 x 0.7) = 30` pods.
4. Survive losing one of three AZs: `30 x 1.5 = 45` pods.
5. Check nodes, IPs, DB connections (`45 pods x pool size`), and dependency capacity.
6. Pre-scale before the event.

**Why →** Capacity numbers must come from measurement plus explicit headroom for failure.

**Interview tip →** Always include the AZ-loss multiplier and the DB connection check.

</details>

### Q64. Your HPA scales out during spikes, but users still see errors for the first 2 minutes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scaling is too slow for the spike: metric delay, scale-up policy, pod startup time, or node provisioning.

**What I'd do →**
1. Measure each stage: when the metric crossed the target, when the HPA scaled, when pods became Ready, whether pods were Pending waiting for nodes.
   ```bash
   kubectl describe hpa checkout -n shop
   kubectl get events -n shop --field-selector reason=FailedScheduling
   ```
2. Tune `behavior.scaleUp` (allow 100% increase per 15s, no stabilization window for scale-up).
3. Reduce startup time: smaller images, faster readiness, preloaded caches.
4. Keep headroom: higher `minReplicas` or a lower target utilization; low-priority placeholder pods to keep spare node capacity.
5. Consider scaling on RPS or queue depth (KEDA) instead of CPU.

**Why →** Reactive autoscaling has inherent delay; headroom covers the gap.

**Interview tip →** Name the stages: metric lag, HPA sync, pod start, node provisioning.

</details>

### Q65. A soak test shows memory climbing steadily over 8 hours. What's next? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic memory or resource leak; short load tests would never find it.

**What I'd do →**
1. Confirm with container memory working set, not just RSS:
   ```promql
   container_memory_working_set_bytes{namespace="shop",container="checkout"}
   ```
2. Check other resources too: open file descriptors, goroutines or threads, DB connections.
3. Take heap profiles at the start and after a few hours and compare.
4. Fix, then repeat the soak test as a release gate for that service.

**Why →** Leaks appear over time; soak tests catch them before they cause nightly OOM incidents.

**Interview tip →** Mention comparing two heap snapshots.

</details>

### Q66. The load test passed in staging, but production fell over at half that load. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Staging was not representative: data size, cache warmth, traffic mix, dependencies or infrastructure differ.

**What I'd do →**
1. Compare: DB size and indexes, instance types, replica counts, cache hit rate, third-party sandbox vs real.
2. Compare traffic mix: staging tested one endpoint; production hits expensive search queries.
3. Use production traffic patterns for the test (replay access logs or weight scenarios by real distribution).
4. Run controlled tests in production off-peak with tagging and abort criteria.

**Why →** Load tests are only as good as their realism.

**Interview tip →** Talk about traffic mix and data volume, not just RPS.

</details>

### Q67. You need to add load testing to CI for every release without making the pipeline slow. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Full load tests take too long for every commit; a layered approach balances speed and safety.

**What I'd do →**
1. Per PR: a short k6 smoke test (1 to 2 minutes) with strict thresholds against an ephemeral environment.
2. Per release candidate: a 10-minute load test at expected peak in staging.
3. Nightly or weekly: soak and stress tests.
4. Fail the pipeline on k6 exit code 99 (threshold failed); keep results as artifacts and trend them in Grafana.

**Why →** Different test types catch different problems at different costs.

**Interview tip →** Show the layered plan and the exit-code gate.

</details>

### Q68. The database is at 85% CPU at peak and growing 5% per month. Plan. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Roughly two to three months until saturation. Databases scale slowly, so act now.

**What I'd do →**
1. Find top queries by total time (`pg_stat_statements`, Performance Insights) and optimize the worst ones.
2. Add caching for read-heavy, rarely changing data.
3. Move reads to replicas where staleness is acceptable.
4. Plan a vertical scale during a maintenance window and test it.
5. Longer term: partitioning or sharding, or splitting workloads.
6. Add a capacity ticket alert at 70% sustained.

**Why →** Optimization buys time cheaply; scaling and architecture changes need lead time.

**Interview tip →** Start with query optimization before throwing hardware at it.

</details>

### Q69. Your k6 test reports 0% errors but the service's own metrics show 3% 5xx during the test. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The two are measuring different things: k6 may treat some statuses as success, retries may be hiding errors, or k6 traffic is not the only traffic.

**What I'd do →**
1. Check k6 `responseCallback`: if set to accept 5xx, `http_req_failed` will not count them.
2. Check if a gateway or service mesh retries failed requests between k6 and the service.
3. Check whether other traffic (background jobs, other clients) is causing the 5xx.
4. Tag k6 requests (`User-Agent` or a header) and filter server metrics by it.

**Why →** Client-side and server-side numbers must reconcile before you trust either.

**Interview tip →** Mention mesh retries hiding errors; it is a real production surprise.

</details>

### Q70. You need to test the system's behaviour when traffic doubles in 30 seconds. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A spike test with an arrival-rate executor.

**What I'd do →**
1. Write a k6 scenario with `ramping-arrival-rate`: steady at baseline, then jump to 2x in 30s, hold, then drop.
   ```javascript
   scenarios: { spike: { executor: 'ramping-arrival-rate', startRate: 100, timeUnit: '1s',
     preAllocatedVUs: 100, maxVUs: 500,
     stages: [{ target: 100, duration: '5m' }, { target: 200, duration: '30s' },
              { target: 200, duration: '5m' }, { target: 100, duration: '1m' }] } }
   ```
2. Watch autoscaling, latency, error rate and queue depth.
3. Compare results to SLO thresholds.

**Why →** Arrival-rate models reflect real traffic surges; VU-based tests underestimate them.

**Interview tip →** Use `ramping-arrival-rate` for spikes.

</details>

### Q71. Your capacity plan assumes linear scaling, but adding pods beyond 40 doesn't increase throughput. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A shared bottleneck limits throughput: database, lock contention, a singleton component, or a third-party limit.

**What I'd do →**
1. Identify the shared resource that saturates as pods increase: DB connections, CPU on the primary, Redis single-thread CPU, a global lock.
2. Check for coordination costs: cache invalidation fan-out, leader election.
3. Fix: connection pooling proxy, read replicas, sharding, removing global locks, async processing.
4. Update the capacity model with the real limit.

**Why →** Horizontal scaling only works until a shared resource saturates (Amdahl's law in practice).

**Interview tip →** Mention Amdahl's law or the Universal Scalability Law.

</details>

---

## Chaos Engineering & Game Days

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q72. Your company has never done chaos engineering. How do you start safely? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start small, in non-production, with clear hypotheses and observability.

**What I'd do →**
1. Ensure basics first: SLO dashboards, alerts, runbooks. Chaos without observability teaches nothing.
2. Pick a simple experiment in staging: kill one pod of a stateless service.
3. Write a hypothesis: "SLI stays above 99.9% and no alert fires."
4. Run it with the team watching, using Chaos Mesh `PodChaos` with `mode: one`.
5. Write up the results and fix any surprises.
6. Gradually expand: more fault types, then production with small blast radius and stop conditions.

**Why →** Confidence and safety grow together.

**Interview tip →** "Observability before chaos" is a strong opening.

</details>

### Q73. A pod-kill experiment caused a 30-second spike of 502s, even though there are 3 replicas. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Replicas exist, but traffic still went to the dying pod: graceful shutdown or endpoint removal is broken.

**What I'd do →**
1. Check if the app handles SIGTERM and drains connections.
2. Add a `preStop` sleep (5 to 10 seconds) so endpoint removal propagates before shutdown.
3. Check readiness probes: new pods should only receive traffic when ready.
4. Check the ingress or load balancer health check and deregistration settings (ALB `deregistration_delay`).
5. Re-run the experiment to confirm.

**Why →** This is exactly what chaos is for: revealing that redundancy is not enough without graceful handling.

**Interview tip →** Mention `preStop` and endpoint propagation delay.

</details>

### Q74. Your team wants to run chaos experiments in production. Leadership is nervous. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concerns are valid; the answer is controls and evidence.

**What I'd do →**
1. Show results from staging experiments and what was fixed.
2. Propose controls: small blast radius (one pod, one AZ, a canary), business hours, team watching, automated stop conditions (SLO burn alarms), short durations.
3. Start with experiments the system should already handle (pod kill), not unknowns.
4. Communicate schedules to stakeholders and on-call.
5. Report findings after each experiment.

**Why →** Controlled experiments are safer than discovering weaknesses during real incidents.

**Interview tip →** Frame it as reducing risk, not adding it.

</details>

### Q75. Design a game day for an AZ failure on EKS. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** AZ loss tests capacity, topology spread, stateful failover, and team response.

**What I'd do →**
1. Hypothesis: "With one AZ unavailable, checkout SLI stays above 99.5% and recovers fully within 10 minutes."
2. Prepare: verify topology spread constraints, PDBs, N+1 AZ capacity, RDS Multi-AZ, and dashboards.
3. Inject with AWS FIS: `aws:network:disrupt-connectivity` on subnets in one AZ, or stop instances tagged with that AZ; stop condition is the SLO burn CloudWatch alarm.
4. Roles: IC, observers, scribe; announce to stakeholders.
5. Observe: pod rescheduling, Pending pods, Karpenter or Cluster Autoscaler behaviour, DB failover time, SLI.
6. Restore and write up findings with action items.

**Why →** AZ failures are realistic and expose capacity and dependency assumptions.

**Interview tip →** Include the stop condition and the capacity check.

</details>

### Q76. A NetworkChaos delay experiment on a dependency caused the caller to fall over completely. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The caller lacks timeouts, bulkheads or circuit breakers, so latency propagated into resource exhaustion.

**What I'd do →**
1. Check the caller's timeout settings; set explicit connect and request timeouts.
2. Add a bulkhead (separate pool or concurrency limit for that dependency).
3. Add a circuit breaker and a fallback.
4. Re-run the experiment and compare.

**Why →** Latency is more dangerous than outright failure because callers wait instead of failing fast.

**Interview tip →** "Slow is worse than down" is a good phrase.

</details>

### Q77. How do you decide which chaos experiments to run first? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Prioritize by likelihood and impact.

**What I'd do →**
1. Review past incidents and near misses: which failure types happened?
2. List dependencies and failure modes (instance loss, latency, DNS, DB failover).
3. Score by likelihood x impact x confidence gap.
4. Start with high-likelihood, medium-impact ones the system should already handle.
5. Re-prioritize as you learn.

**Why →** Experiments should target real risks and assumptions.

**Interview tip →** Base experiments on incident history.

</details>

### Q78. You want to make chaos experiments part of the release process. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Continuous chaos catches resilience regressions introduced by new code.

**What I'd do →**
1. Codify experiments as CRDs in Git (Chaos Mesh or Litmus).
2. In the staging pipeline, after deploy: run a pod-kill and a latency experiment while k6 generates load.
3. Gate on SLO thresholds from k6 and Litmus probes (`promProbe` on error ratio).
4. Fail the pipeline if the hypothesis fails.
5. Schedule weekly production experiments with small blast radius.

**Why →** Resilience degrades silently unless it is tested continuously.

**Interview tip →** Combine k6 load with chaos and SLO gates.

</details>

### Q79. A Litmus experiment reports "Pass", but users saw errors during the run. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The probes did not measure user impact, or ran at the wrong time or frequency.

**What I'd do →**
1. Review the probes: were they checking a health endpoint instead of real traffic?
2. Use a `promProbe` on the SLI (error ratio from real or load-test traffic) with `Continuous` mode.
3. Make sure load was running during the experiment.
4. Re-run and compare.

**Why →** An experiment's verdict is only as good as its steady-state definition.

**Interview tip →** "Steady state must be defined by SLIs."

</details>

### Q80. During a game day, the failover runbook turned out to be outdated and failed. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is a successful game day: it found a problem before a real disaster.

**What I'd do →**
1. Restore the system safely.
2. Record exactly where the runbook was wrong.
3. Update the runbook and automate steps where possible.
4. Schedule a repeat game day to validate.
5. Add a "last tested" date and owner to every DR runbook.

**Why →** Untested runbooks rot; game days keep them honest.

**Interview tip →** Frame failures in game days as wins.

</details>

### Q81. How do you run chaos experiments on stateful workloads safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stateful systems risk data loss; experiments need extra protection.

**What I'd do →**
1. Take fresh backups and verify restore before the experiment.
2. Start in staging with production-like data volume.
3. Start with replica failures, then primary failover; avoid disk corruption experiments in production.
4. Monitor replication lag, data consistency checks, client errors.
5. Have a tested rollback and restore plan.

**Why →** Data loss is often irreversible, so preparation matters more than for stateless services.

**Interview tip →** "Backup and restore test before any stateful chaos."

</details>

---

## Releases & Reliability Patterns

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q82. A canary at 10% shows a 0.3% higher error rate than stable. Promote or roll back? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It depends on the SLO and statistical significance; 0.3% at 10% traffic could be noise or a real regression.

**What I'd do →**
1. Compare against the SLO: on a 99.9% SLO, an extra 0.3% error rate is a 3x burn if it reached 100% of traffic.
2. Check sample size: how many requests did the canary serve?
3. Look at which errors: new error types point to a real regression.
4. If uncertain, keep the canary at 10% longer to gather data, or roll back.
5. Make analysis automatic (Argo Rollouts or Kayenta-style comparison) so humans do not eyeball it.

**Why →** Canary analysis should compare against both stable and SLO with enough data.

**Interview tip →** Convert the difference into burn rate.

</details>

### Q83. A downstream service is slow, and your service's retries are making it worse. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retry amplification: retries add load to an already struggling dependency.

**What I'd do →**
1. Immediately: reduce or disable retries via config or feature flag.
2. Add exponential backoff with jitter and a max of 2 attempts.
3. Add a retry budget (for example, retries at most 10% of requests).
4. Add a circuit breaker to stop calling when error rate is high.
5. Coordinate with the downstream team to add load shedding.

**Why →** Retries help for transient errors, but hurt during overload.

**Interview tip →** Mention retry budgets and circuit breakers together.

</details>

### Q84. You need to roll out a risky database schema change with zero downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Schema changes are hard to roll back. Use expand/contract and backward-compatible steps.

**What I'd do →**
1. Expand: add new columns or tables without removing old ones; use online DDL tools (`pg_repack`, `gh-ost` for MySQL) for large tables.
2. Deploy code that writes to both and reads the old.
3. Backfill data in batches with throttling.
4. Switch reads to the new schema behind a feature flag.
5. After a stable period, stop writing old columns, then drop them in a later release.
6. Test each step on a production-sized copy.

**Why →** Each step is reversible, so a bad step does not cause an outage.

**Interview tip →** "Expand, migrate, contract" with rollback at each step.

</details>

### Q85. A feature flag was turned on for 100% of users and caused errors. How do you handle it and prevent recurrence? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Flags give a fast kill switch, but going straight to 100% skipped progressive rollout.

**What I'd do →**
1. Turn the flag off immediately.
2. Confirm recovery on the SLO dashboard.
3. Postmortem: why did it go to 100% at once? Was the flag change reviewed?
4. Prevent: require progressive rollout (1%, 10%, 50%, 100%) with monitoring, audit logs for flag changes, and annotations on dashboards for flag flips.

**Why →** Flag changes are production changes and need the same safety.

**Interview tip →** "Flag flips are deploys."

</details>

### Q86. Your service calls five dependencies sequentially, and total latency is too high. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sequential calls add up latency, and each adds failure probability.

**What I'd do →**
1. Identify calls that can run in parallel and do them concurrently.
2. Cache results of slow or rarely changing calls.
3. Make non-critical calls async or optional with short timeouts and fallbacks.
4. Set per-call timeouts within the overall deadline.

**Why →** Parallelism and caching reduce latency; soft dependencies reduce failure impact.

**Interview tip →** Mention that total timeout must exceed the sum only for sequential calls.

</details>

### Q87. A third-party API you depend on has rate limits, and you're hitting them during peaks. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hitting a dependency's rate limits causes 429s and failures.

**What I'd do →**
1. Implement client-side rate limiting to stay under the limit.
2. Cache responses where possible.
3. Queue non-urgent requests and process them at a controlled rate.
4. Handle 429 with backoff and respect `Retry-After`.
5. Negotiate higher limits if needed.

**Why →** Respecting limits avoids cascading failures and keeps you in good standing with the provider.

**Interview tip →** Mention honoring `Retry-After`.

</details>

### Q88. A canary deploy passed analysis, but the full rollout caused errors at 100%. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The canary did not represent full-load behaviour: load-dependent bugs, shared resource limits, or traffic not split evenly.

**What I'd do →**
1. Roll back.
2. Investigate load-dependent effects: connection limits, cache stampedes, DB load from a new query multiplied by all pods.
3. Check traffic routing: was the canary getting representative traffic (all regions, all user types)?
4. Add more steps (25%, 50%) with longer pauses and analysis on resource metrics (DB load), not just error rate.

**Why →** Some failures only appear at scale; more granular steps catch them earlier.

**Interview tip →** Mention load-dependent regressions like a new query's DB cost.

</details>

### Q89. You need to implement graceful degradation for an e-commerce product page. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Identify what is essential (product info, price, add to cart) vs optional (recommendations, reviews).

**What I'd do →**
1. Classify dependencies as hard or soft.
2. For soft ones, set short timeouts and render the page without them on failure.
3. For prices, serve cached values with a freshness limit if the pricing service is down.
4. Add feature flags to disable expensive components under load.
5. Test with chaos experiments.

**Why →** Users can still buy even if some features are missing.

**Interview tip →** Talk about hard vs soft dependencies.

</details>

### Q90. Your circuit breaker is opening too often, causing unnecessary failures. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Thresholds may be too sensitive or evaluated on too few requests.

**What I'd do →**
1. Check config: failure threshold, minimum request volume, window size, open duration.
2. Increase the minimum number of calls before evaluating (for example 20).
3. Exclude non-transient errors (4xx) from failure counts.
4. Tune the half-open state to test recovery gradually.
5. Monitor breaker state transitions as metrics.

**Why →** Breakers should open on real problems, not noise.

**Interview tip →** Mention minimum request volume.

</details>

### Q91. A service has no rate limiting, and a buggy client is sending 10x normal traffic. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One client can overwhelm the service and affect everyone.

**What I'd do →**
1. Identify the client (API key, IP, user agent) from logs.
2. Block or throttle it at the edge (WAF, ingress, API gateway).
3. Implement per-client rate limiting with 429 responses.
4. Contact the client owner.
5. Add alerts for per-client traffic anomalies.

**Why →** Rate limiting enforces fairness and protects shared capacity.

**Interview tip →** Mention edge-level blocking as immediate mitigation.

</details>

### Q92. How do you design timeouts across a 3-tier call chain (gateway, API, DB)? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timeouts must nest: each caller's timeout greater than the callee's timeout plus retries.

**What I'd do →**
1. Start from the user-facing budget: gateway timeout 10s.
2. API timeout to DB: query timeout 2s, with at most 1 retry, so worst case about 4s plus backoff.
3. API request timeout: 8s, less than gateway's 10s.
4. Propagate deadlines so downstream services stop work when the caller has given up.
5. Measure p99.9 latency of each hop to make sure timeouts are realistic.

**Why →** Mis-nested timeouts cause wasted work and retries after the user has gone.

**Interview tip →** Draw the nested budget out loud.

</details>

### Q93. Developers want automatic retries on all HTTP calls in the shared client library. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blanket retries risk duplicate writes and retry storms.

**What I'd do →**
1. Retry only idempotent methods (GET, PUT, DELETE) or requests with idempotency keys.
2. Retry only transient failures (connection errors, 503, timeouts).
3. Exponential backoff with jitter, max 2 to 3 attempts.
4. Global retry budget per client.
5. Make retries configurable per call and expose metrics on retry count.

**Why →** Safe defaults in shared libraries prevent organization-wide incidents.

**Interview tip →** Mention idempotency and retry budgets.

</details>

---

## Backup & Disaster Recovery

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q94. You discover nobody has tested a database restore in two years. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** You have unverified backups, which might not work when needed.

**What I'd do →**
1. Immediately run a restore test to a separate environment.
2. Measure restore time and check data integrity.
3. Fix any issues found.
4. Automate regular restore tests (weekly or monthly) with alerts on failure.
5. Document the procedure in a runbook.

**Why →** Backups are only as good as your ability to restore them.

**Interview tip →** "Untested backups are hopes."

</details>

### Q95. The business asks for RPO 0 and RTO 5 minutes for every service. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** This is extremely expensive for every service; tiering is needed.

**What I'd do →**
1. Classify services by business criticality.
2. Explain costs: RPO 0 requires synchronous replication; RTO 5 minutes requires active-active or hot standby.
3. Propose tiers: Tier 1 (payments): near-zero RPO, minutes RTO; Tier 2: RPO 15 minutes, RTO 1 hour; Tier 3: RPO 24 hours, RTO 1 day.
4. Get sign-off on tiers and budget.

**Why →** DR investments should match business impact.

**Interview tip →** Tier services and attach cost to each tier.

</details>

### Q96. Ransomware encrypted your primary database and the backups in the same account. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Backups were not isolated. Recovery depends on any copies outside the blast radius.

**What I'd do →**
1. Declare a security incident; isolate affected systems and involve security and legal.
2. Look for any offsite or cross-account copies, snapshots, or replicas not affected.
3. Restore to a clean environment after confirming the attack vector is closed.
4. Postmortem: implement immutable backups (S3 Object Lock in compliance mode, AWS Backup Vault Lock), cross-account copies, least-privilege backup roles, MFA for deletions.

**Why →** Backups must be protected from the same threats as primary data.

**Interview tip →** Mention immutability and account isolation.

</details>

### Q97. Plan a DR drill for a Kubernetes-based application with a PostgreSQL database. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Validate both the stateless and stateful recovery paths, and measure RTO/RPO.

**What I'd do →**
1. Define objectives: target RTO and RPO.
2. Prepare: backups (Velero for cluster objects, pgBackRest or RDS snapshots for data), DR environment, runbooks.
3. Execute: simulate primary failure, restore the cluster via GitOps and Velero, restore the database to a point in time.
4. Validate: run smoke tests, check data consistency and the latest transaction timestamp to measure actual RPO.
5. Measure actual RTO from start to serving traffic.
6. Write a report and fix gaps.

**Why →** Drills reveal gaps in procedures and assumptions.

**Interview tip →** Measure actual RTO and RPO, not just "it worked".

</details>

### Q98. Your restore takes 6 hours but the RTO is 1 hour. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The DR approach does not meet the requirement.

**What I'd do →**
1. Profile the restore: download time, restore time, index rebuild, application startup, DNS changes.
2. Options: keep a warm standby replica (pilot light), use snapshots instead of logical dumps, parallelize restore, pre-provision infrastructure.
3. Re-test and measure.
4. If cost is too high, renegotiate RTO with the business.

**Why →** RTO must be met by design, not hope.

**Interview tip →** Break down restore time into stages.

</details>

### Q99. A developer accidentally deleted a Kubernetes namespace in production. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Workloads, Services, ConfigMaps, Secrets and PVCs are gone. Recovery depends on GitOps and backups.

**What I'd do →**
1. Declare an incident.
2. Recreate resources from Git (Argo CD or Flux re-sync).
3. Restore persistent data with Velero or from database backups (PV reclaim policy `Retain` may have kept volumes).
4. Restore Secrets from the secret manager (External Secrets Operator) or Velero.
5. Postmortem: restrict delete permissions with RBAC, use admission policies to block namespace deletion for protected namespaces, require break-glass for production deletes.

**Why →** GitOps plus backups makes recovery repeatable; RBAC prevents recurrence.

**Interview tip →** Mention `Retain` reclaim policy and admission control.

</details>

### Q100. How do you prove to auditors that your DR plan works? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auditors need evidence, not documents.

**What I'd do →**
1. Run scheduled DR drills and restore tests.
2. Record results: dates, measured RTO and RPO, issues found, fixes.
3. Keep runbooks versioned with "last tested" dates.
4. Automate restore tests with logs and alerts.
5. Present reports and evidence.

**Why →** Regular, documented testing shows the plan is real.

**Interview tip →** "Evidence over documentation."

</details>

---

# Part 2: Troubleshooting

## Prometheus SLIs & Recording Rules

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. You applied a `PrometheusRule` for your SLO, but `slo:sli_error:ratio_rate5m` returns nothing in Prometheus. 🟢

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

### Q102. `promtool check rules` fails with `could not parse expression: 1:72: parse error: unexpected right parenthesis ')'`. 🟢

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

### Q103. The availability SLI panel shows `NaN` every night between 02:00 and 05:00. 🟢

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

### Q104. Your per-route SLI query fails with `found duplicate series for the match group {route="/api/checkout"} on the right hand-side of the operation`. 🟡

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

### Q105. Prometheus logs `vector contains metrics with the same labelset after applying rule labels` for your SLI recording rule. 🔴

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

### Q106. Your latency SLI using `le="0.3"` returns an empty result, but the histogram exists. 🟡

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

### Q107. The SLI ratio is above 1.0 (or the latency "good" ratio is 1.07). 🟡

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

### Q108. Errors clearly appear in the logs, but the SLI shows zero 5xx for low-traffic routes. 🔴

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

### Q109. A Grafana panel using `rate(http_requests_total[30d])` fails with `query processing would load too many samples into memory in query execution`. 🟡

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

### Q110. `prometheus_rule_group_iterations_missed_total` keeps increasing for the SLO rule group. 🔴

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

### Q111. Your new ServiceMonitor is applied, but the service never appears under Status → Targets. 🟢

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

### Q112. The target shows DOWN with `server returned HTTP status 404 Not Found`. 🟢

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

### Q113. Scrapes fail intermittently with `context deadline exceeded`. 🟡

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

### Q114. Prometheus got OOMKilled after a developer added a `user_id` label to `http_requests_total`. 🟡

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

### Q115. After migrating from ingress-nginx to Gateway API with Envoy, the SLO dashboard is blank and no burn alerts fire. 🟡

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

### Q116. A 1m-window recording rule returns empty results intermittently. Scrape interval is 30s. 🟡

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

### Q117. Joining SLI series with `kube_pod_labels` to add a `team` label fails with `many-to-many matching not allowed: matching labels must be unique on one side`. 🔴

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

### Q118. In Thanos, the request counts used for error-budget events are exactly double what the app logs show. 🔴

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

### Q119. `promtool test rules` reports `FAILED: alertname: CheckoutErrorBudgetBurnFast, time: 10m, exp:[0: Labels:{alertname="CheckoutErrorBudgetBurnFast", severity="page"}], got:[]`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** At `eval_time: 10m` the alert is not yet firing: the input series do not have enough history for the `[1h]` window to exceed the threshold, the `for:` duration has not elapsed, or the expected labels miss labels the rule adds (like `slo`).

**Diagnose →** Evaluate the recording rules at that time in the test with `promql_expr_test`, and print what the alert produces at later times.

**Fix →** Move `eval_time` later (after the long window is above threshold plus `for:`), increase the error ratio in `input_series`, and list every label in `exp_labels`, including ones from recording rules.

**Prevent →** Keep a positive and a negative test case per alert tier, and document expected detection time in the test.

</details>

### Q120. Prometheus refuses to load the rules file: `invalid recording rule name: slo-checkout-error-ratio-5m`. 🟢

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

---

## Burn-Rate Alerts & Alertmanager

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. The burn-rate alert sits in `Pending` for the whole incident and never reaches `Firing`. 🟡

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

### Q122. After a rollback fixed the incident, the page kept firing for another 50 minutes. 🟡

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

### Q123. Checkout was completely down for 25 minutes and no burn-rate alert fired. 🔴

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

### Q124. The alert shows `Firing` in Prometheus, but Alertmanager's UI shows nothing. 🟢

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

### Q125. Alertmanager won't load the new config: `undefined receiver "pager" used in route`. 🟢

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

### Q126. Alertmanager logs `Notify for alerts failed` with `unexpected status code 404: no_service` for the Slack receiver. 🟢

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

### Q127. The same burn-rate incident re-pages the on-call every 5 minutes. 🟡

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

### Q128. A burn-rate alert flaps between firing and resolved several times per hour during a partial outage. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The error ratio hovers right around the threshold, so the short-window condition toggles. Each resolve-then-fire sends new notifications.

**Diagnose →** Graph both window ratios with the threshold line during the incident.

**Fix →** Add `keep_firing_for: 10m` to the rule (Prometheus 2.42+), which keeps the alert firing for a period after the condition clears. Make sure Alertmanager `group_interval` absorbs short resolve/fire cycles.

**Prevent →** Include hovering-around-threshold input series in `promtool` tests.

</details>

### Q129. You created a silence for a maintenance window, but the alert still pages. 🟢

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

### Q130. An inhibition rule should mute service alerts when the database is down, but both alerts arrive. 🟡

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

### Q131. Every alert is delivered twice to PagerDuty from your 3-replica Alertmanager. 🔴

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

### Q132. Your external dead man's switch fires: no Watchdog heartbeat for 10 minutes. 🟡

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

### Q133. The fast-burn alert expression `slo:sli_error:ratio_rate1h > X and slo:sli_error:ratio_rate5m > X` never returns results, even when both series exceed X. 🔴

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

### Q134. Applying your `PrometheusRule` fails with `admission webhook "prometheusrulevalidate.monitoring.coreos.com" denied the request: Rules are not valid`. 🟢

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

### Q135. The 3-day ticket alert never fires, even during a slow multi-day burn. 🔴

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

### Q136. The fast-burn page fired, yet the monthly SLO dashboard shows 99.96% against a 99.9% target. Is the alert wrong? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not wrong. Burn-rate alerts measure how fast the budget is being consumed right now, not whether the SLO is currently met. A 14.4x burn for an hour costs 2% of the monthly budget; left unchecked it would exhaust the budget in about 2 days.

**Diagnose →** Check the 1h error ratio and budget remaining side by side.

**Fix →** Nothing to fix in the alert; respond to the incident. Explain on the dashboard by showing burn rate next to budget remaining.

**Prevent →** Train on-call on burn rate vs budget; include both on the SLO dashboard.

</details>

### Q137. `amtool config routes test severity=page team=payments` returns `default-receiver` instead of `payments-pager`. 🟡

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

---

## Grafana SLO Dashboards

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q138. Your SLO dashboard ConfigMap is applied, but the dashboard never shows up in Grafana. 🟢

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

### Q139. An imported SLO dashboard shows `Datasource ${DS_PROMETHEUS} was not found` on every panel. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The JSON was exported with "Export for sharing externally", which replaces data source references with an input variable that is only resolved through the UI import flow, not through file provisioning.

**Diagnose →** Search the JSON for `${DS_PROMETHEUS}` and check the `__inputs` block.

**Fix →** Replace references with the real data source UID (`{"type":"prometheus","uid":"prometheus"}`) or define a `datasource` template variable and use `${datasource}`.

**Prevent →** Export provisioned dashboards without the external-sharing option and use a data source variable.

</details>

### Q140. The "error budget remaining" stat changes when you change the dashboard time picker from 7 days to 30 days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The query uses `$__range` or the panel's "reduce" over the selected range instead of a fixed SLO window. The budget must always be computed over the SLO window regardless of what range the viewer picks.

**Diagnose →** Inspect the panel query and the stat panel's calculation setting.

**Fix →** Query the fixed-window recording rule (`slo:sli_error:ratio_rate30d`) and set the stat to "Last" (not "Mean"). Optionally add a "relative time" override of `now-30d`.

**Prevent →** Document which panels are window-bound and which follow the time picker.

</details>

### Q141. You just created the SLO recording rules and the 30-day budget panel shows only a few hours of data. 🟡

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

### Q142. The budget remaining panel shows `9,850%` instead of `98.5%`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unit mismatch. The query already multiplies by 100 (returns `98.5`), but the panel unit is "Percent (0.0-1.0)", which multiplies by 100 again. The reverse mistake (ratio `0.985` with unit "Percent (0-100)") shows `0.985%`.

**Diagnose →** Run the query in Explore and check the raw value.

**Fix →** If the value is 0.985, use unit "Percent (0.0-1.0)" (`percentunit`). If you multiply by 100 in the query, use "Percent (0-100)".

**Prevent →** Keep ratios as 0 to 1 in recording rules and format in Grafana with `percentunit`.

</details>

### Q143. SLO panels time out with `504 Gateway Timeout` when viewing 30 days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Panels run raw `rate()` queries over many series and a long range with a small step, which is slow; a proxy (Grafana or ingress) times out first.

**Diagnose →** Use Query inspector to see query time, and check Prometheus query logs or `prometheus_engine_query_duration_seconds`.

**Fix →** Switch panels to recording rules, set a sensible `Min interval` (for example 5m for 30d views), and avoid regex selectors on high-cardinality labels.

**Prevent →** Review dashboards for raw-metric queries; SLO dashboards should read only `slo:*` series.

</details>

### Q144. Deploy annotations stopped appearing on the SLO dashboard after the CI system was migrated. 🟢

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

### Q145. The SLO dashboard's 1h burn-rate panel shows 9x, but the alert evaluated 15x at the same moment. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The panel computes `rate()` itself with `$__rate_interval` or a different window and step, while the alert uses the recorded 1h ratio. Panel step alignment and min interval also change what is displayed.

**Diagnose →** Compare the panel query with the recording rule expression, and query the recorded series directly in Explore.

**Fix →** Make the panel query the same recorded series the alert uses (`slo:sli_error:ratio_rate1h / 0.001`).

**Prevent →** Single source of truth: dashboards and alerts both read recording rules.

</details>

### Q146. After a Helm upgrade of kube-prometheus-stack, all SLO dashboards created in the Grafana UI are gone. 🟡

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

---

## k6 Load Testing

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q147. k6 prints `WARN[0030] Insufficient VUs, reached 100 active VUs and cannot initialize more  executor=constant-arrival-rate scenario=steady`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With an arrival-rate executor, each in-flight request needs a VU. If the system slows down, more VUs are needed (Little's Law: rate x latency). When `maxVUs` is reached, k6 cannot start iterations on schedule and the target rate is not achieved, which hides how bad the slowdown is.

**Diagnose →** Check `dropped_iterations` in the summary, and compute required VUs: `rate x p99 latency (s)`. At 200 RPS and 2 s latency that is 400 VUs.

**Fix →** Raise `maxVUs` (and `preAllocatedVUs` to avoid allocation cost mid-test), or treat the warning as the test result: the system cannot sustain the rate within acceptable latency.

**Prevent →** Add a threshold on `dropped_iterations` (for example `'dropped_iterations': ['count<1']`) so the test fails visibly.

</details>

### Q148. CI fails with `ERRO[0602] thresholds on metrics 'http_req_duration' have been crossed` and exit code 99. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The release candidate is slower than the latency threshold, or the test environment was degraded (noisy neighbour, cold caches, undersized staging), or the load generator itself was saturated.

**Diagnose →** Read the end-of-test summary (`p(95)`, `p(99)`, `http_req_waiting` vs `http_req_connecting`), compare with the previous successful run, and check server-side latency and CPU throttling during the test window.

**Fix →** If server-side latency regressed, block the release and profile the change. If the environment was at fault, fix it and re-run, rather than loosening the threshold.

**Prevent →** Trend k6 results per build (JSON output or Prometheus remote write) and keep test environments sized consistently.

</details>

### Q149. Every request fails with `dial tcp: lookup checkout on 10.96.0.10:53: no such host`. 🟢

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

### Q150. Many requests fail with `request timeout` after exactly 60 seconds. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** k6's default per-request timeout is 60s. The service is hanging (thread pool exhausted, waiting on a dependency without timeouts), so requests never complete.

**Diagnose →** Check server-side: active connections, thread pools, DB pool waits, downstream latency. Check if the ingress has a longer timeout than k6.

**Fix →** Fix the hang on the server (timeouts on dependencies, bulkheads). In the test, set a realistic timeout matching real clients: `http.get(url, { timeout: '10s' })`.

**Prevent →** Every dependency call in the service has an explicit timeout; test with slow dependencies using chaos.

</details>

### Q151. The test starts fine, then errors switch to `dial tcp 10.96.41.12:8080: connect: connection refused`. 🟡

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

### Q152. The k6 load generator logs `socket: too many open files`. 🟡

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

### Q153. At high rates, k6 errors with `dial tcp 10.0.3.15:443: connect: cannot assign requested address`. 🔴

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

### Q154. Latency results look terrible, but the server's histograms show p99 of 80 ms. Load generator CPU is at 100%. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The load generator is the bottleneck: k6 cannot process responses fast enough, so measured times include client-side scheduling delay.

**Diagnose →** Watch generator CPU and memory during the test (`top`, node metrics); compare `http_req_waiting` against server-side latency.

**Fix →** Give the generator more CPU, simplify the script (avoid heavy JSON parsing or large `check` functions per response, use `discardResponseBodies: true` when bodies are not needed), or distribute the test.

**Prevent →** Keep generator CPU below roughly 80% during tests and record it alongside results.

</details>

### Q155. k6 warns: `The test has generated metrics with 100001 unique time series, which is higher than the suggested limit of 100000 and could cause high memory usage`. 🟡

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

### Q156. Streaming k6 results to Prometheus fails with `remote write receiver needs to be enabled with --web.enable-remote-write-receiver`. 🟡

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

### Q157. The script fails at init with `ReferenceError: BASE_URL is not defined`. 🟢

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

### Q158. 20% of checks failed, but k6 still exited with code 0 and CI passed. 🟢

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

### Q159. The k6 Kubernetes Job fails with `The moduleSpecifier "/scripts/test.js" couldn't be found on local disk`. 🟢

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

### Q160. With `constant-vus`, the test reports p99 of 400 ms, but production users see multi-second latency at the same RPS. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Coordinated omission: in a closed model, each VU waits for its response before sending the next, so when the server slows down, k6 sends fewer requests and never measures the queueing real users experience.

**Diagnose →** Compare `http_reqs` rate during slow periods: it drops when latency rises.

**Fix →** Use an open model (`constant-arrival-rate` or `ramping-arrival-rate`) with enough `maxVUs` so the request rate is independent of response time.

**Prevent →** Make arrival-rate executors the default in the load-testing template for user-facing services.

</details>

### Q161. A Grafana k6 dashboard shows no data, although `experimental-prometheus-rw` output reports no errors. 🟡

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

---

## Chaos Mesh, Litmus & AWS FIS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q162. A `NetworkChaos` delay experiment does nothing, and chaos-daemon logs show `RTNETLINK answers: No such file or directory` / `Specified qdisc kind is unknown`. 🟡

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

### Q163. Chaos experiments fail with `dial unix /var/run/docker.sock: connect: no such file or directory` on a containerd cluster. 🟡

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

### Q164. `kubectl apply -f podchaos.yaml` fails with `Internal error occurred: failed calling webhook ... context deadline exceeded`. 🟡

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

### Q165. A `PodChaos` resource is created, but no pod is killed. The status shows no records. 🟢

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

### Q166. You deleted the `NetworkChaos`, but the 200 ms delay is still there and the resource is stuck in `Terminating`. 🔴

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

### Q167. `NetworkChaos` added 300 ms delay to the checkout pods, but the checkout latency SLI did not move. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SLI comes from the app's own histogram, which measures time inside the handler. Delay on the pod's egress packets happens after the handler finishes, so only clients see it.

**Diagnose →** Compare k6 or ingress latency (client side) with the app histogram during the experiment.

**Fix →** Measure the latency SLI at the ingress or load balancer, or inject the delay on a downstream dependency so the caller's handler waits for it.

**Prevent →** Validate that every SLI reacts to a fault you inject; an SLI that does not move during a known fault is blind to that failure mode.

</details>

### Q168. A Litmus experiment pod never starts: `error looking up service account shop/pod-delete-sa: serviceaccount "pod-delete-sa" not found`. 🟢

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

### Q169. Litmus `pod-delete` fails: `pods is forbidden: User "system:serviceaccount:shop:pod-delete-sa" cannot delete resource "pods" in API group "" in the namespace "shop"`. 🟢

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

### Q170. The Litmus `ChaosResult` verdict is `Fail`, with the `promProbe` marked failed, but the app looked healthy. 🟡

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

### Q171. Litmus experiments stay `Awaited` forever in an air-gapped cluster. 🟡

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

### Q172. An AWS FIS experiment fails immediately with a target resolution error saying the target set is empty. 🟡

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

### Q173. FIS fails with `is not authorized to perform: ec2:TerminateInstances on resource: arn:aws:ec2:ap-south-1:123456789012:instance/i-0abc...`. 🟢

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

### Q174. FIS `aws:eks:pod-delete` fails because FIS cannot create its pod in the cluster (`forbidden`). 🔴

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

### Q175. A FIS experiment with a stop condition on the SLO alarm kept running while errors rose. 🔴

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

### Q176. A Chaos Mesh `StressChaos` CPU experiment pegged the node and evicted unrelated pods. 🟡

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

---

## Reliability Patterns in Production

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q177. Clients get 503s with body `upstream connect error or disconnect/reset before headers. reset reason: overflow`, and Envoy logs show flag `UO`. 🔴

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

### Q178. NGINX ingress returns 504 and logs `upstream timed out (110: Connection timed out) while reading response header from upstream`. 🟢

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

### Q179. After a 30-second database blip, Postgres keeps rejecting connections with `FATAL: sorry, too many clients already` for 20 minutes. 🔴

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

### Q180. Customers were charged twice during an incident where the payment API timed out. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client retried a non-idempotent `POST` after a timeout, but the first request had actually succeeded on the server.

**Diagnose →** Match duplicate charges by customer and timestamp in payment logs; check retry configuration in the client library or mesh (`VirtualService` `retries` with `retryOn: 5xx,reset`).

**Fix →** Refund duplicates. Add idempotency keys: the client generates a key per logical payment, the server stores the result per key and returns it on duplicates. Remove automatic retries for non-idempotent calls at the mesh layer.

**Prevent →** Idempotency keys mandatory for all write APIs; retry policies reviewed per route.

</details>

### Q181. Every rolling deploy produces a short burst of 502s on the AWS ALB. 🟡

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

### Q182. Redis restarted and immediately the database CPU hit 100% and the site went down. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cache stampede: all cache misses at once sent the full read load to the database, which was sized only for the cache-miss rate.

**Diagnose →** Correlate cache hit ratio, DB QPS and DB CPU around the restart.

**Fix →** Shed load at the edge, then warm the cache gradually. Add request coalescing (single-flight) so concurrent misses for the same key trigger one DB query, use stale-while-revalidate, and add jitter to TTLs.

**Prevent →** Treat the cache as a hard dependency in capacity planning, or size the DB for cold-cache load; run a cache-flush chaos experiment.

</details>

### Q183. Logs fill with `CallNotPermittedException: CircuitBreaker 'inventory' is OPEN and does not permit further calls` long after inventory recovered. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The breaker cannot close: half-open trial calls keep failing (they time out because the timeout is too short for a cold dependency), `permittedNumberOfCallsInHalfOpenState` is tiny, or `waitDurationInOpenState` is very long.

**Diagnose →** Check breaker state metrics (`resilience4j_circuitbreaker_state`) and the result of half-open calls; call inventory manually from the pod.

**Fix →** Tune `waitDurationInOpenState` (for example 30s), allow several half-open calls, and make sure the call timeout is realistic for a recovering dependency.

**Prevent →** Export breaker state as metrics, alert on breakers open longer than N minutes, and test recovery in chaos experiments.

</details>

### Q184. Your NGINX rate limit returns 503 to throttled clients, and the SLO counts them as errors. 🟢

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

### Q185. The product page fails with `BulkheadFullException: Bulkhead 'recommendations' is full and does not permit further calls`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bulkhead is working as designed (recommendations is slow and its concurrency slots are full), but the caller does not have a fallback, so the exception bubbles up and fails the whole page.

**Diagnose →** Check recommendations latency and bulkhead metrics (`resilience4j_bulkhead_available_concurrent_calls`).

**Fix →** Add a fallback (empty or cached recommendations) so the page renders without that section; fix the slowness in the recommendations service.

**Prevent →** For every soft dependency, define and test the fallback, not just the isolation.

</details>

### Q186. gRPC clients see `rpc error: code = DeadlineExceeded desc = context deadline exceeded` even though the server logs show requests finishing in 200 ms. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deadline is consumed before the request reaches this server: an upstream caller set a short overall deadline and earlier hops used most of it, or queueing (client-side connection limits, server max concurrent streams) delays the start. Server logs measure only processing time.

**Diagnose →** Use distributed traces to see where time is spent across hops; check the `grpc-timeout` header received by the server and client-side queueing metrics.

**Fix →** Rebalance deadlines along the chain, parallelize calls, remove queueing (raise `MaxConcurrentStreams`, add connections), and fail fast when remaining deadline is too small.

**Prevent →** Define latency budgets per hop from the top-level SLO, and propagate deadlines everywhere.

</details>

### Q187. A node drain for maintenance hangs with `Cannot evict pod as it would violate the pod's disruption budget.` 🟢

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

### Q188. The HPA does not scale during a load test: `failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)`. 🟢

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

### Q189. Under load, pods start failing readiness with `Readiness probe failed: Get "http://10.244.1.7:8080/healthz": context deadline exceeded (Client.Timeout exceeded while awaiting headers)`, and the remaining pods collapse. 🔴

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

### Q190. p99 latency spikes every few seconds although CPU usage averages only 40% of the limit. 🟡

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

---

## Backups, DR & Runbook Automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q191. `velero backup describe shop-nightly` shows `Phase: PartiallyFailed`. 🟡

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

### Q192. After restoring a namespace into the DR cluster with Velero, PVCs stay `Pending`. 🔴

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

### Q193. `etcdctl snapshot save` fails with `Error: context deadline exceeded`. 🟡

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

### Q194. A restore test fails with `pg_restore: error: could not execute query: ERROR:  role "app_rw" does not exist`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pg_dump` captures objects and ownership but not cluster-wide roles. The target server lacks the roles referenced by `OWNER TO` and `GRANT` statements.

**Diagnose →** Check roles on the target with `\du` in `psql`.

**Fix →** Restore roles first (`pg_dumpall --roles-only` from the source, or create them via IaC), or restore with `--no-owner --no-privileges` and reapply grants.

**Prevent →** Include roles in the backup set and run automated restore tests into an empty instance, which catch exactly this.

</details>

### Q195. You restored an RDS instance to a point in time, but the application cannot connect. 🟡

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

### Q196. The nightly backup job reports success, but the backup files in S3 are 0 bytes. 🟢

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

### Q197. The backup CronJob silently stopped running. Events show `Cannot determine if job needs to be started: too many missed start time (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.` 🟡

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

### Q198. An auto-remediation job restarted the checkout deployment 412 times this week, and nobody noticed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Automation turned a recurring failure (memory leak, deadlock) into silent toil. Each restart hid the symptom, and nobody alerted on how often the remediation ran.

**Diagnose →** Count remediation runs from automation logs or metrics, and correlate with versions and memory growth.

**Fix →** Open an incident or high-priority ticket to fix the underlying bug; add a rate limit to the automation (for example at most 3 restarts per hour, then page a human).

**Prevent →** Every auto-remediation emits a metric, alerts when it runs more than expected, and creates a ticket for each run.

</details>

### Q199. A Route 53 failover to the DR region took 25 minutes against an RTO of 5 minutes. 🔴

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

### Q200. The runbook linked from a page returns 404, and the on-call engineer improvised for 30 minutes. 🟢

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
