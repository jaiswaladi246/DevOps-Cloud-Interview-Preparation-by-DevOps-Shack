# Python for DevOps — Cheatsheet

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

> Dense reference for daily automation work and interview prep. Python 3.11+ (3.12/3.13 in production images), boto3 1.35+, requests 2.32+, click 8.1+, pytest 8+, moto 5.x, ruff, mypy, uv.

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

## Contents

[Environments](#environments-venv-pip-uv) · [pyproject](#pyprojecttoml-skeleton) · [Language idioms](#language-idioms-for-automation) · [Exceptions](#exceptions-and-context-managers) · [subprocess](#subprocess) · [pathlib and files](#pathlib-os-and-shutil) · [JSON, YAML, CSV](#json-yaml-csv-and-env) · [requests](#requests-with-timeouts-and-retries) · [boto3](#boto3-patterns) · [boto3 errors](#boto3-errors-and-credentials) · [argparse](#argparse) · [click](#click) · [Exit codes](#exit-codes-and-streams) · [logging](#logging) · [Concurrency](#concurrency) · [Kubernetes and Docker SDKs](#kubernetes-and-docker-sdks) · [pytest](#pytest) · [moto](#moto) · [Quality and packaging](#quality-typing-and-packaging) · [Dockerfile](#dockerfile-for-a-python-cli) · [Shell one-liners](#python-one-liners-for-the-shell) · [Errors](#common-errors) · [Top 15](#top-15-interview-one-liners)

---

## Environments: venv, pip, uv

| Task | Command |
|------|---------|
| Create a venv | `python3 -m venv .venv` |
| Activate (bash/zsh) | `. .venv/bin/activate` |
| Activate (PowerShell) | `.venv\Scripts\Activate.ps1` |
| Which interpreter am I using | `which python; python -c "import sys; print(sys.prefix)"` |
| Leave the venv | `deactivate` |
| Upgrade pip inside the venv | `python -m pip install --upgrade pip` |
| Install a project in editable mode with extras | `pip install -e ".[dev]"` |
| Install from a pinned file | `pip install -r requirements.txt` |
| Install with hash checking | `pip install --require-hashes -r requirements.lock` |
| Snapshot what is installed | `pip freeze > requirements.txt` |
| Show a package and its dependencies | `pip show boto3` |
| Find dependency conflicts | `pip check` |
| Outdated packages | `pip list --outdated` |
| Dependency tree | `pipdeptree` (or `uv pip tree`) |
| Compile a lock with hashes (pip-tools) | `pip-compile --generate-hashes -o requirements.lock pyproject.toml` |
| Same with uv | `uv pip compile pyproject.toml --generate-hashes -o requirements.lock` |
| uv project workflow | `uv init`, `uv add boto3`, `uv add --dev pytest`, `uv sync`, `uv run pytest` |
| Run a tool once without installing | `uvx ruff check .` or `pipx run ruff check .` |
| Install a CLI tool globally, isolated | `pipx install awscli-local` |
| Use a specific Python | `uv venv --python 3.12` or `python3.12 -m venv .venv` |
| Always invoke pip via the interpreter | `python -m pip ...` (avoids "pip from another Python") |

Rules of thumb:

- One venv per project, never `sudo pip`. Debian 12+/Ubuntu 23.04+ refuse with `externally-managed-environment` (PEP 668).
- `pyproject.toml` declares ranges; a lock file pins exact versions and hashes. Libraries ship ranges, deployable tools ship a lock.
- `pip freeze` captures whatever is installed (including leftovers); a compiled lock resolves only declared dependencies.

## pyproject.toml skeleton

```toml
[build-system]
requires = ["setuptools>=69", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "opstool"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["boto3>=1.35", "click>=8.1", "requests>=2.32"]

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-cov", "moto[ec2,s3]>=5", "ruff", "mypy"]

[project.scripts]
opstool = "opstool.cli:main"        # creates an `opstool` executable on install

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"

[tool.ruff]
line-length = 100
[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "UP", "S", "SIM"]   # S = bandit security rules

[tool.mypy]
strict = true
```

## Language idioms for automation

| Idiom | Example |
|-------|---------|
| Filter and map | `ids = [i["InstanceId"] for i in insts if i["State"]["Name"] == "running"]` |
| Dict comprehension | `tags = {t["Key"]: t["Value"] for t in inst.get("Tags", [])}` |
| Set operations | `missing = required - set(tags)`; `common = a & b`; `either = a \| b` |
| Lazy generator | `errors = (l for l in open(p, encoding="utf-8") if "ERROR" in l)` |
| Safe dict access | `d.get("key", default)`; nested: `d.get("a", {}).get("b")` |
| Count things | `Counter(statuses).most_common(5)` |
| Group things | `by_env = defaultdict(list); by_env[h["env"]].append(h)` |
| Merge dicts (3.9+) | `cfg = defaults \| env_overrides` |
| Unpacking | `first, *rest = hosts`; `host, _, port = "db:5432".partition(":")` |
| Enumerate and zip | `for n, host in enumerate(hosts, start=1):`; `dict(zip(keys, values))` |
| Sort by key | `sorted(insts, key=lambda i: i["LaunchTime"], reverse=True)` |
| Chunk a list | `itertools.batched(ids, 50)` (3.12+), else `[ids[i:i+50] for i in range(0, len(ids), 50)]` |
| Any / all | `if any(r["CidrIp"] == "0.0.0.0/0" for r in ranges):` |
| Walrus | `while (line := f.readline()):` |
| f-string debug | `f"{instance_id=}"` → `instance_id='i-0abc'` |
| Keyword-only args | `def deploy(env, *, dry_run=False):` |
| Mutable default fix | `def f(tags=None): tags = tags or []` → better `if tags is None: tags = []` |
| Dataclass | `@dataclass(frozen=True) class Finding: id: str; severity: str` |
| Enum | `class Env(StrEnum): DEV = "dev"; PROD = "prod"` (3.11+) |
| match (3.10+) | `match event["type"]: case "push": ... case _: ...` |
| Pattern for wrapping paginated APIs | `def iter_items(): for page in pages: yield from page["Items"]` |

Decorator with `functools.wraps` (a simple retry):

```python
import functools, random, time

def retry(times=5, base=0.5, exceptions=(ConnectionError,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(base * 2 ** (attempt - 1) + random.uniform(0, base))  # backoff + jitter
        return wrapper
    return deco
```

## Exceptions and context managers

| Pattern | Code |
|---------|------|
| Catch narrowly, keep the cause | `except FileNotFoundError as exc: raise ConfigError(f"missing {p}") from exc` |
| Run only if no error | `try: ... except X: ... else: commit() finally: cleanup()` |
| Ignore one specific error | `with contextlib.suppress(FileNotFoundError): p.unlink()` (or `p.unlink(missing_ok=True)`) |
| Custom exception hierarchy | `class OpsError(Exception): ...`, `class ConfigError(OpsError): ...` |
| Group of errors (3.11+) | `except* TimeoutError as eg:` for `ExceptionGroup` from `TaskGroup` |
| Never | bare `except:` (catches `KeyboardInterrupt`, `SystemExit`) or `except Exception: pass` |

```python
from contextlib import contextmanager, ExitStack
import os

@contextmanager
def temporary_env(**changes):
    old = {k: os.environ.get(k) for k in changes}
    os.environ.update(changes)
    try:
        yield
    finally:                       # runs on success, exception, or Ctrl+C
        for k, v in old.items():
            if v is None:
                os.environ.pop(k, None)
            else:
                os.environ[k] = v

with ExitStack() as stack:          # variable number of resources
    files = [stack.enter_context(open(p, encoding="utf-8")) for p in paths]
```

## subprocess

| Task | Code |
|------|------|
| Run, fail on non-zero, capture text | `r = subprocess.run(["kubectl", "get", "ns"], check=True, capture_output=True, text=True, timeout=30)` |
| Output and errors | `r.stdout`, `r.stderr`, `r.returncode` |
| Handle failure | `except subprocess.CalledProcessError as e: log.error(e.stderr)` |
| Handle hang | `except subprocess.TimeoutExpired:` (child is killed) |
| Command not installed | `except FileNotFoundError:` |
| Check a tool exists first | `shutil.which("terraform") or sys.exit("terraform not found")` |
| Inherit output (stream to terminal) | `subprocess.run(cmd, check=True)` with no `capture_output` |
| Feed stdin | `subprocess.run(["kubectl", "apply", "-f", "-"], input=manifest, text=True, check=True)` |
| Extra env vars | `subprocess.run(cmd, env={**os.environ, "TF_IN_AUTOMATION": "1"})` |
| Working directory | `subprocess.run(cmd, cwd="infra/prod")` |
| Merge stderr into stdout | `stdout=subprocess.PIPE, stderr=subprocess.STDOUT` |
| Stream a long command line by line | see below |
| Build a command string safely (for logs only) | `shlex.join(cmd)` |
| Split a user string into args | `shlex.split("ls -la '/tmp/my dir'")` |
| Pipe without a shell | two `Popen` objects, see below |

```python
# Stream output live (e.g. terraform apply) and still get the exit code
with subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True) as p:
    for line in p.stdout:
        log.info(line.rstrip())
if p.returncode != 0:
    raise subprocess.CalledProcessError(p.returncode, cmd)

# Equivalent of: journalctl -u nginx | grep error   (no shell=True)
j = subprocess.Popen(["journalctl", "-u", "nginx", "--no-pager"], stdout=subprocess.PIPE)
g = subprocess.run(["grep", "-i", "error"], stdin=j.stdout, capture_output=True, text=True)
j.stdout.close(); j.wait()
```

Never: `subprocess.run(f"rm -rf {user_input}", shell=True)`. A list of arguments is never parsed by a shell, so `; rm -rf /` stays a literal filename. `os.system` and `os.popen` are the legacy versions of the same mistake.

## pathlib, os and shutil

| Task | Code |
|------|------|
| Build paths | `p = Path("/etc") / "app" / "config.yaml"` |
| Home, cwd, script dir | `Path.home()`, `Path.cwd()`, `Path(__file__).resolve().parent` |
| Parts | `p.name`, `p.stem`, `p.suffix`, `p.parent`, `p.parts` |
| Change suffix or name | `p.with_suffix(".json")`, `p.with_name("other.yaml")` |
| Exists / type | `p.exists()`, `p.is_file()`, `p.is_dir()`, `p.is_symlink()` |
| Read / write text | `p.read_text(encoding="utf-8")`, `p.write_text(s, encoding="utf-8")` |
| Read bytes | `p.read_bytes()` |
| Iterate lines lazily | `with p.open(encoding="utf-8") as f: for line in f:` |
| Make directories | `p.mkdir(parents=True, exist_ok=True)` |
| Glob | `Path("/var/log").glob("*.log")`, recursive: `.rglob("*.gz")` |
| Size and mtime | `st = p.stat(); st.st_size; st.st_mtime` |
| Delete | `p.unlink(missing_ok=True)`, empty dir `p.rmdir()`, tree `shutil.rmtree(p)` |
| Copy / move | `shutil.copy2(src, dst)` (keeps metadata), `shutil.copytree`, `shutil.move` |
| Permissions | `p.chmod(0o600)`; `os.access(p, os.W_OK)` |
| Disk usage | `shutil.disk_usage("/")` → `total, used, free` |
| Archive | `shutil.make_archive("backup", "gztar", root_dir="data")` |
| Env var with default | `os.environ.get("ENV", "dev")`; required: `os.environ["DB_URL"]` (KeyError if missing) |
| Temp dir that cleans itself | `with tempfile.TemporaryDirectory() as d:` |
| Relative path | `p.relative_to(base)` |
| Expand `~` | `Path("~/.kube/config").expanduser()` |

Atomic write (never leave a half-written config):

```python
import os, tempfile
from pathlib import Path

def atomic_write(path: Path, data: str) -> None:
    with tempfile.NamedTemporaryFile("w", dir=path.parent, delete=False, encoding="utf-8") as tmp:
        tmp.write(data)
        tmp.flush()
        os.fsync(tmp.fileno())
    os.replace(tmp.name, path)      # atomic on the same filesystem
```

## JSON, YAML, CSV and .env

| Task | Code |
|------|------|
| Parse JSON string / file | `json.loads(s)`; `json.load(f)` |
| Write JSON | `json.dumps(obj, indent=2, sort_keys=True, default=str)` (`default=str` handles datetimes) |
| Pretty-print from shell | `python -m json.tool file.json` |
| JSON lines (one object per line) | `[json.loads(l) for l in f if l.strip()]` |
| Parse YAML | `yaml.safe_load(f)` |
| Multi-document YAML (`---`) | `list(yaml.safe_load_all(f))` |
| Write YAML keeping key order | `yaml.safe_dump(obj, sort_keys=False, default_flow_style=False)` |
| Read CSV as dicts | `with open(p, newline="", encoding="utf-8") as f: rows = list(csv.DictReader(f))` |
| Write CSV | `w = csv.DictWriter(f, fieldnames=cols); w.writeheader(); w.writerows(rows)` |
| TOML (read-only, stdlib 3.11+) | `tomllib.load(open("pyproject.toml", "rb"))` |
| INI | `cfg = configparser.ConfigParser(); cfg.read("app.ini"); cfg["db"]["host"]` |
| .env file | `python-dotenv`: `load_dotenv()` then `os.environ[...]` |
| Base64 (K8s secrets) | `base64.b64decode(data["password"]).decode()` |

```python
# Edit a Kubernetes manifest: bump the image in every Deployment of a multi-doc file
docs = list(yaml.safe_load_all(Path("app.yaml").read_text(encoding="utf-8")))
for d in docs:
    if d and d.get("kind") == "Deployment":
        for c in d["spec"]["template"]["spec"]["containers"]:
            c["image"] = c["image"].rsplit(":", 1)[0] + ":v1.4.2"
Path("app.yaml").write_text(yaml.safe_dump_all(docs, sort_keys=False), encoding="utf-8")
```

YAML gotchas: `yaml.load` without `SafeLoader` can construct arbitrary Python objects (code execution). PyYAML follows YAML 1.1, so unquoted `no`, `off`, `yes`, `on` become booleans and `0755` becomes an octal int. Quote values when in doubt. Use `ruamel.yaml` when you must preserve comments.

## requests with timeouts and retries

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util import Retry

def make_session(token: str | None = None) -> requests.Session:
    retry = Retry(
        total=5,
        connect=3,
        backoff_factor=0.5,                         # 0.5, 1, 2, 4 ... seconds
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["GET", "HEAD", "PUT", "DELETE", "OPTIONS"],  # idempotent only
        respect_retry_after_header=True,
        raise_on_status=False,
    )
    s = requests.Session()
    s.mount("https://", HTTPAdapter(max_retries=retry, pool_maxsize=20))
    s.headers["User-Agent"] = "opstool/0.1"
    if token:
        s.headers["Authorization"] = f"Bearer {token}"
    return s

s = make_session(os.environ["GITHUB_TOKEN"])
r = s.get("https://api.github.com/orgs/ORG/repos", params={"per_page": 100}, timeout=(3.05, 15))
r.raise_for_status()
```

| Task | Code |
|------|------|
| Always set a timeout | `timeout=(connect, read)`; the default is to wait forever |
| Fail on 4xx/5xx | `r.raise_for_status()` → `requests.HTTPError` |
| JSON body / response | `s.post(url, json=payload, timeout=10)`; `r.json()` |
| Query params | `params={"state": "open", "per_page": 100}` |
| Follow Link-header pagination | `while url: r = s.get(url, timeout=10); ...; url = r.links.get("next", {}).get("url")` |
| Basic auth | `auth=("user", os.environ["PASS"])` |
| Custom CA bundle | `verify="/etc/ssl/certs/corp-ca.pem"` or `REQUESTS_CA_BUNDLE` env var |
| Client certificate (mTLS) | `cert=("client.crt", "client.key")` |
| Stream a large download | `with s.get(url, stream=True, timeout=30) as r: for chunk in r.iter_content(1 << 20): f.write(chunk)` |
| Proxy | `HTTPS_PROXY` env var, or `proxies={"https": "http://proxy:3128"}` |
| Rate limit headers (GitHub) | `r.headers["X-RateLimit-Remaining"]`, `X-RateLimit-Reset` |
| Exception order | `ConnectTimeout` and `SSLError` are subclasses of `ConnectionError`; catch them first |
| Never | `verify=False` in production; retrying a non-idempotent `POST` without an idempotency key |
| Async equivalent | `httpx.AsyncClient(timeout=10, limits=httpx.Limits(max_connections=100))` |
| Mock in tests | `responses` (requests) or `respx` (httpx) |

## boto3 patterns

```python
import boto3
from botocore.config import Config

cfg = Config(retries={"max_attempts": 10, "mode": "adaptive"}, connect_timeout=5, read_timeout=30)
session = boto3.session.Session(profile_name="prod-ro", region_name="ap-south-1")
ec2 = session.client("ec2", config=cfg)

# Paginate EVERY list/describe call
for page in ec2.get_paginator("describe_instances").paginate(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
):
    for res in page["Reservations"]:
        for inst in res["Instances"]:
            print(inst["InstanceId"])

# JMESPath over pages
ids = list(ec2.get_paginator("describe_instances").paginate().search("Reservations[].Instances[].InstanceId"))

# Waiter instead of a sleep loop
ec2.stop_instances(InstanceIds=ids)
ec2.get_waiter("instance_stopped").wait(InstanceIds=ids, WaiterConfig={"Delay": 15, "MaxAttempts": 40})
```

| Task | Code |
|------|------|
| Who am I | `session.client("sts").get_caller_identity()["Arn"]` (needs no IAM permission) |
| Enabled regions | `[r["RegionName"] for r in ec2.describe_regions()["Regions"]]` |
| All regions for a service | `session.get_available_regions("ec2")` (from the SDK's model, not the account) |
| Client vs resource | Clients: 1:1 with the API, thread-safe, complete. Resources: OO layer, feature-frozen, not thread-safe |
| Session per thread | `boto3.session.Session()` inside each worker; sessions are not thread-safe |
| Tags list → dict | `{t["Key"]: t["Value"] for t in obj.get("Tags", [])}` |
| Batch by API limit | e.g. `ec2.create_tags(Resources=batch, Tags=[...])` in chunks of ~1000 |
| Dry run | `ec2.stop_instances(InstanceIds=ids, DryRun=True)` → `ClientError` code `DryRunOperation` means "would succeed" |
| S3 upload/download (multipart, parallel) | `s3.upload_file("a.tgz", "bucket", "key")`, `s3.download_file(...)` |
| S3 list all keys | `s3.get_paginator("list_objects_v2").paginate(Bucket=b, Prefix="logs/")` |
| Presigned URL | `s3.generate_presigned_url("get_object", Params={"Bucket": b, "Key": k}, ExpiresIn=900)` |
| SSM parameter | `ssm.get_parameter(Name="/app/db_url", WithDecryption=True)["Parameter"]["Value"]` |
| Secrets Manager | `json.loads(sm.get_secret_value(SecretId="prod/db")["SecretString"])` |
| CloudWatch metric stats | `cw.get_metric_statistics(Namespace="AWS/EC2", MetricName="CPUUtilization", Dimensions=[...], StartTime=..., EndTime=..., Period=86400, Statistics=["Average"])` |
| Publish a custom metric | `cw.put_metric_data(Namespace="Ops/Audit", MetricData=[{"MetricName": "Findings", "Value": 3}])` |
| Assume a role | `c = sts.assume_role(RoleArn=arn, RoleSessionName="audit")["Credentials"]` then `boto3.session.Session(aws_access_key_id=c["AccessKeyId"], aws_secret_access_key=c["SecretAccessKey"], aws_session_token=c["SessionToken"])` |
| Local endpoint (moto server, LocalStack) | `AWS_ENDPOINT_URL=http://localhost:5000` (boto3 1.28+) or `endpoint_url=` |
| Debug the wire | `boto3.set_stream_logger("botocore", logging.DEBUG)` (logs signed requests, keep out of CI logs) |

## boto3 errors and credentials

```python
from botocore.exceptions import ClientError, NoCredentialsError, BotoCoreError

try:
    s3.head_object(Bucket=b, Key=k)
except ClientError as e:
    code = e.response["Error"]["Code"]          # "404", "AccessDenied", "ThrottlingException" ...
    status = e.response["ResponseMetadata"]["HTTPStatusCode"]
    if code in ("404", "NoSuchKey"):
        ...
    else:
        raise
except NoCredentialsError:
    sys.exit("no AWS credentials")

# Modelled errors also exist as attributes on the client
try:
    s3.get_object(Bucket=b, Key=k)
except s3.exceptions.NoSuchKey:
    ...
```

| Error code | Meaning |
|------------|---------|
| `AccessDenied` / `UnauthorizedOperation` / `AccessDeniedException` | IAM, SCP, permission boundary or resource policy denies it |
| `ThrottlingException` / `RequestLimitExceeded` / `Throttling` / `TooManyRequestsException` | Rate limited; adaptive retries, fewer workers |
| `ExpiredToken` / `ExpiredTokenException` | Temporary credentials expired; re-login or refresh |
| `InvalidClientTokenId` / `AuthFailure` | Wrong keys, or region not enabled for the account |
| `SignatureDoesNotMatch` | Wrong secret key, or system clock skew |
| `DryRunOperation` | Dry run would have succeeded |
| `NoSuchBucket`, `NoSuchKey`, `InvalidInstanceID.NotFound` | Resource missing (or eventual consistency right after creation) |

Credential provider chain, in order: explicit parameters → environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) → web identity (`AWS_WEB_IDENTITY_TOKEN_FILE`, used by IRSA) → shared credentials file → `~/.aws/config` profile (SSO, `role_arn`/`source_profile`, `credential_process`) → container credentials (ECS task role, EKS Pod Identity) → EC2 instance metadata (IMDSv2). In CI, use OIDC; on AWS compute, use the attached role; never commit keys.

## argparse

```python
import argparse, os, sys
from pathlib import Path

def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(prog="opstool", description="Ops helper",
                                formatter_class=argparse.ArgumentDefaultsHelpFormatter)
    p.add_argument("-v", "--verbose", action="count", default=0, help="-v INFO, -vv DEBUG")
    p.add_argument("--region", default=os.environ.get("AWS_REGION", "us-east-1"))
    sub = p.add_subparsers(dest="command", required=True)

    rot = sub.add_parser("rotate", help="Delete old backups")
    rot.add_argument("directory", type=Path)
    rot.add_argument("--keep", type=int, default=7)
    rot.add_argument("--dry-run", action="store_true")
    rot.add_argument("--env", choices=["dev", "staging", "prod"], required=True)
    rot.add_argument("--tag", action="append", default=[], help="repeatable")
    g = rot.add_mutually_exclusive_group()
    g.add_argument("--json", action="store_true")
    g.add_argument("--csv", action="store_true")
    rot.set_defaults(func=cmd_rotate)
    return p

def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)     # argv param makes it testable
    return args.func(args)

if __name__ == "__main__":
    sys.exit(main())
```

| Feature | How |
|---------|-----|
| Flag | `action="store_true"` |
| Repeatable option | `action="append"` |
| Count `-vvv` | `action="count"` |
| Fixed choices | `choices=[...]` |
| Positional with many values | `nargs="+"` (one or more), `nargs="*"` |
| Custom validation | `type=positive_int` where the function raises `argparse.ArgumentTypeError` |
| Usage error from your code | `parser.error("...")` (prints usage, exits 2) |
| Read a file or stdin | `type=argparse.FileType("r")` with `default=sys.stdin` |

## click

```python
import click
from pathlib import Path

@click.group(context_settings={"help_option_names": ["-h", "--help"]})
@click.version_option(package_name="opstool")
@click.option("--log-level", default="INFO", envvar="OPSTOOL_LOG_LEVEL",
              type=click.Choice(["DEBUG", "INFO", "WARNING"], case_sensitive=False))
@click.pass_context
def cli(ctx: click.Context, log_level: str) -> None:
    ctx.obj = {"log_level": log_level}

@cli.command()
@click.argument("names", nargs=-1, required=True)
@click.option("--region", envvar="AWS_REGION", default="us-east-1", show_default=True)
@click.option("--replicas", type=click.IntRange(1, 50), default=2)
@click.option("--dry-run/--no-dry-run", default=True, show_default=True)
@click.option("--out", type=click.Path(dir_okay=False, writable=True, path_type=Path))
@click.confirmation_option(prompt="This restarts production services. Continue?")
@click.pass_obj
def restart(obj: dict, names: tuple[str, ...], region: str, replicas: int,
            dry_run: bool, out: Path | None) -> None:
    """Restart NAMES in REGION."""
    click.echo(f"restarting {', '.join(names)}")        # stdout
    click.echo("warning: slow", err=True)               # stderr
    if dry_run:
        return                                          # exit 0
    if "prod-db" in names:
        raise click.UsageError("use the db runbook")    # exit 2

def main() -> None:
    cli(prog_name="opstool")
```

| Feature | How |
|---------|-----|
| Env var fallback | `envvar="NAME"`; or `auto_envvar_prefix="OPSTOOL"` on the group call |
| Boolean on/off pair | `--dry-run/--no-dry-run` |
| Secret prompt | `@click.option("--token", prompt=True, hide_input=True, envvar="TOKEN")` |
| Validated numbers | `click.IntRange(1, 50)`, `click.FloatRange(0, 100)` |
| Paths | `click.Path(exists=True, file_okay=False, path_type=Path)` |
| Errors | `click.BadParameter`, `click.UsageError` (exit 2), `click.ClickException` (exit 1) |
| Progress bar | `with click.progressbar(items) as bar: for i in bar: ...` |
| Test without a subprocess | `CliRunner().invoke(cli, ["restart", "web"])` → `.exit_code`, `.output`, `.stdout`, `.stderr` (separate stderr by default since click 8.2) |
| Typer | same ideas from type hints: `def restart(names: list[str], dry_run: bool = True)` |

## Exit codes and streams

| Code | Convention |
|------|-----------|
| `0` | Success |
| `1` | General error (also what an uncaught exception gives you) |
| `2` | Usage error (argparse, click); many audit tools also use it for "findings" |
| `3` | Nagios/monitoring "unknown" (0 OK, 1 WARNING, 2 CRITICAL, 3 UNKNOWN) |
| `130` | Terminated by Ctrl+C (128 + SIGINT 2) |
| `137` | Killed by SIGKILL (128 + 9), e.g. OOM killer |
| `143` | SIGTERM (128 + 15) |

- Results to **stdout**, diagnostics to **stderr**, so `tool | jq` works.
- `sys.exit("message")` prints to stderr and exits 1. `sys.exit(2)` exits 2 silently.
- Handle SIGTERM in long-running tools: `signal.signal(signal.SIGTERM, handler)` and finish the current unit of work.

## logging

```python
import logging, sys, json

log = logging.getLogger(__name__)          # in every module; no handlers in library code

def configure(level: str = "INFO", json_logs: bool = False) -> None:   # once, in main()
    handler = logging.StreamHandler(sys.stderr)
    if json_logs:
        class JsonFormatter(logging.Formatter):
            def format(self, r):
                d = {"ts": self.formatTime(r), "level": r.levelname, "logger": r.name, "msg": r.getMessage()}
                if r.exc_info:
                    d["exc"] = self.formatException(r.exc_info)
                return json.dumps(d)
        handler.setFormatter(JsonFormatter())
    else:
        handler.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s"))
    logging.basicConfig(level=level.upper(), handlers=[handler], force=True)
    logging.getLogger("botocore").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
```

| Task | Code |
|------|------|
| Lazy formatting | `log.info("deleted %d snapshots in %s", n, region)` (not f-strings in hot paths) |
| Log a traceback | `log.exception("deploy failed")` inside `except` |
| Add context to every line | `logging.LoggerAdapter(log, {"run_id": rid})` or a `Filter` |
| Level from env | `logging.basicConfig(level=os.environ.get("LOG_LEVEL", "INFO"))` |
| Library packages | `logging.getLogger(__name__).addHandler(logging.NullHandler())` |
| In containers | stdout/stderr only, ideally JSON; no file handlers, the platform ships logs |
| Never | log secrets, tokens, full request bodies or `boto3.set_stream_logger` output in shared CI logs |

## Concurrency

| Workload | Tool |
|----------|------|
| I/O-bound, sync libraries (boto3, requests, paramiko) | `ThreadPoolExecutor` |
| I/O-bound, thousands of calls, async libraries | `asyncio` + `httpx`/`aiohttp` + `Semaphore` |
| CPU-bound (hashing, compression, parsing) | `ProcessPoolExecutor` / `multiprocessing` |
| Call blocking code from async | `await asyncio.to_thread(fn, *args)` |

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=16) as pool:
    futures = {pool.submit(check_region, r): r for r in regions}
    for fut in as_completed(futures):
        region = futures[fut]
        try:
            results[region] = fut.result()
        except Exception as exc:          # one failure must not lose the others
            errors[region] = str(exc)
```

```python
import asyncio, httpx

async def fetch_all(urls: list[str], limit: int = 100) -> list[int | str]:
    sem = asyncio.Semaphore(limit)
    async with httpx.AsyncClient(timeout=10) as client:
        async def one(url: str) -> int | str:
            async with sem:
                try:
                    return (await client.get(url)).status_code
                except httpx.HTTPError as exc:
                    return type(exc).__name__
        async with asyncio.timeout(120):                       # 3.11+
            return await asyncio.gather(*(one(u) for u in urls))

results = asyncio.run(fetch_all(urls))
```

The GIL allows one thread to run Python bytecode at a time but is released during I/O, so threads speed up API fan-out and do nothing for pure CPU work. Free-threaded CPython (3.13t+) is optional and still not what production images use.

## Kubernetes and Docker SDKs

```python
from kubernetes import client, config, watch

try:
    config.load_incluster_config()        # inside a pod (ServiceAccount token)
except config.ConfigException:
    config.load_kube_config()             # ~/.kube/config, honours KUBECONFIG and current context

v1, apps = client.CoreV1Api(), client.AppsV1Api()
for pod in v1.list_namespaced_pod("prod", label_selector="app=web").items:
    restarts = sum(cs.restart_count for cs in pod.status.container_statuses or [])
    print(pod.metadata.name, pod.status.phase, restarts)

apps.patch_namespaced_deployment_scale("web", "prod", {"spec": {"replicas": 5}})

w = watch.Watch()
for ev in w.stream(v1.list_namespaced_event, "prod", timeout_seconds=300):
    print(ev["type"], ev["object"].reason, ev["object"].message)
```

| Task | Code |
|------|------|
| Paginate | `limit=200, _continue=resp.metadata._continue` |
| Handle API errors | `except client.exceptions.ApiException as e: if e.status == 404: ...` |
| Watch restart rule | track `resourceVersion`; on `410 Gone`, re-list and watch again |
| Rollout restart | patch `spec.template.metadata.annotations["kubectl.kubernetes.io/restartedAt"]` with a timestamp |

```python
import docker

d = docker.from_env()                         # honours DOCKER_HOST, e.g. ssh://user@host
d.containers.run("alpine:3.20", ["echo", "hi"], remove=True)
for c in d.containers.list(all=True, filters={"status": "exited"}):
    c.remove()
print(d.images.prune(filters={"dangling": True})["SpaceReclaimed"])
```

SSH with Fabric (paramiko underneath):

```python
from fabric import Connection
with Connection("web1", user="deploy", connect_timeout=10) as c:
    r = c.run("systemctl is-active nginx", warn=True, hide=True)
    if r.exited != 0:
        c.sudo("systemctl restart nginx")
    c.put("app.conf", "/tmp/app.conf")
```

## pytest

| Task | Code / command |
|------|----------------|
| Run quietly | `pytest -q` |
| One file / test / keyword | `pytest tests/test_ec2.py::test_idle -q`; `pytest -k "sg and not ipv6"` |
| Stop at first failure, show locals | `pytest -x -l` |
| Rerun last failures | `pytest --lf` |
| Print output (no capture) | `pytest -s` |
| Show slowest tests | `pytest --durations=10` |
| Coverage with a gate | `pytest --cov=opstool --cov-branch --cov-report=term-missing --cov-fail-under=85` |
| Parallel | `pytest -n auto` (pytest-xdist) |
| Markers | `@pytest.mark.slow`, run with `-m "not slow"`; register them in config with `--strict-markers` |
| Expected exception | `with pytest.raises(ConfigError, match="missing"):` |
| Exit code of a CLI | `with pytest.raises(SystemExit) as e: main(["--bad"])` → `e.value.code == 2` |
| Approximate floats | `assert x == pytest.approx(0.3)` |

```python
import pytest

@pytest.fixture
def config_file(tmp_path):                 # tmp_path: fresh directory per test
    p = tmp_path / "cfg.yaml"
    p.write_text("service: web\nreplicas: 3\n", encoding="utf-8")
    return p

@pytest.mark.parametrize(("line", "expected"), [
    ("/dev/sda1 100 42 58 42% /", ("/", 42)),
    ("tmpfs 10 0 10 0% /run", None),
])
def test_parse(line, expected):
    assert parse_df_line(line) == expected

def test_env_and_patching(monkeypatch, capsys):
    monkeypatch.setenv("ENV", "prod")
    monkeypatch.setattr("opstool.cli.subprocess.run", lambda *a, **k: FakeResult(0, "ok"))
    main(["status"])
    out, err = capsys.readouterr()
    assert "ok" in out and err == ""
```

Built-in fixtures to know: `tmp_path`, `monkeypatch`, `capsys`, `caplog` (assert log records), `request`. Patch where a name is **looked up**: if `cli.py` does `from subprocess import run`, patch `opstool.cli.run`.

## moto

```python
import boto3, pytest
from moto import mock_aws                          # moto 5: one decorator for every service

@pytest.fixture(autouse=True)
def aws_credentials(monkeypatch):                  # tests can never hit a real account
    for k in ("AWS_ACCESS_KEY_ID", "AWS_SECRET_ACCESS_KEY", "AWS_SESSION_TOKEN"):
        monkeypatch.setenv(k, "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")
    monkeypatch.delenv("AWS_PROFILE", raising=False)

@pytest.fixture
def s3():
    with mock_aws():
        yield boto3.client("s3", region_name="us-east-1")

def test_bucket_without_versioning_is_flagged(s3):
    s3.create_bucket(Bucket="logs")
    assert "logs" in audit_versioning(s3)
```

| Tip | Detail |
|-----|--------|
| Create clients inside the mock | A client created before `mock_aws()` starts talks to real AWS |
| Clean EC2 account | `MOTO_EC2_LOAD_DEFAULT_AMIS=false` skips ~1,300 preloaded AMIs and snapshots |
| Inject time | Pass `now=` into your code; moto timestamps are "now" and cannot be backdated |
| Server mode | `moto_server -p 5000` + `AWS_ENDPOINT_URL=http://localhost:5000` to test a whole CLI or another language |
| Old syntax | `mock_ec2`, `mock_s3` were removed in moto 5; use `mock_aws` |
| Alternatives | `botocore.stub.Stubber` for exact request/response assertions; LocalStack for integration tests |

## Quality, typing and packaging

| Task | Command |
|------|---------|
| Lint | `ruff check .` (autofix: `ruff check --fix .`) |
| Format | `ruff format .`; CI: `ruff format --check .` |
| Type check | `mypy src` (strict via config); boto3 types: `pip install "boto3-stubs[ec2,s3]"` |
| Security lint | `ruff check --select S .` or `bandit -r src` |
| Dependency CVEs | `pip-audit -r requirements.lock` |
| Pre-commit hooks | `pre-commit install` with ruff and mypy hooks |
| Build sdist and wheel | `python -m build` (or `uv build`) → `dist/` |
| Inspect a wheel | `unzip -l dist/*.whl` |
| Install a built wheel | `pip install dist/opstool-0.1.0-py3-none-any.whl` |
| Run as a module | `python -m opstool` (needs `__main__.py`) |
| Version from code | `importlib.metadata.version("opstool")` |
| Publish | `twine upload dist/*`, or trusted publishing (OIDC) from GitHub Actions |
| Single-file script with dependencies | PEP 723 header `# /// script` + `dependencies = [...]`, run with `uv run script.py` |

Type hint cheats: `list[str]`, `dict[str, int]`, `str | None`, `Callable[[str], bool]`, `Iterator[dict[str, Any]]`, `TypedDict` for JSON shapes, `Protocol` for duck typing, `from __future__ import annotations` for forward references.

## Dockerfile for a Python CLI

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS build
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
WORKDIR /src
RUN pip install build
COPY pyproject.toml ./
COPY src ./src
RUN python -m build --wheel --outdir /dist \
 && python -m venv /opt/venv \
 && /opt/venv/bin/pip install /dist/*.whl

FROM python:3.12-slim
ENV PATH=/opt/venv/bin:$PATH PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app
COPY --from=build /opt/venv /opt/venv
USER 10001:10001
ENTRYPOINT ["opstool"]
CMD ["--help"]
```

- `PYTHONUNBUFFERED=1` so logs appear immediately in `docker logs` / `kubectl logs`.
- Numeric `USER` so Kubernetes `runAsNonRoot` can verify it.
- `-slim` over `-alpine` for Python: Alpine uses musl, so many wheels must compile from source.
- Distroless (`gcr.io/distroless/python3`) is smaller still, but has no shell for debugging and must match the venv's Python version.

## Python one-liners for the shell

```bash
# Pretty-print / validate JSON
python3 -m json.tool response.json
curl -s https://api.github.com/repos/python/cpython | python3 -c 'import sys, json; print(json.load(sys.stdin)["stargazers_count"])'

# YAML to JSON (needs PyYAML), then use jq
python3 -c 'import sys, yaml, json; print(json.dumps(yaml.safe_load(sys.stdin), indent=2))' < values.yaml | jq '.image'

# Validate every YAML file in a repo
python3 -c 'import sys, yaml; [list(yaml.safe_load_all(open(f))) for f in sys.argv[1:]]' $(git ls-files '*.yaml' '*.yml')

# Serve the current directory on port 8000 (debugging only)
python3 -m http.server 8000 --bind 127.0.0.1

# Base64 decode a Kubernetes secret value
kubectl get secret db -o jsonpath='{.data.password}' | python3 -c 'import sys, base64; print(base64.b64decode(sys.stdin.read()).decode())'

# Random password / token
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'

# Epoch to UTC and back
python3 -c 'import datetime as d; print(d.datetime.fromtimestamp(1767225600, d.UTC))'
python3 -c 'import time; print(int(time.time()))'

# Top 10 client IPs from an access log
python3 -c 'import sys, collections; c = collections.Counter(l.split()[0] for l in sys.stdin if l.strip()); [print(n, ip) for ip, n in c.most_common(10)]' < access.log

# Is a port open?
python3 -c 'import socket, sys; s = socket.create_connection((sys.argv[1], int(sys.argv[2])), timeout=3); print("open")' db.internal 5432

# TLS certificate expiry
python3 -c 'import ssl, socket; h = "example.com"; s = ssl.create_default_context().wrap_socket(socket.create_connection((h, 443), timeout=5), server_hostname=h); print(s.getpeercert()["notAfter"])'

# SHA-256 of a file
python3 -c 'import hashlib, sys; print(hashlib.file_digest(open(sys.argv[1], "rb"), "sha256").hexdigest())' artifact.tgz

# URL-encode a string
python3 -c 'import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=""))' 'a b&c=d'

# Which Python, venv and site-packages am I using
python3 -c 'import sys, site; print(sys.executable, sys.prefix, site.getsitepackages())'

# Time a snippet
python3 -m timeit -s 'xs = list(range(10_000))' 'sum(x * x for x in xs)'

# Profile a script
python3 -m cProfile -s cumtime script.py | head -25
```

## Common errors

| Error | Usual cause → fix |
|-------|-------------------|
| `error: externally-managed-environment` | pip into the OS Python → use a venv or `pipx` |
| `ModuleNotFoundError: No module named 'yaml'` | Package is `PyYAML`, wrong venv active → `pip install PyYAML`, check `which python` |
| `ImportError: cannot import name 'X' from partially initialized module` | Circular import, or your file is named like a library (`yaml.py`, `boto3.py`) → rename, move the import |
| `TypeError: unhashable type: 'list'` | list/dict used as a dict key or set member → use a tuple |
| `UnicodeDecodeError: 'utf-8' codec can't decode byte` | Binary or non-UTF-8 file → `errors="replace"` or the right encoding |
| `json.decoder.JSONDecodeError: Expecting value: line 1 column 1` | Empty body or HTML error page → check `r.status_code` and `r.text` first |
| `yaml.constructor.ConstructorError: could not determine a constructor for the tag` | `safe_load` refusing a custom tag (often `!Ref` in CloudFormation) → use a custom loader or `cfn-flip` |
| `requests.exceptions.SSLError: ... CERTIFICATE_VERIFY_FAILED` | Corporate proxy or private CA → `REQUESTS_CA_BUNDLE=/path/ca.pem`, never `verify=False` |
| `requests.exceptions.ReadTimeout` | Slow upstream → raise read timeout, retry idempotent calls with backoff |
| `botocore.exceptions.NoCredentialsError: Unable to locate credentials` | Nothing in the provider chain → `aws sso login`, set `AWS_PROFILE`, check the role |
| `botocore.exceptions.NoRegionError: You must specify a region.` | No region anywhere → `region_name=` or `AWS_DEFAULT_REGION` |
| `An error occurred (ExpiredToken)` | Session expired mid-run → refreshable credentials, shorter runs, re-login |
| `An error occurred (ThrottlingException) ... Rate exceeded` | Too many calls → adaptive retries, fewer workers, paginators |
| `RuntimeError: This event loop is already running` | `asyncio.run` inside Jupyter or another loop → `await` directly in that context |
| `RuntimeWarning: coroutine 'x' was never awaited` | Called an async function without `await` |
| `subprocess.CalledProcessError: Command '[...]' returned non-zero exit status 1` | Command failed → log `e.stderr` |
| `PermissionError: [Errno 13]` in a container | Non-root UID cannot write a bind mount → `--user $(id -u):$(id -g)` or fix ownership |

---

## Top 15 interview one-liners

1. **Why Python over Bash:** real data structures, error handling, tests and SDKs; Bash for glue under ~50 lines, Python once there is parsing, APIs or logic.
2. **Mutable default argument:** defaults are evaluated once at `def` time, so `def f(x=[])` shares one list; use `None` and create it inside.
3. **Generator vs list:** a generator yields lazily in constant memory; use it for big files and paginated APIs.
4. **`subprocess.run` safely:** pass a list, `check=True`, `capture_output=True`, `text=True`, `timeout=`; never `shell=True` with untrusted input.
5. **`yaml.safe_load`:** plain `yaml.load` can build arbitrary Python objects, which means code execution from a config file.
6. **requests defaults:** no timeout means wait forever; always pass `timeout=(connect, read)`, use a `Session`, and retry only idempotent calls with backoff on 429/5xx.
7. **boto3 client vs resource:** clients map 1:1 to the API and are thread-safe; resources are a feature-frozen OO layer. Sessions are not thread-safe, so one per thread.
8. **Pagination:** every list API is paginated; a script without a paginator silently audits only page one.
9. **boto3 errors:** catch `ClientError` and branch on `e.response["Error"]["Code"]`; use adaptive retry mode for throttling.
10. **Credentials:** never in code; the provider chain finds env vars, SSO profiles, IRSA/Pod Identity, ECS task roles or IMDSv2; CI uses OIDC.
11. **GIL:** one thread runs bytecode at a time, but I/O releases it; threads for API fan-out, processes for CPU, asyncio for thousands of concurrent connections.
12. **Exit codes and streams:** 0 success, non-zero failure, results on stdout, logs on stderr; that is what makes a script usable in CI and pipes.
13. **Logging over print:** levels, timestamps and logger names, configured once in `main()`, JSON to stdout/stderr in containers.
14. **Testing AWS code:** pytest with moto's `mock_aws` exercises real botocore request validation in memory; inject the clock, fake credentials in an autouse fixture.
15. **Reproducible environments:** a venv per project, ranges in `pyproject.toml`, exact pins with hashes in a lock, and a multi-stage non-root image for shipping.

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
