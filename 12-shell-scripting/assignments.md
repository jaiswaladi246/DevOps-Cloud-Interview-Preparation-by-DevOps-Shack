# Shell Scripting — Assignments

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

> 25 practical assignments · 🟢 10 Beginner · 🟡 10 Intermediate · 🔴 5 Advanced

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

Do these on a disposable Ubuntu 24.04 VM or container (`docker run -it --rm ubuntu:24.04 bash`, then `apt-get update && apt-get install -y shellcheck jq curl bats`). Use Bash 5.x unless an assignment says otherwise. Every script you write should pass `shellcheck` with zero findings before you call it done. Open the hint only after you have tried for 10 minutes.

## Table of contents
- [🟢 Beginner](#beginner) — A1–A10
- [🟡 Intermediate](#intermediate) — A11–A20
- [🔴 Advanced](#advanced) — A21–A25

---

## Beginner

### A1. Three ways to run a script 🟢

**Objective:** Understand the shebang, the execute bit, and the difference between running a script and sourcing it.

**Tasks**
1. Write `whereami.sh` with the shebang `#!/usr/bin/env bash` that sets `GREETING=hello`, runs `cd /tmp`, and prints `$PWD`, `$GREETING`, `$0` and `$$`.
2. Run it three ways: `./whereami.sh` (after `chmod +x`), `bash whereami.sh`, and `source whereami.sh`.
3. After each run, print `$PWD` and `$GREETING` in your interactive shell and record what changed.
4. Change the shebang to `#!/bin/sh`, add `[[ -n $GREETING ]] && echo ok`, and run it with `./whereami.sh`. Note the error on Ubuntu.
5. Remove the execute bit and show the exact error message for `./whereami.sh`.

**Expected outcome:** A short table showing that only `source` changed your shell's directory and variables, `$0` differs between the three methods, `/bin/sh` (dash) fails on `[[`, and a missing execute bit gives `Permission denied`.

<details>
<summary>Hints</summary>

`./script` uses the shebang and needs `+x`. `bash script` ignores the shebang. `source` runs the file in the current shell, so `cd` and variables persist, and `$0` stays as your shell's name. On Ubuntu `/bin/sh` is `dash`, which prints `[[: not found`.

</details>

**Bonus:** Add a guard so the script refuses to be sourced: `[[ ${BASH_SOURCE[0]} == "$0" ]] || { echo "do not source me" >&2; return 1; }`.

### A2. Break it, then quote it 🟢

**Objective:** See word splitting and globbing destroy real filenames, then fix every expansion.

**Tasks**
1. Create a test directory containing `report 2026.csv`, `*.txt` (a file literally named with an asterisk), `-rf`, and `notes.txt`.
2. Write `count_lines.sh` that loops `for f in $(ls)` and runs `wc -l $f`. Run it and save the errors.
3. Rewrite it with a glob (`for f in ./*`), quoted expansions and `--` before filenames.
4. Use `printf '[%s]\n' $var` and `printf '[%s]\n' "$var"` to show how many words a value becomes in each case.
5. Run `shellcheck` on both versions and list the warning codes the first one triggers.

**Expected outcome:** The fixed script prints a correct line count for all four files, including `-rf` and `*.txt`, and ShellCheck reports nothing.

<details>
<summary>Hints</summary>

`$(ls)` output is split on spaces and globbed, so `report 2026.csv` becomes two words and `*.txt` expands. Use `for f in ./*; do wc -l -- "$f"; done`. The `./` prefix (or `--`) stops `-rf` being read as options. Expect SC2045 (iterating over `ls`) and SC2086 (unquoted variable).

</details>

**Bonus:** Make the script handle an empty directory without running `wc` on a literal `./*` (`shopt -s nullglob`).

### A3. Parameter expansion instead of external tools 🟢

**Objective:** Replace `basename`, `dirname`, `sed` and `tr` calls with parameter expansion.

**Tasks**
1. Given `path=/var/log/nginx/access.log.2026-11-09.gz`, print the filename, the directory, the name without `.gz`, the extension, and the date part, using only `${...}` forms.
2. Print `$path` with every `/` replaced by `_`, and in upper case.
3. Write `config_value() { ... }` that prints `${1}` if set and non-empty, otherwise a default passed as `$2`, using `${var:-default}`.
4. Make a script that aborts with a clear message if `ENV` is unset, using `${ENV:?...}`.
5. Print the length of `$path` and the first 8 characters of it.

**Expected outcome:** Every value is printed with zero external commands (check with `bash -x`: no `basename`, `sed` or `cut` lines).

<details>
<summary>Hints</summary>

`${path##*/}` (filename), `${path%/*}` (directory), `${path%.gz}`, `${path##*.}`, `${path//\//_}`, `${path^^}`, `${#path}`, `${path:0:8}`. For the date, strip the prefix up to `log.` and the suffix `.gz`: `d=${path##*log.}; d=${d%.gz}`.

</details>

**Bonus:** Write a function that renames every `*.jpeg` file in a directory to `*.jpg` using only parameter expansion and `mv --`.

### A4. A service wrapper with `case` and exit codes 🟢

**Objective:** Use `case`, `[[ ]]` and meaningful exit codes to build a small CLI.

**Tasks**
1. Write `svc.sh ACTION SERVICE` where ACTION is `start`, `stop`, `restart` or `status`.
2. Validate the arguments: wrong count or unknown action prints usage to stderr and exits 2.
3. Validate the service name against `^[a-z0-9@._-]+$` before passing it to `systemctl`.
4. `status` prints `SERVICE is active` or `SERVICE is not active (state)` and exits 0 or 3.
5. Add `-h` that prints usage to stdout and exits 0.

**Expected outcome:** `./svc.sh status ssh; echo $?` prints the state and 0 or 3; `./svc.sh bounce ssh` prints usage and exits 2; `./svc.sh status 'ssh; rm -rf /'` is rejected.

<details>
<summary>Hints</summary>

`case $1 in start|stop|restart) ... ;; status) ... ;; -h) usage; exit 0 ;; *) usage >&2; exit 2 ;; esac`. Validate with `[[ $2 =~ ^[a-z0-9@._-]+$ ]]`. `systemctl is-active --quiet "$svc"` returns 0 only when active; `systemctl is-active "$svc"` prints the state.

</details>

**Bonus:** Add `--dry-run` that prints the `systemctl` command instead of running it.

### A5. Read a CSV without losing data 🟢

**Objective:** Write a `while read` loop that survives spaces, backslashes, a missing final newline and comment lines.

**Tasks**
1. Create `users.csv` with a header `username,full_name,shell`, five rows (one full name containing a backslash, one with leading spaces), a `#` comment line, a blank line, and **no** newline after the last row (`printf` it).
2. Write `list_users.sh` that skips the header, comments and blank lines and prints `username -> full_name (shell)` for each row.
3. Count valid rows in a variable and print the total after the loop.
4. Make sure the total is correct (a `cmd | while` version will print 0: show it, then fix it).
5. Reject rows whose shell is not listed in `/etc/shells`, printing a warning to stderr.

**Expected outcome:** All five rows print exactly as written (backslash and spaces intact), the last row is not skipped, and the total says 5.

<details>
<summary>Hints</summary>

`while IFS=, read -r user name shell || [[ -n $user ]]; do ... done < users.csv`. `-r` keeps backslashes; the `|| [[ -n $user ]]` handles the last line without a newline. A pipeline runs the loop in a subshell, so counters are lost: redirect the file into the loop instead. Skip with `[[ -z $user || $user == \#* || $user == username ]] && continue`. Check shells with `grep -qxF -- "$shell" /etc/shells`.

</details>

**Bonus:** Emit each valid row as a JSON object (one per line) with `jq -cn --arg`, so a name containing quotes or backslashes still produces valid JSON.

### A6. A tiny function library 🟢

**Objective:** Write reusable functions with `local` variables, return codes and output, and source them from two scripts.

**Tasks**
1. Create `lib.sh` with `log LEVEL MSG` (timestamped, to stderr), `die MSG [CODE]`, `require_cmd CMD...` (exit 127 if missing) and `is_port_open HOST PORT` (returns 0 or 1, prints nothing).
2. Use `local` for every variable inside the functions.
3. Write `check.sh` that sources `lib.sh` relative to its own location (so it works from any directory) and checks ports 22 and 80 on localhost.
4. Write `need.sh` that calls `require_cmd jq curl not-a-real-tool` and show the exit code.
5. Demonstrate the difference between a function's **return code** and its **output** with `result=$(fn)` and `if fn; then`.

**Expected outcome:** Both scripts work when run from `/tmp` and from their own directory; `need.sh` exits 127 naming the missing tool; logs go to stderr so `./check.sh > /dev/null` still shows them.

<details>
<summary>Hints</summary>

Source relative to the script: `source "$(dirname -- "${BASH_SOURCE[0]}")/lib.sh"`. Port check without extra tools: `timeout 2 bash -c "</dev/tcp/$1/$2" 2>/dev/null`. `return` sets the status (0 to 255); `echo` produces output you capture with `$(...)`.

</details>

**Bonus:** Make `lib.sh` safe to source twice (a guard variable), and make it refuse to be executed directly.

### A7. Summarize an access log with a pipeline 🟢

**Objective:** Answer real incident questions with `grep`, `awk`, `cut`, `sort`, `uniq` and `head`.

**Tasks**
1. Generate or download an Nginx access log in the combined format with at least 10,000 lines (a small `awk` or Bash generator with random IPs, paths and status codes is fine).
2. Print the top 10 client IPs by request count.
3. Print the count of each status code, sorted by code.
4. Print the top 5 paths that returned any 5xx status, with counts.
5. Print the number of requests per minute for the busiest 5 minutes.
6. Put all four reports in one script, `logreport.sh FILE`, with `set -euo pipefail`.

**Expected outcome:** `./logreport.sh access.log` prints four labelled sections and exits 0; it exits non-zero with a clear message when the file does not exist.

<details>
<summary>Hints</summary>

Fields in the combined format: `$1` IP, `$4` `[09/Nov/2026:10:15:02`, `$7` path, `$9` status. Top IPs: `awk '{print $1}' f | sort | uniq -c | sort -rn | head`. 5xx paths: `awk '$9 ~ /^5/ {print $7}'`. Per minute: `awk '{print substr($4, 2, 17)}'`. `uniq` only merges adjacent lines, so always `sort` first.

</details>

**Bonus:** Rewrite the whole report as a single `awk` program that reads the file once, and compare the run time with `time`.

### A8. Safe cleanup with `find` and `xargs` 🟢

**Objective:** Select files by age, size and name, and delete them safely, including names with spaces and newlines.

**Tasks**
1. Create `/tmp/a8` with 50 files: varied ages (`touch -d '10 days ago'`), some with spaces, one with a newline in its name, and a few larger than 10 MB (`fallocate -l 12M`).
2. Write `cleanup.sh DIR DAYS` that lists files older than DAYS days, with their sizes, and the total size that would be freed.
3. Add `--delete`: without it the script only reports (dry run is the default).
4. Make it NUL-safe end to end (`-print0`, `xargs -0` or `-exec ... +`).
5. Refuse to run when DIR is empty, `/`, or not under `/tmp` or `/var/tmp`.

**Expected outcome:** The report matches `find DIR -type f -mtime +DAYS | wc -l`; `--delete` removes exactly those files including the one with a newline; `./cleanup.sh / 1 --delete` is refused.

<details>
<summary>Hints</summary>

`find "$dir" -type f -mtime +"$days" -printf '%s\t%p\0'` gives size and path, NUL-terminated. Sum with `awk -v RS='\0' -F'\t' '{s+=$1} END {print s}'`. Delete with `find ... -delete` or `-print0 | xargs -0 -r rm -f --`. Guard: `[[ $dir == /tmp/* || $dir == /var/tmp/* ]] || die "refusing"`.

</details>

**Bonus:** Print the freed space in human units with `numfmt --to=iec`.

### A9. A disk alert that really runs from cron 🟢

**Objective:** Write a check that works under cron's minimal environment and leaves evidence that it ran.

**Tasks**
1. Write `diskcheck.sh` that checks every real filesystem (skip `tmpfs`, `overlay`, `squashfs`) and prints `WARNING` at 80% and `CRITICAL` at 90%, for blocks and inodes.
2. Exit 0, 1 or 2 depending on the worst result.
3. Run it with `env -i /path/to/diskcheck.sh` to simulate cron's empty environment and fix whatever breaks.
4. Schedule it every 10 minutes in your user crontab, writing to `~/logs/diskcheck.log` with a timestamp on every line.
5. Fill a filesystem past 80% with `fallocate` and confirm the log shows the warning within 10 minutes; then remove the file.

**Expected outcome:** The crontab line uses absolute paths, the log shows a run every 10 minutes, the warning appears and disappears, and `env -i` runs cleanly.

<details>
<summary>Hints</summary>

`df -P -x tmpfs -x overlay -x squashfs -x devtmpfs` and `df -Pi ...` for inodes; strip `%` with `awk '{sub(/%/, "", $5); print $5, $6}'`. Set `PATH=/usr/local/bin:/usr/bin:/bin` at the top of the script. Crontab: `*/10 * * * * /home/you/bin/diskcheck.sh >> /home/you/logs/diskcheck.log 2>&1`. Timestamp lines inside the script with `printf '%(%F %T)T ...' -1`.

</details>

**Bonus:** Remember the last state in a file and print only when the state changes.

### A10. Generate config files with here-docs 🟢

**Objective:** Use quoted and unquoted here-doc delimiters correctly, and render templates without breaking `$` that belongs to the target format.

**Tasks**
1. Write `mkvhost.sh DOMAIN PORT` that writes an Nginx server block to `./out/DOMAIN.conf` with a here-doc, substituting the domain and upstream port.
2. The block must contain Nginx variables (`$host`, `$remote_addr`, `$proxy_add_x_forwarded_for`) that must **not** be expanded by Bash.
3. Write a second version that keeps the template in `vhost.conf.tmpl` and renders it with `envsubst`, substituting only `DOMAIN` and `PORT`.
4. Validate inputs: the domain must match a hostname regex and the port must be 1 to 65535.
5. Write the output atomically (temp file in the same directory, then `mv`).

**Expected outcome:** Both versions produce identical files; `grep -c '\$host' out/*.conf` shows the Nginx variables survived; running `nginx -t -c` against a minimal wrapper config (or in an `nginx` container) passes.

<details>
<summary>Hints</summary>

In an unquoted here-doc (`<<CONF`) escape Nginx variables as `\$host`. With a quoted delimiter (`<<'CONF'`) nothing expands, which is what you want for templates. `envsubst '${DOMAIN} ${PORT}' < vhost.conf.tmpl` substitutes only the listed variables. Port check: `[[ $port =~ ^[0-9]+$ ]] && (( port >= 1 && port <= 65535 ))`.

</details>

**Bonus:** Use `<<-` with tab indentation inside a function so the template lines up with the code.

---

## Intermediate

### A11. Strict-mode bug hunt 🟡

**Objective:** Learn exactly where `set -euo pipefail` protects you and where it silently does not.

**Tasks**
1. Save this script as `buggy.sh` and run it. Note what it prints and its exit code.
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   count=0
   check() { false; echo "check kept going"; }
   if check; then echo "check passed"; fi
   get() { local v=$(cat /nonexistent); echo "v=[$v]"; }
   get
   errors=$(grep -c ERROR /dev/null)
   ((count++))
   echo "count=$count errors=$errors"
   ```
2. For each line, write down whether strict mode should stop the script there and why it does or does not.
3. Fix the script so that every real failure stops it with a message naming the line, and every expected "no result" is handled explicitly.
4. Add `set -E`, `shopt -s inherit_errexit` and an `ERR` trap that prints `$LINENO`, `$BASH_COMMAND` and the exit status.
5. Prove each fix with a one-line command that fails before and passes after.

**Expected outcome:** A table of five pitfalls (function in `if`, `local x=$(...)`, `grep -c` with no match, `((count++))` from 0, command substitution subshells) and a fixed script with zero ShellCheck findings that fails loudly only on real errors.

<details>
<summary>Hints</summary>

`set -e` is ignored inside anything called from an `if` condition. `local` masks the substitution's status (SC2155): declare and assign separately. `grep -c` prints `0` and exits 1: use `|| true` on that one command. `((count++))` returns 1 when `count` was 0: use `count=$((count + 1))`. A trap: `trap 'echo "ERR line $LINENO: $BASH_COMMAND ($?)" >&2' ERR`.

</details>

**Bonus:** Write the same five pitfalls as five bats tests that each assert the fixed behaviour.

### A12. A real CLI with `getopts` 🟡

**Objective:** Build a command-line interface that humans and CI can both rely on.

**Tasks**
1. Write `release.sh` accepting `-e ENV` (required: `dev`, `staging` or `prod`), `-v VERSION` (required, semantic version), `-n` (dry run), `-y` (skip confirmation) and `-h`.
2. Unknown options and missing option arguments must print a specific error plus usage to stderr and exit 2.
3. Validate `VERSION` against `^v?[0-9]+\.[0-9]+\.[0-9]+$` and `ENV` against an allow-list.
4. For `prod` without `-y`, ask for confirmation only when stdin is a terminal; when it is not (CI), refuse unless `-y` was given.
5. Remaining positional arguments (after `--`) are passed through as extra flags to the deploy command, preserved exactly as given.

**Expected outcome:** `./release.sh -e prod -v 1.4.2 -n -y -- --force "two words"` prints the command it would run with the extra arguments intact; `./release.sh -e qa -v 1.4.2` exits 2; `echo | ./release.sh -e prod -v 1.4.2` refuses without `-y`.

<details>
<summary>Hints</summary>

`while getopts ':e:v:nyh' opt; do case $opt in ...; :) echo "-$OPTARG needs an argument"; usage; exit 2;; \?) echo "unknown option -$OPTARG"; ...; esac; done; shift $((OPTIND - 1))`. The leading `:` enables silent error handling. Terminal check: `[[ -t 0 ]]`. Keep extras as `extra=("$@")` and expand as `"${extra[@]}"`.

</details>

**Bonus:** Support long options (`--env`, `--version`, `--dry-run`) with a manual `while (( $# ))` loop, and explain why `getopt` (with a t) is not portable to macOS.

### A13. Traps that always clean up 🟡

**Objective:** Guarantee cleanup on success, failure, Ctrl+C and `SIGTERM`, and understand what cannot be trapped.

**Tasks**
1. Write `work.sh` that creates a temp directory with `mktemp -d`, writes a lock file, and runs a 60-second loop that writes progress into the temp directory.
2. Install traps so the temp directory and lock are removed on normal exit, on any error, on Ctrl+C (exit 130) and on `SIGTERM` (exit 143).
3. Test each case: let it finish, force an error midway, press Ctrl+C, and `kill -TERM` it from another terminal. Confirm cleanup each time with `ls`.
4. Kill it with `kill -9` and observe what is left behind. Explain why.
5. Make the script detect and remove leftovers from a previous `kill -9` run on start-up, safely.

**Expected outcome:** No leftovers after the first four cases, a documented leftover after `kill -9`, and a start-up routine that cleans it without touching another live run.

<details>
<summary>Hints</summary>

`trap cleanup EXIT`, `trap 'exit 130' INT`, `trap 'exit 143' TERM`: the signal traps call `exit`, which then runs the `EXIT` trap once. Use single quotes in trap strings. `SIGKILL` cannot be trapped. Safe leftover removal: take an `flock` first, then anything left with your naming pattern belongs to a dead run.

</details>

**Bonus:** Run a child process (`sleep 300 &`) from the script and make sure Ctrl+C also stops the child (`kill -- -$$` or tracking PIDs and `wait`).

### A14. Inventory with indexed and associative arrays 🟡

**Objective:** Use arrays to hold structured data and to build commands without string-splitting bugs.

**Tasks**
1. Write an INI-style inventory `hosts.ini` with groups `[web]`, `[db]`, `[cache]`, 2 to 4 hosts each, comments and blank lines.
2. Parse it in pure Bash into an associative array `group -> "space separated hosts"` and an indexed array of all hosts.
3. Implement `inventory.sh list`, `inventory.sh hosts GROUP`, and `inventory.sh count` (hosts per group, sorted by group name).
4. Build an `rsync` command in an array with conditional options (`--dry-run` when `DRY_RUN=1`, `--exclude` entries from a list that contains a pattern with a space) and print it with `printf '%q '`.
5. Reject unknown groups with a helpful message listing the valid ones.

**Expected outcome:** `./inventory.sh hosts db` prints the db hosts one per line; the printed `rsync` command shows each option as a separate, correctly quoted argument.

<details>
<summary>Hints</summary>

`declare -A groups=()`. Detect a section with `[[ $line =~ ^\[([a-z]+)\]$ ]] && group=${BASH_REMATCH[1]}`. Append: `groups[$group]+="$host "`. Sorted keys: `printf '%s\n' "${!groups[@]}" | sort`. Options: `opts=(-a --delete); [[ ${DRY_RUN:-0} == 1 ]] && opts+=(--dry-run); for e in "${excludes[@]}"; do opts+=(--exclude "$e"); done`.

</details>

**Bonus:** Add host variables (`web1 ansible_port=2222`) stored in a second associative array keyed by host.

### A15. JSON and YAML in scripts with `jq` and `yq` 🟡

**Objective:** Query, transform and edit structured data safely from Bash.

**Tasks**
1. Save the output of `kubectl get pods -A -o json` (or any similar JSON; a sample from a kind cluster is fine) to `pods.json`.
2. With `jq`, print a TSV of namespace, pod name, phase and total restart count, sorted by restarts descending.
3. Print pods whose containers were last terminated with reason `OOMKilled`, handling pods with no `lastState` without errors.
4. Pass a namespace from a shell variable into the filter with `--arg`, never by string interpolation.
5. With mikefarah `yq` v4, change `image.tag` in a Helm `values.yaml` to the value of `$TAG`, add a `podAnnotations.deployedBy` key, and show the diff.

**Expected outcome:** A script `k8s-report.sh` producing the TSV and the OOMKilled list; `values.yaml` updated in place with the comments preserved; no `null` or `Cannot iterate over null` errors.

<details>
<summary>Hints</summary>

Restarts: `[.status.containerStatuses[]?.restartCount] | add // 0`. TSV: `[.metadata.namespace, .metadata.name, .status.phase, $r] | @tsv`. OOMKilled: `select(any(.status.containerStatuses[]?; .lastState.terminated.reason? == "OOMKilled"))`. yq: `TAG=1.4.2 yq -i '.image.tag = strenv(TAG) | .podAnnotations.deployedBy = "ci"' values.yaml`.

</details>

**Bonus:** Produce the same report as CSV with `@csv`, and as a Markdown table suitable for a pull request comment.

### A16. A parallel fleet runner 🟡

**Objective:** Run a command across many hosts with bounded concurrency, per-host logs and a reliable summary.

**Tasks**
1. Write `fleet.sh -f HOSTS_FILE -p PARALLEL -- COMMAND...` that runs COMMAND on each host over SSH with `-o BatchMode=yes -o ConnectTimeout=5`.
2. Limit concurrency to PARALLEL jobs at a time.
3. Save each host's stdout and stderr to `logs/HOST.log` and its exit code to `logs/HOST.rc`.
4. Print a summary table (host, exit code, duration) sorted with failures first, and exit with the number of failed hosts (capped at 125).
5. Make Ctrl+C stop all running SSH sessions and still print the partial summary.

**Expected outcome:** Against 10 hosts (containers with sshd, or `localhost` aliases in `~/.ssh/config`), the run never exceeds PARALLEL concurrent sessions and the summary matches the `.rc` files exactly.

<details>
<summary>Hints</summary>

Bound concurrency in Bash: `(( $(jobs -rp | wc -l) >= max )) && wait -n` before each launch. Or delegate: `xargs -a hosts -P "$max" -I{} ./one_host.sh {}`. Use `ssh -n` so SSH does not swallow the loop's stdin. Time each job with `SECONDS`. For Ctrl+C, `trap 'kill $(jobs -p) 2>/dev/null' INT` and then print the summary from what exists.

</details>

**Bonus:** Add `--canary N`: run on the first N hosts, stop if any fails, otherwise continue with the rest.

### A17. From cron to a sandboxed systemd timer 🟡

**Objective:** Convert a cron job into a service and timer pair, with logging, sandboxing and failure alerts.

**Tasks**
1. Take your `diskcheck.sh` from A9 (or any script) and write `diskcheck.service` (`Type=oneshot`, dedicated user, `SyslogIdentifier=`) and `diskcheck.timer` (every 10 minutes, starting 2 minutes after boot).
2. Add sandboxing: `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `NoNewPrivileges=yes`, and only the `ReadWritePaths=` the script needs.
3. Add `OnFailure=` pointing at a template unit `notify@.service` that writes the failing unit name to the journal (or posts to a webhook).
4. Remove the crontab entry, enable the timer, and show `systemctl list-timers` and `journalctl -u diskcheck`.
5. Run `systemd-analyze security diskcheck.service` and improve the score with at least two more directives.

**Expected outcome:** The timer fires every 10 minutes, output is in the journal with your identifier, a forced failure (exit 3) triggers `notify@diskcheck.service.service`, and the security score is lower than at the start.

<details>
<summary>Hints</summary>

Timer: `OnBootSec=2min`, `OnUnitActiveSec=10min`, `[Install] WantedBy=timers.target`. Template: `ExecStart=/usr/bin/logger -t notify "unit %i failed on %H"`. After editing units, `systemctl daemon-reload`. Directives that help the score: `ProtectKernelTunables=yes`, `ProtectControlGroups=yes`, `RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6`, `CapabilityBoundingSet=`, `PrivateDevices=yes`.

</details>

**Bonus:** If your monitoring considers WARNING (exit 1) a valid result, use `SuccessExitStatus=1` so only real failures trigger `OnFailure=`.

### A18. Log housekeeping that is idempotent 🟡

**Objective:** Write a retention script that is safe to run any number of times, concurrently, on a live application directory.

**Tasks**
1. Simulate an app writing `app-YYYY-MM-DD.log` daily: generate 40 days of files and keep a writer appending to today's file.
2. Write `logkeep.sh DIR` that gzips files older than 1 day (never today's), deletes `.gz` files older than 14 days, and then deletes the oldest `.gz` files until the directory is under a size budget (for example 200 MB).
3. Make it idempotent: a second immediate run changes nothing and reports `nothing to do`.
4. Make it safe to run concurrently (two runs at once must not double-compress or race) with `flock`.
5. Verify each gzip before deleting the original (`gzip -t`), and never leave both a `.log` and a `.log.gz` for the same day.

**Expected outcome:** After one run: today's log untouched, days 2 to 14 compressed, older ones gone, total size under budget. A second run and two parallel runs both leave the directory unchanged.

<details>
<summary>Hints</summary>

Select by the date in the name, not just `mtime`: compare `${f//[!0-9]/}` with `$(date -d '1 day ago' +%Y%m%d)`. Compress to a temp name and rename: `gzip -c "$f" > "$f.gz.tmp" && gzip -t "$f.gz.tmp" && mv "$f.gz.tmp" "$f.gz" && rm -- "$f"`. Size: `du -sb "$dir" | cut -f1`. Lock: `exec 9>"$dir/.logkeep.lock"; flock -n 9 || exit 0`.

</details>

**Bonus:** Write the same policy as a `logrotate` config and explain in two sentences when you would prefer each.

### A19. ShellCheck to zero and a bats suite in CI 🟡

**Objective:** Bring three existing scripts to zero ShellCheck findings and cover them with tests that stub external commands.

**Tasks**
1. Pick three of your earlier scripts (for example A4, A9 and A12) and run `shellcheck -x` on them. Fix every finding; for any you suppress, write the reason in the same comment.
2. Add a `.shellcheckrc` with `source-path=SCRIPTDIR` and `external-sources=true`.
3. Write at least 10 bats tests across the three scripts covering success, usage errors, and failure paths.
4. Stub `systemctl`, `df` and `ssh` with scripts in `test/stubs/` placed first on `PATH`, recording their arguments to a log you assert on.
5. Add a `Makefile` with `lint` and `test` targets and a GitHub Actions workflow running `make lint test` on every pull request.

**Expected outcome:** `make lint test` is green locally and in CI; deliberately introducing an unquoted variable makes CI fail on ShellCheck; deliberately breaking an exit code makes a bats test fail.

<details>
<summary>Hints</summary>

In bats, `run cmd` captures `$status` and `$output`; each test gets a fresh `$BATS_TEST_TMPDIR`. Put `export PATH="$BATS_TEST_DIRNAME/stubs:$PATH"` in `setup()`. Remember that a bare `! cmd` in the middle of a test does not fail it; assert counts or use `run !`. Install in CI with `sudo apt-get install -y shellcheck bats`.

</details>

**Bonus:** Add a pre-commit hook (`.git/hooks/pre-commit` or the `pre-commit` framework) that runs ShellCheck on staged `*.sh` files only.

### A20. Release directories, atomic switch and automatic rollback 🟡

**Objective:** Write a deploy script whose switch-over is atomic and whose rollback is automatic when a health check fails.

**Tasks**
1. Create `/srv/demo/releases/` and a `current` symlink. Each release is a directory named by UTC timestamp containing a tiny app (for example `python3 -m http.server` serving a `version.txt`).
2. Write `deploy.sh ARTIFACT.tar.gz` that unpacks into a new release directory, runs a pre-flight check, switches `current` atomically, and restarts the service.
3. Poll a health URL for up to 30 seconds after the switch; if it does not report the new version, switch back to the previous release and exit non-zero.
4. Keep the newest 5 releases, never deleting the one `current` points to.
5. Write `rollback.sh` that switches to the previous release on demand.

**Expected outcome:** A good artifact deploys with zero failed requests (run a `curl` loop during the deploy); a bad artifact (health never passes) is rolled back automatically and the script exits non-zero.

<details>
<summary>Hints</summary>

Atomic switch: `ln -sfn "$new" /srv/demo/current.tmp && mv -T /srv/demo/current.tmp /srv/demo/current` (rename is atomic; `ln -sfn` alone is not). Previous release: `readlink -f /srv/demo/current` before switching. Keep 5: `ls -1d /srv/demo/releases/*/ | sort | head -n -5`, skipping the current target.

</details>

**Bonus:** Add `flock` so two deploys cannot overlap, and log every deploy (who, when, which release, result) to a JSON Lines audit file.

---

## Advanced

### A21. Expansion-order puzzles and production-grade tracing 🔴

**Objective:** Predict Bash's behaviour from first principles and debug scripts with traces that are useful and safe.

**Tasks**
1. Predict, then verify, the output of each line, and explain each result by naming the expansion step responsible:
   ```bash
   x='*'; echo $x "$x"
   a='1 2'; set -- $a; echo $#
   echo {1..3}{a,b}
   v='$HOME'; echo $v "$v" $(echo $v)
   n=5; echo {1..$n}
   IFS=:; p='a:b::c'; set -- $p; echo $#
   f() { echo "$#"; }; arr=('a b' c); f ${arr[@]}; f "${arr[@]}"; f "${arr[*]}"
   ```
2. Write a 100-line script with a subtle bug (your choice) and find it using `bash -x` with a custom `PS4` that shows file, line and function.
3. Send the trace to a separate file with `BASH_XTRACEFD` so stdout and stderr stay clean.
4. Make sure a secret (`API_TOKEN`) never appears in the trace, even with tracing on.
5. Enable tracing on demand with `DEBUG=1` without editing the script.

**Expected outcome:** A written explanation for all seven puzzles, a trace file where every line reads like `+ deploy.sh:42:upload(): curl ...`, and a grep showing the token is absent from the trace.

<details>
<summary>Hints</summary>

Brace expansion happens first, before parameter expansion, which is why `{1..$n}` does not work. Word splitting and globbing apply only to unquoted expansion results. Two adjacent non-whitespace IFS characters (like `::`) delimit an empty field. `PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}(): '`. `exec 7>>trace.log; BASH_XTRACEFD=7; set -x`. Wrap secret use in `{ set +x; } 2>/dev/null; ...; set -x`, and pass secrets via stdin or files rather than arguments.

</details>

**Bonus:** Measure where a slow script spends its time by adding `$EPOCHREALTIME` to `PS4` and sorting the gaps between lines.

### A22. Port a Bash script to POSIX `sh` 🔴

**Objective:** Know exactly which features are Bash-only by porting a real script to `dash` and BusyBox `ash`.

**Tasks**
1. Take a 100+ line Bash script that uses arrays, `[[ ]]`, `${var//a/b}`, `$(< file)`, `<(...)`, `local`, `read -a` and `printf -v` (your A14 or A18 script works).
2. Port it to POSIX `sh` with the shebang `#!/bin/sh`, preserving behaviour.
3. Check it with `shellcheck -s sh` and `checkbashisms` (from `devscripts`), reaching zero findings.
4. Test it under `dash` on Ubuntu and under BusyBox in `alpine:3.20`, with the same bats tests you run against the Bash version.
5. Write a table of every construct you replaced and its POSIX equivalent.

**Expected outcome:** The same test suite passes on Bash, `dash` and BusyBox `ash`; your table lists at least eight replacements.

<details>
<summary>Hints</summary>

Arrays become `set -- ...` positional parameters or newline-separated strings. `[[ $a == b* ]]` becomes `case $a in b*) ...;; esac`. `${var//a/b}` becomes `printf '%s' "$var" | sed 's/a/b/g'`. Process substitution becomes a temp file or a FIFO from `mkfifo`. `local` is not POSIX, though `dash` and BusyBox support it. Run tests per shell with `bats` by invoking the script as `"$SHELL_UNDER_TEST" script.sh`.

</details>

**Bonus:** Measure start-up and run time of the Bash and `dash` versions with `hyperfine` or a `time` loop, and explain the difference.

### A23. Security review and hardening of a hostile-input script 🔴

**Objective:** Find and fix injection, unsafe temp files, secret leakage and privilege problems in a script that processes untrusted input.

**Tasks**
1. Save this script, then list every vulnerability you can find, with a one-line exploit for each:
   ```bash
   #!/bin/bash
   # process uploaded file: ./process.sh "<filename>" "<user>"
   TOKEN=$(cat /etc/app/token)
   cd /srv/uploads
   eval "owner=$2"
   tmp=/tmp/process.$$
   cp $1 $tmp
   curl -s -H "Authorization: Bearer $TOKEN" "https://api.example.com/scan?file=$1" > /tmp/result
   if grep -q clean /tmp/result; then mv $tmp /srv/clean/$1; fi
   ssh backup "gzip /archive/$1"
   rm -rf /srv/uploads/$1
   ```
2. Rewrite it with strict mode, input validation (allow-list regex), no `eval`, `mktemp`, `--` before filenames, the token kept out of `argv` and out of traces, URL-encoded query parameters, and safe remote quoting.
3. Make it refuse to run as root and run the privileged step (moving into `/srv/clean`) through a narrowly scoped `sudo` rule.
4. Write bats tests that feed hostile filenames (`../../etc/passwd`, `$(id)`, `a;rm -rf ~`, `-rf`, a name with a newline) and assert nothing outside the upload directory is touched.
5. Get the rewrite to zero ShellCheck findings.

**Expected outcome:** At least eight vulnerabilities documented with exploits; a hardened script that passes all hostile-input tests; a sudoers rule that allows exactly one command.

<details>
<summary>Hints</summary>

Issues include: `eval` on user input, unquoted `$1` (splitting, globbing, option injection), predictable temp files (symlink attacks), a shared `/tmp/result` (race), the token visible in `ps` via `-H`, path traversal into `/srv/clean`, the remote shell re-parsing `$1`, and an unchecked `cd` before `rm -rf`. Use `curl --config <(printf 'header = "Authorization: Bearer %s"\n' "$TOKEN")`, `jq -rn --arg v "$name" '$v|@uri'` for URL encoding, and `printf %q` for remote arguments.

</details>

**Bonus:** Run the hardened script under `systemd-run --pty -p ProtectSystem=strict -p ReadWritePaths=/srv/uploads ...` and show that a remaining bug still cannot write outside its paths.

### A24. A resumable bulk job that survives `kill -9` 🔴

**Objective:** Process thousands of items reliably: checkpointing, idempotency, retries with backoff and jitter, rate limiting and atomic outputs.

**Tasks**
1. Generate 5,000 item IDs. Write `bulk.sh` that "processes" each item by calling a fake API script that randomly fails (20% of calls exit 1, 2% hang for 60 seconds) and writes a result file per item.
2. Retry failed calls up to 5 times with exponential backoff and random jitter; time out hung calls after 10 seconds.
3. Record completed IDs in a checkpoint file so a restart skips them, and write each result atomically so a killed run never leaves a partial result file.
4. Limit throughput to N calls per second and M in parallel.
5. Kill the job with `kill -9` at random points five times, restarting each time, and prove that every item ends up processed exactly once with a valid result.

**Expected outcome:** After the five kills and a final run, exactly 5,000 valid result files exist, the checkpoint has 5,000 unique IDs, and a summary reports retries, timeouts and permanent failures.

<details>
<summary>Hints</summary>

Backoff: `sleep $(( (2 ** attempt) + RANDOM % 3 ))`. Timeout: `timeout --kill-after=2 10 ./fake_api.sh "$id"`. Atomic result: write to `results/$id.json.tmp.$$`, validate (`jq -e .`), then `mv`. Checkpoint appends with `flock` around `echo "$id" >> done.txt` when running in parallel; on start, load it into an associative array. Order of operations matters: write the result, then checkpoint, and make processing idempotent so a crash between the two is harmless.

</details>

**Bonus:** Expose progress (done, failed, rate, ETA) to a Prometheus textfile collector file, rewritten atomically every 10 seconds.

### A25. When Bash has been outgrown: measure, then decide 🔴

**Objective:** Make an evidence-based decision about rewriting a script, using profiling rather than opinion.

**Tasks**
1. Generate a 2-million-line access log. Write a "naive" Bash report that loops over lines with `while read` and calls `cut`, `date` and `grep` per line to compute requests per minute and the top 10 IPs with 5xx responses.
2. Measure it with `time` and `strace -c -f` (count `clone` and `execve`). Stop it after 5 minutes if needed and extrapolate.
3. Write a builtins-only Bash version (parameter expansion, `[[ =~ ]]`, `printf '%(...)T'`, associative arrays) and measure again.
4. Write a single-`awk` version and a short Python version, and measure both.
5. Write a one-page decision memo: run times, fork counts, readability, testability, team skills and deployment constraints, ending with a recommendation and the criteria that would change it.

**Expected outcome:** A results table showing orders-of-magnitude differences between the naive loop and the `awk`/Python versions, and a memo that a senior engineer could approve or challenge on specific points.

<details>
<summary>Hints</summary>

The naive version forks several processes per line, so its cost is roughly lines times fork cost. `strace -c -f -o stats.txt ./report.sh` summarises syscalls. In `awk`, group with associative arrays: `{ split($4, t, ":"); minute[t[2] ":" t[3]]++ } $9 ~ /^5/ { ip[$1]++ }`. Typical rewrite triggers: complex data structures, JSON-heavy logic, API pagination and retries, concurrency, or more than a few hundred lines.

</details>

**Bonus:** Add the fastest version to a CI job with a performance budget that fails if run time on a fixed sample regresses by more than 20%.

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

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
