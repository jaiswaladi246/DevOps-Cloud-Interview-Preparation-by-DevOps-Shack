# Python for DevOps — Scenarios & Troubleshooting

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

> Q1–Q100 are real-world "what would you do" situations; Q101–Q200 are real error messages with diagnosis and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents

**Scenarios**
- [Automation scripting & CLI design](#automation-scripting--cli-design) — Q1–Q25
- [AWS automation with boto3](#aws-automation-with-boto3) — Q26–Q50
- [Kubernetes, Docker, SSH & fleet automation](#kubernetes-docker-ssh--fleet-automation) — Q51–Q75
- [Testing, CI/CD, packaging & services](#testing-cicd-packaging--services) — Q76–Q100

**Troubleshooting**
- [Environments, imports & packaging errors](#environments-imports--packaging-errors) — Q101–Q125
- [Runtime, file, subprocess & encoding errors](#runtime-file-subprocess--encoding-errors) — Q126–Q150
- [boto3, HTTP & API errors](#boto3-http--api-errors) — Q151–Q175
- [Concurrency, async, tests, services & containers](#concurrency-async-tests-services--containers) — Q176–Q200

---

## Automation scripting & CLI design

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Your team has a 300-line Bash script that parses JSON with `jq`, loops over AWS accounts and retries with `sleep`. It breaks every month. Do you rewrite it in Python? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The script has outgrown Bash: data structures, API calls, retries and error handling are all things Bash does badly.

**What I'd do →**
1. Write down what it actually does today (inputs, outputs, side effects, exit codes) and capture a sample run's output as a reference.
2. Rewrite in Python with boto3, a click CLI, logging and built-in retries (`Config(retries={"mode": "adaptive"})`), keeping the same flags and output so callers do not change.
3. Add pytest tests with moto for the core logic and compare output against the reference run.
4. Ship it side by side for one cycle, then switch callers over.

**Why →** A behaviour-preserving rewrite with tests removes the monthly breakage without surprising the people who depend on it.

**Interview tip →** Say you keep the external contract identical; rewrites fail when they change behaviour and tooling at the same time.

</details>

### Q2. A colleague's script deletes files older than 30 days from `/var/log/app`. You are asked to review it before it goes into cron. What do you look for? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Destructive script, unattended execution. Review for blast radius and safety.

**What I'd do →**
1. Check the path is a constant or validated, never empty: a bug that turns the base into `/` or `""` must abort.
2. Check it uses `Path.rglob` with explicit patterns (`*.log*`) and skips symlinks (`p.is_symlink()`), so it cannot follow a link out of the directory.
3. Confirm age uses `st_mtime` against timezone-consistent `time.time()`.
4. Require a `--dry-run` default or flag, log every deletion, and exit non-zero on errors.
5. Catch `PermissionError` per file and continue, instead of aborting halfway.
6. Ask whether `logrotate` already does this, which is usually the better answer.

**Why →** Cleanup scripts fail dangerously at their edges: empty variables, symlinks and permissions.

**Interview tip →** Mention `logrotate` or `find -mtime +30 -delete`; not every problem needs Python.

</details>

### Q3. You need a script that ops engineers run from laptops and that CI runs nightly. How do you design its interface? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two audiences: humans (readable output, prompts) and machines (stable output, exit codes, no prompts).

**What I'd do →**
1. click CLI with subcommands, every option settable via env var (`envvar=`).
2. `--format table|json`; default to table when `sys.stdout.isatty()` and JSON otherwise.
3. Logs on stderr, data on stdout; `-v/-vv` for verbosity.
4. Documented exit codes: 0 clean, 1 error, 2 findings.
5. No interactive prompt when stdin is not a TTY; destructive actions require `--yes`.
6. Package it with a console script so both laptop (`pipx install`) and CI (`docker run`) use the same entry point.

**Why →** One tool, two consumers, no forked logic.

**Interview tip →** "stdout is an API" is the phrase interviewers like to hear.

</details>

### Q4. A script reads a 15 GB access log to count 5xx errors per endpoint, and the box runs out of memory. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** It almost certainly does `f.read()` or `readlines()`, loading the whole file.

**What I'd do →**
1. Iterate the file object line by line (`for line in f`) and update a `Counter`.
2. Use `gzip.open(path, "rt")` for compressed rotations instead of decompressing to disk.
3. Precompile the regex, or use `line.split()` if the format is fixed.
4. Open with `errors="replace"` so a single bad byte does not kill the run.

```python
codes = Counter()
with open(path, encoding="utf-8", errors="replace") as f:
    for line in f:
        parts = line.split()
        if len(parts) > 8 and parts[8].startswith("5"):
            codes[parts[6]] += 1
```

**Why →** Streaming keeps memory constant regardless of file size.

**Interview tip →** Mention that for recurring analysis the right fix is to ship logs to a system built for it (Loki, CloudWatch Logs Insights, Athena).

</details>

### Q5. You must run the same Python automation on 40 servers that have Python 3.6, 3.9 and 3.12. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Python 3.6 is long end-of-life, and the modern boto3/requests versions no longer support it.

**What I'd do →**
1. Do not depend on the system Python at all. Options: run the tool in a container, ship a self-contained venv built with the right version (`uv python install 3.12` + venv), or use a single-file zipapp built with `shiv`/`pex` targeting a known interpreter.
2. If agentless remote execution is the real need, use Ansible or SSM Run Command and keep the Python on the control side.
3. Open a ticket to upgrade the 3.6 hosts; they are also missing security fixes.

**Why →** Writing to the lowest common denominator freezes you on dead versions and unsupported libraries.

**Interview tip →** Say "the OS Python belongs to the OS"; ship your own runtime.

</details>

### Q6. A cron job that runs a Python script fails silently. Nobody noticed for two weeks. How do you make sure that never happens again? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two problems: the failure itself and the lack of observability.

**What I'd do →**
1. Make the script exit non-zero on any failure (no catch-all `except: pass`).
2. In crontab, call the venv interpreter explicitly and capture output: `0 2 * * * /opt/tool/.venv/bin/python -m tool >> /var/log/tool.log 2>&1`.
3. Add a dead man's switch: on success, ping Healthchecks.io/Cronitor or push `last_success_timestamp_seconds` to a Pushgateway, and alert when it is stale.
4. Better: move it to a systemd timer (journal logs, `OnFailure=` unit) or a Kubernetes CronJob with alerting on failed jobs.

**Why →** Alerting on absence of success catches every failure mode, including "never started".

**Interview tip →** "Alert on missing success, not just on errors" shows real on-call experience.

</details>

### Q7. Product wants a Slack notification whenever a deploy finishes. You have 30 minutes. What do you build? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Small, well-scoped integration; the risk is leaking the webhook URL or blocking the deploy on Slack.

**What I'd do →**
1. Create a Slack incoming webhook; store the URL as a CI secret.
2. Add a tiny script or step:

```python
import os, requests
resp = requests.post(os.environ["SLACK_WEBHOOK_URL"],
                     json={"text": f":rocket: {os.environ['SERVICE']} {os.environ['VERSION']} deployed to prod"},
                     timeout=5)
resp.raise_for_status()
```
3. Run it with `if: always()` and report success or failure; make the notify step non-blocking (`continue-on-error: true`).

**Why →** A notification must never fail a successful deploy.

**Interview tip →** Mention keeping the webhook URL secret; it is a credential.

</details>

### Q8. You inherit a script with hardcoded AWS keys, a hardcoded account ID and a hardcoded region. What is your clean-up plan? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A credential leak (the keys are in Git history) and an inflexible design.

**What I'd do →**
1. Treat the keys as compromised: deactivate and delete them in IAM now, check CloudTrail for misuse.
2. Remove them from code; rely on the default credential chain (profile, SSO, instance role, OIDC in CI).
3. Make region and account inputs: `--region`, `--profile`, or derive the account with `sts.get_caller_identity()`.
4. Purge history if the repo is shared (`git filter-repo`), and add `gitleaks`/push protection to prevent recurrence.

**Why →** Removing keys from the latest commit does not remove them from history or from anyone's clone.

**Interview tip →** Lead with "rotate first, then clean the code"; order matters.

</details>

### Q9. A teammate keeps adding flags to a single script until it has 34 options. How do you restructure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One script is doing several jobs; options interact in confusing ways.

**What I'd do →**
1. Group functionality into subcommands (`tool snapshots cleanup`, `tool ec2 audit`) with a click group.
2. Move shared options (profile, region, verbosity, format) to the group level and pass them via `ctx.obj`.
3. Put rarely changed settings in a config file with env-var overrides.
4. Split the code into modules per subcommand, with a thin CLI layer.

**Why →** Subcommands make the help text navigable and isolate options that only make sense together.

**Interview tip →** Mention that `kubectl` and `aws` are the model: verb/noun subcommands with global flags.

</details>

### Q10. Your script must process 50,000 S3 keys and must be resumable if it crashes midway. How do you design it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-running batch, partial failure is likely, reprocessing must be safe.

**What I'd do →**
1. Make per-item work idempotent (re-copying or re-tagging the same object is harmless).
2. Record progress durably: a checkpoint file written atomically every N items with the last processed key (S3 lists keys in lexicographic order, so `StartAfter=last_key` resumes the listing), or a DynamoDB/SQLite table of done keys.
3. Handle SIGTERM: finish the current item, write the checkpoint, exit.
4. Collect failures in a separate file for retry instead of aborting the run.
5. For bigger jobs, use S3 Batch Operations or fan out via SQS workers.

**Why →** Checkpoint plus idempotency means a crash costs minutes, not a full rerun.

**Interview tip →** Mention `StartAfter`; it shows you know how S3 listing works.

</details>

### Q11. Someone asks you to "just add `shell=True`" because a pipe in `subprocess.run` does not work. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The command includes user-controlled input; `shell=True` would open command injection.

**What I'd do →**
1. Do the pipe in Python: run the first command, process `stdout` in Python (often replacing `grep`/`awk` entirely).
2. If both commands must stay, chain two `Popen` objects with `stdout=PIPE` and `stdin=p1.stdout`.
3. If shell features are truly required with a constant command, use `shell=True` only with values passed through `shlex.quote()`.

```python
p1 = subprocess.Popen(["journalctl", "-u", unit, "--since", "1h ago"], stdout=subprocess.PIPE)
p2 = subprocess.run(["grep", "-c", "ERROR"], stdin=p1.stdout, capture_output=True, text=True)
p1.stdout.close(); p1.wait()
```

**Why →** Avoiding the shell eliminates the injection class instead of trying to sanitize it.

**Interview tip →** Say "list arguments, no shell" as the default rule.

</details>

### Q12. You need to update a value in 200 YAML files across a repo (bump an image tag) without destroying comments and formatting. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PyYAML `safe_load`/`safe_dump` would drop comments, reorder keys and change quoting, making a huge noisy diff.

**What I'd do →**
1. Use `ruamel.yaml` in round-trip mode, which preserves comments, ordering and most formatting.
2. Iterate files with `Path.rglob("*.yaml")`, use `load_all` for multi-doc manifests, change only matching keys, and write back only if something changed.
3. Run it on a branch, review `git diff --stat`, and let CI validate (`kubeconform`, `helm lint`).
4. If the files are Kustomize overlays, `kustomize edit set image` may be simpler.

**Why →** Minimal diffs are reviewable diffs.

**Interview tip →** Naming ruamel.yaml for round-trip editing is a strong signal.

</details>

### Q13. A script works on your Mac but fails in the Linux CI runner with odd path and encoding errors. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic environment drift: case sensitivity, encodings, tool versions.

**What I'd do →**
1. Case sensitivity: macOS's filesystem is case-insensitive by default; `import Config` vs `config.py` works on Mac and fails on Linux.
2. Encoding: always pass `encoding="utf-8"`; CI containers may have no locale set.
3. Hardcoded paths (`/Users/...`, `~`): use `Path.home()` or config.
4. Different Python and dependency versions: pin with a lock file and the same Python version (or run tests in the same container image locally).
5. BSD vs GNU tools in `subprocess` calls (`sed -i` differs).

**Why →** Most "works on my machine" bugs are one of these five.

**Interview tip →** Suggest running tests in the same Docker image locally and in CI.

</details>

### Q14. You need to rotate a credential stored in 12 places (Secrets Manager, two Kubernetes secrets, a Jenkins credential and a config file). How would you automate it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-system change; a half-finished rotation is an outage.

**What I'd do →**
1. Reduce the 12 places to 1: Secrets Manager as source of truth, with External Secrets Operator syncing into Kubernetes and Jenkins reading from it.
2. Script the rotation as phases: create new credential (both valid), update the source of truth, wait for consumers to pick it up (verify via health checks), revoke the old one.
3. Make each step idempotent and log state so a rerun continues from where it stopped.
4. Prefer Secrets Manager's built-in rotation Lambda for supported databases.

**Why →** The dual-validity window is what makes rotation zero-downtime.

**Interview tip →** Say you would fix the sprawl first; automating 12 copies is automating the problem.

</details>

### Q15. Your manager asks for a weekly CSV of every EC2 instance, its owner tag and monthly cost across 8 accounts. How do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-account inventory plus cost data. EC2 APIs do not return cost.

**What I'd do →**
1. Assume a read-only role in each account (`sts.assume_role`) from a central tooling account.
2. List instances per region with paginators, building rows with `InstanceId`, type, state, `Owner` tag (or `"MISSING"`).
3. Get cost from Cost Explorer (`ce.get_cost_and_usage` grouped by a cost allocation tag or resource-level data) or from the CUR in Athena; do not estimate from price lists if real data exists.
4. Write with `csv.DictWriter`, upload to S3, email or post the link.
5. Schedule weekly in CI with OIDC.

**Why →** Inventory from APIs plus cost from billing data is accurate and auditable.

**Interview tip →** Mention activating the `Owner` cost allocation tag; without that, cost by owner is impossible.

</details>

### Q16. Your Python tool takes 9 seconds just to print `--help`. Users complain. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Heavy top-level imports or work at import time (clients created, network calls).

**What I'd do →**
1. Measure: `python -X importtime -m tool --help 2>&1 | sort -t'|' -k2 -n | tail -20`.
2. Move expensive imports (boto3, kubernetes, pandas) inside the subcommand functions that use them.
3. Remove any import-time side effects: no clients, config loading or API calls at module level.
4. Check the venv is not on a slow network filesystem.

**Why →** `--help` should cost only argument parsing; lazy imports make every command start fast.

**Interview tip →** `-X importtime` is the tool to name.

</details>

### Q17. You must call an internal API that sometimes returns 503 for a few seconds during deploys. Your script fails randomly. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Transient errors; the client should retry with backoff, not fail.

**What I'd do →**
1. Use a `requests.Session` with an `HTTPAdapter(max_retries=Retry(total=5, backoff_factor=1, status_forcelist=(502, 503, 504), respect_retry_after_header=True))`.
2. Keep explicit timeouts (`timeout=(3, 10)`).
3. Only retry idempotent methods automatically.
4. Log each retry at WARNING so persistent problems are visible.

**Why →** A few seconds of backoff absorbs deploy blips without masking real outages, because retries are bounded.

**Interview tip →** Also suggest the API team fix it with readiness probes and connection draining.

</details>

### Q18. You need to template 50 config files from a list of services. Python string formatting is getting messy. What do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic templating problem.

**What I'd do →**
1. Use Jinja2 (the same engine as Ansible and many Helm-like tools): templates with loops and conditionals, data from YAML.
2. Enable `undefined=StrictUndefined` so a missing variable is an error, not an empty string.
3. Validate output (parse generated YAML/JSON, run `nginx -t`) before writing.
4. Write only changed files, so diffs and reloads are minimal.

```python
env = jinja2.Environment(loader=jinja2.FileSystemLoader("templates"), undefined=jinja2.StrictUndefined)
out = env.get_template("nginx.conf.j2").render(svc=svc)
```

**Why →** Templates separate structure from data and fail loudly on missing values.

**Interview tip →** Mention `StrictUndefined`; silent empty values are the classic template bug.

</details>

### Q19. A script runs for 6 hours and you want to know where the time goes without restarting it. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need live introspection of a running process.

**What I'd do →**
1. `py-spy top --pid <pid>` to see which functions are consuming time right now.
2. `py-spy dump --pid <pid>` to see each thread's current stack; repeated dumps show if it is stuck on the same call (a network call with no timeout is the usual culprit).
3. `py-spy record -o profile.svg --pid <pid> --duration 60` for a flame graph.
4. Next run: add timing logs per phase and a progress counter.

**Why →** py-spy samples from outside the process with no code changes or restart.

**Interview tip →** "Six hours" is often a missing timeout or sequential API calls; mention both.

</details>

### Q20. You are asked to write a "self-healing" script that restarts a service when its health check fails. What design concerns do you raise? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auto-remediation can hide problems or cause restart storms.

**What I'd do →**
1. Prefer platform features first: systemd `Restart=on-failure` with `StartLimitBurst`, Kubernetes liveness probes, ASG health checks.
2. If a script is still needed: require N consecutive failures before acting, add a cooldown, cap restarts per hour, and alert a human after the cap.
3. Log and emit a metric for every remediation so frequency is visible.
4. Never remediate on a dependency failure (restarting the app does not fix the DB).

**Why →** Unbounded auto-restart turns a degraded service into a flapping one.

**Interview tip →** Say every auto-remediation must also notify; silent healing hides the root cause.

</details>

### Q21. A security review flags that your tool uses `pickle` to cache API responses on disk. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `pickle.load` on a file someone else can write is arbitrary code execution.

**What I'd do →**
1. Switch the cache to JSON (or `sqlite3`), which is data-only.
2. Store it in a user-owned directory with `0600` permissions (`platformdirs.user_cache_dir`).
3. Add expiry metadata so stale data is not trusted forever.

**Why →** JSON cannot execute code on load; pickle can.

**Interview tip →** Know the rule: never unpickle data you did not produce in a location only you can write.

</details>

### Q22. You need to compare the live state of 300 Route 53 records against a desired YAML file and report drift. How do you approach it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Desired vs actual reconciliation, report-only.

**What I'd do →**
1. Load desired records from YAML into a dict keyed by `(name, type)`, normalizing names (trailing dot, lowercase) and sorting values.
2. Fetch actual with the `list_resource_record_sets` paginator, normalized the same way.
3. Compute set differences: missing, extra, changed (compare TTL and sorted values).
4. Output a diff report (Markdown + JSON) and exit 2 if drift exists.
5. Recommend moving the zone into Terraform so drift detection is `terraform plan`.

**Why →** Normalization is the hard part; without it every record looks drifted.

**Interview tip →** Mention normalization explicitly; it is what separates a working diff from noise.

</details>

### Q23. Your CLI needs to show progress for a 20-minute operation, but it also runs in CI. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Progress bars are great in terminals and garbage in CI logs.

**What I'd do →**
1. Use `tqdm` or `rich.progress` only when `sys.stderr.isatty()`; otherwise log a line every N items or every 30 seconds.
2. Write progress to stderr, never stdout.
3. Respect `NO_COLOR` and a `--quiet` flag.

**Why →** Carriage-return progress bars produce thousands of log lines in CI.

**Interview tip →** TTY detection is the one-line answer.

</details>

### Q24. A new joiner asks why your team's scripts all use `logging.getLogger(__name__)` instead of the root logger. How do you explain it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Teaching moment about logger hierarchy.

**What I'd do →**
1. Explain that named loggers show which module emitted a line (`cloudaudit.checks`).
2. Levels can be tuned per module: `logging.getLogger("cloudaudit.checks").setLevel("DEBUG")` without drowning in botocore debug logs.
3. Libraries should never configure the root logger; the application does it once in `main()`.

**Why →** Named loggers give control and context for free.

**Interview tip →** Mention silencing `botocore` and `urllib3` to WARNING as a practical example.

</details>

### Q25. You need to run a one-off data fix against production through a Python script. How do you make it safe? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One-off scripts skip review and testing, and touch production directly.

**What I'd do →**
1. Code review the script like any change; check it into the repo under `oneoff/` with a ticket reference.
2. Dry-run mode that prints exactly what will change, with counts, and a hard cap (`--max-items`).
3. Back up or snapshot affected data first.
4. Run in a controlled environment (bastion or CI job with an audited role), with output captured.
5. Pair with a second engineer during the run; verify after.

**Why →** One-off scripts cause a disproportionate share of incidents because they bypass normal controls.

**Interview tip →** "Peer review plus dry-run plus backup plus cap" is the four-part answer.

</details>

---

## AWS automation with boto3

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. You are asked to write a script that stops all dev EC2 instances at 8 PM and starts them at 8 AM. How do you build it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic cost-saving automation; the risk is stopping the wrong instances.

**What I'd do →**
1. Select only instances tagged `Environment=dev` and `Schedule=office-hours` (opt-in, never "everything not prod").
2. Use a paginator with `Filters` for the tags and the current state.
3. Call `stop_instances`/`start_instances` in batches, log IDs, support `--dry-run` (and `DryRun=True` to verify permissions).
4. Run it as a Lambda on EventBridge Scheduler (cron with time zone), with a role allowed only `ec2:StopInstances`/`StartInstances` on resources with that tag (IAM condition `aws:ResourceTag/Schedule`).

**Why →** Opt-in tags plus IAM conditions make it impossible to stop prod even if the code is wrong.

**Interview tip →** Mention the IAM tag condition; it is defence in depth beyond the code filter.

</details>

### Q27. A script that lists all S3 objects in a bucket reports 1,000 objects, but the console says there are 48,000. What happened, and how do you fix the script? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `list_objects_v2` returns at most 1,000 keys per call; the script read one page.

**What I'd do →**
1. Replace the single call with a paginator:

```python
total = 0
for page in s3.get_paginator("list_objects_v2").paginate(Bucket=bucket, Prefix=prefix):
    total += page.get("KeyCount", 0)
```
2. Audit other scripts for the same bug on any `list_*`/`describe_*` call.
3. For counts and sizes across huge buckets, use S3 Inventory or Storage Lens instead of listing.

**Why →** Every AWS list API is paginated; one page is not the answer.

**Interview tip →** Say "pagination bugs are silent: no error, just wrong data".

</details>

### Q28. You need to find and delete EBS snapshots older than 90 days, but some are used by AMIs and backup plans. How do you write it safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old snapshot cleanup with dependencies; deleting the wrong one breaks AMIs or compliance retention.

**What I'd do →**
1. List with `describe_snapshots(OwnerIds=["self"])` paginator; filter `StartTime < now - 90d` using timezone-aware datetimes.
2. Exclude snapshots referenced by registered AMIs (`describe_images(Owners=["self"])`, collect `BlockDeviceMappings[].Ebs.SnapshotId`).
3. Exclude snapshots managed by AWS Backup or DLM (tags like `aws:backup:source-resource` or `dlm:managed`) and anything tagged `Retain=true`.
4. Dry-run report first; delete with `delete_snapshot`, handling `InvalidSnapshot.InUse` as skip, not failure.
5. Long term, let DLM or AWS Backup own retention.

**Why →** Delete only what you can prove is unowned.

**Interview tip →** Mentioning the AMI cross-check shows you have actually done this.

</details>

### Q29. Your audit script hits `ThrottlingException` / `RequestLimitExceeded` when scanning large accounts. What do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Too many calls too fast, probably from parallelism or per-resource describe calls.

**What I'd do →**
1. Configure `Config(retries={"mode": "adaptive", "max_attempts": 10})` so botocore backs off and rate-limits client-side.
2. Reduce call volume: filter server-side, batch IDs into one describe call, use `get_metric_data` instead of per-instance `get_metric_statistics`.
3. Lower `max_workers` or add a per-service semaphore.
4. Consider AWS Config advanced queries or Resource Explorer for inventory.

**Why →** Fewer calls beats faster retries.

**Interview tip →** Say you would check whether another tool in the same account is consuming the same quota.

</details>

### Q30. You must audit 40 AWS accounts in an Organization from one script. How do you handle credentials? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-account access, centrally.

**What I'd do →**
1. Deploy an `AuditReadOnly` role to every account via StackSets or Terraform, trusting only a role in the security/tooling account.
2. List accounts with `organizations.list_accounts` (paginator) from the management or delegated admin account.
3. For each account, `sts.assume_role` into the audit role and build a fresh Session from the returned credentials.
4. Run accounts in parallel with a bounded pool; catch `AccessDenied` per account and report rather than abort.

**Why →** One trust path, least privilege, no keys in any account.

**Interview tip →** Mention `ExternalId` only matters for third-party access; inside an org, condition on `aws:PrincipalOrgID`.

</details>

### Q31. A Lambda written in Python times out at 15 minutes while processing a big S3 prefix. What are your options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Lambda's hard maximum is 15 minutes; the job is too big for a single invocation.

**What I'd do →**
1. Fan out: one Lambda lists keys and sends batches to SQS; worker Lambdas process batches concurrently.
2. Or use Step Functions Distributed Map over the S3 prefix.
3. Or move to a container job on ECS/Fargate or AWS Batch with no time limit.
4. Optimize per item: reuse clients created outside the handler, parallelize I/O with threads.

**Why →** Lambda is designed for short, parallel work; long sequential jobs belong elsewhere.

**Interview tip →** Mention creating boto3 clients outside the handler so warm invocations reuse connections.

</details>

### Q32. You are asked to tag every untagged resource in an account with `Owner=unknown`. What questions do you ask and how do you implement it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bulk change across services; "untagged" and "every" need definitions.

**What I'd do →**
1. Ask: which tag keys are required, which resource types, and is `unknown` a real value or should we find the creator?
2. Find resources with the Resource Groups Tagging API (`get_resources` paginator) and filter those missing the key. Note it only returns resources that have, or once had, tags; fully untagged resources may need per-service describes or AWS Config.
3. Look up creators via CloudTrail (`lookup_events` for `RunInstances`) to set a real owner where possible.
4. Apply with `tag_resources` in batches of 20 ARNs, dry-run first.
5. Prevent recurrence with tag policies and SCPs requiring tags on create.

**Why →** Backfilling once is pointless without enforcement going forward.

**Interview tip →** Say "find the creator in CloudTrail" before defaulting to `unknown`.

</details>

### Q33. A script must create an S3 bucket if it does not exist, and it runs in parallel CI jobs. How do you make it race-safe? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check-then-create races between jobs.

**What I'd do →**
1. Skip the check; call `create_bucket` and treat `BucketAlreadyOwnedByYou` as success.
2. Treat `BucketAlreadyExists` (someone else owns the name globally) as a real error.
3. Pass `CreateBucketConfiguration={"LocationConstraint": region}` outside `us-east-1`.
4. Apply settings (encryption, public access block, versioning) idempotently after creation.
5. Better: create buckets once in Terraform, not in CI jobs.

**Why →** EAFP avoids the time-of-check/time-of-use race.

**Interview tip →** Know the two error codes and what each means.

</details>

### Q34. You need to find every security group that allows 0.0.0.0/0 on any port other than 80 and 443. How do you write the check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Port ranges and protocol `-1` are the edge cases.

**What I'd do →**
1. Paginate `describe_security_groups`.
2. For each `IpPermissions` entry, check `IpRanges[].CidrIp == "0.0.0.0/0"` and `Ipv6Ranges[].CidrIpv6 == "::/0"`.
3. Handle `IpProtocol == "-1"` (all traffic, no port fields), and ranges (`FromPort` to `ToPort`) that include more than 80/443.
4. Check whether the SG is attached (`describe_network_interfaces` with `group-id` filter) to prioritize.

```python
allowed = {80, 443}
risky = proto == "-1" or not (lo == hi and lo in allowed)
```

**Why →** Rules like `0-65535` or protocol `-1` are the most dangerous and the most commonly missed.

**Interview tip →** Mention IPv6; many audits forget `::/0`.

</details>

### Q35. Finance asks why EC2 spend doubled last month. You have Python and read access. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cost investigation; the data is in Cost Explorer, not EC2.

**What I'd do →**
1. Query Cost Explorer: `ce.get_cost_and_usage(TimePeriod=..., Granularity="DAILY", Metrics=["UnblendedCost"], GroupBy=[{"Type": "DIMENSION", "Key": "USAGE_TYPE"}])` to see when and which usage type jumped.
2. Group by `LINKED_ACCOUNT`, `REGION`, `INSTANCE_TYPE` and by the `Owner` tag to narrow it.
3. Cross-check with CloudTrail `RunInstances` events around the day of the jump.
4. Report with a daily chart and the specific resources.

**Why →** Start from billing data and drill down; do not guess from current inventory.

**Interview tip →** Note that Cost Explorer API calls cost $0.01 each, so cache results.

</details>

### Q36. You need to wait until a newly created RDS instance is available before running migrations. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a waiter instead of a sleep loop.

**What I'd do →**
1. `rds.get_waiter("db_instance_available").wait(DBInstanceIdentifier=name, WaiterConfig={"Delay": 30, "MaxAttempts": 60})`.
2. Catch `botocore.exceptions.WaiterError` and report the last status.
3. Then fetch the endpoint from `describe_db_instances` and check TCP connectivity before migrating.

**Why →** Waiters handle polling, intervals and failure states consistently.

**Interview tip →** Mention that "available" in the API does not guarantee your network path works; check connectivity too.

</details>

### Q37. Your script uses `boto3.resource("ec2")` and a new team member asks whether to keep using resources. What do you advise? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWS has feature-frozen the resource interface.

**What I'd do →**
1. Keep existing working code, but write new code with clients.
2. Clients cover every API and new features; resources will not get new services or operations.
3. Clients return plain dicts, which are easy to test with moto or Stubber and to type with boto3-stubs.

**Why →** Long-term maintainability and full API coverage.

**Interview tip →** Know that the resource interface is frozen; it often comes up.

</details>

### Q38. You need to copy an AMI to three regions and share it with two accounts as part of a release. How do you automate it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-region, multi-account, slow async operations.

**What I'd do →**
1. For each region, call `copy_image(SourceRegionId=..., SourceImageId=..., Name=..., Encrypted=True, KmsKeyId=regional_key)` from a client in the destination region.
2. Wait with `get_waiter("image_available")` in each region (in parallel threads).
3. Share: `modify_image_attribute(ImageId=..., LaunchPermission={"Add": [{"UserId": acct}]})`, and grant the accounts use of the KMS key (or the snapshots will be unusable).
4. Tag copies with source AMI and version; publish IDs to SSM parameters per region.
5. Use EC2 Image Builder distribution settings if available.

**Why →** Encrypted AMI sharing fails silently without KMS grants; that is the usual gotcha.

**Interview tip →** Mention KMS key permissions for shared encrypted snapshots.

</details>

### Q39. A boto3 script works locally with your SSO profile but fails in CI with `NoCredentialsError`. What is the plan? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI has no profile and no credentials configured.

**What I'd do →**
1. Set up GitHub OIDC: an IAM OIDC provider for `token.actions.githubusercontent.com` and a role trusting your repo and branch.
2. Add `permissions: id-token: write` and `aws-actions/configure-aws-credentials@v4` with `role-to-assume` and `aws-region`.
3. Remove any `profile_name="..."` hardcoding from the script; let the default chain pick up env credentials.

**Why →** OIDC gives short-lived credentials with no secrets to store or rotate.

**Interview tip →** Mention restricting the trust policy `sub` claim to `repo:org/name:ref:refs/heads/main`.

</details>

### Q40. You need to process every new file uploaded to an S3 bucket with Python. Design it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Event-driven processing; must handle bursts and failures.

**What I'd do →**
1. S3 event notification to SQS (or EventBridge), with a Lambda or container worker consuming from SQS.
2. Worker parses the event, URL-decodes the key (`urllib.parse.unquote_plus`, keys arrive encoded), processes, and deletes the message only on success.
3. Configure a dead-letter queue and alarm on its depth.
4. Make processing idempotent (S3 can deliver events more than once), keyed on bucket, key and version or ETag.

**Why →** A queue buffers bursts and gives retries plus a DLQ for free.

**Interview tip →** The `unquote_plus` detail catches many people; keys with spaces arrive as `+`.

</details>

### Q41. How would you write a Python script that enforces "no public S3 buckets" across accounts? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detection and remediation; prevention is better.

**What I'd do →**
1. Prevent first: account-level S3 Block Public Access (`s3control.put_public_access_block`) and an SCP that denies changing it.
2. Detect: for each bucket, check `get_public_access_block`, bucket policy status (`get_bucket_policy_status` returns `IsPublic`), and ACLs; handle `NoSuchPublicAccessBlockConfiguration` as "not set".
3. Remediate with an allow-list for intentional public buckets (static sites behind CloudFront should use OAC instead).
4. Report findings; auto-remediate only with approval.

**Why →** Account-level controls stop the problem; scripts verify they stay on.

**Interview tip →** Mention `get_bucket_policy_status`; it saves you parsing policies yourself.

</details>

### Q42. A script that runs `describe_instances` with 2,000 instance IDs fails with a filter limit error. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** EC2 limits the number of values in a filter or ID list per request.

**What I'd do →**
1. Chunk the IDs into batches (for example 200) and call per batch, or use a paginator with no ID list and filter in Python.
2. Write a generic `chunked()` helper and reuse it for other batch-limited APIs (`tag_resources` max 20 ARNs, `delete_objects` max 1,000 keys).

```python
for batch in chunked(ids, 200):
    for page in ec2.get_paginator("describe_instances").paginate(InstanceIds=batch): ...
```

**Why →** Every batch API has a documented limit; chunking is the standard pattern.

**Interview tip →** Quote a couple of real limits; it shows hands-on experience.

</details>

### Q43. You need to rotate IAM access keys older than 90 days for service users and notify owners. What do you build? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keys should not exist at all for most services; the script is a stopgap.

**What I'd do →**
1. Get the credential report (`iam.generate_credential_report`, poll, then `get_credential_report`, parse CSV) for key age and last used.
2. Flag keys older than 90 days; for unused keys (no `last_used_date` in 90 days) deactivate, then delete after a grace period.
3. Notify owners via the `Owner` tag on the user, with a deadline.
4. Track the real fix: move each workload to roles (OIDC, instance profiles, IRSA) and delete the user.

**Why →** The credential report gives everything in one call instead of N `list_access_keys` calls.

**Interview tip →** Say the goal is zero long-lived keys; rotation is risk reduction, not the solution.

</details>

### Q44. You are writing a cleanup Lambda that deletes unattached EBS volumes. How do you avoid deleting something important? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** "Unattached" (`State=available`) does not mean "unwanted".

**What I'd do →**
1. Filter `status=available`, then require a minimum age since detachment (CloudTrail `DetachVolume` time or a `detached-at` tag set by a first pass).
2. Skip volumes tagged `Retain=true` or owned by a known system.
3. Snapshot before deletion (tag the snapshot with source volume and date) for a recovery window.
4. Run in report mode for two weeks before enabling deletion.

**Why →** A two-phase "mark, then sweep" design gives owners time to object.

**Interview tip →** "Mark and sweep with a grace period" is the pattern name to use.

</details>

### Q45. You need to know which IAM permissions your Python tool needs. How do you find the minimum? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege for automation roles.

**What I'd do →**
1. List every boto3 call in the code; map each to its IAM action (usually `service:OperationName`, but not always, e.g. `list_objects_v2` is `s3:ListBucket`).
2. Run the test suite or a sandbox run with CloudTrail and use IAM Access Analyzer policy generation from the CloudTrail activity.
3. Scope resources and conditions where the API supports it (tag conditions, specific ARNs).
4. Document the policy in the README and keep it in Terraform.

**Why →** Generated policies from real activity are accurate; guessing leads to `*`.

**Interview tip →** Mention Access Analyzer policy generation by name.

</details>

### Q46. You want your boto3 code to be testable without moto for pure logic. How do you structure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate decisions from API calls.

**What I'd do →**
1. Fetch functions return plain data (lists of dicts).
2. Rule functions take that data and return findings or actions, with no boto3 inside.
3. Act functions take actions and call APIs.
4. Test rules with plain dicts in milliseconds; test fetch/act with moto or Stubber.

**Why →** Most bugs are in the rules, and rules are trivial to test once they do not touch the network.

**Interview tip →** "Functional core, imperative shell" is the phrase.

</details>

### Q47. CloudWatch shows no `CPUUtilization` data for some instances, and your idle-instance check flags them as idle. What is wrong with the check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** "No data" is being treated as "zero CPU".

**What I'd do →**
1. Treat empty `Datapoints` as unknown and skip (or report separately), never as idle.
2. Check why there is no data: newly launched, stopped most of the period, wrong region or dimension, or the metric name typo.
3. Require a minimum number of datapoints over the window before judging.
4. Consider network and disk metrics too; a low-CPU instance can be a busy proxy.

**Why →** Missing data is not evidence of idleness; acting on it stops real workloads.

**Interview tip →** Our lab's `idle_instances` check skips instances without datapoints for exactly this reason.

</details>

### Q48. A colleague wants to use `boto3` from inside a Kubernetes pod with access keys mounted as a Secret. What do you suggest? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-lived keys in a cluster are avoidable.

**What I'd do →**
1. On EKS, use EKS Pod Identity or IRSA: bind an IAM role to the pod's service account.
2. boto3 picks up the credentials automatically (web identity token or container credentials endpoint); no code change.
3. Scope the role to the specific actions and resources.

**Why →** Short-lived, automatically rotated credentials tied to one workload.

**Interview tip →** Mention that a recent boto3 version is needed for Pod Identity support; old images fail with `NoCredentialsError`.

</details>

### Q49. You must export all CloudWatch alarms in an account to a YAML file for review. How do you build it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Read-only export.

**What I'd do →**
1. Paginate `describe_alarms` (metric and composite alarms).
2. Select relevant fields (name, metric, threshold, comparison, actions, state).
3. Convert datetimes to ISO strings, then `yaml.safe_dump(data, sort_keys=False)`.
4. Commit the YAML to a repo so changes show up as diffs over time.

**Why →** A versioned export turns an opaque console setting into reviewable config.

**Interview tip →** Suggest managing alarms as code (Terraform) as the next step.

</details>

### Q50. Your audit tool takes 40 minutes across 17 regions. How do you get it under 5? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sequential I/O across regions and services.

**What I'd do →**
1. Run regions concurrently with `ThreadPoolExecutor`, one boto3 Session per worker thread.
2. Within a region, run independent checks concurrently too, bounded per service.
3. Replace per-resource calls with batch calls (`get_metric_data`, filters).
4. Skip regions that are disabled or have no resources (`describe_regions`).
5. Measure per-phase timings in logs to target the slowest part.

**Why →** The work is almost entirely network wait, which threads overlap well.

**Interview tip →** Mention Sessions not being thread-safe; it is a common follow-up.

</details>

---

## Kubernetes, Docker, SSH & fleet automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. You need a report of every pod in the cluster with no CPU or memory limits. How do you write it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Read-only audit with the Kubernetes client.

**What I'd do →**
1. `config.load_kube_config()`, then `CoreV1Api().list_pod_for_all_namespaces(limit=500)`, paginating with `_continue`.
2. For each container (including init containers), check `container.resources.limits` for `cpu` and `memory`; `resources.limits` can be `None`.
3. Output namespace, pod, owner (from `metadata.owner_references`), container and missing limits as CSV.
4. Long term: enforce with a LimitRange or an admission policy (Kyverno, Gatekeeper, or ValidatingAdmissionPolicy).

**Why →** Reports find existing gaps; admission policies stop new ones.

**Interview tip →** Report by owner (Deployment), not pod, so teams can fix the source.

</details>

### Q52. You are asked to write a small operator-like script that adds a label to every new namespace. How do you approach it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Event-driven reconciliation; watches can drop and expire.

**What I'd do →**
1. Consider whether a mutating admission policy or Kyverno rule does this declaratively; usually it does.
2. If code is needed, use `kopf` (handles watches, retries, and resumption) or a watch loop that lists first, reconciles all, then watches from the list's `resourceVersion`.
3. Handle `410 Gone` by re-listing, and reconnect on timeouts.
4. Patch idempotently (only if the label is missing).
5. Run in-cluster with a service account allowed only `get, list, watch, patch` on namespaces.

**Why →** Level-triggered reconciliation (look at all current state) survives missed events.

**Interview tip →** Say "level-triggered, not edge-triggered" and mention `410 Gone`.

</details>

### Q53. You need to restart all Deployments in a namespace that reference a rotated Secret. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pods do not reload env vars from Secrets automatically.

**What I'd do →**
1. List Deployments; find ones referencing the secret in `env[].valueFrom.secretKeyRef`, `envFrom[].secretRef` or `volumes[].secret`.
2. Patch each with the `kubectl.kubernetes.io/restartedAt` annotation on the pod template, triggering a rolling restart.
3. Watch rollout status (`status.updatedReplicas == spec.replicas` and `availableReplicas`) before moving to the next.
4. Long term: use Reloader or a checksum annotation in Helm so this is automatic.

**Why →** One-at-a-time rollouts avoid restarting everything simultaneously.

**Interview tip →** Mention that volume-mounted Secrets update in place (eventually) but env vars never do.

</details>

### Q54. Your CI runners keep running out of disk because of Docker images. Write a cleanup plan in Python. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build caches and old images accumulate on long-lived runners.

**What I'd do →**
1. Script with the Docker SDK: `containers.prune()`, `images.prune(filters={"dangling": True})`, then remove images older than N days not in an allow-list, and `api.prune_builds()` for BuildKit cache.
2. Run it as a scheduled job and log space reclaimed.
3. Alert on disk above 80% (node exporter).
4. Better: ephemeral runners that start clean each job.

**Why →** Pruning treats the symptom; ephemeral runners remove the cause.

**Interview tip →** Mention build cache, which `docker image prune` does not touch.

</details>

### Q55. You need to run `df -h` on 300 Linux hosts and list any above 85%. Python or Ansible? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ad-hoc fleet query.

**What I'd do →**
1. If the fleet is in Ansible inventory: `ansible all -m shell -a "df -hP"` or a playbook with a filter; that is the fastest path.
2. In Python, use Fabric's `ThreadingGroup(*hosts).run("df -P", hide=True, warn=True)` and parse output; collect failures separately.
3. On AWS without SSH: SSM Run Command targeting by tag.
4. Long term: node exporter plus an alert, so nobody has to ask.

**Why →** The right answer is often "this should be a metric, not a script".

**Interview tip →** End with the monitoring answer.

</details>

### Q56. A Python script SSHes to hosts with `AutoAddPolicy`. Security says that is unacceptable. What do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blindly trusting host keys allows man-in-the-middle.

**What I'd do →**
1. Pre-populate known_hosts from a trusted source (keys captured at provisioning time, or SSH certificates signed by your CA).
2. `ssh.load_system_host_keys()` or `load_host_keys(path)` and `set_missing_host_key_policy(RejectPolicy())`.
3. Consider SSH certificates (`@cert-authority` in known_hosts) so new hosts are trusted automatically and securely.
4. On AWS, consider SSM Session Manager to remove SSH entirely.

**Why →** Host key verification is what makes SSH resistant to interception.

**Interview tip →** SSH certificates are the scalable answer for dynamic fleets.

</details>

### Q57. You want to validate all Kubernetes manifests in a repo in CI with Python. What would the script check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Policy checks on YAML before it reaches the cluster.

**What I'd do →**
1. Parse every file with `yaml.safe_load_all`, skipping empty docs; fail on YAML errors with file and line.
2. Schema-validate with `kubeconform` (via `subprocess`), not by hand.
3. Custom rules in Python: image tag not `latest`, resources set, `runAsNonRoot: true`, required labels, no `hostPath`.
4. Output findings with file paths; exit non-zero on violations.
5. Consider Conftest/Kyverno CLI to share the same policies with the cluster.

**Why →** Catching policy violations at PR time is cheaper than at admission time.

**Interview tip →** Mention reusing the same policies in CI and in-cluster.

</details>

### Q58. You need to tail logs from all pods of a Deployment at once in a Python tool. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multiple concurrent streams.

**What I'd do →**
1. List pods by the Deployment's selector (`label_selector="app=api"`).
2. For each pod, stream with `read_namespaced_pod_log(name, ns, container=c, follow=True, _preload_content=False)` and iterate `resp.stream()` in a thread.
3. Prefix lines with pod name; push lines into a `queue.Queue`, print from the main thread.
4. Watch for new pods (rollouts) and start streams for them.

**Why →** One thread per stream is simple and the work is I/O bound.

**Interview tip →** Mention `stern` or `kubectl logs -l app=api -f --prefix` as existing tools before building one.

</details>

### Q59. How would you use Python to check that every Docker image running in your cluster comes from your approved registry? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain audit.

**What I'd do →**
1. List all pods; collect `spec.containers[].image`, `initContainers` and `ephemeralContainers`.
2. Normalize references (no registry means `docker.io/library/`); compare against an allowed prefix list (`123456789012.dkr.ecr.ap-south-1.amazonaws.com/`).
3. Also check `status.containerStatuses[].imageID` for the actual digest running.
4. Enforce with an admission policy (Kyverno `verifyImages` or a registry restriction policy).

**Why →** Normalization catches images that look private but are pulled from Docker Hub.

**Interview tip →** Mention implicit `docker.io` and digest vs tag.

</details>

### Q60. You need to back up a database running in a container every night with Python and upload to S3. Design it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scheduled backup, streaming to S3, verified.

**What I'd do →**
1. Run `pg_dump -Fc` via `subprocess.Popen` and stream stdout straight into `s3.upload_fileobj` (no local disk needed).
2. Key by date (`db/2026/11/09/app.dump`), SSE-KMS encryption, lifecycle rules for retention.
3. Check `pg_dump`'s exit code after the upload; delete the object if the dump failed.
4. Weekly restore test into a scratch database.
5. Run as a Kubernetes CronJob with IRSA/Pod Identity, alert on failure.

**Why →** Backups that are never restored are hopes, not backups.

**Interview tip →** Restore testing is what interviewers want to hear.

</details>

### Q61. A Python script needs to run `kubectl` against three clusters. How do you manage contexts safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Risk of running against the wrong cluster.

**What I'd do →**
1. Never rely on the current context; pass `--context` explicitly on every call, or `config.load_kube_config(context=name)` with the Python client.
2. Print the target cluster and require confirmation (or a flag) for prod.
3. Use separate kubeconfig files per environment via `KUBECONFIG` for CI jobs.

**Why →** "Current context" is shared mutable state; someone else's `kubectl config use-context` changes your target.

**Interview tip →** Mention explicit `--context` as a habit.

</details>

### Q62. You need to make 1,000 HTTP health checks every minute against internal endpoints. Threads or asyncio? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** High fan-out I/O on a timer.

**What I'd do →**
1. asyncio with `httpx.AsyncClient` (or aiohttp), a `Semaphore(100)`, per-request timeouts (`timeout=3`), and `asyncio.gather(..., return_exceptions=True)`.
2. Record results as Prometheus metrics (`probe_success`, latency histogram) from a long-running process.
3. Or just deploy Prometheus `blackbox_exporter`, which does exactly this.

**Why →** asyncio handles thousands of sockets in one thread cheaply; threads would need a big pool.

**Interview tip →** Mentioning blackbox_exporter shows you do not reinvent monitoring.

</details>

### Q63. You must drain and replace 50 Kubernetes nodes one at a time with Python, respecting PodDisruptionBudgets. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Controlled rolling node replacement.

**What I'd do →**
1. For each node: cordon (`patch_node(name, {"spec": {"unschedulable": True}})`).
2. Evict pods with the Eviction API (`create_namespaced_pod_eviction`), skipping DaemonSet and mirror pods. A 429 response means a PDB is blocking; back off and retry.
3. Wait until the node has no evictable pods, then terminate the instance (ASG `terminate_instance_in_auto_scaling_group` with `ShouldDecrementDesiredCapacity=False`).
4. Wait for the replacement node to be `Ready` before continuing.
5. Prefer managed node group updates or Karpenter drift, which do this already.

**Why →** The Eviction API respects PDBs; deleting pods directly does not.

**Interview tip →** "Evict, don't delete" is the key line.

</details>

### Q64. Your team wants a ChatOps bot that can scale deployments from Slack. What are your design concerns? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Remote control of production from chat.

**What I'd do →**
1. Verify Slack request signatures (`X-Slack-Signature`, HMAC with timestamp check against replay).
2. Map Slack users to roles; allow-list commands and namespaces; enforce limits (max replicas).
3. The bot's Kubernetes service account gets only `patch` on `deployments/scale` in allowed namespaces.
4. Audit-log every command with user, time and result; post results to the channel.
5. Respond within 3 seconds (Slack's limit), process asynchronously.

**Why →** ChatOps is an authenticated API to production and needs the same controls.

**Interview tip →** Mention signature verification and least-privilege RBAC.

</details>

### Q65. You need to push a config file to 20 servers and restart a service, but only if the file changed. How in Python? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Idempotent config push; Ansible's `template` + handler does this.

**What I'd do →**
1. Compute local SHA-256; on each host run `sha256sum` on the remote file via Fabric.
2. If different: `c.put()` to a temp path, validate (`nginx -t -c ...`), `sudo mv` into place, `sudo systemctl reload`.
3. Roll through hosts in batches and stop on the first failure.
4. Suggest Ansible for anything beyond this.

**Why →** Comparing checksums makes the push idempotent and avoids needless restarts.

**Interview tip →** Validate before reload; a bad config should never reach the running service.

</details>

### Q66. A Kubernetes CronJob running a Python script sometimes runs twice at the same time. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrency policy, or a run that exceeds the schedule interval.

**What I'd do →**
1. Set `concurrencyPolicy: Forbid` on the CronJob.
2. Set `startingDeadlineSeconds` so missed runs do not pile up.
3. Make the script idempotent anyway; Kubernetes CronJobs can, in rare cases, create two jobs.
4. Add an `activeDeadlineSeconds` so a hung run is killed.

**Why →** Forbid plus idempotency covers both the common and rare double runs.

**Interview tip →** Mention that CronJobs are "at least once", not "exactly once".

</details>

### Q67. You need to check TLS certificate expiry for 400 domains and alert at 21 days. How do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Simple but valuable check.

**What I'd do →**
1. For each domain, open a TLS connection with `ssl.create_default_context()` and `socket.create_connection((host, 443), timeout=5)`, wrap with `server_hostname=host`, read `getpeercert()["notAfter"]`, parse with `ssl.cert_time_to_seconds`.
2. Run concurrently with a thread pool.
3. Report domains below 21 days; also report connection or verification failures.
4. Or export as metrics and use blackbox_exporter's `probe_ssl_earliest_cert_expiry`.

**Why →** Expired certificates are a fully preventable outage.

**Interview tip →** Use SNI (`server_hostname`), or you get the wrong certificate on shared hosts.

</details>

### Q68. A Python service in a container takes 30 seconds to stop during deploys. What do you look at? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The process is not handling SIGTERM, so it waits for SIGKILL after the grace period.

**What I'd do →**
1. Check the Dockerfile uses exec-form `ENTRYPOINT ["gunicorn", ...]`, not a shell form where `/bin/sh` is PID 1 and swallows signals.
2. Confirm the app or server handles SIGTERM (gunicorn and uvicorn do; custom loops need a handler).
3. For custom workers, install a `signal.signal(signal.SIGTERM, ...)` handler that stops accepting work and exits.
4. Use `tini` (`docker run --init`) if the app spawns children.

**Why →** Proper signal handling means fast, graceful shutdowns and no dropped requests.

**Interview tip →** "Shell-form CMD makes sh PID 1" is the usual root cause.

</details>

### Q69. You want to build a Docker image from Python in CI and push it to ECR. What does the script need? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Auth to ECR plus build and push.

**What I'd do →**
1. Get a token: `ecr.get_authorization_token()` returns base64 `AWS:<password>` and the proxy endpoint.
2. `docker.from_env().login(username="AWS", password=pw, registry=endpoint)`.
3. Build with `images.build(path=".", tag=f"{repo}:{sha}")` (or shell out to `docker buildx build` for BuildKit features), then `images.push(repo, tag=sha, stream=True, decode=True)` and check each line for `error`.
4. In GitHub Actions, the official `aws-actions/amazon-ecr-login` plus `docker/build-push-action` is simpler.

**Why →** The push stream reports errors inline; ignoring it hides failures.

**Interview tip →** Mention checking the push stream for `errorDetail`.

</details>

### Q70. You need an inventory of all EC2 instances turned into an Ansible dynamic inventory. Python script or plugin? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ansible already ships this.

**What I'd do →**
1. Use the `amazon.aws.aws_ec2` inventory plugin with `keyed_groups` on tags and `filters`.
2. Only write a custom script if the source is something without a plugin (a CMDB API), outputting JSON with `_meta.hostvars` for `--list`.

**Why →** The maintained plugin handles pagination, caching and regions.

**Interview tip →** Knowing the plugin exists is the point.

</details>

### Q71. A script needs to run a long command on remote hosts and survive your laptop disconnecting. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long-running remote work tied to a local SSH session dies with the session.

**What I'd do →**
1. Do not keep the work in the SSH session: start it as a systemd transient unit (`systemd-run --unit=reindex ...`) or inside `tmux`/`nohup`, and poll status.
2. Better: run the orchestration from a CI job or a bastion, not a laptop.
3. On AWS, SSM Run Command keeps running independent of your client and stores output.

**Why →** Remote work should not depend on the controller's network.

**Interview tip →** `systemd-run` is a neat, less-known answer.

</details>

### Q72. You want to expose a small Python service that reports deployment versions of all services in the cluster. How do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Read-only internal API backed by the Kubernetes API.

**What I'd do →**
1. FastAPI app with `/versions` that lists Deployments and returns image tags per container.
2. Cache results for 30 seconds (or use a watch to keep an in-memory cache) to avoid hammering the API server.
3. In-cluster config with a service account allowed `list` on `deployments` only.
4. `/healthz`, `/readyz`, `/metrics`; run under uvicorn with 2 workers behind a ClusterIP Service.

**Why →** Caching protects the API server; scoped RBAC limits exposure.

**Interview tip →** Mention caching; naive per-request list calls on large clusters hurt the control plane.

</details>

### Q73. You discover a Python tool mounts `/var/run/docker.sock` into its container to list containers. What is the risk and alternative? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker socket access equals root on the host.

**What I'd do →**
1. Explain: anyone who compromises the tool can start a privileged container mounting `/` and own the host.
2. Alternatives: use the Kubernetes API (on k8s), cAdvisor or node metrics for read-only data, or a socket proxy (`docker-socket-proxy`) allowing only `GET /containers`.
3. If unavoidable, run the tool isolated, read-only root, minimal network, and audited.

**Why →** Least privilege applies to sockets too.

**Interview tip →** Say "docker.sock is root" in the first sentence.

</details>

### Q74. You need to compare installed package versions across 100 servers to find drift. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet query plus diff.

**What I'd do →**
1. Collect `rpm -qa --qf '%{NAME} %{VERSION}-%{RELEASE}\n'` or `dpkg-query -W` via Fabric/SSM.
2. Parse into `{host: {pkg: version}}`, find packages whose version set across hosts has more than one value.
3. Output a CSV of package, versions and host counts.
4. Long term: immutable images (AMIs/containers) so drift cannot happen.

**Why →** Drift detection points to the real fix: immutable infrastructure.

**Interview tip →** End with immutability.

</details>

### Q75. You need to watch for EC2 instance state changes and post to Slack in near real time. What do you build? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Event-driven, not polling.

**What I'd do →**
1. EventBridge rule on `EC2 Instance State-change Notification` targeting a Python Lambda.
2. Lambda reads `detail.instance-id` and `detail.state`, looks up the `Name` and `Owner` tags with `describe_instances`, posts to Slack.
3. Slack webhook URL in Secrets Manager, cached across invocations.
4. DLQ and retry settings on the target.

**Why →** Events cost nothing when idle and arrive within seconds; polling wastes API calls.

**Interview tip →** "Event-driven over polling" is the principle.

</details>

---

## Testing, CI/CD, packaging & services

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. You join a team whose Python automation repo has no tests. Where do you start? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Untested code that touches infrastructure.

**What I'd do →**
1. Add pytest, ruff and mypy to CI first, even with few tests, so the bar can only go up.
2. Write tests for the most destructive scripts first (cleanup, deletion), using moto and dry-run assertions.
3. Add a test with every bug fix that reproduces the bug.
4. Refactor big scripts into pure functions as you touch them.

**Why →** Risk-first testing gives the most protection per hour.

**Interview tip →** "Test the scripts that delete things first."

</details>

### Q77. Your pytest suite takes 12 minutes. How do you speed it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually network calls, sleeps, or expensive setup repeated per test.

**What I'd do →**
1. `pytest --durations=20` to find the slowest tests.
2. Remove real network and `time.sleep` (mock clocks, mock retries, set backoff to zero in tests).
3. Widen fixture scope for expensive, read-only setup (session-scoped containers).
4. Run in parallel with `pytest-xdist` (`-n auto`), ensuring tests are independent.
5. Split slow integration tests behind a marker (`-m "not integration"`) for PRs.

**Why →** Most slow suites are a few slow tests; measure first.

**Interview tip →** `--durations` is the first command to name.

</details>

### Q78. A test passes locally but fails in CI only when run with the rest of the suite. What do you suspect? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Test pollution: shared state leaking between tests.

**What I'd do →**
1. Run with random order (`pytest-randomly`) and bisect: `pytest -p no:randomly tests/test_a.py tests/test_b.py`.
2. Look for module-level state, singletons, cached clients created at import, env vars set without `monkeypatch`, and files written outside `tmp_path`.
3. With moto, look for clients created outside `mock_aws` (they hit real endpoints or a different mock).

**Why →** Order-dependent tests are always shared-state bugs.

**Interview tip →** Mention clients created at import time, a frequent moto pitfall.

</details>

### Q79. You want a reusable CI pipeline for 15 Python tool repos. What goes in it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standardize quality gates once.

**What I'd do →**
1. A reusable GitHub Actions workflow (`workflow_call`) with inputs for Python version and package path.
2. Steps: checkout, `actions/setup-python` with pip cache (or `astral-sh/setup-uv`), install from lock, `ruff check`, `ruff format --check`, `mypy`, `pytest --cov`, `pip-audit`, build wheel.
3. Build and scan the Docker image (Trivy) where applicable; push on main with SHA tags.
4. Pin actions by SHA; least-privilege `permissions:`.

**Why →** One pipeline definition means consistent gates and one place to fix things.

**Interview tip →** Mention pinning third-party actions by commit SHA.

</details>

### Q80. Your Python Docker image is 1.1 GB. How do you shrink it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Full base image, build tools, caches and tests shipped to production.

**What I'd do →**
1. `python:3.12-slim` instead of `python:3.12`; avoid Alpine unless all deps have musllinux wheels.
2. Multi-stage: build a wheel (or venv) in a builder stage with compilers, copy only the result to the runtime stage.
3. `pip install --no-cache-dir`; `.dockerignore` for `.git`, `.venv`, tests, caches.
4. Install only runtime deps, not `[dev]`.
5. Measure with `docker history` or `dive`.

**Why →** Most size is the base and build toolchain; multi-stage removes both.

**Interview tip →** Explain why Alpine can make Python images slower to build and sometimes bigger.

</details>

### Q81. You want to distribute an internal CLI to 200 engineers. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Distribution plus updates.

**What I'd do →**
1. Publish a wheel to an internal index (CodeArtifact, Artifactory); engineers install with `pipx install acme-ops` or `uv tool install acme-ops` and upgrade the same way.
2. Also publish a Docker image for CI and for people who do not want Python locally.
3. Version with semver and a changelog; add a "new version available" check if needed.
4. Single-file options (`shiv`, `pex`, PyInstaller) for environments without Python.

**Why →** pipx/uv keep each tool isolated and upgradable.

**Interview tip →** pipx is the expected answer for CLI distribution.

</details>

### Q82. A Flask app runs with `app.run()` in production and falls over under load. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The development server is single-process and not built for production.

**What I'd do →**
1. Run under gunicorn: `gunicorn -w 4 -k gthread --threads 4 -b 0.0.0.0:8000 --timeout 30 app:app`.
2. Size workers to CPU (roughly 2 x cores + 1 as a start) and measure.
3. Put it behind a reverse proxy or load balancer; add health endpoints.
4. Set `--graceful-timeout` and handle SIGTERM for clean deploys.

**Why →** A real WSGI server handles concurrency, worker restarts and timeouts.

**Interview tip →** Mention the dev server warning itself says not to use it in production.

</details>

### Q83. You need a webhook receiver for GitHub that triggers deployments. Design it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public endpoint that can deploy to production.

**What I'd do →**
1. FastAPI or Flask endpoint verifying `X-Hub-Signature-256` with HMAC-SHA256 over the raw body and `hmac.compare_digest`.
2. Filter events (`X-GitHub-Event: push` on `refs/heads/main`, or `release` published).
3. Deduplicate on `X-GitHub-Delivery`; enqueue a job and return 202 within GitHub's 10-second timeout.
4. A worker performs the deploy idempotently and reports status back (commit status or deployment API).
5. Restrict ingress to GitHub's published hook IP ranges (from `api.github.com/meta`) as an extra layer.

**Why →** Signature, dedupe, and async processing are the three properties a deploy webhook needs.

**Interview tip →** Ask whether a pull-based model (Argo CD) removes the need for an inbound webhook.

</details>

### Q84. You need to add Prometheus metrics to a Python batch job that runs every 10 minutes. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Short-lived process; Prometheus cannot scrape it reliably.

**What I'd do →**
1. Use a dedicated `CollectorRegistry`, record `job_duration_seconds`, `job_items_processed_total` and `job_last_success_timestamp_seconds`.
2. `push_to_gateway("pushgateway.monitoring:9091", job="reconciler", registry=reg)` at the end (also on failure, with a failure gauge).
3. Alert: `time() - job_last_success_timestamp_seconds > 1800`.
4. If it runs on a host with node exporter, the textfile collector is an alternative.

**Why →** Pushgateway is designed for batch jobs; the last-success timestamp catches every failure mode.

**Interview tip →** Say Pushgateway is only for batch jobs, not for services.

</details>

### Q85. How would you add type hints to a 5,000-line untyped automation codebase without stopping feature work? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gradual typing.

**What I'd do →**
1. Add mypy to CI with a permissive baseline (`ignore_missing_imports`, no `strict`), so it runs and passes.
2. Enable stricter settings per module with `[[tool.mypy.overrides]]`, starting with core modules and new code (`disallow_untyped_defs`).
3. Add `boto3-stubs` and `types-requests`/`types-PyYAML` for library types.
4. Type function signatures first; inference handles most locals.
5. Ratchet: new modules must be strict.

**Why →** Gradual adoption delivers value immediately without a big-bang rewrite.

**Interview tip →** Mention per-module overrides; it is how real teams do it.

</details>

### Q86. Your team wants to move from `pip` + `requirements.txt` to `uv`. What do you evaluate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tooling change; benefits are speed and a real lock file.

**What I'd do →**
1. Measure current CI install time vs `uv pip sync` or `uv sync`.
2. Check compatibility: private indexes and auth, platform-specific wheels, Dependabot/Renovate support for `uv.lock`.
3. Migrate one repo: `pyproject.toml` with dependencies, `uv lock`, CI with `astral-sh/setup-uv` and cache.
4. Keep `uv export --format requirements-txt` available for consumers that need it.

**Why →** Measured pilots avoid tooling churn for its own sake.

**Interview tip →** Mention that uv can also install Python itself, replacing pyenv in CI.

</details>

### Q87. A tool must read secrets at runtime in both GitHub Actions and EKS. How do you design secret loading? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same code, different environments.

**What I'd do →**
1. The code reads configuration from environment variables only (`pydantic-settings` model).
2. In GitHub Actions, secrets come from OIDC + Secrets Manager (`aws secretsmanager get-secret-value` or the official action) into env vars, masked.
3. In EKS, External Secrets Operator syncs Secrets Manager into a Kubernetes Secret mounted as env.
4. The tool never knows where the secret came from.

**Why →** Environment-agnostic code with platform-specific injection keeps the tool simple and testable.

**Interview tip →** "Twelve-factor config" is the reference to make.

</details>

### Q88. You need to test a function that calls `subprocess.run(["terraform", "plan"])`. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Do not run Terraform in unit tests.

**What I'd do →**
1. Patch `subprocess.run` where it is used: `monkeypatch.setattr(mymod.subprocess, "run", fake_run)`, returning a `subprocess.CompletedProcess(args, 2, stdout=..., stderr="")`.
2. Test parsing and decision logic for each return code (0 no changes, 2 changes with `-detailed-exitcode`, 1 error).
3. Assert the exact arguments passed.
4. A separate integration test can run real Terraform against a local backend.

**Why →** Unit tests check your logic; integration tests check the tool.

**Interview tip →** Mention `-detailed-exitcode` semantics.

</details>

### Q89. Code review shows `except Exception as e: logger.error(e)` everywhere in a new tool. What feedback do you give? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Errors are swallowed and tracebacks lost.

**What I'd do →**
1. Catch specific exceptions where you can handle them (`ClientError` with a specific code, `FileNotFoundError`).
2. Use `log.exception(...)` to keep the traceback, and add context (which resource, which region).
3. Re-raise or exit non-zero when you cannot recover.
4. Keep one broad catch at the CLI entry point that logs and exits 1.

**Why →** Swallowed exceptions make tools report success while failing.

**Interview tip →** "Catch narrow, log with context, fail loudly."

</details>

### Q90. You want contract tests for an internal REST API your scripts depend on. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Your scripts break when the API changes shape.

**What I'd do →**
1. Validate responses against the API's OpenAPI schema in a scheduled integration test (`jsonschema` or `schemathesis`).
2. Model responses with pydantic in the client, so shape changes fail loudly at the boundary.
3. Record real responses as fixtures (sanitized) for unit tests with `responses`.
4. Agree on versioning with the API team; consumer-driven contracts (Pact) if the relationship is critical.

**Why →** Boundary validation turns silent data bugs into clear errors.

**Interview tip →** Mention pydantic models as runtime contracts.

</details>

### Q91. You must make a FastAPI service production-ready for Kubernetes. What is your checklist? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** From "works locally" to operable.

**What I'd do →**
1. Run with uvicorn (or gunicorn with uvicorn workers), exec-form entrypoint, non-root user, slim image.
2. `/healthz` (no dependencies) and `/readyz` (dependencies with short timeouts) wired to probes.
3. JSON logs to stdout with request IDs; `/metrics` with request count and latency histograms.
4. Graceful shutdown: handle SIGTERM, a `preStop` sleep for endpoint propagation.
5. Resource requests and limits, config from env, secrets from a secret store.
6. Timeouts on every outbound call.

**Why →** These are the properties every reliable service on Kubernetes needs.

**Interview tip →** The `preStop` sleep detail shows deep Kubernetes knowledge.

</details>

### Q92. Your scheduled audit in GitHub Actions should fail the workflow on critical findings but still upload the report. How do you wire it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need the artifact even when the step fails.

**What I'd do →**
1. Give the audit step an `id` and `continue-on-error: true`.
2. Upload the report with `actions/upload-artifact@v4` and `if: always()`; write the Markdown report to `$GITHUB_STEP_SUMMARY`.
3. A final step fails the job if `steps.audit.outcome == 'failure'`.

**Why →** Separating "run" from "judge" keeps evidence even on failure.

**Interview tip →** Our lab workflow uses exactly this pattern.

</details>

### Q93. Your Python app's latency spikes every few minutes. Prometheus shows the spikes line up with high GC activity. What do you investigate? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Garbage collection pauses or memory pressure.

**What I'd do →**
1. Check the `python_gc_collections_total` and `process_resident_memory_bytes` metrics that `prometheus_client` exports by default.
2. Profile allocations with `tracemalloc` snapshots or `memray` in staging.
3. Look for large object graphs created per request (caching whole API responses, huge dict copies).
4. Tune `gc.freeze()` after startup for pre-fork servers, or adjust thresholds, only after fixing allocation hot spots.
5. Check CPU throttling from container limits, which can masquerade as GC pauses.

**Why →** Allocation reduction fixes GC pauses; tuning only hides them.

**Interview tip →** Mention CPU throttling; it is a common confounder in containers.

</details>

### Q94. A teammate wants to put business logic in a 400-line GitHub Actions YAML with inline Bash. You suggest Python. How do you justify it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Logic in YAML is untestable and hard to debug.

**What I'd do →**
1. Move logic into a Python module in the repo with a CLI, tested with pytest.
2. The workflow just calls `python -m tools.release --version ${{ github.ref_name }}`.
3. Same tool runs locally for debugging.

**Why →** Testable, reviewable, runnable locally, portable across CI systems.

**Interview tip →** "Keep pipelines thin, put logic in versioned code."

</details>

### Q95. You need to guarantee that a Python service's dependencies are exactly what was tested. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reproducibility and supply-chain integrity.

**What I'd do →**
1. Lock all dependencies, including transitive, with hashes (`pip-compile --generate-hashes` or `uv lock`).
2. Install with `pip install --require-hashes --no-deps -r requirements.txt` (or `uv sync --frozen`) in the image build.
3. Build the image once, test that image, promote the same digest to production.
4. Generate an SBOM (Syft) and scan (Trivy/Grype); sign the image (cosign).

**Why →** Hashes stop tampered packages; promoting the same digest guarantees "tested == deployed".

**Interview tip →** "Build once, promote by digest" is the key line.

</details>

### Q96. How would you write an integration test that runs your Docker image against moto in CI? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Test the packaged artifact end to end without AWS.

**What I'd do →**
1. Start moto as a server: `docker run -d -p 5000:5000 motoserver/moto` as a service container.
2. Seed resources with boto3 pointed at `endpoint_url="http://localhost:5000"`.
3. Run the image with `-e AWS_ENDPOINT_URL=http://host:5000 -e AWS_ACCESS_KEY_ID=x -e AWS_SECRET_ACCESS_KEY=x -e AWS_DEFAULT_REGION=us-east-1`; recent boto3 honours `AWS_ENDPOINT_URL` without code changes.
4. Assert exit code and report content.

**Why →** Catches packaging bugs (missing modules, wrong entrypoint) that unit tests cannot.

**Interview tip →** Mention `AWS_ENDPOINT_URL`; it removes the need for test-only code paths.

</details>

### Q97. You need a lightweight internal dashboard of audit results. Do you build a Flask app? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Probably not; a static report may be enough.

**What I'd do →**
1. Generate a static HTML or Markdown report in the audit job and publish to S3 + CloudFront (with auth) or GitHub Pages for private repos.
2. Or push metrics (findings by severity) to Prometheus and use Grafana.
3. Only build a Flask app if users need interactivity (filtering, acknowledging findings).

**Why →** Static output has no server to patch or run.

**Interview tip →** Choosing not to build a service is often the senior answer.

</details>

### Q98. Your Python tool works in Docker on your laptop (arm64 Mac) but CI on x86 fails with `exec format error`. What happened? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The image was built for arm64 and pushed, then run on amd64.

**What I'd do →**
1. Build multi-arch: `docker buildx build --platform linux/amd64,linux/arm64 -t repo:tag --push .`.
2. Or build in CI on amd64 runners only and never push from laptops.
3. Check with `docker manifest inspect repo:tag` or `docker buildx imagetools inspect`.

**Why →** Images are architecture-specific unless built as multi-arch manifests.

**Interview tip →** Say "CI builds images, laptops don't push".

</details>

### Q99. Product asks for a feature-flag check inside a deployment script. How do you keep the script from becoming fragile? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** External dependency in the deploy path.

**What I'd do →**
1. Wrap the flag service call in a small client with a timeout and a safe default (fail closed or open, decided explicitly).
2. Cache the flag value for the duration of the run.
3. Log the decision and the flag value.
4. Unit-test both branches with a mocked client.

**Why →** Deploys must not hang or break because a non-critical service is slow.

**Interview tip →** Explicitly choosing fail-open vs fail-closed is what interviewers look for.

</details>

### Q100. Walk me through how you would present the cloud-ops audit CLI from the lab in an interview. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Project storytelling: problem, design, trade-offs, results.

**What I'd do →**
1. Problem: no visibility into untagged/idle EC2, open security groups and old snapshots across regions.
2. Design: click CLI, boto3 with paginators and adaptive retries, per-region threads with one Session each, findings as dataclasses, JSON/CSV/Markdown renderers, exit code 2 on HIGH findings.
3. Quality: pytest with moto (offline, free), ruff, mypy strict, Docker multi-stage non-root.
4. Operations: weekly GitHub Actions run via OIDC with a read-only role, report as artifact and job summary.
5. Trade-offs: moto gaps (snapshot timestamps) handled with stubs; "no metrics" treated as unknown, not idle.

**Why →** Structure shows engineering judgment, not just code.

**Interview tip →** Always have one trade-off and one thing you would improve next ready.

</details>

---

## Environments, imports & packaging errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `ModuleNotFoundError: No module named 'boto3'` right after you ran `pip install boto3`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pip` belongs to a different interpreter than `python` (system vs venv, `python3.11` vs `python3.12`), or the venv is not activated in this shell, or cron/systemd uses the system Python.

**Diagnose →**
```bash
which python pip; python -c "import sys; print(sys.executable)"
pip --version            # shows which Python it installs into
python -m pip show boto3
```

**Fix →** Install with the interpreter you run: `python -m pip install boto3`. In cron/systemd, call `/opt/tool/.venv/bin/python` explicitly.

**Prevent →** Always use `python -m pip`; use console-script entry points or absolute venv paths in schedulers.

</details>

### Q102. `error: externally-managed-environment` when running `pip install requests` on Ubuntu 24.04. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PEP 668: the system Python is managed by apt and pip refuses to modify it.

**Diagnose →**
```bash
ls /usr/lib/python3*/EXTERNALLY-MANAGED
which python3
```

**Fix →** `python3 -m venv .venv && . .venv/bin/activate && pip install requests`. For CLI tools: `pipx install <tool>`. For OS-level use: `sudo apt install python3-requests`.

**Prevent →** Never install into system Python; bake venvs into images and deploy scripts.

</details>

### Q103. `ImportError: attempted relative import with no known parent package` when running `python cloudaudit/cli.py`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Running a module file directly makes it `__main__` with no package context, so `from .checks import ...` cannot resolve.

**Diagnose →**
```bash
head -20 cloudaudit/cli.py | grep "from \."
```

**Fix →** Run it as a module from the project root: `python -m cloudaudit.cli`, or install the package (`pip install -e .`) and use the console script `cloudaudit`.

**Prevent →** Use the `src/` layout with entry points; never run package modules by file path.

</details>

### Q104. `AttributeError: partially initialized module 'yaml' has no attribute 'safe_load' (most likely due to a circular import)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your own file is named `yaml.py` (or there is a `yaml/` folder) in the script directory, shadowing PyYAML. Less often, a genuine circular import between your modules.

**Diagnose →**
```bash
ls *.py | grep -E '^(yaml|json|requests|boto3|logging)\.py$'
python -c "import yaml; print(yaml.__file__)"
```

**Fix →** Rename the file (`yaml_utils.py`) and delete `__pycache__/`. For true circular imports, move shared code to a third module or import inside the function.

**Prevent →** Never name files after stdlib or third-party packages; ruff rule `A005` flags stdlib-shadowing module names.

</details>

### Q105. `ERROR: Could not find a version that satisfies the requirement boto3==1.40.99 (from versions: ...)` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The version does not exist, your Python version is too old for it (`Requires-Python`), or pip is pointed at a private index/mirror that has not synced it.

**Diagnose →**
```bash
python --version
pip index versions boto3
pip config list; echo $PIP_INDEX_URL
pip install boto3==1.40.99 -v 2>&1 | grep -i "requires-python\|Looking in indexes"
```

**Fix →** Use an existing version, upgrade Python, or fix the index configuration.

**Prevent →** Generate locks with the same Python version as runtime; let Renovate/Dependabot propose versions.

</details>

### Q106. `ERROR: ResolutionImpossible: ... The conflict is caused by: botocore 1.35.x depends on urllib3<1.27 ... requests 2.x depends on urllib3>=2` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two dependencies require incompatible versions of a shared package, often from overly strict pins or an old library.

**Diagnose →**
```bash
pip install pipdeptree && pipdeptree -r -p urllib3
pip install -r requirements.in --dry-run --report - | python -m json.tool | head
```

**Fix →** Upgrade the package with the outdated constraint (newer botocore relaxed its urllib3 bound for Python 3.10+), loosen your own pins to ranges in `requirements.in`, and recompile the lock.

**Prevent →** Pin direct deps with ranges and lock with a resolver; update dependencies regularly in small batches.

</details>

### Q107. `pip install` on `python:3.12-alpine` takes 15 minutes and ends with `error: command 'gcc' failed: No such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No prebuilt musllinux wheel for a package (or its version), so pip builds from sdist, which needs a compiler and headers.

**Diagnose →**
```bash
pip install -v <pkg> 2>&1 | grep -E "Building wheel|Downloading .*\.tar\.gz"
```

**Fix →** Switch to `python:3.12-slim` (glibc, manylinux wheels exist for almost everything). If you must use Alpine, install build deps in a builder stage (`apk add build-base libffi-dev`) and copy only the built wheels.

**Prevent →** Default to slim for Python images; check wheel availability before choosing a base.

</details>

### Q108. `zsh: no matches found: .[dev]` when running `pip install -e .[dev]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** zsh treats `[dev]` as a glob pattern.

**Diagnose →** The error comes from the shell, before pip runs.

**Fix →** Quote it: `pip install -e ".[dev]"`.

**Prevent →** Always quote extras in docs and scripts.

</details>

### Q109. `bash: .venv/bin/activate: No such file or directory` in a CI job, although the venv was created in an earlier step. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The working directory changed, the venv was created in a different job (jobs do not share filesystems), or `python3-venv` is missing so creation failed silently earlier (`ensurepip is not available`).

**Diagnose →**
```bash
pwd; ls -la .venv/bin 2>&1
python3 -m venv .venv; echo $?
```

**Fix →** Create the venv in the same job, use absolute paths, install `python3-venv` on Debian/Ubuntu runners, or use `actions/setup-python` and skip the venv.

**Prevent →** Each CI job is a fresh machine; share artifacts or caches explicitly.

</details>

### Q110. `SyntaxError: invalid syntax` on a line with `match command:` or `X | None` on a server. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server runs an older Python than you developed on: `match` needs 3.10+, `X | None` in annotations evaluated at runtime needs 3.10+, `except*` needs 3.11+.

**Diagnose →**
```bash
python3 --version
head -1 script.py      # check the shebang target
```

**Fix →** Run with the right interpreter (venv or container), or add `from __future__ import annotations` for annotation-only usage.

**Prevent →** Declare `requires-python = ">=3.11"` and test on the minimum version in CI.

</details>

### Q111. `/usr/bin/env: 'python': No such file or directory` when executing a script directly. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shebang `#!/usr/bin/env python`, but modern distros only ship `python3` (no `python` alias).

**Diagnose →**
```bash
head -1 script.py; which python python3
```

**Fix →** Change the shebang to `#!/usr/bin/env python3`, or point it at the venv interpreter, or install `python-is-python3` on Ubuntu.

**Prevent →** Ship tools as packages with console scripts instead of loose files.

</details>

### Q112. `/bin/bash^M: bad interpreter: No such file or directory` or `env: 'python3\r': No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file has Windows CRLF line endings; the kernel reads `python3\r` as the interpreter name.

**Diagnose →**
```bash
file script.py           # "with CRLF line terminators"
head -1 script.py | od -c | head -2
```

**Fix →** `sed -i 's/\r$//' script.py` or `dos2unix script.py`.

**Prevent →** `.gitattributes` with `* text=auto eol=lf`; editor configured for LF.

</details>

### Q113. `pip` warns `WARNING: Running pip as the 'root' user can result in broken permissions` during a Docker build. Should you care? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Installing as root into the global site-packages in a container.

**Diagnose →** Look at the Dockerfile: `RUN pip install ...` before any `USER`.

**Fix →** It is mostly harmless in a throwaway container layer, but the clean approach is a venv in a builder stage (`python -m venv /opt/venv`), copied into the runtime stage, and a non-root `USER` at runtime. Or set `PIP_ROOT_USER_ACTION=ignore` if you accept the trade-off.

**Prevent →** Multi-stage builds with a venv or installed wheel and non-root runtime users.

</details>

### Q114. `ERROR: THESE PACKAGES DO NOT MATCH THE HASHES FROM THE REQUIREMENTS FILE.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The downloaded file differs from the locked hash: a different platform wheel than the one hashed, a re-uploaded file on a private mirror, a proxy altering content, or (rarely) tampering.

**Diagnose →**
```bash
pip download <pkg>==<ver> --no-deps -d /tmp/chk && sha256sum /tmp/chk/*
grep -A3 "<pkg>==" requirements.txt
```

**Fix →** Regenerate the lock with hashes for all target platforms (`pip-compile --generate-hashes` includes all files listed on the index; `uv` does the same). Investigate the mirror if a hash for the same filename changed.

**Prevent →** Lock on CI, not laptops; treat unexplained hash changes as a security incident.

</details>

### Q115. Your package installs fine, but at runtime `FileNotFoundError: .../site-packages/cloudaudit/templates/report.html`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Non-Python files are not included in the wheel; it works in editable/dev mode because files are read from the source tree.

**Diagnose →**
```bash
python -m build && unzip -l dist/*.whl | grep templates
```

**Fix →** Include package data (`[tool.setuptools.package-data] cloudaudit = ["templates/*.html"]`, or `include-package-data = true` with `MANIFEST.in`), and read files with `importlib.resources.files("cloudaudit") / "templates/report.html"` instead of paths relative to `__file__`.

**Prevent →** Test the built wheel in CI (src layout plus installing the wheel, not `-e`).

</details>

### Q116. `cloudaudit: command not found` after `pip install --user .`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** User installs put scripts in `~/.local/bin`, which is not on `PATH`.

**Diagnose →**
```bash
python -m site --user-base     # scripts go in <base>/bin
ls ~/.local/bin; echo $PATH
```

**Fix →** Add `export PATH="$HOME/.local/bin:$PATH"` to the shell profile, or use `pipx install .`, which manages this (`pipx ensurepath`).

**Prevent →** Distribute CLIs with pipx/uv tool; document the PATH step.

</details>

### Q117. `ImportError: cannot import name 'mock_ec2' from 'moto'` after a dependency update. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** moto 5.0 removed per-service decorators (`mock_ec2`, `mock_s3`, ...) in favour of a single `mock_aws`.

**Diagnose →**
```bash
pip show moto | grep Version
grep -rn "from moto import" tests/
```

**Fix →** Replace with `from moto import mock_aws` and `@mock_aws` / `with mock_aws():`. Pin `moto>=5` in dev dependencies.

**Prevent →** Read changelogs for major versions; group test dependencies in Renovate so updates arrive with CI results.

</details>

### Q118. `pkg_resources is deprecated as an API` warnings, or `ModuleNotFoundError: No module named 'pkg_resources'` in a fresh venv. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Old code or a library uses `pkg_resources` (part of setuptools), which is deprecated and not installed by default in new venvs on Python 3.12+.

**Diagnose →**
```bash
grep -rn "pkg_resources" src/ .venv/lib/*/site-packages/ | head
```

**Fix →** Replace with `importlib.metadata.version("pkg")` and `importlib.resources.files(...)`. For third-party code, upgrade the library; as a stopgap install `setuptools`.

**Prevent →** Run tests with `-W error::DeprecationWarning` for your own code.

</details>

### Q119. `ModuleNotFoundError: No module named 'distutils'` on Python 3.12. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `distutils` was removed from the standard library in Python 3.12 (PEP 632); old packages or scripts still import it.

**Diagnose →**
```bash
python --version
pip install <pkg> -v 2>&1 | grep -i distutils
```

**Fix →** Upgrade the offending package (most fixed this long ago). `setuptools` provides a compatibility shim, so installing a recent setuptools often resolves it. Replace `distutils.util.strtobool` and similar in your code with small local functions.

**Prevent →** Test on new Python versions before upgrading base images.

</details>

### Q120. In VS Code tests pass, but in the terminal `pytest` shows `ModuleNotFoundError: No module named 'cloudaudit'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The package is not installed in the active venv (IDE sets `PYTHONPATH` or uses a different interpreter), or `pytest` on PATH is a global one.

**Diagnose →**
```bash
which pytest; python -m pytest --version
python -c "import cloudaudit, sys; print(cloudaudit.__file__, sys.executable)"
```

**Fix →** `pip install -e ".[dev]"` in the venv and run `python -m pytest`. Or set `pythonpath = ["src"]` in `[tool.pytest.ini_options]`.

**Prevent →** Install the project into its venv; avoid relying on IDE path magic.

</details>

### Q121. `ImportError: libssl.so.1.1: cannot open shared object file` after copying a venv from one machine to another. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Venvs are not portable: they hardcode the interpreter path and compiled extensions link against system libraries that differ between hosts (OpenSSL 1.1 vs 3).

**Diagnose →**
```bash
head -1 .venv/bin/pip            # absolute path to original interpreter
ldd .venv/lib/python3*/site-packages/**/*.so 2>/dev/null | grep "not found"
```

**Fix →** Recreate the venv on the target from the lock file. To ship prebuilt, use a container image or build wheels on the target platform.

**Prevent →** Ship code plus lock, or containers; never copy venvs across machines or OS versions.

</details>

### Q122. `pip` hangs or fails with `Could not fetch URL https://pypi.org/simple/...: There was a problem confirming the ssl certificate` behind the corporate proxy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A TLS-intercepting proxy presents a corporate CA that pip's bundled certificates do not trust, or the proxy is not configured.

**Diagnose →**
```bash
echo $HTTPS_PROXY $PIP_CERT
pip install requests -v 2>&1 | grep -i "ssl\|proxy"
openssl s_client -connect pypi.org:443 -showcerts </dev/null | head -20
```

**Fix →** `pip config set global.cert /etc/ssl/certs/corp-ca.pem` (or `PIP_CERT`), set `HTTPS_PROXY`, or use the internal mirror via `PIP_INDEX_URL`. Recent pip versions can also use the OS trust store via `truststore`.

**Prevent →** Bake the corporate CA and index config into base images and runner setup.

</details>

### Q123. Your Docker build reinstalls all Python dependencies on every code change, taking 4 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `COPY . .` happens before `pip install`, so any source change invalidates the install layer cache.

**Diagnose →** Read the Dockerfile order; `docker build --progress=plain` shows which step starts missing cache.

**Fix →**
```dockerfile
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
COPY src/ ./src/
```

**Prevent →** Copy dependency manifests first, source last; use BuildKit cache mounts.

</details>

### Q124. `mypy` reports `error: Library stubs not installed for "requests"` and `Skipping analyzing "boto3": module is installed, but missing library stubs or py.typed marker`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The libraries do not ship type information; mypy needs separate stub packages.

**Diagnose →** Read the hint mypy prints (`python3 -m pip install types-requests`).

**Fix →** `pip install types-requests types-PyYAML "boto3-stubs[ec2,s3]"`, or `mypy --install-types`. For libraries without stubs, set `ignore_missing_imports = true` for that module in an override.

**Prevent →** Keep stub packages in dev dependencies with the same update group as the library.

</details>

### Q125. `ruff` passes locally but fails in CI with different rules or versions. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Different ruff versions (new rules or changed defaults), or config not picked up in CI (running from another directory).

**Diagnose →**
```bash
ruff --version
ruff check --show-settings src/cloudaudit/cli.py | grep -i "select\|target"
```

**Fix →** Pin ruff in dev deps and in the pre-commit config (same version), and set `select` explicitly in `[tool.ruff.lint]` rather than relying on defaults.

**Prevent →** One pinned tool version used by pre-commit, local and CI.

</details>

---

## Runtime, file, subprocess & encoding errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `KeyError: 'Tags'` while processing `describe_instances` output. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWS omits the `Tags` key entirely for untagged resources rather than returning an empty list.

**Diagnose →**
```python
print(json.dumps(inst, default=str, indent=2))   # look for missing keys
```

**Fix →** `tags = {t["Key"]: t["Value"] for t in inst.get("Tags", [])}`. Same for `PublicIpAddress`, `IamInstanceProfile`, `Platform`.

**Prevent →** Use `.get()` for optional fields; type responses with boto3-stubs, which marks them as `NotRequired`.

</details>

### Q127. `TypeError: can't compare offset-naive and offset-aware datetimes` in a snapshot-age check. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** boto3 returns timezone-aware datetimes (`StartTime`) and the code compares with naive `datetime.now()` or `datetime.utcnow()`.

**Diagnose →**
```python
print(snap["StartTime"].tzinfo, datetime.now().tzinfo)
```

**Fix →** `from datetime import UTC, datetime`; `cutoff = datetime.now(UTC) - timedelta(days=90)`.

**Prevent →** Always use aware UTC datetimes; ruff rule `DTZ005` flags naive `now()`.

</details>

### Q128. `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte` reading a log file. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is not UTF-8 (binary, UTF-16 with BOM `0xff 0xfe`, Latin-1), is gzip-compressed, or contains a few corrupted bytes.

**Diagnose →**
```bash
file app.log
head -c 16 app.log | xxd
```

**Fix →** Use the right encoding (`utf-16`, `latin-1`), `gzip.open(path, "rt")` for compressed files, or `errors="replace"` for mostly-UTF-8 logs.

**Prevent →** Detect gzip by magic bytes or extension; standardize application logs on UTF-8.

</details>

### Q129. `UnicodeEncodeError: 'ascii' codec can't encode character '✓'` when printing in a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container has no UTF-8 locale (`LANG` unset, `C`/`POSIX`), so stdout encoding falls back to ASCII on older Pythons or tools.

**Diagnose →**
```bash
python -c "import sys, locale; print(sys.stdout.encoding, locale.getpreferredencoding())"
echo $LANG $LC_ALL
```

**Fix →** Set `ENV LANG=C.UTF-8` (or `PYTHONIOENCODING=utf-8`, or `PYTHONUTF8=1`) in the Dockerfile.

**Prevent →** Base images with a UTF-8 locale; explicit encodings everywhere.

</details>

### Q130. `PermissionError: [Errno 13] Permission denied: '/var/log/app/report.json'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process user cannot write there: wrong owner, non-root container user writing to a root-owned mount, read-only filesystem, or SELinux labels.

**Diagnose →**
```bash
id; ls -ld /var/log/app; ls -l /var/log/app/report.json
mount | grep " /var/log "
ls -Z /var/log/app        # SELinux
```

**Fix →** Write to a directory the user owns (`/tmp`, a volume with correct `fsGroup`/ownership), fix ownership, or for bind mounts add `:Z` on SELinux hosts.

**Prevent →** Make output paths configurable; set Kubernetes `securityContext.fsGroup` for volumes.

</details>

### Q131. `OSError: [Errno 24] Too many open files` in a long-running Python service. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File or socket leaks (files opened without `with`, a new `requests.Session` or boto3 client per request never closed), or the limit is genuinely too low for the concurrency.

**Diagnose →**
```bash
ls /proc/<pid>/fd | wc -l; cat /proc/<pid>/limits | grep "open files"
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

**Fix →** Use `with` for files; create one Session/client and reuse it; close responses (`with s.get(..., stream=True) as r:`). Raise `LimitNOFILE` in systemd or `ulimit -n` only if usage is legitimate.

**Prevent →** Track `process_open_fds` from `prometheus_client` default metrics and alert on growth.

</details>

### Q132. `subprocess.CalledProcessError: Command '['terraform', 'plan']' returned non-zero exit status 1.` with no further detail. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The command failed, but stderr was not captured or not logged.

**Diagnose →**
```python
try:
    subprocess.run(cmd, check=True, capture_output=True, text=True)
except subprocess.CalledProcessError as e:
    print(e.stderr)
```

**Fix →** Capture output and log `e.stdout` and `e.stderr` on failure (or let output stream to the console if not capturing).

**Prevent →** A shared `run()` helper that always logs the command, exit code and stderr on failure.

</details>

### Q133. `FileNotFoundError: [Errno 2] No such file or directory: 'kubectl'` from `subprocess.run`, though `kubectl` works in your terminal. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The executable is not on the `PATH` of the process (cron, systemd and containers have minimal PATHs), or it is a shell alias/function.

**Diagnose →**
```bash
type kubectl; which kubectl
python -c "import shutil; print(shutil.which('kubectl'))"
```

**Fix →** Use the absolute path, set `PATH` in the unit/cron/Dockerfile, or check with `shutil.which()` and exit with a clear message.

**Prevent →** Validate required binaries at startup.

</details>

### Q134. `subprocess.TimeoutExpired: Command '['ssh', 'web1', 'uptime']' timed out after 30 seconds`, and the script hangs anyway. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `subprocess.run(timeout=...)` kills the direct child, but grandchildren (an SSH ControlMaster, a shell's children) keep the pipes open, so reading output blocks.

**Diagnose →**
```bash
ps -ef --forest | grep -A3 ssh
```

**Fix →** Start the child in its own process group (`start_new_session=True`) and on timeout kill the whole group with `os.killpg(proc.pid, signal.SIGKILL)`; add `-o ConnectTimeout=5 -o BatchMode=yes` to SSH so it cannot wait for a password prompt.

**Prevent →** Non-interactive flags on every remote command and process-group cleanup in your `run()` helper.

</details>

### Q135. `json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)` calling `resp.json()`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The response body is empty or not JSON: an HTML error page from a proxy or load balancer, a 204 No Content, a redirect to a login page, or an auth failure.

**Diagnose →**
```python
print(resp.status_code, resp.headers.get("Content-Type"), resp.text[:500])
```

**Fix →** Call `resp.raise_for_status()` before `.json()`, check `Content-Type`, handle 204 explicitly, and fix the underlying auth or URL.

**Prevent →** A client wrapper that logs status and the first bytes of the body on parse failure.

</details>

### Q136. `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref'` loading a CloudFormation template. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CloudFormation short-form intrinsic functions (`!Ref`, `!Sub`, `!GetAtt`) are custom YAML tags that `safe_load` does not know.

**Diagnose →**
```bash
grep -n "!\(Ref\|Sub\|GetAtt\|If\)" template.yaml | head
```

**Fix →** Register constructors on a `SafeLoader` subclass that turn tags into `{"Ref": value}` dicts, or use the `cfn-flip` library to convert, or `cfn-lint` for validation.

**Prevent →** Use a CloudFormation-aware loader for templates; keep plain `safe_load` for ordinary YAML.

</details>

### Q137. `yaml.composer.ComposerError: expected a single document in the stream` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file contains multiple YAML documents separated by `---` (typical Kubernetes manifests).

**Diagnose →**
```bash
grep -c '^---' manifests.yaml
```

**Fix →** `docs = [d for d in yaml.safe_load_all(f) if d]`.

**Prevent →** Always use `safe_load_all` for manifest files.

</details>

### Q138. A YAML value `country: NO` is loaded as `False`, and version `1.10` becomes `1.1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PyYAML implements YAML 1.1, where `yes/no/on/off` (any case) are booleans, and unquoted numbers are parsed as floats.

**Diagnose →**
```python
yaml.safe_load("a: NO\nb: 1.10\nc: 010")   # {'a': False, 'b': 1.1, 'c': 8}
```

**Fix →** Quote strings in the YAML (`"NO"`, `"1.10"`), or use `ruamel.yaml` (YAML 1.2 by default, where `NO` is a string) for files you do not control.

**Prevent →** Lint YAML with `yamllint` (`truthy` rule) and quote version strings.

</details>

### Q139. `csv.Error: field larger than field limit (131072)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A field exceeds the csv module's default size limit, often because a quote is unbalanced and the parser swallows the rest of the file into one field.

**Diagnose →**
```bash
awk -F'"' 'NF % 2 == 0 {print NR": "$0}' data.csv | head
```

**Fix →** Fix the malformed quoting at the source. If fields are genuinely large, raise the limit with `csv.field_size_limit(sys.maxsize)` (on some platforms use a smaller value like `2**31 - 1`).

**Prevent →** Always produce CSV with the csv module (correct quoting), not string joins.

</details>

### Q140. `RecursionError: maximum recursion depth exceeded` in a function that walks nested JSON. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Very deep structures, or a cycle (an object referencing itself), or a missing base case.

**Diagnose →** Read the repeating frames in the traceback; check for self-referencing structures.

**Fix →** Fix the base case; for deep data rewrite as an iterative walk with an explicit stack; track visited `id()`s to break cycles. Raising `sys.setrecursionlimit` is a last resort and can segfault.

**Prevent →** Prefer iterative traversal for untrusted, arbitrarily deep input.

</details>

### Q141. `MemoryError` or the container gets `OOMKilled` while processing a big API export. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Loading the whole dataset into memory: `resp.json()` on a huge body, building giant lists, `pandas.read_csv` on multi-GB files, keeping all pages before processing.

**Diagnose →**
```bash
kubectl describe pod <pod> | grep -A3 "Last State"      # Reason: OOMKilled
python -X tracemalloc=5 script.py                        # or memray run script.py
```

**Fix →** Stream: process page by page with generators, `resp.iter_lines()`/JSON Lines, `pandas` with `chunksize`, write results incrementally.

**Prevent →** Set memory limits and test with production-size data.

</details>

### Q142. `NameError: name 'log' is not defined` only when an error path runs in production. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo or missing import in a rarely executed branch (often an `except` block), which Python only detects when that line runs.

**Diagnose →** Run `ruff check` (`F821 Undefined name`) and `mypy`; both find this statically.

**Fix →** Define/import the name; add a test that exercises the error path.

**Prevent →** Linting in CI catches this class of bug in seconds; test failure paths, not just the happy path.

</details>

### Q143. `TypeError: 'NoneType' object is not subscriptable` in `pod.status.container_statuses[0]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kubernetes client fields are `None` until populated: pending pods have no `container_statuses`; `last_state.terminated` is `None` if the container never terminated.

**Diagnose →**
```python
print(pod.status.phase, pod.status.container_statuses)
```

**Fix →** Guard: `for cs in pod.status.container_statuses or []:` and check `cs.last_state.terminated is not None`.

**Prevent →** Treat every nested API field as optional; use type hints so mypy flags `Optional` access.

</details>

### Q144. A cron job fails with `KeyError: 'HOME'` or cannot find `~/.aws/config`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cron and systemd run with a minimal environment: no `HOME` in some setups, no profile scripts, different user, no `AWS_PROFILE`.

**Diagnose →**
```bash
* * * * * env > /tmp/cron-env.txt     # temporary crontab entry to inspect
```

**Fix →** Set needed variables explicitly in the crontab or systemd unit (`Environment=HOME=/home/ops AWS_PROFILE=audit`), or better use an instance role so no config file is needed.

**Prevent →** Use systemd units with explicit environment; avoid depending on login-shell state.

</details>

### Q145. `OSError: [Errno 18] Invalid cross-device link` when calling `os.rename`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `os.rename`/`os.replace` cannot move across filesystems (for example `/tmp` tmpfs to `/data` on disk, or between Docker volumes).

**Diagnose →**
```bash
df /tmp /data      # different devices
```

**Fix →** Use `shutil.move()` (copies then deletes), or create the temp file in the destination directory so the final `os.replace` stays atomic on one filesystem.

**Prevent →** For atomic writes, always create temp files next to the target (`tempfile.NamedTemporaryFile(dir=target_dir)`).

</details>

### Q146. Logs appear twice for every message after you added a handler. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A handler was added to your named logger and the root logger also has one, so records propagate and are emitted twice; or `basicConfig`/`addHandler` runs on every function call or import.

**Diagnose →**
```python
print(logging.getLogger("cloudaudit").handlers, logging.getLogger().handlers)
```

**Fix →** Configure handlers once, on the root logger only, in the entry point; or set `propagate = False` on the logger that has its own handler.

**Prevent →** Modules only call `getLogger(__name__)`; only `main()` configures.

</details>

### Q147. `logging.basicConfig(level=logging.DEBUG)` has no effect; you still see only warnings. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `basicConfig` does nothing if the root logger already has handlers, which happens when an imported library or earlier code configured logging first.

**Diagnose →**
```python
print(logging.getLogger().handlers)
```

**Fix →** Call `logging.basicConfig(..., force=True)` (3.8+) at the start of `main()`, or set the level on the specific logger.

**Prevent →** Do not configure logging at import time in any module.

</details>

### Q148. Your script ends with `BrokenPipeError: [Errno 32] Broken pipe` when piped to `head`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `head` exits after reading enough lines and closes the pipe; Python's next write to stdout fails.

**Diagnose →** `python report.py | head -5` reproduces it.

**Fix →** Handle it at the entry point:

```python
try:
    main()
except BrokenPipeError:
    devnull = os.open(os.devnull, os.O_WRONLY)
    os.dup2(devnull, sys.stdout.fileno())
    sys.exit(1)
```
The Python docs recommend this pattern.

**Prevent →** Include it in your CLI template; click already handles many cases.

</details>

### Q149. `ValueError: too many values to unpack (expected 2)` parsing `KEY=VALUE` lines from an env file. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value contains `=` (base64 strings, URLs with query strings), so `line.split("=")` returns more than two parts.

**Diagnose →** Print the failing line.

**Fix →** `key, _, value = line.partition("=")` or `line.split("=", 1)`; skip blank lines and `#` comments; strip surrounding quotes.

**Prevent →** Use `python-dotenv` for env files rather than hand-written parsers.

</details>

### Q150. `UnboundLocalError: cannot access local variable 'count' where it is not associated with a value` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The function assigns to `count`, which makes it local everywhere in the function, but reads it before the assignment (intending the global); or the variable is only assigned inside an `if`/`try` branch that did not run.

**Diagnose →** Find all assignments to the name inside the function.

**Fix →** Initialize the variable before branches, pass it in as a parameter and return the new value, or use a class/closure. Avoid `global`.

**Prevent →** ruff and mypy flag possibly-undefined variables; keep functions free of hidden global state.

</details>

---

## boto3, HTTP & API errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. `botocore.exceptions.NoCredentialsError: Unable to locate credentials` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No source in the credential chain: no env vars, no `~/.aws` profile for this user, no instance/pod role, or IMDS unreachable (hop limit in containers, IMDSv2-only).

**Diagnose →**
```bash
aws sts get-caller-identity
env | grep ^AWS_; ls -la ~/.aws
curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60"
```

**Fix →** Configure the right source: `aws sso login --profile x` + `AWS_PROFILE`, attach an instance profile, configure OIDC in CI, IRSA/Pod Identity in EKS. For containers on EC2 using IMDS, raise the metadata hop limit to 2.

**Prevent →** Document the expected credential source; check `get_caller_identity()` at startup and log the ARN.

</details>

### Q152. `botocore.exceptions.NoRegionError: You must specify a region.` even though `AWS_REGION` is set. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** boto3 reads `AWS_DEFAULT_REGION` and the profile's `region`, not `AWS_REGION`.

**Diagnose →**
```bash
env | grep REGION
python -c "import boto3; print(boto3.Session().region_name)"
```

**Fix →** Set `AWS_DEFAULT_REGION`, add `region` to the profile, or pass `region_name=` explicitly.

**Prevent →** Pass `region_name` explicitly in multi-region code; set both variables in CI.

</details>

### Q153. `botocore.exceptions.ClientError: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The identity lacks `ec2:DescribeInstances`, an SCP or permission boundary denies it, or you are in the wrong account/role.

**Diagnose →**
```bash
aws sts get-caller-identity
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names ec2:DescribeInstances
```
For EC2 write calls the message includes an encoded authorization message: `aws sts decode-authorization-message --encoded-message <msg>`.

**Fix →** Add the permission to the role (least privilege), or switch to the correct role.

**Prevent →** Keep the tool's IAM policy in code next to the tool; test with a sandbox role.

</details>

### Q154. `An error occurred (ExpiredToken) when calling the ... operation: The security token included in the request is expired` midway through a long job. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job built a Session from static STS credentials (manual `assume_role`) that expired after 1 hour; or SSO session expired.

**Diagnose →** Check how the Session is created and the job duration vs `DurationSeconds`.

**Fix →** Use a profile with `role_arn` + `source_profile`/`credential_source` so botocore refreshes automatically, or use `botocore.credentials.RefreshableCredentials` with a refresh callback. Increase role `MaxSessionDuration` if needed.

**Prevent →** Let botocore manage assume-role; avoid passing raw keys into `Session()`.

</details>

### Q155. `An error occurred (RequestLimitExceeded) when calling the DescribeInstances operation (reached max retries: 4): Request limit exceeded.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many EC2 API calls per second from your tool (parallel workers, per-resource describes) or other tools sharing the account's quota.

**Diagnose →** Count calls per phase with debug logs; CloudTrail shows the call volume by user agent.

**Fix →** `Config(retries={"mode": "adaptive", "max_attempts": 10})`, reduce concurrency, batch IDs into fewer calls, filter server-side.

**Prevent →** Design scanners around batch APIs and bounded concurrency from the start.

</details>

### Q156. `An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation: The security token included in the request is invalid.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deleted or wrong access key, a stale `AWS_SESSION_TOKEN` exported alongside new keys, or calling a region not enabled for the account.

**Diagnose →**
```bash
env | grep ^AWS_
aws configure list
```

**Fix →** Unset stale variables (`unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN`), re-authenticate, or enable the region.

**Prevent →** Avoid exporting keys in shells; use profiles/SSO.

</details>

### Q157. `An error occurred (SignatureDoesNotMatch)` or `RequestTimeTooSkewed` from AWS. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host clock is off by more than 5 minutes (SigV4 includes a timestamp), the secret key is wrong (copy/paste whitespace), or a proxy alters the request.

**Diagnose →**
```bash
date -u; timedatectl status; chronyc tracking
```

**Fix →** Sync time (`chronyd`/`systemd-timesyncd`, Amazon Time Sync at 169.254.169.123). Re-copy credentials.

**Prevent →** NTP on every host and container host; monitor clock offset.

</details>

### Q158. `botocore.exceptions.EndpointConnectionError: Could not connect to the endpoint URL: "https://ec2.ap-south-1.amazonaws.com/"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No network path: private subnet without NAT or VPC endpoint, DNS failure, proxy required, security group/NACL egress blocked, or a typo in the region.

**Diagnose →**
```bash
nslookup ec2.ap-south-1.amazonaws.com
curl -sv https://ec2.ap-south-1.amazonaws.com/ -o /dev/null
echo $HTTPS_PROXY $NO_PROXY
```

**Fix →** Add a NAT gateway or interface VPC endpoints (`com.amazonaws.ap-south-1.ec2`), set `HTTPS_PROXY`, or fix the region name.

**Prevent →** Network checks in your deployment smoke tests; VPC endpoints for private workloads.

</details>

### Q159. `botocore.errorfactory.NoSuchKey: An error occurred (NoSuchKey) when calling the GetObject operation`, but `head_object` on another key returns `ClientError` with code `404`. Why the inconsistency? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `GetObject` returns an XML error body with code `NoSuchKey`, which boto3 maps to the modelled exception. `HeadObject` responses have no body, so the error code is just the HTTP status `404`. Also, without `s3:ListBucket` permission, S3 returns `403 AccessDenied` for missing keys instead of 404.

**Diagnose →**
```python
except ClientError as e: print(e.response["Error"]["Code"], e.response["ResponseMetadata"]["HTTPStatusCode"])
```

**Fix →** Handle both: `if e.response["Error"]["Code"] in ("404", "NoSuchKey")`. Grant `s3:ListBucket` if you need to distinguish missing from forbidden.

**Prevent →** Wrap S3 existence checks in one helper with tests for each case.

</details>

### Q160. `An error occurred (AccessDenied) when calling the PutObject operation: Access Denied` although the IAM policy allows `s3:PutObject`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something else denies: the bucket policy (for example requiring `aws:kms` encryption or TLS), SSE-KMS key policy missing `kms:GenerateDataKey` for your role, an SCP, a VPC endpoint policy, or Object Ownership/ACL settings when passing `ACL=`.

**Diagnose →**
```bash
aws s3api get-bucket-policy --bucket b | jq -r .Policy | jq .
aws s3api get-bucket-encryption --bucket b
```
Check CloudTrail for the denied event and its `errorMessage`.

**Fix →** Add the required headers (`ServerSideEncryption="aws:kms"`, `SSEKMSKeyId=...`), grant KMS permissions, or adjust the bucket policy.

**Prevent →** Document bucket requirements; test uploads with the real role in staging.

</details>

### Q161. `botocore.exceptions.WaiterError: Waiter InstanceRunning failed: Waiter encountered a terminal failure state: For expression "Reservations[].Instances[].State.Name" we matched expected path: "terminated"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The instance terminated right after launch: insufficient capacity, EBS/KMS key permission issue on an encrypted volume, invalid AMI/instance type combination, or a Spot interruption.

**Diagnose →**
```bash
aws ec2 describe-instances --instance-ids i-... --query "Reservations[].Instances[].StateReason"
```
Common reason: `Client.InternalError: Client error on launch` (often KMS permissions for encrypted EBS).

**Fix →** Fix the root cause shown in `StateReason` (grant the role/ASG service-linked role access to the KMS key, change AZ or instance type).

**Prevent →** Log `StateReason` automatically when a waiter fails.

</details>

### Q162. `requests.exceptions.SSLError: HTTPSConnectionPool(host='artifactory.internal', port=443): Max retries exceeded ... [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server uses an internal CA (or a TLS-intercepting proxy) that requests' `certifi` bundle does not trust; or the server does not send its intermediate certificate.

**Diagnose →**
```bash
openssl s_client -connect artifactory.internal:443 -showcerts </dev/null
python -c "import certifi; print(certifi.where())"
```

**Fix →** `REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt` (with the corporate CA installed), or `verify="/path/corp-ca.pem"`; fix the server chain if intermediates are missing. Never `verify=False`.

**Prevent →** Install corporate CAs in base images and set `REQUESTS_CA_BUNDLE` and `SSL_CERT_FILE` there.

</details>

### Q163. `requests.exceptions.ConnectionError: HTTPSConnectionPool(host='api.example.com', port=443): Max retries exceeded with url: /v1/items (Caused by NameResolutionError(...: Failed to resolve 'api.example.com'))` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS failure: wrong hostname, private DNS not reachable from this network, `/etc/resolv.conf` misconfigured in the container, or no network.

**Diagnose →**
```bash
getent hosts api.example.com; nslookup api.example.com
cat /etc/resolv.conf
```

**Fix →** Correct the hostname or DNS configuration; connect to the VPN/VPC that resolves private zones.

**Prevent →** Retry transient DNS errors with backoff; alert on resolution failures separately from HTTP errors.

</details>

### Q164. A script hangs forever on `requests.get(url)` and never errors. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `timeout` argument; requests waits indefinitely for a server that accepted the connection but never responds (or a black-holed network path).

**Diagnose →**
```bash
py-spy dump --pid <pid>     # shows the thread blocked in socket recv
```

**Fix →** Always pass `timeout=(connect, read)`, e.g. `timeout=(3.05, 30)`.

**Prevent →** Wrap requests in a Session subclass or helper that enforces a default timeout; Bandit (via ruff `S113`) flags requests without timeouts.

</details>

### Q165. `requests.exceptions.HTTPError: 429 Client Error: Too Many Requests for url: https://api.github.com/...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You exceeded the rate limit (GitHub: 5,000 requests/hour per token for REST, plus secondary limits on concurrency and content creation).

**Diagnose →**
```python
print(resp.headers.get("Retry-After"), resp.headers.get("X-RateLimit-Remaining"), resp.headers.get("X-RateLimit-Reset"))
```

**Fix →** Honour `Retry-After` / wait until `X-RateLimit-Reset`; use `Retry(status_forcelist=[429], respect_retry_after_header=True)`; reduce concurrency; use conditional requests (ETag) and GraphQL to cut call volume; use a GitHub App token for higher limits.

**Prevent →** Track remaining quota and slow down proactively.

</details>

### Q166. `urllib3.exceptions.MaxRetryError: ... (Caused by ResponseError('too many 503 error responses'))` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your retry adapter retried on 503 until `total` was exhausted; the upstream is genuinely down, not briefly unavailable.

**Diagnose →** Check upstream health and how long retries took (backoff settings).

**Fix →** Treat it as an outage: fail clearly, alert, and do not retry harder. Consider a circuit breaker so you stop hammering a failing service.

**Prevent →** Bound total retry time; expose retry counts as metrics so persistent failure is visible early.

</details>

### Q167. `401 Unauthorized` from an internal API only for requests that follow a redirect. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** requests strips the `Authorization` header when redirected to a different host (to avoid leaking credentials), and the redirect target needs auth.

**Diagnose →**
```python
r = s.get(url, allow_redirects=False); print(r.status_code, r.headers.get("Location"))
```

**Fix →** Call the final URL directly, or handle the redirect yourself and re-add auth only if the new host is trusted.

**Prevent →** Configure clients with canonical endpoints; keep the security behaviour, do not disable it globally.

</details>

### Q168. `requests` uploads work locally but fail in Kubernetes with `ProxyError: Unable to connect to proxy`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `HTTPS_PROXY` is set in the pod (inherited from a base image or cluster-wide injection) but the proxy is unreachable from this namespace, or `NO_PROXY` does not exclude internal hosts like `*.svc.cluster.local` and `169.254.169.254`.

**Diagnose →**
```bash
kubectl exec <pod> -- env | grep -i proxy
```

**Fix →** Fix `NO_PROXY` (`.svc,.cluster.local,10.0.0.0/8,169.254.169.254`), or unset the proxy for in-cluster traffic; `session.trust_env = False` disables env proxies for a specific session.

**Prevent →** Standardize proxy variables at the platform level and test from a pod.

</details>

### Q169. `kubernetes.client.exceptions.ApiException: (403) Reason: Forbidden ... "pods is forbidden: User \"system:serviceaccount:ops:auditor\" cannot list resource \"pods\" in API group \"\" at the cluster scope"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service account has no RBAC permission for that verb/resource/scope (a namespaced Role cannot grant cluster-scope list).

**Diagnose →**
```bash
kubectl auth can-i list pods --all-namespaces --as=system:serviceaccount:ops:auditor
kubectl get rolebinding,clusterrolebinding -A -o wide | grep auditor
```

**Fix →** Create a ClusterRole with `get, list, watch` on `pods` and a ClusterRoleBinding to the service account (or use namespaced calls with a Role per namespace).

**Prevent →** Ship RBAC manifests with the tool and test with `kubectl auth can-i`.

</details>

### Q170. `kubernetes.config.config_exception.ConfigException: Service host/port is not set.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `load_incluster_config()` was called outside a pod (laptop, CI), where `KUBERNETES_SERVICE_HOST` is not set.

**Diagnose →** `env | grep KUBERNETES_SERVICE`.

**Fix →** Fall back to kubeconfig:

```python
try:
    config.load_incluster_config()
except config.ConfigException:
    config.load_kube_config()
```

**Prevent →** Use this fallback pattern in every tool that runs both in and out of cluster.

</details>

### Q171. `ApiException: (410) Reason: Gone ... "too old resource version: 123456 (789012)"` in a watch loop. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The watch resumed from a `resourceVersion` older than the API server's retained history (etcd compaction).

**Diagnose →** The 410 appears after reconnects or long idle periods.

**Fix →** On 410, re-list to get a fresh `resource_version`, reconcile the full state, then start a new watch from that version. Use `allow_watch_bookmarks=True` to keep the version fresh.

**Prevent →** Use frameworks (kopf) or informer-style code that handles relist; design reconciliation to be level-triggered.

</details>

### Q172. `paramiko.ssh_exception.SSHException: Server '10.0.1.15' not found in known_hosts` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default policy is `RejectPolicy`, and the host key is not in the loaded known_hosts (or you never called `load_system_host_keys()`).

**Diagnose →**
```bash
ssh-keygen -F 10.0.1.15
```

**Fix →** Load known hosts (`client.load_system_host_keys()`), and add the host key from a trusted source (provisioning output, `ssh-keyscan` over a trusted network, SSH CA). Avoid switching to `AutoAddPolicy` in production.

**Prevent →** Distribute host keys or SSH certificates as part of provisioning.

</details>

### Q173. `paramiko.ssh_exception.AuthenticationException: Authentication failed.` while `ssh` from the same machine works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** paramiko is not using the same key or agent as OpenSSH: `~/.ssh/config` settings (`IdentityFile`, `User`, `ProxyJump`) are not read by paramiko automatically; the key type or format is unsupported by an old paramiko (for example the newer OpenSSH key format or ed25519 on very old versions); RSA SHA-1 signatures rejected by the server.

**Diagnose →**
```python
paramiko.util.log_to_file("/tmp/paramiko.log")   # then inspect auth attempts
```

**Fix →** Pass `username` and `key_filename` explicitly, or use Fabric, which reads `~/.ssh/config`. Upgrade paramiko.

**Prevent →** Keep SSH settings explicit in tool config; pin a current paramiko.

</details>

### Q174. `docker.errors.DockerException: Error while fetching server API version: ('Connection aborted.', FileNotFoundError(2, 'No such file or directory'))` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Docker SDK cannot find the daemon socket: Docker not running, rootless/desktop socket path differs, or `DOCKER_HOST` unset inside a container without the socket mounted.

**Diagnose →**
```bash
ls -l /var/run/docker.sock; docker context ls; echo $DOCKER_HOST
```

**Fix →** Start Docker, set `DOCKER_HOST` to the right socket (`unix:///run/user/1000/docker.sock` for rootless, or the Desktop context's path), or mount the socket if the design truly requires it.

**Prevent →** Read the active `docker context` and set `DOCKER_HOST` in the tool's environment.

</details>

### Q175. `ImportError: urllib3 v2 only supports OpenSSL 1.1.1+, currently the 'ssl' module is compiled with 'OpenSSL 1.0.2k-fips'` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An old system Python (Amazon Linux 2, CentOS 7) linked against OpenSSL 1.0.2, with a fresh install pulling urllib3 2.x.

**Diagnose →**
```bash
python -c "import ssl; print(ssl.OPENSSL_VERSION)"
pip show urllib3 | grep Version
```

**Fix →** Use a modern Python (container, `uv python install`, newer OS like Amazon Linux 2023). As a short-term workaround, pin `urllib3<2`.

**Prevent →** Do not build on end-of-life OS Pythons; Amazon Linux 2 itself reached end of support on 30 June 2026.

</details>

---

## Concurrency, async, tests, services & containers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. `RuntimeError: asyncio.run() cannot be called from a running event loop` in a Jupyter notebook or inside an async framework. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An event loop is already running (Jupyter, FastAPI, another `asyncio.run` caller), and `asyncio.run` tries to start a new one.

**Diagnose →** Check whether the calling code is itself inside a coroutine or a notebook cell.

**Fix →** In Jupyter, use top-level `await main()`. Inside async code, `await` the coroutine or `asyncio.create_task(...)` instead of calling `asyncio.run`. From sync code in another thread, use `asyncio.run_coroutine_threadsafe(coro, loop)`.

**Prevent →** Only call `asyncio.run` once, at the program's entry point.

</details>

### Q177. `RuntimeWarning: coroutine 'fetch' was never awaited` and nothing happens. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You called an `async def` function without `await` (or without scheduling it), so only a coroutine object was created.

**Diagnose →** Find the call site named in the warning; run with `PYTHONASYNCIODEBUG=1` to get the creation traceback.

**Fix →** `result = await fetch(url)`, or `asyncio.create_task(fetch(url))` and keep a reference to the task.

**Prevent →** mypy and ruff (`RUF006` for dangling tasks) catch many of these; type-check async code.

</details>

### Q178. An asyncio health-checker is no faster than a sequential loop. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Blocking calls inside coroutines (`requests.get`, `time.sleep`, boto3), which freeze the event loop; or awaiting each task one by one in a `for` loop instead of running them concurrently.

**Diagnose →**
```bash
PYTHONASYNCIODEBUG=1 python checker.py   # logs "Executing <Task ...> took 1.2 seconds"
```

**Fix →** Use async clients (`httpx.AsyncClient`, aiohttp), `await asyncio.sleep`, offload unavoidable blocking calls with `asyncio.to_thread`, and run concurrently with `gather` or `TaskGroup`.

**Prevent →** Code review rule: no sync I/O inside `async def`.

</details>

### Q179. `Task was destroyed but it is pending!` at shutdown. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tasks created with `create_task` were still running when the loop closed; or a task was garbage collected because no reference was kept.

**Diagnose →** Enable asyncio debug mode for the task's creation traceback.

**Fix →** Keep references to tasks (a set, or `TaskGroup`), cancel and await them on shutdown:

```python
for t in tasks: t.cancel()
await asyncio.gather(*tasks, return_exceptions=True)
```

**Prevent →** Use `asyncio.TaskGroup` for structured concurrency so tasks cannot outlive their scope.

</details>

### Q180. `OSError: [Errno 24] Too many open files` from an asyncio scanner with 20,000 targets. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All coroutines start at once and each opens a socket; no concurrency bound.

**Diagnose →** `ulimit -n`; count concurrent connections (`ss -s`).

**Fix →** Wrap each request in `async with asyncio.Semaphore(200)`, and configure client connection limits (`httpx.Limits(max_connections=200)`, aiohttp `TCPConnector(limit=200)`).

**Prevent →** Always bound concurrency explicitly; never `gather` an unbounded list of network calls.

</details>

### Q181. A `ThreadPoolExecutor` job finishes "successfully" but half the work is missing, with no errors logged. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Exceptions inside worker functions are stored in the futures and only raised when `.result()` is called; the code submitted tasks and never checked results.

**Diagnose →** Iterate futures and call `f.exception()`.

**Fix →**
```python
for f in as_completed(futures):
    try:
        f.result()
    except Exception:
        log.exception("task %s failed", futures[f])
        failures += 1
sys.exit(1 if failures else 0)
```

**Prevent →** Always consume results; fail the run when any task fails.

</details>

### Q182. Intermittent `KeyError: 'credential_provider'` or `KeyError: 'endpoint_resolver'` when creating boto3 clients from many threads. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clients created concurrently from the shared default session (`boto3.client(...)` inside worker threads). Session creation and component loading are not thread-safe.

**Diagnose →** The error appears only under concurrency and not in sequential runs.

**Fix →** Create one `boto3.session.Session()` per thread and build clients from it, or create the clients in the main thread before starting workers and share them (clients are thread-safe).

**Prevent →** Make "Session per thread" a team convention; our lab's `audit_region` does this.

</details>

### Q183. A multiprocessing script fails on macOS with `RuntimeError: An attempt has been made to start a new process before the current process has finished its bootstrapping phase.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `spawn` start method (default on macOS and Windows) re-imports the main module in each child; without an `if __name__ == "__main__":` guard, the children try to start pools themselves.

**Diagnose →** Check for top-level code that creates `Pool`/`ProcessPoolExecutor`.

**Fix →** Move the entry code under `if __name__ == "__main__": main()`.

**Prevent →** Always guard entry points; avoid side effects at import time.

</details>

### Q184. `_pickle.PicklingError: Can't pickle <function <lambda> at 0x...>` with `ProcessPoolExecutor`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Arguments and functions sent to worker processes must be picklable; lambdas, nested functions, open files, locks and boto3 clients are not.

**Diagnose →** The traceback names the object that failed.

**Fix →** Use a module-level function, pass plain data (IDs, dicts), and create clients inside the worker (a pool `initializer` can set them up once per process).

**Prevent →** Use processes only for CPU-bound work on plain data; use threads for API calls.

</details>

### Q185. A script with threads ignores Ctrl-C and must be killed with `kill -9`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The main thread is blocked in `join()` or `executor.shutdown(wait=True)` and worker threads (non-daemon) keep running; signals are only delivered to the main thread.

**Diagnose →** `py-spy dump --pid <pid>` shows the main thread waiting in `join`.

**Fix →** Use a `threading.Event` checked by workers; in the main thread, wait with timeouts (`as_completed(..., timeout=...)` loops or `event.wait(1)`), catch `KeyboardInterrupt`, set the event, and `shutdown(cancel_futures=True)`.

**Prevent →** Build cooperative cancellation into long-running workers from day one.

</details>

### Q186. pytest with moto fails with `botocore.exceptions.ClientError: An error occurred (InvalidClientTokenId) when calling the ... operation` (it is reaching real AWS). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client was created outside the `mock_aws` context (at module import, in a session-scoped fixture created before the mock started), so it talks to real AWS with fake credentials.

**Diagnose →** Check where clients are created relative to `mock_aws()` / `@mock_aws`.

**Fix →** Create clients inside the mock (in fixtures that depend on the mock fixture), not at import time. Set fake credentials in an autouse fixture so a leak fails fast instead of using your real profile.

**Prevent →** No module-level clients; dependency-inject clients into functions.

</details>

### Q187. A moto-based test expects old snapshots to be flagged, but `describe_snapshots` always shows `StartTime` as now. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** moto stamps resources with the current time and does not let you set creation timestamps through the API.

**Diagnose →** Print `StartTime` in the test.

**Fix →** Test the age logic with a stub (`unittest.mock.MagicMock` paginator or botocore `Stubber`) returning controlled timestamps, or freeze/travel time (`time-machine`) so "now" during the check is far in the future relative to creation.

**Prevent →** Separate the "is it old?" rule into a pure function that takes `now` as a parameter.

</details>

### Q188. `fixture 'mocker' not found` in a pytest run on CI. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `mocker` comes from the `pytest-mock` plugin, which is installed locally but not in CI's dependencies.

**Diagnose →**
```bash
pip show pytest-mock
pytest --fixtures | grep mocker
```

**Fix →** Add `pytest-mock` to dev dependencies and the lock file.

**Prevent →** CI installs exactly from the lock; never rely on globally installed plugins.

</details>

### Q189. Tests pass individually but `pytest -n auto` (xdist) causes random failures. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tests share external state across processes: the same temp file path, same port, same S3 bucket name in a real environment, or a session fixture assumed to run once.

**Diagnose →** Run with `-n 2 -p no:randomly` and look for colliding resource names.

**Fix →** Use `tmp_path`, ephemeral ports (bind to port 0), unique names (`f"test-{uuid4().hex[:8]}"`), and `worker_id`-aware fixtures for shared resources.

**Prevent →** Treat every test as potentially concurrent.

</details>

### Q190. `click.testing.CliRunner` test shows `exit_code == 1` with empty output. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unhandled exception inside the command; CliRunner catches it, sets exit code 1, and stores it in `result.exception`.

**Diagnose →**
```python
print(result.exception); import traceback; traceback.print_exception(*result.exc_info)
```

**Fix →** Fix the underlying error; while debugging, use `CliRunner().invoke(main, args, catch_exceptions=False)` so pytest shows the traceback.

**Prevent →** Include `result.output` and `result.exception` in assertion messages.

</details>

### Q191. Flask app in Docker is unreachable: `curl: (52) Empty reply from server` or connection refused on the published port. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server binds to `127.0.0.1` inside the container (Flask's default), so it only accepts connections from inside the container.

**Diagnose →**
```bash
docker exec <c> ss -ltnp        # shows 127.0.0.1:5000
docker logs <c> | grep "Running on"
```

**Fix →** Bind to all interfaces: `gunicorn -b 0.0.0.0:8000 app:app` or `flask run --host 0.0.0.0`.

**Prevent →** Production server config in the image's `CMD`, not the dev server.

</details>

### Q192. Gunicorn logs `[CRITICAL] WORKER TIMEOUT (pid:42)` and requests fail with 502 from the load balancer. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A request took longer than gunicorn's `--timeout` (30s default) on a sync worker, usually a slow downstream call without a timeout, or CPU-bound work.

**Diagnose →** Correlate timeouts with endpoints in access logs; `py-spy dump` on a worker during a slow request.

**Fix →** Add timeouts to outbound calls, move long jobs to a background queue, use `gthread` or async workers for I/O-bound handlers, and raise `--timeout` only if the work is legitimately long.

**Prevent →** Latency histograms per endpoint and timeouts on all external calls.

</details>

### Q193. FastAPI returns `422 Unprocessable Entity` for a webhook from a third party. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The request body does not match the pydantic model: missing fields, wrong types, or the sender posts `application/x-www-form-urlencoded` (Slack slash commands do) instead of JSON.

**Diagnose →** Read the response body; FastAPI lists the failing fields and reasons. Log `request.headers["content-type"]`.

**Fix →** Adjust the model (optional fields, correct types), or accept `Form(...)` parameters / raw `Request` body for non-JSON senders.

**Prevent →** Model webhooks from real sample payloads and test with them.

</details>

### Q194. GitHub webhook signature verification fails for every request, though the secret is correct. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The HMAC is computed over re-serialized JSON (`json.dumps(request.json)`) instead of the raw body bytes, or the secret has trailing whitespace/newline from the secret store, or comparing against `X-Hub-Signature` (SHA-1) instead of `X-Hub-Signature-256`.

**Diagnose →** Log lengths (never values) of the secret and body; compare computed vs received prefix.

**Fix →** Use `request.get_data()` (Flask) or `await request.body()` (FastAPI) for the HMAC, `.strip()` the secret, and compare `"sha256=" + hexdigest` with `hmac.compare_digest`.

**Prevent →** Unit test the verifier with a known payload, secret and signature.

</details>

### Q195. Prometheus shows counters from your gunicorn app jumping up and down between scrapes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiple gunicorn workers each keep their own metrics; each scrape hits a different worker.

**Diagnose →** Compare `/metrics` output from repeated curls; values differ per request.

**Fix →** Enable `prometheus_client` multiprocess mode: set `PROMETHEUS_MULTIPROC_DIR` (empty, writable) before startup, serve metrics through `MultiProcessCollector`, call `multiprocess.mark_process_dead(worker.pid)` in gunicorn's `child_exit` hook.

**Prevent →** Use the multiprocess setup in your service template whenever workers > 1.

</details>

### Q196. `ValueError: Duplicated timeseries in CollectorRegistry: {'requests_total', 'requests_created', 'requests'}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The same metric is registered twice: module imported twice under different names, metric defined inside a function or app factory that runs more than once, or tests creating the app repeatedly.

**Diagnose →** Find where the metric is constructed and how often that code runs.

**Fix →** Define metrics once at module level; in app factories or tests, pass a dedicated `CollectorRegistry()` per instance.

**Prevent →** Keep metric definitions in one module; use a fresh registry in tests.

</details>

### Q197. A Python container in Kubernetes is `OOMKilled` while the app reports only 200 MB used. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Several gunicorn workers each using memory (4 workers x 200 MB), memory growth after fork as copy-on-write pages are touched, native memory (C extensions, large buffers) outside Python's view, or a limit set too low for the worker count.

**Diagnose →**
```bash
kubectl top pod <pod> --containers
kubectl exec <pod> -- ps -o pid,rss,cmd -e
```

**Fix →** Right-size workers to the memory limit, use `--max-requests`/`--max-requests-jitter` to recycle leaky workers, preload the app (`--preload`) to share memory, profile with `memray` for leaks.

**Prevent →** Load-test with the production worker count and track `container_memory_working_set_bytes`.

</details>

### Q198. Your Python logs do not appear in `docker logs` or `kubectl logs` until the container exits. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** stdout is block-buffered when not attached to a TTY, so output sits in a buffer.

**Diagnose →** Logs appear in bursts or only at exit.

**Fix →** Set `ENV PYTHONUNBUFFERED=1` in the Dockerfile (or run `python -u`), and use `logging` to stderr (unbuffered by default).

**Prevent →** Include `PYTHONUNBUFFERED=1` in every Python base image.

</details>

### Q199. The scheduled GitHub Actions audit fails with `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role trust policy's `sub` condition does not match the token (wrong repo or branch, or the job uses an `environment:`, which changes `sub` to `repo:org/repo:environment:prod`), the `aud` is not `sts.amazonaws.com`, the OIDC provider is missing, or the workflow lacks `permissions: id-token: write`.

**Diagnose →** Check the trust policy conditions against the workflow; scheduled runs use the default branch, so `sub` is `repo:org/repo:ref:refs/heads/main`.

**Fix →** Align the `token.actions.githubusercontent.com:sub` condition (`StringLike` with the exact pattern), add `id-token: write`, verify the provider and audience.

**Prevent →** Keep the trust policy in Terraform next to the workflow; test with `workflow_dispatch` on main before relying on the schedule.

</details>

### Q200. The containerized audit CLI exits 0 in GitHub Actions but the report file is missing from the uploaded artifact. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container wrote the report to a path inside the container that is not bind-mounted, or the mount exists but the non-root container user (UID 10001) cannot write to the runner-owned directory, and the tool wrote to stdout instead or failed to write.

**Diagnose →**
```bash
ls -la reports/
docker run --rm -v "$PWD/reports:/out" --entrypoint id ghcr.io/org/cloudaudit:sha
```

**Fix →** Mount the output directory and write to it (`-o /out/report.md`), and run the container with the runner's UID (`--user "$(id -u):$(id -g)"`) or make the directory writable. Make the tool exit non-zero if it cannot write the output file.

**Prevent →** Add an explicit check step (`test -s reports/report.md`) before upload.

</details>

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

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
