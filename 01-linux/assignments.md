# Linux — Assignments

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

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

Do these on a disposable Ubuntu 24.04 or Rocky Linux 9 VM (or a container with `systemd` where noted). Never practice destructive commands on a machine you care about. Open the hint only after you have tried for 10 minutes.

## Table of contents
- [🟢 Beginner](#beginner) — A1–A10
- [🟡 Intermediate](#intermediate) — A11–A20
- [🔴 Advanced](#advanced) — A21–A25

---

## Beginner

### A1. Filesystem navigation and hierarchy 🟢

**Objective:** Get fluent with the directory layout and basic file operations.

**Tasks**
1. Create `~/lab/a1/{docs,src,tmp}` with one command.
2. Create five files in `src` using `touch`, and copy two into `docs`.
3. Move one file from `src` to `tmp` and rename it to `old.txt`.
4. Use `tree` (or `find`) to show the structure, then delete `tmp` recursively.
5. Print what `/etc`, `/var/log`, `/proc` and `/usr/local` are used for, in your own words.

**Expected outcome:** The final tree has `docs` (2 files) and `src` (4 files), `tmp` is gone, and you can explain the four directories.

<details>
<summary>Hints</summary>

`mkdir -p ~/lab/a1/{docs,src,tmp}`, `touch ~/lab/a1/src/f{1..5}.txt`, `cp`, `mv`, `rm -r`. Brace expansion is done by the shell, not by `mkdir`.

</details>

**Bonus:** Do task 1 to 4 in a single script that exits non-zero if any step fails.

### A2. Permissions and ownership 🟢

**Objective:** Read and set permissions in both symbolic and octal form.

**Tasks**
1. Create `secret.txt` and set it so only the owner can read and write.
2. Create a script `hello.sh` and make it executable only for the owner.
3. Create a directory `shared` where the group `devs` can write.
4. Show `ls -l` output and decode every permission string by hand.
5. Show that a user who is not in `devs` cannot enter `shared` after `chmod 770`.

**Expected outcome:** `secret.txt` is `-rw-------`, `hello.sh` is `-rwx------`, `shared` is `drwxrwx---` owned by group `devs`.

<details>
<summary>Hints</summary>

`chmod 600`, `chmod u+x`, `groupadd devs`, `chgrp devs shared`, `chmod 770 shared`. Test with `sudo -u otheruser ls shared`. Remember a directory needs `x` to be entered.

</details>

**Bonus:** Make new files in `shared` inherit group `devs` automatically using the setgid bit (`chmod g+s`) and prove it.

### A3. Users and groups 🟢

**Objective:** Manage accounts the way a sysadmin would.

**Tasks**
1. Create users `alice` and `bob` with home directories and bash as the shell.
2. Add `alice` to `sudo` (Debian/Ubuntu) or `wheel` (RHEL), but not `bob`.
3. Lock `bob`'s password, then unlock it.
4. Set a password-expiry policy for `alice` of 90 days maximum.
5. Delete `bob` including his home directory.

**Expected outcome:** `id alice` shows the admin group, `chage -l alice` shows 90 days, `getent passwd bob` returns nothing.

<details>
<summary>Hints</summary>

`useradd -m -s /bin/bash`, `usermod -aG sudo alice` (the `-a` matters, without it you replace the group list), `passwd -l/-u`, `chage -M 90 alice`, `userdel -r bob`. Check `/etc/passwd`, `/etc/shadow` and `/etc/group`.

</details>

**Bonus:** Write a script that reads usernames from a file and creates them idempotently (skipping existing ones).

### A4. Text processing pipeline 🟢

**Objective:** Use `grep`, `awk`, `sed`, `sort` and `uniq` on a real log.

**Tasks**
1. Generate a fake access log with 1,000 lines (or use `/var/log/auth.log` or `journalctl` output).
2. Count requests per IP and show the top 5.
3. Extract all lines with HTTP status 5xx.
4. Replace every IP address with `x.x.x.x` using `sed` (for anonymising).
5. Print the 3rd and 7th fields of each line with `awk`.

**Expected outcome:** A one-liner pipeline for each task, each producing correct output on your sample.

<details>
<summary>Hints</summary>

`awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5`. For 5xx: `awk '$9 ~ /^5/' access.log`. For anonymising: `sed -E 's/([0-9]{1,3}\.){3}[0-9]{1,3}/x.x.x.x/g'`.

</details>

**Bonus:** Show requests per minute with `cut -c` on the timestamp, then find the peak minute.

### A5. Find files and clean up safely 🟢

**Objective:** Use `find` and `du` to locate space hogs and stale files.

**Tasks**
1. Create 20 files of mixed sizes with `dd` or `fallocate`, some backdated with `touch -d`.
2. Find files larger than 10 MB.
3. Find files older than 7 days.
4. Show the five largest directories under `/var` with `du`.
5. Delete the old files using `find ... -delete`, after first listing what would go.

**Expected outcome:** Only files that match both criteria are removed, and you ran the list command before the delete command.

<details>
<summary>Hints</summary>

`find . -type f -size +10M`, `find . -type f -mtime +7`, `du -xh /var --max-depth=2 | sort -rh | head`, `find . -type f -mtime +7 -print` then swap `-print` for `-delete`. Put `-delete` last, since order matters.

</details>

**Bonus:** Find files by owner or permissions (for example world-writable) with `-perm -0002`.

### A6. Processes and signals 🟢

**Objective:** Observe, signal and control processes.

**Tasks**
1. Start `sleep 600` in the background, find its PID with `pgrep` and `ps`.
2. Pause it with `SIGSTOP`, resume with `SIGCONT` and verify the state letter in `ps`.
3. Terminate it with `SIGTERM`.
4. Write a bash script that traps `SIGTERM` and `SIGINT`, prints a message and exits cleanly; send it signals from another terminal.
5. Change the priority of a CPU-burning loop with `nice` and `renice`.

**Expected outcome:** You can show states `S`, `T`, and the trap message on `kill -15`, and the NI column changing in `top`.

<details>
<summary>Hints</summary>

`kill -STOP PID`, `ps -o pid,stat,cmd -p PID`, `trap 'echo bye; exit 0' TERM INT`. For the loop: `yes > /dev/null &`, then `renice +10 -p PID`. `kill -9` cannot be trapped.

</details>

**Bonus:** Create a zombie process on purpose (a parent that does not `wait`) and identify it.

### A7. Install and manage packages and services 🟢

**Objective:** Install software and manage it with systemd.

**Tasks**
1. Install `nginx` with `apt` or `dnf`.
2. Check that it is enabled and running, and where its config and logs live (`dpkg -L` or `rpm -ql`).
3. Stop it, disable it, then enable and start it in one command.
4. View its last 20 log lines through `journalctl`.
5. Find which package owns `/usr/sbin/nginx`.

**Expected outcome:** `systemctl is-enabled nginx` and `is-active` return `enabled` and `active`, and you can name the owning package.

<details>
<summary>Hints</summary>

`systemctl enable --now nginx`, `journalctl -u nginx -n 20`, `dpkg -S /usr/sbin/nginx` or `rpm -qf /usr/sbin/nginx`. On Debian-based systems services start on install, on RHEL they do not.

</details>

**Bonus:** Hold the package at its current version (`apt-mark hold`) or lock it (`dnf versionlock`).

### A8. SSH keys and config 🟢

**Objective:** Set up key-based login and a client config.

**Tasks**
1. Generate an `ed25519` key pair with a passphrase.
2. Install the public key on a second machine (or a second local user) with `ssh-copy-id`.
3. Create `~/.ssh/config` with a `Host` alias that sets user, hostname and key.
4. Disable password authentication on the server and confirm it is refused.
5. Use `ssh-agent` so you type the passphrase once.

**Expected outcome:** `ssh myalias` logs in with the key, and `ssh -o PubkeyAuthentication=no` fails with `Permission denied (publickey)`.

<details>
<summary>Hints</summary>

`ssh-keygen -t ed25519`, `chmod 600 ~/.ssh/config`, `eval "$(ssh-agent)"; ssh-add`. Server side: `sshd -t` before `systemctl reload ssh`. Permissions on `~/.ssh` (700) and `authorized_keys` (600) are the number one reason keys fail.

</details>

**Bonus:** Copy a file with `scp` and with `rsync -avz -e ssh` and compare the speed on a second run.

### A9. Disk usage, mounts and fstab 🟢

**Objective:** Attach a new disk and mount it persistently.

**Tasks**
1. Create a loop-backed "disk": `truncate -s 1G disk.img; losetup -f --show disk.img`.
2. Partition it or use it whole, format it with `ext4`.
3. Mount it on `/mnt/data`, write a file, then unmount.
4. Add an entry using the filesystem **UUID** to `/etc/fstab` with `nofail`.
5. Test with `mount -a` and show it with `findmnt` and `df -hT`.

**Expected outcome:** `/mnt/data` mounts with `mount -a`, using a UUID entry, without a reboot.

<details>
<summary>Hints</summary>

`mkfs.ext4 /dev/loop0`, `blkid /dev/loop0`, line: `UUID=<uuid> /mnt/data ext4 defaults,nofail 0 2`. Always run `mount -a` after editing `fstab`, since a bad line can drop you into emergency mode at boot. Detach with `losetup -d`.

</details>

**Bonus:** Show the difference in `df -h` and `df -i` after creating 100,000 empty files.

### A10. Basic network diagnostics 🟢

**Objective:** Inspect network configuration and test connectivity.

**Tasks**
1. Show your IP addresses, default route and DNS servers.
2. List all listening TCP ports and the owning processes.
3. Resolve `example.com` with `dig` and with `getent hosts`; explain why they can differ.
4. Test whether port 443 is reachable on a remote host using `nc -vz` and `curl -v`.
5. Trace the path with `traceroute` or `mtr`.

**Expected outcome:** A short note identifying your interface, gateway, resolver and the first slow hop (if any).

<details>
<summary>Hints</summary>

`ip -br a`, `ip r`, `resolvectl status`, `ss -tulpn`, `dig +short example.com`, `nc -vz host 443`. `getent hosts` follows `nsswitch.conf` (including `/etc/hosts`), while `dig` talks directly to DNS.

</details>

**Bonus:** Add a temporary entry to `/etc/hosts` and show which tool sees it and which does not.

---

## Intermediate

### A11. Write a production systemd service 🟡

**Objective:** Run an app as a managed, hardened service.

**Tasks**
1. Write a small app (Python `http.server` is fine) that listens on port 9000.
2. Create a dedicated system user with `nologin`.
3. Write `/etc/systemd/system/myapp.service` with `User=`, `Restart=on-failure`, and `Environment=`.
4. Add at least five hardening options (`NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`, ...).
5. Kill the process with `-9` and prove it comes back; check `systemd-analyze security myapp`.

**Expected outcome:** Service is `active`, restarts within seconds after a kill, and has an exposure score below 5.0 (MEDIUM or better).

<details>
<summary>Hints</summary>

`systemctl daemon-reload` after each edit. Use `ReadWritePaths=` with `ProtectSystem=strict`. Use `journalctl -u myapp -f` to watch restarts. Status `203/EXEC` means a bad ExecStart path.

</details>

**Bonus:** Add `MemoryMax=` and `CPUQuota=` and prove the limit with a memory-hungry process.

### A12. systemd timer replacing cron 🟡

**Objective:** Schedule a job with a systemd timer.

**Tasks**
1. Write a oneshot service that appends a timestamp to `/var/log/tick.log`.
2. Write a timer that runs it every 2 minutes after boot and with `Persistent=true`.
3. Enable the timer and verify with `systemctl list-timers`.
4. Validate a calendar expression with `systemd-analyze calendar "Mon..Fri 02:30"`.
5. Stop the machine's timer for 5 minutes and show catch-up behaviour on restart.

**Expected outcome:** `tick.log` receives entries; `list-timers` shows NEXT and LAST columns populated.

<details>
<summary>Hints</summary>

Enable the **timer** (`systemctl enable --now tick.timer`), not the service. Use `OnUnitActiveSec=2min` and `OnBootSec=1min`. `Persistent=true` only works with `OnCalendar=`.

</details>

**Bonus:** Add `RandomizedDelaySec=` and `OnFailure=` pointing to a unit that logs a failure message.

### A13. Bash backup script with retention 🟡

**Objective:** Write a robust, cron-ready backup script.

**Tasks**
1. Write `backup.sh` that archives a directory to `/var/backups` with a timestamped name.
2. Use `set -euo pipefail`, a `trap` for cleanup and `flock` to prevent overlap.
3. Verify the archive (`tar -tzf`) and write a SHA-256 file.
4. Delete backups older than 7 days.
5. Schedule it with cron at 02:00 and test a restore into a temp directory.

**Expected outcome:** Exit code 0 on success and non-zero when the source is missing, a log line for each run, restored data equals source (`diff -r`).

<details>
<summary>Hints</summary>

Cron has a minimal `PATH`, so use absolute paths. `exec 9>/run/x.lock; flock -n 9 || exit 1`. Test failure with a wrong `SRC` and check `echo $?`. Use `find ... -mtime +7 -delete` for retention.

</details>

**Bonus:** Add an off-host copy with `rsync` or `aws s3 cp` and a freshness check that alerts if the newest backup is over 26 hours old.

### A14. logrotate for a custom application log 🟡

**Objective:** Configure and test rotation without waiting a day.

**Tasks**
1. Write an app that logs a line per second to `/var/log/myapp/app.log`.
2. Create `/etc/logrotate.d/myapp` with `daily`, `rotate 7`, `compress`, `delaycompress`, `missingok`, `notifempty`.
3. Dry-run with `logrotate -d` and force with `logrotate -f`.
4. Solve the "app keeps writing to the rotated file" problem, first with `copytruncate`, then with a `postrotate` signal.
5. Explain the trade-off between the two.

**Expected outcome:** After forcing, you see `app.log`, `app.log.1` (uncompressed because of `delaycompress`), and new lines go to the new `app.log`.

<details>
<summary>Hints</summary>

`copytruncate` can lose a few lines between copy and truncate; `postrotate` with `kill -HUP` or `systemctl reload` needs the app to reopen files. Check state in `/var/lib/logrotate/status`. Permissions errors often need `su user group`.

</details>

**Bonus:** Rotate by size (`size 50M`) and show it triggers only when the size passes.

### A15. nginx reverse proxy and load balancing 🟡

**Objective:** Front two app instances with nginx.

**Tasks**
1. Start two backends on ports 8001 and 8002 that print different identifiers.
2. Configure an `upstream` with both servers and `proxy_pass` to it.
3. Pass `Host`, `X-Real-IP` and `X-Forwarded-For` headers.
4. Show round-robin with `for i in {1..6}; do curl -s localhost; done`.
5. Stop one backend and show failover, then check the error log.

**Expected outcome:** Alternating responses; with one backend down the other serves all requests with no 5xx visible after the first failed attempt.

<details>
<summary>Hints</summary>

`nginx -t && nginx -s reload`. Use `max_fails=2 fail_timeout=10s` on upstream servers. `ip_hash;` gives sticky behaviour. The error log shows `connect() failed (111: Connection refused)` for the dead backend.

</details>

**Bonus:** Add a rate limit with `limit_req_zone` and prove it returns 503 (or 429 with `limit_req_status 429`) under `ab` or `hey`.

### A16. Firewall with nftables or ufw 🟡

**Objective:** Build a default-deny firewall safely.

**Tasks**
1. Set default deny incoming, allow established/related, allow loopback.
2. Allow SSH from one source subnet only, and 80/443 from anywhere.
3. Add a rate limit on new SSH connections.
4. Log and drop everything else with a limit on the log rate.
5. Persist the rules across reboot and test from another host with `nmap`.

**Expected outcome:** `nmap` from outside shows only 80 and 443 open (22 filtered unless from the allowed subnet), and rules survive a reboot.

<details>
<summary>Hints</summary>

For ufw: `ufw limit 22/tcp`, `ufw allow from 10.0.0.0/24 to any port 22`. For nftables: write `/etc/nftables.conf`, check with `nft -c -f`, `systemctl enable nftables`. Schedule a rollback (`at now + 5 min` with `ufw disable`) before testing remotely.

</details>

**Bonus:** Block an abusive IP with an nftables set that has a timeout.

### A17. LVM: create, extend and shrink 🟡

**Objective:** Manage storage flexibly with LVM.

**Tasks**
1. Create two loop devices and make them physical volumes.
2. Create a volume group, then a 500 MB logical volume with `xfs` or `ext4`.
3. Mount it, fill it to 90% and extend the LV online with `lvextend -r`.
4. Add the second PV to the VG to get extra space first if needed.
5. Take a snapshot, change data, and roll back.

**Expected outcome:** `df -h` shows the larger filesystem without unmounting, and the snapshot restores the old content.

<details>
<summary>Hints</summary>

`pvcreate`, `vgcreate`, `lvcreate -L 500M -n data vg0`, `lvextend -r -L +300M /dev/vg0/data`, `vgextend`. `xfs` cannot shrink. Use `lvs`, `vgs`, `pvs` to inspect. Snapshot: `lvcreate -s -n snap -L 200M vg0/data`, merge with `lvconvert --merge`.

</details>

**Bonus:** Replace a "failing" PV with `pvmove` and remove it from the VG without downtime.

### A18. Performance triage under load 🟡

**Objective:** Identify the bottleneck with the right tool.

**Tasks**
1. Generate CPU load with `stress-ng --cpu 2`, then find it with `top` and `pidstat`.
2. Generate disk I/O with `dd` or `fio` and spot it with `iostat -xz 1` and `iotop`.
3. Generate memory pressure with `stress-ng --vm 1 --vm-bytes 80%` and watch `vmstat 1`.
4. For each case, write one sentence: what metric proved the bottleneck.
5. Compare load average with `nproc`, and explain what a load of 8 means on 4 cores.

**Expected outcome:** A table of three scenarios with symptom, tool, and metric (for example `%util` near 100 and high `await` for disk).

<details>
<summary>Hints</summary>

High `wa` in `top` means waiting on I/O. `si`/`so` columns in `vmstat` mean swapping. `free -m` "available" is the real headroom. `mpstat -P ALL 1` shows single-core saturation.

</details>

**Bonus:** Use `sar` (from `sysstat`) to look back at yesterday's CPU history.

### A19. Troubleshoot a broken server (planted faults) 🟡

**Objective:** Diagnose faults methodically instead of guessing.

**Tasks**
1. Have a friend (or a script) break five things: fill `/var` inodes, stop nginx, change a file's permissions, break `fstab`, add a bad DNS server.
2. For each symptom write what you checked first and why.
3. Fix each one and note the root cause.
4. Write a runbook of five entries: symptom, command, fix.
5. Time yourself; aim for under 5 minutes per fault.

**Expected outcome:** All five faults fixed, with a runbook that another engineer can follow.

<details>
<summary>Hints</summary>

`df -h` and `df -i`, `systemctl --failed`, `journalctl -xe`, `namei -l /path` for permission issues, `findmnt --verify` for fstab, `resolvectl query example.com` for DNS. Check resources first, then the service, then the app.

</details>

**Bonus:** Automate the faults as a script so you can re-run the exercise with different random picks.

### A20. Harden SSH and add fail2ban 🟡

**Objective:** Reduce the attack surface of remote access.

**Tasks**
1. Create `/etc/ssh/sshd_config.d/10-hardening.conf` with `PermitRootLogin no`, `PasswordAuthentication no`, `AllowUsers`, `MaxAuthTries 3`.
2. Validate with `sshd -t` and `sshd -T`; test from a second session before closing the first.
3. Install `fail2ban` and enable the `sshd` jail with a 1 hour ban.
4. Trigger a ban with failed logins and view it with `fail2ban-client status sshd`.
5. Unban your IP and add it to `ignoreip`.

**Expected outcome:** Root and password logins fail, repeated failures create a ban visible in the firewall.

<details>
<summary>Hints</summary>

Check for cloud-init drop-ins that re-enable passwords. On Ubuntu 24.04 use `backend = systemd`. `fail2ban-client set sshd unbanip IP`. Keep console access as a fallback.

</details>

**Bonus:** Add `AllowGroups ssh-users`, move to certificate-based SSH or enable `AuthenticationMethods publickey,keyboard-interactive` for 2FA.

---

## Advanced

### A21. Build a container by hand with namespaces and cgroups 🔴

**Objective:** Prove that a container is just isolated Linux processes.

**Tasks**
1. Extract a minimal root filesystem (for example `docker export $(docker create alpine) | tar -x -C rootfs`, or `debootstrap`).
2. Start a shell with `unshare --pid --mount --uts --net --fork --mount-proc chroot rootfs /bin/sh`.
3. Show that PID 1 inside is your shell and the hostname is independent.
4. Create a cgroup v2 at `/sys/fs/cgroup/demo`, set `memory.max` to 50M and `cpu.max` to `20000 100000`, and move the process in.
5. Run a memory hog inside and show the OOM kill in `dmesg` and `memory.events`.

**Expected outcome:** A shell with its own PID, UTS and mount namespace limited to 50 MB and 20% of one CPU, with evidence of the OOM kill.

<details>
<summary>Hints</summary>

`echo $$ > /sys/fs/cgroup/demo/cgroup.procs`, `echo 50M > memory.max`. You may need `echo +memory +cpu > /sys/fs/cgroup/cgroup.subtree_control`. `lsns` and `/proc/<pid>/ns/` show namespaces. Use `nsenter -t PID -n ip a` to enter from outside.

</details>

**Bonus:** Give the namespace network access with a veth pair, a bridge and `iptables` MASQUERADE.

### A22. Trace a hung or slow process 🔴

**Objective:** Find the root cause of a stuck process without restarting it.

**Tasks**
1. Write (or use) a program that hangs: one blocked on a read from a FIFO, one in a busy loop, one blocked on a lock file.
2. Use `strace -p PID -f -tt` to see the syscall each one is stuck in.
3. Use `ls -l /proc/PID/fd`, `cat /proc/PID/stack` (root), `cat /proc/PID/wchan` and `lsof -p PID`.
4. Use `perf top -p PID` or `perf record -g` to profile the busy loop.
5. Write the findings: what each tool showed and which would you use first in production.

**Expected outcome:** For each program you can state the exact blocking call, the file or resource involved, and the evidence.

<details>
<summary>Hints</summary>

A process stuck in `read(3, ...` on an FD that `/proc/PID/fd/3` shows as a FIFO or socket tells you who it waits for. State `D` with a stack in `nfs_` or `io_schedule` points to storage. `strace -c` summarises time per syscall. Mind the overhead of strace on production.

</details>

**Bonus:** Use `bpftrace` or `bcc` (`opensnoop`, `execsnoop`) to trace system-wide without stopping the process.

### A23. Kernel tuning for a high-traffic server 🔴

**Objective:** Tune and validate `sysctl` and limits with measured evidence.

**Tasks**
1. Baseline with `wrk` or `ab` against nginx and record requests per second and errors.
2. Inspect and raise `net.core.somaxconn`, `net.ipv4.tcp_max_syn_backlog`, `fs.file-max` and the service `LimitNOFILE`.
3. Tune `net.ipv4.ip_local_port_range`, enable `tcp_tw_reuse` for outbound-heavy clients and set `vm.swappiness`.
4. Persist in `/etc/sysctl.d/99-tuning.conf` and apply with `sysctl --system`.
5. Re-run the benchmark, compare, and document which change helped and which did not.

**Expected outcome:** A before/after table with a justification for every kept setting, and no cargo-culted values.

<details>
<summary>Hints</summary>

`ss -s` and `nstat -az | grep -i listen` show overflows. `Too many open files` is solved by `LimitNOFILE` in the unit, not just sysctl. Change one variable at a time. Do not "tune" `tcp_tw_recycle` (removed from the kernel).

</details>

**Bonus:** Plot a latency histogram (p50/p99) before and after, and test with irqbalance or `taskset` CPU pinning.

### A24. Incident: OOM kills and disk-full postmortem 🔴

**Objective:** Reproduce two classic outages and write the postmortem.

**Tasks**
1. Reproduce an OOM kill (service with no memory limit plus a leak) and find it with `dmesg -T | grep -i 'out of memory'` and `journalctl -k`.
2. Read the OOM report: which process was chosen, `oom_score_adj`, memory at the time.
3. Prevent a repeat with `MemoryMax=`, `OOMScoreAdjust=` and an alert on `available` memory.
4. Reproduce a full-disk event where `df` shows 100% but `du` shows less (deleted-but-open file); locate it with `lsof +L1` and fix without restart if possible.
5. Write a one-page blameless postmortem for each with timeline, root cause, detection gap and action items.

**Expected outcome:** Two postmortems, each with evidence commands and a prevention that is testable.

<details>
<summary>Hints</summary>

`: > /proc/PID/fd/N` truncates a deleted open file. `systemd-oomd` and cgroup-level OOM differ from global OOM. Check `df -i` too. Rotated logs without a reopen are the usual deleted-file cause.

</details>

**Bonus:** Write a Prometheus alert rule for `node_memory_MemAvailable_bytes` and `node_filesystem_avail_bytes` with predictive `predict_linear`.

### A25. CIS-style hardening and audit with auditd 🔴

**Objective:** Bring a server near a recognised security baseline and prove it.

**Tasks**
1. Remove unneeded packages and disable unused services; list listening sockets before and after.
2. Set kernel hardening in `/etc/sysctl.d/`: `kernel.kptr_restrict`, `net.ipv4.conf.all.rp_filter`, `net.ipv4.conf.all.accept_redirects=0`, `kernel.randomize_va_space=2`.
3. Mount `/tmp` with `nodev,nosuid,noexec` and verify an executable cannot run from it.
4. Install `auditd` and write rules watching `/etc/passwd`, `/etc/sudoers`, `/etc/ssh/sshd_config` and `execve` by root; trigger and search events with `ausearch`.
5. Run `lynis audit system` (or OpenSCAP) before and after, and record the score and the top five fixes.

**Expected outcome:** Lynis hardening index improves measurably; audit events show who changed a watched file.

<details>
<summary>Hints</summary>

`auditctl -w /etc/passwd -p wa -k identity`, persist in `/etc/audit/rules.d/*.rules` and `augenrules --load`. `ausearch -k identity -i`. Beware `noexec` on `/tmp` breaking installers; test your package manager. Make changes via configuration management so they are reproducible.

</details>

**Bonus:** Express the whole hardening as an Ansible role and run it idempotently (second run changes nothing).

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

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
