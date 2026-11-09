# Linux — Cheatsheet

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

> Dense command and config reference for DevOps work, grouped by task.

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Files and navigation](#files-and-navigation)
- [Permissions and ownership](#permissions-and-ownership)
- [Users, groups and sudo](#users-groups-and-sudo)
- [Text processing](#text-processing)
- [Processes and signals](#processes-and-signals)
- [systemd and logs](#systemd-and-logs)
- [Disk, filesystems and LVM](#disk-filesystems-and-lvm)
- [Networking](#networking)
- [Firewall](#firewall)
- [SSH](#ssh)
- [Performance](#performance)
- [Packages](#packages)
- [Bash scripting](#bash-scripting)
- [cron, timers and logrotate](#cron-timers-and-logrotate)
- [Containers internals](#containers-internals)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Files and navigation

| Task | Command |
|------|---------|
| List with sizes, newest last | `ls -lhtr` |
| Show tree, depth 2 | `tree -L 2` |
| Find by name | `find /etc -name "*.conf" -type f` |
| Find large files | `find / -xdev -type f -size +100M 2>/dev/null` |
| Find modified in last day | `find . -type f -mtime -1` |
| Find and delete old files | `find /tmp -type f -mtime +7 -delete` |
| Find then run command | `find . -name "*.log" -exec gzip {} +` |
| Disk usage by directory | `du -xh --max-depth=1 /var \| sort -rh \| head` |
| Free space and type | `df -hT` |
| Inodes | `df -i` |
| File type / metadata | `file x`, `stat x` |
| Follow a log | `tail -F /var/log/syslog` |
| Copy preserving attrs | `cp -a src dst` |
| Sync with delete | `rsync -avz --delete src/ host:/dst/` |
| Archive / extract | `tar -czf a.tgz dir/`, `tar -xzf a.tgz -C /dest` |
| Links | `ln file hard`, `ln -s target soft` |
| Which command / path | `type -a cmd`, `which cmd` |
| Locate file owner pkg | `dpkg -S /path`, `rpm -qf /path` |

```bash
# Redirection
cmd > out.txt          # stdout, overwrite
cmd >> out.txt         # append
cmd 2> err.txt         # stderr
cmd > all.txt 2>&1     # both
cmd &> all.txt         # both (bash)
cmd 2>/dev/null        # discard errors
cmd | tee file.txt     # screen and file
```

## Permissions and ownership

| Mode | Meaning |
|------|---------|
| `644` | `rw-r--r--` regular file |
| `600` | `rw-------` private key, secrets |
| `755` | `rwxr-xr-x` script, directory |
| `700` | `rwx------` `~/.ssh` |
| `1777` | sticky bit, `/tmp` |
| `2775` | setgid directory, group inheritance |
| `4755` | setuid binary (for example `passwd`) |

```bash
chmod u+x,g-w,o-rwx file       # symbolic
chmod -R 750 dir
chown user:group file
chown -R www-data: /var/www
umask 027                       # new files 640, dirs 750
getfacl file
setfacl -m u:alice:rw file      # ACL for one user
setfacl -R -d -m g:devs:rwx dir # default ACL
namei -l /path/to/file          # permissions along the path
lsattr / chattr +i file         # immutable flag
find / -xdev -perm -4000 -type f   # list setuid files
```

## Users, groups and sudo

```bash
useradd -m -s /bin/bash alice        # create user
useradd --system -s /usr/sbin/nologin svc   # service account
usermod -aG sudo alice               # add to group (-a is essential)
passwd -l alice / passwd -u alice    # lock / unlock
chage -l alice; chage -M 90 alice    # password aging
userdel -r alice
id alice; groups alice; getent passwd alice
last -n 10; lastb -n 10; w; who      # logins
visudo -cf /etc/sudoers.d/file       # validate sudoers
sudo -l                              # what can I run
su - alice                           # login shell as alice
```

Sudoers snippet: `alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx`

## Text processing

```bash
grep -rniE "error|fail" /var/log/app/   # recursive, case-insens., regex
grep -v '^#' file | grep -v '^$'         # strip comments and blanks
grep -C 3 "pattern" file                 # context lines
awk '{print $1}' f | sort | uniq -c | sort -rn | head
awk -F: '$3>=1000 {print $1}' /etc/passwd
awk '$9>=500' access.log
sed -i 's/old/new/g' file                # in place
sed -n '10,20p' file                     # print range
cut -d, -f1,3 file.csv
sort -k2 -n file; sort -u file
tr 'a-z' 'A-Z'; tr -d '\r'
wc -l; head -n 20; tail -n 20
xargs -n1 -P4 cmd                        # parallel
jq '.items[].metadata.name' file.json
diff -u a b; comm -12 <(sort a) <(sort b)
```

## Processes and signals

| Signal | No. | Use |
|--------|-----|-----|
| `SIGHUP` | 1 | Reload config (daemons) |
| `SIGINT` | 2 | Ctrl-C |
| `SIGKILL` | 9 | Force kill, cannot be caught |
| `SIGTERM` | 15 | Graceful stop (default) |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | Pause / resume |
| `SIGUSR1` | 10 | App-defined (nginx reopen logs) |

```bash
ps aux --sort=-%mem | head           # top memory
ps -eo pid,ppid,stat,cmd --forest
pgrep -fa nginx; pkill -TERM -f myapp
kill -15 PID; kill -9 PID; killall name
top -o %CPU; htop
nice -n 10 cmd; renice +5 -p PID
nohup cmd & ; disown ; jobs; fg; bg
lsof -p PID; lsof -i :80; lsof +L1   # deleted-but-open files
ls -l /proc/PID/fd; cat /proc/PID/status
strace -p PID -f -tt; strace -c cmd
```

Process states in `ps`: `R` running, `S` sleeping, `D` uninterruptible I/O, `T` stopped, `Z` zombie.

## systemd and logs

| Task | Command |
|------|---------|
| Status / start / stop | `systemctl status\|start\|stop\|restart\|reload svc` |
| Enable at boot and start now | `systemctl enable --now svc` |
| Failed units | `systemctl --failed` |
| Reload unit files | `systemctl daemon-reload` |
| Show effective unit | `systemctl cat svc` |
| Edit with drop-in | `systemctl edit svc` |
| Dependencies | `systemctl list-dependencies svc` |
| Timers | `systemctl list-timers --all` |
| Boot time breakdown | `systemd-analyze blame` |
| Security score | `systemd-analyze security svc` |
| Default target | `systemctl get-default`, `set-default multi-user.target` |
| Rescue / emergency | `systemctl rescue`, add `systemd.unit=emergency.target` to kernel line |

```bash
journalctl -u nginx -f                      # follow a unit
journalctl -u nginx --since "1 hour ago"
journalctl -p err -b                        # errors this boot
journalctl -b -1                            # previous boot
journalctl -k                               # kernel messages
journalctl --disk-usage; journalctl --vacuum-time=7d
journalctl _PID=123
```

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My app
After=network-online.target
Wants=network-online.target

[Service]
User=myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=3
NoNewPrivileges=true
ProtectSystem=strict
PrivateTmp=true
ReadWritePaths=/var/lib/myapp
LimitNOFILE=65536
MemoryMax=500M

[Install]
WantedBy=multi-user.target
```

```ini
# Timer pair: myjob.timer
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
[Install]
WantedBy=timers.target
```

## Disk, filesystems and LVM

```bash
lsblk -f; blkid; findmnt                 # inventory
fdisk -l; parted -l
fdisk /dev/sdb                           # interactive, GPT with g
mkfs.ext4 /dev/sdb1; mkfs.xfs /dev/sdb1
mount /dev/sdb1 /mnt; umount /mnt
mount -a; findmnt --verify               # test fstab
tune2fs -l /dev/sdb1                     # ext4 info
resize2fs /dev/vg0/data                  # grow ext4
xfs_growfs /mnt/data                     # grow xfs (mounted)
fsck -n /dev/sdb1                        # read-only check (unmounted)
iostat -xz 1; iotop -oPa
ncdu /var                                # interactive usage
```

| LVM task | Command |
|----------|---------|
| Create PV, VG, LV | `pvcreate /dev/sdb; vgcreate vg0 /dev/sdb; lvcreate -L 10G -n data vg0` |
| Extend LV and FS | `lvextend -r -L +5G /dev/vg0/data` |
| Add disk to VG | `vgextend vg0 /dev/sdc` |
| Snapshot | `lvcreate -s -n snap -L 2G vg0/data` |
| Move data off PV | `pvmove /dev/sdb` |
| Inspect | `pvs; vgs; lvs` |

`/etc/fstab` line: `UUID=xxxx-xxxx /data xfs defaults,nofail,noatime 0 2`

Full disk but `du` looks small? Check `df -i` (inodes), `lsof +L1` (deleted but open), and hidden mounts over a directory.

## Networking

| Task | Command |
|------|---------|
| Addresses | `ip -br a` |
| Routes | `ip r`, `ip route get 8.8.8.8` |
| Listening sockets and process | `ss -tulpn` |
| Established by state | `ss -s`, `ss -tan state established` |
| Who connects to port 443 | `ss -tn sport = :443` |
| DNS lookup | `dig +short example.com`, `dig @8.8.8.8 example.com MX` |
| Resolver in use | `resolvectl status` |
| Resolve like apps do | `getent hosts name` |
| HTTP debug | `curl -sv -o /dev/null https://host`, `curl -I url` |
| Time breakdown | `curl -w "%{time_namelookup} %{time_connect} %{time_starttransfer} %{time_total}\n" -o /dev/null -s url` |
| Port test | `nc -vz host 443` |
| Path | `mtr -rw host`, `traceroute host` |
| Packet capture | `tcpdump -ni any port 53 -c 20`, `tcpdump -w cap.pcap` |
| ARP / neighbours | `ip neigh` |
| Interface stats | `ip -s link`, `ethtool eth0` |
| Scan | `nmap -sT -p- host` |

```bash
# Temporary and persistent config
ip addr add 10.0.0.5/24 dev eth1
ip route add 10.1.0.0/16 via 10.0.0.1
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-forward.conf && sysctl --system
```

## Firewall

```bash
# ufw
ufw default deny incoming; ufw default allow outgoing
ufw allow 22/tcp; ufw allow 'Nginx Full'
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw limit 22/tcp; ufw status numbered; ufw delete 3; ufw --force enable

# firewalld (RHEL)
firewall-cmd --permanent --add-service=http && firewall-cmd --reload
firewall-cmd --list-all

# iptables (legacy view)
iptables -L -n -v --line-numbers
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP

# nftables
nft list ruleset
nft add rule inet filter input tcp dport 443 accept
nft -c -f /etc/nftables.conf            # syntax check
```

Order matters: allow `ESTABLISHED,RELATED` and loopback and your SSH rule **before** setting default DROP.

## SSH

```bash
ssh-keygen -t ed25519 -C "name@host"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
ssh -J bastion user@private-host           # jump host
ssh -L 5432:db.internal:5432 user@bastion  # local port forward
ssh -D 1080 user@host                      # SOCKS proxy
ssh -o StrictHostKeyChecking=accept-new user@host
scp -r dir user@host:/path; rsync -avz -e ssh src/ user@host:/dst/
ssh-add -l; eval "$(ssh-agent)"
sshd -t; sshd -T | grep -i passwordauth
```

```text
# ~/.ssh/config
Host prod
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
```

```text
# /etc/ssh/sshd_config.d/10-hardening.conf
PermitRootLogin no
PasswordAuthentication no
AllowUsers deploy
MaxAuthTries 3
```

Permissions: `~/.ssh` is 700, `authorized_keys` and private keys 600.

## Performance

| Question | Tool |
|----------|------|
| Load, uptime | `uptime`, `nproc` |
| CPU per core | `mpstat -P ALL 1` |
| CPU per process | `pidstat 1`, `top -H` |
| Memory real headroom | `free -m` (read `available`) |
| Swapping | `vmstat 1` (`si`, `so`) |
| Disk saturation | `iostat -xz 1` (`%util`, `await`, `aqu-sz`) |
| Per-process I/O | `iotop -oPa`, `pidstat -d 1` |
| Network throughput | `sar -n DEV 1`, `iftop`, `nload` |
| History | `sar -u -f /var/log/sysstat/saDD` |
| OOM kills | `dmesg -T \| grep -i 'killed process'` |
| Profile | `perf top`, `perf record -g -p PID` |
| Open file limits | `ulimit -n`, `cat /proc/PID/limits` |

USE method: for each resource check Utilization, Saturation, Errors. Load average of N on N cores means about 100% busy; compare with `nproc`.

## Packages

| Task | Debian/Ubuntu | RHEL/Rocky |
|------|---------------|------------|
| Refresh metadata | `apt update` | `dnf makecache` |
| Install | `apt install -y pkg` | `dnf install -y pkg` |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Remove | `apt remove pkg` / `purge` | `dnf remove pkg` |
| Search | `apt search x` | `dnf search x` |
| Show info | `apt show pkg` | `dnf info pkg` |
| List files | `dpkg -L pkg` | `rpm -ql pkg` |
| Owner of file | `dpkg -S /path` | `rpm -qf /path` |
| Hold version | `apt-mark hold pkg` | `dnf versionlock add pkg` |
| Auto security updates | `unattended-upgrades` | `dnf-automatic` |
| History | `/var/log/apt/history.log` | `dnf history` |

## Bash scripting

```bash
#!/usr/bin/env bash
set -euo pipefail            # exit on error, unset var, pipe failure
IFS=$'\n\t'
trap 'echo "failed at line $LINENO" >&2' ERR
trap 'rm -f "$TMP"' EXIT

TMP=$(mktemp)
: "${ENV:?ENV is required}"  # fail if unset
name=${1:-default}

if [[ -f "$file" && -s "$file" ]]; then echo "exists, non-empty"; fi
for f in /var/log/*.log; do [[ -e $f ]] || continue; gzip -k "$f"; done
while IFS= read -r line; do echo "$line"; done < input.txt
case "$1" in start) ... ;; stop) ... ;; *) echo "usage"; exit 2 ;; esac

log() { printf '%s %s\n' "$(date -Is)" "$*"; }
exec 9>/run/job.lock; flock -n 9 || exit 1     # single instance
```

| Test | Meaning |
|------|---------|
| `-f` / `-d` / `-e` | file / directory / exists |
| `-s` / `-x` / `-r` | non-empty / executable / readable |
| `-z` / `-n` | empty string / non-empty string |
| `-eq -ne -lt -gt` | numeric compare |
| `$?`, `$#`, `$@`, `$$`, `$!` | last exit code, arg count, all args, PID, last bg PID |

Always quote variables (`"$var"`), lint with `shellcheck script.sh`, debug with `bash -x script.sh`.

## cron, timers and logrotate

```text
# m h dom mon dow  command        (/etc/cron.d adds a user field after dow)
0 2 * * *      /usr/local/bin/backup.sh
*/5 * * * *    /usr/local/bin/healthcheck.sh
30 3 * * 1-5   /usr/local/bin/report.sh
@reboot        /usr/local/bin/startup.sh
```

```bash
crontab -l; crontab -e; crontab -u user -l
journalctl -u cron -S today        # Debian: cron; RHEL: crond
systemd-analyze calendar "Mon..Fri 02:30"
```

```text
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp adm
    postrotate
        systemctl reload myapp >/dev/null 2>&1 || true
    endscript
}
```

Test: `logrotate -d /etc/logrotate.d/myapp` (dry run), `logrotate -f ...` (force). Cron runs with a minimal `PATH` and no profile, so use absolute paths.

## Containers internals

```bash
lsns                                   # list namespaces
unshare --pid --mount --uts --net --fork --mount-proc bash
nsenter -t PID -n -p ip a              # enter another process's namespaces
cat /sys/fs/cgroup/cgroup.controllers  # cgroups v2
mkdir /sys/fs/cgroup/demo
echo 50M > /sys/fs/cgroup/demo/memory.max
echo "20000 100000" > /sys/fs/cgroup/demo/cpu.max
echo $$ > /sys/fs/cgroup/demo/cgroup.procs
capsh --print; getpcaps PID            # capabilities
```

Namespaces: `pid`, `net`, `mnt`, `uts`, `ipc`, `user`, `cgroup`, `time`. They control what a process **sees**; cgroups control what it can **use**.

---

## Top 15 interview one-liners

1. **Find what is listening on a port:** `ss -tulpn | grep :8080` (or `lsof -i :8080`).
2. **Disk full, find the culprit:** `du -xh / --max-depth=2 2>/dev/null | sort -rh | head -20`, then check `df -i` and `lsof +L1`.
3. **Top 10 IPs hitting nginx:** `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10`.
4. **Kill gracefully, then forcefully:** `kill -15 PID; sleep 5; kill -9 PID`.
5. **Follow a service's logs:** `journalctl -u myapp -f --since "10 min ago"`.
6. **Find files changed in the last hour:** `find / -xdev -type f -mmin -60 2>/dev/null`.
7. **Run something at 2 AM daily:** `0 2 * * * /path/script.sh` in cron, or a systemd timer with `OnCalendar=*-*-* 02:00` and `Persistent=true`.
8. **Why is the load high but CPU idle?** Tasks in `D` state waiting on I/O; check `iostat -xz 1`, `vmstat 1` (`wa`), and `ps -eo stat,cmd | grep '^D'`.
9. **Memory "used" is high, is that bad?** Read `available` in `free -m`, since page cache is reclaimable; worry about swapping (`si`/`so`) and OOM kills in `dmesg`.
10. **Permission denied, find where:** `namei -l /full/path/to/file`, then check SELinux (`ls -Z`, `ausearch -m avc`) or AppArmor.
11. **Zombie process:** it is dead but not reaped; kill or fix the parent (`ps -o ppid= -p PID`), you cannot kill a zombie itself.
12. **Change a setting without reboot and keep it:** `sysctl -w key=value` for now, `/etc/sysctl.d/99-x.conf` plus `sysctl --system` for persistence.
13. **Safely edit SSH config:** keep a second session open, `sshd -t`, then `systemctl reload ssh`.
14. **Hard link vs soft link:** a hard link is another name for the same inode (same filesystem, survives deletion of the original name); a soft link is a path pointer that breaks if the target moves.
15. **What is a container, really?** A normal process with namespaces (what it sees), cgroups (what it can use), capabilities and seccomp (what it may do), on a shared kernel.

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
