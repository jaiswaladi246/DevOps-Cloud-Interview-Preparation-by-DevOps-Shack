# SRE & Reliability — Core Concepts

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

> Core concepts, math, comparisons, best practices and rapid-fire commands for Site Reliability Engineering · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to SRE & Reliability](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [SRE Foundations & SRE vs DevOps](#sre-foundations--sre-vs-devops) — Q1–Q20
- [SLIs, SLOs & SLAs](#slis-slos--slas) — Q21–Q50
- [Error Budgets & Burn-Rate Alerting](#error-budgets--burn-rate-alerting) — Q51–Q75
- [Availability Math](#availability-math) — Q76–Q90
- [Toil, On-Call & Alert Design](#toil-on-call--alert-design) — Q91–Q110
- [Incident Management & Postmortems](#incident-management--postmortems) — Q111–Q132
- [Capacity, Load Testing & Chaos Engineering](#capacity-load-testing--chaos-engineering) — Q133–Q155
- [Release Engineering & Reliability Patterns](#release-engineering--reliability-patterns) — Q156–Q178
- [Backup, DR, Runbooks & SRE Metrics](#backup-dr-runbooks--sre-metrics) — Q179–Q192
- [Best Practices & Rapid-Fire Commands](#best-practices--rapid-fire-commands) — Q193–Q200

---

## SRE Foundations & SRE vs DevOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is Site Reliability Engineering? 🟢

<details>
<summary><b>Answer</b></summary>

SRE is an engineering discipline that applies software engineering to operations problems. Instead of scaling operations by hiring more people to do manual work, SRE teams write software and automation, define reliability as measurable targets (SLOs), and use error budgets to balance reliability against feature velocity. The term comes from Google, where Ben Treynor Sloss described it as "what happens when you ask a software engineer to design an operations team."

</details>

### Q2. How does SRE differ from DevOps? 🟢

<details>
<summary><b>Answer</b></summary>

DevOps is a culture and a set of principles: shared ownership, automation, fast feedback, small batches. SRE is a concrete, opinionated implementation of those principles with specific mechanisms:

- SLOs and error budgets decide how much risk is acceptable.
- A toil cap (about 50% of time) forces automation.
- Blameless postmortems are mandatory for significant incidents.
- SRE can hand operational responsibility back to a team that ignores reliability.

The common one-liner is "`class SRE implements interface DevOps`". DevOps tells you what to value; SRE tells you how to measure and enforce it.

</details>

### Q3. What are the core responsibilities of an SRE team? 🟢

<details>
<summary><b>Answer</b></summary>

- Define and track SLIs/SLOs with product owners, and enforce the error-budget policy.
- Run on-call and lead incident response, then drive postmortems and follow-ups.
- Eliminate toil through automation and self-service tooling.
- Capacity planning, load testing and performance work.
- Make releases safe: progressive delivery, automated rollback, change management.
- Improve observability so problems are detected before users report them.
- Review designs (production readiness reviews) for new services and big launches.

</details>

### Q4. What is the "50% rule" in SRE? 🟢

<details>
<summary><b>Answer</b></summary>

Google's guideline caps operational work (tickets, manual tasks, on-call interrupts) at 50% of an SRE's time. The other 50% must go to engineering work that improves reliability or reduces future toil. If ops load consistently exceeds the cap, the excess is redirected to the product development team (tickets, pages) until the system is improved. The cap is what stops an SRE team from turning back into a traditional ops team that only scales by headcount.

</details>

### Q5. Why does SRE say 100% reliability is the wrong target? 🟢

<details>
<summary><b>Answer</b></summary>

Because users cannot tell the difference between 99.999% and 100%: their own Wi-Fi, ISP and phone are far less reliable than that. Each extra nine costs exponentially more (redundancy, slower releases, more process) while delivering almost no user-visible benefit. A 100% target also means zero error budget, so every change becomes unacceptable risk and the team stops shipping. The right target is "reliable enough that users are happy", expressed as an SLO.

</details>

### Q6. What engagement models do SRE teams use with product teams? 🔴

<details>
<summary><b>Answer</b></summary>

- **Embedded SRE:** an SRE sits inside a product team for a period, fixing reliability gaps hands-on.
- **Consulting / advisory:** SREs review designs, SLOs and readiness but do not own the pager.
- **Full ownership (Google-style):** SRE takes over on-call for a service that has passed a production readiness review, with a hand-back clause.
- **Platform / infrastructure SRE:** SRE builds shared tooling (observability, deploy pipelines, SLO platform) used by everyone.
- **"You build it, you run it" with SRE enablement:** product teams carry the pager; SRE provides tools, training and incident leadership.

Most mid-sized companies end up with platform SRE plus enablement, because full ownership needs a large team.

</details>

### Q7. What is a Production Readiness Review (PRR)? 🟡

<details>
<summary><b>Answer</b></summary>

A PRR is a structured check that a service is ready to run in production (or ready for SRE to take over its pager). Typical checklist:

- SLIs/SLOs defined, dashboards and burn-rate alerts in place.
- Runbooks linked from every alert, on-call rotation staffed.
- Capacity tested with load tests; autoscaling and limits configured.
- Redundancy across zones, graceful shutdown, health checks.
- Safe deploys: canary, rollback tested, feature flags for risky changes.
- Backups with tested restores, defined RTO/RPO.
- Dependencies listed with their SLOs, and timeouts/retries set for each.

The output is a list of gaps with owners, not a pass/fail stamp.

</details>

### Q8. What is the difference between an SRE and a traditional sysadmin or ops engineer? 🟢

<details>
<summary><b>Answer</b></summary>

A traditional ops engineer keeps systems running mostly through manual work, and ops headcount grows with the number of servers or services. An SRE treats operations as a software problem: they automate the work, measure reliability with SLOs, and push back on product teams using error budgets. SREs are expected to write production-quality code (Go, Python), design systems, and lead incidents. The ops team scales linearly with load; an SRE team should scale sub-linearly.

</details>

### Q9. What is the difference between reliability, availability and resilience? 🟡

<details>
<summary><b>Answer</b></summary>

- **Availability:** the fraction of time or requests the service is usable. One SLI type.
- **Reliability:** the broader property that the service does what users expect: available, fast enough, correct, durable. Measured by a set of SLOs.
- **Resilience:** the ability to absorb failures and recover: redundancy, graceful degradation, self-healing. It is how you achieve reliability when components fail.

A service can be available (returns 200) but unreliable (returns stale or wrong data), which is why we define more than one SLI.

</details>

### Q10. What are the four golden signals? 🟢

<details>
<summary><b>Answer</b></summary>

From the Google SRE book, the four signals to monitor for any user-facing system:

1. **Latency:** time to serve a request, tracked separately for successes and failures (fast errors hide real latency).
2. **Traffic:** demand, for example requests per second.
3. **Errors:** rate of failed requests, explicit (5xx) or implicit (wrong content, too slow).
4. **Saturation:** how full the service is, the most constrained resource (CPU, memory, connection pool, queue depth).

Latency and errors usually become SLIs; traffic and saturation are for diagnosis and capacity.

</details>

### Q11. Compare RED and USE methods. When do you use each? 🟡

<details>
<summary><b>Answer</b></summary>

- **RED** (Rate, Errors, Duration) is for request-driven services: every microservice dashboard should show requests per second, error rate, and latency percentiles.
- **USE** (Utilization, Saturation, Errors) is for resources: for each CPU, disk, network interface, or connection pool, check how busy it is, how much work is queued, and error counts.

In practice we put RED on top of each service dashboard (it maps to SLIs) and use USE when drilling into why a service is slow, for example a node with high CPU run-queue length or a DB pool with waiting connections.

</details>

### Q12. What does "hope is not a strategy" mean in SRE? 🟢

<details>
<summary><b>Answer</b></summary>

It means reliability claims must be backed by evidence and mechanisms, not optimism. "Backups should work" is hope; a weekly automated restore test is a strategy. "The failover should be fine" is hope; a quarterly DR drill with a measured RTO is a strategy. In interviews, use it to explain why you test rollbacks, restores, failovers and alerts rather than assuming they work.

</details>

### Q13. What is the role of automation in SRE, and what are its risks? 🔴

<details>
<summary><b>Answer</b></summary>

Automation removes toil, makes operations consistent and fast, and encodes expertise so it scales beyond one person. The risks:

- Automation executes mistakes at machine speed (a bad script that drains every node).
- It can hide how the system works, so humans lose the skill to operate manually when automation fails.
- Automation without guardrails becomes an outage source.

Mitigations: dry-run modes, rate limits and blast-radius limits (one AZ at a time), idempotency, audit logs, and manual override paths that are tested.

</details>

### Q14. What is "error budget" thinking's effect on the dev vs ops conflict? 🟡

<details>
<summary><b>Answer</b></summary>

Historically developers want to ship fast and ops want stability, so they fight over every release. An error budget gives both sides one shared, objective number. While budget remains, developers can ship as fast as they like, including risky experiments. When it is exhausted, releases slow down until reliability recovers. Nobody has to win an argument; the data decides, and both teams own the same outcome.

</details>

### Q15. What is a "service owner" and why does SRE insist on one? 🟢

<details>
<summary><b>Answer</b></summary>

A service owner is the team accountable for a service's SLOs, on-call, runbooks and roadmap trade-offs. Without a clear owner, alerts go to nobody, postmortem action items rot, and SLO misses have no consequences. In production we record ownership in a service catalog (Backstage, OpsLevel, or a YAML file in Git) with team, Slack channel, on-call schedule, SLO dashboard and runbook links, and alert routing reads from it.

</details>

### Q16. What is Platform Engineering and how does it relate to SRE? 🟡

<details>
<summary><b>Answer</b></summary>

Platform Engineering builds an internal developer platform: golden paths for creating, deploying and operating services (templates, CI/CD, Kubernetes, observability, secrets). SRE focuses on the reliability of services in production. They overlap heavily: SRE teams often build the reliability parts of the platform (SLO tooling, alert templates, deploy safety, chaos tooling). A good platform bakes SRE practices in by default, so every new service gets dashboards, SLO alerts and canary deploys without asking.

</details>

### Q17. What does "observability" mean to an SRE, compared to monitoring? 🟡

<details>
<summary><b>Answer</b></summary>

Monitoring answers known questions: "is error rate above X?" with predefined dashboards and alerts. Observability is the ability to ask new questions about system behaviour from the outside, using high-cardinality telemetry (metrics, structured logs, traces, profiles) without shipping new code. SREs need monitoring to detect SLO burn and observability to explain *why* it burns, for example slicing failed requests by region, version, customer tier and dependency in a trace store.

</details>

### Q18. What is "defense in depth" for reliability? 🟡

<details>
<summary><b>Answer</b></summary>

Layering independent safeguards so one failure does not become an outage:

- Code review and tests catch bugs.
- Canary analysis catches what tests miss.
- Feature flags limit exposure and give a kill switch.
- Timeouts, retries and circuit breakers contain dependency failures.
- Multi-AZ redundancy and autoscaling absorb infrastructure failures.
- Burn-rate alerts and runbooks shorten whatever still gets through.
- Backups and DR protect against the worst cases.

Postmortems often reveal several layers failed at once; the fix is to strengthen more than one.

</details>

### Q19. What is the "SRE book" family and what does each cover? 🟢

<details>
<summary><b>Answer</b></summary>

Google published three free books at sre.google:

- **Site Reliability Engineering** (2016): principles and practices: SLOs, toil, monitoring, incident management, postmortems.
- **The Site Reliability Workbook** (2018): practical implementation: how to write SLOs, multi-window burn-rate alerting, on-call, case studies.
- **Building Secure and Reliable Systems** (2020): design for security and reliability together.

The multi-window, multi-burn-rate alerting table that interviewers ask about comes from the Workbook's "Alerting on SLOs" chapter.

</details>

### Q20. What is the "wheel of misfortune"? 🟡

<details>
<summary><b>Answer</b></summary>

A role-play training exercise where an experienced engineer acts as the "game master" and walks a trainee through a past (or invented) incident. The trainee says what they would check and do; the game master replies with what the dashboards and logs would show. It builds on-call confidence, tests runbooks, and spreads tribal knowledge without waiting for a real outage. We run one per new on-call engineer before their first shadow shift.

</details>

---

## SLIs, SLOs & SLAs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. Define SLI, SLO and SLA with an example of each. 🟢

<details>
<summary><b>Answer</b></summary>

- **SLI (indicator):** a measurement. "Proportion of `/checkout` requests that return non-5xx in under 500 ms."
- **SLO (objective):** a target for the SLI over a window. "99.9% of `/checkout` requests are good over a rolling 28 days."
- **SLA (agreement):** a contract with consequences. "If monthly availability falls below 99.5%, the customer receives a 10% service credit."

The SLO is internal and stricter; the SLA is external and looser.

</details>

### Q22. Why should the SLA be looser than the SLO? 🟢

<details>
<summary><b>Answer</b></summary>

So you get an internal warning, and time to react, before you breach a contract. If the SLO is 99.9% and the SLA is 99.5%, the burn-rate alerts and error-budget policy kick in long before customers are owed credits. If they were equal, every SLO miss would be a financial and legal event, and teams would hide problems rather than act on them.

</details>

### Q23. Why are SLIs best expressed as a ratio of good events to valid events? 🟡

<details>
<summary><b>Answer</b></summary>

A ratio is bounded between 0 and 100%, comparable across services, and maps directly to an error budget (`1 - SLO` of the valid events may be bad). It also scales with traffic: a 1-minute blip at 3 AM costs less budget than the same blip at peak, which matches real user impact. The format is:

```text
SLI = good events / valid events
availability: non-5xx responses / all responses excluding health checks
latency:      responses under 300 ms / all responses
```

</details>

### Q24. What is the difference between request-based and time-based (window-based) SLIs? 🔴

<details>
<summary><b>Answer</b></summary>

- **Request-based:** counts good requests over total requests. "99.9% of requests succeed." Weighted by traffic, so it reflects user impact closely. Best default for high-traffic services.
- **Time-based:** counts good time slices. "99.9% of 1-minute windows had error rate below 1%." Every minute counts equally, so a quiet-hour outage costs the same as a peak-hour one.

Time-based works better for low-traffic services, where one failed request out of ten can swing a request ratio wildly, and for things like batch or probe-based checks.

</details>

### Q25. Where should you measure an SLI, and what are the trade-offs? 🔴

<details>
<summary><b>Answer</b></summary>

| Point | Pros | Cons |
|---|---|---|
| Application metrics | Rich labels, easy | Misses requests that never reach the app (LB failures, crashed pods) |
| Load balancer / ingress | Sees all server-side failures, one place | Misses client network and DNS problems |
| Synthetic probes | Tests the full path from outside | Low volume, only tests scripted paths |
| Client / RUM | True user experience | Noisy, delayed, includes users' own network issues |

In production we usually take the SLI from the load balancer or ingress (for example ALB or NGINX ingress metrics) and back it with synthetic probes for low-traffic paths.

</details>

### Q26. What types of SLIs exist beyond availability and latency? 🟡

<details>
<summary><b>Answer</b></summary>

- **Quality:** proportion of responses served without degradation (not from a fallback).
- **Freshness:** proportion of data updated within X minutes (pipelines, caches, search indexes).
- **Correctness:** proportion of outputs that are right (validated against a known-good source).
- **Coverage / throughput:** proportion of records processed by a batch job within its deadline.
- **Durability:** proportion of stored objects still readable (storage systems).

Pick the type by user journey: a dashboard user cares about freshness; an API client cares about availability and latency.

</details>

### Q27. Which responses should count as "bad" in an availability SLI? 🟡

<details>
<summary><b>Answer</b></summary>

Count as bad anything that is the service's fault: 5xx responses, timeouts, connection resets, and 429s or 503s returned because the service is overloaded. Usually exclude client errors (400, 401, 403, 404) because the request itself was invalid. Be careful: a bug that returns 400 for valid input or 404 for existing objects is a real failure, so some teams track specific 4xx codes as bad. Also exclude health checks and synthetic traffic from the denominator, or they dilute the ratio.

</details>

### Q28. Why are latency SLOs written as "X% of requests under Y ms" rather than averages? 🟡

<details>
<summary><b>Answer</b></summary>

Averages hide the tail: a mean of 100 ms can include 5% of users waiting 3 seconds. "99% of requests under 300 ms" bounds the experience of almost every user and converts cleanly to good/total events with histogram buckets. It also composes into an error budget the same way availability does. Many teams use two thresholds: "90% under 100 ms and 99% under 500 ms" to capture both typical and tail experience.

</details>

### Q29. How do you write an availability SLI recording rule in Prometheus? 🟡

<details>
<summary><b>Answer</b></summary>

Record the error ratio (bad / total) per window, so alerts and dashboards reuse it cheaply:

```yaml
groups:
  - name: checkout-sli
    rules:
      - record: slo:sli_error:ratio_rate5m
        labels:
          slo: checkout-availability
        expr: |
          sum(rate(http_requests_total{job="checkout",code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="checkout"}[5m]))
```

Repeat for 30m, 1h, 2h, 6h, 1d and 3d windows. The naming convention `level:metric:operations` makes it obvious what the series is.

</details>

### Q30. How do you build a latency SLI from a Prometheus histogram? 🟡

<details>
<summary><b>Answer</b></summary>

Use the bucket whose `le` equals your threshold as "good" and `_count` as "valid". The threshold must be an actual bucket boundary, otherwise you are measuring the next bucket up.

```promql
sum(rate(http_request_duration_seconds_bucket{job="checkout",le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count{job="checkout"}[5m]))
```

That gives the fraction under 300 ms. Do not use `histogram_quantile()` for SLO math: percentiles cannot be averaged or turned into an error budget, while bucket ratios can.

</details>

### Q31. Why shouldn't you build an SLO on `histogram_quantile(0.99, ...)`? 🔴

<details>
<summary><b>Answer</b></summary>

Several reasons:

- A quantile is interpolated within a bucket, so it is an estimate whose accuracy depends on bucket layout.
- Quantiles cannot be aggregated across instances or over time; averaging p99s is mathematically meaningless.
- "p99 under 300 ms" checked per minute is a time-based SLI that ignores how many requests were slow.
- It does not translate into an error budget.

The bucket ratio `le="0.3" / count` is exact at the boundary, aggregates correctly with `sum`, and gives a good/total ratio that feeds burn-rate alerts. Keep p99 panels for diagnosis.

</details>

### Q32. How do you choose an SLO target for a new service? 🟡

<details>
<summary><b>Answer</b></summary>

1. Measure current performance for a few weeks; the SLO should start near what you already deliver, not an aspiration.
2. Ask what users need: talk to product and look at complaints, churn, support tickets around past incidents.
3. Check dependencies: you cannot exceed the composite availability of hard dependencies.
4. Pick a target slightly below current performance so the budget is meaningful but not instantly exhausted.
5. Review quarterly and tighten or loosen with evidence.

Start with an "aspirational" SLO tracked but not enforced if the team is new to it.

</details>

### Q33. What window should an SLO use: rolling or calendar? 🟡

<details>
<summary><b>Answer</b></summary>

- **Rolling window** (for example 28 or 30 days): always reflects the recent past, so a bad day affects decisions for a month and then ages out. Best for operational decisions and alerting.
- **Calendar window** (monthly, quarterly): resets on a fixed date, aligns with SLAs and business reporting, but encourages "we have budget left at month end, ship everything."

We use a rolling 28 days for engineering (four whole weeks, so each weekday appears equally often) and a calendar month for SLA reporting.

</details>

### Q34. What is an SLO for a dependency you don't own, and how do you handle it? 🔴

<details>
<summary><b>Answer</b></summary>

You cannot set an SLO for someone else's service, but you can:

- Read their published SLO or SLA (for example a cloud provider's) and treat it as an input to your own composite availability.
- Measure their actual performance from your side (client-side SLI on calls to the dependency).
- Design around it when it is lower than you need: caching, fallbacks, async processing, a second provider, or making the feature soft (degrade instead of fail).
- Agree internal SLOs with internal teams that are at least as strict as what your service needs.

The rule of thumb: a critical dependency should be roughly one nine better than the service that depends on it.

</details>

### Q35. What is a "critical user journey" (CUJ) and how does it relate to SLOs? 🟡

<details>
<summary><b>Answer</b></summary>

A CUJ is a sequence of user actions that matters to the business: "search, add to cart, check out", "log in and view the dashboard". SLOs should be defined per CUJ rather than per microservice, because users experience journeys, not services. A journey SLO might combine several SLIs (login availability, search latency, checkout availability), each with its own target, and is usually measured at the edge or with a synthetic transaction that walks the journey.

</details>

### Q36. How many SLOs should a service have? 🟡

<details>
<summary><b>Answer</b></summary>

Few: usually one to three per critical user journey, typically one availability and one latency SLO. Too many SLOs dilute attention, create alert noise, and nobody can explain what "good" means. If a metric would not change any decision when it is missed, it is a dashboard metric, not an SLO.

</details>

### Q37. What is an SLO document and what does it contain? 🟡

<details>
<summary><b>Answer</b></summary>

A short, versioned document (often Markdown or YAML in the service repo) that records:

- The service, owner and the user journeys covered.
- Each SLI with its exact definition and data source (the PromQL query).
- The target and window.
- What is excluded (health checks, planned maintenance, client errors).
- The error-budget policy and who signed off (engineering and product).
- Review cadence and change history.

Writing it down prevents arguments during an incident about what counts.

</details>

### Q38. What is OpenSLO, and what do Sloth and Pyrra do? 🔴

<details>
<summary><b>Answer</b></summary>

- **OpenSLO** is a vendor-neutral YAML specification for declaring SLOs, SLIs and alert policies as code.
- **Sloth** is a generator: you write a short SLO spec (service, objective, error and total queries) and it produces Prometheus recording rules for every window plus multi-window burn-rate alerts. It runs as a CLI or a Kubernetes controller with a `PrometheusServiceLevel` CRD.
- **Pyrra** does the same with a `ServiceLevelObjective` CRD and adds a UI showing budget and burn rate.

They remove the copy-paste errors of hand-writing 7 recording rules and 4 alerts per SLO.

```bash
sloth generate -i slos/checkout.yaml -o rules/checkout-slo.rules.yaml
```

</details>

### Q39. What is a synthetic monitor and when do you use one as an SLI source? 🟡

<details>
<summary><b>Answer</b></summary>

A synthetic monitor is a scripted probe that calls your service on a schedule from outside (Prometheus Blackbox Exporter, Grafana Synthetic Monitoring, CloudWatch Synthetics, Checkly). Use it when:

- Traffic is too low for request-based SLIs to be stable.
- You need to test the full path including DNS, TLS, CDN and load balancer.
- You want to detect failure before real users arrive (early morning).

```promql
avg_over_time(probe_success{job="blackbox",instance="https://shop.example.com/health"}[5m])
```

</details>

### Q40. How do planned maintenance windows interact with SLOs? 🟡

<details>
<summary><b>Answer</b></summary>

Two schools: exclude announced maintenance from the SLI (common in SLAs), or count it against the error budget (the SRE preference). Counting it is more honest, because users still could not use the service, and it creates an incentive to make maintenance zero-downtime. If you exclude it, define exactly how (a maintenance flag metric, or a recording rule that multiplies by a `maintenance == 0` series) so it cannot be abused.

</details>

### Q41. What is the difference between an SLO and a KPI? 🟡

<details>
<summary><b>Answer</b></summary>

A KPI measures business performance (conversion rate, revenue per user, signups). An SLO measures whether the service is reliable enough for users, with an error budget and a policy attached. They correlate: when checkout availability drops, conversion drops. Showing that correlation is a strong way to justify an SLO target to product leadership.

</details>

### Q42. What should you do when users are unhappy but the SLO is green? 🟡

<details>
<summary><b>Answer</b></summary>

The SLI is measuring the wrong thing or in the wrong place. Common causes: measuring at the app while failures happen at the load balancer or CDN; averaging away a bad region or customer tier; a latency threshold looser than what users tolerate; a broken journey whose requests return 200 with error content. Fix the SLI definition first (add measurement points, slice by region), then revisit the target. An SLO that does not track user happiness is worse than none.

</details>

### Q43. And what if the SLO keeps failing but nobody complains? 🟡

<details>
<summary><b>Answer</b></summary>

Either the target is stricter than users need, or the SLI counts things users do not see (retried requests that eventually succeed, internal batch calls). Check the definition, then lower the target with product sign-off. Keeping an SLO that is always red trains everyone to ignore it and burns engineering time on reliability nobody values.

</details>

### Q44. How do you handle SLIs for low-traffic services? 🔴

<details>
<summary><b>Answer</b></summary>

With 100 requests a day, one failure is a 1% error rate, so burn-rate alerts flap. Options:

- Add synthetic traffic to raise the volume and catch outages between real requests.
- Use time-based SLIs (good minutes / total minutes) instead of request ratios.
- Combine several small services or endpoints into one SLO.
- Lengthen alert windows and require a minimum request count before alerting:

```promql
slo:sli_error:ratio_rate1h > (14.4 * 0.001)
and on() sum(rate(http_requests_total{job="admin"}[1h])) * 3600 > 50
```

</details>

### Q45. What is the "SLI specification vs SLI implementation" distinction? 🔴

<details>
<summary><b>Answer</b></summary>

The **specification** is what you want to measure, in user terms: "the proportion of checkout attempts that complete successfully." The **implementation** is how you actually measure it: "non-5xx responses for `POST /api/checkout` at the ingress, from `nginx_ingress_controller_requests`." One spec can have several implementations with different trade-offs (server logs, LB metrics, client telemetry). Writing both down makes gaps visible, for example "our implementation misses requests rejected by the CDN."

</details>

### Q46. Should SLOs be per customer, per region, or global? 🔴

<details>
<summary><b>Answer</b></summary>

A global SLO can look healthy while one region or a large customer is down, because the healthy majority averages them out. In production:

- Keep a global SLO for reporting.
- Alert per region or per cell (failure domain), since that is where outages usually happen.
- For B2B, track an SLI for top-tier customers or per tenant if SLAs are per customer.

Watch cardinality: per-tenant recording rules for thousands of tenants get expensive; do it only for tenants with contracts.

</details>

### Q47. What are SLOs for batch jobs and data pipelines? 🟡

<details>
<summary><b>Answer</b></summary>

Batch users care about freshness and completeness, not request latency:

- **Freshness:** "99% of the time, the reporting table is less than 2 hours old."
- **Completeness / coverage:** "99.9% of daily records are processed by 06:00."
- **Correctness:** "99.99% of output rows pass validation."

Implement freshness with a timestamp metric the job pushes when it finishes:

```promql
time() - max(pipeline_last_success_timestamp_seconds{pipeline="orders-etl"}) < 7200
```

</details>

### Q48. How do you get product managers to agree to an SLO? 🟡

<details>
<summary><b>Answer</b></summary>

Talk in their terms: show past incidents, what they cost (conversion, tickets, churn), and what each extra nine costs in engineering time and slower releases. Offer two or three concrete targets with the trade-off spelled out ("99.9% means we can absorb about 43 minutes of failure a month and keep shipping daily; 99.99% means 4 minutes, and we would need multi-region work first"). Make the error-budget policy a joint decision, signed by both engineering and product leads.

</details>

### Q49. What is the "SLO as a contract between teams" idea for internal services? 🟡

<details>
<summary><b>Answer</b></summary>

When team A's service calls team B's, team B publishes an SLO, and team A designs to it. Team A sets timeouts, retries and fallbacks based on B's latency and availability targets, and does not page B for problems within B's budget. It replaces "your service is flaky" arguments with "you are burning budget at 6x, here is the graph." It also makes it clear when A needs a better SLO than B offers, which is a roadmap conversation, not an incident.

</details>

### Q50. What are common mistakes teams make with their first SLOs? 🟢

<details>
<summary><b>Answer</b></summary>

- Picking 99.99% because it sounds good, with no data or dependency analysis.
- Measuring CPU, memory or pod restarts instead of user experience.
- Too many SLOs, none of them tied to decisions.
- No error-budget policy, so missing the SLO changes nothing.
- Alerting on raw SLI thresholds ("error rate > 1% for 5 minutes") instead of burn rate.
- Never reviewing the SLO after launch.

</details>

---

## Error Budgets & Burn-Rate Alerting

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. What is an error budget and how do you calculate it? 🟢

<details>
<summary><b>Answer</b></summary>

The error budget is the amount of unreliability the SLO allows: `1 - SLO`. For a request-based SLO it is a count of bad requests; for time it is minutes of downtime.

```text
SLO 99.9% over 30 days, 50 million requests expected
budget ratio  = 0.1%
budget events = 50,000,000 x 0.001 = 50,000 failed requests
budget time   = 30 x 24 x 60 x 0.001 = 43.2 minutes of full outage
```

</details>

### Q52. What is an error-budget policy? 🟢

<details>
<summary><b>Answer</b></summary>

A pre-agreed document that says what the team does at different budget levels. Example:

- Budget above 50%: ship normally, experiments allowed.
- Budget 25 to 50%: extra review for risky changes, prioritize top reliability bug.
- Budget exhausted: freeze all non-critical releases (security fixes and reliability fixes still ship) until the rolling SLI is back within SLO; at least one engineer per team works on reliability action items.
- A single incident consuming more than 20% of budget requires a postmortem.

It is signed by engineering and product leadership so it is enforced without negotiation during a crisis.

</details>

### Q53. What happens when the error budget is exhausted? Is it a punishment? 🟡

<details>
<summary><b>Answer</b></summary>

It is not a punishment; it is a signal that users are getting less reliability than agreed, so the team rebalances work. Typically feature releases pause, the team focuses on postmortem action items and the top causes of budget burn, and release velocity resumes once the SLI recovers. If budget is constantly exhausted, either the system needs investment or the SLO is wrong; both are leadership decisions, made with data.

</details>

### Q54. What should you do if the error budget is never spent? 🟡

<details>
<summary><b>Answer</b></summary>

Unspent budget means you are either over-investing in reliability or not taking enough risk. Use it: ship faster, run chaos experiments in production, do planned failovers, migrate infrastructure, upgrade databases. Google even describes planned outages (the "Chubby planned outage") to stop other teams depending on more availability than promised. If users are happy at the current level, you may also consider whether the SLO could stay where it is while you slow down reliability spend.

</details>

### Q55. What is burn rate? 🟡

<details>
<summary><b>Answer</b></summary>

Burn rate is how fast you are consuming the error budget, relative to the rate that would use exactly the whole budget over the SLO window:

```text
burn rate = observed error ratio / (1 - SLO)
SLO 99.9%, current error ratio 1.44%  ->  burn rate = 0.0144 / 0.001 = 14.4
```

A burn rate of 1 means you will end the window with exactly zero budget left. 14.4 means a 30-day budget is gone in about 2 days (`30 / 14.4 = 2.08` days).

</details>

### Q56. How much budget does an alert at burn rate B over window W consume? 🔴

<details>
<summary><b>Answer</b></summary>

```text
budget consumed = B x W / SLO period
```

For a 30-day (720-hour) SLO:

| Burn rate | Long window | Budget consumed when it fires |
|---|---|---|
| 14.4 | 1 h | 14.4 x 1 / 720 = 2% |
| 6 | 6 h | 6 x 6 / 720 = 5% |
| 3 | 1 d | 3 x 24 / 720 = 10% |
| 1 | 3 d | 1 x 72 / 720 = 10% |

The thresholds are chosen backwards from "page me when 2% of the monthly budget burned in an hour", which is significant but not yet a disaster.

</details>

### Q57. Why is alerting on a raw error-rate threshold ("5xx > 1% for 5 minutes") a bad idea? 🟡

<details>
<summary><b>Answer</b></summary>

It has poor precision and recall at the same time. A 5-minute spike to 1.5% pages someone at 3 AM even though it used a tiny slice of budget. Meanwhile a steady 0.5% error rate (5x burn on a 99.9% SLO) never fires, yet it exhausts the monthly budget in 6 days. The threshold also has no link to what users were promised. Burn-rate alerts tie the page directly to SLO risk.

</details>

### Q58. Explain multi-window, multi-burn-rate alerting. 🔴

<details>
<summary><b>Answer</b></summary>

From the SRE Workbook: several alert rules, each with a burn-rate threshold and two windows, a long one and a short one (1/12 of the long). The alert fires only when **both** windows exceed the threshold.

| Severity | Long window | Short window | Burn rate |
|---|---|---|---|
| Page | 1h | 5m | 14.4 |
| Page | 6h | 30m | 6 |
| Ticket | 1d | 2h | 3 |
| Ticket | 3d | 6h | 1 |

- The **long window** gives precision: the problem consumed a meaningful slice of budget.
- The **short window** gives fast reset: once you fix it, the 5m ratio drops and the alert clears instead of firing for another hour.
- **Multiple burn rates** catch both sudden outages (fast, paging) and slow leaks (ticket).

</details>

### Q59. Write the fast-burn page alert for a 99.9% SLO in Prometheus. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
- alert: CheckoutErrorBudgetBurnFast
  expr: |
    (
      slo:sli_error:ratio_rate1h{slo="checkout-availability"} > (14.4 * 0.001)
      and
      slo:sli_error:ratio_rate5m{slo="checkout-availability"} > (14.4 * 0.001)
    )
    or
    (
      slo:sli_error:ratio_rate6h{slo="checkout-availability"} > (6 * 0.001)
      and
      slo:sli_error:ratio_rate30m{slo="checkout-availability"} > (6 * 0.001)
    )
  for: 2m
  labels:
    severity: page
  annotations:
    summary: "Checkout is burning its 30d error budget fast"
    runbook_url: "https://runbooks.example.com/checkout/slo-burn"
```

The `or` combines both paging tiers into one alert; many teams keep them as two named alerts instead, to show which tier fired.

</details>

### Q60. Why does the short window equal 1/12 of the long window? 🔴

<details>
<summary><b>Answer</b></summary>

It is a pragmatic ratio from the SRE Workbook. Short enough that the short-window ratio drops below threshold within minutes after recovery (5 minutes for a 1-hour window), so the alert resets fast. Long enough that it does not flap on single scrapes and still confirms the problem is ongoing. It is not a law; some teams use 1/10 or fixed values like 5m/30m/2h/6h. What matters is that you test the reset time and detection time with `promtool` or replayed data.

</details>

### Q61. What is detection time for a burn-rate alert, and how do you estimate it? 🔴

<details>
<summary><b>Answer</b></summary>

For a full outage (100% errors) and a 99.9% SLO, the 1h window ratio reaches the 1.44% threshold after `0.0144 x 60 = 0.86` minutes, so the alert condition holds in under a minute, plus the `for` duration and scrape/eval intervals. For a smaller incident at error ratio `e`, the time is roughly `threshold / e x window`:

```text
e = 5%:  0.0144 / 0.05 x 60 min ≈ 17 minutes to trigger the 1h/14.4 page
e = 2%:  0.0144 / 0.02 x 60 min ≈ 43 minutes
```

Big outages are detected fast; small ones take longer, which is correct because they burn budget slowly.

</details>

### Q62. Why does a burn-rate alert need a `for:` clause, and how long should it be? 🟡

<details>
<summary><b>Answer</b></summary>

`for:` requires the condition to hold across several evaluations before firing, which absorbs single-scrape glitches (a target restarting, a counter reset). Because the multi-window condition already filters noise, keep it short: 1 to 2 minutes for paging tiers, 5 to 15 minutes for ticket tiers. A long `for:` on top of a 1-hour window just adds detection delay.

</details>

### Q63. How do you compute "error budget remaining" in PromQL? 🔴

<details>
<summary><b>Answer</b></summary>

```promql
1 - (
  slo:sli_error:ratio_rate30d{slo="checkout-availability"}
  / (1 - 0.999)
)
```

1 means the full budget is left, 0 means exhausted, negative means you are over budget. Display it as a percentage stat panel with thresholds at 50%, 25% and 0%. Note that `rate(...[30d])` is expensive; compute it in a recording rule in its own group with a longer `interval` (for example 5m) so it does not run on every evaluation cycle.

</details>

### Q64. What is the problem with computing a 30-day ratio as `avg_over_time` of a 5-minute ratio? 🔴

<details>
<summary><b>Answer</b></summary>

`avg_over_time(slo:sli_error:ratio_rate5m[30d])` weighs every 5-minute window equally regardless of traffic, so it becomes a time-based SLI. A 5-minute outage at 3 AM with 10 requests counts as much as one at peak with 100,000. For a request-based SLO, compute the ratio of sums instead:

```promql
sum(increase(http_requests_total{job="checkout",code=~"5.."}[30d]))
/
sum(increase(http_requests_total{job="checkout"}[30d]))
```

or record per-window error and total counts separately and divide the sums.

</details>

### Q65. What should a burn-rate alert's annotations contain? 🟢

<details>
<summary><b>Answer</b></summary>

- `summary`: which SLO and how fast it is burning.
- `description`: current error ratio and burn rate, rendered with templates.
- `runbook_url`: the runbook for this alert.
- `dashboard`: link to the SLO dashboard with the right time range.

```yaml
annotations:
  summary: "{{ $labels.slo }} burning error budget"
  description: "1h error ratio is {{ $value | humanizePercentage }}"
  runbook_url: "https://runbooks.example.com/slo-burn"
```

The on-call engineer should be able to act from the page alone.

</details>

### Q66. How does Alertmanager routing fit with page vs ticket burn alerts? 🔴

<details>
<summary><b>Answer</b></summary>

Alert rules set `severity: page` or `severity: ticket`, and Alertmanager routes on that label:

```yaml
route:
  receiver: default-null
  group_by: [alertname, slo]
  routes:
    - matchers: ['severity="page"']
      receiver: pagerduty-checkout
    - matchers: ['severity="ticket"']
      receiver: jira-checkout
      repeat_interval: 24h
```

Grouping by `slo` means several firing tiers for the same SLO arrive as one notification.

</details>

### Q67. What is the difference between symptom-based and cause-based alerts? 🟢

<details>
<summary><b>Answer</b></summary>

- **Symptom-based:** users are affected: high error rate, slow responses, SLO burn. Always page-worthy if fast.
- **Cause-based:** something that might cause symptoms: high CPU, disk 80%, one pod restarting, replication lag.

Page on symptoms; turn causes into tickets, dashboards or non-paging alerts. An exception is predictive cause alerts with a clear deadline, for example "disk will be full in 4 hours" or "certificate expires in 7 days."

</details>

### Q68. How do you alert on a latency SLO with burn rates? 🔴

<details>
<summary><b>Answer</b></summary>

Exactly like availability, using the latency "bad" ratio (requests slower than the threshold):

```yaml
- record: slo:sli_latency_error:ratio_rate1h
  labels: {slo: checkout-latency}
  expr: |
    1 - (
      sum(rate(http_request_duration_seconds_bucket{job="checkout",le="0.3"}[1h]))
      /
      sum(rate(http_request_duration_seconds_count{job="checkout"}[1h]))
    )
```

Then apply the same 14.4/6/3/1 thresholds multiplied by `1 - SLO` (for a 99% latency SLO, `14.4 * 0.01`).

</details>

### Q69. Can one incident consume more than 100% of the error budget? What does that mean? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. A 2-hour full outage on a 99.9% / 30-day SLO consumes `120 / 43.2 ≈ 278%` of the budget. The rolling SLI stays below target until the incident ages out of the window (30 days later), so the error-budget policy keeps the freeze in place unless leadership explicitly grants an exception. It is a strong signal for a deep postmortem and reliability investment.

</details>

### Q70. How do you calculate budget consumed by a specific incident? 🔴

<details>
<summary><b>Answer</b></summary>

```text
budget consumed = bad events during incident / total budget events
example: 18,000 failed requests, monthly budget 50,000 -> 36% consumed
```

In Prometheus, with the incident window known:

```promql
sum(increase(http_requests_total{job="checkout",code=~"5.."}[45m] @ 1731150000))
/
(0.001 * sum(increase(http_requests_total{job="checkout"}[30d] @ 1731150000)))
```

The `@` modifier evaluates at the incident end timestamp. Put the number in the postmortem.

</details>

### Q71. Why do some teams use different SLO windows for alerting and reporting? 🔴

<details>
<summary><b>Answer</b></summary>

Alerting needs a rolling window so the burn rate reflects the recent past at any moment, independent of calendar boundaries. Reporting to leadership and customers is usually calendar-based (monthly, quarterly) to match SLAs and invoices. The alerting thresholds assume a specific period length (720 h for 30 days), so if you change the SLO window to 28 days you must recompute what a 14.4x burn means (2% of 672 h is 13.44x for an hour).

</details>

### Q72. What is error budget "burn-down" and how do you visualise it? 🟡

<details>
<summary><b>Answer</b></summary>

A burn-down chart shows remaining budget over the SLO window, like a sprint burn-down. It starts at 100% and falls with every incident. Steep drops show incidents; a steady slope shows a chronic low-level error rate. In Grafana, plot `1 - (error ratio over the window / (1 - SLO))` as a time series with a red threshold at 0. Reviewing it weekly makes reliability trends visible to product owners.

</details>

### Q73. What is the risk of alerting on error budget remaining (e.g. "budget below 10%")? 🟡

<details>
<summary><b>Answer</b></summary>

Remaining budget is a lagging, slow-moving signal. When it crosses 10%, the damage is already done, and it stays below 10% for days, so the alert either fires continuously or has to be silenced. It is useful as a ticket or a weekly report trigger for the error-budget policy, but not as a page. Burn rate tells you how fast things are getting worse right now, which is what on-call needs.

</details>

### Q74. How do you test burn-rate alert rules before deploying them? 🔴

<details>
<summary><b>Answer</b></summary>

With `promtool test rules` and synthetic input series:

```yaml
rule_files: [checkout-slo.rules.yaml]
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'http_requests_total{job="checkout",code="200"}'
        values: '0+600x120'
      - series: 'http_requests_total{job="checkout",code="503"}'
        values: '0+120x120'
    alert_rule_test:
      - eval_time: 60m
        alertname: CheckoutErrorBudgetBurnFast
        exp_alerts:
          - exp_labels: {severity: page, slo: checkout-availability}
```

Add a negative case (0.05% errors) that must produce `exp_alerts: []`. Run it in CI next to `promtool check rules`.

</details>

### Q75. What happens to burn-rate alerts during a traffic drop to zero? 🔴

<details>
<summary><b>Answer</b></summary>

With zero traffic the ratio is `0/0`, which PromQL returns as `NaN`, and `NaN > x` is false, so burn-rate alerts go silent. That is dangerous if traffic dropped because the service is unreachable (DNS broken, LB misconfigured). Cover it with:

- A traffic-drop alert comparing current rate to the same time last week:
  `sum(rate(http_requests_total{job="checkout"}[10m])) < 0.3 * sum(rate(http_requests_total{job="checkout"}[10m] offset 1w))`
- Synthetic probes from outside, which keep generating events.
- An `absent()` alert for the metric disappearing entirely.

</details>

---

## Availability Math

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. How much downtime per month does each "nine" allow? 🟢

<details>
<summary><b>Answer</b></summary>

For a 30-day month (43,200 minutes):

| SLO | Downtime per 30 days | Per year |
|---|---|---|
| 99% | 7.2 hours | 3.65 days |
| 99.5% | 3.6 hours | 1.83 days |
| 99.9% | 43.2 minutes | 8.76 hours |
| 99.95% | 21.6 minutes | 4.38 hours |
| 99.99% | 4.32 minutes | 52.6 minutes |
| 99.999% | 25.9 seconds | 5.26 minutes |

Formula: `allowed downtime = period x (1 - SLO)`.

</details>

### Q77. How do you calculate composite availability of services in series? 🟢

<details>
<summary><b>Answer</b></summary>

If a request needs every component to work (hard dependencies in series), multiply the availabilities:

```text
LB 99.99% x API 99.9% x DB 99.95%
= 0.9999 x 0.999 x 0.9995 = 0.9984 -> 99.84%
```

The composite is always lower than the weakest component. That is why a long chain of synchronous microservice calls is a reliability problem in itself.

</details>

### Q78. How do you calculate availability of redundant components in parallel? 🟢

<details>
<summary><b>Answer</b></summary>

If the system works when at least one of n independent replicas works:

```text
A = 1 - (1 - a)^n
two replicas at 99%:   1 - 0.01^2 = 99.99%
three replicas at 99%: 1 - 0.01^3 = 99.9999%
```

This assumes truly independent failures and instant, reliable failover. In practice the failover mechanism, shared dependencies and correlated failures (same bad deploy, same AZ) set the real limit.

</details>

### Q79. Why do the parallel-availability numbers lie in practice? 🔴

<details>
<summary><b>Answer</b></summary>

- **Correlated failures:** replicas share the same code, config, kernel and region, so one bad deploy or expired certificate takes all of them down.
- **Failover is a component too:** health checks, DNS TTLs, load balancer reaction time, and leader election have their own failure rates and delays.
- **Capacity:** if two replicas each run at 70% and one dies, the survivor gets 140% and falls over.
- **Shared dependencies:** both replicas call the same database.

Real redundancy needs diversity (different AZs, staggered deploys) and N+1 capacity, plus testing failover regularly.

</details>

### Q80. Your service depends on five services each at 99.9%. What availability can you promise? 🟡

<details>
<summary><b>Answer</b></summary>

If all are hard dependencies: `0.999^5 = 0.995`, about 99.5%, before counting your own failures. You cannot promise 99.9% unless you remove hard dependencies: cache responses, make calls asynchronous through a queue, degrade gracefully when a non-critical dependency fails, or demand better SLOs from the dependencies. This calculation is a classic interview question; showing you would turn soft dependencies into degradations is the senior answer.

</details>

### Q81. What is the difference between a hard and a soft dependency? 🟡

<details>
<summary><b>Answer</b></summary>

A **hard** (critical) dependency must work for the request to succeed: checkout cannot complete without the payment service. A **soft** dependency improves the response but is optional: recommendations, personalization, reviews. Soft dependencies should have short timeouts and fallbacks so their failure degrades rather than fails the request. Classifying dependencies is part of every production readiness review.

</details>

### Q82. What are MTBF and MTTR, and how do they relate to availability? 🟡

<details>
<summary><b>Answer</b></summary>

- **MTBF:** mean time between failures (how often things break).
- **MTTR:** mean time to repair/restore (how long each failure lasts).

```text
availability = MTBF / (MTBF + MTTR)
MTBF 30 days (43,200 min), MTTR 60 min -> 43200 / 43260 = 99.86%
cut MTTR to 10 min                       -> 43200 / 43210 = 99.98%
```

Cutting MTTR is usually cheaper than preventing every failure: faster detection, one-click rollback, and good runbooks raise availability directly.

</details>

### Q83. Convert 99.95% monthly availability into allowed downtime per week and per day. 🟡

<details>
<summary><b>Answer</b></summary>

```text
per week: 7 x 24 x 60 = 10,080 min x 0.0005 = 5.04 minutes
per day:  1,440 min x 0.0005 = 0.72 minutes ≈ 43 seconds
```

Useful sanity check: a single 10-minute deploy outage per week already breaks 99.95%.

</details>

### Q84. A deploy causes 3 minutes of 100% errors every release, and you release 20 times a month. What's the max SLO you can meet? 🟡

<details>
<summary><b>Answer</b></summary>

```text
downtime = 20 x 3 = 60 minutes per month
availability = 1 - 60 / 43,200 = 99.86%
```

You cannot meet 99.9% from deploys alone, before any real incident. The fix is not fewer deploys; it is zero-downtime deploys: rolling updates with readiness probes, connection draining, `preStop` hooks and backward-compatible schema changes.

</details>

### Q85. How do you calculate availability for a multi-region active-active setup? 🔴

<details>
<summary><b>Answer</b></summary>

With two regions each at 99.9% and global load balancing that fails over automatically:

```text
regions in parallel: 1 - 0.001^2 = 99.9999%
times global LB/DNS availability (say 99.99%): ≈ 99.99%
```

The global routing layer and any shared global dependency (a single-region database primary, a global config service, the CI/CD system that pushes bad config everywhere) end up dominating. Also each region needs capacity for 100% of traffic during failover, which doubles the cost.

</details>

### Q86. What is the "nines don't add, they multiply" trap with SLAs from cloud providers? 🟡

<details>
<summary><b>Answer</b></summary>

Teams read "EC2 99.99%, RDS 99.95%, ELB 99.99%" and claim 99.99% for their app. In series that is roughly `0.9999 x 0.9995 x 0.9999 ≈ 99.93%`, before your own code, deploys and config. Provider SLAs also only promise service credits, not that the outage will not happen, and usually apply per region with Multi-AZ deployment. Your SLO has to be built from measured dependency behaviour and your own failure history.

</details>

### Q87. What is the difference between availability measured by uptime vs by successful requests? 🔴

<details>
<summary><b>Answer</b></summary>

Uptime-based availability counts minutes the service was "up" (often a health check passing). Request-based availability counts successful requests over total. A service can pass health checks while failing 30% of real requests (one bad shard, one broken endpoint), so uptime overstates reliability. Request-based also weights incidents by traffic. Use request-based SLIs for services with steady traffic; use uptime-style probes for low traffic and as a secondary signal.

</details>

### Q88. How does retrying affect composite availability? 🔴

<details>
<summary><b>Answer</b></summary>

If failures are independent and transient, one retry turns a dependency at availability `a` into roughly `1 - (1 - a)^2` for that call: 99% becomes 99.99%. But this only holds for transient, uncorrelated failures. When the dependency is overloaded or down, retries do not help and multiply load (a 3-layer stack each retrying 3 times sends 27x traffic to the bottom). So retries improve availability for blips and harm it during real outages unless you add backoff, jitter and retry budgets.

</details>

### Q89. What is a "cell-based architecture" and how does it change availability math? 🔴

<details>
<summary><b>Answer</b></summary>

The system is split into independent cells (full stacks), each serving a subset of customers, with a thin routing layer in front. A failure (bad deploy, poison request, overload) hits one cell, so the blast radius is `1/N` of customers instead of everyone. Availability per customer stays similar, but the fraction of users affected by any single incident drops, and deploys can roll cell by cell with real-traffic verification. AWS uses this pattern widely; the cost is operational complexity and duplicated infrastructure.

</details>

### Q90. Why might you set a lower SLO than your system currently achieves? 🔴

<details>
<summary><b>Answer</b></summary>

Because users will come to depend on whatever you actually deliver (Hyrum's law for reliability). If the system happens to run at 99.99% but you only commit to 99.9%, you keep budget for migrations, experiments and fast releases, and downstream teams design for the realistic number, adding their own fallbacks. Over-delivering consistently creates an implicit SLA that you will eventually break.

</details>

---

## Toil, On-Call & Alert Design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q91. What is toil? Give the defining characteristics. 🟢

<details>
<summary><b>Answer</b></summary>

Toil is operational work tied to running a service that is:

- **Manual:** a human runs it.
- **Repetitive:** done again and again.
- **Automatable:** a machine could do it.
- **Tactical:** interrupt-driven, reactive.
- **No enduring value:** the service is no better afterwards.
- **Scales linearly** with traffic or service count.

Examples: manually restarting a leaking service, copying configs between environments, resizing disks by hand, processing the same access request tickets.

</details>

### Q92. What is NOT toil, even though it feels like it? 🟡

<details>
<summary><b>Answer</b></summary>

- Overhead such as meetings, planning, hiring and HR tasks (it is overhead, not toil).
- Incident response that produces learning and action items.
- Designing and writing automation.
- One-off migrations that permanently improve the system.

Distinguishing them matters because the toil cap is about work that grows with the service and returns nothing; mislabeling everything as toil hides the real automation candidates.

</details>

### Q93. How do you measure toil? 🟡

<details>
<summary><b>Answer</b></summary>

- Tag tickets and on-call tasks (`toil:cert-renewal`, `toil:disk-resize`) and report hours per category monthly.
- Run periodic surveys or time-tracking for a sample week per quarter.
- Count pages and interrupts per shift, and which alerts generated them.
- Track ticket volume growth against service growth (linear growth means toil).

Then rank categories by `hours per month x growth rate` and automate from the top.

</details>

### Q94. How do you decide what toil to automate first? 🟡

<details>
<summary><b>Answer</b></summary>

Prioritize by total time cost, frequency, error risk and growth: a 5-minute task done 40 times a week (about 3.3 hours weekly) beats a 2-hour task done quarterly. Also weigh risk: a manual step that causes outages when done wrong deserves automation even if rare. Compare automation effort with time saved over a year, but include intangibles: fewer interrupts, faster response, consistent outcomes. Start with self-service (let teams do it themselves safely), then full automation.

</details>

### Q95. What makes an alert page-worthy? 🟢

<details>
<summary><b>Answer</b></summary>

A page must be:

1. **Urgent:** waiting until business hours would hurt users or the budget.
2. **Actionable:** the responder can do something, and the runbook says what.
3. **Real user impact or imminent impact:** symptom-based, ideally SLO burn.
4. **Requires human judgment:** if a script could fix it, automate the fix instead.

Anything else becomes a ticket, a dashboard, or gets deleted.

</details>

### Q96. What is alert fatigue and what causes it? 🟢

<details>
<summary><b>Answer</b></summary>

Alert fatigue is when on-call engineers receive so many alerts, especially non-actionable or false ones, that they start ignoring or auto-acknowledging them, and miss the real one. Causes: cause-based alerts (CPU, single pod restarts), thresholds without duration, duplicates from several tools, alerts nobody owns, flapping alerts, and alerts left on "temporarily" after an incident. It degrades both reliability and engineers' health.

</details>

### Q97. How do you reduce a noisy pager? 🟡

<details>
<summary><b>Answer</b></summary>

1. Export the last 30 days of pages and group by alert name.
2. For each alert, check: was it actionable? Did anyone do anything? Was it a symptom?
3. Delete alerts with no action taken, convert cause alerts into tickets, merge duplicates.
4. Replace threshold alerts with SLO burn-rate alerts.
5. Add `for:` durations and Alertmanager grouping and inhibition.
6. Review pages weekly in the on-call handoff and set a target (for example fewer than 2 pages per shift).

</details>

### Q98. What is Alertmanager inhibition and when is it useful? 🔴

<details>
<summary><b>Answer</b></summary>

Inhibition suppresses some alerts while another alert is firing. Classic uses: suppress all warnings for a cluster when the whole cluster is unreachable, or suppress per-service alerts when the database they depend on is down.

```yaml
inhibit_rules:
  - source_matchers: ['alertname="ClusterUnreachable"']
    target_matchers: ['severity=~"warning|page"']
    equal: [cluster]
```

The `equal` field ensures you only inhibit alerts in the same cluster.

</details>

### Q99. What is the difference between grouping, inhibition and silencing in Alertmanager? 🟡

<details>
<summary><b>Answer</b></summary>

- **Grouping:** batches related alerts into one notification (`group_by: [alertname, cluster]`), so 50 pods failing produces one page.
- **Inhibition:** automatically mutes alerts while a related, more important alert fires.
- **Silencing:** manual, time-boxed mute by matchers, for planned maintenance or a known issue being worked on.

```bash
amtool silence add alertname=CheckoutErrorBudgetBurnFast --duration=2h --comment="INC-482 mitigation in progress"
```

</details>

### Q100. How should an on-call rotation be structured? 🟡

<details>
<summary><b>Answer</b></summary>

- **Primary and secondary** on-call, with automatic escalation if primary does not acknowledge in 5 to 10 minutes.
- **Enough people:** at least 6 to 8 per rotation for single-site 24/7, so each person is on call about one week in six or eight; or follow-the-sun across time zones so nobody is paged at night.
- **Shift length:** one week is common; 12-hour shifts for follow-the-sun.
- **Handoff:** a written summary of open issues and a short meeting.
- **Compensation and recovery:** time off after a bad night, pay or time-in-lieu.
- **Ownership:** the people who build the service share the pager.

</details>

### Q101. How many pages per on-call shift is sustainable? 🟡

<details>
<summary><b>Answer</b></summary>

Google's guideline is at most two incidents per 12-hour shift, because each real incident (diagnosis, mitigation, follow-up, postmortem) takes hours. More than that and engineers cannot handle them properly or do any engineering work. Track pages per shift, out-of-hours pages, and time to acknowledge as on-call health metrics, and treat sustained overload as a reliability problem with its own action items.

</details>

### Q102. What is an escalation policy? 🟡

<details>
<summary><b>Answer</b></summary>

A rule set in the paging tool (PagerDuty, Opsgenie, Grafana OnCall, incident.io) defining who is notified and when if a page is not acknowledged: primary immediately, secondary after 10 minutes, the team lead or manager after 20 minutes. It also covers escalation to other teams (database, network, security) and to leadership for SEV-1s. Test it: an escalation policy that pages a person who left the company is a common postmortem finding.

</details>

### Q103. What is the difference between a page, a ticket and a log-only alert? 🟢

<details>
<summary><b>Answer</b></summary>

- **Page:** interrupts a human now, any time of day. Reserved for urgent, user-impacting issues.
- **Ticket:** needs a human within a business day or a few days: slow burn, disk filling in a week, certificate expiring in 14 days.
- **Log-only / dashboard:** information useful during diagnosis, never routed to a person.

Every alert definition should state which of the three it is, and severity labels should drive routing.

</details>

### Q104. Why should every alert link to a runbook? 🟢

<details>
<summary><b>Answer</b></summary>

The on-call engineer may be woken at 3 AM, new to the team, or unfamiliar with the service. A runbook link in the alert annotation tells them what the alert means, how to confirm impact, the first mitigation steps, and who to escalate to. It reduces MTTR, spreads knowledge beyond the original author, and makes alert quality reviewable: if you cannot write a runbook for an alert, it probably is not actionable.

</details>

### Q105. What should an on-call handoff include? 🟡

<details>
<summary><b>Answer</b></summary>

- Incidents and pages during the shift, with links.
- Ongoing issues, active silences and their expiry.
- Risky changes or launches planned for the next shift.
- Alerts that were noisy or wrong, with tickets to fix them.
- Any tool or access problems found.

A 15-minute weekly handoff meeting plus a written note in a shared doc keeps context from being lost.

</details>

### Q106. How do you onboard a new engineer to on-call? 🟡

<details>
<summary><b>Answer</b></summary>

1. Read the service architecture, SLOs and the top runbooks.
2. Wheel-of-misfortune sessions on past incidents.
3. Shadow the primary for one rotation (receive pages, do not own them).
4. Reverse shadow: they own the pager with an experienced engineer as backup.
5. Join the rotation normally, with a clear escalation path.

Also verify access before the first shift: paging app, VPN, cloud console, kubeconfig, dashboards, break-glass credentials.

</details>

### Q107. What is "follow-the-sun" on-call? 🟢

<details>
<summary><b>Answer</b></summary>

Teams in different time zones (for example India, Europe, US West) each cover their own daytime hours, handing off at the end of their day. Nobody is paged at night, which improves response quality and retention. It needs at least two or three sites with enough engineers who know the service, consistent runbooks, and a disciplined handoff, because context loss at handoff is the main risk.

</details>

### Q108. Why do teams track MTTA, and what is a good value? 🟡

<details>
<summary><b>Answer</b></summary>

MTTA (mean time to acknowledge) measures how quickly a human responds to a page. High MTTA points to fatigue (pages are ignored), bad routing (wrong person), or broken notification settings. For paging alerts a typical target is under 5 minutes, with escalation to secondary at 5 to 10 minutes. Very low MTTA with no action afterwards may mean people are acknowledging pages to silence them.

</details>

### Q109. What is the difference between a "heartbeat" or "dead man's switch" alert and normal alerts? 🟡

<details>
<summary><b>Answer</b></summary>

A dead man's switch is an alert that always fires; the receiver (PagerDuty, Healthchecks.io, Opsgenie heartbeat) raises an alarm when it **stops** arriving. It proves the whole alerting pipeline (Prometheus, rule evaluation, Alertmanager, network to the paging service) works. kube-prometheus-stack ships one called `Watchdog`:

```yaml
- alert: Watchdog
  expr: vector(1)
  labels:
    severity: none
```

Without it, a crashed Alertmanager looks exactly like a quiet night.

</details>

### Q110. Should developers be on call for their own services? 🟡

<details>
<summary><b>Answer</b></summary>

Generally yes ("you build it, you run it"): the feedback loop from production pain to code changes is shortest, and teams write better alerts and safer releases when they get the pages. It needs support: SRE-built tooling and alert templates, sane rotation sizes, training, runbooks, and a toil budget. Very large or critical shared infrastructure (databases, networking, Kubernetes platform) usually stays with dedicated SRE or platform teams.

</details>

---

## Incident Management & Postmortems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q111. What is an incident, and how is it different from a bug or an alert? 🟢

<details>
<summary><b>Answer</b></summary>

An incident is an unplanned event that degrades or threatens service for users and needs a coordinated response now. An alert is a signal that may or may not indicate an incident. A bug is a defect that may cause incidents but can often wait in the backlog. Declaring an incident triggers the process: roles, a dedicated channel, a timeline, status updates, and later a postmortem.

</details>

### Q112. How do you define severity levels? 🟢

<details>
<summary><b>Answer</b></summary>

By user and business impact, not by technical difficulty. A typical scale:

| Severity | Definition | Response |
|---|---|---|
| SEV-1 | Critical journey down for most users, data loss, security breach | Page IC immediately, exec updates, status page |
| SEV-2 | Major degradation or a critical journey down for a subset (one region, key customer) | Page on-call and IC, status page |
| SEV-3 | Minor degradation, workaround exists | On-call during business hours |
| SEV-4 | Cosmetic or internal-only impact | Ticket |

Publish examples for each level so people can classify in seconds.

</details>

### Q113. What are the roles in incident response (ICS-style)? 🟢

<details>
<summary><b>Answer</b></summary>

- **Incident Commander (IC):** owns the incident: coordinates, delegates, decides, keeps the timeline. Does not debug.
- **Operations lead:** leads the hands-on investigation and mitigation, directs responders.
- **Communications lead:** writes internal and external updates (stakeholders, status page, support) on a fixed cadence.
- **Scribe (optional):** records the timeline, decisions and actions.
- **Subject matter experts:** pulled in by Ops for specific systems.

For a small incident one person may hold several roles; the IC explicitly says who holds what.

</details>

### Q114. Why shouldn't the Incident Commander debug the problem? 🟡

<details>
<summary><b>Answer</b></summary>

Debugging needs deep focus; coordination needs a wide view. If the IC is inside a terminal, nobody is tracking who is doing what, whether stakeholders were updated, whether to escalate, or whether a rollback should happen now. Incidents drag on when the most senior engineer becomes both IC and debugger and stops making decisions. The IC's job is to ask "what is our current hypothesis, who is on it, what is the next decision, and when do we check in?"

</details>

### Q115. What does "mitigate first, root cause later" mean? 🟢

<details>
<summary><b>Answer</b></summary>

During an incident the goal is to restore service, not to understand it. If a deploy happened 10 minutes before errors began, roll it back now, even if you are not sure it is the cause. Other fast mitigations: fail over to another region, disable a feature flag, scale up, block abusive traffic, restart a stuck component. Capture evidence first if cheap (logs, a heap dump, `kubectl describe`), then mitigate. Full root cause analysis happens in the postmortem.

</details>

### Q116. When should you declare an incident? 🟢

<details>
<summary><b>Answer</b></summary>

Early: as soon as there is a real chance users are affected and it needs coordination, more than one person, or more than about 15 minutes. Declaring is cheap (open a channel, assign IC) and you can downgrade or close it. Declaring late is expensive: lost time, uncoordinated responders, and stakeholders finding out from customers. Many companies encourage anyone, including support staff, to declare.

</details>

### Q117. What goes into an incident timeline, and why keep it live? 🟡

<details>
<summary><b>Answer</b></summary>

Timestamped entries (UTC) for: first symptom, detection (alert or report), acknowledgement, declaration, each hypothesis tested and its result, each mitigation action with who did it, status updates sent, mitigation confirmed, resolution. Keeping it live in the incident channel or tool (incident.io, FireHydrant, a pinned doc) means new responders can catch up without interrupting, and the postmortem timeline writes itself instead of being reconstructed from memory.

</details>

### Q118. How often and what should incident status updates say? 🟢

<details>
<summary><b>Answer</b></summary>

On a fixed cadence (every 30 minutes for SEV-1, every hour for SEV-2) even if nothing changed. Each update states: current impact (who, what), what we know, what we are doing, and when the next update is. Avoid speculation on root cause and internal jargon in external updates.

```text
[14:30 UTC] Investigating: about 20% of checkout attempts in EU are failing.
We have identified a recent change and are rolling it back. Next update 15:00 UTC.
```

</details>

### Q119. What is a blameless postmortem? 🟢

<details>
<summary><b>Answer</b></summary>

A written review after an incident that focuses on how the system and processes allowed the failure, not on who made a mistake. The assumption is that everyone acted reasonably given the information, tools and pressure they had. If an engineer could delete production data with one command, the problem is the missing guardrail, not the engineer. Blamelessness makes people report problems honestly, which is the only way to find real contributing factors.

</details>

### Q120. What sections does a good postmortem contain? 🟢

<details>
<summary><b>Answer</b></summary>

- Title, date, severity, authors, status.
- **Summary:** two or three sentences of what happened.
- **Impact:** users affected, duration, error budget consumed, revenue or SLA effect.
- **Timeline** in UTC.
- **Root cause and contributing factors.**
- **Detection:** how we found out and how long it took.
- **Resolution:** what fixed it.
- **What went well / what went poorly / where we got lucky.**
- **Action items:** each with an owner, priority, ticket link and due date.
- **Lessons learned.**

</details>

### Q121. When is a postmortem required? 🟢

<details>
<summary><b>Answer</b></summary>

Define triggers in advance, for example:

- Any SEV-1 or SEV-2.
- User-visible downtime or data loss beyond a threshold.
- A single incident consuming more than 20% of the error budget.
- Any data loss or security incident.
- Manual intervention like a rollback or failover in production.
- A near miss with an interesting lesson, if anyone requests it.

Postmortems are typically drafted within 2 to 5 business days and reviewed in a meeting.

</details>

### Q122. Explain the 5 whys with an example. 🟢

<details>
<summary><b>Answer</b></summary>

Ask "why" repeatedly until you reach a cause you can fix systemically.

1. Why did checkout fail? The payment service returned 500s.
2. Why? Its database connection pool was exhausted.
3. Why? A new query held connections for 30 seconds.
4. Why? It did a full table scan because an index was missing.
5. Why was that not caught? Migrations are not tested against production-sized data, and there is no query time alert.

Action items come from the last answers: add the index, add a slow-query check to CI with realistic data, add pool saturation alerts.

</details>

### Q123. What are the limitations of the 5 whys? 🟡

<details>
<summary><b>Answer</b></summary>

- It implies a single linear causal chain, while complex outages have several contributing factors interacting.
- Results depend on who asks; different people reach different "root causes".
- It tends to stop at human error ("why? the engineer ran the wrong command") unless facilitated well.
- It ignores what went right.

Use it as a tool inside a broader analysis: list contributing factors across code, process, tooling, monitoring and organization, and ask "how" questions ("how did this make sense to the person at the time?").

</details>

### Q124. What makes a good postmortem action item? 🟡

<details>
<summary><b>Answer</b></summary>

- **Specific and verifiable:** "add alert on DB pool saturation above 90% for 5m", not "improve monitoring".
- **Owned** by one person, with a **due date** and a **priority**.
- **Tracked** in the normal backlog with a postmortem label, reviewed until done.
- **Balanced** across prevention (stop it happening), detection (find it faster) and mitigation (recover faster).
- **Proportionate:** a few high-value items beat twenty that never get done.

</details>

### Q125. Why include "where we got lucky" in a postmortem? 🟡

<details>
<summary><b>Answer</b></summary>

Because luck is a hidden risk. "The incident happened at 10 AM when the database expert was online" or "the bad config only hit one region because deploys happened to be slow that day" means next time could be much worse. Writing it down turns lucky accidents into designed safeguards (documented DB runbook, deliberate region-by-region rollout).

</details>

### Q126. How do you keep postmortems blameless in practice? 🟡

<details>
<summary><b>Answer</b></summary>

- Use roles, not names, in the narrative ("the on-call engineer"), or names only for factual actions.
- Ask "what information did they have?" and "what made this action seem right?" rather than "why did you do that?"
- Leadership publicly thanks people for honest reporting and never uses postmortems in performance reviews.
- Facilitate the review with a neutral person who stops blame language.
- Focus action items on systems, tools and processes.

</details>

### Q127. What is an incident review meeting and how should it run? 🟡

<details>
<summary><b>Answer</b></summary>

A 30 to 60 minute meeting where the postmortem author walks through the timeline and contributing factors, participants correct facts and add context, and the group agrees on action items and owners. A facilitator keeps it blameless and on time. Many companies also hold a weekly reliability review where leaders read recent postmortems, track action item completion, and spot patterns across incidents.

</details>

### Q128. What is the difference between root cause and contributing factors? 🔴

<details>
<summary><b>Answer</b></summary>

"Root cause" suggests one trigger that, if removed, prevents the incident. In complex systems an outage usually needs several conditions at once: a latent bug, a config change, peak traffic, missing alerts, a failover that was never tested. Each is a **contributing factor**. Focusing on one root cause (often the last change) misses the defensive layers that also failed. Modern postmortems list the trigger and all contributing factors, and add action items for several of them.

</details>

### Q129. What tools are used to manage incidents? 🟢

<details>
<summary><b>Answer</b></summary>

- **Paging and on-call:** PagerDuty, Opsgenie, Grafana OnCall / Grafana IRM, Splunk On-Call.
- **Incident coordination:** incident.io, FireHydrant, Rootly, or Slack with a bot that opens channels and tracks roles.
- **Status pages:** Atlassian Statuspage, Instatus, incident.io status pages.
- **Postmortems:** templates in Confluence, Notion, Google Docs, or the incident tool itself.

The tool matters less than having one consistent process everyone knows.

</details>

### Q130. What is a "near miss" and why review it? 🟡

<details>
<summary><b>Answer</b></summary>

A near miss is an event that could have caused an incident but did not, by luck or a last-moment catch: a destructive Terraform plan noticed in review, a certificate renewed hours before expiry, a failover that worked only because traffic was low. Reviewing near misses is cheap learning without user impact, and they often reveal the same weaknesses that later cause real outages.

</details>

### Q131. How do you handle communications for a security incident differently? 🔴

<details>
<summary><b>Answer</b></summary>

Security incidents need a smaller, need-to-know channel (the attacker might be reading the normal one), involvement of security, legal and privacy teams early, careful evidence preservation (snapshots, logs) before mitigation destroys it, and external communication reviewed by legal because of disclosure obligations (for example GDPR's 72-hour notification to authorities). The incident structure (IC, Ops, Comms) stays the same, but the Comms lead works with legal and the IC coordinates with the security lead.

</details>

### Q132. What metrics show whether incident management is improving? 🟡

<details>
<summary><b>Answer</b></summary>

- Incident count by severity and by service, trend over quarters.
- MTTD, MTTA and MTTR (median and p90, not just mean).
- Percentage of incidents detected by alerts vs by customers.
- Postmortem completion rate and time to publish.
- Action item completion rate and age.
- Repeat incidents (same contributing factor).

Use them for trends and learning, not targets for individuals, or people will start avoiding declaring incidents.

</details>

---

## Capacity, Load Testing & Chaos Engineering

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q133. What is capacity planning in SRE? 🟢

<details>
<summary><b>Answer</b></summary>

Making sure there is enough capacity, at acceptable cost, to serve forecast demand within SLO, including during failures. It combines:

- **Demand forecast:** organic growth trend plus inorganic events (launches, marketing campaigns, seasonal peaks).
- **Capacity per unit:** how much load one instance handles within SLO, measured with load tests.
- **Headroom:** spare capacity for failure domains (lose one AZ), deploys (surge pods), and spikes.
- **Lead time:** how long it takes to get more (minutes for autoscaling, weeks for reserved capacity or quota increases).

</details>

### Q134. What is N+1 and N+2 redundancy for capacity? 🟢

<details>
<summary><b>Answer</b></summary>

N is the number of units needed to serve peak load. **N+1** adds one spare so you survive losing any single unit; **N+2** survives one planned outage (a deploy or maintenance) plus one unplanned failure at the same time. For zones: if you need 6 pods at peak across 3 AZs and want to survive losing an AZ, you need 3 pods per AZ (9 total), because the remaining two AZs must carry the full load.

</details>

### Q135. How do you calculate how many instances you need? 🟡

<details>
<summary><b>Answer</b></summary>

```text
peak forecast           = 4,000 RPS in 6 months
per-pod capacity at SLO = 250 RPS (from load test, at p99 < 300 ms)
target utilization      = 70% (headroom for spikes)
pods needed             = 4000 / (250 x 0.7) ≈ 23
survive 1 of 3 AZs down = 23 x 3/2 ≈ 35 pods
```

Measure per-pod capacity with a load test, not CPU guesses, and redo the test when the code changes significantly.

</details>

### Q136. What is the difference between load, stress, spike, soak and breakpoint testing? 🟢

<details>
<summary><b>Answer</b></summary>

- **Load test:** expected production load; verify SLOs hold.
- **Stress test:** beyond expected load; see how the system degrades and recovers.
- **Spike test:** sudden jump in traffic; check autoscaling and queueing.
- **Soak (endurance) test:** normal load for hours; find memory leaks, connection leaks, disk growth.
- **Breakpoint test:** ramp until failure to find maximum capacity.
- **Smoke test:** tiny load to check the script and system work at all.

</details>

### Q137. What is k6 and why do SRE teams use it? 🟢

<details>
<summary><b>Answer</b></summary>

k6 is an open-source load-testing tool from Grafana Labs. You write tests in JavaScript, and the engine (written in Go) runs them efficiently from a single binary. SRE teams like it because tests are code (reviewed in Git), thresholds turn SLOs into pass/fail criteria with a non-zero exit code for CI, it supports open-model arrival-rate executors, and it can stream results to Prometheus or Grafana Cloud. The k6 Operator runs distributed tests on Kubernetes.

</details>

### Q138. Write a minimal k6 test with SLO thresholds. 🟡

<details>
<summary><b>Answer</b></summary>

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 50,
  duration: '5m',
  thresholds: {
    http_req_failed: ['rate<0.001'],                 // 99.9% availability
    http_req_duration: ['p(95)<200', 'p(99)<300'],   // latency in ms
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/api/checkout`);
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

Run with `k6 run -e BASE_URL=https://staging.example.com test.js`. If any threshold fails, k6 exits with code 99, which fails the CI job.

</details>

### Q139. What is the difference between open and closed workload models in load testing? 🔴

<details>
<summary><b>Answer</b></summary>

- **Closed model** (`constant-vus`, `ramping-vus`): a fixed number of virtual users, each waiting for its response before sending the next. When the system slows down, the request rate drops automatically, which hides the problem (coordinated omission).
- **Open model** (`constant-arrival-rate`, `ramping-arrival-rate`): requests arrive at a fixed rate regardless of response time, like real users on the internet. When the system slows, requests pile up and latency explodes, which is what happens in production.

For SLO validation of public services, use arrival-rate executors.

```javascript
scenarios: { steady: { executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s',
  duration: '10m', preAllocatedVUs: 50, maxVUs: 300 } }
```

</details>

### Q140. What are the key k6 built-in metrics? 🟢

<details>
<summary><b>Answer</b></summary>

- `http_reqs`: total requests and rate.
- `http_req_failed`: rate of failed requests (by default status >= 400 or network error).
- `http_req_duration`: total time (sending + waiting + receiving), excludes connection setup.
- `http_req_waiting`: time to first byte.
- `http_req_connecting`, `http_req_tls_handshaking`: connection costs.
- `iterations`, `vus`, `checks`, `data_received`, `data_sent`.

Thresholds can target tagged sub-metrics, for example `'http_req_duration{name:checkout}': ['p(99)<500']`.

</details>

### Q141. What is "the knee" of a latency curve and why does it matter for capacity? 🔴

<details>
<summary><b>Answer</b></summary>

As load increases, latency stays roughly flat until a resource saturates (CPU, DB connections, thread pool), then rises sharply: that bend is the knee. Running near or past it means small traffic increases cause big latency spikes and timeouts. Capacity planning targets a utilization comfortably below the knee (often 60 to 75% of the load where it appears), and a breakpoint test with a ramping arrival rate is how you find it.

</details>

### Q142. What is Little's Law and how is it useful for SREs? 🔴

<details>
<summary><b>Answer</b></summary>

`L = λ x W`: the average number of requests in the system equals the arrival rate times the average time each spends in the system.

```text
500 RPS x 0.2 s latency = 100 concurrent requests in flight
if latency rises to 2 s: 500 x 2 = 1,000 in flight
```

It tells you how many concurrent connections, threads, or DB pool slots you need, and why latency increases cause sudden resource exhaustion: a slow dependency multiplies in-flight requests, filling pools and queues.

</details>

### Q143. Should you load test in production? 🔴

<details>
<summary><b>Answer</b></summary>

Staging rarely matches production in data size, traffic mix, cache warmth or infrastructure, so production tests find real limits. If you do it:

- Run during low traffic, ramp gradually, and watch SLO burn live.
- Tag synthetic traffic (header like `X-Load-Test: true`) so it is excluded from SLIs and analytics and can be blocked instantly.
- Use test accounts and avoid side effects (no real payments, emails).
- Define abort criteria (k6 `abortOnFail` thresholds, or a burn-rate alert stops the test).
- Announce it to on-call and dependent teams.

</details>

### Q144. What is chaos engineering? 🟢

<details>
<summary><b>Answer</b></summary>

The discipline of experimenting on a system to build confidence that it can withstand turbulent conditions in production. It is not randomly breaking things: you define steady state, form a hypothesis ("if one checkout pod dies, the SLI stays above 99.9%"), inject a controlled fault, measure, and learn. Netflix popularized it with Chaos Monkey, which randomly terminated instances to force engineers to build resilient services.

</details>

### Q145. What are the principles or steps of a chaos experiment? 🟡

<details>
<summary><b>Answer</b></summary>

1. **Define steady state** with measurable outputs, ideally SLIs (error ratio, p99 latency, orders per minute).
2. **Hypothesize** that steady state continues in both control and experiment groups.
3. **Introduce real-world faults:** instance or pod loss, latency, packet loss, dependency errors, AZ failure, CPU or memory stress.
4. **Minimize blast radius:** start small (one pod, a canary, 1% of traffic) and in non-production first.
5. **Define abort conditions** and automate them.
6. **Run, observe, compare** with the hypothesis.
7. **Fix weaknesses** and repeat, eventually automating the experiment continuously.

</details>

### Q146. What is Chaos Mesh and what faults can it inject? 🟡

<details>
<summary><b>Answer</b></summary>

Chaos Mesh is a CNCF chaos platform for Kubernetes. A controller manager and a privileged `chaos-daemon` DaemonSet inject faults declared as CRDs:

- `PodChaos`: `pod-kill`, `pod-failure`, `container-kill`.
- `NetworkChaos`: `delay`, `loss`, `duplicate`, `corrupt`, `partition`, `bandwidth`.
- `StressChaos`: CPU and memory stress.
- `IOChaos`, `TimeChaos`, `DNSChaos`, `HTTPChaos`, `KernelChaos`, `JVMChaos`.
- `Schedule` for recurring experiments and `Workflow` for multi-step experiments.

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata: {name: kill-one-checkout, namespace: shop}
spec:
  action: pod-kill
  mode: one
  selector: {namespaces: [shop], labelSelectors: {app: checkout}}
```

</details>

### Q147. What is LitmusChaos and how does it differ from Chaos Mesh? 🟡

<details>
<summary><b>Answer</b></summary>

LitmusChaos is another CNCF chaos project for Kubernetes. Its model uses `ChaosExperiment` (the fault definition, often from ChaosHub), `ChaosEngine` (binds an experiment to an application) and `ChaosResult` (verdict). Its standout feature is **probes**: `httpProbe`, `cmdProbe`, `k8sProbe` and `promProbe` check steady-state during the run and decide a Pass/Fail verdict. ChaosCenter provides a UI and workflow builder. Chaos Mesh is simpler to start with (one CRD per fault, applied with kubectl); Litmus is stronger for verdict-driven, pipeline-integrated experiments.

</details>

### Q148. What is AWS Fault Injection Service (FIS)? 🟡

<details>
<summary><b>Answer</b></summary>

AWS FIS is a managed chaos service. You create an **experiment template** with **actions** (for example `aws:ec2:terminate-instances`, `aws:ec2:stop-instances`, `aws:rds:failover-db-cluster`, `aws:eks:pod-delete`, `aws:network:disrupt-connectivity`, `aws:fis:inject-api-throttle-error`), **targets** selected by ARN or tags with a selection mode like `COUNT(1)` or `PERCENT(25)`, an IAM role, and **stop conditions** tied to CloudWatch alarms. If the alarm fires, FIS stops the experiment automatically.

```bash
aws fis start-experiment --experiment-template-id EXT1a2b3c4d5e6f7
aws fis get-experiment --id EXP9z8y7x6w5v4u3
```

</details>

### Q149. What are stop conditions or abort criteria, and why are they mandatory? 🟡

<details>
<summary><b>Answer</b></summary>

Stop conditions automatically halt a chaos experiment or load test when real harm starts: an SLO burn-rate alarm, error ratio above a threshold, or a business metric (orders per minute) dropping. They limit damage if the hypothesis is wrong and make it safe to run experiments in production. In AWS FIS they are CloudWatch alarms in the template; in Litmus, failed probes in `Edge` mode stop and fail the run; in k6, thresholds with `abortOnFail: true`.

</details>

### Q150. What is a game day? 🟢

<details>
<summary><b>Answer</b></summary>

A scheduled exercise where a team deliberately injects failure (AZ outage, database failover, dependency down) into a real or production-like system and practices detecting, responding and recovering, often with the full incident process. Goals: validate resilience, alerts, runbooks and failover, and train people. Plan it with a hypothesis, blast radius, abort criteria, roles and communication, then write up findings like a postmortem.

</details>

### Q151. What faults should you test first in a Kubernetes environment? 🟡

<details>
<summary><b>Answer</b></summary>

In rough order of value:

1. Pod kill for each critical deployment (are there enough replicas, readiness probes, graceful shutdown?).
2. Node drain or node failure (PodDisruptionBudgets, anti-affinity, topology spread).
3. Dependency latency (timeouts and circuit breakers).
4. Dependency errors or unavailability (fallbacks, retries).
5. DNS failure or slowness (CoreDNS is a frequent hidden single point of failure).
6. AZ loss (capacity headroom, zonal spread).
7. CPU and memory stress (limits, throttling, OOM behaviour).

</details>

### Q152. What is blast radius and how do you limit it in chaos experiments? 🟡

<details>
<summary><b>Answer</b></summary>

Blast radius is the portion of users or systems an experiment (or failure) can affect. Limit it with: selector scope (one pod with `mode: one`, a canary deployment, one namespace), percentage modes (`fixed-percent: 10`), short durations (`duration: "60s"`), running in one AZ or cell, during business hours with the team watching, and automatic stop conditions. Expand gradually as confidence grows.

</details>

### Q153. What is the difference between vertical and horizontal scaling, and when does each run out? 🟡

<details>
<summary><b>Answer</b></summary>

**Vertical** scaling adds resources to one instance (bigger node, more CPU/memory per pod); it is simple but hits hardware limits and usually needs a restart. **Horizontal** scaling adds more instances behind a load balancer; it scales further and improves redundancy, but needs stateless services or sharded state. Horizontal scaling runs out at shared bottlenecks: a single database primary, a lock, a rate-limited third-party API, connection limits.

</details>

### Q154. How does autoscaling relate to capacity planning? 🔴

<details>
<summary><b>Answer</b></summary>

Autoscaling (HPA, KEDA, Cluster Autoscaler, Karpenter, AWS Auto Scaling groups) handles short-term variation, but capacity planning is still needed because:

- Autoscalers react with a delay (metrics window, pod startup, node provisioning), so spikes need headroom.
- They hit limits: `maxReplicas`, cloud quotas, IP addresses in subnets, database connection limits.
- Some resources do not autoscale: databases, licenses, third-party rate limits.
- Cost and reserved-capacity commitments need forecasts.

Load test the autoscaling behaviour itself with a spike test.

</details>

### Q155. What is "chaos as continuous verification" and how do you get there? 🔴

<details>
<summary><b>Answer</b></summary>

Mature teams run experiments automatically and continuously, like tests, so regressions in resilience are caught: a new deploy that breaks graceful shutdown fails the next scheduled pod-kill experiment. Steps: start with manual game days, codify experiments as CRDs or FIS templates in Git, add SLO-based probes and stop conditions, run them on a schedule in staging (Chaos Mesh `Schedule`, Litmus cron workflows), then in production with small blast radius, and alert on experiment failures like any other test.

</details>

---

## Release Engineering & Reliability Patterns

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q156. Why are releases the biggest source of incidents, and what does SRE do about it? 🟢

<details>
<summary><b>Answer</b></summary>

Most production incidents are triggered by change: code deploys, config changes, infrastructure changes, feature flag flips. SRE reduces the risk with small, frequent releases (smaller diff, easier rollback), progressive rollout (canary, region by region), automated analysis against SLIs, fast and tested rollback, feature flags to decouple deploy from release, and change freezes tied to error budget or peak events.

</details>

### Q157. What is a canary release? 🟢

<details>
<summary><b>Answer</b></summary>

Deploying a new version to a small subset of traffic or instances (for example 5%), comparing its error rate and latency against the stable version, and gradually increasing traffic only if it is healthy. If metrics degrade, traffic shifts back to stable automatically. It limits the blast radius of a bad release to a small fraction of users for a short time.

</details>

### Q158. What is progressive delivery? 🟡

<details>
<summary><b>Answer</b></summary>

An umbrella term for releasing changes gradually with automated control: canaries, blue-green with analysis, feature flag rollouts by cohort, and region-by-region deploys, each gated on metrics. Tools: **Argo Rollouts** (`Rollout` CRD with `canary` steps and `AnalysisTemplate` queries), **Flagger** (automates canaries with Istio, Linkerd, NGINX, Gateway API), and feature flag platforms (LaunchDarkly, Unleash, Flagsmith, OpenFeature SDKs).

</details>

### Q159. Show an Argo Rollouts canary with a Prometheus analysis. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: {name: checkout}
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - analysis: {templates: [{templateName: success-rate}]}
        - setWeight: 50
        - pause: {duration: 10m}
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: {name: success-rate}
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      failureLimit: 1
      successCondition: result[0] >= 0.999
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{job="checkout",code!~"5.."}[2m]))
            / sum(rate(http_requests_total{job="checkout"}[2m]))
```

The Rollout spec also needs `selector` and `template` like a Deployment. If the analysis fails, Argo Rollouts aborts and shifts traffic back to stable.

</details>

### Q160. Blue-green vs canary: when do you choose each? 🟡

<details>
<summary><b>Answer</b></summary>

- **Blue-green:** run the full new version alongside the old, switch all traffic at once, keep the old one for instant rollback. Good for simple stacks and when mixing versions is hard; costs double capacity, and the switch exposes 100% of users at once.
- **Canary:** shift traffic gradually with analysis. Smaller blast radius and real-traffic validation, but needs traffic splitting and both versions must coexist (compatible APIs and schemas).

Many teams use canary for stateless services and blue-green for things like database-heavy or batch systems.

</details>

### Q161. What are feature flags and how do they improve reliability? 🟢

<details>
<summary><b>Answer</b></summary>

Feature flags are runtime switches that turn code paths on or off without a deploy. They separate deploying code from releasing a feature: code ships dark, then the flag ramps to internal users, 1%, 10%, 100%. If something goes wrong, turning the flag off is a seconds-long mitigation, much faster than a rollback. Flags also enable kill switches for expensive features during overload. Risks: flag debt (old flags never removed) and untested flag combinations.

</details>

### Q162. What is a rollback vs a roll-forward, and when do you choose each? 🟡

<details>
<summary><b>Answer</b></summary>

A **rollback** reverts to the previous known-good version; it should be the default mitigation because it is fast and well-tested. A **roll-forward** ships a new fix. Roll forward only when rollback is impossible or more dangerous, for example after an irreversible database migration or when the previous version has a security vulnerability. To keep rollback possible, make schema changes backward compatible (expand/contract) and keep the previous artifacts and configs available.

</details>

### Q163. What is the expand/contract (parallel change) pattern for database migrations? 🔴

<details>
<summary><b>Answer</b></summary>

Split breaking schema changes into backward-compatible steps:

1. **Expand:** add the new column or table; old code ignores it.
2. Deploy code that writes to both old and new, and backfill existing rows.
3. Deploy code that reads from the new.
4. **Contract:** stop writing the old column, then drop it in a later release.

At every step the previous code version still works, so any deploy can be rolled back safely.

</details>

### Q164. Why should every network call have a timeout, and how do you choose the value? 🟢

<details>
<summary><b>Answer</b></summary>

Without a timeout, a hung dependency holds threads, connections and memory until the caller runs out and fails too, spreading the outage. Pick timeouts from the dependency's measured latency (a bit above its p99.9) and the caller's own budget: the total of downstream timeouts plus retries must fit inside the caller's timeout. Set connect timeouts short (hundreds of milliseconds) and request timeouts based on the operation. Never rely on library defaults; many are infinite or minutes long.

</details>

### Q165. What are deadlines and deadline propagation? 🔴

<details>
<summary><b>Answer</b></summary>

A deadline is an absolute time by which the whole request must finish. Instead of each hop having an independent timeout, the remaining budget is passed downstream (gRPC does this natively with `grpc-timeout`; HTTP services can use a header). Each service checks the remaining time and stops work early if it cannot finish, rather than doing work whose result nobody will receive. This prevents wasted load during slowdowns and keeps total latency bounded.

</details>

### Q166. How should retries be implemented safely? 🟡

<details>
<summary><b>Answer</b></summary>

- Retry only **transient** errors (timeouts, 503, connection reset), never 4xx client errors.
- Retry only **idempotent** operations, or use idempotency keys for writes.
- Use **exponential backoff with jitter**, a small **max attempts** (2 or 3), and an overall deadline.
- Retry at **one layer** of the stack, not every layer.
- Use a **retry budget** (for example retries at most 10% of requests) so retries cannot multiply load during an outage.
- Respect `Retry-After` headers.

</details>

### Q167. Why is jitter important in retry backoff? 🟡

<details>
<summary><b>Answer</b></summary>

Without jitter, all clients that failed at the same moment retry at the same moments (1s, 2s, 4s), creating synchronized waves that keep hammering a recovering service: the thundering herd. Jitter randomizes the wait so retries spread out. "Full jitter" picks a random delay between 0 and the exponential cap:

```python
import random
def backoff(attempt, base=0.1, cap=5.0):
    return random.uniform(0, min(cap, base * 2 ** attempt))
```

</details>

### Q168. What is a retry storm and how does it happen? 🔴

<details>
<summary><b>Answer</b></summary>

When a dependency slows down, every layer above it times out and retries, multiplying load on the struggling service exactly when it can least handle it. With three layers each making 3 attempts, one user request can turn into `3 x 3 x 3 = 27` calls to the bottom service. The dependency never recovers because retry traffic keeps it saturated (a metastable failure). Prevention: retry at one layer, retry budgets, circuit breakers, backoff with jitter, and load shedding at the dependency.

</details>

### Q169. What is a circuit breaker? 🟢

<details>
<summary><b>Answer</b></summary>

A circuit breaker wraps calls to a dependency and tracks failures. States:

- **Closed:** calls pass through; failures are counted.
- **Open:** after failures exceed a threshold, calls fail immediately (or use a fallback) without hitting the dependency, for a cool-down period.
- **Half-open:** after cool-down, a few trial calls are allowed; success closes the circuit, failure opens it again.

It protects the dependency from extra load and the caller from waiting on timeouts. Implementations: Resilience4j (Java), Polly (.NET), Envoy/Istio outlier detection and connection pool limits.

</details>

### Q170. How does Istio implement circuit breaking? 🔴

<details>
<summary><b>Answer</b></summary>

Through a `DestinationRule`: `connectionPool` limits concurrent connections and pending requests (excess gets an immediate 503 with response flag `UO`, upstream overflow), and `outlierDetection` ejects hosts that return consecutive errors.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: {name: inventory}
spec:
  host: inventory.shop.svc.cluster.local
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

</details>

### Q171. What is the bulkhead pattern? 🔴

<details>
<summary><b>Answer</b></summary>

Named after ship compartments: isolate resources so a failure in one part cannot sink the whole. In services, give each dependency its own thread pool, connection pool or concurrency limit, so a slow recommendation service can only exhaust its own 20 threads, not the 200 that serve checkout. At infrastructure level, separate node pools, separate clusters for critical workloads, and cell-based architectures are bulkheads too.

</details>

### Q172. What is graceful degradation? Give examples. 🟢

<details>
<summary><b>Answer</b></summary>

Serving a reduced but useful experience when a component fails or the system is overloaded, instead of an error:

- Show cached prices or product data when the pricing service is slow.
- Hide recommendations or reviews when those services fail.
- Serve a static page from the CDN when the backend is down.
- Disable expensive search filters during overload.
- Queue writes (orders) for later processing instead of rejecting them.

Design degradation modes in advance and test them with chaos experiments.

</details>

### Q173. What is the difference between rate limiting and load shedding? 🟡

<details>
<summary><b>Answer</b></summary>

- **Rate limiting** caps how many requests a client, API key or tenant can make (for example 100 requests per second per key), returning 429. It enforces fairness and protects against abuse or bugs in one client.
- **Load shedding** drops excess work when the service itself is near capacity, regardless of who sent it, returning 503 quickly, ideally dropping low-priority requests first.

Both keep the service within capacity so it can serve the remaining requests within SLO, instead of everyone timing out.

</details>

### Q174. Explain token bucket rate limiting. 🟡

<details>
<summary><b>Answer</b></summary>

A bucket holds up to `B` tokens and refills at `R` tokens per second. Each request consumes a token; if the bucket is empty, the request is rejected (429) or delayed. It allows bursts up to `B` while enforcing an average rate `R`. NGINX's `limit_req` uses the closely related leaky bucket:

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
location /api/ {
    limit_req zone=api burst=20 nodelay;
    limit_req_status 429;
}
```

</details>

### Q175. What is a metastable failure? 🔴

<details>
<summary><b>Answer</b></summary>

A state where the system stays broken even after the trigger is gone, because a feedback loop sustains the overload. Example: a brief DB slowdown causes timeouts; clients retry; retries double the load; the DB stays slow; more timeouts. Or a cache flush causes all requests to hit the DB, which becomes too slow to refill the cache. Recovery usually needs shedding load hard (block traffic, disable retries, warm the cache slowly). Prevention: retry budgets, circuit breakers, load shedding, and request coalescing.

</details>

### Q176. What is idempotency and why does it matter for reliability? 🔴

<details>
<summary><b>Answer</b></summary>

An idempotent operation produces the same result when applied once or several times. It makes retries safe: retrying `PUT /orders/123` or a `DELETE` is harmless, but retrying a `POST /payments` could charge twice. For non-idempotent writes, clients send an idempotency key (for example a UUID in an `Idempotency-Key` header) and the server stores the result per key, returning the stored result for duplicates. Message consumers need the same property, because most queues guarantee at-least-once delivery.

</details>

### Q177. How do readiness probes and graceful shutdown contribute to release reliability on Kubernetes? 🔴

<details>
<summary><b>Answer</b></summary>

- **Readiness probes** keep a pod out of Service endpoints until it can serve, so rolling updates do not send traffic to pods still starting.
- **Graceful shutdown:** on termination Kubernetes sends SIGTERM; the app should stop accepting new work, finish in-flight requests and exit within `terminationGracePeriodSeconds`.
- A `preStop` sleep of a few seconds covers the delay before endpoint removal propagates to kube-proxy and ingress controllers.

```yaml
lifecycle:
  preStop:
    exec: {command: ["sleep", "10"]}
terminationGracePeriodSeconds: 40
```

Together with `maxUnavailable: 0` and a PodDisruptionBudget, deploys and node drains cause zero errors.

</details>

### Q178. What is a change freeze and is it a good practice? 🟡

<details>
<summary><b>Answer</b></summary>

A period where non-essential changes are blocked: before a big sales event, during holidays when staffing is thin, or when the error budget is exhausted. Used deliberately and briefly, it reduces risk at critical times. Overused, it creates big-bang releases afterwards (larger, riskier diffs). SRE prefers budget-based freezes (data-driven) plus extra safeguards (slower canaries, more approvals) over long calendar freezes.

</details>

---

## Backup, DR, Runbooks & SRE Metrics

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q179. What are RTO and RPO? 🟢

<details>
<summary><b>Answer</b></summary>

- **RTO (Recovery Time Objective):** the maximum acceptable time to restore service after a disaster. "Checkout must be back within 1 hour."
- **RPO (Recovery Point Objective):** the maximum acceptable data loss, measured in time. "We can lose at most 5 minutes of orders."

RPO drives backup frequency and replication (hourly snapshots give RPO up to 1 hour; synchronous replication gives near-zero). RTO drives the DR architecture and automation. Both are business decisions with cost trade-offs.

</details>

### Q180. Describe the four classic DR strategies. 🟡

<details>
<summary><b>Answer</b></summary>

| Strategy | How it works | RTO / RPO | Cost |
|---|---|---|---|
| Backup and restore | Backups copied to another region; rebuild everything on disaster | Hours / hours | Lowest |
| Pilot light | Core data replicated live (DB replica); compute off or minimal, scaled up on disaster | Tens of minutes / minutes | Low |
| Warm standby | Scaled-down full copy running in DR region, scaled up on failover | Minutes / seconds to minutes | Medium |
| Multi-site active-active | Full capacity in several regions serving traffic | Near zero / near zero | Highest |

Pick per service based on its RTO/RPO, not one strategy for everything.

</details>

### Q181. Why is "we have backups" not enough? 🟢

<details>
<summary><b>Answer</b></summary>

A backup is only proven by a successful restore. Common failures: backups silently failing for weeks, backups that are incomplete (missing a schema or encryption key), restores that take far longer than the RTO, backups stored in the same account or region as the thing they protect, or backups deleted by the same ransomware or bad script. Automate regular restore tests, measure restore time, alert on backup job failures and age, and keep immutable copies (S3 Object Lock, AWS Backup Vault Lock) in a separate account.

</details>

### Q182. What is the 3-2-1 backup rule? 🟡

<details>
<summary><b>Answer</b></summary>

Keep at least **3** copies of data, on **2** different types of storage or services, with **1** copy offsite (another region or account). A modern extension is 3-2-1-1-0: one copy immutable or air-gapped, and zero errors in verified restores. In AWS terms: production data, snapshots in AWS Backup, and a cross-account, cross-region copy with Vault Lock.

</details>

### Q183. How do you back up a Kubernetes cluster? 🟡

<details>
<summary><b>Answer</b></summary>

- **Declarative state:** keep manifests in Git (GitOps), so the cluster can be rebuilt by re-syncing Argo CD or Flux.
- **etcd snapshots** for self-managed control planes: `etcdctl snapshot save /backup/etcd-$(date +%F).db`.
- **Velero** for namespaced resources and persistent volumes (via snapshots or file-level backup with Kopia):

```bash
velero backup create shop-daily --include-namespaces shop --ttl 720h
velero schedule create shop-nightly --schedule "0 2 * * *" --include-namespaces shop
velero restore create --from-backup shop-daily
```

- **Databases:** use database-native backups (pg_dump, WAL archiving, RDS snapshots), not just volume snapshots, for consistency.

</details>

### Q184. What is a runbook and what makes a good one? 🟢

<details>
<summary><b>Answer</b></summary>

A runbook is a documented procedure for handling a specific alert or operational task. A good runbook:

- Is linked directly from the alert.
- Explains what the alert means and the user impact.
- Gives commands to confirm the problem and check impact (copy-pasteable, with dashboard links).
- Lists mitigations in order of safety.
- Says when and to whom to escalate.
- Has an owner and a "last tested" date.

Short and current beats long and stale.

</details>

### Q185. What is the difference between a runbook and a playbook? 🟢

<details>
<summary><b>Answer</b></summary>

Usage varies, but commonly a **runbook** is a specific, step-by-step procedure for one alert or task ("CheckoutDBConnectionsHigh: check pool metrics, kill idle-in-transaction sessions, scale pool"). A **playbook** is broader guidance for a class of situations ("how we run a SEV-1", "how we respond to a suspected data breach"), covering roles, communication and decision-making rather than commands.

</details>

### Q186. How do runbooks evolve into automation? 🔴

<details>
<summary><b>Answer</b></summary>

1. Write the manual runbook during or after an incident.
2. Turn repeated command sequences into scripts referenced from the runbook.
3. Make the scripts safe: dry-run, confirmation, idempotent, logged.
4. Expose them as self-service or ChatOps commands (Rundeck, AWS Systems Manager Automation, Slack bot).
5. Trigger them automatically from alerts when they are proven safe (auto-remediation), with rate limits and a page if they fail or run too often.

Automated remediation that runs frequently is hiding a bug; track it and fix the underlying issue.

</details>

### Q187. Define MTTD, MTTA, MTTR and MTBF. 🟢

<details>
<summary><b>Answer</b></summary>

- **MTTD (mean time to detect):** from incident start to detection (alert fires or report received).
- **MTTA (mean time to acknowledge):** from alert to a human acknowledging it.
- **MTTR (mean time to restore/recover/resolve):** from start to service restored. Define which variant you use.
- **MTBF (mean time between failures):** average time between incidents.

Reduce MTTD with SLO burn alerts and synthetic monitoring; reduce MTTR with fast rollback, runbooks and practice.

</details>

### Q188. Why are mean-based incident metrics (MTTR) criticized? 🔴

<details>
<summary><b>Answer</b></summary>

Incident durations have a long-tailed distribution: most are short, a few are very long. The mean is dominated by outliers, swings wildly quarter to quarter with small samples, and does not tell you whether typical incidents got better. Comparing MTTR across teams is also misleading because incident definitions differ. Better: report median and p90, show the distribution, and pair numbers with qualitative learning from postmortems. Some practitioners argue to drop MTTR as a target entirely and focus on the learning.

</details>

### Q189. What are the DORA metrics? 🟢

<details>
<summary><b>Answer</b></summary>

From the DevOps Research and Assessment program (Accelerate / State of DevOps reports):

- **Deployment frequency:** how often you deploy to production.
- **Lead time for changes:** commit to running in production.
- **Change failure rate:** percentage of deployments that cause a failure needing remediation.
- **Failed deployment recovery time** (formerly "time to restore service"): how long it takes to recover from a failed deployment.

The first two measure throughput, the last two stability. The research shows high performers are good at both; recent reports also discuss rework rate as an additional stability signal.

</details>

### Q190. How do you measure change failure rate in practice? 🔴

<details>
<summary><b>Answer</b></summary>

```text
change failure rate = deployments that caused an incident, rollback or hotfix / total deployments
```

Implementation: record every deployment as an event (CI/CD webhook, Argo CD notifications, deployment annotations in Grafana), and link incidents and rollbacks to the deployment that triggered them (incident tool field, or a rollback event referencing the version). Count per service per month. Ambiguity about "what counts as failure" must be decided up front and documented.

</details>

### Q191. How do SRE metrics and DORA metrics relate? 🟡

<details>
<summary><b>Answer</b></summary>

DORA measures delivery performance; SRE metrics measure service reliability and response. They connect through change: change failure rate shows how much risk each deploy carries, failed deployment recovery time overlaps with MTTR, and error budget burn attributable to deploys tells you whether release safety (canaries, flags) is working. A team with high deployment frequency, low change failure rate and healthy SLOs is the goal: speed and stability reinforce each other.

</details>

### Q192. How does observability drive reliability work beyond alerting? 🟡

<details>
<summary><b>Answer</b></summary>

- **SLO dashboards** prioritize work: which journeys burn the most budget and why.
- **Traces** show which dependency dominates latency and where to add caching or timeouts.
- **Error analytics** (top errors by endpoint, version, region) target fixes with the most user impact.
- **Deploy annotations** correlate regressions with releases.
- **Saturation trends** feed capacity planning.
- **Postmortems** add missing telemetry as action items, so the next incident is diagnosed faster.

The rule: every incident should leave the system more observable than before.

</details>

---

## Best Practices & Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q193. What are your top SLO and alerting best practices? 🟡

<details>
<summary><b>Answer</b></summary>

- One to three SLOs per critical user journey, measured close to the user.
- SLIs as good/valid ratios; latency via histogram buckets, never averages.
- Recording rules for every window; SLO definitions in Git, generated with Sloth or Pyrra.
- Multi-window, multi-burn-rate alerts: page on 14.4x and 6x, ticket on 3x and 1x.
- Every alert has an owner, severity label, runbook URL and dashboard link.
- `promtool check rules` and `promtool test rules` in CI.
- A Watchdog dead man's switch and a traffic-drop alert to cover the blind spots.
- Quarterly SLO review with product.

</details>

### Q194. What are your top incident-management best practices? 🟡

<details>
<summary><b>Answer</b></summary>

- Clear, impact-based severity definitions with examples.
- Declare early; anyone may declare.
- Assign IC, Ops and Comms explicitly; the IC does not debug.
- One incident channel, live timeline, fixed update cadence.
- Mitigate first: rollback, flag off, failover.
- Blameless postmortem for every SEV-1/2 within a week, with owned action items tracked to done.
- Practice: wheel of misfortune, game days, DR drills.

</details>

### Q195. Rapid-fire: Prometheus and promtool commands every SRE should know. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
promtool check rules slo-rules.yaml                 # syntax and validity
promtool test rules slo-rules.test.yaml             # unit test alerts
promtool check config prometheus.yml
promtool query instant http://localhost:9090 'slo:sli_error:ratio_rate1h'
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].name'
curl -s localhost:9090/api/v1/alerts | jq '.data.alerts[] | {alertname: .labels.alertname, state}'
curl -X POST localhost:9090/-/reload                # needs --web.enable-lifecycle
```

</details>

### Q196. Rapid-fire: Alertmanager and amtool commands. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
amtool check-config alertmanager.yml
amtool --alertmanager.url=http://localhost:9093 alert query severity=page
amtool --alertmanager.url=http://localhost:9093 silence add alertname=CheckoutErrorBudgetBurnFast \
  --duration=1h --comment="INC-482 rollback in progress" --author="oncall"
amtool --alertmanager.url=http://localhost:9093 silence query
amtool --alertmanager.url=http://localhost:9093 silence expire <silence-id>
amtool config routes test --config.file=alertmanager.yml severity=page slo=checkout-availability
```

</details>

### Q197. Rapid-fire: k6 commands and flags. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
k6 run script.js                                  # run with options from script
k6 run --vus 20 --duration 2m script.js           # override load
k6 run -e BASE_URL=http://checkout:8080 script.js # pass env, read via __ENV.BASE_URL
k6 run --out json=results.json script.js          # raw metrics
K6_PROMETHEUS_RW_SERVER_URL=http://prometheus:9090/api/v1/write \
  k6 run -o experimental-prometheus-rw script.js  # stream to Prometheus remote write
k6 inspect script.js                              # show resolved options
echo $?                                           # 99 means thresholds failed
```

</details>

### Q198. Rapid-fire: Chaos Mesh and kubectl commands for chaos experiments. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl apply -f podchaos-kill.yaml
kubectl get podchaos,networkchaos,stresschaos -A
kubectl describe networkchaos inventory-latency -n shop   # check status.conditions and events
kubectl annotate networkchaos inventory-latency -n shop experiment.chaos-mesh.org/pause=true
kubectl delete networkchaos inventory-latency -n shop     # stop and recover
kubectl get pods -n chaos-mesh                             # controller-manager, chaos-daemon, dashboard
kubectl logs -n chaos-mesh ds/chaos-daemon --tail=50
```

</details>

### Q199. Rapid-fire: commands to check service health quickly during an incident on Kubernetes. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl rollout history deploy/checkout -n shop
kubectl rollout undo deploy/checkout -n shop            # fastest mitigation after a bad deploy
kubectl rollout status deploy/checkout -n shop
kubectl get events -n shop --sort-by=.lastTimestamp | tail -20
kubectl top pods -n shop --sort-by=cpu
kubectl get endpointslices -n shop -l kubernetes.io/service-name=checkout
kubectl scale deploy/checkout -n shop --replicas=10
kubectl argo rollouts abort checkout -n shop            # abort a canary
```

</details>

### Q200. Rapid-fire: what is the one number you'd put on a service's front-page dashboard, and why? 🟢

<details>
<summary><b>Answer</b></summary>

Error budget remaining for its most important SLO (with the current burn rate next to it). It answers in one glance: are users getting the reliability we promised, and how much risk can we take right now? Everything else (RPS, CPU, pod counts, p99) is for diagnosis below the fold. If a manager, a product owner and the on-call engineer all look at the same number, reliability conversations become decisions instead of debates.

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
