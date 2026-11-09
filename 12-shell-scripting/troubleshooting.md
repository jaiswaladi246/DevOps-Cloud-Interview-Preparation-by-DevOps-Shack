# Shell Scripting — Troubleshooting Drills

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

> Real Shell Scripting errors and symptoms: likely causes, diagnosis, fix and prevention · 100 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Shell Scripting](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Troubleshooting: syntax, parsing and interpreter errors](#troubleshooting-syntax-parsing-and-interpreter-errors) — Q1–Q20
- [Troubleshooting: quoting, expansion and tool-input bugs](#troubleshooting-quoting-expansion-and-tool-input-bugs) — Q21–Q40
- [Troubleshooting: strict mode, exit codes, pipelines and traps](#troubleshooting-strict-mode-exit-codes-pipelines-and-traps) — Q41–Q60
- [Troubleshooting: cron and systemd timers](#troubleshooting-cron-and-systemd-timers) — Q61–Q80
- [Troubleshooting: tools, portability, permissions and environment](#troubleshooting-tools-portability-permissions-and-environment) — Q81–Q100

---

## Troubleshooting: syntax, parsing and interpreter errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `./deploy.sh` fails with `bash: ./deploy.sh: /bin/bash^M: bad interpreter: No such file or directory` (or, on Bash 5.2, `cannot execute: required file not found`). 🟢

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

### Q2. `bash: ./backup.sh: Permission denied` right after cloning the repo. 🟢

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

### Q3. `deploy.sh: line 14: syntax error near unexpected token `fi'` 🟢

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

### Q4. `install.sh: line 87: syntax error: unexpected end of file` 🟢

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

### Q5. `warning: here-document at line 12 delimited by end-of-file (wanted `EOF')` and the rest of the script never runs. 🟢

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

### Q6. `bash: [: missing `]'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `[` is a command and `]` must be its last, separate argument. Missing space (`[ -f "$f"]`), using `&&` inside `[ ]` (`[ -f a && -f b ]`), or an unquoted variable expanding to nothing in the wrong place.

**Diagnose →** `bash -x script.sh` shows the exact `[` command after expansion.

**Fix →** `[ -f "$f" ]`, `[ -f a ] && [ -f b ]`, or use Bash's `[[ -f a && -f b ]]`.

**Prevent →** Prefer `[[ ]]` in Bash scripts; ShellCheck catches missing spaces (SC1020) and `&&` inside `[ ]` (SC2107).

</details>

### Q7. A script prints `/bin/sh: 1: [[: not found` and then continues as if the test were false. 🟢

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

### Q8. Under `sh`, the script fails with `Syntax error: "(" unexpected`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash-only syntax parsed by dash: an array assignment `hosts=(a b c)`, `function name() {`, or `<(...)` process substitution (which gives `Syntax error: redirection unexpected` instead).

**Diagnose →** `shellcheck -s sh script.sh` lists each bashism with the line number; `checkbashisms script.sh` does the same.

**Fix →** Run it with Bash (fix the shebang or invocation), or port: arrays to `set -- a b c` or newline-separated strings, `function f()` to `f()`, process substitution to a temp file.

**Prevent →** Decide per script: Bash with an explicit shebang, or POSIX sh checked with `shellcheck -s sh`.

</details>

### Q9. `script.sh: 12: Bad substitution` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A Bash-only parameter expansion run by dash: `${var//a/b}`, `${var^^}`, `${var:0:3}`, `${!name}`, `${arr[@]}`. Or a typo like `${var-name with space}` or `$(var)` confusion.

**Diagnose →** Check the shell (`ps -p $$ -o comm=` inside the script, or the shebang) and line 12.

**Fix →** Run with Bash, or use POSIX alternatives: `$(printf '%s' "$var" | sed 's/a/b/g')`, `$(printf '%s' "$var" | tr '[:lower:]' '[:upper:]')`, `${var%%.*}`/`${var#*-}` (those are POSIX).

**Prevent →** `shellcheck -s sh` flags these as SC3060 and friends.

</details>

### Q10. `deploy.sh: line 3: APP_ENV: command not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Spaces around `=` in an assignment: `APP_ENV = prod` runs a command named `APP_ENV`. Variants: `APP_ENV= prod` (sets `APP_ENV` to empty for the command `prod`), or a leading `$` (`$APP_ENV=prod`).

**Diagnose →** `sed -n 3p deploy.sh`.

**Fix →** `APP_ENV=prod` with no spaces.

**Prevent →** ShellCheck reports SC1068 ("Don't put spaces around the = in assignments").

</details>

### Q11. Running `bash script.sh` prints `script.sh: line 2: $'\r': command not found` on every blank line. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CRLF line endings again. Running through `bash` avoids the shebang error, but every blank line now contains just `\r`, which Bash tries to run as a command. Other lines may fail subtly (`cd /srv/app\r` → `No such file or directory`).

**Diagnose →** `cat -A script.sh | head` shows `^M$` at line ends.

**Fix →** `sed -i 's/\r$//' script.sh`.

**Prevent →** `.gitattributes` with `*.sh text eol=lf`; an `.editorconfig` with `end_of_line = lf`.

</details>

### Q12. A script that uses `done < <(kubectl get pods -o name)` fails with `Syntax error: redirection unexpected` in CI but works locally. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI step runs the script with `sh` (dash or BusyBox ash), which does not support process substitution. Common in GitLab CI `script:` blocks on Alpine images, or `sh ./script.sh` in a Makefile.

**Diagnose →** Print the shell in the job: `echo "$0"; readlink -f /proc/$$/exe`.

**Fix →** Invoke with Bash (`bash ./script.sh`, install `bash` in Alpine images with `apk add --no-cache bash`), or replace the construct: `kubectl get pods -o name > "$tmp"; while ...; done < "$tmp"`.

**Prevent →** Pin the interpreter via shebang and execute scripts directly, never through `sh`.

</details>

### Q13. `bash: local: can only be used in a function` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `local` used at the top level of a script, often after moving code out of a function, or in a file that is `source`d at top level.

**Diagnose →** Find the line in the error and check whether it is inside a function body.

**Fix →** Remove `local` for top-level variables, or wrap the code in a function (`main() { ...; }; main "$@"`), which is better practice anyway.

**Prevent →** Keep all logic in functions with a single `main "$@"` call at the bottom.

</details>

### Q14. On a Mac, `declare -A config` fails with `declare: -A: invalid option`. 🟡

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

### Q15. `mapfile: command not found` on one machine but not another. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bash older than 4.0 (macOS 3.2), or the script is not running under Bash at all (zsh, dash, ash), where `mapfile`/`readarray` do not exist.

**Diagnose →** `echo "$BASH_VERSION"` inside the script (empty means not Bash).

**Fix →** Run with Bash 4+; for portability, replace with a read loop: `arr=(); while IFS= read -r l; do arr+=("$l"); done < <(cmd)`.

**Prevent →** Version guard at the top of scripts that use Bash 4 features.

</details>

### Q16. In your terminal, `echo "Deploy done!"` fails with `bash: !": event not found`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interactive history expansion: `!` inside double quotes refers to history. It does not happen in scripts (history expansion is off in non-interactive shells), which is why the same line works in a script.

**Diagnose →** `echo $-` shows `H` when history expansion is on.

**Fix →** Use single quotes (`echo 'Deploy done!'`), escape it (`\!`), put `!` at the end outside quotes, or `set +H` in the interactive shell.

**Prevent →** Use single quotes for literal strings when testing interactively.

</details>

### Q17. A script with the shebang `#!/usr/bin/env bash -e` fails with `/usr/bin/env: 'bash -e': No such file or directory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Linux passes everything after the interpreter path as **one** argument, so `env` looks for a program named `bash -e`.

**Diagnose →** The error message shows the combined string in quotes; newer coreutils adds `use -[v]S to pass options in shebang lines`.

**Fix →** Keep the shebang to `#!/usr/bin/env bash` and put options on the next line: `set -Eeuo pipefail`. (`#!/usr/bin/env -S bash -e` works with coreutils 8.30+, but not everywhere.)

**Prevent →** Treat the shebang as interpreter only; all options go in `set`.

</details>

### Q18. A Docker container exits immediately with `exec /entrypoint.sh: exec format error`. The script runs fine with `sh /entrypoint.sh`. 🟡

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

### Q19. `[[ $line =~ ^ERROR (.*)$ ]]` fails with `syntax error in conditional expression`. 🔴

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

### Q20. A cron job intermittently fails with random syntax errors like `line 212: syntax error near unexpected token `}'`, but only on deploy days. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The deploy overwrites the script **in place** while a cron run is executing it. Bash reads scripts incrementally by byte offset, so the running shell continues reading the new file from the old offset and hits garbage.

**Diagnose →** Correlate failure times with deploy times; check how the deploy writes files (`cp` over the target modifies the same inode; `mv` replaces it).

**Fix →** Deploy scripts atomically: write to a temp file in the same directory and `mv` it over the original (the running shell keeps the old inode open). Also wrap script bodies in `main() { ...; }; main "$@"` so the whole file is parsed before execution.

**Prevent →** Use `install` or `mv`-based deployment, never `cp` or `>` onto a live script.

</details>

## Troubleshooting: quoting, expansion and tool-input bugs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q21. `bash: [: too many arguments` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unquoted variable inside `[ ]` contains spaces or glob characters and expands to several words: `[ $status = "active" ]` with `status="not found"`. Also happens with `[ -f $file ]` when `$file` matches multiple files via globbing.

**Diagnose →** `bash -x` shows the expanded test, e.g. `+ '[' not found = active ']'`.

**Fix →** Quote: `[ "$status" = active ]`, or use `[[ $status == active ]]`.

**Prevent →** Quote every expansion; ShellCheck SC2086.

</details>

### Q22. `bash: [: =: unary operator expected` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The left-hand variable is empty and unquoted, so `[ $env = prod ]` becomes `[ = prod ]`. Usually a missing argument, a failed command substitution, or a typo in the variable name.

**Diagnose →** `bash -x` shows `+ '[' = prod ']'`. Check where the variable is set.

**Fix →** `[ "$env" = prod ]` or `[[ $env == prod ]]`, and validate required values early with `: "${env:?env is required}"`.

**Prevent →** `set -u` catches unset (not empty) variables; quoting handles empty ones.

</details>

### Q23. A disk check fails with `[: 93%: integer expression expected` 🟢

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

### Q24. `rm $file` fails with `rm: cannot remove 'monthly': No such file or directory` and `rm: cannot remove 'report.csv'`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `file="monthly report.csv"` was word-split into two arguments because it was not quoted. A name with `*` would be worse: it would expand to other files.

**Diagnose →** `printf '[%s]\n' $file` shows the split.

**Fix →** `rm -- "$file"`.

**Prevent →** Quote every expansion; test scripts with filenames containing spaces, quotes and leading dashes.

</details>

### Q25. A cleanup script logs `cp: cannot stat '/var/log/app/*.log': No such file or directory` on quiet days. 🟢

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

### Q26. `echo "start" > $logfile` fails with `bash: $logfile: ambiguous redirect` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The redirect target expanded to zero words (variable empty or unset) or to several words (path contains spaces), so Bash cannot pick one file.

**Diagnose →** `declare -p logfile` to see the value; check where it is set (typo, a config not loaded, a failed `$(...)`).

**Fix →** Quote and validate: `logfile=${LOG_FILE:?LOG_FILE not set}; echo start >> "$logfile"`.

**Prevent →** Quote redirect targets and use `set -u` so unset variables fail loudly.

</details>

### Q27. `hosts[${#hosts[@]}-1]` style code fails with `bash: hosts: bad array subscript`. 🟡

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

### Q28. Sourcing a shared library a second time fails with `bash: CONFIG_DIR: readonly variable`, and the script exits under `set -e`. 🟡

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

### Q29. A report script fails only at 08:00 and 09:00 with `bash: 08: value too great for base (error token is "08")`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `hour=$(date +%H)` gives `08` and `09`, and Bash arithmetic treats a leading zero as octal, where 8 and 9 are invalid. At 00–07 it silently works, which is why it went unnoticed.

**Diagnose →** `bash -c 'h=08; echo $((h + 1))'` reproduces it.

**Fix →** Force base 10: `$(( 10#$hour + 1 ))`, or ask for an unpadded value (`date +%-H` with GNU date).

**Prevent →** Use `10#` whenever doing arithmetic on dates, months or zero-padded IDs; test with edge values.

</details>

### Q30. A loop counts errors, but after the loop the count is always 0. 🟡

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

### Q31. A script that reads `hosts.txt` and SSHes to each host only processes the first host, with no error. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ssh` inherits the loop's stdin and reads the rest of `hosts.txt` as input for the remote command, so the loop ends after one iteration. `docker exec -i`, `kubectl exec -i` and `mysql` do the same.

**Diagnose →** Add `echo "next: $host"` at the top of the loop; it prints once.

**Fix →** `ssh -n "$host" ...` (stdin from `/dev/null`), `ssh ... < /dev/null`, or read the list on another descriptor: `while read -r host <&3; do ...; done 3< hosts.txt`.

**Prevent →** Always use `ssh -n` (or `-o BatchMode=yes` plus `< /dev/null`) inside read loops.

</details>

### Q32. `sed -i "s/^image:.*/image: $IMAGE/" deploy.yaml` fails with `sed: -e expression #1, char 32: unknown option to `s'` 🟡

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

### Q33. `awk "{print $2}" file` prints whole lines instead of the second column. 🟡

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

### Q34. A `while read` loop silently skips the last line of a config file. 🟢

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

### Q35. `gzip /var/log/app/*.log` fails with `bash: /usr/bin/gzip: Argument list too long` 🟡

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

### Q36. `find ... | xargs rm` fails with `xargs: unmatched single quote; by default quotes are special to xargs unless you use the -0 option` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A filename contains an apostrophe (`O'Brien report.pdf`). Default `xargs` parses quotes and whitespace, so it chokes, and names with spaces would be split.

**Diagnose →** `find ... | grep "'"` finds the offending names.

**Fix →** NUL-separated end to end: `find /data -name '*.tmp' -print0 | xargs -0 -r rm -f --`, or `find /data -name '*.tmp' -delete`.

**Prevent →** Always pair `-print0` with `xargs -0`, or avoid `xargs` for deletes entirely.

</details>

### Q37. A script builds `OPTS="--delete --exclude=.git"` and runs `rsync "$OPTS" src/ dst/`. rsync fails with `rsync: --delete --exclude=.git: unknown option`. 🔴

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

### Q38. `ssh web1 "rm -rf /srv/app/releases/$old"` deleted the wrong directory when `$old` contained a space. How did that happen? 🔴

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

### Q39. `jq -r '.items[].metadata.name'` fails with `jq: error (at <stdin>:1): Cannot iterate over null (null)` 🟡

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

### Q40. `curl -s "$API/status" | jq .state` fails with `jq: parse error: Invalid numeric literal at line 1, column 10` 🟡

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

### Q41. A script with `set -e` stops silently right after `((retries++))` on its first pass. No error message, exit code 1. 🟡

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

### Q42. `notify.sh: line 9: SLACK_WEBHOOK: unbound variable` — but the webhook is meant to be optional. 🟢

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

### Q43. On a CentOS 7 / Amazon Linux 2 host, `deploy.sh "${extra_args[@]}"` fails with `extra_args[@]: unbound variable`. It works on your laptop. 🔴

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

### Q44. `local release=$(curl -fsS "$API/latest")` fails with a 503, yet the function happily continues and deploys an empty release name. 🟡

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

### Q45. A check `procs=$(ps aux | grep -v grep | grep -c '[n]ginx: worker')` kills the script under `set -euo pipefail` whenever nginx is down, before the alert code runs. 🟡

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

### Q46. A script that does `first=$(sort -u huge.txt | head -n 1)` exits with status 141 under `pipefail`, but only on large inputs. 🔴

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

### Q47. You added `trap 'echo "failed at line $LINENO" >&2' ERR`, but when a command inside a function fails, nothing is printed and the script just exits. 🟡

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

### Q48. `backup` calls `upload_to_s3`, which fails, yet `if backup; then echo "Backup OK"; fi` prints "Backup OK". `set -e` is on. 🔴

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

### Q49. The backup failed and printed errors, but cron and the monitoring wrapper both recorded exit code 0. The last line of the script is `main "$@" 2>&1 | tee -a "$LOG"`. 🟡

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

### Q50. Temp directories pile up in `/tmp` even though the script has `trap "rm -rf $workdir" EXIT`. 🟢

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

### Q51. Pressing Ctrl+C during `while read -r host; do ssh -n "$host" uptime; done < hosts.txt` only skips to the next host. You have to press it 40 times. 🟡

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

### Q52. `docker stop` on your container always takes 10 seconds and ends with `Exited (137)`. The entrypoint is a Bash script that starts the app. 🔴

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

### Q53. A health-check wrapper reports `check failed: unknown exit code 124` for `timeout 20 ./check_db.sh`. What does 124 mean, and what else can `timeout` return? 🟢

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

### Q54. The error log says `ERROR: curl failed with exit code 0`, which is useless. The code is `if ! curl -fsS "$URL" -o "$out"; then log ERROR "curl failed with exit code $?"; fi`. 🟡

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

### Q55. `result=$(compute_checksums; echo done)` returns `done` even when `compute_checksums` fails, despite `set -e`. 🔴

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

### Q56. A script dies immediately with exit code 1 and no output. The first command is `read -r VERSION < VERSION`. The file contains `1.4.2`. 🟡

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

### Q57. A script launches 8 uploads in the background with `&` and then calls `wait`. Three uploads failed, but the script exits 0. 🟡

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

### Q58. A wrapper logs `stages: 0` for a failed `pg_dump | gzip > db.gz`. The code reads `${PIPESTATUS[@]}` on the line after an `if`. 🟡

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

### Q59. An `ERR` trap runs `send_alert`, but when the alert itself fails (Slack is down), the script loops, printing the same error repeatedly until it is killed. 🔴

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

### Q60. The nightly export was `Killed` (dmesg shows `Out of memory: Killed process 4121 (export.sh)`). The `EXIT` trap never ran, and a half-written `report.csv` was published to the dashboard. 🟡

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

### Q61. A new job in `/etc/cron.d/opskit` never runs. Syslog shows `cron[812]: (*system*opskit) ERROR (Missing newline before EOF, this crontab file will be ignored)`. 🟢

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

### Q62. You copied a line from `crontab -e` into `/etc/cron.d/backup`, and syslog shows `Error: bad username; while reading /etc/cron.d/backup`. Meanwhile a teammate's user crontab mails `/bin/sh: 1: root: not found` every night. 🟢

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

### Q63. A script works when you run it, but the cron log email says `/opt/opskit/bin/backup.sh: line 22: aws: command not found`. 🟢

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

### Q64. This crontab line never produces a log file, and cron mails `/bin/sh: 1: Syntax error: end of file unexpected (expecting ")")`: `0 2 * * * /opt/backup.sh > /var/log/backup-$(date +%F).log 2>&1` 🟡

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

### Q65. A cron job fails intermittently, but there are no emails and no logs. Syslog shows `(CRON) info (No MTA installed, discarding output)`. 🟢

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

### Q66. A cron job that runs `docker exec -it db pg_dumpall` fails with `the input device is not a TTY`. 🟡

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

### Q67. Two schedules misbehave: `*/90 * * * *` runs every hour, not every 90 minutes, and `0 3 1-7 * 1` ("first Monday of the month") runs every Monday and on days 1 to 7. 🟡

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

### Q68. A job in root's crontab works, but the same job moved to the `deploy` user's crontab mails `sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper`. 🟢

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

### Q69. `systemctl status opskit-health.service` shows `Main process exited, code=exited, status=203/EXEC`. 🟢

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

### Q70. A new service fails instantly with `status=217/USER` and the journal says `Failed to determine user credentials: No such process`. 🟢

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

### Q71. You edited the timer from 5 minutes to 1 minute, restarted it, but it still fires every 5 minutes. `systemctl` printed `Warning: The unit file, source configuration file or drop-ins of opskit-health.timer changed on disk. Run 'systemctl daemon-reload' to reload units.` 🟢

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

### Q72. `systemctl enable opskit-backup.timer` prints `The unit files have no installation config (WantedBy=, RequiredBy=, ...)`, and after a reboot the timer is inactive. 🟡

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

### Q73. A backup unit restarts in a loop and then stops with `opskit-backup.service: Start request repeated too quickly.` and `Failed with result 'start-limit-hit'`. 🟡

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

### Q74. The journal shows `Failed to parse calendar specification, ignoring: daily 02:00` and then `opskit-backup.timer: Timer unit lacks value setting. Refusing.` 🟡

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

### Q75. The backup works with `sudo -u opskit /opt/opskit/bin/backup.sh`, but from its systemd unit it fails with `Unable to locate credentials. You can configure credentials by running "aws configure".` 🟡

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

### Q76. After hardening the unit with `ProtectSystem=strict`, the backup fails with `tar: /var/backups/opskit/etc-20261109.tar.gz: Cannot open: Read-only file system`. 🔴

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

### Q77. `journalctl -u opskit-health` returns nothing for your user, and prints `Hint: You are currently not seeing messages from other users and the system. Users in groups 'adm', 'systemd-journal' can see all messages.` 🟢

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

### Q78. Every run of a cron job logs `another run is in progress, exiting` for hours, even though `ps` shows no copy of the script running. It uses `exec 9>/run/opskit/backup.lock; flock -n 9 || exit 0`. 🔴

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

### Q79. A timer with only `OnUnitActiveSec=10min` was enabled and started, but `systemctl list-timers` shows `NEXT: -` and the job has never run. 🔴

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

### Q80. You dropped `logrotate-app.sh` into `/etc/cron.daily/` with mode 0755, but it never runs. `run-parts --test /etc/cron.daily` does not list it. 🟡

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

### Q81. On a Mac, `sed -i 's/replicas: 2/replicas: 3/' deploy.yaml` fails with `sed: 1: "deploy.yaml": extra characters at the end of d command`. 🟢

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

### Q82. A cleanup script fails on a developer's Mac with `date: illegal option -- d`. 🟢

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

### Q83. A script that works on Ubuntu fails in an Alpine-based CI image with `grep: unrecognized option: P`. 🟡

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

### Q84. An `awk` one-liner works on your RHEL box but fails on Ubuntu with `awk: line 1: function gensub never defined`. 🟡

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

### Q85. `comm -23 expected.txt installed.txt` prints `comm: file 1 is not in sorted order`, even though both files were produced with `sort`. 🟡

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

### Q86. `grep ERROR /var/log/app/app.log` prints only `grep: /var/log/app/app.log: binary file matches`. 🟢

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

### Q87. On macOS, `tr '[:lower:]' '[:upper:]' < export.csv` fails with `tr: Illegal byte sequence`, and `sed` gives `sed: RE error: illegal byte sequence`. 🟡

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

### Q88. `jq -r '.[] | select(.name == $svc) | .port' services.json` fails with `jq: error: $svc is not defined at <top-level>, line 1:`. 🟡

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

### Q89. A pipeline step `yq r values.yaml image.tag` fails with `Error: unknown command "r" for "yq"`, and on another runner `yq '.image.tag' values.yaml` fails with `yq: error: argument files: can't open '.image.tag'`. 🟡

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

### Q90. `sudo echo 'vm.swappiness=10' > /etc/sysctl.d/99-tuning.conf` fails with `bash: /etc/sysctl.d/99-tuning.conf: Permission denied`. 🟢

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

### Q91. `kubectl` works for you, but in a script `sudo kubectl apply -f deploy.yaml` fails with `sudo: kubectl: command not found`. 🟡

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

### Q92. A bootstrap step `curl -fsSL "$URL" -o /tmp/install.sh && chmod +x /tmp/install.sh && /tmp/install.sh` fails with `bash: /tmp/install.sh: Permission denied`, despite the `chmod`. 🟡

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

### Q93. A cleanup logged `cd: /srv/app/current/cache: No such file or directory` and then deleted the contents of the deploy user's home directory. The code was `cd $CACHE_DIR; rm -rf *`. 🟢

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

### Q94. A fan-out script hangs the box, and new SSH logins fail with `bash: fork: retry: Resource temporarily unavailable`. 🔴

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

### Q95. `rsync -az ./dist/ deploy@web1:/srv/app/` fails with `protocol version mismatch -- is your shell clean?`, while `ssh deploy@web1` works fine. 🔴

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

### Q96. A container entrypoint renders `nginx.conf.template` with `envsubst`, and nginx fails with `nginx: [emerg] invalid number of arguments in "proxy_set_header" directive in /etc/nginx/conf.d/default.conf:12`. 🟡

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

### Q97. A load check fails with `bash: ((: 4.75: syntax error: invalid arithmetic operator (error token is ".75")`. 🟢

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

### Q98. Behind the corporate proxy, the alert script fails with `curl: (60) SSL certificate problem: unable to get local issuer certificate`. Someone suggests adding `-k`. 🟡

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

### Q99. A Bash script copied into an `alpine` image fails with `env: can't execute 'bash': No such file or directory`. 🟢

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

### Q100. A log-summary script that took 2 minutes now takes 3 hours since traffic grew. `strace -c -f` shows millions of `clone` and `execve` calls. 🔴

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
