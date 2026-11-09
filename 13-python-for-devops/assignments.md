# Python for DevOps — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Write real code, run it, and commit it to a repo you can show in an interview. Each assignment builds a small tool you would actually use at work.

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Hunt down mutability bugs |
| 2 | 🟢 | Stream-parse an Nginx access log with generators |
| 3 | 🟢 | Reproducible environments with venv and a lock file |
| 4 | 🟢 | YAML and JSON config converter with validation |
| 5 | 🟢 | Disk usage alert with subprocess |
| 6 | 🟢 | Backup rotation CLI with argparse |
| 7 | 🟢 | Replace print with logging |
| 8 | 🟢 | Server inventory report from CSV |
| 9 | 🟢 | HTTP health checker with timeouts and exit codes |
| 10 | 🟢 | Exceptions, cleanup and your own context manager |
| 11 | 🟡 | GitHub API client with retries, pagination and rate limits |
| 12 | 🟡 | Multi-region EC2 inventory with boto3 |
| 13 | 🟡 | S3 bucket security audit |
| 14 | 🟡 | Pod restart reporter with the Kubernetes client |
| 15 | 🟡 | TLS certificate expiry checker with a thread pool |
| 16 | 🟡 | A real pytest suite for your earlier tools |
| 17 | 🟡 | EC2 office-hours scheduler tested with moto |
| 18 | 🟡 | Docker host janitor with the Docker SDK |
| 19 | 🟡 | FastAPI service with health, readiness and metrics |
| 20 | 🟡 | Package, containerize and gate a tool in CI |
| 21 | 🔴 | asyncio fan-out to thousands of endpoints |
| 22 | 🔴 | A Kubernetes watch loop that survives 410 Gone |
| 23 | 🔴 | Signed, idempotent GitHub webhook receiver |
| 24 | 🔴 | Organization-wide AWS audit with assumed roles |
| 25 | 🔴 | Rolling restart orchestrator over SSH with Fabric |

---

## 🟢 Beginner

### Assignment 1. Hunt down mutability bugs 🟢

**Objective:** Understand references, mutability and copying well enough to spot the bugs that cause "random" config corruption in automation scripts.

**Tasks**
1. Write `add_tag(tag, tags=[])` that appends and returns the list. Call it three times without the second argument and print the result each time. Then fix it.
2. Build a default config dict with a nested `"tags": {"team": "platform"}`. Create `prod = default.copy()`, change `prod["tags"]["team"]`, and show that `default` changed too. Fix it with `copy.deepcopy`.
3. Create a 3×3 grid with `[[0] * 3] * 3`, set one cell, and explain the output. Fix it with a comprehension.
4. Write a function that receives a list of hostnames and removes the ones ending in `.decom` while iterating over it. Show the bug, then fix it two ways (iterate over a copy, build a new list).
5. Try to use a list and a dict as dict keys. Record the exact `TypeError` text and explain hashability.
6. Write a one-paragraph explanation of `is` vs `==`, with an example where they differ.

**Expected outcome / acceptance criteria**
- A single `mutability.py` that prints "buggy" and "fixed" output for each case.
- The mutable-default fix uses `None` as the sentinel.
- You can explain in one sentence why `copy()` is shallow and when that matters.

<details>
<summary>Hints</summary>

Default arguments are evaluated once, when `def` runs, so the same list object is reused. `[[0] * 3] * 3` repeats a reference to one inner list. The `TypeError` is `unhashable type: 'list'`. `a is b` compares identity (`id()`), `==` compares value; `[1] == [1]` is `True`, `[1] is [1]` is `False`.

</details>

**Bonus:** Write the same "accidental shared state" bug with a class attribute (`class Host: tags = []`) and fix it in `__init__`.

### Assignment 2. Stream-parse an Nginx access log with generators 🟢

**Objective:** Process a log file of any size in constant memory and produce the numbers an on-call engineer asks for.

**Tasks**
1. Generate a 1 GB sample access log in combined format (write a small generator script, or concatenate a real log many times).
2. Write `read_lines(path)` as a generator that yields one line at a time with `encoding="utf-8", errors="replace"`.
3. Parse each line with a compiled regex into a dict: IP, timestamp, method, path, status, bytes, user agent. Skip and count lines that do not match.
4. Report the top 10 client IPs, requests per status class (2xx, 3xx, 4xx, 5xx), the top 10 paths returning 5xx, and total bytes served.
5. Accept the path as an argument and also read from stdin when the path is `-`, so `zcat access.log.gz | python logstats.py -` works.
6. Measure peak memory with `/usr/bin/time -v` (Linux) and keep it under 50 MB for the 1 GB file.

**Expected outcome / acceptance criteria**
- Memory stays flat regardless of file size.
- Malformed lines are counted, not fatal.
- Output is readable text by default and JSON with `--json`.

<details>
<summary>Hints</summary>

`collections.Counter` with `.most_common(10)`. Compile the regex once outside the loop. `sys.stdin` is already an iterator of lines. Never call `.readlines()` or `.read()` on the whole file. Status class: `f"{status // 100}xx"`.

</details>

**Bonus:** Support `.gz` files directly with `gzip.open(path, "rt")` based on the suffix, and add a `--since "10 minutes ago"` filter using the parsed timestamp.

### Assignment 3. Reproducible environments with venv and a lock file 🟢

**Objective:** Make "works on my machine" impossible for a small Python tool.

**Tasks**
1. Create a project with `python3 -m venv .venv`, activate it, and install `requests` and `PyYAML`.
2. Try `pip install requests` with the venv deactivated on a Debian/Ubuntu host and record the `externally-managed-environment` error.
3. Write a `pyproject.toml` that declares the two dependencies with lower bounds.
4. Produce a fully pinned lock file with hashes: `uv pip compile pyproject.toml --generate-hashes -o requirements.lock` (or `pip-compile --generate-hashes`).
5. Delete `.venv`, recreate it, and install with `pip install --require-hashes -r requirements.lock`. Confirm identical versions with `pip freeze`.
6. Install `httpie` (or any CLI) with `pipx` and explain why it should not live in the project venv.

**Expected outcome / acceptance criteria**
- A README section with the exact commands to recreate the environment.
- `pip freeze` output is identical before and after recreation.
- You can explain the difference between `pip freeze > requirements.txt` and a compiled lock file.

<details>
<summary>Hints</summary>

`pip freeze` captures whatever happens to be installed, including leftovers, and records no hashes. A compiled lock resolves from declared dependencies only. `--require-hashes` fails the install if any package (including transitive ones) has no hash. `which python` inside the venv must point to `.venv/bin/python`.

</details>

**Bonus:** Repeat the whole exercise with `uv venv` and `uv sync` and compare wall-clock time with `time`.

### Assignment 4. YAML and JSON config converter with validation 🟢

**Objective:** Read, validate and write configuration files safely, the way deployment tooling has to.

**Tasks**
1. Write `cfgconv.py in.yaml out.json` and the reverse, choosing the direction from the file suffixes.
2. Use `yaml.safe_load` and `yaml.safe_dump(..., sort_keys=False)`. Show what `yaml.load` without a safe loader would allow with a `!!python/object/apply:os.system` payload (in a throwaway container) and explain why you never use it.
3. Validate that the config has `service` (string), `replicas` (int 1–50) and `env` (one of `dev`, `staging`, `prod`). Report every error, not just the first.
4. Write the output atomically: write to a temp file in the same directory, then `os.replace`.
5. Use `pathlib.Path` everywhere and always pass `encoding="utf-8"`.
6. Exit `0` on success, `1` on validation errors (printed to stderr), `2` on usage errors.

**Expected outcome / acceptance criteria**
- A YAML file with `replicas: "3"` (string) is rejected with a clear message.
- Interrupting the script mid-write never leaves a half-written output file.
- Round-tripping YAML → JSON → YAML preserves data and key order.

<details>
<summary>Hints</summary>

`tempfile.NamedTemporaryFile("w", dir=out.parent, delete=False, encoding="utf-8")`, write, `flush()`, `os.fsync()`, close, then `os.replace(tmp.name, out)`. `os.replace` is atomic only within one filesystem, which is why the temp file lives in the same directory. Remember YAML 1.1 surprises: `no`, `on`, and `off` become booleans with PyYAML.

</details>

**Bonus:** Replace the hand-written validation with a `pydantic` model and compare the error messages.

### Assignment 5. Disk usage alert with subprocess 🟢

**Objective:** Run external commands safely, parse their output, and turn the result into an exit code a cron job or monitoring agent can use.

**Tasks**
1. Run `df -P -k` with `subprocess.run([...], capture_output=True, text=True, check=True, timeout=10)`.
2. Parse each filesystem's mount point and use percentage. Ignore `tmpfs`, `devtmpfs` and `overlay`.
3. Accept `--warn 80 --crit 90` and print one line per filesystem over the warning threshold.
4. Exit `0` OK, `1` warning, `2` critical, `3` unknown (the Nagios plugin convention), and explain why that convention is useful.
5. Handle `FileNotFoundError` (command missing), `CalledProcessError` (non-zero exit, show its stderr) and `TimeoutExpired` (hung NFS mount) separately.
6. Compare your parsing with `shutil.disk_usage("/")` and explain when you would use each.

**Expected outcome / acceptance criteria**
- No `shell=True` anywhere.
- Setting `--warn 1` on any machine produces a warning and exit code 1.
- A hung command is killed after the timeout, not waited on forever.

<details>
<summary>Hints</summary>

`df -P` gives POSIX output with one line per filesystem, which is far easier to parse than plain `df`. The use column looks like `42%`; strip the `%`. `df -P -x tmpfs -x devtmpfs` filters on GNU coreutils. To test the timeout path, run `["sleep", "30"]` with `timeout=1`.

</details>

**Bonus:** Add `--json` output and a `--exclude-mount` option that accepts glob patterns matched with `fnmatch`.

### Assignment 6. Backup rotation CLI with argparse 🟢

**Objective:** Build a small, safe, destructive tool with a proper command-line interface.

**Tasks**
1. Write `rotate.py DIR --pattern "db-*.sql.gz" --keep 7` that deletes all but the newest N matching files.
2. Add `--dry-run` (default off) that prints what would be deleted and deletes nothing.
3. Add `--older-than-days N` as an alternative policy; make `--keep` and `--older-than-days` mutually exclusive with `add_mutually_exclusive_group`.
4. Refuse to run if `DIR` does not exist, is not a directory, or is `/`.
5. Sort by modification time with `Path.stat().st_mtime` and delete with `Path.unlink()`.
6. Print deleted files to stdout and errors to stderr. Exit `0` on success, `1` if any deletion failed, `2` on invalid arguments (argparse does this for you).

**Expected outcome / acceptance criteria**
- `--help` is clear and shows defaults.
- `--dry-run` and a real run on the same input list exactly the same files.
- A file that cannot be deleted (permissions) is reported and the rest are still processed.

<details>
<summary>Hints</summary>

`parser.add_argument("--keep", type=int)`, `parser.add_argument("--dry-run", action="store_true")`. Use `formatter_class=argparse.ArgumentDefaultsHelpFormatter`. `sorted(dir.glob(pattern), key=lambda p: p.stat().st_mtime, reverse=True)[keep:]` gives the files to delete. Create test files with old mtimes using `os.utime(path, (ts, ts))`.

</details>

**Bonus:** Add grandfather-father-son retention: keep 7 daily, 4 weekly and 12 monthly backups.

### Assignment 7. Replace print with logging 🟢

**Objective:** Make a script's output useful both for humans at a terminal and for a log pipeline in a container.

**Tasks**
1. Take your tool from Assignment 5 or 6 and replace every diagnostic `print` with a module-level `log = logging.getLogger(__name__)`.
2. Configure logging once, in `main()`, to stderr with timestamp, level and logger name.
3. Add `-v/--verbose` (DEBUG) and `-q/--quiet` (WARNING) flags.
4. Add `--log-format json` that emits one JSON object per line with `ts`, `level`, `logger`, `msg`, and `exc` when an exception is logged.
5. Use `log.exception(...)` inside an `except` block and show the traceback appears in both formats.
6. Use lazy formatting (`log.debug("found %d files", n)`) and explain why it beats f-strings in hot paths.

**Expected outcome / acceptance criteria**
- Results still go to stdout; only diagnostics go to stderr, so `tool | jq` keeps working.
- No `logging.basicConfig` or handler setup inside library modules.
- `--log-format json` output parses line by line with `jq`.

<details>
<summary>Hints</summary>

Subclass `logging.Formatter` and override `format(record)`; `self.formatException(record.exc_info)` renders the traceback. Third-party loggers such as `urllib3` or `botocore` can be quietened with `logging.getLogger("botocore").setLevel(logging.WARNING)`.

</details>

**Bonus:** Add a `RotatingFileHandler` that is enabled only with `--log-file`, and explain why containers should not use it.

### Assignment 8. Server inventory report from CSV 🟢

**Objective:** Turn a messy spreadsheet export into clean reports, a very common "first Python task" in ops teams.

**Tasks**
1. Create `servers.csv` with columns `hostname,ip,env,os,owner,cpu,ram_gb` and at least 30 rows, including a few with blank owners, duplicated hostnames, invalid IPs and inconsistent case in `env` (`Prod`, `PROD`, `prod`).
2. Read it with `csv.DictReader`, normalize `env` to lowercase, and validate IPs with the `ipaddress` module.
3. Write `invalid.csv` with the rejected rows plus a `reason` column.
4. Produce a summary per environment: host count, total CPU, total RAM, hosts with no owner.
5. Write the summary as CSV, as JSON, and as a Markdown table.
6. Generate an Ansible INI inventory grouped by environment from the valid rows.

**Expected outcome / acceptance criteria**
- Duplicates are detected and reported, not silently merged.
- The generated inventory works with `ansible-inventory -i hosts.ini --list`.
- No row is lost: valid count plus invalid count equals input count.

<details>
<summary>Hints</summary>

Open CSV files with `newline=""` as the `csv` docs require. `ipaddress.ip_address(value)` raises `ValueError` for invalid input. `collections.defaultdict(list)` makes grouping by environment one line.

</details>

**Bonus:** Accept the input from a Google Sheets "Publish to web as CSV" URL with `requests`, and cache it locally with an ETag.

### Assignment 9. HTTP health checker with timeouts and exit codes 🟢

**Objective:** Write the small script every team ends up with, but correctly: timeouts, clear failure reasons and a CI-friendly exit code.

**Tasks**
1. Read a list of URLs and expected status codes from a YAML file.
2. Use one `requests.Session` and call each URL with `timeout=(3.05, 10)`.
3. Record status code, latency in milliseconds (`response.elapsed`) and the failure reason: timeout, connection error, SSL error, or unexpected status.
4. Optionally check that the body contains a given string (`expect_text`).
5. Print a table and exit `0` if all checks pass, `1` if any fail.
6. Demonstrate each failure type: an unroutable IP (`10.255.255.1`) for a connect timeout, `https://expired.badssl.com/` for SSL, `https://httpbin.org/status/503` for a bad status.

**Expected outcome / acceptance criteria**
- No request can hang for longer than the configured timeouts.
- Each failure category shows a distinct, human-readable reason.
- The script never disables certificate verification.

<details>
<summary>Hints</summary>

Catch `requests.exceptions.ConnectTimeout`, `ReadTimeout`, `SSLError` and `ConnectionError` (order matters: `SSLError` and `ConnectTimeout` are subclasses of `ConnectionError`). Without a `timeout` argument, `requests` waits forever.

</details>

**Bonus:** Run it every minute from cron for an hour, append results to a CSV, and compute p50/p95 latency with `statistics.quantiles`.

### Assignment 10. Exceptions, cleanup and your own context manager 🟢

**Objective:** Write error handling that keeps context, cleans up reliably and never hides bugs.

**Tasks**
1. Find (or write) code with a bare `except:` and `except Exception: pass`. Explain what each swallows, including `KeyboardInterrupt` for the bare version.
2. Write `load_config(path)` that raises a custom `ConfigError` with `raise ConfigError(...) from exc` for missing files and YAML errors. Show the chained traceback.
3. Use `try / except / else / finally` in one function and explain when each block runs.
4. Write a `@contextmanager` called `lockfile(path)` that creates a lock file with `os.open(path, os.O_CREAT | os.O_EXCL)`, yields, and always removes it. A second process must fail immediately with a clear message.
5. Write `temporary_env(**vars)` that sets environment variables inside a `with` block and restores the previous values afterwards, even on error.
6. Use `contextlib.ExitStack` to open a variable number of files safely.

**Expected outcome / acceptance criteria**
- Killing the script with Ctrl+C inside the `with lockfile(...)` block still removes the lock.
- Running two copies concurrently: the second exits with code 1 and "already running".
- No bare `except:` remains.

<details>
<summary>Hints</summary>

`O_EXCL` makes creation fail with `FileExistsError` if the file exists, which is atomic. In a `@contextmanager`, put the `yield` inside `try/finally` or the cleanup will not run on exceptions. `KeyboardInterrupt` derives from `BaseException`, not `Exception`.

</details>

**Bonus:** Replace the lock file with `fcntl.flock` and explain why it handles a crashed process (stale lock) better than `O_EXCL`.

---

## 🟡 Intermediate

### Assignment 11. GitHub API client with retries, pagination and rate limits 🟡

**Objective:** Call a real REST API the way production code must: pooled connections, retries with backoff, full pagination and rate-limit awareness.

**Tasks**
1. Create a fine-grained GitHub token with read-only repository metadata access and read it from `GITHUB_TOKEN`; never hard-code it.
2. Build a `requests.Session` with an `HTTPAdapter` and `urllib3.util.Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504], allowed_methods=["GET"], respect_retry_after_header=True)`.
3. List every repository in an organization (pick a large public one), following the `Link: rel="next"` header via `response.links`.
4. For each repo, report default branch, last push date, open issue count, archived flag, and whether branch protection is enabled on the default branch (expect `404` without admin rights and handle it).
5. Read `X-RateLimit-Remaining` and `X-RateLimit-Reset`; when remaining drops below 50, sleep until reset and log why.
6. Output repos not pushed to in 365 days as CSV.

**Expected outcome / acceptance criteria**
- The script retrieves every page (the count matches the org's public repo count on the web UI).
- A forced `503` (test against `https://httpbin.org/status/503`) is retried with growing delays visible in DEBUG logs.
- The token never appears in logs or exceptions.

<details>
<summary>Hints</summary>

`session.headers.update({"Authorization": f"Bearer {token}", "Accept": "application/vnd.github+json", "X-GitHub-Api-Version": "2022-11-28"})`. Use `per_page=100`. `response.links.get("next", {}).get("url")` is `None` on the last page. Enable `logging.getLogger("urllib3").setLevel(logging.DEBUG)` to see retries.

</details>

**Bonus:** Switch to the GraphQL API to fetch the same data in far fewer requests and compare the rate-limit cost.

### Assignment 12. Multi-region EC2 inventory with boto3 🟡

**Objective:** Produce an accurate inventory of every EC2 instance in every enabled region, handling pagination, errors and credentials properly.

**Tasks**
1. Use a `boto3.session.Session` with a named profile (SSO is ideal) and verify identity with `sts.get_caller_identity()`.
2. Discover enabled regions with `ec2.describe_regions()`.
3. For each region, use the `describe_instances` paginator and collect ID, Name tag, type, state, private and public IP, launch time, AMI, VPC, and IMDS setting (`MetadataOptions.HttpTokens`).
4. Catch `botocore.exceptions.ClientError` per region, log `e.response["Error"]["Code"]`, and continue.
5. Configure `botocore.config.Config(retries={"mode": "standard", "max_attempts": 10})`.
6. Write the inventory to CSV and print a summary: count by region, by state, and the number of instances still allowing IMDSv1 (`HttpTokens == "optional"`).

**Expected outcome / acceptance criteria**
- Works with more than 1,000 instances (prove pagination with `PaginationConfig={"PageSize": 5}` on a small account).
- A region that returns `AuthFailure` or `UnauthorizedOperation` does not stop the run.
- No access keys in code or in the repository.

<details>
<summary>Hints</summary>

`paginator.paginate(PaginationConfig={"PageSize": 5})` forces many pages even on a tiny account. Instances are nested under `Reservations[].Instances[]`. Tags are a list of `{"Key", "Value"}` dicts, and some instances have no `Tags` key at all.

</details>

**Bonus:** Run the regions in parallel with `ThreadPoolExecutor`, creating one session per thread, and report the speed-up.

### Assignment 13. S3 bucket security audit 🟡

**Objective:** Audit every bucket in an account for the settings security teams ask about, and learn to read boto3 error codes that mean "not configured".

**Tasks**
1. List buckets with `s3.list_buckets()` (use the `list_buckets` paginator on current boto3) and find each bucket's region with `get_bucket_location` (`None` means `us-east-1`).
2. Using a client in the bucket's region, check Block Public Access (`get_public_access_block`), default encryption (`get_bucket_encryption`), versioning (`get_bucket_versioning`), and whether the bucket policy is public (`get_bucket_policy_status`).
3. Treat `NoSuchPublicAccessBlockConfiguration`, `ServerSideEncryptionConfigurationNotFoundError` and `NoSuchBucketPolicy` as "not configured", not as failures.
4. Also check account-level Block Public Access with the `s3control` client.
5. Assign a severity per bucket and write a Markdown report sorted by severity.
6. Test with three buckets you create: one compliant, one without versioning, one with Block Public Access disabled (do not attach a public policy).

**Expected outcome / acceptance criteria**
- Missing configurations are reported correctly, not as crashes.
- Buckets in other regions are checked with a client for that region.
- The report lists account-level and bucket-level findings separately.

<details>
<summary>Hints</summary>

`code = e.response["Error"]["Code"]` inside `except ClientError as e`. Since January 2023 new objects get SSE-S3 encryption by default, so "no encryption" is rare now; explain what that changes. Account ID for `s3control` comes from `sts.get_caller_identity()["Account"]`.

</details>

**Bonus:** Write moto tests for each "not configured" path so the error-code handling is proven offline.

### Assignment 14. Pod restart reporter with the Kubernetes client 🟡

**Objective:** Use the official `kubernetes` Python client to answer a question on-call engineers ask daily: what is crash-looping and why?

**Tasks**
1. Start a kind cluster and deploy three workloads: a healthy one, one that exits with code 1 every few seconds, and one with a memory limit low enough to be OOMKilled.
2. Load credentials with `config.load_kube_config()` locally and fall back to `config.load_incluster_config()` when `KUBERNETES_SERVICE_HOST` is set.
3. List pods in all namespaces with `CoreV1Api().list_pod_for_all_namespaces()` and use `_continue` with `limit=200` to paginate.
4. For each container report restart count, current state, last termination reason (`OOMKilled`, `Error`) and exit code.
5. Print pods with more than N restarts, sorted by restart count, and show the last 20 log lines of the previous container instance for the worst one (`read_namespaced_pod_log(..., previous=True, tail_lines=20)`).
6. Handle `kubernetes.client.exceptions.ApiException` with status 403 and 404 gracefully.

**Expected outcome / acceptance criteria**
- The OOMKilled pod shows reason `OOMKilled` and exit code 137.
- The tool works both from your laptop and as a Job inside the cluster with a ServiceAccount that has only `get`/`list` on pods and `get` on `pods/log`.
- Pagination is used even if your cluster is small.

<details>
<summary>Hints</summary>

The last crash lives in `pod.status.container_statuses[i].last_state.terminated`. Pass `_continue=resp.metadata._continue` until it is empty. For the in-cluster run you need a Role or ClusterRole plus binding; `kubectl auth can-i list pods --as=system:serviceaccount:ns:sa` verifies it.

</details>

**Bonus:** Post the report to a Slack incoming webhook when any pod crosses the threshold, and run it as a CronJob every 15 minutes.

### Assignment 15. TLS certificate expiry checker with a thread pool 🟡

**Objective:** Check hundreds of endpoints quickly with bounded concurrency, and understand why threads help for I/O-bound work.

**Tasks**
1. Build a list of 200 public HTTPS hostnames (a top-sites list works).
2. Write `cert_expiry(host, port=443, timeout=5)` with `ssl.create_default_context()`, `socket.create_connection` and `wrap_socket(server_hostname=host)`, and parse `notAfter` from `getpeercert()`.
3. Run it sequentially and time it.
4. Run it with `concurrent.futures.ThreadPoolExecutor(max_workers=32)` and `as_completed`, and time it again.
5. Report hosts expiring within 21 days, hosts that failed verification (`ssl.SSLCertVerificationError`), and hosts that timed out, as separate groups.
6. Exit `2` if any certificate expires within 7 days.

**Expected outcome / acceptance criteria**
- The threaded run is at least 10 times faster than the sequential run.
- One slow or dead host never blocks the others beyond its timeout.
- Verification failures are reported, never bypassed with `CERT_NONE`.

<details>
<summary>Hints</summary>

`ssl.cert_time_to_seconds(cert["notAfter"])` converts the date string. Use `with socket.create_connection((host, port), timeout=timeout) as sock, ctx.wrap_socket(sock, server_hostname=host) as tls:`. Store futures in a dict `{future: host}` so you know which host each result belongs to.

</details>

**Bonus:** Rewrite it with `asyncio.open_connection(host, 443, ssl=ctx)` and a semaphore, and compare code size and speed with the thread version.

### Assignment 16. A real pytest suite for your earlier tools 🟡

**Objective:** Learn the pytest features you will use every day by testing code you already wrote.

**Tasks**
1. Restructure the tools from Assignments 5 and 6 so logic lives in importable functions and `main()` only parses arguments.
2. Use `tmp_path` to create fake backup files with controlled mtimes and test rotation, including `--dry-run`.
3. Use `@pytest.mark.parametrize` to test the disk-usage parser with at least six `df -P` outputs, including an empty one and a malformed line.
4. Use `monkeypatch.setattr(subprocess, "run", fake_run)` (or patch where the name is looked up) to simulate `CalledProcessError` and `TimeoutExpired`.
5. Use `capsys` to assert what went to stdout and what went to stderr, and assert exit codes with `pytest.raises(SystemExit)`.
6. Add `pytest-cov` and reach 90% branch coverage (`--cov-branch`).

**Expected outcome / acceptance criteria**
- `pytest -q` passes with no network access and no root.
- Every exit code path has a test.
- Tests run in under 2 seconds.

<details>
<summary>Hints</summary>

If your module does `from subprocess import run`, patch `yourmodule.run`, not `subprocess.run`; you patch where the name is used. `exc_info.value.code` holds the exit code from `pytest.raises(SystemExit) as exc_info`. Shared setup belongs in `conftest.py` fixtures.

</details>

**Bonus:** Add `hypothesis` property tests for the parser (any generated `df` line either parses or is rejected cleanly, never crashes).

### Assignment 17. EC2 office-hours scheduler tested with moto 🟡

**Objective:** Build a cost-saving automation that changes infrastructure, and prove it safe with offline tests before it ever touches AWS.

**Tasks**
1. Write a function `plan(instances, now)` that returns which instances to stop and which to start, based on the tag `Schedule=office-hours` (on 08:00–20:00 Monday–Friday in the timezone given by tag `TZ`, default `Asia/Kolkata`).
2. Write `apply(ec2, plan, dry_run=True)` that calls `stop_instances` / `start_instances` in batches of 50 and passes `DryRun=True` to AWS when dry-run is on (catch the `DryRunOperation` error code as success).
3. Skip instances tagged `Schedule:override=true`, instances in an Auto Scaling group (`aws:autoscaling:groupName` tag), and anything not in `running`/`stopped`.
4. Use `zoneinfo.ZoneInfo` for time zones; inject `now` so tests control the clock.
5. Write moto tests (`@mock_aws` or a fixture) covering weekday, weekend, timezone edge, override tag and ASG members.
6. Package it as a Lambda handler and an EventBridge schedule rule (`cron(0 * ? * * *)`) in Terraform or SAM.

**Expected outcome / acceptance criteria**
- Tests pass with no AWS credentials.
- Dry-run never changes state; a test proves it by checking instance state afterwards.
- ASG members are never stopped.

<details>
<summary>Hints</summary>

Keep `plan()` pure (no AWS calls), so most tests need no moto at all. In moto, `describe_instances` reflects stops and starts immediately. AWS answers a successful dry run with `ClientError` code `DryRunOperation`, and a denied one with `UnauthorizedOperation`.

</details>

**Bonus:** Add a Slack message summarizing what was stopped and the estimated hourly saving based on instance type pricing you keep in a small dict.

### Assignment 18. Docker host janitor with the Docker SDK 🟡

**Objective:** Automate CI-runner disk cleanup with the Docker SDK instead of fragile shell pipelines.

**Tasks**
1. Connect with `docker.from_env()` and print the engine version and disk usage (`client.df()`).
2. List exited containers older than N hours (parse `container.attrs["State"]["FinishedAt"]`) and remove them, skipping any with the label `janitor.keep=true`.
3. Remove dangling images, then images not used by any container and older than N days, except those matching an allowlist of repositories.
4. Prune build cache and unused volumes behind separate flags (`--volumes` must be explicit; it can delete data).
5. Support `--dry-run` and print reclaimed space in human-readable units.
6. Handle `docker.errors.APIError` (image in use, conflict) per item and continue.

**Expected outcome / acceptance criteria**
- A container labelled `janitor.keep=true` is never removed.
- Dry-run and real run report the same candidate list.
- The tool works against a remote engine by setting `DOCKER_HOST=ssh://user@host`.

<details>
<summary>Hints</summary>

`client.containers.list(all=True, filters={"status": "exited"})`. `client.images.prune(filters={"dangling": True})` returns `SpaceReclaimed`. Docker timestamps have nanoseconds; trim to microseconds before `datetime.fromisoformat` on Python older than 3.11.

</details>

**Bonus:** Expose `janitor_reclaimed_bytes_total` with `prometheus_client` via the textfile collector so node_exporter picks it up.

### Assignment 19. FastAPI service with health, readiness and metrics 🟡

**Objective:** Build the small internal service pattern every platform team runs, with correct probes and Prometheus instrumentation.

**Tasks**
1. Create a FastAPI app with `GET /healthz` (process alive, no dependency checks) and `GET /readyz` (checks Redis with a 1-second timeout; returns 503 when it is unreachable).
2. Add `GET /api/deploys` that returns recent deploys stored in Redis.
3. Instrument with `prometheus_client`: a `Counter` of requests by method, route template and status, and a `Histogram` of request duration. Mount `make_asgi_app()` at `/metrics`.
4. Use route templates (`/api/deploys/{id}`) as label values, never raw paths, and explain why.
5. Run with `uvicorn app:app --host 0.0.0.0 --port 8000` in a non-root container, and write Kubernetes liveness, readiness and startup probes for it.
6. Stop Redis and show that the pod becomes NotReady but is not restarted.

**Expected outcome / acceptance criteria**
- `curl /metrics` shows your counter and histogram with low-cardinality labels.
- Killing Redis flips `/readyz` to 503 while `/healthz` stays 200.
- Graceful shutdown: in-flight requests complete on SIGTERM.

<details>
<summary>Hints</summary>

Use a middleware to time requests; read the matched route from `request.scope.get("route")` to get the template path. Liveness probes that check dependencies cause restart storms when a shared dependency fails. If you later run several worker processes, set `PROMETHEUS_MULTIPROC_DIR` and use multiprocess mode.

</details>

**Bonus:** Add structured JSON access logs with a request ID header (`X-Request-ID`) propagated to every log line.

### Assignment 20. Package, containerize and gate a tool in CI 🟡

**Objective:** Take any tool from earlier assignments from "a script" to "a versioned artifact with a quality gate".

**Tasks**
1. Move the code to a `src/` layout with a `pyproject.toml`, a `[project.scripts]` console entry point and a `dev` extra.
2. Build a wheel with `python -m build` and install it into a fresh venv to prove the entry point works.
3. Add `ruff` (lint and format) and `mypy --strict`, and fix every finding.
4. Write a multi-stage Dockerfile: build the wheel in one stage, copy only a venv into a `python:3.12-slim` runtime stage, run as a numeric non-root UID.
5. Add a GitHub Actions workflow that runs lint, type check and tests on Python 3.11–3.13, then builds the image and pushes it to GHCR on `main`, tagged with the commit SHA.
6. Scan the image with Trivy in the workflow and fail on CRITICAL vulnerabilities.

**Expected outcome / acceptance criteria**
- `pip install dist/*.whl && yourtool --help` works in a clean venv.
- `docker run --rm --entrypoint id image` shows a non-root UID.
- The workflow fails if a test, lint, type check or CRITICAL CVE fails.

<details>
<summary>Hints</summary>

Copy `pyproject.toml` before the source in the Dockerfile so dependency layers cache. A venv can be copied between stages when both use the same base image and path. `aquasecurity/trivy-action` with `severity: CRITICAL` and `exit-code: "1"` gates the build.

</details>

**Bonus:** Publish the wheel to TestPyPI from CI using trusted publishing (OIDC), with no API token stored.

---

## 🔴 Advanced

### Assignment 21. asyncio fan-out to thousands of endpoints 🔴

**Objective:** Write high-concurrency async code that is bounded, cancellable and honest about failures, and know when it beats threads.

**Tasks**
1. Start a local test server (FastAPI or `aiohttp.web`) with endpoints that respond fast, slowly (random 0–3 s), with 500s, and that never respond.
2. With `httpx.AsyncClient` (or `aiohttp.ClientSession`), call 5,000 URLs against it. Bound concurrency with `asyncio.Semaphore(200)` and set connection pool limits to match.
3. Give every request a timeout and wrap the whole run in `asyncio.timeout(60)` (3.11+). Collect successes, timeouts and errors separately.
4. Use `asyncio.TaskGroup` and explain what happens to sibling tasks when one raises; then switch to `gather(..., return_exceptions=True)` and compare.
5. Handle Ctrl+C: cancel outstanding tasks, close the client, and print partial results.
6. Implement the same with `ThreadPoolExecutor` and `requests` at 200 workers, and compare throughput, memory and file descriptors (`ls /proc/<pid>/fd | wc -l`).

**Expected outcome / acceptance criteria**
- At no point are more than 200 requests in flight (prove it with a counter in the test server).
- A hung endpoint never extends total runtime beyond the global timeout.
- No `RuntimeWarning: coroutine ... was never awaited` and no unclosed client warnings.
- A short write-up of when you would choose asyncio over threads.

<details>
<summary>Hints</summary>

Blocking calls (`time.sleep`, `requests.get`, boto3) inside a coroutine freeze the entire event loop; use `asyncio.to_thread` if you must call them. `httpx.Limits(max_connections=200, max_keepalive_connections=50)`. A `TaskGroup` cancels the remaining tasks when one fails and raises an `ExceptionGroup`, which you handle with `except*`.

</details>

**Bonus:** Add a token-bucket rate limiter (for example 500 requests per second) on top of the semaphore and show the difference between concurrency limits and rate limits.

### Assignment 22. A Kubernetes watch loop that survives 410 Gone 🔴

**Objective:** Build a minimal controller-style watcher that runs for days without missing events or leaking memory.

**Tasks**
1. Watch Deployments across all namespaces with `watch.Watch().stream(apps_v1.list_deployment_for_all_namespaces, resource_version=rv, timeout_seconds=300)`.
2. Start with a full list to obtain the initial `resourceVersion`, then watch from it. On each event, record the object's new `resourceVersion`.
3. When the stream ends normally, restart from the last `resourceVersion`. When you get `ApiException` with status 410 (`Expired: too old resource version`), re-list and start over.
4. Enforce a policy: any Deployment without the labels `team` and `cost-center` gets a Kubernetes Event (or an annotation `policy.example.com/violation`) and a log line. Make the action idempotent.
5. Handle SIGTERM: stop the watch, finish the current event, exit 0 within 10 seconds.
6. Run it in-cluster as a Deployment with a ServiceAccount and the minimum RBAC, and expose `/healthz` that fails if no event or bookmark was processed for 10 minutes.

**Expected outcome / acceptance criteria**
- Deleting and recreating 500 Deployments in a loop produces no missed or duplicated actions.
- Forcing a 410 (restart from a very old `resourceVersion`) recovers automatically and is logged.
- Memory is flat over a one-hour run.

<details>
<summary>Hints</summary>

Use `allow_watch_bookmarks=True` so the server sends `BOOKMARK` events that advance your `resourceVersion` even when nothing changes. A plain `while True` around the stream without tracking `resourceVersion` replays from "now" and misses events in between. Real controllers use an informer cache plus a work queue; `kopf` is the Python framework for that.

</details>

**Bonus:** Rewrite it with `kopf` as an operator and compare the amount of code and the failure handling you get for free.

### Assignment 23. Signed, idempotent GitHub webhook receiver 🔴

**Objective:** Build a webhook endpoint that is secure against forged requests, safe under retries, and observable under load.

**Tasks**
1. Write a Flask or FastAPI endpoint `POST /webhook/github` that verifies `X-Hub-Signature-256` using HMAC-SHA256 over the **raw** request body and `hmac.compare_digest`. Reject with 401 on mismatch.
2. Make processing idempotent with the `X-GitHub-Delivery` ID: store seen IDs in Redis with `SET key 1 NX EX 86400` and return 200 for duplicates without reprocessing.
3. Acknowledge quickly (GitHub times out after 10 seconds): enqueue the event (Redis list, RQ or Celery) and process it in a worker that, for `push` events to `main`, triggers a deployment by calling a downstream API with retries.
4. Run the web tier under gunicorn with 4 workers and expose Prometheus metrics in multiprocess mode (`PROMETHEUS_MULTIPROC_DIR`, `MultiProcessCollector`), counting deliveries by event type and result.
5. Write tests for valid signature, invalid signature, missing header, replayed delivery and unsupported event type.
6. Expose it with a tunnel (ngrok or cloudflared) and configure a real repository webhook; redeliver an event from the GitHub UI and show it is deduplicated.

**Expected outcome / acceptance criteria**
- A request with a body modified after signing is rejected.
- Redelivering the same event never triggers a second deployment.
- Metrics are correct across all gunicorn workers, not just the one that served `/metrics`.

<details>
<summary>Hints</summary>

Signature: `"sha256=" + hmac.new(secret, raw_body, hashlib.sha256).hexdigest()`. Re-serializing parsed JSON changes bytes and breaks the signature, so verify before parsing. In multiprocess mode, call `multiprocess.mark_process_dead(worker.pid)` in gunicorn's `child_exit` hook and clear the directory on start.

</details>

**Bonus:** Add a dead-letter queue for jobs that fail after N retries and an admin endpoint to replay them.

### Assignment 24. Organization-wide AWS audit with assumed roles 🔴

**Objective:** Scale a single-account boto3 audit to every account and region in an AWS Organization, safely and fast.

**Tasks**
1. From a management or delegated-admin account, list active accounts with the `organizations` `list_accounts` paginator.
2. For each account, call `sts.assume_role` into a read-only role (`OrganizationAccountAccessRole` in a lab, a dedicated `AuditReadOnly` role in production) with an `ExternalId` and a `RoleSessionName` that identifies the tool.
3. Build a `boto3.session.Session` from the temporary credentials, then run a check (for example the security group check from the lab) in every enabled region of that account.
4. Fan out with a bounded pool over (account, region) pairs. Make sure credentials are refreshed if a run takes longer than the session duration, or use `botocore` refreshable credentials.
5. Aggregate into one report with per-account summaries and a list of accounts where assume-role failed and why (`AccessDenied`, SCP-denied regions).
6. Run it on a schedule as an ECS Fargate task or Lambda (mind the 15-minute limit) with its own least-privilege task role.

**Expected outcome / acceptance criteria**
- One failing account never stops the scan; failures are listed in the report.
- No long-lived credentials anywhere; every session is temporary and named.
- CloudTrail in a member account shows `AssumeRole` with your session name.
- The run time for 20 accounts × 17 regions is under 5 minutes.

<details>
<summary>Hints</summary>

`creds = sts.assume_role(RoleArn=..., RoleSessionName="cloudaudit", ExternalId=...)["Credentials"]`, then `boto3.session.Session(aws_access_key_id=creds["AccessKeyId"], aws_secret_access_key=creds["SecretAccessKey"], aws_session_token=creds["SessionToken"])`. Role chaining caps sessions at one hour. A `~/.aws/config` profile with `role_arn` and `source_profile` gives automatic refresh for free when running locally.

</details>

**Bonus:** Write findings to a central S3 bucket as partitioned JSON (`account=/region=/date=`) and query trends with Athena.

### Assignment 25. Rolling restart orchestrator over SSH with Fabric 🔴

**Objective:** Build a safe rolling operation across a fleet: batches, health gates, abort thresholds and a clear audit trail.

**Tasks**
1. Bring up five Linux hosts (Vagrant, containers with sshd, or small EC2 instances) running a simple web service under systemd with a `/healthz` endpoint.
2. Using `fabric.Connection` with key-based auth, implement `rolling_restart(hosts, batch_size=2, max_failures=1)`: for each batch, drain (touch a maintenance file the load balancer health check reads), `sudo systemctl restart app`, wait for `/healthz` to return 200 with a timeout, then undrain.
3. Abort the rollout when failures exceed `max_failures`, leave failed hosts drained, and print exactly which hosts were touched.
4. Enforce host-key verification: load `known_hosts`, and fail on unknown hosts unless `--accept-new` is given (never `AutoAddPolicy` silently).
5. Set connect and command timeouts, and run commands with `warn=True` so you can inspect `result.exited` and `result.stderr` yourself.
6. Write an append-only JSON-lines audit log: who, when, host, action, result.

**Expected outcome / acceptance criteria**
- Breaking the service on one host (bad config) stops the rollout after that batch, with healthy hosts untouched.
- No more than `batch_size` hosts are ever out of service at once.
- `--dry-run` prints the full plan, batch by batch, without connecting.
- You can explain when you would use this instead of Ansible (`serial` and `max_fail_percentage`), and when you would not.

<details>
<summary>Hints</summary>

`Connection(host, user=..., connect_timeout=10, connect_kwargs={"key_filename": key})`. `conn.sudo("systemctl restart app", warn=True, hide=True)`. Paramiko's `RejectPolicy` is the safe default; `AutoAddPolicy` accepts any key, including an attacker's. Poll health with `requests.get(url, timeout=2)` in a loop with a deadline, not a fixed `sleep`.

</details>

**Bonus:** Run the batches in parallel with `fabric.ThreadingGroup` and add a canary phase: one host first, a 2-minute soak with an error-rate check, then the rest.

---

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
