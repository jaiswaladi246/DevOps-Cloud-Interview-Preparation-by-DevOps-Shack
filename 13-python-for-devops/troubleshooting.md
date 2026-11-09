# Python for DevOps — Troubleshooting Drills

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

> Real Python for DevOps errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Environments, imports & packaging errors](#environments-imports--packaging-errors) — Q1–Q25
- [Runtime, file, subprocess & encoding errors](#runtime-file-subprocess--encoding-errors) — Q26–Q50
- [boto3, HTTP & API errors](#boto3-http--api-errors) — Q51–Q75
- [Concurrency, async, tests, services & containers](#concurrency-async-tests-services--containers) — Q76–Q100

---

## Environments, imports & packaging errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `ModuleNotFoundError: No module named 'boto3'` right after you ran `pip install boto3`. 🟢

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

### Q2. `error: externally-managed-environment` when running `pip install requests` on Ubuntu 24.04. 🟢

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

### Q3. `ImportError: attempted relative import with no known parent package` when running `python cloudaudit/cli.py`. 🟡

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

### Q4. `AttributeError: partially initialized module 'yaml' has no attribute 'safe_load' (most likely due to a circular import)` 🟢

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

### Q5. `ERROR: Could not find a version that satisfies the requirement boto3==1.40.99 (from versions: ...)` 🟢

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

### Q6. `ERROR: ResolutionImpossible: ... The conflict is caused by: botocore 1.35.x depends on urllib3<1.27 ... requests 2.x depends on urllib3>=2` 🔴

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

### Q7. `pip install` on `python:3.12-alpine` takes 15 minutes and ends with `error: command 'gcc' failed: No such file or directory`. 🟡

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

### Q8. `zsh: no matches found: .[dev]` when running `pip install -e .[dev]`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** zsh treats `[dev]` as a glob pattern.

**Diagnose →** The error comes from the shell, before pip runs.

**Fix →** Quote it: `pip install -e ".[dev]"`.

**Prevent →** Always quote extras in docs and scripts.

</details>

### Q9. `bash: .venv/bin/activate: No such file or directory` in a CI job, although the venv was created in an earlier step. 🟡

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

### Q10. `SyntaxError: invalid syntax` on a line with `match command:` or `X | None` on a server. 🟢

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

### Q11. `/usr/bin/env: 'python': No such file or directory` when executing a script directly. 🟢

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

### Q12. `/bin/bash^M: bad interpreter: No such file or directory` or `env: 'python3\r': No such file or directory`. 🟢

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

### Q13. `pip` warns `WARNING: Running pip as the 'root' user can result in broken permissions` during a Docker build. Should you care? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Installing as root into the global site-packages in a container.

**Diagnose →** Look at the Dockerfile: `RUN pip install ...` before any `USER`.

**Fix →** It is mostly harmless in a throwaway container layer, but the clean approach is a venv in a builder stage (`python -m venv /opt/venv`), copied into the runtime stage, and a non-root `USER` at runtime. Or set `PIP_ROOT_USER_ACTION=ignore` if you accept the trade-off.

**Prevent →** Multi-stage builds with a venv or installed wheel and non-root runtime users.

</details>

### Q14. `ERROR: THESE PACKAGES DO NOT MATCH THE HASHES FROM THE REQUIREMENTS FILE.` 🔴

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

### Q15. Your package installs fine, but at runtime `FileNotFoundError: .../site-packages/cloudaudit/templates/report.html`. 🟡

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

### Q16. `cloudaudit: command not found` after `pip install --user .`. 🟢

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

### Q17. `ImportError: cannot import name 'mock_ec2' from 'moto'` after a dependency update. 🟡

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

### Q18. `pkg_resources is deprecated as an API` warnings, or `ModuleNotFoundError: No module named 'pkg_resources'` in a fresh venv. 🟡

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

### Q19. `ModuleNotFoundError: No module named 'distutils'` on Python 3.12. 🟡

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

### Q20. In VS Code tests pass, but in the terminal `pytest` shows `ModuleNotFoundError: No module named 'cloudaudit'`. 🟢

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

### Q21. `ImportError: libssl.so.1.1: cannot open shared object file` after copying a venv from one machine to another. 🔴

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

### Q22. `pip` hangs or fails with `Could not fetch URL https://pypi.org/simple/...: There was a problem confirming the ssl certificate` behind the corporate proxy. 🟡

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

### Q23. Your Docker build reinstalls all Python dependencies on every code change, taking 4 minutes. 🟢

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

### Q24. `mypy` reports `error: Library stubs not installed for "requests"` and `Skipping analyzing "boto3": module is installed, but missing library stubs or py.typed marker`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The libraries do not ship type information; mypy needs separate stub packages.

**Diagnose →** Read the hint mypy prints (`python3 -m pip install types-requests`).

**Fix →** `pip install types-requests types-PyYAML "boto3-stubs[ec2,s3]"`, or `mypy --install-types`. For libraries without stubs, set `ignore_missing_imports = true` for that module in an override.

**Prevent →** Keep stub packages in dev dependencies with the same update group as the library.

</details>

### Q25. `ruff` passes locally but fails in CI with different rules or versions. 🟢

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

## Runtime, file, subprocess & encoding errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `KeyError: 'Tags'` while processing `describe_instances` output. 🟢

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

### Q27. `TypeError: can't compare offset-naive and offset-aware datetimes` in a snapshot-age check. 🟢

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

### Q28. `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte` reading a log file. 🟡

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

### Q29. `UnicodeEncodeError: 'ascii' codec can't encode character '✓'` when printing in a container. 🟡

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

### Q30. `PermissionError: [Errno 13] Permission denied: '/var/log/app/report.json'` 🟡

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

### Q31. `OSError: [Errno 24] Too many open files` in a long-running Python service. 🟡

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

### Q32. `subprocess.CalledProcessError: Command '['terraform', 'plan']' returned non-zero exit status 1.` with no further detail. 🟢

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

### Q33. `FileNotFoundError: [Errno 2] No such file or directory: 'kubectl'` from `subprocess.run`, though `kubectl` works in your terminal. 🟡

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

### Q34. `subprocess.TimeoutExpired: Command '['ssh', 'web1', 'uptime']' timed out after 30 seconds`, and the script hangs anyway. 🔴

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

### Q35. `json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)` calling `resp.json()`. 🟢

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

### Q36. `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref'` loading a CloudFormation template. 🟡

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

### Q37. `yaml.composer.ComposerError: expected a single document in the stream` 🟢

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

### Q38. A YAML value `country: NO` is loaded as `False`, and version `1.10` becomes `1.1`. 🟡

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

### Q39. `csv.Error: field larger than field limit (131072)` 🟡

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

### Q40. `RecursionError: maximum recursion depth exceeded` in a function that walks nested JSON. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Very deep structures, or a cycle (an object referencing itself), or a missing base case.

**Diagnose →** Read the repeating frames in the traceback; check for self-referencing structures.

**Fix →** Fix the base case; for deep data rewrite as an iterative walk with an explicit stack; track visited `id()`s to break cycles. Raising `sys.setrecursionlimit` is a last resort and can segfault.

**Prevent →** Prefer iterative traversal for untrusted, arbitrarily deep input.

</details>

### Q41. `MemoryError` or the container gets `OOMKilled` while processing a big API export. 🟡

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

### Q42. `NameError: name 'log' is not defined` only when an error path runs in production. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo or missing import in a rarely executed branch (often an `except` block), which Python only detects when that line runs.

**Diagnose →** Run `ruff check` (`F821 Undefined name`) and `mypy`; both find this statically.

**Fix →** Define/import the name; add a test that exercises the error path.

**Prevent →** Linting in CI catches this class of bug in seconds; test failure paths, not just the happy path.

</details>

### Q43. `TypeError: 'NoneType' object is not subscriptable` in `pod.status.container_statuses[0]`. 🟢

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

### Q44. A cron job fails with `KeyError: 'HOME'` or cannot find `~/.aws/config`. 🟡

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

### Q45. `OSError: [Errno 18] Invalid cross-device link` when calling `os.rename`. 🔴

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

### Q46. Logs appear twice for every message after you added a handler. 🟡

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

### Q47. `logging.basicConfig(level=logging.DEBUG)` has no effect; you still see only warnings. 🟡

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

### Q48. Your script ends with `BrokenPipeError: [Errno 32] Broken pipe` when piped to `head`. 🔴

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

### Q49. `ValueError: too many values to unpack (expected 2)` parsing `KEY=VALUE` lines from an env file. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value contains `=` (base64 strings, URLs with query strings), so `line.split("=")` returns more than two parts.

**Diagnose →** Print the failing line.

**Fix →** `key, _, value = line.partition("=")` or `line.split("=", 1)`; skip blank lines and `#` comments; strip surrounding quotes.

**Prevent →** Use `python-dotenv` for env files rather than hand-written parsers.

</details>

### Q50. `UnboundLocalError: cannot access local variable 'count' where it is not associated with a value` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The function assigns to `count`, which makes it local everywhere in the function, but reads it before the assignment (intending the global); or the variable is only assigned inside an `if`/`try` branch that did not run.

**Diagnose →** Find all assignments to the name inside the function.

**Fix →** Initialize the variable before branches, pass it in as a parameter and return the new value, or use a class/closure. Avoid `global`.

**Prevent →** ruff and mypy flag possibly-undefined variables; keep functions free of hidden global state.

</details>

## boto3, HTTP & API errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `botocore.exceptions.NoCredentialsError: Unable to locate credentials` 🟢

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

### Q52. `botocore.exceptions.NoRegionError: You must specify a region.` even though `AWS_REGION` is set. 🟡

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

### Q53. `botocore.exceptions.ClientError: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation.` 🟢

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

### Q54. `An error occurred (ExpiredToken) when calling the ... operation: The security token included in the request is expired` midway through a long job. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job built a Session from static STS credentials (manual `assume_role`) that expired after 1 hour; or SSO session expired.

**Diagnose →** Check how the Session is created and the job duration vs `DurationSeconds`.

**Fix →** Use a profile with `role_arn` + `source_profile`/`credential_source` so botocore refreshes automatically, or use `botocore.credentials.RefreshableCredentials` with a refresh callback. Increase role `MaxSessionDuration` if needed.

**Prevent →** Let botocore manage assume-role; avoid passing raw keys into `Session()`.

</details>

### Q55. `An error occurred (RequestLimitExceeded) when calling the DescribeInstances operation (reached max retries: 4): Request limit exceeded.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many EC2 API calls per second from your tool (parallel workers, per-resource describes) or other tools sharing the account's quota.

**Diagnose →** Count calls per phase with debug logs; CloudTrail shows the call volume by user agent.

**Fix →** `Config(retries={"mode": "adaptive", "max_attempts": 10})`, reduce concurrency, batch IDs into fewer calls, filter server-side.

**Prevent →** Design scanners around batch APIs and bounded concurrency from the start.

</details>

### Q56. `An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation: The security token included in the request is invalid.` 🟢

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

### Q57. `An error occurred (SignatureDoesNotMatch)` or `RequestTimeTooSkewed` from AWS. 🟡

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

### Q58. `botocore.exceptions.EndpointConnectionError: Could not connect to the endpoint URL: "https://ec2.ap-south-1.amazonaws.com/"` 🟡

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

### Q59. `botocore.errorfactory.NoSuchKey: An error occurred (NoSuchKey) when calling the GetObject operation`, but `head_object` on another key returns `ClientError` with code `404`. Why the inconsistency? 🔴

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

### Q60. `An error occurred (AccessDenied) when calling the PutObject operation: Access Denied` although the IAM policy allows `s3:PutObject`. 🔴

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

### Q61. `botocore.exceptions.WaiterError: Waiter InstanceRunning failed: Waiter encountered a terminal failure state: For expression "Reservations[].Instances[].State.Name" we matched expected path: "terminated"` 🟡

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

### Q62. `requests.exceptions.SSLError: HTTPSConnectionPool(host='artifactory.internal', port=443): Max retries exceeded ... [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate` 🟢

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

### Q63. `requests.exceptions.ConnectionError: HTTPSConnectionPool(host='api.example.com', port=443): Max retries exceeded with url: /v1/items (Caused by NameResolutionError(...: Failed to resolve 'api.example.com'))` 🟢

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

### Q64. A script hangs forever on `requests.get(url)` and never errors. 🟢

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

### Q65. `requests.exceptions.HTTPError: 429 Client Error: Too Many Requests for url: https://api.github.com/...` 🟡

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

### Q66. `urllib3.exceptions.MaxRetryError: ... (Caused by ResponseError('too many 503 error responses'))` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Your retry adapter retried on 503 until `total` was exhausted; the upstream is genuinely down, not briefly unavailable.

**Diagnose →** Check upstream health and how long retries took (backoff settings).

**Fix →** Treat it as an outage: fail clearly, alert, and do not retry harder. Consider a circuit breaker so you stop hammering a failing service.

**Prevent →** Bound total retry time; expose retry counts as metrics so persistent failure is visible early.

</details>

### Q67. `401 Unauthorized` from an internal API only for requests that follow a redirect. 🔴

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

### Q68. `requests` uploads work locally but fail in Kubernetes with `ProxyError: Unable to connect to proxy`. 🔴

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

### Q69. `kubernetes.client.exceptions.ApiException: (403) Reason: Forbidden ... "pods is forbidden: User \"system:serviceaccount:ops:auditor\" cannot list resource \"pods\" in API group \"\" at the cluster scope"` 🟢

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

### Q70. `kubernetes.config.config_exception.ConfigException: Service host/port is not set.` 🟢

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

### Q71. `ApiException: (410) Reason: Gone ... "too old resource version: 123456 (789012)"` in a watch loop. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The watch resumed from a `resourceVersion` older than the API server's retained history (etcd compaction).

**Diagnose →** The 410 appears after reconnects or long idle periods.

**Fix →** On 410, re-list to get a fresh `resource_version`, reconcile the full state, then start a new watch from that version. Use `allow_watch_bookmarks=True` to keep the version fresh.

**Prevent →** Use frameworks (kopf) or informer-style code that handles relist; design reconciliation to be level-triggered.

</details>

### Q72. `paramiko.ssh_exception.SSHException: Server '10.0.1.15' not found in known_hosts` 🟢

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

### Q73. `paramiko.ssh_exception.AuthenticationException: Authentication failed.` while `ssh` from the same machine works. 🟡

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

### Q74. `docker.errors.DockerException: Error while fetching server API version: ('Connection aborted.', FileNotFoundError(2, 'No such file or directory'))` 🟢

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

### Q75. `ImportError: urllib3 v2 only supports OpenSSL 1.1.1+, currently the 'ssl' module is compiled with 'OpenSSL 1.0.2k-fips'` 🔴

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

## Concurrency, async, tests, services & containers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `RuntimeError: asyncio.run() cannot be called from a running event loop` in a Jupyter notebook or inside an async framework. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An event loop is already running (Jupyter, FastAPI, another `asyncio.run` caller), and `asyncio.run` tries to start a new one.

**Diagnose →** Check whether the calling code is itself inside a coroutine or a notebook cell.

**Fix →** In Jupyter, use top-level `await main()`. Inside async code, `await` the coroutine or `asyncio.create_task(...)` instead of calling `asyncio.run`. From sync code in another thread, use `asyncio.run_coroutine_threadsafe(coro, loop)`.

**Prevent →** Only call `asyncio.run` once, at the program's entry point.

</details>

### Q77. `RuntimeWarning: coroutine 'fetch' was never awaited` and nothing happens. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You called an `async def` function without `await` (or without scheduling it), so only a coroutine object was created.

**Diagnose →** Find the call site named in the warning; run with `PYTHONASYNCIODEBUG=1` to get the creation traceback.

**Fix →** `result = await fetch(url)`, or `asyncio.create_task(fetch(url))` and keep a reference to the task.

**Prevent →** mypy and ruff (`RUF006` for dangling tasks) catch many of these; type-check async code.

</details>

### Q78. An asyncio health-checker is no faster than a sequential loop. 🔴

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

### Q79. `Task was destroyed but it is pending!` at shutdown. 🔴

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

### Q80. `OSError: [Errno 24] Too many open files` from an asyncio scanner with 20,000 targets. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** All coroutines start at once and each opens a socket; no concurrency bound.

**Diagnose →** `ulimit -n`; count concurrent connections (`ss -s`).

**Fix →** Wrap each request in `async with asyncio.Semaphore(200)`, and configure client connection limits (`httpx.Limits(max_connections=200)`, aiohttp `TCPConnector(limit=200)`).

**Prevent →** Always bound concurrency explicitly; never `gather` an unbounded list of network calls.

</details>

### Q81. A `ThreadPoolExecutor` job finishes "successfully" but half the work is missing, with no errors logged. 🟡

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

### Q82. Intermittent `KeyError: 'credential_provider'` or `KeyError: 'endpoint_resolver'` when creating boto3 clients from many threads. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clients created concurrently from the shared default session (`boto3.client(...)` inside worker threads). Session creation and component loading are not thread-safe.

**Diagnose →** The error appears only under concurrency and not in sequential runs.

**Fix →** Create one `boto3.session.Session()` per thread and build clients from it, or create the clients in the main thread before starting workers and share them (clients are thread-safe).

**Prevent →** Make "Session per thread" a team convention; our lab's `audit_region` does this.

</details>

### Q83. A multiprocessing script fails on macOS with `RuntimeError: An attempt has been made to start a new process before the current process has finished its bootstrapping phase.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `spawn` start method (default on macOS and Windows) re-imports the main module in each child; without an `if __name__ == "__main__":` guard, the children try to start pools themselves.

**Diagnose →** Check for top-level code that creates `Pool`/`ProcessPoolExecutor`.

**Fix →** Move the entry code under `if __name__ == "__main__": main()`.

**Prevent →** Always guard entry points; avoid side effects at import time.

</details>

### Q84. `_pickle.PicklingError: Can't pickle <function <lambda> at 0x...>` with `ProcessPoolExecutor`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Arguments and functions sent to worker processes must be picklable; lambdas, nested functions, open files, locks and boto3 clients are not.

**Diagnose →** The traceback names the object that failed.

**Fix →** Use a module-level function, pass plain data (IDs, dicts), and create clients inside the worker (a pool `initializer` can set them up once per process).

**Prevent →** Use processes only for CPU-bound work on plain data; use threads for API calls.

</details>

### Q85. A script with threads ignores Ctrl-C and must be killed with `kill -9`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The main thread is blocked in `join()` or `executor.shutdown(wait=True)` and worker threads (non-daemon) keep running; signals are only delivered to the main thread.

**Diagnose →** `py-spy dump --pid <pid>` shows the main thread waiting in `join`.

**Fix →** Use a `threading.Event` checked by workers; in the main thread, wait with timeouts (`as_completed(..., timeout=...)` loops or `event.wait(1)`), catch `KeyboardInterrupt`, set the event, and `shutdown(cancel_futures=True)`.

**Prevent →** Build cooperative cancellation into long-running workers from day one.

</details>

### Q86. pytest with moto fails with `botocore.exceptions.ClientError: An error occurred (InvalidClientTokenId) when calling the ... operation` (it is reaching real AWS). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client was created outside the `mock_aws` context (at module import, in a session-scoped fixture created before the mock started), so it talks to real AWS with fake credentials.

**Diagnose →** Check where clients are created relative to `mock_aws()` / `@mock_aws`.

**Fix →** Create clients inside the mock (in fixtures that depend on the mock fixture), not at import time. Set fake credentials in an autouse fixture so a leak fails fast instead of using your real profile.

**Prevent →** No module-level clients; dependency-inject clients into functions.

</details>

### Q87. A moto-based test expects old snapshots to be flagged, but `describe_snapshots` always shows `StartTime` as now. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** moto stamps resources with the current time and does not let you set creation timestamps through the API.

**Diagnose →** Print `StartTime` in the test.

**Fix →** Test the age logic with a stub (`unittest.mock.MagicMock` paginator or botocore `Stubber`) returning controlled timestamps, or freeze/travel time (`time-machine`) so "now" during the check is far in the future relative to creation.

**Prevent →** Separate the "is it old?" rule into a pure function that takes `now` as a parameter.

</details>

### Q88. `fixture 'mocker' not found` in a pytest run on CI. 🟢

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

### Q89. Tests pass individually but `pytest -n auto` (xdist) causes random failures. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tests share external state across processes: the same temp file path, same port, same S3 bucket name in a real environment, or a session fixture assumed to run once.

**Diagnose →** Run with `-n 2 -p no:randomly` and look for colliding resource names.

**Fix →** Use `tmp_path`, ephemeral ports (bind to port 0), unique names (`f"test-{uuid4().hex[:8]}"`), and `worker_id`-aware fixtures for shared resources.

**Prevent →** Treat every test as potentially concurrent.

</details>

### Q90. `click.testing.CliRunner` test shows `exit_code == 1` with empty output. 🟢

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

### Q91. Flask app in Docker is unreachable: `curl: (52) Empty reply from server` or connection refused on the published port. 🟢

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

### Q92. Gunicorn logs `[CRITICAL] WORKER TIMEOUT (pid:42)` and requests fail with 502 from the load balancer. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A request took longer than gunicorn's `--timeout` (30s default) on a sync worker, usually a slow downstream call without a timeout, or CPU-bound work.

**Diagnose →** Correlate timeouts with endpoints in access logs; `py-spy dump` on a worker during a slow request.

**Fix →** Add timeouts to outbound calls, move long jobs to a background queue, use `gthread` or async workers for I/O-bound handlers, and raise `--timeout` only if the work is legitimately long.

**Prevent →** Latency histograms per endpoint and timeouts on all external calls.

</details>

### Q93. FastAPI returns `422 Unprocessable Entity` for a webhook from a third party. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The request body does not match the pydantic model: missing fields, wrong types, or the sender posts `application/x-www-form-urlencoded` (Slack slash commands do) instead of JSON.

**Diagnose →** Read the response body; FastAPI lists the failing fields and reasons. Log `request.headers["content-type"]`.

**Fix →** Adjust the model (optional fields, correct types), or accept `Form(...)` parameters / raw `Request` body for non-JSON senders.

**Prevent →** Model webhooks from real sample payloads and test with them.

</details>

### Q94. GitHub webhook signature verification fails for every request, though the secret is correct. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The HMAC is computed over re-serialized JSON (`json.dumps(request.json)`) instead of the raw body bytes, or the secret has trailing whitespace/newline from the secret store, or comparing against `X-Hub-Signature` (SHA-1) instead of `X-Hub-Signature-256`.

**Diagnose →** Log lengths (never values) of the secret and body; compare computed vs received prefix.

**Fix →** Use `request.get_data()` (Flask) or `await request.body()` (FastAPI) for the HMAC, `.strip()` the secret, and compare `"sha256=" + hexdigest` with `hmac.compare_digest`.

**Prevent →** Unit test the verifier with a known payload, secret and signature.

</details>

### Q95. Prometheus shows counters from your gunicorn app jumping up and down between scrapes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiple gunicorn workers each keep their own metrics; each scrape hits a different worker.

**Diagnose →** Compare `/metrics` output from repeated curls; values differ per request.

**Fix →** Enable `prometheus_client` multiprocess mode: set `PROMETHEUS_MULTIPROC_DIR` (empty, writable) before startup, serve metrics through `MultiProcessCollector`, call `multiprocess.mark_process_dead(worker.pid)` in gunicorn's `child_exit` hook.

**Prevent →** Use the multiprocess setup in your service template whenever workers > 1.

</details>

### Q96. `ValueError: Duplicated timeseries in CollectorRegistry: {'requests_total', 'requests_created', 'requests'}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The same metric is registered twice: module imported twice under different names, metric defined inside a function or app factory that runs more than once, or tests creating the app repeatedly.

**Diagnose →** Find where the metric is constructed and how often that code runs.

**Fix →** Define metrics once at module level; in app factories or tests, pass a dedicated `CollectorRegistry()` per instance.

**Prevent →** Keep metric definitions in one module; use a fresh registry in tests.

</details>

### Q97. A Python container in Kubernetes is `OOMKilled` while the app reports only 200 MB used. 🔴

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

### Q98. Your Python logs do not appear in `docker logs` or `kubectl logs` until the container exits. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** stdout is block-buffered when not attached to a TTY, so output sits in a buffer.

**Diagnose →** Logs appear in bursts or only at exit.

**Fix →** Set `ENV PYTHONUNBUFFERED=1` in the Dockerfile (or run `python -u`), and use `logging` to stderr (unbuffered by default).

**Prevent →** Include `PYTHONUNBUFFERED=1` in every Python base image.

</details>

### Q99. The scheduled GitHub Actions audit fails with `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role trust policy's `sub` condition does not match the token (wrong repo or branch, or the job uses an `environment:`, which changes `sub` to `repo:org/repo:environment:prod`), the `aud` is not `sts.amazonaws.com`, the OIDC provider is missing, or the workflow lacks `permissions: id-token: write`.

**Diagnose →** Check the trust policy conditions against the workflow; scheduled runs use the default branch, so `sub` is `repo:org/repo:ref:refs/heads/main`.

**Fix →** Align the `token.actions.githubusercontent.com:sub` condition (`StringLike` with the exact pattern), add `id-token: write`, verify the provider and audience.

**Prevent →** Keep the trust policy in Terraform next to the workflow; test with `workflow_dispatch` on main before relying on the schedule.

</details>

### Q100. The containerized audit CLI exits 0 in GitHub Actions but the report file is missing from the uploaded artifact. 🟡

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
