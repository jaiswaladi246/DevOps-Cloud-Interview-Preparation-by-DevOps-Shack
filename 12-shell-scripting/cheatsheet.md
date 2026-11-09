# Shell Scripting — Cheatsheet

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

> Dense Bash reference for DevOps work: syntax, expansions, tests, error handling, text tools, scheduling and quality tooling, grouped by task. Bash 5.x unless noted.

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Script skeleton](#script-skeleton)
- [Running and debugging](#running-and-debugging)
- [Variables and quoting](#variables-and-quoting)
- [Special parameters and variables](#special-parameters-and-variables)
- [Parameter expansion](#parameter-expansion)
- [Arrays](#arrays)
- [Test operators](#test-operators)
- [Conditionals and loops](#conditionals-and-loops)
- [Functions](#functions)
- [Redirection, here-docs and process substitution](#redirection-here-docs-and-process-substitution)
- [Exit codes](#exit-codes)
- [Strict mode: set -euo pipefail](#strict-mode-set--euo-pipefail)
- [trap and signals](#trap-and-signals)
- [getopts](#getopts)
- [grep](#grep)
- [sed](#sed)
- [awk](#awk)
- [cut, sort, uniq, tr and friends](#cut-sort-uniq-tr-and-friends)
- [find](#find)
- [xargs](#xargs)
- [jq](#jq)
- [yq (mikefarah v4)](#yq-mikefarah-v4)
- [cron syntax](#cron-syntax)
- [systemd timers](#systemd-timers)
- [Locking, timeouts and retries](#locking-timeouts-and-retries)
- [ShellCheck and bats](#shellcheck-and-bats)
- [Bash vs POSIX sh](#bash-vs-posix-sh)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Script skeleton

```bash
#!/usr/bin/env bash
# name.sh - one line saying what this does
set -Eeuo pipefail
shopt -s inherit_errexit nullglob
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LC_ALL=C
SCRIPT_DIR=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
readonly SCRIPT_DIR                          # assign first: readonly X=$(cmd) hides cmd's failure

log()  { printf '%(%Y-%m-%dT%H:%M:%S%z)T %-5s %s\n' -1 "$1" "${*:2}" >&2; }
die()  { log ERROR "$1"; exit "${2:-1}"; }
need() { command -v "$1" >/dev/null || die "missing command: $1" 127; }

tmpdir=""
cleanup() { [[ -n $tmpdir ]] && rm -rf -- "$tmpdir"; }
trap cleanup EXIT
trap 'log ERROR "line $LINENO: $BASH_COMMAND exited $?"' ERR
trap 'exit 130' INT
trap 'exit 143' TERM

main() {
  need jq
  log INFO "starting from $SCRIPT_DIR"
  tmpdir=$(mktemp -d)
  exec 9>/run/lock/name.lock
  flock -n 9 || { log WARN "already running"; exit 0; }
  # ... work ...
}

main "$@"
```

Save it, `chmod +x`, and check it with `shellcheck -x name.sh` before the first real run.

## Running and debugging

| Task | Command |
|------|---------|
| Run with shebang (needs `+x`) | `./script.sh` |
| Run, ignoring shebang | `bash script.sh` |
| Run in current shell (keeps `cd`, vars) | `source script.sh` or `. script.sh` |
| Syntax check only | `bash -n script.sh` |
| Trace every command | `bash -x script.sh` |
| Trace a section | `set -x; ...; set +x` |
| Better trace prefix | `export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}(): '` |
| Trace to a separate file | `exec 7>>trace.log; BASH_XTRACEFD=7; set -x` |
| Trace without echoing `set +x` | `{ set +x; } 2>/dev/null` |
| Verbose (print lines as read) | `bash -v script.sh` |
| Run with an empty environment (cron-like) | `env -i bash ./script.sh` |
| What is this name? | `type -a cmd`, `command -v cmd` |
| Lint | `shellcheck -x script.sh` |
| Lint for POSIX sh | `shellcheck -s sh script.sh` |
| Format check | `shfmt -d -i 2 -ci script.sh` |
| Count syscalls / forks | `strace -c -f ./script.sh` |
| Time a run | `time ./script.sh` |

## Variables and quoting

| Form | Meaning |
|------|---------|
| `name=value` | Assign (no spaces around `=`) |
| `"$var"` | Expand as one word, no globbing. Default choice |
| `'literal $var'` | Nothing expands |
| `$'tab\there\n'` | ANSI-C quoting: `\t`, `\n`, `\x41`, `é` |
| `"$(cmd)"` | Command substitution, trailing newlines stripped |
| `$((a + b * 2))` | Arithmetic expansion (integers only) |
| `export VAR=x` | Pass to child processes |
| `VAR=x cmd` | Set only for that one command |
| `readonly VAR=x`, `declare -r` | Constant |
| `declare -i n=5` | Integer attribute |
| `declare -l s`, `declare -u s` | Auto lower / upper case on assignment |
| `declare -n ref=var` | Name reference (Bash 4.3+) |
| `local v` | Function-scoped (dynamic scope) |
| `unset var`, `unset -f fn` | Remove variable / function |
| `printf -v out '%05d' 42` | Format into a variable, no fork |
| `"$@"` | All args, each as its own word. Use this |
| `"$*"` | All args joined by first char of `IFS` into one word |

## Special parameters and variables

| Name | Value |
|------|-------|
| `$0` | Script name as invoked |
| `$1` … `${10}` | Positional arguments (braces from 10) |
| `$#` | Number of positional arguments |
| `$?` | Exit status of the last command |
| `$$` | PID of the shell (the parent, even inside subshells) |
| `$BASHPID` | PID of the current (sub)shell |
| `$!` | PID of the last background job |
| `$_` | Last argument of the previous command |
| `$-` | Current shell flags (`i` = interactive) |
| `${PIPESTATUS[@]}` | Exit status of each stage of the last pipeline |
| `$LINENO`, `$FUNCNAME`, `${BASH_SOURCE[0]}` | Current line, function stack, file |
| `$BASH_COMMAND` | Command being executed (useful in traps) |
| `$BASH_REMATCH` | Captures from the last `[[ =~ ]]` |
| `$BASH_VERSINFO` | Version array, e.g. `(( BASH_VERSINFO[0] >= 5 ))` |
| `$RANDOM`, `$SRANDOM` | 0–32767; 32-bit random (5.1+) |
| `$SECONDS` | Seconds since the shell started (assignable) |
| `$EPOCHSECONDS`, `$EPOCHREALTIME` | Unix time, with microseconds (5.0+) |
| `$IFS` | Field separators, default space, tab, newline |
| `$OPTIND`, `$OPTARG` | `getopts` state |
| `$PWD`, `$OLDPWD`, `$HOME`, `$UID`, `$EUID` | Self-explanatory; `$EUID` is 0 for root |

## Parameter expansion

| Form | Result for `p=/var/log/app.log.gz` (or as noted) |
|------|---------|
| `${var:-def}` | `def` if unset **or empty** (`${var-def}`: only if unset) |
| `${var:=def}` | As above, and assign it |
| `${var:?msg}` | Exit with `msg` if unset or empty |
| `${var:+alt}` | `alt` if set and non-empty, else nothing |
| `${#p}` | Length: `20` |
| `${p#*/}` | Remove shortest prefix match: `var/log/app.log.gz` |
| `${p##*/}` | Remove longest prefix match (basename): `app.log.gz` |
| `${p%/*}` | Remove shortest suffix match (dirname): `/var/log` |
| `${p%%.*}` | Remove longest suffix match: `/var/log/app` |
| `${p%.gz}` | Strip a known suffix: `/var/log/app.log` |
| `${p##*.}` | Extension: `gz` |
| `${p/log/LOG}` | Replace first: `/var/LOG/app.log.gz` |
| `${p//log/LOG}` | Replace all: `/var/LOG/app.LOG.gz` |
| `${p/#\/var/\/srv}` | Replace at start: `/srv/log/app.log.gz` |
| `${p/%.gz/.zst}` | Replace at end: `/var/log/app.log.zst` |
| `${p:5:3}` | Substring (offset 5, length 3): `log` |
| `${p: -6}` | Last 6 chars (note the space): `log.gz` |
| `${s^}`, `${s^^}` | Upper-case first / all (4.0+) |
| `${s,}`, `${s,,}` | Lower-case first / all |
| `${s@Q}` | Quoted for reuse as shell input (4.4+) |
| `${!ref}` | Indirect: value of the variable named by `ref` |
| `${!prefix@}` | Names of variables starting with `prefix` |
| `${arr[@]+"${arr[@]}"}` | Empty-array-safe expansion under `set -u` on Bash < 4.4 |

Patterns in `#`, `%`, `/` are globs, not regex. `shopt -s extglob` adds `?(a|b)`, `*(…)`, `+(…)`, `@(…)`, `!(…)`.

## Arrays

```bash
# Indexed
arr=(one "two words" three)
arr+=(four)                       # append
echo "${arr[0]}" "${arr[-1]}"     # first, last (negative index 4.3+)
echo "${#arr[@]}"                 # length: 4
echo "${!arr[@]}"                 # indices: 0 1 2 3
echo "${arr[@]:1:2}"              # slice: two words three
for x in "${arr[@]}"; do printf '[%s]\n' "$x"; done
unset 'arr[1]'                    # remove element (indices are not renumbered)
mapfile -t lines < file.txt       # file -> array, one line per element (readarray)
mapfile -t pods < <(kubectl get pods -o name)
IFS=, read -r -a fields <<< "a,b,c"
printf '%s\n' "${arr[@]}" | sort  # sort an array

# Associative (Bash 4+)
declare -A port=([web]=80 [db]=5432)
port[cache]=6379
echo "${port[db]}"
for k in "${!port[@]}"; do echo "$k=${port[$k]}"; done   # order is not defined
[[ -v port[web] ]] && echo "has web"                      # key exists (4.3+)
unset 'port[cache]'

# Commands in arrays, never in strings
cmd=(rsync -a --delete --exclude 'node_modules')
[[ $dry == 1 ]] && cmd+=(--dry-run)
"${cmd[@]}" src/ dst/
```

## Test operators

**Files** (`[[ -f $f ]]`)

| Op | True if | Op | True if |
|----|---------|----|---------|
| `-e` | exists | `-s` | exists and size > 0 |
| `-f` | regular file | `-d` | directory |
| `-L` / `-h` | symlink | `-p` | named pipe |
| `-S` | socket | `-b` / `-c` | block / char device |
| `-r` `-w` `-x` | readable / writable / executable by you | `-O` | owned by you |
| `a -nt b` | a newer than b | `a -ot b` | a older than b |
| `a -ef b` | same inode | `-t 0` | fd 0 is a terminal |

**Strings**

| Op | True if |
|----|---------|
| `-z $s` | empty |
| `-n $s` | not empty |
| `$a == $b` | equal (`=` also works) |
| `$a != $b` | not equal |
| `$a < $b`, `$a > $b` | sorts before / after (inside `[[ ]]`) |
| `$s == *.log` | glob match (right side unquoted, `[[ ]]` only) |
| `$s =~ ^v[0-9]+$` | regex match (right side unquoted; captures in `BASH_REMATCH`) |
| `-v name` | variable is set (4.2+) |

**Integers**

| `[[ ]]` / `[ ]` | `(( ))` | Meaning |
|-----------------|---------|---------|
| `-eq` | `==` | equal |
| `-ne` | `!=` | not equal |
| `-lt` / `-le` | `<` / `<=` | less than / or equal |
| `-gt` / `-ge` | `>` / `>=` | greater than / or equal |

**`[ ]` vs `[[ ]]` vs `(( ))`**

| | `[ ]` / `test` | `[[ ]]` | `(( ))` |
|---|---|---|---|
| Type | Command (POSIX) | Bash keyword | Bash arithmetic |
| Word splitting of vars | Yes: quote everything | No | No |
| `&&`, `\|\|` inside | No (use `-a`/`-o`, deprecated) | Yes | Yes |
| Glob / regex match | No | Yes | No |
| Use for | POSIX `sh` scripts | Strings and files in Bash | Numbers |

Leading zeros make `(( ))` read numbers as octal: `(( 10#$hour > 8 ))`. Decimals are not supported: compare with `awk`.

## Conditionals and loops

```bash
if [[ -f $conf ]]; then ...; elif [[ -d $conf ]]; then ...; else ...; fi
if grep -q ERROR "$log"; then ...; fi          # test any command's exit status
[[ -n $x ]] && echo set || echo empty          # not a real if/else: echo set failing runs both

case $1 in
  start|up)   do_start ;;
  stop)       do_stop ;;
  *.tar.gz)   extract "$1" ;;
  -h|--help)  usage; exit 0 ;;
  *)          usage >&2; exit 2 ;;
esac

for f in /var/log/app/*.log; do ...; done       # glob (+ shopt -s nullglob)
for i in {1..10}; do ...; done                  # brace range (literal numbers only)
for (( i = 0; i < n; i++ )); do ...; done       # C-style
while IFS= read -r line || [[ -n $line ]]; do ...; done < file   # line by line, last line safe
while IFS=: read -r user _ uid _; do ...; done < /etc/passwd     # split fields
while read -r -d '' f; do ...; done < <(find . -type f -print0)  # NUL-safe
until curl -fsS "$url" >/dev/null; do sleep 2; done
select opt in deploy rollback quit; do ...; break; done          # interactive menu
break 2; continue                                                # leave / skip (nested levels)
```

## Functions

```bash
fn() {                         # or: function fn { ... }
  local name=$1 count=${2:-1}  # local = function scope
  local out
  out=$(some_cmd) || return 1  # separate declare and assign (SC2155)
  printf '%s\n' "$out"         # "return" data by printing it
  return 0                     # status 0-255
}
result=$(fn arg)               # capture output (runs in a subshell)
if fn arg >/dev/null; then ...; fi  # use the status
declare -f fn                  # print the definition
export -f fn                   # make it available to bash -c children (xargs, parallel)
```

Bash scope is **dynamic**: a function sees its caller's `local` variables. A function called in `$(...)` cannot change the caller's variables.

## Redirection, here-docs and process substitution

| Form | Meaning |
|------|---------|
| `cmd > f` / `cmd >> f` | stdout to file (truncate / append) |
| `cmd 2> f` | stderr to file |
| `cmd > f 2>&1` | both to file (order matters) |
| `cmd &> f`, `cmd &>> f` | both to file (Bash) |
| `cmd 2>&1 \| tee -a log` | both to screen and log |
| `cmd < f` | stdin from file |
| `cmd 2>/dev/null` | discard stderr |
| `>&2 echo "msg"` | print to stderr |
| `exec >> log 2>&1` | redirect the rest of the script |
| `exec > >(tee -a log) 2>&1` | rest of the script to screen and log |
| `exec 9> file` / `exec 9>&-` | open / close fd 9 |
| `exec {fd}> file` | let Bash pick a free fd number |
| `cmd <<< "$var"` | here-string as stdin |
| `diff <(cmd1) <(cmd2)` | process substitution as a file |
| `cmd > >(gzip > out.gz)` | output process substitution |
| `set -o noclobber`, `>\|` | refuse to overwrite / force overwrite |

```bash
cat <<EOF             # unquoted delimiter: $vars and $(cmd) expand
host=$HOST date=$(date +%F)
EOF
cat <<'EOF'           # quoted delimiter: nothing expands (templates, nginx $host)
proxy_set_header Host $host;
EOF
	cat <<-EOF        # <<- strips leading TABS (not spaces), for indented code
	indented with tabs
	EOF
sudo tee /etc/app.conf > /dev/null <<'EOF'   # here-doc into a root-owned file
key=value
EOF
```

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error (also `grep` no match, `diff` files differ) |
| `2` | Misuse: bad option or syntax (`grep`/`diff` real error) |
| `124` | `timeout` expired |
| `125` | `timeout` itself failed |
| `126` | Found but not executable (permission, `noexec`) |
| `127` | Command not found |
| `128+N` | Killed by signal N |
| `130` | SIGINT (Ctrl+C) |
| `137` | SIGKILL (`kill -9`, OOM killer, `docker stop` timeout) |
| `141` | SIGPIPE (writer after reader closed: `cmd \| head`) |
| `143` | SIGTERM (`kill`, `systemctl stop`, `docker stop`) |
| `255` | `ssh` connection error; `exit -1` wraps to 255 |
| `0/1/2/3` | Monitoring plugins: OK / WARNING / CRITICAL / UNKNOWN |

```bash
cmd; rc=$?                                   # save immediately
cmd || rc=$?                                 # same, safe under set -e
if ! cmd; then echo "failed"; fi             # $? here is 0 (the negation)
a | b | c; echo "${PIPESTATUS[@]}"           # status of each stage
exit "$rc"                                   # propagate
```

## Strict mode: set -euo pipefail

| Option | Effect |
|--------|--------|
| `set -e` (errexit) | Exit when an unhandled command fails |
| `set -u` (nounset) | Error on unset variables (`${var:-}` for optional ones) |
| `set -o pipefail` | Pipeline fails if any stage fails |
| `set -E` (errtrace) | `ERR` trap is inherited by functions and subshells |
| `shopt -s inherit_errexit` | Command substitutions inherit `-e` (4.4+) |
| `set -x` / `set +x` | Trace on / off |
| `set -f` | Disable globbing |
| `set -C` | noclobber |

**Where `set -e` does not stop you**

| Situation | Example | Fix |
|-----------|---------|-----|
| Condition of `if`, `while`, `until` | `if f; then` (whole body of `f` ignores `-e`) | Check steps inside `f` with `\|\| return 1` |
| Left side of `&&` / `\|\|` | `f && echo ok` | Same |
| After `!` | `! cmd` | Explicit `if` |
| `local`/`export`/`readonly` with substitution | `local x=$(false)` | `local x; x=$(false)` |
| Command substitution body | `x=$(false; echo hi)` | `inherit_errexit`, one command per `$()` |
| Background jobs | `cmd &` then `wait` | `wait "$pid"` per job |
| Arithmetic that evaluates to 0 kills you | `((i++))` with `i=0` | `i=$((i + 1))` |
| Expected "no match" kills you | `n=$(grep -c x f)` | `\|\| true` on that command |
| Pipeline SIGPIPE under `pipefail` | `cmd \| head -1` returns 141 | Read all input, or accept 141 |

## trap and signals

```bash
trap 'rm -rf -- "$tmp"' EXIT            # single quotes: expands when the trap runs
trap cleanup EXIT                        # or a function
trap 'on_err $LINENO' ERR                # with set -E
trap 'exit 130' INT; trap 'exit 143' TERM   # stop loops cleanly; EXIT trap still runs
trap '' HUP                              # ignore a signal
trap - INT                               # restore default
trap -p                                  # list traps
kill -l                                  # list signal names/numbers
kill -TERM "$pid"; wait "$pid"           # polite stop, then reap
kill -0 "$pid" 2>/dev/null && echo alive # existence check, sends nothing
```

| Signal | No. | Default | Trappable | Typical source |
|--------|-----|---------|-----------|----------------|
| HUP | 1 | terminate | yes | terminal closed, "reload" convention |
| INT | 2 | terminate | yes | Ctrl+C |
| QUIT | 3 | core dump | yes | Ctrl+\ |
| KILL | 9 | terminate | **no** | `kill -9`, OOM killer |
| TERM | 15 | terminate | yes | `kill`, systemd, Docker, Kubernetes |
| STOP | 19 | stop | **no** | `kill -STOP` |
| TSTP | 20 | stop | yes | Ctrl+Z |
| USR1/USR2 | 10/12 | terminate | yes | app-defined (log reopen) |
| CHLD | 17 | ignore | yes | child exited |

Container entrypoints: make the app PID 1 with `exec app "$@"`, or forward signals: `app & pid=$!; trap 'kill -TERM $pid' TERM INT; wait $pid; wait $pid`.

## getopts

```bash
usage() { echo "usage: ${0##*/} -e ENV [-v VERSION] [-n] [-h] [--] [args...]"; }
env="" version=latest dry_run=0
while getopts ':e:v:nh' opt; do        # leading ':' = we handle errors; 'e:' = takes a value
  case $opt in
    e) env=$OPTARG ;;
    v) version=$OPTARG ;;
    n) dry_run=1 ;;
    h) usage; exit 0 ;;
    :) echo "option -$OPTARG requires a value" >&2; usage >&2; exit 2 ;;
    \?) echo "unknown option -$OPTARG" >&2; usage >&2; exit 2 ;;
  esac
done
shift $((OPTIND - 1))                  # "$@" is now the remaining positional args
[[ $env =~ ^(dev|staging|prod)$ ]] || { usage >&2; exit 2; }
```

`getopts` is a POSIX builtin with short options only. Long options: a manual `while (( $# )); do case $1 in --env) env=$2; shift 2 ;; --env=*) env=${1#*=}; shift ;; --) shift; break ;; esac; done`. GNU `getopt` (external) supports long options but is not available by default on macOS.

---

## grep

| Task | Command |
|------|---------|
| Extended regex | `grep -E 'error\|fatal' app.log` |
| Case-insensitive, line numbers | `grep -in 'timeout' app.log` |
| Invert (exclude) | `grep -v '^#' conf` |
| Count matching lines | `grep -c ' 500 ' access.log` |
| Only the matching part | `grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' access.log` |
| Fixed string, whole line | `grep -qxF -- "$user" allowed.txt` |
| Whole word | `grep -w 'id' file` |
| Quiet, for `if` | `if grep -q 'READY' log; then` |
| Recursive, include glob | `grep -rn --include='*.yaml' 'image:' k8s/` |
| List matching files, NUL-separated | `grep -rlZ 'old.example.com' /etc/nginx` |
| Context lines | `grep -C 3 'panic' app.log` (`-A` after, `-B` before) |
| Stop after first match | `grep -m 1 'Started' app.log` |
| Treat binary as text | `grep -a 'ERROR' app.log` |
| Patterns from a file | `grep -Ff patterns.txt app.log` |
| Search compressed logs | `zgrep 'req-8f2a' /var/log/app/*.gz` |
| PCRE (GNU only) | `grep -oP 'version=\K[0-9.]+' file` |

Exit codes: 0 match, 1 no match, 2 error. Under `set -e`/`pipefail`, "no match" is a failure unless you handle it.

## sed

| Task | Command |
|------|---------|
| Replace all on each line | `sed 's/old/new/g' file` |
| Edit in place with backup (GNU and BSD) | `sed -i.bak 's/old/new/g' file` |
| In place, no backup (GNU) | `sed -i 's/old/new/g' file` |
| Different delimiter for paths | `sed 's\|/usr/local\|/opt\|g' file` |
| ERE and back-references | `sed -E 's/^([a-z]+)=(.*)$/\2=\1/' file` |
| Print lines 10–20 | `sed -n '10,20p' file` |
| Print matching lines | `sed -n '/ERROR/p' file` |
| Delete comments and blank lines | `sed -e '/^\s*#/d' -e '/^\s*$/d' file` |
| Delete a line range | `sed '5,10d' file` |
| Replace only on matching lines | `sed '/^server_name/s/old/new/' file` |
| Insert / append a line (GNU) | `sed '/^\[main\]/a key=value' file`, `sed '1i # header' file` |
| Print between two patterns | `sed -n '/BEGIN CERT/,/END CERT/p' file` |
| Strip CRLF | `sed -i 's/\r$//' script.sh` |
| Strip ANSI colour codes | `sed -E 's/\x1b\[[0-9;]*m//g' log` |
| Use a shell variable | `sed "s/^image: .*/image: ${IMAGE//\//\\/}/" f` (escape `/`, `&`) |

For YAML or JSON, use `yq` or `jq`, not `sed`.

## awk

```bash
awk '{print $1, $NF}' file                       # first and last field
awk -F: '$3 >= 1000 {print $1}' /etc/passwd     # field separator + condition
awk -F, -v OFS='\t' '{print $2, $1}' in.csv     # output separator
awk 'NR==1 {next} {print}' file                 # skip header
awk 'NR % 2 == 0' file                          # every second line
awk '{s += $5} END {print s}' file              # sum a column
awk '{s += $NF; n++} END {if (n) printf "%.2f\n", s/n}' file   # average
awk '$9 >= 500 {c[$7]++} END {for (k in c) print c[k], k}' access.log | sort -rn | head   # count by key
awk '!seen[$0]++' file                          # dedupe keeping order (no sort)
awk 'length > 200' file                         # long lines
awk -v t="$THRESH" '$5+0 > t+0' file            # pass shell vars with -v
awk '/START/,/END/' file                        # range between patterns
awk 'BEGIN { exit !(4.75 > 4.5) }'              # float comparison as exit status
awk '{ gsub(/[0-9]+ms/, ""); print }' file      # substitute (POSIX: sub, gsub, match, substr, split)
awk -F'"' '{print $2}' access.log               # the quoted request line
awk 'FNR==NR {a[$1]; next} $1 in a' ids.txt data.txt   # join: lines in data whose key is in ids
```

| Variable | Meaning |
|----------|---------|
| `$0`, `$1..$NF` | Whole line, fields |
| `NR`, `FNR` | Line number overall, per file |
| `NF` | Number of fields |
| `FS`, `OFS` | Input / output field separator |
| `RS`, `ORS` | Input / output record separator |
| `FILENAME` | Current file |

`gensub`, `asort`, `strftime` and `PROCINFO` are gawk extensions; Debian/Ubuntu default `awk` is mawk, Alpine's is BusyBox.

## cut, sort, uniq, tr and friends

| Task | Command |
|------|---------|
| Field by delimiter | `cut -d, -f2,4 file.csv` |
| Characters | `cut -c1-10 file` |
| Sort numeric, reverse | `sort -rn` |
| Sort by column 3 numeric, comma-separated | `sort -t, -k3,3n file.csv` |
| Sort human sizes / versions | `sort -h`, `sort -V` |
| Unique lines | `sort -u` |
| Count duplicates (needs sorted input) | `sort \| uniq -c \| sort -rn` |
| Only duplicated / only unique lines | `uniq -d`, `uniq -u` |
| Lines in A not in B (both sorted) | `comm -23 <(sort a) <(sort b)` |
| Join on a key | `join -t, <(sort -t, -k1,1 a) <(sort -t, -k1,1 b)` |
| Translate / delete / squeeze | `tr 'a-z' 'A-Z'`, `tr -d '\r'`, `tr -s ' '` |
| Count lines / words / bytes | `wc -l`, `wc -w`, `wc -c` |
| Merge lines side by side | `paste -d, a b`, `paste -sd, list` (join lines with commas) |
| Align columns | `column -t -s,` |
| First / last lines, follow | `head -n 20`, `tail -n 50`, `tail -F log` |
| Reverse lines | `tac file` |
| Shuffle / sample | `shuf -n 5 file` |
| Split a big file | `split -l 100000 -d big.log part_` |
| Copy stdout to a file too | `cmd \| tee -a out.log` |
| Human-readable numbers | `numfmt --to=iec 1073741824` (1.0G) |
| Byte-order sorting, faster and stable | `LC_ALL=C sort` |

## find

| Task | Command |
|------|---------|
| By name (glob), case-insensitive | `find /etc -iname '*.conf' -type f` |
| Not descending into other filesystems | `find / -xdev -type f -size +500M` |
| Modified more than 7 days ago | `find /var/log -type f -mtime +7` |
| Modified in the last 30 minutes | `find . -mmin -30` |
| Newer than a reference file | `find . -newer /tmp/stamp` |
| Depth control | `find . -maxdepth 1 -mindepth 1` |
| Prune a directory | `find . -path ./node_modules -prune -o -name '*.js' -print` |
| Empty files / dirs | `find . -empty` |
| By owner / permission | `find / -xdev -nouser`, `find / -xdev -perm -4000 -type f` (setuid) |
| World-writable files | `find / -xdev -type f -perm -0002` |
| Delete (put `-delete` last) | `find /tmp/app -type f -mtime +2 -delete` |
| Run once per batch | `find . -name '*.log' -exec gzip -- {} +` |
| Run once per file | `find . -name '*.sh' -exec shellcheck {} \;` |
| Custom output | `find . -type f -printf '%TY-%Tm-%Td %s %p\n'` (GNU) |
| NUL-separated for pipes | `find . -type f -print0 \| xargs -0 ...` |
| Sizes per directory | `du -xh --max-depth=1 /var \| sort -rh \| head` |

Expressions are evaluated left to right: `find . -delete -name '*.tmp'` deletes **everything**.

## xargs

| Task | Command |
|------|---------|
| NUL-safe | `find . -name '*.tmp' -print0 \| xargs -0 -r rm -f --` |
| Do nothing on empty input (GNU) | `xargs -r` |
| N arguments per command | `xargs -n 1` |
| Placeholder | `xargs -I{} cp {} /backup/` |
| Parallel | `xargs -P 8 -n 1 ./process.sh` |
| From a file | `xargs -a hosts.txt -P 10 -I{} ssh -n {} uptime` |
| Show commands as run | `xargs -t` |
| Call a shell function | `export -f fn; xargs -I{} bash -c 'fn "$1"' _ {}` |

Exit status 123 means at least one invocation failed; 124 means a command exited 255; 125 killed by signal; 126/127 not runnable / not found.

## jq

```bash
jq . file.json                                   # pretty-print, validate
jq -r '.items[].metadata.name'                   # raw strings, one per line
jq -r '.items[]? | .name'                        # ? = no error if missing/null
jq '.spec.replicas // 1'                         # default for null/missing
jq -e '.status == "ok"' >/dev/null               # exit 1 if false/null (for if)
jq -c '.[]'                                      # compact, one object per line (JSON Lines)
jq -s 'length' events.jsonl                      # slurp JSON Lines into an array
jq 'keys'                                        # inspect structure: top-level keys
jq '.items | length'                             # count elements
jq '.items | type'                               # array, object, null ...
jq '.[] | select(.state == "running" and .cpu > 80)'
jq 'map(select(.tags.Owner == null)) | length'
jq -r '.[] | [.name, .ip, .state] | @tsv'        # TSV (also @csv, @json, @base64, @uri, @sh)
jq -r '.[] | "\(.name)=\(.value)"'               # string interpolation
jq 'group_by(.zone) | map({zone: .[0].zone, n: length})'
jq 'to_entries | map("\(.key)=\(.value)") | .[]' # object -> key=value lines
jq '.metadata.labels.team = "ops"'               # set a field
jq 'del(.metadata.managedFields)'                # remove a field
jq --arg ns "$NS" '.items[] | select(.metadata.namespace == $ns)'   # shell string in
jq --argjson n "$COUNT" '.replicas = $n'         # shell number/JSON in
jq -n --arg msg "$MSG" '{text: $msg}'            # build JSON safely (Slack payload)
jq -n --rawfile body notes.txt '{body: $body}'   # file contents as a string
jq -s '.[0] * .[1]' base.json override.json      # deep merge
```

## yq (mikefarah v4)

```bash
yq '.image.tag' values.yaml                      # read
yq -i '.image.tag = "1.4.2"' values.yaml         # edit in place (comments kept)
TAG=1.4.2 yq -i '.image.tag = strenv(TAG)' values.yaml   # value from env
yq '.spec.template.spec.containers[].image' deploy.yaml
yq 'select(.kind == "Deployment") | .metadata.name' all.yaml   # multi-document files
yq -i 'del(.metadata.annotations)' deploy.yaml
yq -o=json '.' values.yaml | jq ...              # YAML -> JSON
yq -P '.' file.json                              # JSON -> pretty YAML
yq ea '. as $i ireduce ({}; . * $i)' base.yaml prod.yaml   # merge files (eval-all)
yq '.data | keys' configmap.yaml
yq --version                                     # check it is mikefarah v4, not python-yq
```

## cron syntax

```text
┌───────── minute        0-59
│ ┌─────── hour          0-23
│ │ ┌───── day of month  1-31
│ │ │ ┌─── month         1-12 or jan-dec
│ │ │ │ ┌─ day of week   0-7 (0 and 7 = Sunday) or sun-sat
│ │ │ │ │
* * * * *  command                    # user crontab (crontab -e)
* * * * *  user  command              # /etc/crontab and /etc/cron.d/* have a USER field
```

| Schedule | Expression |
|----------|------------|
| Every 5 minutes | `*/5 * * * *` |
| Every hour at :17 | `17 * * * *` |
| Daily at 02:30 | `30 2 * * *` |
| Weekdays at 09:00 | `0 9 * * 1-5` |
| Every 15 min, 09:00–17:45, Mon–Fri | `*/15 9-17 * * 1-5` |
| First day of the month, 03:00 | `0 3 1 * *` |
| Sundays at 04:00 | `0 4 * * 0` |
| Quarterly | `0 0 1 1,4,7,10 *` |
| At boot / shortcuts | `@reboot`, `@hourly`, `@daily`, `@weekly`, `@monthly` |

| Command | Purpose |
|---------|---------|
| `crontab -e`, `crontab -l`, `crontab -l -u deploy` | Edit / list / list another user's |
| `crontab -r` | Delete your whole crontab (no confirmation) |
| `grep CRON /var/log/syslog`, `journalctl -u cron` | Did it run? (`crond` on RHEL) |
| `run-parts --test /etc/cron.daily` | What `cron.daily` will run (Debian skips names with dots) |

Gotchas: cron uses `/bin/sh` with `PATH=/usr/bin:/bin` and no profile; escape `%` as `\%`; when both day-of-month and day-of-week are restricted, either matching triggers the job; output is mailed to `MAILTO` or discarded; files in `/etc/cron.d` must end with a newline and be root-owned, not writable by others. `CRON_TZ=` is supported by cronie (RHEL family); check `man 5 crontab` before relying on it elsewhere.

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""
*/5 * * * * opskit flock -n /run/lock/health.lock /opt/opskit/bin/opskit-health >> /var/log/opskit/health.log 2>&1
30 2 * * *  root   /opt/backup.sh > "/var/log/backup-$(date +\%F).log" 2>&1
```

## systemd timers

```ini
# /etc/systemd/system/job.service
[Unit]
Description=Nightly job
OnFailure=notify@%n.service

[Service]
Type=oneshot
User=svc
ExecStart=/opt/job/bin/run.sh
EnvironmentFile=-/etc/job/job.env
TimeoutStartSec=1h
Nice=10
ProtectSystem=strict
ReadWritePaths=/var/lib/job
PrivateTmp=yes
NoNewPrivileges=yes
```

```ini
# /etc/systemd/system/job.timer
[Unit]
Description=Run job nightly

[Timer]
OnCalendar=*-*-* 02:30:00
RandomizedDelaySec=15min
Persistent=true

[Install]
WantedBy=timers.target
```

| `OnCalendar=` | Meaning |
|---------------|---------|
| `hourly`, `daily`, `weekly`, `monthly` | Shorthands (`daily` = `*-*-* 00:00:00`) |
| `*-*-* 02:30:00` | Every day at 02:30 |
| `*:0/15` | Every 15 minutes |
| `Mon..Fri *-*-* 09:00:00` | Weekdays at 09:00 |
| `Sat *-*-01..07 04:00:00` | First Saturday of the month (weekday AND date) |
| `*-*-01 03:00:00` | First of the month |
| `*-*-* 02:30:00 Asia/Kolkata` | In a specific time zone (systemd 235+) |

| Monotonic option | Relative to |
|------------------|-------------|
| `OnBootSec=5min` | Boot |
| `OnUnitActiveSec=10min` | Last activation of the service (needs a first run: pair with `OnBootSec=`) |
| `OnUnitInactiveSec=10min` | Last time the service finished |
| `OnActiveSec=` | Timer activation |

| Command | Purpose |
|---------|---------|
| `systemctl daemon-reload` | After editing unit files |
| `systemctl enable --now job.timer` | Enable the **timer**, not the service |
| `systemctl list-timers --all` | NEXT / LAST run of every timer |
| `systemctl start job.service` | Run now, outside the schedule |
| `systemctl status job.service` | Last result, exit status |
| `journalctl -u job.service -n 50 --no-pager` | Logs of the job |
| `journalctl -u job -f`, `--since "1 hour ago"` | Follow / time-filter |
| `systemd-analyze calendar '*:0/15' --iterations=3` | Validate a calendar expression |
| `systemd-analyze verify job.service job.timer` | Lint unit files |
| `systemd-analyze security job.service` | Sandboxing score |
| `systemd-run --on-calendar='*:0/5' /opt/x.sh` | Transient timer, no files |
| `systemctl reset-failed job.service` | Clear a failed state |

## Locking, timeouts and retries

```bash
# One instance at a time
exec 9>/run/lock/job.lock; flock -n 9 || { echo "already running"; exit 0; }
flock -n /run/lock/job.lock /opt/job.sh              # wrapper form (cron)
flock -w 30 /run/lock/job.lock cmd                   # wait up to 30 s

# Timeouts
timeout 30 cmd                                       # TERM after 30 s, exit 124
timeout --kill-after=5 30 cmd                        # then KILL after 5 more
curl -fsS --connect-timeout 3 --max-time 10 "$url"
ssh -o ConnectTimeout=5 -o BatchMode=yes host cmd
read -r -t 10 answer || answer=default

# Retry with exponential backoff and jitter
retry() {
  local max=$1 n=1; shift
  until "$@"; do
    (( n >= max )) && return 1
    sleep $(( (2 ** n) + RANDOM % 3 )); n=$((n + 1))
  done
}
retry 5 curl -fsS "$url" -o out.json

# Wait for a dependency
timeout 120 bash -c 'until pg_isready -h db -q; do sleep 2; done'
timeout 60 bash -c 'until </dev/tcp/db/5432; do sleep 1; done' 2>/dev/null

# Atomic file replacement (same filesystem)
tmp=$(mktemp "${out}.XXXXXX"); generate > "$tmp" && mv -f -- "$tmp" "$out"

# Bounded parallelism in pure Bash (4.3+)
for h in "${hosts[@]}"; do
  (( $(jobs -rp | wc -l) >= 10 )) && wait -n
  check "$h" &
done; wait
```

## ShellCheck and bats

| Code | Problem | Fix |
|------|---------|-----|
| SC2086 | Unquoted variable: splitting and globbing | `"$var"` |
| SC2046 | Unquoted `$(...)` | `"$(...)"` or `mapfile` |
| SC2155 | `local x=$(cmd)` masks the exit status | Declare, then assign |
| SC2164 | `cd dir` without a failure check | `cd dir \|\| exit` |
| SC2045 | Iterating over `ls` output | Use a glob |
| SC2064 | Trap string in double quotes expands now | Single quotes |
| SC2034 | Variable assigned but never used | Remove it, or `export` / `# shellcheck disable` with reason |
| SC2181 | `if [ $? -ne 0 ]` | `if ! cmd; then` |
| SC2068 | Unquoted `$@` | `"$@"` |
| SC2002 | Useless `cat` | `cmd < file` |
| SC1091 | Cannot follow `source` | `shellcheck -x`, `.shellcheckrc` `source-path=SCRIPTDIR` |
| SC2016 | `$` in single quotes will not expand | Intended? add a reasoned disable |
| SC1017 | Literal carriage return (CRLF) | `sed -i 's/\r$//'` |

```bash
shellcheck -x -S warning bin/*.sh          # severity threshold: error, warning, info, style
shellcheck -f gcc bin/*.sh                 # compiler-style output for editors and CI
# shellcheck disable=SC2016  # reason goes here, on the same line
```

```bash
#!/usr/bin/env bats
# test/backup.bats
setup() {
  export PATH="$BATS_TEST_DIRNAME/stubs:$PATH"   # stub aws, curl, systemctl
  export STUB_LOG="$BATS_TEST_TMPDIR/calls.log"
}

@test "fails with exit 2 on unknown option" {
  run ./bin/backup.sh -z
  [ "$status" -eq 2 ]
  [[ $output == *"unknown option"* ]]
}

@test "uploads to the right key" {
  run ./bin/backup.sh
  [ "$status" -eq 0 ]
  grep -q 's3 cp .* s3://bucket/hosts/' "$STUB_LOG"
}
```

`bats test/` runs every `.bats` file; `bats --count test/` counts tests; `run -127 cmd` asserts an exit code (needs `bats_require_minimum_version 1.5.0`); a bare `! cmd` mid-test does not fail it.

## Bash vs POSIX sh

| Bash | POSIX `sh` (dash, BusyBox ash) |
|------|-------------------------------|
| `[[ $a == b* ]]` | `case $a in b*) ... ;; esac` |
| `[[ $v =~ re ]]` | `printf '%s\n' "$v" \| grep -Eq 're'` |
| `arr=(a b)`, `"${arr[@]}"` | `set -- a b`, `"$@"` |
| `declare -A` | Not available: files or `key=value` lines |
| `${var//a/b}` | `printf '%s' "$var" \| sed 's/a/b/g'` |
| `${var^^}` | `printf '%s' "$var" \| tr '[:lower:]' '[:upper:]'` |
| `$(< file)` | `$(cat file)` |
| `<(cmd)` | Temp file or `mkfifo` |
| `cmd &> f` | `cmd > f 2>&1` |
| `(( i++ ))` | `i=$((i + 1))` |
| `function f { }` | `f() { }` |
| `echo -e`, `$'\t'` | `printf '\t'` |
| `source file` | `. ./file` |
| `read -a`, `mapfile` | Loop with `read` |
| `local` | Not POSIX, but supported by dash and BusyBox |
| `set -o pipefail` | In POSIX since 2024 (Issue 8) and in BusyBox ash, but dash 0.5.12 (Ubuntu 24.04) fails with `set: Illegal option -o pipefail` |

Check: `shellcheck -s sh script`, `checkbashisms script`, and run it with `dash script`.

## Top 15 interview one-liners

```bash
# 1. Top 10 client IPs in an access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -n 10

# 2. Endpoints returning the most 5xx
awk '$9 ~ /^5/ {print $7}' access.log | sort | uniq -c | sort -rn | head

# 3. Biggest directories on this filesystem
du -xh --max-depth=1 / 2>/dev/null | sort -rh | head -n 15

# 4. Delete files older than 14 days, NUL-safe
find /var/log/app -type f -name '*.gz' -mtime +14 -print0 | xargs -0 -r rm -f --

# 5. Replace a hostname in every config file that contains it
grep -rlZ 'old.example.com' /etc/nginx | xargs -0 -r sed -i.bak 's/old\.example\.com/new.example.com/g'

# 6. Deleted files still holding disk space
lsof -nP +L1 | awk 'NR==1 || $7 > 100*1024*1024'

# 7. Retry a flaky command 5 times with backoff
for i in {1..5}; do curl -fsS "$URL" -o out.json && break; sleep $((2 ** i)); done

# 8. Wait until a port accepts connections (max 60 s)
timeout 60 bash -c 'until </dev/tcp/127.0.0.1/5432; do sleep 1; done' 2>/dev/null

# 9. Pods that are not Running, as namespace/name
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase != "Running") | "\(.metadata.namespace)/\(.metadata.name)"'

# 10. Run a command on many hosts, 10 at a time
xargs -a hosts.txt -P 10 -I{} ssh -n -o ConnectTimeout=5 {} 'uptime' 2>&1

# 11. Days until a TLS certificate expires
end=$(echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2); echo $(( ($(date -d "$end" +%s) - $(date +%s)) / 86400 ))

# 12. Compare installed packages on two servers
diff <(ssh web1 'dpkg-query -W -f="\${Package}\n"' | sort) <(ssh web2 'dpkg-query -W -f="\${Package}\n"' | sort)

# 13. HTTP status distribution of 50 requests
for i in {1..50}; do curl -s -o /dev/null -w '%{http_code}\n' "$URL"; done | sort | uniq -c

# 14. Run a job only if no other copy is running
flock -n /run/lock/backup.lock /opt/backup.sh || echo "skipped: already running"

# 15. Follow a log and alert on a pattern, line by line
tail -Fn0 /var/log/app/app.log | grep --line-buffered 'OutOfMemory' | while read -r line; do logger -t oom-watch "$line"; done
```

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
