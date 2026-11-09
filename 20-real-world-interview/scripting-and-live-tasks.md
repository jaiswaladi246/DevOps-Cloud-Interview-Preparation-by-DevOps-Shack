# Real-World Interview — Scripting & Live Hands-On Tasks

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

> The live round: write the script, the Dockerfile, the manifest or the pipeline on a shared screen, or fix a broken container in 10 minutes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Real-World Interview](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Bash scripting tasks](#bash-scripting-tasks) — Q1–Q25
- [awk, sed, grep and jq one-liners](#awk-sed-grep-and-jq-one-liners) — Q26–Q50
- [Python for DevOps](#python-for-devops) — Q51–Q75
- [Write-a-Dockerfile tasks](#write-a-dockerfile-tasks) — Q76–Q95
- [Write-a-Kubernetes-manifest tasks](#write-a-kubernetes-manifest-tasks) — Q96–Q120
- [Write-a-pipeline tasks](#write-a-pipeline-tasks) — Q121–Q140
- [Write-Terraform tasks](#write-terraform-tasks) — Q141–Q160
- [Write-an-Ansible-playbook tasks](#write-an-ansible-playbook-tasks) — Q161–Q180
- [Live debugging tasks](#live-debugging-tasks) — Q181–Q200

> **How the live round works.** Interviewers watch *how* you work: do you start with a safe skeleton, test on a small input, quote your variables, and talk through the trick? Every answer below gives a working solution and a **Key trick →** line, which is the sentence you say out loud while typing.

---

## Bash scripting tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Write the safe header you start every Bash script with, and explain each part. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

`-e` exits on any failing command, `-u` errors on unset variables, `-o pipefail` makes a pipeline fail if any stage fails (not just the last). The narrowed `IFS` stops word-splitting on spaces in filenames.

**Key trick →** Without `pipefail`, `false | true` succeeds, so a broken `curl | jq` silently passes.

</details>

### Q2. Write a script that alerts when any filesystem is above 80% usage. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
THRESHOLD="${1:-80}"

df -P -x tmpfs -x devtmpfs | awk -v t="$THRESHOLD" 'NR>1 {
  gsub("%","",$5)
  if ($5+0 > t) printf "ALERT: %s mounted on %s is at %s%%\n", $1, $6, $5
}'
```

**Key trick →** `df -P` forces POSIX one-line-per-filesystem output, so long device names never wrap and break the column numbers.

</details>

### Q3. Parse an Nginx access log and print the top 10 client IPs by request count. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

**Key trick →** `uniq -c` only collapses adjacent lines, so you must `sort` before it, then `sort -rn` to rank by the count.

</details>

### Q4. Write a backup script that tars a directory with a timestamp and keeps only the last 7 backups. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
SRC="/var/www/app"
DEST="/backups"
KEEP=7
TS="$(date +%Y%m%d-%H%M%S)"

mkdir -p "$DEST"
tar -czf "$DEST/app-$TS.tar.gz" -C "$(dirname "$SRC")" "$(basename "$SRC")"

# rotate: delete everything except the newest $KEEP
ls -1t "$DEST"/app-*.tar.gz | tail -n +$((KEEP + 1)) | xargs -r rm -f --
echo "Backup done: app-$TS.tar.gz"
```

**Key trick →** `tail -n +8` prints from line 8 onwards (everything older than the newest 7), and `xargs -r` does nothing when the list is empty.

</details>

### Q5. Write a health-check loop that polls a URL until it returns 200, with a timeout. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
URL="${1:?usage: $0 <url> [timeout]}"
TIMEOUT="${2:-120}"
end=$((SECONDS + TIMEOUT))

until [ "$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$URL")" = "200" ]; do
  if (( SECONDS >= end )); then
    echo "Timed out after ${TIMEOUT}s waiting for $URL" >&2
    exit 1
  fi
  echo "Waiting for $URL ..."
  sleep 5
done
echo "$URL is healthy"
```

**Key trick →** `$SECONDS` is a built-in counter of seconds since the shell started, so no `date` arithmetic is needed; `--max-time` stops one hung request from blowing the timeout.

</details>

### Q6. Parse command-line flags `-e <env>`, `-v` (verbose) and `-h` in a Bash script. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
ENV=""; VERBOSE=0

usage() { echo "Usage: $0 -e <env> [-v]"; exit 1; }

while getopts ":e:vh" opt; do
  case "$opt" in
    e) ENV="$OPTARG" ;;
    v) VERBOSE=1 ;;
    h) usage ;;
    :) echo "Option -$OPTARG needs a value" >&2; usage ;;
    \?) echo "Unknown option -$OPTARG" >&2; usage ;;
  esac
done
shift $((OPTIND - 1))

[ -z "$ENV" ] && usage
(( VERBOSE )) && echo "Deploying to $ENV, remaining args: $*"
```

**Key trick →** The leading `:` in `":e:vh"` turns on silent error mode so you handle `:` (missing value) and `?` (unknown flag) yourself.

</details>

### Q7. Use `trap` to clean up a temp directory on exit, even on Ctrl+C or error. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
TMP="$(mktemp -d)"
cleanup() { rm -rf "$TMP"; echo "cleaned $TMP"; }
trap cleanup EXIT
trap 'echo "interrupted"; exit 130' INT TERM

curl -sfo "$TMP/release.tgz" https://example.com/release.tgz
tar -xzf "$TMP/release.tgz" -C "$TMP"
```

**Key trick →** Trapping `EXIT` covers normal exit, `set -e` failures and the explicit `exit 130` from the INT handler, so cleanup runs in all three cases.

</details>

### Q8. Write a script that checks if a list of services is running and restarts any that are not. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -uo pipefail
for svc in nginx docker sshd; do
  if ! systemctl is-active --quiet "$svc"; then
    echo "$(date -Is) $svc is down, restarting" | tee -a /var/log/svc-watch.log
    systemctl restart "$svc" || echo "FAILED to restart $svc" >&2
  fi
done
```

**Key trick →** `systemctl is-active --quiet` returns only an exit code, which is perfect for `if`. `-e` is left off deliberately so one failed restart doesn't stop the loop.

</details>

### Q9. Read a file of hostnames and run `uptime` on each over SSH in parallel, max 10 at a time. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
xargs -a hosts.txt -P 10 -I{} \
  ssh -o BatchMode=yes -o ConnectTimeout=5 {} 'echo "{}: $(uptime)"' 2>&1
```

Without xargs, using Bash job control:

```bash
while read -r host; do
  ssh -n -o BatchMode=yes -o ConnectTimeout=5 "$host" uptime | sed "s/^/$host: /" &
  while (( $(jobs -rp | wc -l) >= 10 )); do wait -n; done
done < hosts.txt
wait
```

**Key trick →** `ssh -n` (or BatchMode with `xargs`) stops ssh from swallowing the rest of stdin, which is why naive `while read` + ssh loops only process the first host.

</details>

### Q10. Find and delete log files older than 14 days in /var/log/app, printing what was deleted. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
find /var/log/app -type f -name '*.log' -mtime +14 -print -delete
```

**Key trick →** Put `-delete` last. If it comes before the filters, `find` deletes everything it walks past.

</details>

### Q11. Write a function that retries a command up to N times with exponential backoff. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
retry() {
  local max="$1"; shift
  local n=1 delay=2
  until "$@"; do
    if (( n >= max )); then
      echo "Command failed after $n attempts: $*" >&2
      return 1
    fi
    echo "Attempt $n failed, retrying in ${delay}s..." >&2
    sleep "$delay"
    n=$((n + 1)); delay=$((delay * 2))
  done
}

retry 5 curl -sf https://api.example.com/health
```

**Key trick →** `"$@"` passes the command and its arguments through intact, quoting included, so it works for any command.

</details>

### Q12. Check whether a script is run as root and exit with a clear message if not. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
if [[ $EUID -ne 0 ]]; then
  echo "This script must be run as root (try: sudo $0)" >&2
  exit 1
fi
```

**Key trick →** `$EUID` is the effective UID, so it is correct under `sudo`. `$USER` can be inherited and lie.

</details>

### Q13. Write a script that prevents two copies of itself from running at the same time (e.g. from cron). 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
exec 200>/var/lock/nightly-sync.lock
flock -n 200 || { echo "Another run is in progress, exiting"; exit 0; }

# ... actual work ...
rsync -a /data/ backup:/data/
```

**Key trick →** `flock` locks a file descriptor that the kernel releases when the process dies. A PID file can go stale after a crash, a flock can't.

</details>

### Q14. Count HTTP status codes from an access log and alert if 5xx is more than 5% of requests. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
LOG=/var/log/nginx/access.log

read -r total errors < <(awk '{t++; if ($9 ~ /^5/) e++} END {print t+0, e+0}' "$LOG")
pct=$(awk -v e="$errors" -v t="$total" 'BEGIN { if (t == 0) print 0; else printf "%.2f", e*100/t }')
echo "total=$total 5xx=$errors pct=$pct%"

if awk -v p="$pct" 'BEGIN { exit !(p > 5) }'; then
  echo "ALERT: 5xx rate $pct% above 5%" >&2
  exit 2
fi
```

**Key trick →** Bash only does integer maths, so float comparison goes through `awk`. `exit !(cond)` maps true to exit code 0 so it can be used in an `if`.

</details>

### Q15. Write a script to rotate a log file manually: compress the current one and truncate it without restarting the app. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
LOG=/var/log/app/app.log
TS=$(date +%F-%H%M)
cp "$LOG" "$LOG.$TS"
: > "$LOG"            # truncate in place, same inode
gzip "$LOG.$TS"
find /var/log/app -name 'app.log.*.gz' -mtime +30 -delete
```

**Key trick →** `mv` would leave the app writing to the old inode. Copy-then-truncate (logrotate's `copytruncate`) keeps the same file handle valid, at the cost of possibly losing a few lines written between the copy and the truncate.

</details>

### Q16. Validate that required environment variables are set before a deploy script runs. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
required=(AWS_REGION CLUSTER_NAME IMAGE_TAG)
missing=0
for v in "${required[@]}"; do
  if [[ -z "${!v:-}" ]]; then echo "Missing env var: $v" >&2; missing=1; fi
done
(( missing )) && exit 1
```

Shorter for one variable: `: "${IMAGE_TAG:?IMAGE_TAG must be set}"`.

**Key trick →** `${!v}` is indirect expansion, meaning "the value of the variable whose name is in `$v`". The `:-` stops `set -u` from aborting on it.

</details>

### Q17. Write a script that monitors a log file in real time and sends a Slack message on "ERROR". 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
WEBHOOK="${SLACK_WEBHOOK:?}"
tail -Fn0 /var/log/app/app.log | grep --line-buffered 'ERROR' | while read -r line; do
  payload=$(jq -n --arg t "$(hostname): $line" '{text: $t}')
  curl -s -X POST -H 'Content-Type: application/json' -d "$payload" "$WEBHOOK" >/dev/null
done
```

**Key trick →** `tail -F` follows the file across rotation (by name, not inode), and `--line-buffered` stops grep from holding output in a 4 KB buffer. `jq -n --arg` escapes quotes in the log line safely.

</details>

### Q18. Write a script that compares two directories and lists files present in one but not the other. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
diff <(cd dirA && find . -type f | sort) <(cd dirB && find . -type f | sort)
# or only names unique to dirA:
comm -23 <(cd dirA && find . -type f | sort) <(cd dirB && find . -type f | sort)
```

**Key trick →** Process substitution `<(...)` turns command output into a file path, and `comm` needs both inputs sorted.

</details>

### Q19. Write a Bash script that deploys to a list of servers one by one and stops on the first failure (a rolling deploy). 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
VERSION="${1:?version required}"
mapfile -t HOSTS < hosts.txt

for h in "${HOSTS[@]}"; do
  echo "==> $h: draining"
  ssh -n "$h" "sudo /opt/lb/deregister.sh"
  echo "==> $h: deploying $VERSION"
  ssh -n "$h" "sudo /opt/app/deploy.sh $VERSION"
  for i in {1..30}; do
    if ssh -n "$h" "curl -sf localhost:8080/health" >/dev/null; then break; fi
    if (( i == 30 )); then echo "$h failed health check, aborting rollout" >&2; exit 1; fi
    sleep 2
  done
  ssh -n "$h" "sudo /opt/lb/register.sh"
  echo "==> $h: OK"
done
```

**Key trick →** `set -e` plus an explicit health gate per host means a bad version is stopped after one server, never all of them. `mapfile` reads the host list safely into an array.

</details>

### Q20. What is the difference between `$*` and `"$@"`? Show it in a script. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
show() { for a in "$@"; do echo "[$a]"; done; }
show "a b" c      # [a b] [c]
f() { show $*; }; f "a b" c   # [a] [b] [c]
```

`"$@"` expands to each argument as a separate quoted word. Unquoted `$*` (or `$@`) re-splits on spaces.

**Key trick →** Always forward arguments with `"$@"`.

</details>

### Q21. Write a script that checks SSL certificate expiry for a list of domains and warns under 30 days. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
WARN_DAYS=30
while read -r domain; do
  end=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
        | openssl x509 -noout -enddate | cut -d= -f2)
  end_epoch=$(date -d "$end" +%s)
  days=$(( (end_epoch - $(date +%s)) / 86400 ))
  if (( days < WARN_DAYS )); then
    echo "WARN: $domain expires in $days days ($end)"
  else
    echo "OK:   $domain ($days days)"
  fi
done < domains.txt
```

**Key trick →** `-servername` sends SNI. Without it, a shared host returns its default certificate and you check the wrong expiry. (`openssl x509 -checkend 2592000` is a shortcut that gives a yes/no answer.)

</details>

### Q22. Write a script that reads a CSV of `username,group` and creates the users idempotently. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
while IFS=, read -r user group; do
  [[ -z "$user" || "$user" == "username" ]] && continue   # skip blanks/header
  getent group "$group" >/dev/null || groupadd "$group"
  if id "$user" &>/dev/null; then
    usermod -aG "$group" "$user"
    echo "updated $user"
  else
    useradd -m -s /bin/bash -G "$group" "$user"
    echo "created $user"
  fi
done < users.csv
```

**Key trick →** Check first with `id`/`getent`, then act. That makes the script safe to run again, which is the same idea Ansible modules are built on.

</details>

### Q23. Write a one-file script that measures how long each step of a deploy takes. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
step() {
  local name="$1"; shift
  local start=$SECONDS
  echo "--> $name"
  "$@"
  echo "<-- $name took $((SECONDS - start))s"
}

step "pull image"   docker pull myapp:1.4.2
step "run migrate"  docker run --rm myapp:1.4.2 ./migrate
step "restart"      systemctl restart myapp
```

**Key trick →** Wrapping commands in a function that takes `"$@"` gives you timing, logging and error handling in one place.

</details>

### Q24. Write a script that finds the 10 largest files under / without crossing into other filesystems. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
find / -xdev -type f -size +100M -printf '%s %p\n' 2>/dev/null | sort -rn | head -10 \
  | awk '{printf "%.1f MB %s\n", $1/1048576, $2}'
```

**Key trick →** `-xdev` keeps `find` off /proc, NFS mounts and other volumes. `-printf '%s'` gives bytes for exact sorting.

</details>

### Q25. Write a script that waits for a TCP port (e.g. a database) to become reachable before starting the app. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail
HOST="${DB_HOST:-db}"; PORT="${DB_PORT:-5432}"; TIMEOUT=60

for ((i = 0; i < TIMEOUT; i++)); do
  if (echo > "/dev/tcp/$HOST/$PORT") 2>/dev/null; then
    echo "$HOST:$PORT is up"
    exec "$@"
  fi
  sleep 1
done
echo "Gave up waiting for $HOST:$PORT" >&2
exit 1
```

Use it as `ENTRYPOINT ["/wait-for.sh", "python", "app.py"]`.

**Key trick →** `/dev/tcp/host/port` is a Bash built-in pseudo-device, so no `nc` is needed in the image. `exec "$@"` replaces the shell so the app becomes PID 1 and receives signals.

</details>

## awk, sed, grep and jq one-liners

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Print only the lines containing "ERROR" with 3 lines of context, and show line numbers. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -n -C 3 'ERROR' app.log
```

**Key trick →** `-C` gives context both ways (`-A` after, `-B` before) and `-n` adds line numbers so you can jump to the spot in `less +<n>`.

</details>

### Q27. Sum the response sizes (field 10) in an Nginx access log and print the total in MB. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
awk '$10 ~ /^[0-9]+$/ {s += $10} END {printf "%.2f MB\n", s/1024/1024}' access.log
```

**Key trick →** The regex guard skips `-` values (no body), which would otherwise be counted as 0 silently or break other tools.

</details>

### Q28. Replace `DEBUG` with `INFO` in a config file in place, keeping a backup. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sed -i.bak 's/^LOG_LEVEL=DEBUG$/LOG_LEVEL=INFO/' app.conf
```

**Key trick →** Anchor with `^` and `$` so you only change the exact setting line. `-i.bak` writes `app.conf.bak` first (on macOS `-i` always requires an argument).

</details>

### Q29. Extract all unique email addresses from a file. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -Eoi '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}' file.txt | sort -fu
```

**Key trick →** `-o` prints only the matched part, not the whole line. `sort -fu` de-duplicates without caring about case.

</details>

### Q30. From `kubectl get pods -A -o json`, list pods that are not Running, as namespace/name/phase. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o json | jq -r '
  .items[] | select(.status.phase != "Running")
  | "\(.metadata.namespace)/\(.metadata.name)\t\(.status.phase)"'
```

**Key trick →** `select()` filters and `\( )` is jq string interpolation. `-r` prints raw strings without JSON quotes. (Quick alternative: `kubectl get pods -A --field-selector=status.phase!=Running`.)

</details>

### Q31. Print the 3rd column of a CSV whose fields are separated by commas, skipping the header. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
awk -F, 'NR > 1 {print $3}' data.csv
```

**Key trick →** `NR` is the overall line number. Note that this naive split breaks on quoted fields containing commas; for real CSV use `python -c 'import csv…'` or `mlr`.

</details>

### Q32. Delete all blank lines and comment lines from a config file and show what's actually active. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
# or with sed
sed -e '/^\s*#/d' -e '/^\s*$/d' /etc/ssh/sshd_config
```

**Key trick →** One regex alternation `(#|$)` catches both comment lines and empty lines, including lines that contain only spaces.

</details>

### Q33. Find which process is listening on port 8080 in one line. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo ss -ltnp 'sport = :8080'
# or
sudo lsof -iTCP:8080 -sTCP:LISTEN -n -P
```

**Key trick →** `ss` filters natively, so there's no need to grep. `-p` needs root to show processes owned by other users.

</details>

### Q34. Using jq, get the list of EC2 instance IDs and private IPs that are in "running" state. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  | jq -r '.Reservations[].Instances[] | [.InstanceId, .PrivateIpAddress] | @tsv'
```

Or with the built-in JMESPath: `--query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]' --output text`.

**Key trick →** Instances are nested under `Reservations`, so you need `.Reservations[].Instances[]`. `@tsv` gives clean columns.

</details>

### Q35. Count requests per hour from an access log with timestamps like `[09/Nov/2026:14:23:01 +0000]`. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
awk -F'[][]' '{split($2, t, ":"); print t[1]":"t[2]}' access.log | sort | uniq -c
```

**Key trick →** Using `[` and `]` as field separators isolates the timestamp. Splitting on `:` then gives the date and the hour.

</details>

### Q36. Print lines between two markers, e.g. from "BEGIN CONFIG" to "END CONFIG". 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sed -n '/BEGIN CONFIG/,/END CONFIG/p' file.txt
# exclude the markers themselves
awk '/END CONFIG/{f=0} f; /BEGIN CONFIG/{f=1}' file.txt
```

**Key trick →** sed address ranges `/a/,/b/` are inclusive. The awk flag pattern, with the order of the three rules, controls whether the markers are printed.

</details>

### Q37. Update a value inside a JSON file with jq (set `.replicas` to 5) and write it back. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
jq '.replicas = 5' config.json > config.json.tmp && mv config.json.tmp config.json
# with a shell variable
jq --argjson n "$REPLICAS" '.replicas = $n' config.json > tmp && mv tmp config.json
```

**Key trick →** `jq ... > config.json` truncates the file before jq reads it, so always go through a temp file. Use `--argjson` for numbers and `--arg` for strings.

</details>

### Q38. Show the top 5 memory-consuming processes in one line. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -6
```

**Key trick →** `--sort=-%mem` sorts descending inside `ps`, and `head -6` allows for the header row.

</details>

### Q39. Find the slowest 10 requests in a log where the last field is response time in seconds. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
awk '{print $NF, $7}' access.log | sort -k1,1 -gr | head -10
```

**Key trick →** `$NF` is the last field, whatever the number of columns. `sort -g` handles decimals and scientific notation; `-n` is fine for plain decimals.

</details>

### Q40. Using sed, insert a line after a matching line, e.g. add `include /etc/nginx/extra.conf;` after `http {`. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sed -i '/^http {/a\    include /etc/nginx/extra.conf;' /etc/nginx/nginx.conf
```

**Key trick →** `a\` appends after the match and `i\` inserts before it. Run without `-i` first to preview. This is not idempotent, so in automation use Ansible `lineinfile` instead.

</details>

### Q41. Convert a JSON array of objects into CSV with jq. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
jq -r '(.[0] | keys_unsorted) as $k | ($k | @csv), (.[] | [.[$k[]]] | @csv)' users.json
```

**Key trick →** Take the header from the first object's keys, then pull values in that same key order for every row. `@csv` handles quoting.

</details>

### Q42. Grep recursively for a string in all `.yaml` files, ignoring the `.git` and `node_modules` folders. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -rn --include='*.yaml' --exclude-dir={.git,node_modules} 'imagePullPolicy' .
```

**Key trick →** `--include` and `--exclude-dir` are far faster than piping `find` output through grep. `rg` (ripgrep) does this by default and respects `.gitignore`.

</details>

### Q43. Print the average, min and max of a numeric column with awk. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
awk 'NR==1 {min=max=$1} {s+=$1; if ($1<min) min=$1; if ($1>max) max=$1}
     END {printf "avg=%.2f min=%s max=%s n=%d\n", s/NR, min, max, NR}' latencies.txt
```

**Key trick →** Seed `min` and `max` from the first row. Starting at 0 gives a wrong min for all-positive data.

</details>

### Q44. Calculate the p95 latency from a file of numbers using only shell tools. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sort -n latencies.txt | awk '{a[NR]=$1} END {i=int(NR*0.95); if (i<1) i=1; print "p95=" a[i]}'
```

**Key trick →** Percentiles need sorted data. Index at `NR*0.95` (nearest-rank method). The `if` keeps tiny datasets from indexing 0.

</details>

### Q45. Using jq, group a list of pods by node and count them. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o json | jq -r '
  [.items[] | {node: (.spec.nodeName // "unscheduled")}]
  | group_by(.node) | map({node: .[0].node, count: length})
  | sort_by(-.count)[] | "\(.count)\t\(.node)"'
```

**Key trick →** `group_by` needs a key and returns arrays. `//` is jq's "alternative" operator for null values (Pending pods have no `nodeName`).

</details>

### Q46. Remove ANSI colour codes from a CI log. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sed -E 's/\x1B\[[0-9;]*[A-Za-z]//g' build.log > build.clean.log
```

**Key trick →** ANSI codes are `ESC [ params letter`. GNU sed understands `\x1B` for the escape byte.

</details>

### Q47. Find all Kubernetes YAML files that set `privileged: true`, and print file and line. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
grep -rnE --include='*.y*ml' '^\s*privileged:\s*true' manifests/
```

**Key trick →** `*.y*ml` matches both `.yaml` and `.yml`. Anchoring with optional leading whitespace avoids matching comments that mention the word.

</details>

### Q48. Extract the value of a key from a YAML file in a shell script without Python. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
yq '.spec.template.spec.containers[0].image' deployment.yaml
# set a value in place (mikefarah yq v4)
yq -i '.spec.replicas = 3' deployment.yaml
```

**Key trick →** Use `yq` (the mikefarah Go binary), not `grep | awk`: YAML has nesting, multi-doc files and quoting that line tools get wrong.

</details>

### Q49. Join two files on a common key column (like a SQL join) using shell tools. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
join -t, -1 1 -2 1 <(sort -t, -k1,1 users.csv) <(sort -t, -k1,1 orders.csv)
# awk version, no sorting required:
awk -F, 'NR==FNR {name[$1]=$2; next} ($1 in name) {print $0 "," name[$1]}' users.csv orders.csv
```

**Key trick →** `NR==FNR` is true only while awk reads the first file, so you load a lookup map first and then stream the second file against it.

</details>

### Q50. Find duplicate lines in a file and show how many times each appears. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sort file.txt | uniq -cd | sort -rn
# keep first occurrence, preserve original order:
awk '!seen[$0]++' file.txt
```

**Key trick →** `uniq -d` shows only duplicated lines. `awk '!seen[$0]++'` is the classic order-preserving de-duplicate.

</details>

## Python for DevOps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Write a Python script using boto3 that lists all running EC2 instances with their Name tag. 🟢

<details>
<summary><b>Answer</b></summary>

```python
import boto3

ec2 = boto3.client("ec2", region_name="ap-south-1")
paginator = ec2.get_paginator("describe_instances")
filters = [{"Name": "instance-state-name", "Values": ["running"]}]

for page in paginator.paginate(Filters=filters):
    for res in page["Reservations"]:
        for inst in res["Instances"]:
            tags = {t["Key"]: t["Value"] for t in inst.get("Tags", [])}
            print(inst["InstanceId"], inst["InstanceType"], tags.get("Name", "-"))
```

**Key trick →** Always use a paginator. A plain `describe_instances` returns only the first page in large accounts. Use `inst.get("Tags", [])` because untagged instances have no `Tags` key at all.

</details>

### Q52. Write a script that finds unattached EBS volumes and deletes them (with a dry-run flag). 🟡

<details>
<summary><b>Answer</b></summary>

```python
import argparse
import boto3

parser = argparse.ArgumentParser()
parser.add_argument("--apply", action="store_true", help="actually delete")
args = parser.parse_args()

ec2 = boto3.client("ec2")
pages = ec2.get_paginator("describe_volumes").paginate(
    Filters=[{"Name": "status", "Values": ["available"]}]
)
for page in pages:
    for v in page["Volumes"]:
        print(f"{v['VolumeId']} {v['Size']}GiB created {v['CreateTime']:%Y-%m-%d}")
        if args.apply:
            ec2.delete_volume(VolumeId=v["VolumeId"])
            print("  deleted")
```

**Key trick →** `status=available` means not attached. Default to dry-run and require `--apply`; interviewers check whether you make destructive scripts safe by default.

</details>

### Q53. Call a REST API with `requests`, handle timeouts and non-2xx responses properly. 🟢

<details>
<summary><b>Answer</b></summary>

```python
import requests

try:
    r = requests.get(
        "https://api.github.com/repos/kubernetes/kubernetes",
        headers={"Accept": "application/vnd.github+json"},
        timeout=(3.05, 10),  # connect, read
    )
    r.raise_for_status()
    print(r.json()["stargazers_count"])
except requests.exceptions.Timeout:
    print("Request timed out")
except requests.exceptions.HTTPError as e:
    print(f"HTTP error: {e.response.status_code} {e.response.text[:200]}")
```

**Key trick →** `requests` has **no default timeout**, so a call without one can hang forever. `raise_for_status()` turns 4xx/5xx into exceptions.

</details>

### Q54. Add retry with exponential backoff to HTTP calls without writing the loop yourself. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(
    total=5,
    backoff_factor=1,                  # sleeps grow exponentially between attempts
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "PUT", "DELETE"],
    respect_retry_after_header=True,
)
s = requests.Session()
s.mount("https://", HTTPAdapter(max_retries=retry))
resp = s.get("https://api.example.com/items", timeout=5)
```

**Key trick →** `POST` is left out of `allowed_methods` on purpose, because retrying a non-idempotent call can create duplicates. `Retry-After` is honoured for 429s.

</details>

### Q55. Write your own retry decorator with exponential backoff and jitter. 🔴

<details>
<summary><b>Answer</b></summary>

```python
import functools
import random
import time

def retry(times=5, base=0.5, exceptions=(Exception,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as e:
                    if attempt == times:
                        raise
                    sleep = base * 2 ** (attempt - 1) + random.uniform(0, base)
                    print(f"{fn.__name__} failed ({e}); retry {attempt} in {sleep:.1f}s")
                    time.sleep(sleep)
        return wrapper
    return deco

@retry(times=4, exceptions=(ConnectionError, TimeoutError))
def fetch():
    ...
```

**Key trick →** Jitter stops many clients from retrying in lockstep and hammering a recovering service. Only retry specific, transient exceptions.

</details>

### Q56. Parse a YAML file and print every container image in a multi-document Kubernetes manifest. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import sys
import yaml

with open(sys.argv[1]) as f:
    for doc in yaml.safe_load_all(f):
        if not doc:
            continue
        spec = doc.get("spec", {})
        pod = spec.get("template", {}).get("spec", spec)   # Deployment or bare Pod
        for c in pod.get("containers", []) + pod.get("initContainers", []):
            print(f"{doc['kind']}/{doc['metadata']['name']}: {c['image']}")
```

**Key trick →** Use `safe_load_all` for files separated by `---`, and `safe_load` never `load`, because plain `yaml.load` without a safe Loader can construct arbitrary Python objects.

</details>

### Q57. Run a shell command from Python, capture output, and fail properly on error. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import subprocess

result = subprocess.run(
    ["kubectl", "get", "nodes", "-o", "name"],
    capture_output=True, text=True, check=True, timeout=30,
)
nodes = result.stdout.split()
print(f"{len(nodes)} nodes")
```

On failure, `check=True` raises `CalledProcessError` with `.returncode` and `.stderr`.

**Key trick →** Pass a list and avoid `shell=True`. Then no shell parses your input, which removes command injection when arguments come from users.

</details>

### Q58. Write a script that reads a JSON file of services and checks each one's health endpoint concurrently. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import json
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

with open("services.json") as f:
    services = json.load(f)   # [{"name": "api", "url": "https://.../health"}, ...]

def check(svc):
    try:
        r = requests.get(svc["url"], timeout=5)
        return svc["name"], r.status_code, r.elapsed.total_seconds()
    except requests.RequestException as e:
        return svc["name"], "DOWN", str(e)[:60]

with ThreadPoolExecutor(max_workers=20) as pool:
    for fut in as_completed(pool.submit(check, s) for s in services):
        print(*fut.result())
```

**Key trick →** Health checks are I/O-bound, so threads give a big speed-up despite the GIL. `as_completed` prints results as they arrive instead of in submit order.

</details>

### Q59. Write a boto3 script that finds S3 buckets without default encryption or with public access not blocked. 🔴

<details>
<summary><b>Answer</b></summary>

```python
import boto3
from botocore.exceptions import ClientError

s3 = boto3.client("s3")
for b in s3.list_buckets()["Buckets"]:
    name = b["Name"]
    issues = []
    try:
        pab = s3.get_public_access_block(Bucket=name)["PublicAccessBlockConfiguration"]
        if not all(pab.values()):
            issues.append("public access block partially off")
    except ClientError as e:
        if e.response["Error"]["Code"] == "NoSuchPublicAccessBlockConfiguration":
            issues.append("no public access block")
        else:
            raise
    try:
        s3.get_bucket_encryption(Bucket=name)
    except ClientError as e:
        if e.response["Error"]["Code"] == "ServerSideEncryptionConfigurationNotFoundError":
            issues.append("no default encryption")
    print(f"{name}: {', '.join(issues) or 'OK'}")
```

**Key trick →** boto3 reports "not configured" as a `ClientError` with a specific code, so branch on `e.response["Error"]["Code"]` instead of catching everything. (Since January 2023 new buckets get SSE-S3 by default, so the encryption check mostly flags old buckets.)

</details>

### Q60. Write a script that tails a log file and counts error types every 60 seconds. 🔴

<details>
<summary><b>Answer</b></summary>

```python
import re
import time
from collections import Counter

PATTERN = re.compile(r"ERROR\s+\[(?P<kind>[\w.]+)\]")

def follow(path):
    with open(path) as f:
        f.seek(0, 2)                      # start at end of file
        while True:
            line = f.readline()
            if not line:
                time.sleep(0.5)
                yield None
                continue
            yield line

counts, window_start = Counter(), time.time()
for line in follow("/var/log/app/app.log"):
    if line and (m := PATTERN.search(line)):
        counts[m["kind"]] += 1
    if time.time() - window_start >= 60:
        print(time.strftime("%H:%M"), dict(counts.most_common(5)))
        counts.clear()
        window_start = time.time()
```

**Key trick →** The generator yields `None` when idle so the 60-second window still flushes during quiet periods. Compile the regex once outside the loop.

</details>

### Q61. Parse an environment-specific config: load base YAML and override with environment values. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import os
import yaml

def deep_merge(base, override):
    for k, v in override.items():
        if isinstance(v, dict) and isinstance(base.get(k), dict):
            deep_merge(base[k], v)
        else:
            base[k] = v
    return base

env = os.environ.get("APP_ENV", "dev")
with open("config/base.yaml") as f:
    cfg = yaml.safe_load(f)
with open(f"config/{env}.yaml") as f:
    cfg = deep_merge(cfg, yaml.safe_load(f) or {})
cfg["db"]["password"] = os.environ["DB_PASSWORD"]   # secrets from env, not files
print(cfg)
```

**Key trick →** A shallow `dict.update` replaces whole nested sections. A recursive merge overrides only the keys that differ.

</details>

### Q62. Write a CLI tool with argparse that has subcommands `deploy` and `rollback`. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import argparse

def deploy(a):   print(f"deploying {a.version} to {a.env}")
def rollback(a): print(f"rolling back {a.env}")

p = argparse.ArgumentParser(prog="shipit")
sub = p.add_subparsers(dest="cmd", required=True)

d = sub.add_parser("deploy");   d.add_argument("version"); d.set_defaults(func=deploy)
r = sub.add_parser("rollback");                            r.set_defaults(func=rollback)
for sp in (d, r):
    sp.add_argument("--env", choices=["dev", "staging", "prod"], required=True)

args = p.parse_args()
args.func(args)
```

**Key trick →** `set_defaults(func=...)` sends each subcommand to its handler with no if/elif chain, and `choices=` validates input for free.

</details>

### Q63. Use boto3 to rotate through all regions and count running instances in each. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import boto3

regions = [r["RegionName"] for r in boto3.client("ec2", region_name="us-east-1").describe_regions()["Regions"]]
for region in regions:
    ec2 = boto3.client("ec2", region_name=region)
    n = sum(
        len(res["Instances"])
        for page in ec2.get_paginator("describe_instances").paginate(
            Filters=[{"Name": "instance-state-name", "Values": ["running"]}])
        for res in page["Reservations"]
    )
    if n:
        print(f"{region}: {n}")
```

**Key trick →** `describe_regions` returns only regions enabled for the account (opt-in regions excluded unless enabled), so you won't get auth errors from disabled regions.

</details>

### Q64. Write a script that sends a Slack alert when a Kubernetes deployment has unavailable replicas, using the official client. 🔴

<details>
<summary><b>Answer</b></summary>

```python
import os
import requests
from kubernetes import client, config

try:
    config.load_incluster_config()       # running inside a pod
except config.ConfigException:
    config.load_kube_config()            # local ~/.kube/config

apps = client.AppsV1Api()
problems = []
for d in apps.list_deployment_for_all_namespaces().items:
    want = d.spec.replicas or 0
    ready = d.status.available_replicas or 0
    if ready < want:
        problems.append(f"{d.metadata.namespace}/{d.metadata.name}: {ready}/{want}")

if problems:
    requests.post(os.environ["SLACK_WEBHOOK"],
                  json={"text": "Degraded deployments:\n" + "\n".join(problems)},
                  timeout=5)
```

**Key trick →** Try in-cluster config first, then fall back to kubeconfig, so the same script runs as a CronJob and on your laptop. Status fields are `None` (not 0) when no replica is available.

</details>

### Q65. Compare two JSON files and print keys that differ. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import json
import sys

def diff(a, b, path=""):
    for k in sorted(set(a) | set(b)):
        p = f"{path}.{k}" if path else k
        if k not in a:   print(f"+ {p} = {b[k]!r}")
        elif k not in b: print(f"- {p} = {a[k]!r}")
        elif isinstance(a[k], dict) and isinstance(b[k], dict):
            diff(a[k], b[k], p)
        elif a[k] != b[k]:
            print(f"~ {p}: {a[k]!r} -> {b[k]!r}")

with open(sys.argv[1]) as f1, open(sys.argv[2]) as f2:
    diff(json.load(f1), json.load(f2))
```

**Key trick →** Use the union of key sets to find additions and removals, and recurse only when both sides are dicts.

</details>

### Q66. Read a large log file (10 GB) and count lines matching a pattern without running out of memory. 🟢

<details>
<summary><b>Answer</b></summary>

```python
count = 0
with open("/var/log/huge.log", "r", errors="replace") as f:
    for line in f:              # streams line by line
        if " 500 " in line:
            count += 1
print(count)
```

**Key trick →** Iterating the file object streams it. `f.read()` or `readlines()` would load all 10 GB into memory. `errors="replace"` survives stray non-UTF-8 bytes.

</details>

### Q67. Write a script that creates an S3 bucket only if it doesn't exist, in a non-us-east-1 region. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import boto3
from botocore.exceptions import ClientError

region, bucket = "ap-south-1", "devopsshack-artifacts-123456"
s3 = boto3.client("s3", region_name=region)
try:
    s3.head_bucket(Bucket=bucket)
    print("exists")
except ClientError as e:
    if e.response["Error"]["Code"] in ("404", "NoSuchBucket"):
        s3.create_bucket(
            Bucket=bucket,
            CreateBucketConfiguration={"LocationConstraint": region},
        )
        print("created")
    else:
        raise   # 403 = exists but owned by someone else
```

**Key trick →** Outside us-east-1 you must pass `LocationConstraint`, or the call fails with `IllegalLocationConstraintException`. A 403 from `head_bucket` means the name is taken by someone else.

</details>

### Q68. Write a function that parses `key=value` lines (like a .env file) into a dict, ignoring comments. 🟢

<details>
<summary><b>Answer</b></summary>

```python
def parse_env(path):
    env = {}
    with open(path) as f:
        for raw in f:
            line = raw.strip()
            if not line or line.startswith("#") or "=" not in line:
                continue
            key, _, value = line.partition("=")
            env[key.strip()] = value.strip().strip('"').strip("'")
    return env
```

**Key trick →** `partition("=")` splits only on the first `=`, so values like `DB_URL=postgres://u:p@h/db?sslmode=require` stay intact.

</details>

### Q69. Launch an EC2 instance with boto3 and wait until it passes status checks, using a waiter. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import boto3

ec2 = boto3.client("ec2")
resp = ec2.run_instances(ImageId="ami-0abcdef1234567890", InstanceType="t3.micro",
                         MinCount=1, MaxCount=1)
iid = resp["Instances"][0]["InstanceId"]

ec2.get_waiter("instance_status_ok").wait(
    InstanceIds=[iid], WaiterConfig={"Delay": 15, "MaxAttempts": 40})
print(f"{iid} passed status checks")
```

**Key trick →** Waiters handle polling and timeouts for you. `instance_running` only means the state is running; `instance_status_ok` waits for the system and instance status checks to pass.

</details>

### Q70. Write a Python script that cleans up ECR images older than 30 days, keeping the 10 newest. 🔴

<details>
<summary><b>Answer</b></summary>

```python
from datetime import datetime, timedelta, timezone
import boto3

ecr = boto3.client("ecr")
repo, keep, cutoff = "myapp", 10, datetime.now(timezone.utc) - timedelta(days=30)

images = []
for page in ecr.get_paginator("describe_images").paginate(repositoryName=repo):
    images.extend(page["imageDetails"])
images.sort(key=lambda i: i["imagePushedAt"], reverse=True)

doomed = [{"imageDigest": i["imageDigest"]}
          for i in images[keep:] if i["imagePushedAt"] < cutoff]
for chunk in (doomed[i:i + 100] for i in range(0, len(doomed), 100)):
    ecr.batch_delete_image(repositoryName=repo, imageIds=chunk)
print(f"deleted {len(doomed)} images")
```

**Key trick →** `batch_delete_image` accepts at most 100 IDs per call, hence the chunking. Say in the interview that an ECR lifecycle policy does this natively; the script is for custom rules such as "never delete what's deployed".

</details>

### Q71. Write a script that validates all YAML files in a repo and exits non-zero on any error (for CI). 🟡

<details>
<summary><b>Answer</b></summary>

```python
import pathlib
import sys
import yaml

bad = 0
for path in pathlib.Path(".").rglob("*.y*ml"):
    if ".git" in path.parts:
        continue
    try:
        list(yaml.safe_load_all(path.read_text()))
    except yaml.YAMLError as e:
        print(f"{path}: {e}")
        bad += 1
sys.exit(1 if bad else 0)
```

**Key trick →** `safe_load_all` returns a lazy generator, so wrap it in `list()` to force every document to be parsed. The exit code is what fails the CI step.

</details>

### Q72. Use Python to get a secret from AWS Secrets Manager and cache it. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import functools
import json
import boto3

sm = boto3.client("secretsmanager")

@functools.lru_cache(maxsize=32)
def get_secret(name: str) -> dict:
    value = sm.get_secret_value(SecretId=name)["SecretString"]
    return json.loads(value)

db = get_secret("prod/app/db")
print(db["username"])
```

**Key trick →** Caching avoids per-request API calls (cost and throttling). In production add a TTL (or use AWS's caching client) so rotated secrets are picked up.

</details>

### Q73. Write a script that finds IAM access keys older than 90 days. 🟡

<details>
<summary><b>Answer</b></summary>

```python
from datetime import datetime, timezone
import boto3

iam = boto3.client("iam")
now = datetime.now(timezone.utc)
for page in iam.get_paginator("list_users").paginate():
    for u in page["Users"]:
        for k in iam.list_access_keys(UserName=u["UserName"])["AccessKeyMetadata"]:
            age = (now - k["CreateDate"]).days
            if age > 90 and k["Status"] == "Active":
                print(f"{u['UserName']} {k['AccessKeyId']} {age} days")
```

**Key trick →** boto3 returns timezone-aware datetimes, so `now` must be aware too, or the subtraction raises `TypeError`. At scale, `generate_credential_report` is one call instead of N.

</details>

### Q74. Write a minimal Prometheus exporter in Python that exposes a custom metric. 🔴

<details>
<summary><b>Answer</b></summary>

```python
import time
import requests
from prometheus_client import Gauge, start_http_server

QUEUE_DEPTH = Gauge("app_queue_depth", "Messages waiting in queue", ["queue"])

def collect():
    data = requests.get("http://localhost:15672/api/queues",
                        auth=("guest", "guest"), timeout=5).json()
    for q in data:
        QUEUE_DEPTH.labels(queue=q["name"]).set(q.get("messages", 0))

if __name__ == "__main__":
    start_http_server(9100)          # serves /metrics
    while True:
        collect()
        time.sleep(15)
```

**Key trick →** Use a Gauge for values that go up and down and a Counter only for monotonic totals. Keep label values bounded (queue names, not message IDs) to avoid a cardinality explosion.

</details>

### Q75. Write a script that sorts EC2 instances by monthly cost estimate using a price map. 🟡

<details>
<summary><b>Answer</b></summary>

```python
import boto3
from collections import defaultdict

HOURLY = {"t3.micro": 0.0104, "t3.medium": 0.0416, "m5.large": 0.096}  # sample on-demand prices
ec2 = boto3.client("ec2")
cost = defaultdict(float)

for page in ec2.get_paginator("describe_instances").paginate(
        Filters=[{"Name": "instance-state-name", "Values": ["running"]}]):
    for r in page["Reservations"]:
        for i in r["Instances"]:
            team = next((t["Value"] for t in i.get("Tags", []) if t["Key"] == "team"), "untagged")
            cost[team] += HOURLY.get(i["InstanceType"], 0) * 730

for team, c in sorted(cost.items(), key=lambda kv: kv[1], reverse=True):
    print(f"{team:15} ${c:,.2f}/month")
```

**Key trick →** 730 is the average number of hours in a month. Grouping by a `team` tag shows who owns the spend. For real numbers, use Cost Explorer (`ce.get_cost_and_usage`) grouped by tag.

</details>

## Write-a-Dockerfile tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Write a production Dockerfile for a Node.js API using a multi-stage build. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps  --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Key trick →** Copy only the lockfiles before `npm ci` so the dependency layer stays cached until dependencies change. The final stage ships prod dependencies only and runs as the built-in `node` user.

</details>

### Q77. Write a Dockerfile for a Python Flask app that runs with gunicorn as a non-root user. 🟢

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd --system --uid 10001 appuser
USER 10001
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "3", "app:app"]
```

**Key trick →** `PYTHONUNBUFFERED=1` makes logs appear in `docker logs` immediately, and binding to `0.0.0.0` (not 127.0.0.1) makes the port reachable from outside the container.

</details>

### Q78. Write a Dockerfile for a Go service that produces a tiny image (under 20 MB). 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

**Key trick →** `CGO_ENABLED=0` builds a static binary that needs no libc, so it runs on distroless/static (or `scratch`). `-s -w` strips debug symbols.

</details>

### Q79. Write a multi-stage Dockerfile for a Java Spring Boot app built with Maven. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -q package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 10001 spring
COPY --from=build /src/target/*.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

**Key trick →** Caching `dependency:go-offline` on `pom.xml` alone avoids re-downloading the internet on every code change. `MaxRAMPercentage` sizes the heap from the container memory limit.

</details>

### Q80. Write a Dockerfile for a static React site served by Nginx. 🟢

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`nginx.conf` with SPA routing:

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  location / { try_files $uri $uri/ /index.html; }
}
```

**Key trick →** `try_files ... /index.html` sends client-side routes like `/dashboard` to the SPA instead of returning 404. The final image has no Node at all.

</details>

### Q81. This Dockerfile rebuilds dependencies on every code change. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

Given:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

Fixed:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Key trick →** Order layers from least to most frequently changing. `COPY . .` before install invalidates the pip layer on every source edit.

</details>

### Q82. Write a `.dockerignore` for a typical Node or Python project and explain why it matters. 🟢

<details>
<summary><b>Answer</b></summary>

```gitignore
.git
.gitignore
node_modules
__pycache__/
*.pyc
.venv/
.env
*.log
coverage/
dist/
Dockerfile*
docker-compose*.yml
.github/
```

**Key trick →** It shrinks the build context (faster builds), stops local `node_modules` overwriting the in-image install, and keeps `.env` secrets and `.git` history out of the image.

</details>

### Q83. Add a HEALTHCHECK to a Dockerfile for a web app without installing curl. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM python:3.12-slim
# ...
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2).status == 200 else 1)"
```

**Key trick →** Use the runtime the image already has (Python, Node, or the app binary's own `healthcheck` subcommand) instead of adding curl. Note that Kubernetes ignores `HEALTHCHECK`; it uses probes.

</details>

### Q84. Write a Dockerfile that uses a build secret (e.g. a private npm token) without leaking it into image layers. 🔴

<details>
<summary><b>Answer</b></summary>

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine
WORKDIR /app
COPY package*.json .npmrc ./
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    npm ci --omit=dev
COPY . .
CMD ["node", "server.js"]
```

`.npmrc` contains `//registry.npmjs.org/:_authToken=${NPM_TOKEN}`. Build with:

```bash
docker build --secret id=npm_token,env=NPM_TOKEN -t app .
```

**Key trick →** BuildKit secret mounts exist only for that single `RUN` step and are never written to a layer. `ARG`/`ENV` values stay visible in `docker history`. (The `env=` form of the mount needs a recent Dockerfile frontend; the older form mounts the secret at `/run/secrets/npm_token` instead.)

</details>

### Q85. Fix the PID 1 / signal problem: the container takes 10 seconds to stop. 🟡

<details>
<summary><b>Answer</b></summary>

Given:

```dockerfile
CMD npm start
```

Fixed:

```dockerfile
CMD ["node", "server.js"]
```

If you need a shell wrapper, end it with `exec "$@"`, or add an init:

```bash
docker run --init myapp
```

**Key trick →** Shell-form `CMD` runs `/bin/sh -c`, which becomes PID 1 and doesn't forward SIGTERM, so Docker waits the 10-second grace period and then sends SIGKILL. Exec form makes the app PID 1 directly. `npm` also doesn't forward signals reliably.

</details>

### Q86. Write a docker-compose.yml for an app, Postgres and Redis, where the app waits for a healthy database. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
services:
  app:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://app:app@db:5432/app
      REDIS_URL: redis://cache:6379/0
    depends_on:
      db:    { condition: service_healthy }
      cache: { condition: service_started }
  db:
    image: postgres:16
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: app }
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      retries: 10
  cache:
    image: redis:7-alpine
volumes:
  pgdata:
```

**Key trick →** Plain `depends_on` only orders startup. `condition: service_healthy` actually waits for the healthcheck to pass. Services reach each other by service name (`db`) on the Compose network.

</details>

### Q87. Shrink a 1.2 GB Python image. What do you change? 🔴

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM python:3.12 AS build
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels
COPY . .
USER 10001
CMD ["python", "app.py"]
```

**Key trick →** Compile wheels in the full image (which has gcc and headers), then install them into `-slim`, so the compilers never ship. Use `--no-cache-dir`, a `.dockerignore`, and `dive` to find big layers.

</details>

### Q88. Write a Dockerfile where the app's config differs per environment without rebuilding. 🟢

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
ENV APP_ENV=production LOG_LEVEL=info
CMD ["python", "app.py"]
```

```bash
docker run -e APP_ENV=staging -e LOG_LEVEL=debug -v ./staging.yaml:/app/config.yaml:ro myapp:1.4.2
```

**Key trick →** `ENV` sets defaults and the runtime `-e` (or Kubernetes `env`/ConfigMap) overrides them. The same image digest then runs everywhere.

</details>

### Q89. Explain and use `ARG` vs `ENV` in a Dockerfile to stamp a version into the image. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM alpine:3.20
ARG VERSION=dev
ARG GIT_SHA=unknown
LABEL org.opencontainers.image.version=$VERSION \
      org.opencontainers.image.revision=$GIT_SHA
ENV APP_VERSION=$VERSION
```

```bash
docker build --build-arg VERSION=1.4.2 --build-arg GIT_SHA=$(git rev-parse --short HEAD) .
```

**Key trick →** `ARG` exists only at build time; copying it into `ENV` keeps it at runtime. OCI labels let `docker inspect` show which commit an image came from.

</details>

### Q90. Write a Dockerfile that builds multi-architecture images (amd64 and arm64) and push them. 🔴

<details>
<summary><b>Answer</b></summary>

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.23 AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```

```bash
docker buildx create --use --name multi
docker buildx build --platform linux/amd64,linux/arm64 -t myrepo/app:1.0.0 --push .
```

**Key trick →** `--platform=$BUILDPLATFORM` runs the compiler natively and cross-compiles with `TARGETARCH`. That is much faster than emulating arm64 through QEMU.

</details>

### Q91. Write a Dockerfile that runs as non-root but still needs to write to a directory. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY --chown=node:node . .
RUN mkdir -p /app/uploads && chown node:node /app/uploads
USER node
VOLUME ["/app/uploads"]
CMD ["node", "server.js"]
```

**Key trick →** Create and `chown` the writable directory *before* `USER`, because after it you no longer have permission. In Kubernetes, `securityContext.fsGroup` handles volume ownership.

</details>

### Q92. Write a Dockerfile for a cron-style job container that runs a script once and exits. 🟢

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM alpine:3.20
RUN apk add --no-cache bash curl jq aws-cli
COPY --chmod=0755 backup.sh /usr/local/bin/backup.sh
USER 65534:65534
ENTRYPOINT ["/usr/local/bin/backup.sh"]
```

**Key trick →** Don't run crond inside the container. Let the scheduler (a Kubernetes CronJob or an EventBridge rule plus ECS task) start it, and let the exit code report success. `COPY --chmod` avoids an extra `RUN chmod` layer.

</details>

### Q93. Pin a base image so builds are reproducible and safe from tag changes. 🔴

<details>
<summary><b>Answer</b></summary>

```dockerfile
FROM python:3.12-slim@sha256:<digest-from-registry>
```

```bash
docker buildx imagetools inspect python:3.12-slim   # shows the index digest
```

**Key trick →** Tags are mutable and digests are not. Let Renovate bump the digest through PRs so you still get security patches, but each change is reviewed and traceable.

</details>

### Q94. Write the `docker run` command that runs a container with hardened runtime settings. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
docker run -d --name api \
  --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --user 10001:10001 \
  --memory 512m --cpus 1 --pids-limit 200 \
  -p 8080:8080 myapp:1.4.2
```

**Key trick →** A read-only root filesystem plus dropping all capabilities blocks most post-exploit actions. `--pids-limit` stops fork bombs, and `no-new-privileges` blocks setuid escalation.

</details>

### Q95. Write a Dockerfile that uses BuildKit cache mounts to speed up package installs. 🟡

<details>
<summary><b>Answer</b></summary>

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Key trick →** A cache mount keeps the pip/npm/go cache between builds without storing it in the image. Even when the layer is invalidated, downloads come from the local cache.

</details>

## Write-a-Kubernetes-manifest tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q96. Write a Deployment and Service for an Nginx app with 3 replicas, resources and probes. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app: web }
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 256Mi }
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
```

**Key trick →** The Service `selector` and the Deployment's `selector.matchLabels` must both match the **pod template labels**, otherwise the Service has no endpoints.

</details>

### Q97. Add an HPA that scales the Deployment between 3 and 15 pods at 70% CPU. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Key trick →** Utilization is a percentage of the container's CPU **request**, so the HPA shows `<unknown>` if requests aren't set or metrics-server isn't installed.

</details>

### Q98. Fix this broken Deployment YAML. 🟢

<details>
<summary><b>Answer</b></summary>

Broken:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: "2"
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api
        image: myrepo/api:1.0
        ports:
        - containerPort: "8080"
```

Fixed:

```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myrepo/api:1.0
          ports:
            - containerPort: 8080
```

**Key trick →** Three bugs: `replicas` and `containerPort` must be integers, not strings, and the template label `app: backend` doesn't match the selector, which the API server rejects with `selector does not match template labels`. Run `kubectl apply --dry-run=server -f` to catch these before applying.

</details>

### Q99. Write a ConfigMap and Secret and inject them as environment variables and as a mounted file. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: info
  app.yaml: |
    featureX: true
---
apiVersion: v1
kind: Secret
metadata: { name: app-secret }
type: Opaque
stringData:
  DB_PASSWORD: s3cr3t
---
# in the pod spec
containers:
  - name: app
    image: myapp:1.0
    envFrom:
      - configMapRef: { name: app-config }
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef: { name: app-secret, key: DB_PASSWORD }
    volumeMounts:
      - { name: cfg, mountPath: /etc/app, readOnly: true }
volumes:
  - name: cfg
    configMap:
      name: app-config
      items: [{ key: app.yaml, path: app.yaml }]
```

**Key trick →** `stringData` lets you write plain text and the API server base64-encodes it. Env vars don't update when the ConfigMap changes, but mounted files do (eventually, and not when mounted with `subPath`).

</details>

### Q100. Write an Ingress that routes `/api` to the api service and `/` to the web service with TLS. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 8080 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```

**Key trick →** `pathType` is required in `networking.k8s.io/v1`, and `ingressClassName` picks the controller. cert-manager fills the `app-tls` Secret automatically through the annotation.

</details>

### Q101. Write a pod spec with a hardened securityContext. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile: { type: RuntimeDefault }
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: { drop: ["ALL"] }
      volumeMounts:
        - { name: tmp, mountPath: /tmp }
  volumes:
    - name: tmp
      emptyDir: {}
```

**Key trick →** This passes the Pod Security Standards "restricted" profile. With a read-only root filesystem, give the app an `emptyDir` for `/tmp` or it crashes on its first temp file.

</details>

### Q102. Write a CronJob that runs a backup every night at 02:00 and keeps limited history. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 600
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myrepo/pg-backup:1.2
              envFrom: [{ secretRef: { name: db-backup-creds } }]
```

**Key trick →** `concurrencyPolicy: Forbid` stops a slow backup from overlapping the next run. `timeZone` (stable since Kubernetes 1.27) avoids UTC confusion. Job pods need `restartPolicy` set to `OnFailure` or `Never`.

</details>

### Q103. Write a StatefulSet for a 3-node Redis or Postgres with per-pod storage and a headless Service. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: v1
kind: Service
metadata: { name: pg }
spec:
  clusterIP: None
  selector: { app: pg }
  ports: [{ port: 5432 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: pg }
spec:
  serviceName: pg
  replicas: 3
  selector: { matchLabels: { app: pg } }
  template:
    metadata: { labels: { app: pg } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports: [{ containerPort: 5432 }]
          env:
            - { name: PGDATA, value: /var/lib/postgresql/data/pgdata }
          volumeMounts:
            - { name: data, mountPath: /var/lib/postgresql/data }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources: { requests: { storage: 20Gi } }
```

**Key trick →** The headless Service (`clusterIP: None`) gives stable DNS names like `pg-0.pg.<ns>.svc.cluster.local`, and `volumeClaimTemplates` gives each pod its own PVC that survives rescheduling. The StatefulSet alone doesn't do replication; that needs an operator (CloudNativePG, Zalando) or app config.

</details>

### Q104. Write a NetworkPolicy that allows only the frontend to reach the backend on port 8080, and denies everything else. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress, namespace: shop }
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: backend-from-frontend, namespace: shop }
spec:
  podSelector: { matchLabels: { app: backend } }
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: frontend } }
      ports:
        - { protocol: TCP, port: 8080 }
```

**Key trick →** An empty `podSelector: {}` selects every pod in the namespace, which makes it the default deny. NetworkPolicies only take effect if the CNI enforces them (Calico, Cilium, or the VPC CNI with network policy enabled).

</details>

### Q105. Write RBAC that lets a CI service account deploy only in namespace `staging`. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: ci-deployer, namespace: staging }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: deployer, namespace: staging }
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: ci-deployer, namespace: staging }
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: staging
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

Verify: `kubectl auth can-i delete deployments -n staging --as=system:serviceaccount:staging:ci-deployer`.

**Key trick →** A Role plus RoleBinding is namespace-scoped. Use a ClusterRole only when you need cluster-wide reach. Don't grant `secrets` access unless the pipeline really needs it.

</details>

### Q106. Write a PodDisruptionBudget and topology spread so node drains never take the app down. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: api } }
---
# in the Deployment pod template
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector: { matchLabels: { app: api } }
```

**Key trick →** The PDB stops `kubectl drain` from evicting below 2 ready pods, and the spread puts replicas in different zones. With `replicas: 2` and `minAvailable: 2`, a drain blocks forever, so leave room.

</details>

### Q107. Write a Deployment with a zero-downtime rolling update strategy and graceful shutdown. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 10
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: api
          image: myrepo/api:1.5.0
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
```

**Key trick →** `maxUnavailable: 0` keeps full capacity during the rollout. The `preStop` sleep gives kube-proxy and the load balancer time to drop the pod from endpoints before the app gets SIGTERM, which prevents 502s during deploys.

</details>

### Q108. Write an init container that waits for the database before the app starts. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z postgres 5432; do echo waiting for db; sleep 2; done"]
  containers:
    - name: app
      image: myapp:1.0
```

**Key trick →** Init containers run to completion, one at a time, before app containers start. Still make the app retry its connections, because the database can disappear later too.

</details>

### Q109. Fix this Service that shows no endpoints. 🟢

<details>
<summary><b>Answer</b></summary>

Broken:

```yaml
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector:
    app: api
    tier: backend
  ports:
    - port: 80
      targetPort: 8000
# pods are labelled: app=api, and listen on 8080
```

Fixed:

```yaml
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

Check with `kubectl get endpointslices -l kubernetes.io/service-name=api` and `kubectl get pods --show-labels`.

**Key trick →** The selector is an AND of all labels, so the extra `tier: backend` matched nothing. Even after fixing that, a wrong `targetPort` gives connection refused. Fix both.

</details>

### Q110. Write a sidecar pattern: an app container plus a log-shipper reading a shared volume. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  volumes:
    - name: logs
      emptyDir: {}
  initContainers:
    - name: log-shipper
      image: fluent/fluent-bit:3.1
      restartPolicy: Always          # native sidecar
      volumeMounts: [{ name: logs, mountPath: /var/log/app, readOnly: true }]
  containers:
    - name: app
      image: legacy-app:2.3
      volumeMounts: [{ name: logs, mountPath: /var/log/app }]
```

**Key trick →** An init container with `restartPolicy: Always` is a native sidecar (on by default since Kubernetes 1.29). It starts before the app and is stopped after it, which also fixes Jobs that never complete because the sidecar keeps running.

</details>

### Q111. Write a ResourceQuota and LimitRange for a team namespace. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: team-a, namespace: team-a }
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.memory: 60Gi
    pods: "100"
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata: { name: defaults, namespace: team-a }
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 100m, memory: 128Mi }
      default:        { memory: 256Mi }
      max:            { cpu: "4", memory: 8Gi }
```

**Key trick →** Once a quota covers CPU and memory, pods without requests are rejected. The LimitRange injects defaults so teams don't hit `must specify requests` errors.

</details>

### Q112. Write a Job that processes 50 work items with 5 running in parallel. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: process-items }
spec:
  completions: 50
  parallelism: 5
  completionMode: Indexed
  backoffLimit: 6
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: myrepo/worker:1.0
          command: ["python", "worker.py"]
          # each pod reads its index from the JOB_COMPLETION_INDEX env var
```

**Key trick →** `completionMode: Indexed` gives each pod a unique `JOB_COMPLETION_INDEX` (0–49), so you can shard work without a queue. `ttlSecondsAfterFinished` garbage-collects the finished Job.

</details>

### Q113. Fix this YAML that fails with "mapping values are not allowed here". 🟢

<details>
<summary><b>Answer</b></summary>

Broken:

```yaml
containers:
  - name: app
    image: myapp:1.0
    command: ["sh", "-c", "echo starting: now && ./run"]
    env:
      - name: GREETING
        value: Hello: world
```

Fixed:

```yaml
    env:
      - name: GREETING
        value: "Hello: world"
```

**Key trick →** An unquoted `: ` inside a plain scalar starts a new mapping. Quote values that contain `: `, `#`, or start with `*`, `&`, `{` or `[`. Also quote `"yes"`, `"no"`, `"on"` and numbers you want as strings.

</details>

### Q114. Write a Deployment that uses the downward API to expose the pod name and node name to the app. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
env:
  - name: POD_NAME
    valueFrom: { fieldRef: { fieldPath: metadata.name } }
  - name: NODE_NAME
    valueFrom: { fieldRef: { fieldPath: spec.nodeName } }
  - name: MEM_LIMIT
    valueFrom:
      resourceFieldRef: { containerName: app, resource: limits.memory }
```

**Key trick →** The downward API lets the app log which pod and node it runs on, or size its caches from its own memory limit, without calling the Kubernetes API.

</details>

### Q115. Write an HPA that scales on a custom metric (requests per second) with scale-down stabilisation. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric: { name: http_requests_per_second }
        target: { type: AverageValue, averageValue: "100" }
  behavior:
    scaleUp:
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{ type: Pods, value: 2, periodSeconds: 60 }]
```

**Key trick →** Custom metrics need an adapter (Prometheus Adapter or KEDA) serving `custom.metrics.k8s.io`. The `behavior` block scales up fast and down slowly, which prevents flapping.

</details>

### Q116. Write a Kustomize overlay that changes replicas and image tag for production. 🟡

<details>
<summary><b>Answer</b></summary>

```text
base/kustomization.yaml       -> resources: [deployment.yaml, service.yaml]
overlays/prod/kustomization.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
  - ../../base
images:
  - name: myrepo/api
    newTag: "1.5.0"
replicas:
  - name: api
    count: 6
patches:
  - path: resources-patch.yaml
```

Render: `kubectl kustomize overlays/prod` or apply with `kubectl apply -k overlays/prod`.

**Key trick →** The `images` and `replicas` transformers change values without a patch file. CI bumps tags with `kustomize edit set image myrepo/api:1.5.1`.

</details>

### Q117. Write a minimal Helm chart template for a Deployment with values for image and replicas. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels: {{- include "app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

**Key trick →** The `checksum/config` annotation changes whenever the ConfigMap changes, which forces a rolling restart. `nindent` keeps YAML indentation correct.

</details>

### Q118. Write a pod spec that pins pods to GPU nodes using node affinity and tolerations. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
spec:
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["g5.xlarge", "g5.2xlarge"]
  containers:
    - name: trainer
      image: myrepo/trainer:1.0
      resources:
        limits: { nvidia.com/gpu: 1 }
```

**Key trick →** A taint keeps other pods *off* GPU nodes and the toleration *allows* these pods on, but only affinity *forces* them there. You need both halves. The GPU limit needs the NVIDIA device plugin.

</details>

### Q119. Fix the probe configuration that keeps restarting a slow-starting Java app. 🟡

<details>
<summary><b>Answer</b></summary>

Broken:

```yaml
livenessProbe:
  httpGet: { path: /actuator/health, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

Fixed:

```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 5
  failureThreshold: 30        # up to 150s to start
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
```

**Key trick →** The app needed about 60s to start, but liveness killed it after 20s, causing a CrashLoopBackOff. A `startupProbe` holds off liveness until the first success.

</details>

### Q120. Write an External Secrets Operator manifest that syncs a secret from AWS Secrets Manager. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: { name: aws-sm, namespace: prod }
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef: { name: eso-reader }   # IRSA-annotated SA
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds, namespace: prod }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-sm, kind: SecretStore }
  target: { name: db-creds }
  data:
    - secretKey: password
      remoteRef: { key: prod/app/db, property: password }
```

**Key trick →** The Kubernetes Secret is generated, never committed to Git, and `refreshInterval` picks up rotations. Check which API version your installed ESO release serves; newer releases add `external-secrets.io/v1`.

</details>

## Write-a-pipeline tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q121. Write a GitHub Actions workflow that runs tests on every PR and push to main. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm test -- --ci
```

**Key trick →** `cache: npm` in `setup-node` caches downloads keyed on the lockfile, and a top-level `permissions: contents: read` makes the `GITHUB_TOKEN` least-privilege.

</details>

### Q122. Write a workflow that builds a Docker image and pushes it to ECR using OIDC (no access keys). 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
name: build-push
on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-ecr-push
          aws-region: ap-south-1
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ steps.ecr.outputs.registry }}/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Key trick →** `id-token: write` lets the job request an OIDC token that AWS trades for short-lived credentials. The IAM role's trust policy must restrict the `sub` claim to this repo and branch.

</details>

### Q123. Write a matrix job that tests on Python 3.10, 3.11 and 3.12 on Linux and Windows. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        python: ["3.10", "3.11", "3.12"]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
          cache: pip
      - run: pip install -r requirements.txt
      - run: pytest -q
```

**Key trick →** Quote `"3.10"`, because YAML reads an unquoted `3.10` as the float `3.1`. `fail-fast: false` lets the other jobs finish so you see every failing combination.

</details>

### Q124. Write a declarative Jenkinsfile with build, test, Docker push and a manual approval before prod. 🔴

<details>
<summary><b>Answer</b></summary>

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "myrepo/app:${env.GIT_COMMIT.take(7)}"
  }
  options {
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  stages {
    stage('Build') { steps { sh 'mvn -B -DskipTests package' } }
    stage('Test')  {
      steps { sh 'mvn -B test' }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }
    stage('Docker') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub',
                         usernameVariable: 'U', passwordVariable: 'P')]) {
          sh '''
            echo "$P" | docker login -u "$U" --password-stdin
            docker build -t "$IMAGE" .
            docker push "$IMAGE"
          '''
        }
      }
    }
    stage('Deploy prod') {
      when { branch 'main' }
      input {
        message 'Deploy to production?'
        ok 'Deploy'
      }
      steps { sh "helm upgrade --install app ./chart --set image.tag=${env.GIT_COMMIT.take(7)}" }
    }
  }
  post { failure { echo 'Build failed - notify the team here' } }
}
```

**Key trick →** Use single-quoted `sh '''...'''` so Groovy doesn't interpolate the secret; the shell expands `$P` instead. Jenkins warns about Groovy string interpolation of secrets. An `input` inside a stage with `agent any` holds an executor while it waits, so in real setups put the approval stage on `agent none`.

</details>

### Q125. Write a GitHub Actions deploy job that requires approval for production using environments. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: ./deploy.sh staging
  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - run: ./deploy.sh production
```

**Key trick →** Required reviewers, wait timers and branch restrictions are configured on the `production` environment in repo settings. The job pauses until someone approves, and environment-scoped secrets are only released then.

</details>

### Q126. Write a reusable workflow and call it from another repo. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# org/ci-templates/.github/workflows/node-ci.yml
name: node-ci
on:
  workflow_call:
    inputs:
      node-version: { type: string, default: "22" }
    secrets:
      NPM_TOKEN: { required: false }
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ inputs.node-version }}", cache: npm }
      - run: npm ci && npm test
```

```yaml
# service repo
jobs:
  ci:
    uses: org/ci-templates/.github/workflows/node-ci.yml@v1
    with: { node-version: "20" }
    secrets: inherit
```

**Key trick →** `workflow_call` turns a workflow into a callable unit. Pin it with `@v1` (or a SHA) so template changes roll out on purpose.

</details>

### Q127. Write a workflow that only runs the backend job when files under `backend/` change. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  pull_request:
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
    steps:
      - uses: actions/checkout@v4
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            backend:
              - 'backend/**'
  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make -C backend test
```

**Key trick →** The trigger-level `paths:` filter skips the whole workflow, which leaves required status checks pending forever. A job-level filter lets a skipped job report success.

</details>

### Q128. Write a GitLab CI pipeline with build, test and deploy stages, caching and manual prod deploy. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
stages: [build, test, deploy]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - docker build -t "$IMAGE" .
    - docker push "$IMAGE"

test:
  stage: test
  image: node:22
  cache:
    key: { files: [package-lock.json] }
    paths: [.npm/]
  script:
    - npm ci --cache .npm --prefer-offline
    - npm test

deploy_prod:
  stage: deploy
  image: alpine/helm:3.16.2
  script:
    - helm upgrade --install app ./chart --set image.tag=$CI_COMMIT_SHORT_SHA
  environment: { name: production, url: https://app.example.com }
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

**Key trick →** A cache key based on the lockfile only refreshes the cache when dependencies change. `rules` with `when: manual` adds a play button only on the default branch.

</details>

### Q129. Write a workflow step that scans the image with Trivy and fails on HIGH/CRITICAL vulnerabilities. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: myapp:${{ github.sha }}
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: "1"
    format: table
```

CLI equivalent: `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 myapp:tag`.

**Key trick →** `exit-code: 1` is what actually fails the build, because Trivy exits 0 by default. `ignore-unfixed` stops the gate blocking on CVEs nobody can patch yet. Pin the action to a version (or a SHA).

</details>

### Q130. Write a Terraform plan-on-PR, apply-on-merge workflow. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
name: terraform
on:
  pull_request: { paths: ["infra/**"] }
  push: { branches: [main], paths: ["infra/**"] }

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  tf:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: infra } }
    concurrency: { group: tf-infra, cancel-in-progress: false }
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.8 }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-terraform
          aws-region: ap-south-1
      - run: terraform init -input=false
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -input=false -out=tfplan
      - if: github.event_name == 'pull_request'
        run: terraform show -no-color tfplan > plan.txt
      - if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const plan = require('fs').readFileSync('infra/plan.txt', 'utf8').slice(0, 60000);
            github.rest.issues.createComment({ ...context.repo, issue_number: context.issue.number,
              body: "```\n" + plan + "\n```" });
      - if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -input=false tfplan
```

**Key trick →** `concurrency` with `cancel-in-progress: false` queues runs so two applies never race on state, and applying the saved `tfplan` guarantees you apply exactly what was planned.

</details>

### Q131. Write a Jenkinsfile that runs stages in parallel. 🟡

<details>
<summary><b>Answer</b></summary>

```groovy
pipeline {
  agent none
  stages {
    stage('Checks') {
      parallel {
        stage('Unit')  { agent { docker { image 'node:22' } } steps { sh 'npm ci && npm test' } }
        stage('Lint')  { agent { docker { image 'node:22' } } steps { sh 'npm ci && npm run lint' } }
        stage('Trivy fs') {
          agent {
            docker {
              image 'aquasec/trivy:0.56.2'
              args '--entrypoint='
            }
          }
          steps { sh 'trivy fs --exit-code 1 --severity CRITICAL .' }
        }
      }
    }
  }
}
```

**Key trick →** `agent none` at the top with an agent per parallel stage gives each branch its own executor. Add `failFast true` to abort siblings when one fails.

</details>

### Q132. Write a workflow that cuts a release: tag push builds artifacts and creates a GitHub Release. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  push:
    tags: ["v*.*.*"]
permissions:
  contents: write
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make dist VERSION=${GITHUB_REF_NAME}
      - run: gh release create "$GITHUB_REF_NAME" dist/* --generate-notes
        env:
          GH_TOKEN: ${{ github.token }}
```

**Key trick →** `GITHUB_REF_NAME` is the tag name (e.g. `v1.4.2`). Creating releases needs `contents: write`. `--generate-notes` builds the changelog from merged PRs.

</details>

### Q133. Write a scheduled workflow that runs nightly and can also be triggered manually with an input. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
on:
  schedule:
    - cron: "30 20 * * *"     # 20:30 UTC = 02:00 IST
  workflow_dispatch:
    inputs:
      env:
        type: choice
        options: [staging, prod]
        default: staging
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Checking drift in ${{ inputs.env || 'staging' }}"
```

**Key trick →** Cron in Actions is always UTC and can be delayed under load. `inputs.env` is empty on scheduled runs, so use `||` to give a default.

</details>

### Q134. Write a workflow that passes a value (image tag) from one job to another. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - id: meta
        run: echo "tag=$(date +%Y%m%d)-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.tag }}"
```

**Key trick →** Write `name=value` to `$GITHUB_OUTPUT` (the old `::set-output` is deprecated), expose it as a job output, and read it with `needs.<job>.outputs`. Jobs run on separate machines, so files and environment variables don't carry over.

</details>

### Q135. Write a GitOps image-bump step: CI updates the image tag in a manifests repo. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Bump image in env repo
  env:
    GH_TOKEN: ${{ secrets.ENV_REPO_TOKEN }}
    TAG: ${{ github.sha }}
  run: |
    git clone "https://x-access-token:${GH_TOKEN}@github.com/org/env-repo.git"
    cd env-repo/apps/api/overlays/staging
    kustomize edit set image "myrepo/api=myrepo/api:${TAG}"
    git config user.name "ci-bot"
    git config user.email "ci-bot@users.noreply.github.com"
    git commit -am "api: deploy ${TAG::7} to staging"
    for i in 1 2 3; do git pull --rebase && git push && break; sleep $((i*2)); done
```

**Key trick →** CI never talks to the cluster. Argo CD watches the env repo and syncs it. The pull-rebase-push retry loop handles concurrent pushes from other services' pipelines.

</details>

### Q136. Write a workflow with concurrency so a newer push cancels an older running deploy for the same branch. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

**Key trick →** Grouping by `github.ref` means one active run per branch. For production deploys or Terraform, set `cancel-in-progress: false`, because killing an apply halfway is worse than waiting.

</details>

### Q137. Write a Jenkins shared library step `deployApp(env, tag)` and use it. 🔴

<details>
<summary><b>Answer</b></summary>

```groovy
// vars/deployApp.groovy in the shared library repo
def call(String envName, String tag) {
  stage("Deploy ${envName}") {
    withCredentials([file(credentialsId: "kubeconfig-${envName}", variable: 'KUBECONFIG')]) {
      sh """
        helm upgrade --install app ./chart \
          --namespace ${envName} --set image.tag=${tag} \
          --wait --timeout 5m --atomic
      """
    }
  }
}
```

```groovy
// Jenkinsfile
@Library('platform-lib@v2') _
pipeline {
  agent any
  stages {
    stage('Deploy') { steps { script { deployApp('staging', env.GIT_COMMIT.take(7)) } } }
  }
}
```

**Key trick →** Files in `vars/` become global steps. Pin the library with `@v2`. `helm --atomic` rolls back automatically if the release doesn't become ready.

</details>

### Q138. Write a workflow step that caches Terraform providers and Go modules. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.terraform.d/plugin-cache
      ~/go/pkg/mod
    key: ${{ runner.os }}-deps-${{ hashFiles('**/.terraform.lock.hcl', '**/go.sum') }}
    restore-keys: ${{ runner.os }}-deps-
- run: echo "TF_PLUGIN_CACHE_DIR=$HOME/.terraform.d/plugin-cache" >> "$GITHUB_ENV" && mkdir -p ~/.terraform.d/plugin-cache
```

**Key trick →** Key the cache on lockfile hashes so it's invalidated exactly when dependencies change. `restore-keys` gives a partial hit to start from.

</details>

### Q139. Fix this GitHub Actions workflow that never triggers. 🟡

<details>
<summary><b>Answer</b></summary>

Broken (saved as `.github/workflow/ci.yaml`):

```yaml
on:
  push:
    branch: main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

Fixed (saved as `.github/workflows/ci.yaml`):

```yaml
on:
  push:
    branches: [main]
```

**Key trick →** Two bugs: the folder must be `workflows` (plural), and the filter key is `branches`, not `branch`. Actions shows the invalid key as an error on the Actions tab; `actionlint` catches both locally.

</details>

### Q140. Write a pipeline step that runs a smoke test after deploy and rolls back on failure. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Deploy
  run: |
    kubectl -n prod set image deploy/api api=myrepo/api:${{ github.sha }}
    kubectl -n prod rollout status deploy/api --timeout=300s
- name: Smoke test
  id: smoke
  run: |
    for path in /health /api/v1/status; do
      code=$(curl -s -o /dev/null -w '%{http_code}' "https://api.example.com${path}")
      echo "$path -> $code"; [ "$code" = "200" ] || exit 1
    done
- name: Roll back
  if: failure()
  run: kubectl -n prod rollout undo deploy/api && kubectl -n prod rollout status deploy/api
```

**Key trick →** `rollout status` exits non-zero if the rollout doesn't finish within the timeout, and `if: failure()` runs the rollback when any earlier step fails, whether that is the deploy or the smoke test.

</details>

## Write-Terraform tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q141. Write Terraform for an S3 bucket with versioning, encryption and public access blocked. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
resource "aws_s3_bucket" "artifacts" {
  bucket = "devopsshack-artifacts-${var.env}"
}

resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "aws:kms" }
  }
}

resource "aws_s3_bucket_public_access_block" "artifacts" {
  bucket                  = aws_s3_bucket.artifacts.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Key trick →** Since AWS provider v4, versioning, encryption and the rest are separate resources, not inline blocks on `aws_s3_bucket`. Interviewers check whether you know the current provider style.

</details>

### Q142. Write the remote backend configuration with S3 and state locking. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket       = "devopsshack-tfstate"
    key          = "network/prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

**Key trick →** `use_lockfile = true` (Terraform 1.10+) uses S3's native conditional writes for locking, so a DynamoDB table is no longer needed (`dynamodb_table` is the older method and is deprecated). Backend blocks can't use variables.

</details>

### Q143. Write a VPC with public and private subnets across 3 AZs using `count` or `for_each`. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
data "aws_availability_zones" "available" { state = "available" }

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "main" }
}

resource "aws_subnet" "public" {
  for_each                = { for i, az in local.azs : az => i }
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.key
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, each.value)
  map_public_ip_on_launch = true
  tags = { Name = "public-${each.key}" }
}

resource "aws_subnet" "private" {
  for_each          = { for i, az in local.azs : az => i }
  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, each.value + 10)
  tags = { Name = "private-${each.key}" }
}
```

**Key trick →** `cidrsubnet("10.0.0.0/16", 8, n)` returns `10.0.n.0/24`. Keying `for_each` by AZ name means removing one AZ doesn't renumber and recreate the others, which is what happens with `count`.

</details>

### Q144. Write a reusable module for an EC2 instance with inputs, outputs and validation. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
# modules/ec2/variables.tf
variable "name" { type = string }
variable "instance_type" {
  type    = string
  default = "t3.micro"
  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 family is allowed in this module."
  }
}
variable "subnet_id" { type = string }
variable "tags" {
  type    = map(string)
  default = {}
}

# modules/ec2/main.tf
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "this" {
  ami           = data.aws_ami.al2023.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  metadata_options { http_tokens = "required" }   # IMDSv2 only
  tags = merge(var.tags, { Name = var.name })
}

# modules/ec2/outputs.tf
output "id"         { value = aws_instance.this.id }
output "private_ip" { value = aws_instance.this.private_ip }
```

```hcl
module "web" {
  source        = "./modules/ec2"
  name          = "web-1"
  instance_type = "t3.small"
  subnet_id     = aws_subnet.private["ap-south-1a"].id
}
```

**Key trick →** `validation` blocks reject bad input at plan time with a readable message. Requiring IMDSv2 via `http_tokens = "required"` is a security default worth mentioning.

</details>

### Q145. Write a security group that allows HTTPS from anywhere and SSH only from an office CIDR. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
resource "aws_security_group" "web" {
  name   = "web"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "https" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_ingress_rule" "ssh_office" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = var.office_cidr
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "all" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

**Key trick →** The per-rule resources are the current recommendation. Mixing inline `ingress` blocks with separate rule resources makes them fight on every apply. Terraform removes AWS's default allow-all egress, so you must declare egress yourself.

</details>

### Q146. Create multiple IAM users from a list with `for_each`, and output their ARNs as a map. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
variable "users" {
  type    = set(string)
  default = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "this" {
  for_each = var.users
  name     = each.value
}

output "user_arns" {
  value = { for name, u in aws_iam_user.this : name => u.arn }
}
```

**Key trick →** `for_each` over a set keys each resource by name (`aws_iam_user.this["bob"]`), so removing "alice" deletes only alice. With `count`, bob and carol would shift index and be recreated.

</details>

### Q147. Write Terraform that looks up an existing VPC by tag instead of hardcoding its ID. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
data "aws_vpc" "shared" {
  tags = { Name = "shared-prod" }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  tags = { Tier = "private" }
}

output "private_subnet_ids" { value = data.aws_subnets.private.ids }
```

**Key trick →** Data sources read existing infrastructure without managing it. The plan fails if zero (or for `aws_vpc`, more than one) match, which is a useful guardrail.

</details>

### Q148. Write a dynamic block that creates ingress rules from a variable list. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
variable "ingress" {
  type = list(object({ port = number, cidr = string, desc = string }))
  default = [
    { port = 80,  cidr = "0.0.0.0/0",   desc = "http"  },
    { port = 443, cidr = "0.0.0.0/0",   desc = "https" },
    { port = 22,  cidr = "10.0.0.0/8",  desc = "ssh internal" },
  ]
}

resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress
    content {
      description = ingress.value.desc
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

**Key trick →** Inside `content`, the iterator is named after the block (`ingress.value`). Use dynamic blocks sparingly, because they hurt readability.

</details>

### Q149. Write Terraform for an IAM role that EC2 can assume, with a least-privilege S3 read policy. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
data "aws_iam_policy_document" "assume_ec2" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

data "aws_iam_policy_document" "read_artifacts" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.artifacts.arn}/*"]
  }
  statement {
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.artifacts.arn]
  }
}

resource "aws_iam_role" "app" {
  name               = "app-ec2"
  assume_role_policy = data.aws_iam_policy_document.assume_ec2.json
}

resource "aws_iam_role_policy" "read" {
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.read_artifacts.json
}

resource "aws_iam_instance_profile" "app" {
  name = "app-ec2"
  role = aws_iam_role.app.name
}
```

**Key trick →** `ListBucket` applies to the bucket ARN and `GetObject` to `bucket/*`. Mixing them up is the most common IAM bug. EC2 attaches roles through an instance profile, not directly.

</details>

### Q150. Use `locals`, `terraform.workspace` or tfvars to vary instance size by environment. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
variable "env" { type = string }

locals {
  sizes = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "m6i.large"
  }
  instance_type = lookup(local.sizes, var.env, "t3.micro")
  common_tags = {
    Environment = var.env
    ManagedBy   = "terraform"
  }
}
```

```bash
terraform plan -var-file=envs/prod.tfvars
```

**Key trick →** A map plus `lookup` keeps the environment logic in one place. Many teams prefer separate directories or state per environment over workspaces, so a wrong `workspace select` can't apply dev changes to prod.

</details>

### Q151. Import an existing, manually created S3 bucket into Terraform. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "legacy-reports-bucket"
}

resource "aws_s3_bucket" "legacy" {
  bucket = "legacy-reports-bucket"
}
```

```bash
terraform plan -generate-config-out=generated.tf   # optional: let Terraform write the resource
terraform apply
```

**Key trick →** The `import` block (Terraform 1.5+) is code-reviewed and planned like any other change, unlike the CLI `terraform import`. Aim for a plan showing "1 to import, 0 to change".

</details>

### Q152. Rename a resource in code without Terraform destroying and recreating it. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
moved {
  from = aws_instance.web
  to   = module.web.aws_instance.this
}
```

CLI alternative: `terraform state mv aws_instance.web module.web.aws_instance.this`.

**Key trick →** `moved` blocks (Terraform 1.1+) record the refactor in code, so every workspace and teammate gets the move on their next plan. `state mv` is a one-off manual step.

</details>

### Q153. Protect a production database from accidental `terraform destroy`. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
resource "aws_db_instance" "prod" {
  identifier          = "prod-db"
  engine              = "postgres"
  instance_class      = "db.r6g.large"
  allocated_storage   = 100
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "prod-db-final"

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [password]
  }
}
```

**Key trick →** `prevent_destroy` fails the plan in Terraform, and `deletion_protection` stops the delete in AWS itself, even from the console. Use both.

</details>

### Q154. Write an EKS cluster with a managed node group using the community module. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "devopsshack-prod"
  cluster_version = "1.31"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  cluster_endpoint_public_access           = true
  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    general = {
      instance_types = ["m6i.large"]
      min_size       = 2
      max_size       = 6
      desired_size   = 3
      capacity_type  = "ON_DEMAND"
    }
  }

  tags = { Environment = "prod" }
}
```

**Key trick →** Pin the module's major version, because v19 to v20 changed authentication to EKS access entries. Put nodes in private subnets, and `enable_cluster_creator_admin_permissions` keeps you from locking yourself out.

</details>

### Q155. Write a `for` expression that builds a map of subnet IDs by AZ and a filtered list. 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
locals {
  subnet_by_az = { for s in aws_subnet.private : s.availability_zone => s.id }

  big_instances = [for name, inst in var.instances : name if inst.size != "small"]

  upper_names = [for n in var.names : upper(n)]
}
```

**Key trick →** `{ for k, v in ... : key => value }` builds a map and `[ ... ]` builds a list. A trailing `if` filters. Duplicate map keys are an error unless you add `...` to group them.

</details>

### Q156. Write a conditional resource that only exists in production. 🟢

<details>
<summary><b>Answer</b></summary>

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count               = var.env == "prod" ? 1 : 0
  alarm_name          = "high-cpu-${var.env}"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  statistic           = "Average"
  period              = 300
  evaluation_periods  = 2
  threshold           = 80
  comparison_operator = "GreaterThanThreshold"
}

output "alarm_arn" {
  value = one(aws_cloudwatch_metric_alarm.high_cpu[*].arn)
}
```

**Key trick →** `count = cond ? 1 : 0` toggles a resource. `one(...[*].arn)` returns `null` when it doesn't exist, instead of an index error.

</details>

### Q157. Write Terraform that reads outputs from another state (the network stack) in the app stack. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "devopsshack-tfstate"
    key    = "network/prod/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.small"
  subnet_id     = data.terraform_remote_state.network.outputs.private_subnet_ids[0]
}
```

**Key trick →** This couples the stacks and needs read access to the whole network state file, secrets included. Many teams prefer publishing values to SSM Parameter Store and reading them with `aws_ssm_parameter`, or using data source lookups by tag.

</details>

### Q158. Generate a random password, store it in Secrets Manager and pass it to RDS without it appearing in code. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
resource "aws_db_instance" "app" {
  identifier                  = "app-db"
  engine                      = "postgres"
  engine_version              = "16"
  instance_class              = "db.t4g.medium"
  allocated_storage           = 50
  username                    = "app_admin"
  manage_master_user_password = true      # RDS creates and rotates it in Secrets Manager
  storage_encrypted           = true
  skip_final_snapshot         = false
  final_snapshot_identifier   = "app-db-final"
}

output "db_secret_arn" {
  value = aws_db_instance.app.master_user_secret[0].secret_arn
}
```

**Key trick →** `manage_master_user_password` keeps the password out of Terraform state entirely. A `random_password` resource would still be stored in plain text in state. Apps read the secret ARN at runtime.

</details>

### Q159. Write a `terraform test` file that asserts the bucket name follows a convention. 🔴

<details>
<summary><b>Answer</b></summary>

```hcl
# tests/bucket.tftest.hcl
variables {
  env = "dev"
}

run "bucket_name_convention" {
  command = plan

  assert {
    condition     = startswith(aws_s3_bucket.artifacts.bucket, "devopsshack-")
    error_message = "Bucket name must start with devopsshack-"
  }
  assert {
    condition     = aws_s3_bucket_versioning.artifacts.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled"
  }
}
```

Run with `terraform test`.

**Key trick →** The native test framework (Terraform 1.6+) runs `plan`-only tests with no real resources created, which is fast enough for every PR. `command = apply` gives full integration tests against a sandbox account.

</details>

### Q160. Fix this Terraform that fails with "Cycle" or "Reference to undeclared resource". 🟡

<details>
<summary><b>Answer</b></summary>

Broken:

```hcl
resource "aws_security_group" "app" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]
  }
}
resource "aws_security_group" "db" {
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

Fixed: create both groups without inline rules, then add the rules as separate resources:

```hcl
resource "aws_security_group" "app" {
  name   = "app"
  vpc_id = var.vpc_id
}

resource "aws_security_group" "db" {
  name   = "db"
  vpc_id = var.vpc_id
}

resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id            = aws_security_group.db.id
  referenced_security_group_id = aws_security_group.app.id
  from_port                    = 5432
  to_port                      = 5432
  ip_protocol                  = "tcp"
}
```

**Key trick →** Two groups whose inline rules reference each other form a dependency cycle. Splitting the rules into separate resources breaks the cycle, because both groups now exist before either rule is created.

</details>

## Write-an-Ansible-playbook tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Write a playbook that installs and starts Nginx on Ubuntu web servers. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Install and run nginx
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Start and enable nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run: `ansible-playbook -i inventory.ini site.yml --check --diff`, then without `--check`.

**Key trick →** `cache_valid_time` stops apt from refreshing on every run, and modules with `state:` are idempotent, so a second run reports `changed=0`.

</details>

### Q162. Write an inventory with groups, a group of groups, and per-group variables. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
# inventory.yml
all:
  children:
    web:
      hosts:
        web1: { ansible_host: 10.0.1.10 }
        web2: { ansible_host: 10.0.1.11 }
      vars:
        http_port: 80
    db:
      hosts:
        db1: { ansible_host: 10.0.2.10 }
    prod:
      children:
        web:
        db:
      vars:
        ansible_user: ubuntu
        ansible_ssh_private_key_file: ~/.ssh/prod.pem
```

Check it: `ansible-inventory -i inventory.yml --graph`.

**Key trick →** `prod` is a group of groups, so its vars apply to all web and db hosts. For anything beyond a few lines, put vars in `group_vars/prod.yml` instead of the inventory.

</details>

### Q163. Write a playbook that deploys a config file from a Jinja2 template and restarts the service only if it changed. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: web
  become: true
  vars:
    worker_connections: 2048
  tasks:
    - name: Deploy nginx.conf
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: "0644"
        validate: nginx -t -c %s
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```jinja
events { worker_connections {{ worker_connections }}; }
```

**Key trick →** A handler runs once at the end of the play, and only if a notifying task reported `changed`. `validate:` tests the rendered file before it replaces the live one, so a bad template can't break Nginx.

</details>

### Q164. Write a role structure for a Node.js app and show what goes in each folder. 🟡

<details>
<summary><b>Answer</b></summary>

```text
roles/nodeapp/
  defaults/main.yml     # lowest-priority vars users override: app_port: 3000
  vars/main.yml         # internal constants, high priority
  tasks/main.yml        # the steps
  handlers/main.yml     # restart nodeapp
  templates/app.service.j2
  files/                # static files to copy
  meta/main.yml         # dependencies, platforms
```

```yaml
# site.yml
- hosts: app
  become: true
  roles:
    - role: nodeapp
      vars:
        app_port: 8080
```

Scaffold with `ansible-galaxy role init roles/nodeapp`.

**Key trick →** Put tunables in `defaults/` (easy to override) and not `vars/` (hard to override). That one distinction comes up in almost every Ansible interview.

</details>

### Q165. Write a task that creates multiple users with SSH keys from a list. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
vars:
  devs:
    - { name: alice, key: "ssh-ed25519 AAAA... alice@laptop" }
    - { name: bob,   key: "ssh-ed25519 AAAA... bob@laptop" }
tasks:
  - name: Create users
    ansible.builtin.user:
      name: "{{ item.name }}"
      groups: sudo
      append: true
      shell: /bin/bash
    loop: "{{ devs }}"

  - name: Add SSH keys
    ansible.posix.authorized_key:
      user: "{{ item.name }}"
      key: "{{ item.key }}"
    loop: "{{ devs }}"
```

**Key trick →** `append: true` adds the group without removing the user's existing groups. Without it, `groups:` replaces the whole list.

</details>

### Q166. Write a rolling deploy playbook: 2 servers at a time, removed from the load balancer during the update. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: web
  become: true
  serial: 2
  max_fail_percentage: 0
  pre_tasks:
    - name: Deregister from target group
      community.aws.elb_target:
        target_group_arn: "{{ tg_arn }}"
        target_id: "{{ ec2_id }}"
        state: absent
      delegate_to: localhost
      become: false

  tasks:
    - name: Deploy new release
      ansible.builtin.unarchive:
        src: "https://artifacts.example.com/app-{{ version }}.tgz"
        dest: /opt/app
        remote_src: true
      notify: Restart app
    - ansible.builtin.meta: flush_handlers
    - name: Wait for health
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
      register: h
      retries: 20
      delay: 3
      until: h.status == 200

  post_tasks:
    - name: Register back
      community.aws.elb_target:
        target_group_arn: "{{ tg_arn }}"
        target_id: "{{ ec2_id }}"
        state: present
      delegate_to: localhost
      become: false

  handlers:
    - name: Restart app
      ansible.builtin.systemd: { name: app, state: restarted }
```

**Key trick →** `serial: 2` processes hosts in batches, `max_fail_percentage: 0` stops the rollout on the first failure, and `flush_handlers` restarts the app *before* the health check instead of at the end of the play.

</details>

### Q167. Encrypt a database password with Ansible Vault and use it in a playbook. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-vault encrypt_string 'S3cr3t!' --name 'db_password' >> group_vars/prod/vault.yml
ansible-playbook site.yml --vault-password-file ~/.vault_pass   # or --ask-vault-pass
```

```yaml
- name: Configure app
  ansible.builtin.template:
    src: app.env.j2
    dest: /etc/app/app.env
    mode: "0600"
  no_log: true
```

**Key trick →** `encrypt_string` encrypts a single value, so the rest of the vars file stays reviewable in PRs. `no_log: true` keeps the secret out of task output.

</details>

### Q168. Write a playbook that behaves differently on Ubuntu and RHEL hosts. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: all
  become: true
  tasks:
    - name: Load OS-specific vars
      ansible.builtin.include_vars: "{{ ansible_facts['os_family'] }}.yml"
      # Debian.yml: pkg_name: apache2   RedHat.yml: pkg_name: httpd

    - name: Install web server
      ansible.builtin.package:
        name: "{{ pkg_name }}"
        state: present

    - name: Open firewall on RHEL
      ansible.posix.firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled
      when: ansible_facts['os_family'] == "RedHat"
```

**Key trick →** The generic `package` module plus per-OS var files avoids an `apt` task and a `yum` task for every package. `os_family` groups distros (Debian covers Ubuntu, RedHat covers Rocky and Alma).

</details>

### Q169. Write a task that runs a command only if a file doesn't exist, without breaking idempotency. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Initialise the app database once
  ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /var/lib/app/.db_initialised
```

**Key trick →** `command` and `shell` always report `changed`. `creates:` (or `removes:`) makes them skip when the marker exists, so the task is idempotent and reports correctly.

</details>

### Q170. Register output of a command and use it in a later task with a condition. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Check current app version
  ansible.builtin.command: /opt/app/bin/app --version
  register: app_ver
  changed_when: false
  failed_when: false

- name: Upgrade if outdated
  ansible.builtin.include_tasks: upgrade.yml
  when: app_ver.rc != 0 or target_version not in app_ver.stdout
```

**Key trick →** `changed_when: false` marks read-only checks as unchanged. `failed_when: false` lets you handle a missing binary (rc != 0) yourself instead of failing the play.

</details>

### Q171. Write a playbook that hardens SSH: no root login, no password auth. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: all
  become: true
  tasks:
    - name: Harden sshd
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?{{ item.key }}\\s"
        line: "{{ item.key }} {{ item.value }}"
        validate: /usr/sbin/sshd -t -f %s
      loop:
        - { key: PermitRootLogin, value: "no" }
        - { key: PasswordAuthentication, value: "no" }
        - { key: MaxAuthTries, value: "3" }
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      ansible.builtin.service: { name: ssh, state: restarted }   # "sshd" on RHEL
```

**Key trick →** The `regexp` matches commented-out defaults too, so the line is replaced, not duplicated. `validate` with `sshd -t` stops you from locking yourself out with a broken config.

</details>

### Q172. Write a dynamic inventory for AWS EC2 grouped by tags. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
# inventory/aws_ec2.yml  (filename must end in aws_ec2.yml)
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters:
  instance-state-name: running
  tag:Project: devopsshack
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.Environment
    prefix: env
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
```

```bash
ansible-inventory -i inventory/aws_ec2.yml --graph
ansible-playbook -i inventory/aws_ec2.yml site.yml --limit role_web
```

**Key trick →** `keyed_groups` creates groups like `role_web` and `env_prod` from tags, so playbooks target roles and never hardcode IPs. It needs `boto3` and the `amazon.aws` collection.

</details>

### Q173. Write a block/rescue/always structure for a deploy with automatic rollback. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
- name: Deploy with rollback
  block:
    - name: Switch symlink to new release
      ansible.builtin.file:
        src: "/opt/app/releases/{{ version }}"
        dest: /opt/app/current
        state: link
      notify: Restart app
    - ansible.builtin.meta: flush_handlers
    - name: Health check
      ansible.builtin.uri: { url: "http://localhost:8080/health" }
      register: hc
      retries: 10
      delay: 3
      until: hc.status == 200
  rescue:
    - name: Roll back symlink
      ansible.builtin.file:
        src: "/opt/app/releases/{{ previous_version }}"
        dest: /opt/app/current
        state: link
    - name: Restart on old release
      ansible.builtin.systemd: { name: app, state: restarted }
    - ansible.builtin.fail:
        msg: "Deploy of {{ version }} failed, rolled back to {{ previous_version }}"
  always:
    - name: Record deploy attempt
      ansible.builtin.lineinfile:
        path: /var/log/deploys.log
        line: "{{ now(utc=true).isoformat() }} {{ version }} {{ 'FAILED' if ansible_failed_task is defined else 'OK' }}"
        create: true
```

**Key trick →** `rescue` runs only if a task in `block` fails, which works like try/except. The explicit `fail` at the end keeps the host marked as failed after the rollback so `serial` stops the rollout.

</details>

### Q174. Write a playbook that gathers disk usage from all servers and writes a single report on the control node. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: all
  gather_facts: true
  tasks:
    - name: Build report on control node
      ansible.builtin.template:
        src: disk_report.j2
        dest: ./disk_report.txt
      delegate_to: localhost
      run_once: true
```

```jinja
{% for h in ansible_play_hosts %}
{% for m in hostvars[h].ansible_facts.mounts if m.mount == '/' %}
{{ h }}: {{ ((1 - m.size_available / m.size_total) * 100) | round(1) }}% used on /
{% endfor %}
{% endfor %}
```

**Key trick →** `run_once` plus `delegate_to: localhost` runs one task for the whole play, and `hostvars[h]` reads facts gathered from every other host.

</details>

### Q175. Install Docker on Ubuntu with Ansible using the official repository. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: docker_hosts
  become: true
  tasks:
    - name: Prereqs
      ansible.builtin.apt: { name: [ca-certificates, curl], state: present, update_cache: true }

    - name: Docker GPG key
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc
        mode: "0644"

    - name: Docker repo
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{ ansible_facts['distribution_release'] }} stable"
        state: present

    - name: Install Docker
      ansible.builtin.apt:
        name: [docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin]
        state: present
        update_cache: true

    - name: Let ubuntu user run docker
      ansible.builtin.user: { name: ubuntu, groups: docker, append: true }
```

**Key trick →** `signed-by` scopes the key to this one repo. `apt-key` is deprecated, so don't use it.

</details>

### Q176. Write a playbook that waits for hosts to come back after a reboot, only when a reboot is needed. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: all
  become: true
  serial: "25%"
  tasks:
    - name: Upgrade packages
      ansible.builtin.apt: { upgrade: dist, update_cache: true }

    - name: Check if reboot required
      ansible.builtin.stat: { path: /var/run/reboot-required }
      register: rr

    - name: Reboot if needed
      ansible.builtin.reboot:
        reboot_timeout: 600
        msg: "Reboot by Ansible for kernel update"
      when: rr.stat.exists
```

**Key trick →** The `reboot` module reboots, waits for SSH and checks the host is back, with no fragile `wait_for_connection` dance. `serial: "25%"` keeps 75% of the fleet serving.

</details>

### Q177. Fix this playbook: the handler never runs, and the loop uses deprecated syntax. 🟡

<details>
<summary><b>Answer</b></summary>

Broken:

```yaml
tasks:
  - name: copy config
    copy: src=app.conf dest=/etc/app/app.conf
    notify: restart App
  - name: install pkgs
    apt: name={{ item }} state=present
    with_items:
      - git
      - curl
handlers:
  - name: restart app
    service: name=app state=restarted
```

Fixed:

```yaml
tasks:
  - name: Copy config
    ansible.builtin.copy:
      src: app.conf
      dest: /etc/app/app.conf
    notify: Restart app
  - name: Install packages
    ansible.builtin.apt:
      name: [git, curl]
      state: present
handlers:
  - name: Restart app
    ansible.builtin.service:
      name: app
      state: restarted
```

**Key trick →** Handler names are matched exactly, case included, so `restart App` didn't match `restart app`. Passing a list to `apt` installs all packages in one transaction, which is faster than a loop.

</details>

### Q178. Write an ad-hoc command set for quick live tasks: ping, run a command, copy a file, restart a service. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible all -i inventory.yml -m ansible.builtin.ping
ansible web -i inventory.yml -a "uptime"
ansible web -i inventory.yml -b -m ansible.builtin.copy -a "src=motd dest=/etc/motd mode=0644"
ansible web -i inventory.yml -b -m ansible.builtin.service -a "name=nginx state=restarted" --forks 5
```

**Key trick →** `-b` means become (sudo), and `--forks` controls parallelism (default 5). The `ping` module tests SSH and Python on the host, not ICMP.

</details>

### Q179. Write a playbook that uses tags so you can run only the config part. 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt: { name: nginx, state: present }
      tags: [install]
    - name: Deploy config
      ansible.builtin.template: { src: nginx.conf.j2, dest: /etc/nginx/nginx.conf }
      tags: [config]
      notify: Reload nginx
  handlers:
    - name: Reload nginx
      ansible.builtin.service: { name: nginx, state: reloaded }
```

```bash
ansible-playbook site.yml --tags config
ansible-playbook site.yml --list-tags
```

**Key trick →** Tags let you rerun one slice of a big playbook in seconds. Handlers notified by tagged tasks still run.

</details>

### Q180. Write a playbook that runs a database migration once, even though it targets 5 app servers. 🔴

<details>
<summary><b>Answer</b></summary>

```yaml
# Play 1: migrate exactly once, on one host
- hosts: app[0]
  become: true
  tasks:
    - name: Run migrations
      ansible.builtin.command: /opt/app/bin/migrate --to "{{ version }}"
      register: mig
      changed_when: "'No migrations to run' not in mig.stdout"

# Play 2: rolling deploy across all app servers
- hosts: app
  become: true
  serial: 1
  roles:
    - nodeapp
```

**Key trick →** `run_once` runs on the first host *of the current batch*, so combined with `serial: 1` it would run 5 times. A separate first play targeting `app[0]` guarantees exactly one run, and if it fails the deploy play never starts. Without `serial`, `run_once: true` on the task is enough.

</details>

## Live debugging tasks

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q181. A pod is in CrashLoopBackOff. You have 10 minutes. Walk through your commands. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pod api-7d9f -o wide
kubectl describe pod api-7d9f            # Events, Last State, Exit Code, Reason
kubectl logs api-7d9f --previous         # logs of the crashed container, not the new one
kubectl get pod api-7d9f -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

Read the exit code: `1` = app error (check the logs), `137` = SIGKILL (often OOMKilled, see `Reason`), `127` = command not found, `126` = not executable.

**Key trick →** `--previous` is the flag people forget. Without it you see logs of the container that just started and has nothing useful yet.

</details>

### Q182. The pod shows `OOMKilled`. Diagnose and fix it. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe pod api-7d9f | grep -A3 'Last State'     # Reason: OOMKilled, Exit Code: 137
kubectl top pod api-7d9f --containers                     # current usage vs limit
kubectl get pod api-7d9f -o jsonpath='{.spec.containers[0].resources}'
```

Fix: raise `resources.limits.memory` to match the real peak (from metrics, not a guess), or fix the app. For the JVM, set `-XX:MaxRAMPercentage=75` so the heap fits inside the limit; for Node, `--max-old-space-size`.

**Key trick →** OOMKilled means the container exceeded its **memory limit**, enforced by the kernel cgroup. Raising requests doesn't help; the limit, or the app's memory use, has to change.

</details>

### Q183. A pod is stuck in `ImagePullBackOff`. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe pod web-5c8 | sed -n '/Events/,$p'
# "manifest unknown"                  -> wrong tag
# "pull access denied" / "401"        -> private registry, missing imagePullSecret
# "no basic auth credentials" (ECR)   -> node role lacks ECR read permission
# "i/o timeout"                       -> node has no route to the registry (NAT/VPC endpoint)
kubectl create secret docker-registry regcred --docker-server=ghcr.io \
  --docker-username=bot --docker-password="$TOKEN"
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```

**Key trick →** The exact event message tells you which of the four causes it is. Read it before changing anything.

</details>

### Q184. A pod is `Pending` forever. What do you check? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe pod worker-0 | sed -n '/Events/,$p'
# "0/6 nodes are available: 3 Insufficient cpu, 3 node(s) had untolerated taint"
kubectl describe nodes | grep -A5 'Allocated resources'
kubectl get pvc                          # unbound PVC also keeps pods Pending
kubectl get nodes --show-labels | grep -i zone
```

Fixes: lower requests or add nodes (check the cluster autoscaler or Karpenter logs), add a toleration, fix the nodeSelector or affinity typo, or fix the StorageClass or zone mismatch for the PVC.

**Key trick →** The scheduler's event message breaks down why each node was rejected. Scheduling uses **requests**, not real usage, which is why a "mostly idle" cluster can still report Insufficient cpu.

</details>

### Q185. The Service returns "connection refused" but pods are Running. Debug it. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get endpointslices -l kubernetes.io/service-name=api    # empty = selector mismatch or pods not Ready
kubectl get svc api -o yaml | grep -A5 -E 'selector|ports'
kubectl get pods -l app=api -o wide
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
#   curl -v http://api:80/health            # via Service
#   curl -v http://<pod-ip>:8080/health     # direct to pod
kubectl exec api-xyz -- ss -ltn          # is the app listening on 0.0.0.0:8080 or only 127.0.0.1?
```

**Key trick →** Test pod IP directly, then the Service. If the pod IP works but the Service doesn't, it's the selector or `targetPort`. If the pod IP fails too, the app is bound to `127.0.0.1` or listening on the wrong port.

</details>

### Q186. A Docker container exits immediately after `docker run -d`. Find out why. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker ps -a --filter name=api               # STATUS: Exited (1) 3 seconds ago
docker logs api
docker inspect api --format '{{.State.ExitCode}} {{.State.Error}} {{.Config.Cmd}} {{.Config.Entrypoint}}'
docker run --rm -it --entrypoint sh myapp:1.0    # get a shell in the same image and run the command by hand
```

**Key trick →** A container lives only as long as its main process. `-d` with a command that finishes (or a service that daemonises itself, like `nginx` without `daemon off;`) exits at once. Overriding `--entrypoint` lets you look inside.

</details>

### Q187. `docker build` fails with "COPY failed: file not found in build context". Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker build -t app -f docker/Dockerfile .     # context is "." (repo root), not docker/
cat .dockerignore                              # is the file excluded?
```

Paths in `COPY` are relative to the **build context** (the last argument), not to the Dockerfile's location, and files listed in `.dockerignore` are invisible to the build.

**Key trick →** Run from the repo root with `-f path/to/Dockerfile`, and check `.dockerignore` before anything else.

</details>

### Q188. The app in a container can't reach the database container: "could not translate host name db". 🟡

<details>
<summary><b>Answer</b></summary>

```bash
docker network ls
docker inspect api --format '{{json .NetworkSettings.Networks}}' | jq 'keys'
docker inspect db  --format '{{json .NetworkSettings.Networks}}' | jq 'keys'
docker network create appnet
docker network connect appnet api
docker network connect appnet db
docker exec api getent hosts db
```

**Key trick →** Container name DNS only works on **user-defined** networks, not the default `bridge`. Both containers must share a network (Compose creates one automatically).

</details>

### Q189. A container works locally but fails in Kubernetes with "permission denied" writing to /app/data. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl exec -it api-xyz -- id                       # uid=10001 (non-root, set by securityContext)
kubectl exec -it api-xyz -- ls -ld /app/data         # owned by root:root
kubectl get pod api-xyz -o jsonpath='{.spec.securityContext}'
```

Fix: set `securityContext.fsGroup: 10001` so mounted volumes are group-owned by the pod, or `chown` the directory in the Dockerfile. With `readOnlyRootFilesystem: true`, mount an `emptyDir` at that path.

**Key trick →** Locally you ran as root, but the cluster enforces non-root and possibly a read-only root filesystem. Compare `id` with the directory's owner.

</details>

### Q190. A Deployment rollout is stuck: "Waiting for deployment rollout to finish: 1 out of 3 new replicas have been updated". 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl rollout status deploy/api --timeout=60s
kubectl get rs -l app=api                      # new ReplicaSet has pods not Ready
kubectl describe pod -l app=api | grep -E 'Readiness|Warning'
kubectl rollout history deploy/api
kubectl rollout undo deploy/api                # mitigate first
```

**Key trick →** With `maxUnavailable: 0` the rollout waits for the new pod to become **Ready**. A failing readiness probe (wrong path or port, a dependency down) stalls it safely. Roll back first, then debug.

</details>

### Q191. DNS inside the cluster is failing intermittently. Debug it. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
kubectl run dnstest --rm -it --image=nicolaka/netshoot -- bash
#   cat /etc/resolv.conf          # nameserver, search domains, ndots:5
#   dig api.default.svc.cluster.local
#   for i in $(seq 50); do dig +short google.com >/dev/null || echo fail; done
kubectl -n kube-system top pods -l k8s-app=kube-dns
```

Common causes: too few CoreDNS replicas (CPU throttling), conntrack races on UDP, `ndots:5` multiplying external lookups, or NetworkPolicy blocking port 53 egress. Fixes: scale CoreDNS or use NodeLocal DNSCache, set `dnsConfig.options ndots: 2`, and use FQDNs with a trailing dot for external names.

**Key trick →** `ndots:5` means `api.stripe.com` is first tried against every search domain, so one lookup becomes 4 to 5 queries. Mentioning it shows real production experience.

</details>

### Q192. A Linux server's disk is 100% full and the app is down. Fix it in 10 minutes. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
df -h                                   # which mount
df -i                                   # or inodes exhausted?
sudo du -xh --max-depth=1 /var | sort -rh | head
sudo journalctl --vacuum-size=500M
sudo find /var/log -name '*.gz' -mtime +7 -delete
sudo lsof +L1                           # deleted files still held open by a process
docker system df && docker system prune -f
```

**Key trick →** If `df` says full but `du` doesn't add up, a process is holding a deleted file open (`lsof +L1`). Truncate it with `: > /proc/<pid>/fd/<n>` or restart that process.

</details>

### Q193. The server has high load average but CPU usage looks low. What's happening? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
uptime
top                       # look at %wa (iowait) and processes in state D
vmstat 1 5                # 'b' column = blocked on I/O, 'wa' = iowait
iostat -xz 1 3            # %util and await per device
ps -eo state,pid,cmd | awk '$1=="D"'
```

**Key trick →** Linux load average counts runnable **and** uninterruptible (D state, usually I/O) tasks. High load with idle CPU means slow disk or a hung NFS mount, not compute.

</details>

### Q194. curl to an internal service on an EC2 instance hangs (timeout, not refused). Debug it. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
curl -v --connect-timeout 5 http://10.0.2.15:8080/health
nc -vz -w 3 10.0.2.15 8080
# on the target:
sudo ss -ltnp | grep 8080          # listening, and on 0.0.0.0?
sudo iptables -L -n | head; sudo ufw status
# in AWS: security group inbound 8080 from the source SG/CIDR, NACL both directions, route tables
```

**Key trick →** "Refused" means the packet arrived and nothing was listening (or a host firewall sent a reject). A **timeout** means packets were dropped silently, which in AWS is almost always a security group or NACL. Remember NACLs are stateless and need the ephemeral return ports allowed.

</details>

### Q195. A Kubernetes node is `NotReady`. Diagnose it. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl describe node ip-10-0-1-23 | sed -n '/Conditions/,/Addresses/p'
# MemoryPressure / DiskPressure / PIDPressure / Ready=False "Kubelet stopped posting node status"
# on the node (SSM/SSH):
systemctl status kubelet containerd
journalctl -u kubelet --since "15 min ago" | tail -50
df -h /var/lib/containerd /var/lib/kubelet
crictl ps -a | head
```

Typical causes: disk full (images or logs), kubelet certificate expired, containerd crashed, CNI pod failing on the node, or node lost network to the API server. Mitigate with `kubectl cordon` and `kubectl drain --ignore-daemonsets --delete-emptydir-data`, and replace the node if it's managed.

**Key trick →** Node conditions tell you which kind of pressure. `journalctl -u kubelet` tells you the rest. In a cloud node group, replacing beats repairing.

</details>

### Q196. The application returns 502 from the ALB intermittently after each deploy. Fix it. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
aws elbv2 describe-target-health --target-group-arn "$TG_ARN"
kubectl get deploy api -o jsonpath='{.spec.template.spec.containers[0].lifecycle}'
kubectl get deploy api -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}'
```

Cause: pods are killed while the ALB still sends them traffic. Deregistration lags behind pod termination. Fix: add a `preStop` sleep (10–15s), make `terminationGracePeriodSeconds` larger than the sleep plus drain time, use pod readiness gates with the AWS Load Balancer Controller, and handle SIGTERM gracefully in the app. Also check that the app's keep-alive timeout is longer than the ALB idle timeout (60s default).

**Key trick →** 502s that appear only during deploys are almost always a termination race. 502s at random are often a keep-alive mismatch, where the app closes idle connections before the ALB does.

</details>

### Q197. Terraform apply fails with "Error acquiring the state lock". What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
terraform plan            # shows the lock ID, who holds it and since when
# confirm no CI job or teammate is actually running an apply
terraform force-unlock <LOCK_ID>
```

**Key trick →** Only force-unlock after confirming the holder is dead (for example, a crashed CI runner). Unlocking during a live apply can corrupt state. CI concurrency groups prevent most of these.

</details>

### Q198. An Ansible playbook fails with "UNREACHABLE! ... Permission denied (publickey)". Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible web1 -i inventory.yml -m ping -vvvv        # shows the exact ssh command used
ssh -i ~/.ssh/prod.pem ubuntu@10.0.1.10            # reproduce manually
chmod 600 ~/.ssh/prod.pem
```

Check `ansible_user` (ubuntu vs ec2-user vs admin), `ansible_ssh_private_key_file`, and that the key is in the target user's `authorized_keys`.

**Key trick →** `-vvvv` prints the full SSH command Ansible runs, which you can paste into a terminal to reproduce. Wrong username is the cause most of the time.

</details>

### Q199. A GitHub Actions job fails with "Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity". Fix it. 🔴

<details>
<summary><b>Answer</b></summary>

Check, in order:

1. The workflow has `permissions: id-token: write`.
2. The account has an IAM OIDC provider for `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`.
3. The role trust policy matches the token's `sub` claim:

```json
"Condition": {
  "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
  "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:org/app:ref:refs/heads/main" }
}
```

4. If the job uses `environment: production`, the `sub` is `repo:org/app:environment:production`, not the branch form.

**Key trick →** The environment-versus-branch `sub` format is the usual culprit. The job worked until someone added `environment:` to it.

</details>

### Q200. Live task: a pod can't reach an external API, but other pods can. Find the cause. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pod payments-abc -o wide                     # namespace, node, IP
kubectl get networkpolicy -n payments                    # an egress policy only in this namespace?
kubectl describe networkpolicy -n payments
kubectl exec -n payments payments-abc -- nslookup api.stripe.com    # DNS egress (UDP/TCP 53) allowed?
kubectl exec -n payments payments-abc -- curl -sv --max-time 5 https://api.stripe.com
kubectl get pod payments-abc -o jsonpath='{.spec.dnsPolicy} {.spec.hostNetwork}'
```

Typical finding: a default-deny egress NetworkPolicy in that namespace allows the app's port but not DNS to kube-dns on port 53. Fix by adding an egress rule to the `kube-system` CoreDNS pods on UDP and TCP 53, plus 443 to the external range. Other possibilities: per-pod security groups (EKS Security Groups for Pods), or the pod landing on a node in a subnet without a NAT route.

**Key trick →** "Only this pod fails" means compare what is unique to it: namespace policies, node or subnet, service account, and pod-level security groups.

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
