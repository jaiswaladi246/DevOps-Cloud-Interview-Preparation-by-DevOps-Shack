# SRE & Reliability — Scenario-Based Questions

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

> Real-world SRE & Reliability situations and how to reason through them in an interview · 125 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [SLO Design & Error Budget Decisions](#slo-design--error-budget-decisions) — Q1–Q16
- [Alerting & On-Call](#alerting--on-call) — Q17–Q30
- [Running Incidents](#running-incidents) — Q31–Q50
- [Postmortems & Reliability Culture](#postmortems--reliability-culture) — Q51–Q60
- [Capacity & Load Testing](#capacity--load-testing) — Q61–Q71
- [Chaos Engineering & Game Days](#chaos-engineering--game-days) — Q72–Q81
- [Releases & Reliability Patterns](#releases--reliability-patterns) — Q82–Q93
- [Backup & Disaster Recovery](#backup--disaster-recovery) — Q94–Q100
- [Observability & Telemetry Decisions](#observability--telemetry-decisions) — Q101–Q125

---

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

## Observability & Telemetry Decisions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Your team has metrics and logs but no tracing. A request to checkout fans out to 9 services and nobody can tell where the 2 seconds go. How do you introduce tracing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Metrics tell you *that* checkout is slow, logs tell you *something* happened in one service; only traces show the critical path across nine hops. The goal is useful traces quickly, not a perfect rollout.

**What I'd do →**
1. Pick OpenTelemetry as the standard so we are not locked to a vendor backend.
2. Deploy an OpenTelemetry Collector (agent DaemonSet plus a gateway Deployment) and a backend such as Grafana Tempo or Jaeger.
3. Start with zero-code auto-instrumentation via the OpenTelemetry Operator:
   ```yaml
   apiVersion: opentelemetry.io/v1alpha1
   kind: Instrumentation
   metadata:
     name: default
     namespace: shop
   spec:
     exporter:
       endpoint: http://otel-collector.observability:4318
     propagators: [tracecontext, baggage]
     sampler:
       type: parentbased_traceidratio
       argument: "0.1"
   ```
   Then annotate deployments with `instrumentation.opentelemetry.io/inject-java: "true"` (or `-python`, `-nodejs`).
4. Instrument the checkout path first, end to end, including the ingress so trace context starts at the edge.
5. Add manual spans only where auto-instrumentation is blind (queue consumers, batch steps).
6. Link traces to metrics with exemplars and to logs by injecting `trace_id` into every log line.

**Why →** One fully traced critical journey proves value in a week; instrumenting everything shallowly proves nothing.

**Interview tip →** Say "W3C `traceparent` propagation end to end" — broken propagation is the number one reason traces look fragmented.

</details>

### Q102. The monthly observability bill doubled in two months and finance wants it cut by 40%. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Telemetry cost grows with series count, log volume and retention, usually driven by a few noisy sources. Cut waste first, signal last.

**What I'd do →**
1. Find the top series by metric name:
   ```promql
   topk(20, count by (__name__) ({__name__=~".+"}))
   ```
   and per-job sample ingestion with `scrape_samples_scraped`.
2. Find the top log producers by namespace and app (Loki: `sum by (namespace) (bytes_over_time({namespace=~".+"}[1h]))`).
3. Drop unused metrics at scrape time with `metric_relabel_configs` (`action: drop`) — histograms with 30 buckets nobody queries are classic.
4. Cut debug/info logs in production, sample repetitive success logs, keep all errors.
5. Tier retention: 15 days hot, 13 months downsampled for SLO reporting.
6. Use tail sampling for traces: keep 100% of errors and slow traces, 5% of the rest.
7. Assign telemetry cost per team with a dashboard so the incentive sits with the owner.

**Why →** Most telemetry is never queried. Removing it costs nothing in reliability; removing SLO inputs costs a lot.

**Interview tip →** Say "I never cut the SLI metrics or error logs — everything else is negotiable."

</details>

### Q103. A developer adds `customer_id` as a label to a request counter "for better debugging". You catch it in code review. What do you say? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unbounded label values create one time series per customer per route per status code. With a million customers that is millions of series and a Prometheus OOM waiting to happen.

**What I'd do →**
1. Explain the cardinality math: `routes x codes x methods x customers`.
2. Keep metric labels bounded: route template, method, status class, region.
3. Put `customer_id` in logs and as a span attribute on traces, where high cardinality is cheap.
4. If per-customer SLOs are truly needed (enterprise tenants), use a bounded `tier` or `tenant` label for the top 50 accounts and "other" for the rest.
5. Add a CI or scrape guard: `sample_limit` on the ServiceMonitor and a `promtool`/lint rule for forbidden labels.

**Why →** Metrics answer "how much and how often"; logs and traces answer "which one". Mixing them breaks the metrics system.

**Interview tip →** "Metrics for aggregates, traces and logs for identities" is a clean one-liner.

</details>

### Q104. You need to pick a trace sampling strategy for a service doing 20,000 requests per second. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keeping 100% of traces at 20k RPS is expensive and mostly useless; but naive head sampling at 1% throws away the rare errors you actually want.

**What I'd do →**
1. Use parent-based head sampling at the SDK (for example 10%) to cap overhead and keep traces complete across services.
2. Add tail sampling in an OpenTelemetry Collector gateway that sees whole traces:
   ```yaml
   processors:
     tail_sampling:
       decision_wait: 10s
       policies:
         - name: errors
           type: status_code
           status_code: {status_codes: [ERROR]}
         - name: slow
           type: latency
           latency: {threshold_ms: 1000}
         - name: baseline
           type: probabilistic
           probabilistic: {sampling_percentage: 2}
   ```
3. Route spans by trace ID to the same collector instance with the `loadbalancing` exporter, otherwise tail sampling sees partial traces.
4. Keep RED metrics generated from 100% of spans (spanmetrics connector) so sampling never affects SLO numbers.
5. Monitor dropped spans and collector memory.

**Why →** Head sampling controls cost; tail sampling controls *which* traces survive. Metrics derived before sampling keep aggregates accurate.

**Interview tip →** Mention the `loadbalancing` exporter — interviewers know tail sampling silently breaks without trace-ID affinity.

</details>

### Q105. Every team has built their own Grafana dashboards with different panels. During incidents, nobody can find anything. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dashboard sprawl slows diagnosis. Responders need a predictable path: service overview first, then drill down.

**What I'd do →**
1. Define a standard service dashboard template: SLO status and budget, RED (rate, errors, duration), saturation (CPU, memory, throttling, pool usage), dependencies, recent deploys as annotations.
2. Generate it from code (Grafonnet, Jsonnet or a Terraform Grafana provider) with the service name as a variable.
3. Provision dashboards through Git and the Grafana sidecar, not the UI.
4. Link alerts to the right dashboard via a `dashboard_url` annotation.
5. Archive dashboards with no views in 90 days.

**Why →** Consistency means a responder who has never seen a service can still read its dashboard in 30 seconds.

**Interview tip →** "Every alert links to a runbook and a dashboard" shows operational maturity.

</details>

### Q106. A service's logs are unstructured strings like `Order 123 failed for user 9 after 3 retries`. Searching them during incidents is painful. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Free-text logs force regex searches and make aggregation impossible. Structured logs turn logs into queryable data.

**What I'd do →**
1. Agree a JSON log schema: `timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`, plus event fields such as `order_id`, `retry_count`, `error_type`.
2. Switch the logger to JSON output (Logback JSON encoder, `structlog`, `pino`, `zap`).
3. Inject `trace_id` automatically from the OpenTelemetry context.
4. Query by field instead of regex, for example in Loki:
   ```logql
   {app="orders"} | json | error_type="PaymentTimeout" | retry_count >= 3
   ```
5. Keep high-cardinality fields (order IDs) as log fields, not as Loki stream labels.

**Why →** Structured logs make "how many orders failed with PaymentTimeout in the last 10 minutes" a one-line query.

**Interview tip →** Mention that in Loki, labels must stay low-cardinality while JSON fields can be anything.

</details>

### Q107. Your Prometheus is a single instance holding everything. It is at 90% memory and the team wants 13 months of retention for SLO reports. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A single Prometheus is a single point of failure, cannot scale memory forever, and is not built for year-long retention.

**What I'd do →**
1. Shard scraping by function (platform vs apps) or use functional Prometheus instances per cluster, each with short local retention (`--storage.tsdb.retention.time=15d`).
2. Run Prometheus in HA pairs with identical config and `external_labels` (`cluster`, `replica`).
3. Ship blocks or remote-write to long-term storage: Thanos (sidecar + object storage), Grafana Mimir, or a managed service (Amazon Managed Service for Prometheus).
4. Deduplicate HA replicas at query time (Thanos `--query.replica-label=replica`).
5. Enable downsampling (5m and 1h resolutions) for long ranges; keep SLO reports on recording rules so they query few series.
6. Reduce cardinality before scaling — it is the cheapest fix.

**Why →** Local Prometheus is great for alerting with short retention; object storage is cheap for history. Separating the two scales both.

**Interview tip →** Keep alerting rules evaluated close to the data (local Prometheus) so alerts work even if long-term storage is down.

</details>

### Q108. An incident happened at 03:10, but logs from the affected pods are missing for 03:05–03:20. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A gap during an incident is often caused by the incident itself: pods restarted, nodes died, or the log pipeline was overloaded at the exact moment volume spiked.

**What I'd do →**
1. Check whether the pods were restarted or evicted (container logs vanish with the container): `kubectl get events -n shop --sort-by=.lastTimestamp`.
2. Check the log agent (Fluent Bit / Promtail / OTel Collector) for backpressure, retries, or `dropped` counters.
3. Check the backend for ingestion limits (Loki `429 Too Many Requests` or rate-limit errors).
4. Recover what exists: `kubectl logs --previous`, node journal, cloud load balancer access logs.
5. Fix: filesystem buffering on the agent, larger burst limits for incident spikes, and alerts on agent drop counters.

**Why →** Observability that fails during incidents is the worst kind; the pipeline needs headroom for 10x bursts.

**Interview tip →** "Monitor the monitoring" — alert on log-agent drops and ingestion errors.

</details>

### Q109. You're asked to add synthetic monitoring for a public web app. What do you check and from where? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Synthetics catch outages when there is no real traffic (night, new regions) and test paths end to end from outside, including DNS, CDN and TLS.

**What I'd do →**
1. Pick 3–5 critical journeys: homepage, login, search, add to cart, a read-only checkout step.
2. Run probes from several regions outside our cloud (Grafana Synthetic Monitoring, Datadog Synthetics, CloudWatch Synthetics, or Blackbox Exporter in another provider).
3. Simple HTTP checks every minute; scripted browser checks every 5–10 minutes.
4. Check TLS expiry and DNS resolution along the way:
   ```promql
   probe_ssl_earliest_cert_expiry - time() < 86400 * 14
   ```
5. Alert only when probes fail from multiple locations, to avoid paging on one bad network path.
6. Use dedicated test accounts and tag synthetic traffic so it can be excluded from business metrics.

**Why →** Synthetics are a reliable "is it up for users" signal independent of our own infrastructure.

**Interview tip →** Mention multi-location quorum ("2 of 3 regions failing") to avoid false pages.

</details>

### Q110. Engineers keep asking "is it the network?" during every incident, and nobody can prove either way. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Without network-level telemetry, the network becomes the default suspect. We need data that confirms or rules it out quickly.

**What I'd do →**
1. Add client-side metrics on every outbound call: connect time, TLS time, time to first byte, errors by type (timeout, reset, refused).
2. Enable network observability: Cilium Hubble or VPC Flow Logs, plus node metrics (`node_netstat_TcpExt_TCPRetransSegs`, conntrack usage, NIC drops).
3. Deploy Blackbox Exporter probes between AZs and to key dependencies.
4. Add a "network" row to the standard dashboard: retransmits, DNS latency (CoreDNS), packet drops.
5. In incidents, compare server-side latency with client-side latency: if the server says 50 ms and the client says 2 s, the gap is in between.

**Why →** Comparing client and server views localizes delay within minutes and ends guesswork.

**Interview tip →** "Client-side minus server-side latency equals the network plus queueing" is a useful heuristic to state.

</details>

### Q111. Alerts are defined in Prometheus, Datadog, CloudWatch and Grafana, and on-call does not know where an alert came from. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fragmented alert sources mean inconsistent routing, duplicate pages and alerts that bypass review.

**What I'd do →**
1. Inventory every alert source and rule; record owner, severity and destination.
2. Route all of them through one incident platform (PagerDuty, Opsgenie, Grafana OnCall) with consistent service mapping.
3. Standardize alert labels: `service`, `team`, `severity`, `runbook_url`, and a `source` tag.
4. Manage rules as code per tool (PrometheusRule, Terraform for CloudWatch and Datadog monitors).
5. Consolidate over time: SLO burn alerts in one place, infrastructure alerts in another, nothing duplicated.

**Why →** On-call needs one inbox with consistent context, regardless of which tool detected the problem.

**Interview tip →** Saying "one pager, consistent labels, rules as code" covers most of what interviewers look for.

</details>

### Q112. A team wants to use logs to compute their SLO because they don't have metrics. Is that OK? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Logs can be a valid SLI source, but they are usually slower, costlier, and lossy under pressure. It's a stepping stone, not the goal.

**What I'd do →**
1. Accept it short term if access logs are complete (load balancer or ingress logs are good sources).
2. Derive metrics from logs instead of querying raw logs each time: Loki recording rules, CloudWatch metric filters, or an OTel `count` connector.
   ```logql
   sum(rate({app="billing"} | json | status >= 500 [5m]))
   / sum(rate({app="billing"} | json [5m]))
   ```
3. Validate the log-derived SLI against a known incident to confirm it moves when users suffer.
4. Plan native metrics (Prometheus client or OTel metrics) as the long-term source.

**Why →** Log pipelines drop data under load — exactly when the SLI matters. Metrics are cheaper and more reliable for SLOs.

**Interview tip →** Point out that log sampling or drops would make the SLO look *better* during incidents.

</details>

### Q113. During an incident, a trace shows a 1.8 s gap with no spans between the API and the database call. What do you investigate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A gap means time was spent in code or waiting on something not instrumented: a connection pool queue, a lock, GC, thread pool starvation, or DNS.

**What I'd do →**
1. Check connection pool metrics around the time (HikariCP `hikaricp_connections_pending`, `pool wait time`).
2. Check GC pauses and CPU throttling on the pod (`container_cpu_cfs_throttled_periods_total`).
3. Check thread pool/event loop metrics (executor queue depth, Node.js event loop lag).
4. Use a continuous profiler (Pyroscope, Parca) for the same time window to see where CPU or wall time went.
5. Add spans around pool acquisition and outbound DNS lookups so the gap becomes visible next time.

**Why →** Traces only show what is instrumented; gaps point at resource waiting inside the process.

**Interview tip →** Mention continuous profiling as the fourth signal next to metrics, logs and traces.

</details>

### Q114. Your Grafana and Prometheus run in the same cluster as the apps. The cluster had an outage and you were blind. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Monitoring that shares fate with what it monitors disappears exactly when needed.

**What I'd do →**
1. Move alerting and long-term storage out of the workload cluster: a dedicated monitoring cluster or a managed service.
2. Keep a lightweight local agent (Prometheus in agent mode or OTel Collector) that remote-writes out.
3. Add an external dead man's switch so silence itself pages (healthchecks.io, PagerDuty heartbeat, Grafana OnCall heartbeat).
4. Add external synthetic checks from outside the cloud provider.
5. Make sure dashboards and runbooks are reachable when SSO or the cluster is down (break-glass access).

**Why →** Monitoring needs independent failure domains to tell you about failures.

**Interview tip →** "Monitoring must not share fate with the system it watches."

</details>

### Q115. You want to correlate deploys with error spikes automatically. How do you set it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** "What changed?" is the first incident question. Changes should be visible on every graph without searching CI.

**What I'd do →**
1. Emit a deploy event from CI/CD or Argo CD to Grafana annotations:
   ```bash
   curl -s -X POST "$GRAFANA_URL/api/annotations" \
     -H "Authorization: Bearer $GRAFANA_TOKEN" -H 'Content-Type: application/json' \
     -d '{"tags":["deploy","checkout"],"text":"checkout v2.14.1 by ci"}'
   ```
2. Also expose a `deploy_info{service,version}` metric or use `kube_deployment_status_observed_generation` changes.
3. Add the version label to request metrics so errors can be split by version during rollouts.
4. Include config changes and feature-flag changes in the same event stream.
5. Show annotations on the standard service dashboard.

**Why →** Most incidents are caused by changes; visible changes cut diagnosis time dramatically.

**Interview tip →** Mention feature-flag flips as changes too — they are often missed.

</details>

### Q116. Your on-call engineers get alerts with only "HighErrorRate firing" and spend 10 minutes finding context. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alerts lacking context waste the most expensive minutes of an incident.

**What I'd do →**
1. Add annotations to every alert:
   ```yaml
   annotations:
     summary: "Checkout 5xx at {{ $value | humanizePercentage }} (SLO 99.9%)"
     description: "Burning error budget 14x. Started {{ $labels.namespace }}/{{ $labels.service }}."
     runbook_url: "https://runbooks.example.com/checkout/high-error-rate"
     dashboard_url: "https://grafana.example.com/d/checkout"
   ```
2. Template Slack and PagerDuty messages to show summary, value, links.
3. Include top labels (route, region) when available.
4. Review alert quality in on-call handoffs.

**Why →** Good annotations shorten time to mitigation and help newer engineers act confidently.

**Interview tip →** Say the alert must answer "what is broken, how bad, where to look first".

</details>

### Q117. Leadership wants "AI-powered anomaly detection" to replace your SLO alerts. How do you respond? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Anomaly detection finds "unusual", not "bad for users". It is useful as a supplement but makes a poor sole paging signal.

**What I'd do →**
1. Explain the difference: SLO burn alerts are tied to user impact and a budget; anomalies fire on harmless changes (traffic dips on holidays, new feature behaviour).
2. Propose a pilot: run anomaly detection on business metrics (orders per minute, sign-ups) as tickets or Slack-only signals.
3. Measure precision: how many anomaly alerts led to real action vs noise over 30 days.
4. Keep SLO burn alerts as the paging source.
5. Use anomaly detection where SLOs are weak: detecting silent failures like "orders dropped 40% but no errors".

**Why →** Paging must be precise and explainable; anomaly scores are hard to reason about at 3 AM.

**Interview tip →** Highlight business-metric anomalies as the best use case — they catch silent failures SLOs miss.

</details>

### Q118. A service returns HTTP 200 with an error message in the body for failures. Your availability SLI says 100%. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Status-code SLIs cannot see failures hidden in 200 responses. The SLI is lying.

**What I'd do →**
1. Short term, add an app-level metric for business outcome: `orders_total{result="success|failure"}`.
2. Change the SLI to use that outcome metric rather than HTTP code.
3. Work with the team to return correct status codes (4xx for client errors, 5xx for server failures).
4. Add a synthetic check that validates response content, not only status.
5. Check for similar patterns elsewhere (GraphQL returns 200 with `errors` array by design — handle with a dedicated metric).

**Why →** An SLI must reflect user success; status codes are only a proxy.

**Interview tip →** Mention GraphQL explicitly — interviewers love that edge case.

</details>

### Q119. Two teams use different definitions of "latency" for the same service — one at the ingress, one in the app — and their numbers disagree in reviews. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both are right about different things. The disagreement is a missing SLI specification.

**What I'd do →**
1. Write the SLI spec in the SLO doc: "latency measured at the ingress controller from request received to last byte sent".
2. Keep the app latency as a diagnostic metric, not an SLO.
3. Explain the gap (queueing, TLS, network) and graph both on the same panel.
4. Name recording rules clearly: `slo:ingress_latency_good:ratio_rate5m` vs `app:handler_duration_seconds:p99`.

**Why →** A single agreed measurement point removes recurring arguments.

**Interview tip →** "Specification vs implementation" — define what you measure, then where.

</details>

### Q120. Your Kubernetes cluster has kube-state-metrics, node-exporter and cAdvisor, but no app-level metrics. The CTO asks if you can do SLOs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Infrastructure metrics show resource state, not user experience. But the ingress or service mesh already sees every request.

**What I'd do →**
1. Use ingress controller metrics (`nginx_ingress_controller_requests`, `nginx_ingress_controller_request_duration_seconds_bucket`) or Envoy/Istio metrics (`istio_requests_total`) as the first SLI source.
2. Or use cloud load balancer metrics (ALB `HTTPCode_Target_5XX_Count`, `TargetResponseTime`).
3. Build availability and latency SLOs on those today.
4. Add app metrics later for journeys that need business outcomes.

**Why →** Edge metrics are often the best SLI source anyway: they see the user's request.

**Interview tip →** "Start with what the ingress already measures" shows pragmatism.

</details>

### Q121. An alert fires on `up == 0` for a target that's scraped through a service. Pods are healthy. You keep getting woken up. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `up` measures whether Prometheus can scrape, not whether users are affected. It is a cause-based alert paging humans.

**What I'd do →**
1. Downgrade `up == 0` alerts for single targets to ticket severity.
2. Page only on symptom-based SLO burn alerts.
3. Use `absent()` or aggregated up ratios for "whole job gone" cases:
   ```promql
   sum(up{job="checkout"}) / count(up{job="checkout"}) < 0.5
   ```
4. Fix the root cause of flapping scrapes (timeouts, scrape interval vs scrape timeout, network policies).

**Why →** One unscrapable pod among many replicas rarely affects users; paging on it trains people to ignore pages.

**Interview tip →** "Page on symptoms, ticket on causes."

</details>

### Q122. You're designing observability for an event-driven system (Kafka consumers, no HTTP). What are the SLIs? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No requests means no request-based SLIs. Users care whether events are processed correctly and on time.

**What I'd do →**
1. Freshness: age of the oldest unprocessed event, or end-to-end delay from produce timestamp to processed timestamp.
2. Correctness: ratio of successfully processed events vs those sent to a dead-letter queue.
3. Throughput/coverage: events processed vs events produced per window.
4. Measure consumer lag in time, not only offsets (Burrow, kafka-lag-exporter, or `kafka_consumergroup_lag` plus produce rate).
5. Propagate trace context in message headers so traces connect producer and consumer.
6. SLO example: 99% of events processed within 60 s; 99.9% processed without DLQ.

**Why →** Freshness and correctness map to what downstream users experience.

**Interview tip →** "Lag in seconds, not messages" — offset lag is meaningless without throughput context.

</details>

### Q123. The team uses `histogram_quantile(0.99, ...)` on dashboards and wants to average p99 across 12 pods. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Percentiles cannot be averaged; the average of per-pod p99s is not the service p99.

**What I'd do →**
1. Aggregate histogram buckets first, then compute the quantile:
   ```promql
   histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="checkout"}[5m])))
   ```
2. Remove `avg(histogram_quantile(...))` panels.
3. Check bucket boundaries match the SLO threshold (for example a `0.3` bucket for a 300 ms target).
4. Consider native histograms in Prometheus for better accuracy.

**Why →** Averaging percentiles hides tail latency, which is what users feel.

**Interview tip →** "Sum the buckets, then take the quantile" is the line.

</details>

### Q124. A team wants to add debug logging to every request in production to troubleshoot an intermittent bug. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Debug logs for all traffic multiply log cost and can hurt performance; usually you only need them for a slice.

**What I'd do →**
1. Enable debug dynamically for a subset: a header, a feature flag, or one canary pod.
2. Use trace sampling with full attributes on errors instead.
3. Set a time limit and an owner for the change.
4. Make sure no secrets or PII appear in debug logs.

**Why →** Targeted debug data solves the bug without a cost spike or new risk.

**Interview tip →** Mention runtime log-level switching without redeploying.

</details>

### Q125. You have four signals (metrics, logs, traces, profiles) in four tools. Incident responders waste time jumping between them. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Signals are only useful if you can pivot between them in one or two clicks with the same context.

**What I'd do →**
1. Standardize shared resource attributes via OpenTelemetry: `service.name`, `service.version`, `deployment.environment`, `k8s.namespace.name`.
2. Enable exemplars on latency histograms so a spike links to an actual trace ID.
3. Inject `trace_id` in logs and configure Grafana data source links (Tempo ↔ Loki "derived fields", traces to profiles).
4. Put all four in one UI where possible (Grafana with Prometheus/Mimir, Loki, Tempo, Pyroscope) or ensure deep links work.
5. Practice the pivot in game days: alert → dashboard → exemplar trace → logs → profile.

**Why →** Correlation cuts MTTR more than adding any single new signal.

**Interview tip →** Say "consistent resource attributes are the glue" — without them, correlation doesn't work.

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
