# Linux — Troubleshooting Drills

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

> Realistic Linux symptoms with exact error text, diagnosis commands and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Disk space, inodes and filesystems](#disk-space-inodes-and-filesystems) — Q1–Q25
- [Permissions, ownership, SELinux and mounts](#permissions-ownership-selinux-and-mounts) — Q26–Q50
- [Processes, memory and OOM](#processes-memory-and-oom) — Q51–Q75
- [Load, CPU and I/O performance](#load-cpu-and-io-performance) — Q76–Q100
- [systemd services and journald](#systemd-services-and-journald) — Q101–Q125
- [Networking, DNS and firewalls](#networking-dns-and-firewalls) — Q126–Q150
- [SSH, sudo and authentication](#ssh-sudo-and-authentication) — Q151–Q175
- [Boot, time, cron and kernel](#boot-time-cron-and-kernel) — Q176–Q200

---

## Disk space, inodes and filesystems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Writes fail with `No space left on device`, but `df -h` shows the filesystem is only 40% used. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inodes are exhausted (millions of tiny files), or you're looking at the wrong mount.

**Diagnose →**
```bash
df -i /var                          # IUse% at 100%?
sudo find /var -xdev -type f | cut -d/ -f2-4 | sort | uniq -c | sort -rn | head
df -h "$(dirname /path/that/fails)"
```

**Fix →** Delete the small-file pile (session files, mail queue, cache dirs) with `find <dir> -type f -mtime +7 -delete`. If ext4 legitimately needs more inodes, it must be reformatted with `-i`/`-N`, or move the data to XFS.

**Prevent →** Alert on `df -i` as well as `df -h`; set cleanup for session/cache directories.

</details>

### Q2. `df -h` shows `/` at 100%, but `du -sh /` only adds up to 12 GB on a 50 GB disk. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A deleted file is still held open by a process (often a log deleted with `rm` while the app writes to it), or data is hidden under a mount point.

**Diagnose →**
```bash
sudo lsof -nP +L1 | sort -k7 -n | tail      # NLINK 0 = deleted but open
sudo lsof -nP | grep '(deleted)'
```

**Fix →** Restart or reload the process holding the file so it releases it. If you can't restart, truncate through the fd: `sudo truncate -s 0 /proc/<PID>/fd/<FD>`.

**Prevent →** Rotate logs with logrotate (`postrotate` signal or `copytruncate`), never `rm` active logs.

</details>

### Q3. `du` and `df` disagree by 20 GB and `lsof +L1` shows nothing. What else could hide the space? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files written into a directory before a filesystem was mounted on top of it, reserved blocks, or snapshot/metadata space.

**Diagnose →**
```bash
sudo mkdir /mnt/rootonly
sudo mount --bind / /mnt/rootonly
sudo du -xsh /mnt/rootonly/{var,data,srv,mnt} 2>/dev/null
sudo tune2fs -l /dev/nvme0n1p1 | grep -i 'reserved block'
```

**Fix →** Delete or move the hidden files seen through the bind mount, then `umount /mnt/rootonly`. For a data disk, reduce reserved blocks: `tune2fs -m 1 /dev/sdb1`.

**Prevent →** Use `nofail` and monitor that data mounts are actually mounted, so apps don't silently write to root.

</details>

### Q4. A cron job that runs `rm /var/spool/app/*` fails with `/bin/rm: Argument list too long`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The glob expands to more arguments than `ARG_MAX` allows.

**Diagnose →**
```bash
getconf ARG_MAX
find /var/spool/app -maxdepth 1 -type f | wc -l
```

**Fix →** `find /var/spool/app -maxdepth 1 -type f -delete`, or `find ... -print0 | xargs -0 rm`.

**Prevent →** Use `find` in scripts for anything that can grow; clean the directory more often.

</details>

### Q5. `touch /data/test` returns `touch: cannot touch '/data/test': Read-only file system`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kernel remounted it read-only after I/O or filesystem errors (`errors=remount-ro`), it was mounted `ro`, or the underlying device is read-only (EBS issue, snapshot).

**Diagnose →**
```bash
findmnt /data
dmesg -T | grep -Ei 'EXT4-fs error|XFS|I/O error|remount'
sudo smartctl -H /dev/sdb 2>/dev/null
```

**Fix →** If it was a mount option, `mount -o remount,rw /data`. If errors caused it: stop apps, unmount, run `e2fsck -f` or `xfs_repair`, then mount again. Replace the disk if errors are hardware.

**Prevent →** Alert on filesystem errors in kernel logs and on SMART health.

</details>

### Q6. After a power failure, mounting an XFS volume fails with `mount: /data: wrong fs type, bad option, bad superblock on /dev/sdb1`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Corrupted log/superblock, wrong device name, or the XFS log needs replay.

**Diagnose →**
```bash
lsblk -f
sudo blkid /dev/sdb1
dmesg -T | tail -30
sudo xfs_repair -n /dev/sdb1       # dry run
```

**Fix →** Mount once normally to replay the log; if it fails, run `xfs_repair /dev/sdb1`. Use `xfs_repair -L` (zero the log) only as a last resort, as it can lose recent changes. Take a snapshot first.

**Prevent →** UPS/redundancy, and storage with write cache protection.

</details>

### Q7. `mount -a` says `mount: /backup: can't find UUID=3f1a...` after someone recreated the filesystem. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `mkfs` generated a new UUID; fstab still has the old one.

**Diagnose →**
```bash
sudo blkid
grep backup /etc/fstab
```

**Fix →** Update fstab with the new UUID, then `sudo systemctl daemon-reload && sudo mount -a`.

**Prevent →** Use labels (`mkfs.xfs -L backup` and `LABEL=backup`) when disks get recreated, and add `nofail` to non-critical mounts.

</details>

### Q8. Disk filled up overnight and `du` points at `/var/log/journal`, which is 18 GB. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No size limit on persistent journald, plus a chatty service logging in a loop.

**Diagnose →**
```bash
journalctl --disk-usage
journalctl --since "1 hour ago" -o short | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

**Fix →** `sudo journalctl --vacuum-size=1G`, fix or quiet the noisy service, and set `SystemMaxUse=1G` in `/etc/systemd/journald.conf`, then `systemctl restart systemd-journald`.

**Prevent →** Keep the size cap in your baseline; use `RateLimitBurst`/`RateLimitIntervalSec` against log storms.

</details>

### Q9. `/var/lib/docker` keeps filling the root disk on a build host. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dangling images, build cache and stopped containers accumulate.

**Diagnose →**
```bash
docker system df -v
sudo du -sh /var/lib/docker/*
```

**Fix →** `docker system prune -af --filter "until=72h"` and `docker builder prune -af`. Move Docker's data root to a separate volume (`"data-root": "/data/docker"` in `/etc/docker/daemon.json`).

**Prevent →** Scheduled prune job on build hosts and a dedicated volume for container data.

</details>

### Q10. An NFS client shows `df` hanging forever, and the shell becomes unusable. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An NFS server is unreachable and the mount is `hard`, so `statfs()` blocks in D state.

**Diagnose →**
```bash
mount -t nfs,nfs4
df -h -x nfs -x nfs4          # works without touching NFS
rpcinfo -t nfsserver nfs 4 ; showmount -e nfsserver
ps -eo pid,stat,cmd | awk '$2 ~ /D/'
```

**Fix →** Restore the server or network. If the server is gone for good, `umount -f -l /mnt/share`.

**Prevent →** Use autofs for rarely used shares; monitor NFS servers; avoid NFS mounts on paths that system tools scan.

</details>

### Q11. NFS-mounted files show owner `nobody` and `chown` fails with `Operation not permitted`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NFSv4 ID mapping domain mismatch between client and server, or `root_squash` applying to root on the client.

**Diagnose →**
```bash
grep -i domain /etc/idmapd.conf       # on client and server
nfsidmap -l 2>/dev/null
cat /etc/exports                      # on server
```

**Fix →** Set the same `Domain = example.internal` in `/etc/idmapd.conf` on both, run `nfsidmap -c`, remount. Do ownership changes on the server, or use `no_root_squash` only where truly required.

**Prevent →** Central identity with matching UIDs; consistent idmapd config in your baseline.

</details>

### Q12. `xfs_growfs /data` fails with `xfs_growfs: /data is not a mounted XFS filesystem` right after `lvextend`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You passed the device instead of the mount point on an older xfsprogs, the filesystem is ext4, or a typo in the path.

**Diagnose →**
```bash
findmnt -no FSTYPE,SOURCE /data
lsblk -f
```

**Fix →** For XFS: `xfs_growfs /data` (mount point). For ext4: `resize2fs /dev/vg0/data`. Next time use `lvextend -r` to resize both in one step.

**Prevent →** Use `lvextend -r` in runbooks.

</details>

### Q13. `lvextend` fails with `Insufficient free space: 2560 extents needed, but only 0 available`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The volume group has no free extents.

**Diagnose →**
```bash
sudo vgs
sudo pvs
lsblk
```

**Fix →** Add a disk (`pvcreate`, `vgextend`), or grow the existing PV after enlarging the disk (`growpart` then `pvresize /dev/sda2`), then retry `lvextend -r`.

**Prevent →** Leave headroom in the VG and alert on `vg_free`.

</details>

### Q14. A disk shows up in `lsblk` but `pvcreate` says `Cannot use /dev/sdc: device is partitioned`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Leftover partition table or filesystem signatures from a previous use.

**Diagnose →**
```bash
lsblk -f /dev/sdc
sudo wipefs /dev/sdc
```

**Fix →** After confirming the disk is really unused: `sudo wipefs -a /dev/sdc`, then `pvcreate /dev/sdc`.

**Prevent →** Wipe disks when decommissioning; always double-check the device name before wiping.

</details>

### Q15. `ls` on a directory prints `ls: cannot access 'file': Input/output error` and `dmesg` shows `blk_update_request: I/O error, dev sda, sector ...`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Failing disk (bad sectors), controller or cable fault, or a cloud volume impairment.

**Diagnose →**
```bash
dmesg -T | grep -Ei 'I/O error|medium|reset'
sudo smartctl -a /dev/sda | grep -Ei 'Reallocated|Pending|Uncorrectable'
cat /proc/mdstat
```

**Fix →** Copy off readable data immediately (`ddrescue /dev/sda /mnt/new/sda.img /mnt/new/map` for a failing disk). Replace the disk or restore the volume from snapshot, then `fsck` the copy.

**Prevent →** RAID or replicated storage, SMART monitoring, tested backups.

</details>

### Q16. An LVM snapshot used for backups suddenly shows `Invalid` and the backup failed. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The snapshot's copy-on-write space filled because the origin changed more than expected.

**Diagnose →**
```bash
sudo lvs -o lv_name,origin,data_percent,lv_attr
dmesg | grep -i snapshot
```

**Fix →** Remove it (`lvremove`) and create a bigger one (`-L` sized to the change rate during the backup), or enable autoextend: `snapshot_autoextend_threshold = 70` and `snapshot_autoextend_percent = 20` in `/etc/lvm/lvm.conf`.

**Prevent →** Keep snapshots short-lived and monitor `data_percent`.

</details>

### Q17. A swap file won't activate: `swapon: /swapfile: insecure permissions 0644, 0600 suggested` followed by `swapon failed: Invalid argument`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The permissions warning is cosmetic; `Invalid argument` usually means the file has holes (created with `fallocate` on a filesystem that doesn't support it for swap, like older XFS or btrfs) or wasn't `mkswap`ed.

**Diagnose →**
```bash
ls -l /swapfile; file /swapfile
findmnt -no FSTYPE /
```

**Fix →**
```bash
sudo swapoff /swapfile 2>/dev/null; sudo rm /swapfile
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
```
On btrfs use `btrfs filesystem mkswapfile`.

**Prevent →** Use `dd` for swap files in scripts that must work on any filesystem.

</details>

### Q18. `rm -rf` on a directory reports `Directory not empty` even though `ls -la` shows nothing inside. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something is mounted inside it, a process keeps creating files, or (on NFS) `.nfsXXXX` silly-rename files for files still open on a client.

**Diagnose →**
```bash
findmnt -R /path/dir
ls -la /path/dir; sudo lsof +D /path/dir
```

**Fix →** Unmount nested mounts, stop the writing process, or close the files on the NFS client (the `.nfs*` files disappear afterwards). Then retry.

**Prevent →** Stop services before cleanup; avoid deleting files still in use on NFS.

</details>

### Q19. A file can't be deleted even by root: `rm: cannot remove 'resolv.conf': Operation not permitted`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The immutable (`i`) or append-only (`a`) attribute is set.

**Diagnose →**
```bash
lsattr /etc/resolv.conf
```

**Fix →** `sudo chattr -i /etc/resolv.conf`, make the change, then re-apply if that was intentional.

**Prevent →** Document when you set `chattr +i`; config tools fail strangely when they hit immutable files.

</details>

### Q20. A database server shows `EXT4-fs warning: ext4_dx_add_entry: Directory index full!` and inserts start failing. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A single directory has reached the ext4 htree limit (millions of entries).

**Diagnose →**
```bash
dmesg -T | grep -i 'index full'
sudo find /data -xdev -type d -exec sh -c 'echo "$(ls -f "$1" | wc -l) $1"' _ {} \; | sort -rn | head
sudo tune2fs -l /dev/sdb1 | grep features
```

**Fix →** Move files into subdirectories. On newer kernels you can enable `large_dir`: `tune2fs -O large_dir /dev/sdb1` (check kernel and e2fsprogs support first).

**Prevent →** Application-level directory sharding; alert on directory entry counts.

</details>

### Q21. After restoring files from a tar backup, the app fails because every file is owned by `root`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The archive was extracted as a non-root user (ownership dropped) or with `--no-same-owner`, or the UIDs differ on the new host.

**Diagnose →**
```bash
tar -tvzf backup.tgz | head
ls -ln /srv/app | head
getent passwd app
```

**Fix →** Re-extract as root with `tar --same-owner -xpzf backup.tgz`, or `chown -R app:app /srv/app`. Use `--numeric-owner` when UIDs matter more than names.

**Prevent →** Back up and restore as root with `-p` (preserve permissions), and keep UIDs consistent.

</details>

### Q22. An app logs `Disk quota exceeded` but `df` shows plenty of space. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app's user or group has reached its quota.

**Diagnose →**
```bash
sudo quota -u appuser
sudo repquota -a                     # ext4
sudo xfs_quota -x -c 'report -h' /home   # XFS
```

**Fix →** Clean up the user's files or raise the limit with `setquota` / `xfs_quota -x -c 'limit bhard=20g appuser'`.

**Prevent →** Warn on soft-limit breaches before the hard limit hits.

</details>

### Q23. Mounting a disk cloned from another server fails: `mount: /mnt/clone: wrong fs type ... ` and `dmesg` says `XFS (nvme1n1p1): Filesystem has duplicate UUID`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The clone has the same XFS UUID as a mounted filesystem (common when mounting a snapshot of the root volume on the same instance).

**Diagnose →**
```bash
sudo blkid
dmesg -T | tail
```

**Fix →** Mount with `-o nouuid` for a temporary mount, or give it a new UUID permanently: `sudo xfs_admin -U generate /dev/nvme1n1p1`.

**Prevent →** Use `nouuid` in rescue runbooks.

</details>

### Q24. Disk usage on `/tmp` keeps growing and `du` shows a few huge files named like `/tmp/php*` or `/tmp/sort*`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Applications or tools (`sort`, uploads, DB temp files) writing big temp files and not cleaning up, or crashing mid-way.

**Diagnose →**
```bash
sudo du -ah /tmp | sort -h | tail
sudo lsof +D /tmp | head
systemctl status systemd-tmpfiles-clean.timer
```

**Fix →** Remove files not held open; set `TMPDIR` for big jobs to a dedicated volume.

**Prevent →** Make sure `systemd-tmpfiles-clean.timer` runs with an age rule (`/usr/lib/tmpfiles.d/tmp.conf`), or mount `/tmp` as size-limited tmpfs.

</details>

### Q25. A RAID 1 array reports `[U_]` in `/proc/mdstat` and the server is slow. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One mirror member failed or dropped out; the array may also be resyncing, which consumes I/O.

**Diagnose →**
```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
dmesg -T | grep -Ei 'md0|raid1|sd[a-z]'
sudo smartctl -a /dev/sdb
```

**Fix →** Replace the failed disk and `mdadm /dev/md0 --add /dev/sdX`. If a member just dropped out but is healthy, re-add it: `mdadm /dev/md0 --re-add /dev/sdb1`. Limit resync impact with `echo 50000 > /proc/sys/dev/raid/speed_limit_max`.

**Prevent →** `mdmonitor` with email alerts; a write-intent bitmap (`mdadm --grow --bitmap=internal /dev/md0`) for faster resyncs.

</details>

## Permissions, ownership, SELinux and mounts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. nginx returns 403 and logs `open() "/srv/www/index.html" failed (13: Permission denied)`, even after `chmod -R 777 /srv/www`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux is denying access because the files have the wrong context (`default_t` or `user_home_t` instead of `httpd_sys_content_t`), or a parent directory lacks `x` for nginx.

**Diagnose →**
```bash
getenforce
ls -Z /srv/www/index.html
sudo ausearch -m AVC -ts recent | grep nginx
namei -l /srv/www/index.html          # permissions on every path component
```

**Fix →**
```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/www(/.*)?'
sudo restorecon -Rv /srv/www
sudo chmod -R u=rwX,go=rX /srv/www    # undo the 777
```

**Prevent →** Use standard paths (`/var/www`) or register custom paths with `semanage fcontext` in config management.

</details>

### Q27. A script fails with `bash: ./deploy.sh: Permission denied` even though `ls -l` shows `-rwxr-xr-x`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The filesystem is mounted with `noexec` (common on `/tmp`, `/home` or USB/NFS mounts).

**Diagnose →**
```bash
findmnt -no OPTIONS --target ./deploy.sh
```

**Fix →** Run it through the interpreter (`bash ./deploy.sh`), or move it to a filesystem without `noexec`. Don't drop `noexec` from `/tmp` just for this.

**Prevent →** Keep scripts in `/opt` or `/usr/local/bin`; set `TMPDIR` for installers that execute from `/tmp`.

</details>

### Q28. A script fails with `/bin/bash^M: bad interpreter: No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Windows CRLF line endings; the kernel looks for an interpreter literally named `bash\r`.

**Diagnose →**
```bash
file deploy.sh            # "with CRLF line terminators"
head -1 deploy.sh | cat -A
```

**Fix →** `sed -i 's/\r$//' deploy.sh` or `dos2unix deploy.sh`.

**Prevent →** Add `*.sh text eol=lf` to `.gitattributes` and a pre-commit check.

</details>

### Q29. A user gets `-bash: cd: /srv/project: Permission denied` although they're in the group that owns `/srv/project` (mode `770`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Their current session was started before they were added to the group, or a parent directory blocks traversal.

**Diagnose →**
```bash
id                         # in their session
id alice                   # from the database
namei -l /srv/project
```

**Fix →** Log out and back in (or `newgrp devteam`). Add `x` for the group on parent directories if missing.

**Prevent →** Tell users group changes need a new login; restart services after adding service users to groups.

</details>

### Q30. Apache can't connect to a backend: `AH00957: HTTP: attempt to connect to 127.0.0.1:8080 (localhost) failed` and `ausearch` shows `denied { name_connect }`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux blocks `httpd_t` from making outbound network connections by default.

**Diagnose →**
```bash
sudo ausearch -m AVC -c httpd -ts recent
getsebool httpd_can_network_connect
```

**Fix →** `sudo setsebool -P httpd_can_network_connect on` (or the narrower `httpd_can_network_relay`). If the backend port is unusual, label it: `semanage port -a -t http_port_t -p tcp 8080` (8080 is usually already `http_cache_port_t`, which is allowed).

**Prevent →** Include required SELinux booleans in the role that deploys the reverse proxy.

</details>

### Q31. A service fails to bind to a custom port: `bind() to 0.0.0.0:8888 failed (13: Permission denied)` while running as root. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux only allows the service's domain to bind ports with specific labels.

**Diagnose →**
```bash
sudo ausearch -m AVC -ts recent | grep name_bind
sudo semanage port -l | grep -w 8888
```

**Fix →** `sudo semanage port -a -t http_port_t -p tcp 8888` (use the type matching the service), then restart.

**Prevent →** Prefer standard ports or label custom ones in automation.

</details>

### Q32. Files moved with `mv` from a home directory into `/var/www/html` give 403 errors, but files created there directly work. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `mv` keeps the original SELinux context (`user_home_t`); `cp` creates new files with the destination's context.

**Diagnose →**
```bash
ls -Z /var/www/html
matchpathcon /var/www/html/page.html
```

**Fix →** `sudo restorecon -Rv /var/www/html`.

**Prevent →** Deploy with `cp`, `rsync` or packages, or run `restorecon` as part of deployment.

</details>

### Q33. After restoring `/etc/shadow` from backup on an SELinux system, nobody can log in and logs show `pam_unix(sshd:auth): authentication failure`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The restored file has the wrong SELinux label (or wrong mode), so login processes can't read it.

**Diagnose →** From console or rescue:
```bash
ls -lZ /etc/shadow            # expect ----------  root root system_u:object_r:shadow_t
sudo ausearch -m AVC -ts today | grep shadow
```

**Fix →** `restorecon -v /etc/shadow` and `chmod 000 /etc/shadow` (RHEL) or `640 root:shadow` (Debian). If you can't log in at all, boot with `rd.break` or rescue media, fix the file and `touch /.autorelabel`.

**Prevent →** Restore system files with tools that keep xattrs (`tar --selinux`, `rsync -X`) and always run `restorecon` after.

</details>

### Q34. A container with a bind mount gets `Permission denied` reading files on a RHEL/Fedora host, even with matching UIDs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux blocks the container (`container_t`) from host-labelled files.

**Diagnose →**
```bash
ls -Z /srv/data
sudo ausearch -m AVC -ts recent | grep container_t
```

**Fix →** Mount with a relabel flag: `-v /srv/data:/data:Z` (private) or `:z` (shared between containers). Don't use `:Z` on system directories like `/home` or `/etc`.

**Prevent →** Keep container data in dedicated directories labelled `container_file_t`.

</details>

### Q35. `sudo -u app touch /var/log/app/x.log` works, but the app itself (running as `app` under systemd) gets `EACCES` on the same path. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd sandboxing: `ProtectSystem=strict` makes `/var` read-only unless listed in `ReadWritePaths=`, or `ProtectHome`, `PrivateTmp`, or an SELinux domain differs between the shell and the service.

**Diagnose →**
```bash
systemctl show app -p ProtectSystem -p ReadWritePaths -p ProtectHome
systemctl cat app
sudo ausearch -m AVC -ts recent
```

**Fix →** Add `ReadWritePaths=/var/log/app` (or `LogsDirectory=app`, which creates and permits `/var/log/app`), then `daemon-reload` and restart.

**Prevent →** Use `StateDirectory=`, `LogsDirectory=`, `RuntimeDirectory=` instead of hand-made directories.

</details>

### Q36. An upload directory is set `2775` and owned by group `web`, but new files still get the user's primary group. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Files are moved in (keep their group) rather than created there, or the app sets the group explicitly, or the directory was replaced by a deploy without setgid.

**Diagnose →**
```bash
stat -c '%A %U:%G %n' /srv/uploads
ls -l /srv/uploads | head
```

**Fix →** Re-apply `chmod g+s /srv/uploads`, `chgrp -R web /srv/uploads`; change the app to create files in place, or use a default ACL: `setfacl -d -m g:web:rwX /srv/uploads`.

**Prevent →** Enforce directory modes in config management.

</details>

### Q37. `chmod` on a file under `/mnt/usb` returns success but the permissions don't change. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The filesystem (FAT/exFAT/NTFS) doesn't store Unix permissions; they come from mount options.

**Diagnose →**
```bash
findmnt -no FSTYPE,OPTIONS /mnt/usb
```

**Fix →** Remount with the desired owner and mask: `mount -o uid=1000,gid=1000,umask=022 /dev/sdb1 /mnt/usb`.

**Prevent →** Use ext4/XFS for anything needing Unix permissions.

</details>

### Q38. Users can delete each other's files in a shared `/srv/shared` directory with mode `777`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without the sticky bit, anyone with write on a directory can delete any file in it.

**Diagnose →**
```bash
stat -c '%A' /srv/shared
```

**Fix →** `chmod 1777 /srv/shared` (or better, a group with `2770` + sticky: `chmod 3770`).

**Prevent →** Use group-based shared dirs instead of world-writable ones.

</details>

### Q39. An ACL was set with `setfacl -m u:bob:rw file`, but Bob still gets `Permission denied` writing to it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ACL mask limits effective permissions (a later `chmod` reduced the mask), or Bob can't traverse a parent directory.

**Diagnose →**
```bash
getfacl file          # look for "#effective:r--"
namei -l /path/to/file
```

**Fix →** `setfacl -m m::rw file` to raise the mask, and give `x` on parent directories (`setfacl -m u:bob:x /path /path/to`).

**Prevent →** Remember `chmod g=` changes the ACL mask on files with ACLs.

</details>

### Q40. A setuid binary you built doesn't run as root: it still runs with the caller's UID. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The filesystem is mounted `nosuid`, the binary is a script (the kernel ignores setuid on scripts), or the owner isn't root.

**Diagnose →**
```bash
ls -l /opt/tool/bin/helper            # expect -rwsr-xr-x root
findmnt -no OPTIONS --target /opt/tool/bin/helper
file /opt/tool/bin/helper
```

**Fix →** Avoid setuid entirely: use a narrow sudo rule or file capabilities (`setcap cap_net_raw+ep helper`). If setuid is really needed, place a compiled binary on a filesystem without `nosuid`.

**Prevent →** Treat any setuid requirement as a security review item.

</details>

### Q41. `ping` fails for normal users with `ping: socket: Operation not permitted` on a minimal image. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ping` lost its file capability (`cap_net_raw`) when copied or installed by a tool that strips xattrs, and unprivileged ICMP sockets are disabled.

**Diagnose →**
```bash
getcap /usr/bin/ping
sysctl net.ipv4.ping_group_range
```

**Fix →** `sudo setcap cap_net_raw+ep /usr/bin/ping`, or allow ICMP sockets: `sysctl -w net.ipv4.ping_group_range="0 2147483647"` (persist in `/etc/sysctl.d/`). Reinstalling the `iputils` package also restores it.

**Prevent →** Use image build tools that preserve xattrs (`tar --xattrs`).

</details>

### Q42. A user's crontab edit fails with `You (alice) are not allowed to use this program (crontab)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `/etc/cron.allow` exists and doesn't list the user, or the user is in `/etc/cron.deny`.

**Diagnose →**
```bash
ls -l /etc/cron.allow /etc/cron.deny
grep -w alice /etc/cron.allow /etc/cron.deny
```

**Fix →** Add the user to `cron.allow` (or remove them from `cron.deny`) if policy allows.

**Prevent →** Manage these files in config management with the approved user list.

</details>

### Q43. AppArmor on Ubuntu blocks MySQL from using a new data directory: `mysqld: Can't create/write to file '/data/mysql/...' (Errcode: 13)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `usr.sbin.mysqld` AppArmor profile only allows `/var/lib/mysql`.

**Diagnose →**
```bash
sudo aa-status | grep mysqld
sudo journalctl -k | grep -i apparmor | grep DENIED
```

**Fix →** Add the path to `/etc/apparmor.d/local/usr.sbin.mysqld`:
```
/data/mysql/ r,
/data/mysql/** rwk,
```
Then `sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.mysqld` and restart MySQL. Or bind-mount `/data/mysql` onto `/var/lib/mysql`.

**Prevent →** Use the `local/` override files so package updates don't wipe your changes.

</details>

### Q44. Writing to a file on an NFS mount fails with `Permission denied` for root, but works for the file owner. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `root_squash` on the export maps client root to `nobody`.

**Diagnose →**
```bash
cat /etc/exports           # on the server
ls -ln /mnt/share/file
```

**Fix →** Perform the action as the owning user (`sudo -u app ...`), or use `no_root_squash` only for trusted admin hosts.

**Prevent →** Run apps as consistent non-root UIDs across clients.

</details>

### Q45. After a package update, an app can't read its key: `Permission denied` on `/etc/pki/tls/private/app.key`, and the mode looks right. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux label on the key is wrong (it was copied from elsewhere, `admin_home_t`), or a policy update tightened rules for that domain.

**Diagnose →**
```bash
ls -lZ /etc/pki/tls/private/app.key
matchpathcon /etc/pki/tls/private/app.key
sudo ausearch -m AVC -ts today -i | tail
```

**Fix →** `sudo restorecon -v /etc/pki/tls/private/app.key`. If the denial is for a custom domain, add a reviewed local policy module (`audit2allow -M appkey`).

**Prevent →** Place keys with `install -m 600 -Z` or run `restorecon` in the deploy step.

</details>

### Q46. A shell script works when run with `bash script.sh` but fails with `sh: 3: [[: not found` from cron. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cron or a missing shebang runs it with `/bin/sh` (dash on Debian/Ubuntu), which doesn't support bash features.

**Diagnose →**
```bash
head -1 script.sh
ls -l /bin/sh
```

**Fix →** Add `#!/usr/bin/env bash` as the first line and call the script directly, or set `SHELL=/bin/bash` in the crontab.

**Prevent →** Run `shellcheck` and always include a shebang.

</details>

### Q47. A file shows as owned by a numeric UID like `1003` instead of a username. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user was deleted, the files came from another system with a different UID mapping, or the identity service (SSSD/LDAP) is unreachable.

**Diagnose →**
```bash
getent passwd 1003
systemctl status sssd
```

**Fix →** Restore the identity service, recreate the user with that UID, or `chown` to the correct current user.

**Prevent →** Use `userdel` with a file-ownership cleanup step; keep UIDs consistent across hosts.

</details>

### Q48. `sudo` prints `sudo: /etc/sudoers.d/ops is world writable` and then fails. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A sudoers include file has unsafe permissions or ownership, so sudo refuses it.

**Diagnose →**
```bash
ls -l /etc/sudoers /etc/sudoers.d/
```

**Fix →** As root (via `pkexec`, console, or another working sudo rule): `chmod 0440 /etc/sudoers.d/ops && chown root:root /etc/sudoers.d/ops`. Check syntax with `visudo -c`.

**Prevent →** Deploy sudoers files with `validate: visudo -cf %s` and `mode: 0440` in Ansible.

</details>

### Q49. An application running as a non-root user can't write core dumps or create files larger than 1 GB: `File size limit exceeded (core dumped)`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `RLIMIT_FSIZE` is set (via `/etc/security/limits.conf` `fsize`, shell `ulimit -f`, or the systemd unit `LimitFSIZE=`).

**Diagnose →**
```bash
cat /proc/<PID>/limits | grep -i 'file size'
grep -r fsize /etc/security/limits.conf /etc/security/limits.d/
systemctl show app -p LimitFSIZE
```

**Fix →** Remove or raise the limit in the right place (`LimitFSIZE=infinity` for systemd services; `limits.conf` only applies to PAM sessions), then restart.

**Prevent →** Know that systemd services ignore `limits.conf`; set limits in the unit.

</details>

### Q50. After mounting a new volume over `/var/www/uploads`, the app can no longer write there. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The new filesystem's root directory is owned by `root:root` with `755`, hiding the old directory's ownership; on SELinux hosts the new mount root has the wrong label.

**Diagnose →**
```bash
ls -ldZ /var/www/uploads
findmnt /var/www/uploads
```

**Fix →** `chown www-data:www-data /var/www/uploads` (on the mounted FS), set the mode, and `restorecon -Rv /var/www/uploads`.

**Prevent →** Include ownership and labels in the step that adds a mount.

</details>

## Processes, memory and OOM

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. A Java service disappears without an error in its own log. `dmesg` shows `Out of memory: Killed process 4121 (java) total-vm:... anon-rss:...`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kernel OOM killer picked the biggest process when the host (or its cgroup) ran out of memory.

**Diagnose →**
```bash
dmesg -T | grep -Ei 'out of memory|oom-kill|killed process'
journalctl -k -b | grep -i oom
free -h; ps aux --sort=-rss | head
```

**Fix →** Find out who used the memory (the OOM report lists every process's RSS). Size the JVM heap to fit (`-Xmx` well below RAM, leaving room for metaspace, threads and the OS), add RAM, or move other workloads off.

**Prevent →** Alert on memory available and on `oom_kill` counters (`/proc/vmstat`), and set per-service memory limits.

</details>

### Q52. A service gets killed with `Memory cgroup out of memory: Killed process ...` but the host has 20 GB free. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service's cgroup limit (`MemoryMax=` in systemd or a container limit) was reached, not host memory.

**Diagnose →**
```bash
systemctl show app -p MemoryMax -p MemoryCurrent -p MemoryPeak
cat /sys/fs/cgroup/system.slice/app.service/memory.events   # oom_kill count
```

**Fix →** Raise the limit if the usage is legitimate, or fix the leak. Use `MemoryHigh=` slightly below `MemoryMax=` to throttle before killing.

**Prevent →** Track `memory.current` against the limit per service.

</details>

### Q53. `ps` shows dozens of processes in state `Z` with `<defunct>`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Zombie processes: children have exited, but the parent hasn't called `wait()` to reap them.

**Diagnose →**
```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
ps -o pid,cmd -p <PPID>
```

**Fix →** You can't kill a zombie; it's already dead. Signal the parent (`kill -CHLD <PPID>`) or restart the parent; once the parent exits, PID 1 adopts and reaps the zombies.

**Prevent →** Fix the parent's child handling; in containers use an init like `tini` (`docker run --init`).

</details>

### Q54. Zombie count keeps climbing into the thousands and new processes fail with `fork: Resource temporarily unavailable`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Zombies hold PIDs; the user or system has hit `pids.max`, `ulimit -u` (nproc), or `kernel.pid_max`.

**Diagnose →**
```bash
ps -eo stat | grep -c Z
cat /proc/sys/kernel/pid_max
ulimit -u
systemctl show app -p TasksMax -p TasksCurrent
```

**Fix →** Restart the parent that isn't reaping (frees all its zombies at once). Temporarily raise `TasksMax=` or `nproc` only if the load is legitimate.

**Prevent →** Fix the reaping bug; alert on zombie count and task counts per service.

</details>

### Q55. An app crashes under load with `java.io.IOException: Too many open files` (or `EMFILE`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process hit its open-file limit (`RLIMIT_NOFILE`), often 1024 by default, or it's leaking file descriptors/sockets.

**Diagnose →**
```bash
cat /proc/<PID>/limits | grep 'open files'
ls /proc/<PID>/fd | wc -l
sudo lsof -p <PID> | awk '{print $5}' | sort | uniq -c | sort -rn | head
```

**Fix →** For a systemd service, set `LimitNOFILE=65536` in a drop-in, `daemon-reload` and restart. If the count grows forever, fix the leak (unclosed sockets/files).

**Prevent →** Monitor fd count vs limit per process.

</details>

### Q56. You raised `nofile` in `/etc/security/limits.conf` to 65536, but the service still shows `Max open files 1024`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `limits.conf` is applied by PAM to login sessions; systemd services don't go through PAM.

**Diagnose →**
```bash
systemctl show app -p LimitNOFILE
cat /proc/$(systemctl show -p MainPID --value app)/limits | grep 'open files'
```

**Fix →**
```bash
sudo systemctl edit app
# [Service]
# LimitNOFILE=65536
sudo systemctl restart app
```

**Prevent →** Set limits in unit files; use `DefaultLimitNOFILE=` in `/etc/systemd/system.conf` for a host-wide default.

</details>

### Q57. The whole host can't open files: `VFS: file-max limit 98304 reached` in `dmesg`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The system-wide file handle limit was reached, usually by one leaking process.

**Diagnose →**
```bash
cat /proc/sys/fs/file-nr          # allocated  unused  max
for p in /proc/[0-9]*; do echo "$(ls $p/fd 2>/dev/null | wc -l) $(cat $p/comm 2>/dev/null)"; done | sort -rn | head
```

**Fix →** Restart the leaking process. Raise the limit if needed: `sysctl -w fs.file-max=2097152`, persisted in `/etc/sysctl.d/`.

**Prevent →** Alert on `file-nr` usage; set per-service `LimitNOFILE` so one process can't take everything.

</details>

### Q58. `kill -9 <PID>` doesn't kill a process; it stays in state `D`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process is in uninterruptible sleep, waiting on I/O in the kernel (dead NFS server, failing disk, hung storage driver). Signals are only delivered when it leaves that state.

**Diagnose →**
```bash
ps -o pid,stat,wchan:32,cmd -p <PID>
sudo cat /proc/<PID>/stack
dmesg -T | grep -Ei 'hung_task|blocked for more than'
```

**Fix →** Fix what it's waiting on: restore the NFS server, `umount -f` the dead mount, or replace the failing device. If the kernel is stuck, a reboot is the only option.

**Prevent →** Monitor storage and NFS health; avoid hard NFS mounts on critical paths.

</details>

### Q59. A process ignores `kill <PID>` (SIGTERM) and keeps running. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process catches or ignores SIGTERM, is busy in a long shutdown, or you're signalling a wrapper and not the real worker.

**Diagnose →**
```bash
grep -E 'SigCgt|SigIgn' /proc/<PID>/status
pstree -p <PID>
```

**Fix →** Wait for graceful shutdown; check the app's logs. If it's truly stuck, `kill -KILL <PID>`. For process trees, signal the group: `kill -TERM -- -<PGID>`.

**Prevent →** Make apps handle SIGTERM properly; in containers, make the app PID 1 or use `exec` in entrypoint scripts.

</details>

### Q60. The server has 64 GB RAM, `free` shows only 2 GB `free`, and the team wants to add RAM. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Page cache is using the rest, which is normal.

**Diagnose →**
```bash
free -h                         # check "available"
cat /proc/pressure/memory       # some/full stall percentages
vmstat 1 5                      # si/so swap activity
```

**Fix →** Nothing, if `available` is healthy and there's no swapping or memory pressure.

**Prevent →** Alert on `MemAvailable` and PSI, never on "free".

</details>

### Q61. Memory usage keeps climbing but no process in `top` shows high RSS. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kernel memory: slab caches (dentries/inodes from millions of files), a leaking kernel module, tmpfs/shm usage, or huge pages reserved.

**Diagnose →**
```bash
grep -E 'Slab|SReclaimable|SUnreclaim|Shmem|HugePages_Total|Hugetlb' /proc/meminfo
sudo slabtop -o | head -20
df -h -t tmpfs
```

**Fix →** Shmem: clean up `/dev/shm` or tmpfs files. Large `SReclaimable` dentry cache is reclaimable under pressure (test with `echo 2 > /proc/sys/vm/drop_caches`, don't schedule it). Growing `SUnreclaim` suggests a kernel/driver leak: update the kernel or driver.

**Prevent →** Graph slab and shmem in monitoring, not just process RSS.

</details>

### Q62. A Node.js process crashes with `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** V8's heap limit was reached (default depends on version and available memory), not the OS limit; or a leak.

**Diagnose →**
```bash
node -e 'console.log(v8.getHeapStatistics().heap_size_limit/1024/1024)'
ps -o rss,cmd -p <PID>
```

**Fix →** Raise the heap if usage is legitimate: `NODE_OPTIONS=--max-old-space-size=3072`, keeping it below the container/cgroup limit. Take heap snapshots (`--heapsnapshot-near-heap-limit=2`) to find leaks.

**Prevent →** Align runtime heap limits with cgroup limits; monitor heap usage.

</details>

### Q63. A server becomes unresponsive for minutes at a time and then recovers; `vmstat` during the event shows high `si`/`so`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Heavy swapping (thrashing): the working set exceeds RAM and the kernel spends its time moving pages.

**Diagnose →**
```bash
vmstat 1
sar -W 1 5                         # pswpin/s pswpout/s
for f in /proc/[0-9]*/status; do awk '/^Name|^VmSwap/{printf "%s ", $2} END{print ""}' $f; done | sort -k2 -n | tail
cat /proc/pressure/memory
```

**Fix →** Find the process pushing memory, reduce its footprint or limit it with `MemoryHigh=`. Add RAM if the workload legitimately needs it.

**Prevent →** Use `systemd-oomd` or earlyoom to kill runaway processes before the host thrashes.

</details>

### Q64. The OOM killer keeps killing `sshd` and the database instead of the leaking batch job. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** OOM victim selection is based on `oom_score` (memory use plus `oom_score_adj`); priorities aren't set.

**Diagnose →**
```bash
for p in $(pgrep -d' ' 'sshd|postgres|batch'); do echo "$p $(cat /proc/$p/oom_score) $(cat /proc/$p/comm)"; done
```

**Fix →** Protect critical services in their units: `OOMScoreAdjust=-900` for the DB and sshd (`-1000` disables killing). Make the batch job preferred: `OOMScoreAdjust=500` and cap it with `MemoryMax=`.

**Prevent →** Every service gets a memory limit and an OOM priority in its unit.

</details>

### Q65. A process runs much slower than on other identical hosts; `top` shows it at 100% in `%st`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CPU steal: the hypervisor is giving CPU time to other VMs (noisy neighbour, or burstable instance out of credits).

**Diagnose →**
```bash
mpstat -P ALL 1 5          # %steal column
top                        # "st" in the Cpu(s) line
```
In AWS check `CPUCreditBalance` for t-family instances.

**Fix →** Move to a non-burstable instance type, enable unlimited mode, or migrate the VM (stop/start moves it to another host in many clouds).

**Prevent →** Alert on steal time; avoid burstable instances for steady CPU workloads.

</details>

### Q66. A process consumes a lot of CPU in `sy` (system) time instead of `us` (user). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many syscalls (tiny reads/writes, busy polling), lock contention, page faults, or heavy context switching.

**Diagnose →**
```bash
pidstat -u -w -p <PID> 1 5          # %system, cswch/s
sudo strace -c -f -p <PID>          # syscall counts for a few seconds
sudo perf top -p <PID>
```

**Fix →** Depends on findings: bigger buffers, fewer threads, use `epoll` instead of polling loops, fix spin locks. Report the hot syscalls to devs.

**Prevent →** Profile under load in staging; watch `%sys` in dashboards.

</details>

### Q67. `top` shows a process using 400% CPU. Is it broken? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `top` reports per-process CPU relative to a single core; 400% means four cores fully busy, normal for multi-threaded apps.

**Diagnose →**
```bash
nproc
top -H -p <PID>        # per-thread view
```

**Fix →** Nothing, if expected. If not, find hot threads with `top -H` and match TIDs to a thread dump (Java uses hex TIDs in `jstack`).

**Prevent →** Normalise CPU metrics by core count in dashboards.

</details>

### Q68. A command hangs forever, and `strace -p` shows it stuck on `futex(...)` or `read(3, ...)`. How do you dig in? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Waiting on a lock held by another thread (futex), or blocked reading from a pipe, socket or file that never delivers.

**Diagnose →**
```bash
ls -l /proc/<PID>/fd/3              # what is fd 3?
sudo lsof -p <PID> | grep ' 3'
sudo cat /proc/<PID>/stack
gdb -p <PID> -batch -ex 'thread apply all bt'   # if debuginfo is available
```

**Fix →** If it's a socket, check the remote end (DB, API). If it's a lock, find the thread holding it from the backtrace. Add timeouts in the code.

**Prevent →** Use timeouts for all network calls; capture thread dumps automatically on hang detection.

</details>

### Q69. `top` shows load and CPU fine, but an app reports frequent `java.lang.OutOfMemoryError: unable to create native thread`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Thread limits, not heap: `TasksMax=` of the systemd unit (default often 15% of `pid_max` or 4915 on some versions), `ulimit -u`, or `pids.max` in a container.

**Diagnose →**
```bash
ls /proc/<PID>/task | wc -l
systemctl show app -p TasksMax -p TasksCurrent
cat /proc/<PID>/limits | grep processes
```

**Fix →** Raise `TasksMax=` (or `infinity`) and `LimitNPROC=` in the unit if the thread count is legitimate; otherwise fix thread pool sizing.

**Prevent →** Bound thread pools in the app; monitor task counts.

</details>

### Q70. A binary fails with `error while loading shared libraries: libssl.so.1.1: cannot open shared object file: No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The binary was built for another distro/version and the library version isn't installed (e.g. OpenSSL 1.1 vs 3 on newer distros).

**Diagnose →**
```bash
ldd ./app | grep 'not found'
ldconfig -p | grep libssl
```

**Fix →** Install the matching compat package if available, rebuild the binary for the current distro, or run it in a container with the right base image. Avoid symlinking `libssl.so.3` to `.so.1.1`; ABIs differ.

**Prevent →** Build artefacts on the same OS version you deploy to, or ship containers.

</details>

### Q71. An app installed in `/opt/app/lib` can't find its own libraries even though they're there. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `/opt/app/lib` isn't in the dynamic linker's search path, and the binary has no RPATH.

**Diagnose →**
```bash
ldd /opt/app/bin/app
readelf -d /opt/app/bin/app | grep -Ei 'rpath|runpath'
```

**Fix →** Add `/opt/app/lib` to `/etc/ld.so.conf.d/app.conf` and run `sudo ldconfig`, or set `Environment=LD_LIBRARY_PATH=/opt/app/lib` in the service unit.

**Prevent →** Package apps with proper RPATH or a wrapper script.

</details>

### Q72. A process has been running for weeks and you need to know which config file and environment it started with. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Config may have changed on disk since start; you need the process's own view.

**Diagnose →**
```bash
tr '\0' ' ' < /proc/<PID>/cmdline; echo
sudo tr '\0' '\n' < /proc/<PID>/environ
sudo ls -l /proc/<PID>/cwd /proc/<PID>/exe
sudo ls -l /proc/<PID>/fd | grep -E '\.conf|\.yml'
```

**Fix →** Compare with the current files; if they differ, plan a reload/restart.

**Prevent →** Reload services after config changes via config management handlers.

</details>

### Q73. A program crashes with `Segmentation fault (core dumped)` but you can't find the core file. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd-coredump captures cores into the journal storage, or `ulimit -c 0` disables them.

**Diagnose →**
```bash
cat /proc/sys/kernel/core_pattern
coredumpctl list
ulimit -c
```

**Fix →** `coredumpctl info <PID>` for the stack, `coredumpctl debug <PID>` to open in gdb, `coredumpctl dump <PID> -o core.file` to export.

**Prevent →** Keep `systemd-coredump` storage limits (`/etc/systemd/coredump.conf`) sensible, and install debuginfo in staging.

</details>

### Q74. After a deploy, a process uses far more memory than before, with high `VSZ` but normal `RSS`. Should you worry? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** VSZ counts reserved virtual address space (thread stacks, mmap'd files, allocator arenas), not real memory use.

**Diagnose →**
```bash
ps -o pid,vsz,rss,cmd -p <PID>
sudo pmap -x <PID> | tail -5
grep -E 'VmRSS|RssAnon|RssFile' /proc/<PID>/status
```

**Fix →** Nothing, if RSS and PSS are stable. glibc arenas can inflate VSZ; `MALLOC_ARENA_MAX=2` reduces it for many-threaded apps if needed.

**Prevent →** Monitor RSS/PSS and cgroup `memory.current`, not VSZ.

</details>

### Q75. A process is slow and `perf` or `top` shows it constantly page-faulting (`majflt/s` high in `pidstat -r`). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Memory pages are being read back from disk: swap-ins, or its memory-mapped files are being evicted from page cache because of memory pressure.

**Diagnose →**
```bash
pidstat -r -p <PID> 1 5          # majflt/s
grep -E 'VmSwap' /proc/<PID>/status
cat /proc/pressure/memory /proc/pressure/io
```

**Fix →** Give the host or cgroup more memory, reduce other memory consumers, or protect the working set with `MemoryLow=` / `MemoryMin=` for that service.

**Prevent →** Size hosts for the working set plus cache; watch major faults and PSI.

</details>

## Load, CPU and I/O performance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Load average is 35 on an 8-core server, but `top` shows CPU 85% idle. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Linux load counts tasks in uninterruptible sleep (state `D`) as well as runnable ones. Many processes blocked on slow disk or NFS raise load without using CPU.

**Diagnose →**
```bash
vmstat 1 5                         # high "b" column and "wa"
ps -eo pid,stat,wchan:30,cmd | awk '$2 ~ /^D/'
iostat -xz 1 3                     # %util, r_await/w_await
cat /proc/pressure/io
```

**Fix →** Find the blocked resource: a saturated disk (reduce I/O, move workload, faster volume), or a hung NFS mount (restore server, unmount). Load drops as soon as the blocked tasks complete.

**Prevent →** Alert on I/O PSI and disk latency, not just load average.

</details>

### Q77. `top` shows `wa` at 60% and the application is crawling. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CPUs are idle waiting for disk I/O to complete; the disk is the bottleneck.

**Diagnose →**
```bash
iostat -xz 1 5            # which device, %util, await
sudo iotop -oPa           # which process
pidstat -d 1 5
```

**Fix →** Stop or throttle the heavy I/O process (`ionice -c3 -p <PID>`), move it off-peak, or upgrade storage (more IOPS/throughput).

**Prevent →** Separate volumes for logs, data and backups; monitor disk await.

</details>

### Q78. `iostat` shows a disk at `%util 100` but throughput is only 5 MB/s. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Random small I/O on a device with low IOPS (HDD or small cloud volume), or the cloud volume is throttled at its IOPS limit / out of burst credits.

**Diagnose →**
```bash
iostat -xz 1 5           # r/s, w/s, rareq-sz, wareq-sz, aqu-sz
```
In AWS check `BurstBalance` and `VolumeQueueLength` for gp2, or provisioned IOPS for gp3/io2.

**Fix →** Move to a volume with more IOPS (gp3 with provisioned IOPS, NVMe), or reduce random I/O (batch writes, caching).

**Prevent →** Size volumes for IOPS, not just capacity. Note `%util` is misleading on SSD/NVMe arrays that handle requests in parallel; prefer await and queue size.

</details>

### Q79. CPU usage is moderate, but the app's latency doubled after moving to a larger VM with two NUMA nodes. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Memory accessed across NUMA nodes (remote memory is slower), or threads bouncing between sockets.

**Diagnose →**
```bash
lscpu | grep -i numa
numactl --hardware
numastat -p <PID>               # memory per node
perf stat -e cache-misses -p <PID> -- sleep 10
```

**Fix →** Pin the process and its memory to a node: `numactl --cpunodebind=0 --membind=0 app`, or run one instance per node. Check `kernel.numa_balancing` settings for your workload.

**Prevent →** Test on the target instance type; for latency-sensitive services consider instances that fit in one NUMA node.

</details>

### Q80. A server's CPU usage is 100% and `top` shows `kworker` or `ksoftirqd` at the top. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ksoftirqd` high means the kernel is overwhelmed by soft interrupts, typically network packet processing (traffic flood, single queue NIC). `kworker` high can be I/O, a driver bug or ACPI issues.

**Diagnose →**
```bash
mpstat -P ALL 1 3          # %soft per CPU, often one CPU at 100%
cat /proc/softirqs; watch -d -n1 'cat /proc/interrupts | head -30'
sar -n DEV 1 3
sudo perf top -g
```

**Fix →** Spread interrupts across CPUs: enable multiple NIC queues (`ethtool -L eth0 combined 8`), run `irqbalance`, enable RPS/RFS. If it's an attack, filter upstream.

**Prevent →** Use NICs/instances with multi-queue support; monitor softirq per CPU.

</details>

### Q81. One CPU core sits at 100% while the others are idle, and the app can't go faster. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A single-threaded app or a hot lock, or all network interrupts land on CPU0.

**Diagnose →**
```bash
mpstat -P ALL 1 3          # is it %usr or %soft/%irq?
top -H -p <PID>            # one hot thread?
cat /proc/interrupts | grep -E 'eth0|ens'
```

**Fix →** For app bottlenecks: run more processes/workers (e.g. nginx `worker_processes auto`, gunicorn workers). For IRQs: `irqbalance` or multi-queue NICs.

**Prevent →** Load-test to find single-thread limits before production.

</details>

### Q82. `uptime` shows load averages `0.5, 4.0, 12.0`. What does that tell you? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The three numbers are 1, 5 and 15-minute averages; load was high and is now falling. The incident is ending.

**Diagnose →**
```bash
uptime
sar -q -s 08:00:00 -e 09:30:00      # historical run queue from sysstat
journalctl --since "20 min ago" -p warning
```

**Fix →** Find what ran in the past 15 minutes (cron, backup, deploy) using sar history and logs.

**Prevent →** Keep sysstat collecting (`systemctl enable --now sysstat`) for after-the-fact analysis.

</details>

### Q83. A server is slow and `vmstat` shows a high `cs` (context switches) value, 500k per second. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many threads competing for CPUs, lock contention, or very chatty IPC/network patterns.

**Diagnose →**
```bash
vmstat 1 5
pidstat -w 1 5                    # cswch/s (voluntary) and nvcswch/s (involuntary) per process
ps -eLf | wc -l                   # total threads
```

**Fix →** High involuntary switches: too many runnable threads; reduce thread pools to near core count. High voluntary: threads blocking on locks or I/O; investigate contention.

**Prevent →** Size thread pools deliberately; load test.

</details>

### Q84. `sar` isn't available and you need historical CPU and disk data from last night's incident. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** sysstat isn't installed or the collector timer isn't enabled.

**Diagnose →**
```bash
systemctl status sysstat sysstat-collect.timer
ls /var/log/sa/ /var/log/sysstat/ 2>/dev/null
```

**Fix →** For now, use other sources: monitoring (Prometheus/CloudWatch), `journalctl` for that window, `dmesg -T`. Install and enable sysstat: `dnf install sysstat && systemctl enable --now sysstat`.

**Prevent →** Include sysstat and a metrics agent in the base image.

</details>

### Q85. `free` shows plenty of memory, but `dmesg` shows `page allocation failure: order:4` and network drivers fail. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Memory fragmentation: the kernel can't find enough contiguous pages for a high-order allocation, even with free memory available.

**Diagnose →**
```bash
cat /proc/buddyinfo
dmesg -T | grep -i 'page allocation failure'
sysctl vm.min_free_kbytes
```

**Fix →** Trigger compaction: `echo 1 > /proc/sys/vm/compact_memory`. Raise `vm.min_free_kbytes` moderately so the kernel keeps a reserve.

**Prevent →** Tune `min_free_kbytes` on high-throughput network hosts, keep kernels and drivers updated.

</details>

### Q86. Disk latency spikes every 30 seconds and the app stalls briefly. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dirty page writeback bursts: the kernel accumulates writes in memory and flushes large batches, saturating the disk.

**Diagnose →**
```bash
watch -n1 'grep -E "Dirty|Writeback" /proc/meminfo'
sysctl vm.dirty_ratio vm.dirty_background_ratio vm.dirty_expire_centisecs
iostat -xz 1
```

**Fix →** Flush earlier and in smaller batches with byte limits:
```bash
sysctl -w vm.dirty_background_bytes=67108864
sysctl -w vm.dirty_bytes=268435456
```
Persist in `/etc/sysctl.d/`.

**Prevent →** On large-RAM hosts use `*_bytes` rather than percentage settings.

</details>

### Q87. A backup job makes the production database slow every night. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The backup saturates disk I/O or network shared with the database.

**Diagnose →**
```bash
iotop -oPa         # during the backup window
iostat -xz 1
sar -n DEV 1
```

**Fix →** Run the backup with lower priority (`ionice -c3 nice -n19`), rate-limit (`rsync --bwlimit`, `pv -L`), take backups from a replica, or use storage snapshots.

**Prevent →** Schedule backups away from peak; use a replica for backups.

</details>

### Q88. `iostat` shows `await` of 200 ms on an NVMe disk that should do under 1 ms. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Queue saturation from too many concurrent requests, device throttling (cloud limits), thermal throttling, or a failing device.

**Diagnose →**
```bash
iostat -xz 1 5                      # aqu-sz large?
sudo nvme smart-log /dev/nvme0n1     # temperature, media errors
cat /sys/block/nvme0n1/queue/scheduler
dmesg -T | grep -i nvme
```

**Fix →** Reduce concurrency or spread I/O across devices; for cloud volumes raise provisioned IOPS/throughput; replace hardware showing media errors.

**Prevent →** Monitor latency per device and cloud throttling metrics.

</details>

### Q89. The app is slow, but CPU, memory, and disk all look fine. Where do you look next? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Waiting on something external: DNS, a slow downstream API/DB, network retransmits, lock contention, or connection pool exhaustion.

**Diagnose →**
```bash
ss -tnp state established | wc -l
nstat -az | grep -Ei 'retrans|timeout'
strace -f -tt -T -e trace=network -p <PID> 2>&1 | head -50   # slow connect()/recv()
time getent hosts api.internal
```

**Fix →** Fix the slow dependency: DNS caching, increase pool size, timeouts with retries, or fix packet loss.

**Prevent →** Distributed tracing and per-dependency latency metrics.

</details>

### Q90. Running `ls` or any command on the server takes several seconds to start. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Name service lookups timing out (user/group lookups to an unreachable LDAP/SSSD server), a slow shell profile, or a hung NFS mount in `PATH`.

**Diagnose →**
```bash
time /bin/true
strace -f -tt -e trace=connect,openat ls 2>&1 | tail -20
grep -E 'passwd|group' /etc/nsswitch.conf
systemctl status sssd
```

**Fix →** Restore the directory service, or order `files` before `sss`/`ldap` in nsswitch. Remove NFS paths from `PATH`.

**Prevent →** Enable SSSD caching and monitor directory reachability.

</details>

### Q91. CPU frequency seems low and benchmarks are 30% slower than expected on bare metal. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CPU frequency governor set to `powersave`, BIOS power profile, or thermal throttling.

**Diagnose →**
```bash
cpupower frequency-info
grep MHz /proc/cpuinfo | head
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
dmesg -T | grep -i thermal
```

**Fix →** `sudo cpupower frequency-set -g performance` or `tuned-adm profile throughput-performance`. Set the BIOS profile to performance; fix cooling.

**Prevent →** Use `tuned` profiles in your baseline.

</details>

### Q92. `top` itself is slow to refresh and `ps` takes seconds on a host with 50,000 processes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reading `/proc` for that many processes is costly; often a fork bomb or a runaway process spawner.

**Diagnose →**
```bash
ps -eo user= | sort | uniq -c | sort -rn | head
ps -eo comm= | sort | uniq -c | sort -rn | head
```

**Fix →** Stop the spawner (`pkill -STOP -u <user>` first to freeze, then kill), fix the script or service.

**Prevent →** Set `TasksMax=` per service and `nproc` limits per user.

</details>

### Q93. A VM's clock-sensitive app reports timing glitches, and `dmesg` shows `clocksource: Switched to clocksource ...`  or `TSC unstable`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The kernel detected an unreliable clocksource (TSC) and switched to a slower one (hpet/acpi_pm), hurting time-related syscalls and accuracy.

**Diagnose →**
```bash
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
dmesg -T | grep -i clocksource
```

**Fix →** Use the hypervisor-recommended clocksource (`kvm-clock`, `tsc` on Nitro, `hyperv_clocksource_tsc_page`), set via `clocksource=` kernel parameter if needed.

**Prevent →** Follow cloud vendor guidance for clocksource on each instance family.

</details>

### Q94. `top` shows the CPU fully busy in `ni` (nice). Should you be concerned? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Low-priority (niced) processes are using spare CPU; that's what nice is for.

**Diagnose →**
```bash
ps -eo pid,ni,pcpu,cmd --sort=-pcpu | head
```

**Fix →** Nothing, if normal-priority workloads are still responsive. If they suffer, lower the niced job further with cgroups (`CPUWeight=`).

**Prevent →** Use cgroup weights for predictable sharing.

</details>

### Q95. After enabling auditd with many rules, the server's CPU usage and latency rose noticeably. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Broad syscall rules (e.g. `-S open` or `execve` for all users) generate huge event volumes, or the audit backlog fills and the kernel blocks.

**Diagnose →**
```bash
sudo auditctl -s                 # backlog, lost
sudo auditctl -l
sudo aureport --summary
```

**Fix →** Narrow rules with filters (`-F auid>=1000 -F auid!=unset`, specific paths), raise `-b` backlog size, and set `--backlog_wait_time` appropriately.

**Prevent →** Benchmark audit rules in staging; follow a curated rule set.

</details>

### Q96. A server's disk I/O is high, yet `iotop` shows no process doing I/O. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kernel threads doing writeback (`kworker`/`flush`), journal commits (`jbd2`), swap, RAID resync, or I/O from another VM on shared storage.

**Diagnose →**
```bash
sudo iotop -oPa -k          # include kernel threads
cat /proc/mdstat
vmstat 1 5                  # si/so for swap
sudo biosnoop-bpfcc 2>/dev/null | head    # per-I/O process attribution (bcc tools)
```

**Fix →** Depends on source: limit RAID resync speed, address swapping, tune dirty page writeback.

**Prevent →** Have eBPF tools (bcc/bpftrace) installed for attribution.

</details>

### Q97. Response times degrade as a host's uptime grows; restarting the app fixes it for a few days. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A resource leak inside the app: memory, file descriptors, threads, connections or a growing in-memory cache.

**Diagnose →**
```bash
ps -o pid,etime,rss,nlwp -p <PID>
ls /proc/<PID>/fd | wc -l
```
Graph RSS, fd count and thread count over days.

**Fix →** Hand the trend data to the developers along with heap and thread dumps. Meanwhile, use scheduled rolling restarts as a stopgap.

**Prevent →** Add per-process resource metrics to your dashboards.

</details>

### Q98. A process is throttled even though the host has idle CPU, and `cpu.stat` shows a large `nr_throttled`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The process's cgroup CPU quota (`CPUQuota=` or a container CPU limit) is too low for its bursty usage.

**Diagnose →**
```bash
systemctl show app -p CPUQuotaPerSecUSec
cat /sys/fs/cgroup/system.slice/app.service/cpu.max
cat /sys/fs/cgroup/system.slice/app.service/cpu.stat    # nr_throttled, throttled_usec
```

**Fix →** Raise the quota, or use `CPUWeight=` (a proportional share) instead of a hard cap for latency-sensitive services.

**Prevent →** Monitor throttling; size CPU limits against peak usage, not average.

</details>

### Q99. `vmstat` shows `r` (run queue) at 40 on a 4-core box and the CPU is 100% `us`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Genuine CPU saturation: far more runnable work than cores.

**Diagnose →**
```bash
vmstat 1 5
top -c -o %CPU
pidstat -u 1 5
```

**Fix →** Find the top consumers, kill or limit runaway work, scale out or move to more cores; optimise hot code with profiling.

**Prevent →** Capacity alerts on run queue length relative to core count.

</details>

### Q100. Network throughput tops out at 1 Gbps on a 10 Gbps instance. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Single TCP stream limited by window size or latency, driver/feature issues (no ENA/virtio enhancements), a 1 Gbps link negotiated, or instance network limits.

**Diagnose →**
```bash
ethtool eth0 | grep -i speed
ethtool -i eth0                 # driver
iperf3 -c peer -P 8             # parallel streams improve it?
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

**Fix →** Use parallel streams, raise TCP buffer maximums for high-latency links, update drivers, and check instance type network specs.

**Prevent →** Benchmark the network when choosing instance types.

</details>

## systemd services and journald

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `systemctl status myapp` shows `Main process exited, code=exited, status=203/EXEC`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd couldn't execute `ExecStart`: wrong path, file not executable, missing shebang or interpreter, CRLF line endings, `noexec` mount, or SELinux blocking execution (e.g. a binary in a home directory).

**Diagnose →**
```bash
systemctl cat myapp | grep ExecStart
ls -l /opt/myapp/bin/start.sh
head -1 /opt/myapp/bin/start.sh | cat -A
findmnt -no OPTIONS --target /opt/myapp/bin/start.sh
sudo ausearch -m AVC -ts recent | grep start.sh
```

**Fix →** Use an absolute path, `chmod +x`, add a correct shebang, convert line endings, move it off `noexec` mounts, or `restorecon -v` the file. Then `systemctl daemon-reload && systemctl restart myapp`.

**Prevent →** Test units in CI or staging; keep service binaries under `/opt` or `/usr/local/bin`.

</details>

### Q102. A service fails with `status=217/USER`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user in `User=` doesn't exist on this host (typo, or not created yet by provisioning).

**Diagnose →**
```bash
systemctl cat myapp | grep -E 'User|Group'
getent passwd appuser
```

**Fix →** Create the user (`useradd --system appuser`) or correct the name, then restart. Or use `DynamicUser=yes` if the service needs no fixed identity.

**Prevent →** Create service users in the same automation step that installs the unit, ordered before the start.

</details>

### Q103. A service fails with `status=200/CHDIR`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `WorkingDirectory=` doesn't exist or the service user can't enter it.

**Diagnose →**
```bash
systemctl cat myapp | grep WorkingDirectory
sudo -u appuser test -x /opt/myapp && echo ok
namei -l /opt/myapp
```

**Fix →** Create the directory with correct ownership, fix parent permissions, or prefix with `-` (`WorkingDirectory=-/opt/myapp`) to make it optional.

**Prevent →** Use `StateDirectory=` / `RuntimeDirectory=` so systemd creates directories with the right owner.

</details>

### Q104. A unit fails with `status=226/NAMESPACE`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd couldn't set up the mount namespace for sandboxing options: a path in `ReadWritePaths=`, `ReadOnlyPaths=`, `BindPaths=` or `InaccessiblePaths=` doesn't exist, or the kernel/container doesn't allow namespaces.

**Diagnose →**
```bash
journalctl -u myapp -b | grep -i namespace
systemctl cat myapp | grep -E 'Paths=|Protect|Private'
```

**Fix →** Create the missing paths or prefix them with `-` to ignore missing ones (`ReadWritePaths=-/var/lib/myapp`). Inside unprivileged containers, drop the sandboxing options that need namespaces.

**Prevent →** Use `StateDirectory=`/`LogsDirectory=` so paths exist before the namespace is built.

</details>

### Q105. You edited a unit file, restarted the service, but the old behaviour persists. `systemctl status` shows `Warning: The unit file, source configuration file or drop-ins of myapp.service changed on disk`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd hasn't reloaded unit definitions.

**Diagnose →**
```bash
systemctl status myapp | grep -i warning
systemctl cat myapp
```

**Fix →** `sudo systemctl daemon-reload && sudo systemctl restart myapp`.

**Prevent →** Use `systemctl edit`, which reloads automatically, and add a daemon-reload handler in Ansible (`systemd: daemon_reload: true`).

</details>

### Q106. You edited `/usr/lib/systemd/system/nginx.service` directly and after a package update your change vanished. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Vendor unit files are owned by the package and get overwritten.

**Diagnose →**
```bash
systemctl cat nginx            # shows which files are used
systemd-delta --type=extended,overridden
```

**Fix →** Put the change in a drop-in: `systemctl edit nginx` (creates `/etc/systemd/system/nginx.service.d/override.conf`).

**Prevent →** Never edit `/usr/lib/systemd/system/`; `systemd-delta` in audits finds local overrides.

</details>

### Q107. `systemctl start myapp` returns immediately with success, but the app isn't running a second later and `status` shows `inactive (dead)` with `code=exited, status=0/SUCCESS`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ExecStart` runs a script that launches the app in the background and exits (`&` or `nohup`), so systemd sees the main process end and cleans up the cgroup.

**Diagnose →**
```bash
systemctl cat myapp
journalctl -u myapp -b
```

**Fix →** Make the script `exec` the app in the foreground (`exec /opt/myapp/bin/server`), and keep `Type=simple` or `Type=exec`.

**Prevent →** Avoid wrapper scripts that daemonise; let systemd supervise directly.

</details>

### Q108. A service stays in `activating (start)` for 90 seconds and then fails with `Failed with result 'timeout'`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Type=notify` but the app never sends `READY=1`, `Type=forking` but the parent never exits, or a `ExecStartPre` that hangs. Sometimes the app really needs longer to start.

**Diagnose →**
```bash
systemctl show myapp -p Type -p TimeoutStartUSec -p NotifyAccess
journalctl -u myapp -b
```

**Fix →** Use the right `Type` (`simple`/`exec` if the app doesn't support sd_notify), fix the hanging pre-step, or raise `TimeoutStartSec=` for slow starts.

**Prevent →** Match `Type=` to how the app actually behaves; document required start time.

</details>

### Q109. A service starts fine manually but fails at boot because the network isn't up: `bind: Cannot assign requested address`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It binds to a specific IP that isn't configured yet when the service starts.

**Diagnose →**
```bash
journalctl -b -u myapp
systemctl list-dependencies myapp --after | grep network
systemctl is-enabled NetworkManager-wait-online systemd-networkd-wait-online
```

**Fix →** Add `After=network-online.target` and `Wants=network-online.target`, and make sure the matching wait-online service is enabled. Alternatively, bind to `0.0.0.0`, or allow binding to absent addresses with `net.ipv4.ip_nonlocal_bind=1` (useful for VIPs).

**Prevent →** Prefer `network-online.target` for services that bind to specific addresses.

</details>

### Q110. `systemctl enable myapp` prints `The unit files have no installation config (WantedBy=, RequiredBy=, Also=, Alias= settings in the [Install] section...)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The unit has no `[Install]` section, so there's nothing to enable.

**Diagnose →**
```bash
systemctl cat myapp | grep -A3 '\[Install\]'
```

**Fix →** Add:
```ini
[Install]
WantedBy=multi-user.target
```
Then `daemon-reload` and `systemctl enable myapp`.

**Prevent →** Use a standard unit template for new services.

</details>

### Q111. A service fails with `status=209/STDOUT` after you added `StandardOutput=append:/var/log/myapp/out.log`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** systemd couldn't open the log file: the directory doesn't exist, or sandboxing/SELinux prevents it.

**Diagnose →**
```bash
ls -ld /var/log/myapp
journalctl -u myapp -b | head
sudo ausearch -m AVC -ts recent
```

**Fix →** Use `LogsDirectory=myapp` so systemd creates `/var/log/myapp` with the right owner, or create it manually. Simplest: drop the file and let output go to the journal.

**Prevent →** Prefer journald plus log shipping over custom log files.

</details>

### Q112. Logs from a service appear in `journalctl` with long delays or arrive in bursts. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app's stdout is block-buffered when not attached to a terminal (Python, C stdio), so output is flushed only when the buffer fills.

**Diagnose →**
```bash
journalctl -u myapp -f        # compare timestamps with app events
```

**Fix →** Disable buffering: for Python `Environment=PYTHONUNBUFFERED=1`, or `stdbuf -oL` in `ExecStart`, or flush in the app.

**Prevent →** Standardise logging libraries that flush per line.

</details>

### Q113. journald shows `Suppressed 12345 messages from myapp.service`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** journald rate limiting kicked in because the service logged too fast (default 10,000 messages per 30 s per service in recent versions).

**Diagnose →**
```bash
journalctl -u systemd-journald -b | grep Suppressed
grep -E 'RateLimit' /etc/systemd/journald.conf
```

**Fix →** Reduce log volume in the app first (debug logging left on?). If high volume is legitimate, set per-unit limits: `LogRateLimitIntervalSec=30s` and `LogRateLimitBurst=50000` in the service unit.

**Prevent →** Keep log levels at INFO in production; alert on suppression messages.

</details>

### Q114. `journalctl` is very slow and returns `Journal file ... is truncated, ignoring file` warnings. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Corrupted journal files (unclean shutdown, disk full while writing), or a huge number of journal files.

**Diagnose →**
```bash
journalctl --verify
journalctl --disk-usage
ls /var/log/journal/*/ | wc -l
```

**Fix →** Rotate and clean: `journalctl --rotate && journalctl --vacuum-time=7d`. Move corrupted files out of the directory (journald keeps working). Restart `systemd-journald`.

**Prevent →** Keep disk space headroom and size limits (`SystemMaxUse=`, `SystemMaxFileSize=`).

</details>

### Q115. A service using a template unit fails: `Failed to start worker@.service: Unit name worker@.service is missing the instance name`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You tried to start the template itself instead of an instance.

**Diagnose →**
```bash
systemctl list-units 'worker@*'
```

**Fix →** Start an instance: `systemctl start worker@emails`. To enable instances at boot, `systemctl enable worker@emails`.

**Prevent →** Document instance names in the runbook.

</details>

### Q116. A service is masked and can't be started: `Failed to start foo.service: Unit foo.service is masked.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Someone ran `systemctl mask` (or a package/hardening script did), linking the unit to `/dev/null`.

**Diagnose →**
```bash
systemctl is-enabled foo
ls -l /etc/systemd/system/foo.service
```

**Fix →** If it should run: `systemctl unmask foo && systemctl enable --now foo`. Check with the team first; masking is usually deliberate.

**Prevent →** Track masks in config management with a reason.

</details>

### Q117. A oneshot unit shows `active (exited)` and running `systemctl start` again does nothing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `RemainAfterExit=yes` keeps the unit "active", so starting it again is a no-op.

**Diagnose →**
```bash
systemctl show myjob -p Type -p RemainAfterExit
```

**Fix →** Use `systemctl restart myjob` to run it again, or remove `RemainAfterExit` if the unit is meant to run repeatedly (e.g. from a timer).

**Prevent →** Use `RemainAfterExit=yes` only for "set up once" units.

</details>

### Q118. A service dies when you log out of SSH, even though you started it with `systemctl --user start`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user's systemd manager stops when the last session ends because lingering isn't enabled.

**Diagnose →**
```bash
loginctl show-user $USER -p Linger
```

**Fix →** `sudo loginctl enable-linger <user>`.

**Prevent →** Enable linger for any account running user services or rootless containers.

</details>

### Q119. Processes started from an SSH session (e.g. `tmux`) get killed when the user logs out on a newer distro. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `KillUserProcesses=yes` in `/etc/systemd/logind.conf` kills all processes in the session scope at logout.

**Diagnose →**
```bash
grep -i KillUserProcesses /etc/systemd/logind.conf
loginctl show-session $XDG_SESSION_ID
```

**Fix →** Set `KillUserProcesses=no` (or add users to `KillExcludeUsers=`), restart `systemd-logind` carefully, or run long tasks with `systemd-run --user --scope` plus linger.

**Prevent →** Run long-lived work as services, not inside interactive sessions.

</details>

### Q120. A service's `ExecStartPre` fails and the main service never starts: `Control process exited, code=exited, status=1/FAILURE`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A pre-start step (config test, migration, directory creation) returned non-zero.

**Diagnose →**
```bash
journalctl -u myapp -b --no-pager | tail -30
systemctl cat myapp | grep ExecStartPre
```
Run the pre-step by hand as the service user.

**Fix →** Fix the config error it reports (e.g. `nginx -t` output). If a step is optional, prefix with `-` (`ExecStartPre=-/usr/bin/cleanup`) so failure is ignored.

**Prevent →** Validate config in CI before deploy.

</details>

### Q121. A unit stopped working after you added `DynamicUser=yes`; the app can't find its data in `/var/lib/myapp`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `DynamicUser=yes`, the UID changes and `/var/lib/myapp` is owned by the old user; dynamic users can only write to `StateDirectory=`, which lives under `/var/lib/private/`.

**Diagnose →**
```bash
systemctl show myapp -p DynamicUser -p StateDirectory
ls -ld /var/lib/myapp /var/lib/private/myapp
```

**Fix →** Set `StateDirectory=myapp`; systemd migrates ownership and symlinks `/var/lib/myapp` to `/var/lib/private/myapp`. Move existing data into it if needed.

**Prevent →** Plan state directories when adopting `DynamicUser`.

</details>

### Q122. `systemctl` commands hang or fail with `Failed to connect to bus: No such file or directory` inside a container or via `su`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No systemd running as PID 1 (most containers), or for `--user`, no `XDG_RUNTIME_DIR`/user bus because the session was started with `su` instead of a full login.

**Diagnose →**
```bash
ps -p 1 -o comm=
echo $XDG_RUNTIME_DIR $DBUS_SESSION_BUS_ADDRESS
```

**Fix →** In containers, run the process directly (no systemd). For user units, log in properly (`machinectl shell user@` or SSH) or set `XDG_RUNTIME_DIR=/run/user/$(id -u)`.

**Prevent →** Use `machinectl shell` or `sudo -iu user` with linger for user-unit management.

</details>

### Q123. A service restarts every few seconds; the journal shows `Main process exited, code=killed, status=9/KILL` but nobody runs kill. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cgroup memory limit (`MemoryMax=`) triggers the OOM killer, systemd-oomd kills it under pressure, or a watchdog (`WatchdogSec=`) timeout expires.

**Diagnose →**
```bash
journalctl -u myapp -b | grep -Ei 'oom|watchdog'
journalctl -u systemd-oomd -b
systemctl show myapp -p MemoryMax -p WatchdogUSec -p NRestarts
dmesg -T | grep -i oom
```

**Fix →** Raise or fix the memory limit; if it's the watchdog, make the app send `WATCHDOG=1` in time or increase `WatchdogSec=`.

**Prevent →** Alert on `NRestarts` growth and OOM events per unit.

</details>

### Q124. After a reboot, a service that depends on an NFS mount starts before the mount and writes to the local disk. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No dependency between the service and the mount; the service starts as soon as networking is up.

**Diagnose →**
```bash
systemctl list-dependencies myapp | grep mount
systemctl status mnt-share.mount
```

**Fix →** Add `RequiresMountsFor=/mnt/share` to the `[Unit]` section, which adds `Requires=` and `After=` on the mount unit.

**Prevent →** Always declare mount dependencies for services using network storage.

</details>

### Q125. systemd fails a unit with `Failed to load environment files: No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `EnvironmentFile=` points to a file that doesn't exist.

**Diagnose →**
```bash
systemctl cat myapp | grep EnvironmentFile
ls -l /etc/myapp/myapp.env
```

**Fix →** Create the file or make it optional with a leading dash: `EnvironmentFile=-/etc/myapp/myapp.env`.

**Prevent →** Deploy the env file before the unit; validate in your deployment playbook.

</details>

## Networking, DNS and firewalls

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `curl http://10.0.1.20:8080` returns `curl: (7) Failed to connect to 10.0.1.20 port 8080: Connection refused`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host is reachable but nothing listens on that port/IP (service down, bound to `127.0.0.1` only), or a firewall rule actively rejects with a TCP RST/ICMP.

**Diagnose →**
```bash
# on the server
sudo ss -tlnp | grep 8080              # listening? on 0.0.0.0 or 127.0.0.1?
systemctl status myapp
sudo nft list ruleset | grep -i reject
```

**Fix →** Start the service, or change its bind address to `0.0.0.0`/the right interface; remove the reject rule if it's wrong.

**Prevent →** Health checks that test from outside the host, not just `localhost`.

</details>

### Q127. The same request instead hangs and ends with `curl: (28) Failed to connect to 10.0.1.20 port 8080 after 130000 ms: Connection timed out`. How is this different from "refused"? 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Packets are silently dropped: a firewall/security group with DROP, wrong routing, host down, or the SYN-ACK can't get back (asymmetric routing).

**Diagnose →**
```bash
ping -c3 10.0.1.20
traceroute -T -p 8080 10.0.1.20
# on the server: are SYNs arriving?
sudo tcpdump -ni any 'tcp port 8080 and tcp[tcpflags] & tcp-syn != 0'
```

**Fix →** If no SYNs arrive, fix the security group, NACL or network firewall upstream. If SYNs arrive but no reply leaves, check the host firewall and return routes.

**Prevent →** Remember the rule: refused means you reached the host; timeout means something dropped the packets.

</details>

### Q128. `ping google.com` fails with `ping: google.com: Temporary failure in name resolution`, but `ping 8.8.8.8` works. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS is broken: no or wrong nameservers, resolver service down, or port 53 blocked.

**Diagnose →**
```bash
cat /etc/resolv.conf
resolvectl status 2>/dev/null
dig google.com @8.8.8.8          # does a public resolver work?
dig google.com                   # using system config
```

**Fix →** Point to working resolvers via NetworkManager/netplan (`nmcli con mod eth0 ipv4.dns "10.0.0.2"`), restart `systemd-resolved` if it's the one failing, or open UDP/TCP 53 egress.

**Prevent →** Two resolvers in config; monitor DNS resolution from hosts.

</details>

### Q129. `dig api.internal` works, but the application and `curl` fail with `Could not resolve host: api.internal`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `dig` queries DNS directly and skips NSS; apps use `getaddrinfo()` which follows `/etc/nsswitch.conf`, `/etc/hosts` and search domains. A bad `hosts:` line, or a `.local`/mDNS rule, or the app running in a different network namespace, can differ.

**Diagnose →**
```bash
getent hosts api.internal          # uses the same path as apps
grep '^hosts' /etc/nsswitch.conf
cat /etc/resolv.conf               # search, options ndots
```

**Fix →** Fix nsswitch (`hosts: files dns`), add the correct `search` domain, or fix the container's DNS config.

**Prevent →** Test with `getent hosts`, not `dig`, when debugging application resolution.

</details>

### Q130. `/etc/resolv.conf` changes you made disappear after a reboot or a few minutes. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file is managed by NetworkManager, systemd-resolved (symlink to `/run/systemd/resolve/stub-resolv.conf`), netplan or DHCP client.

**Diagnose →**
```bash
ls -l /etc/resolv.conf
head -5 /etc/resolv.conf          # "Generated by NetworkManager" etc.
```

**Fix →** Configure DNS in the owning tool: `nmcli con mod`, netplan `nameservers:`, or `/etc/systemd/resolved.conf` `DNS=`. Then reapply.

**Prevent →** Never hand-edit a generated `resolv.conf`.

</details>

### Q131. DNS lookups take exactly 5 seconds before succeeding. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The first nameserver in `resolv.conf` is dead and the resolver times out (default 5 s) before trying the next. Also common: parallel A/AAAA queries dropped by a firewall/conntrack race.

**Diagnose →**
```bash
cat /etc/resolv.conf
for ns in $(awk '/^nameserver/{print $2}' /etc/resolv.conf); do dig +time=2 +tries=1 example.com @$ns | grep -E 'status|Query time'; done
```

**Fix →** Remove the dead nameserver; add `options timeout:1 attempts:2 rotate`, or `single-request-reopen` for the A/AAAA issue.

**Prevent →** Monitor each resolver; run a local caching resolver (systemd-resolved, dnsmasq, unbound).

</details>

### Q132. Short names like `db01` don't resolve, but `db01.corp.example.com` does. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or wrong `search` domain.

**Diagnose →**
```bash
grep -E '^(search|domain)' /etc/resolv.conf
resolvectl domain
```

**Fix →** Set the search domain in the network config: `nmcli con mod eth0 ipv4.dns-search corp.example.com && nmcli con up eth0`.

**Prevent →** Prefer FQDNs in application configs anyway.

</details>

### Q133. A host can reach servers in its own subnet but nothing beyond: `connect: Network is unreachable`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No default route (missing gateway, DHCP failure, misconfigured static config).

**Diagnose →**
```bash
ip route
ip -4 addr
nmcli dev show eth0 | grep -i gateway
```

**Fix →** Add it now: `sudo ip route add default via 10.0.1.1 dev eth0`, then persist in NetworkManager/netplan (`ipv4.gateway`).

**Prevent →** Validate network config with `netplan try` or a test after changes.

</details>

### Q134. An application gets `connect: No route to host` to a server that is up and responds to ping from elsewhere. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target's firewall rejects with `icmp-host-prohibited` (firewalld/iptables default REJECT), or ARP fails on the local segment.

**Diagnose →**
```bash
ip neigh show 10.0.1.20           # FAILED/INCOMPLETE = ARP problem
sudo tcpdump -ni eth0 'icmp or arp'
# on target
sudo firewall-cmd --list-all
```

**Fix →** Open the port on the target (`firewall-cmd --add-port=8080/tcp --permanent && firewall-cmd --reload`), or fix the L2 issue (VLAN, IP conflict).

**Prevent →** Remember: firewalld's reject message looks like a routing error.

</details>

### Q135. `ss -tlnp` shows the app listening on `127.0.0.1:5432`, and remote clients can't connect. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service is bound to loopback only (PostgreSQL's default `listen_addresses = 'localhost'`).

**Diagnose →**
```bash
sudo ss -tlnp | grep 5432
grep listen_addresses /var/lib/pgsql/data/postgresql.conf
```

**Fix →** Set `listen_addresses = '10.0.1.20'` (or `'*'` with firewall restrictions), allow clients in `pg_hba.conf`, restart PostgreSQL, open the firewall.

**Prevent →** Bind to specific private IPs, not `0.0.0.0`, and firewall the port.

</details>

### Q136. After adding firewalld rules, they work until the next reboot and then disappear. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rules were added at runtime without `--permanent`.

**Diagnose →**
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --list-all
```

**Fix →** `sudo firewall-cmd --runtime-to-permanent`, or add rules with `--permanent` then `--reload`.

**Prevent →** Manage firewall rules with config management (Ansible `firewalld` module with `permanent: true, immediate: true`).

</details>

### Q137. `dmesg` is flooded with `nf_conntrack: table full, dropping packet` and new connections randomly fail. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The connection tracking table is full: high connection rates on a NAT gateway, load balancer or busy host with stateful firewall rules.

**Diagnose →**
```bash
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
sudo conntrack -S | head
sudo conntrack -L 2>/dev/null | awk '{print $1, $4}' | sort | uniq -c | sort -rn | head
```

**Fix →** Raise the limit: `sysctl -w net.netfilter.nf_conntrack_max=1048576` (and `nf_conntrack_buckets` via module options), shorten timeouts (`nf_conntrack_tcp_timeout_time_wait`), or exempt high-volume traffic with `notrack` rules.

**Prevent →** Alert at 80% of `nf_conntrack_max`; size for peak.

</details>

### Q138. A server shows packet loss and the interface counters keep rising: `ip -s link` shows `RX dropped` increasing. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The NIC ring buffer or kernel backlog overflows under bursts, CPU can't keep up with softirqs, or packets for unknown protocols/VLANs are discarded.

**Diagnose →**
```bash
ip -s link show eth0
ethtool -S eth0 | grep -Ei 'drop|miss|fifo|no_buff'
ethtool -g eth0                    # ring sizes
cat /proc/net/softnet_stat         # 2nd column = backlog drops
```

**Fix →** Increase ring buffers (`ethtool -G eth0 rx 4096`), raise `net.core.netdev_max_backlog`, spread IRQs/queues across CPUs.

**Prevent →** Graph NIC drop counters; tune before peak traffic.

</details>

### Q139. Two servers have the same IP and connections randomly reset; logs show `IPv4: martian source` or ARP flapping. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** IP address conflict on the segment.

**Diagnose →**
```bash
sudo arping -D -I eth0 -c 3 10.0.1.25       # duplicate detection; replies = conflict
ip neigh show 10.0.1.25
journalctl -k | grep -i 'duplicate\|martian'
```

**Fix →** Find the other MAC owner (switch tables, DHCP leases, cloud console) and re-address one host.

**Prevent →** IPAM/DHCP reservations instead of hand-assigned addresses.

</details>

### Q140. Traffic arrives on the second NIC of a multi-homed server but replies leave through the first NIC and get dropped. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Asymmetric routing: the single default route sends replies out the wrong interface; `rp_filter` or upstream firewalls drop them.

**Diagnose →**
```bash
ip route; ip rule
sudo tcpdump -ni eth1 port 443 &
sudo tcpdump -ni eth0 port 443
sysctl net.ipv4.conf.all.rp_filter
```

**Fix →** Policy routing so replies from eth1's IP use eth1's gateway:
```bash
ip route add default via 10.0.2.1 dev eth1 table 100
ip rule add from 10.0.2.15 table 100
```
Persist via NetworkManager (`ipv4.routing-rules`) or netplan `routing-policy`. Set `rp_filter=2` (loose) if needed.

**Prevent →** Document every multi-homed host's routing design.

</details>

### Q141. `curl https://api.example.com` fails with `curl: (60) SSL certificate problem: unable to get local issuer certificate`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server doesn't send the intermediate certificate, the host's CA bundle is outdated, or a corporate TLS-inspecting proxy uses an internal CA not trusted by the host.

**Diagnose →**
```bash
openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts </dev/null | grep -E 's:|i:'
rpm -q ca-certificates || dpkg -l ca-certificates
```

**Fix →** Fix the server's chain (send full chain). For an internal CA: copy it to `/etc/pki/ca-trust/source/anchors/` and `update-ca-trust` (RHEL) or `/usr/local/share/ca-certificates/` and `update-ca-certificates` (Debian). Never use `curl -k` as the fix.

**Prevent →** Keep `ca-certificates` updated; check chains in monitoring.

</details>

### Q142. TLS connections fail with `certificate is not yet valid` or `certificate has expired` while the certificate is fine on other hosts. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host's clock is wrong.

**Diagnose →**
```bash
date -u
timedatectl
chronyc tracking
```

**Fix →** Fix time sync (see chrony), then retry. A big jump may need `chronyc makestep`.

**Prevent →** Monitor clock offset; it breaks TLS, Kerberos and token validation.

</details>

### Q143. A newly added iptables rule to allow port 8080 has no effect. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rule order: it was appended (`-A`) after a catch-all REJECT/DROP; or firewalld/nftables manage the real ruleset and your iptables rule went to a different table (iptables-nft vs legacy).

**Diagnose →**
```bash
sudo iptables -L INPUT -n --line-numbers -v
sudo nft list ruleset | less
systemctl is-active firewalld nftables ufw
```

**Fix →** Insert before the reject: `iptables -I INPUT 5 -p tcp --dport 8080 -j ACCEPT`, or better, use the active manager (`firewall-cmd`, `ufw`, nft config).

**Prevent →** Use one firewall manager per host; never mix them.

</details>

### Q144. Docker containers lose network access after you restart firewalld or flush iptables. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker inserts its own NAT and FORWARD rules; flushing or reloading the firewall wipes them.

**Diagnose →**
```bash
sudo iptables -t nat -L DOCKER -n
sudo nft list ruleset | grep -i docker
```

**Fix →** `sudo systemctl restart docker` to recreate its rules. Put custom rules in the `DOCKER-USER` chain so Docker leaves them alone.

**Prevent →** Avoid `iptables -F` on container hosts; manage rules in `DOCKER-USER`.

</details>

### Q145. `ssh` and `curl` to a remote host work, but `git clone` over HTTPS hangs on large repos only. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A proxy or middlebox timing out long transfers, a broken HTTP proxy setting in git, or a stateful firewall killing long-lived connections.

**Diagnose →**
```bash
git config --show-origin --get-regexp 'http.*proxy'
env | grep -i proxy
GIT_TRACE=1 GIT_CURL_VERBOSE=1 git clone https://... 2>&1 | tail -40
```

**Fix →** Correct or remove proxy settings, set `NO_PROXY` for internal hosts, use shallow clones (`--depth 1`) or SSH transport, and raise idle timeouts on the middlebox.

**Prevent →** Central, documented proxy config (`/etc/environment`, systemd `Environment=`).

</details>

### Q146. A service can't connect to an external API only from inside a systemd unit, but works from your shell. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Proxy variables exist in your shell profile but not in the unit; or `IPAddressDeny=`/`RestrictAddressFamilies=`/`PrivateNetwork=` in the unit blocks it.

**Diagnose →**
```bash
env | grep -i proxy
systemctl show myapp -p Environment -p IPAddressDeny -p PrivateNetwork -p RestrictAddressFamilies
```

**Fix →** Add `Environment=HTTPS_PROXY=http://proxy:3128 NO_PROXY=.internal` to the unit, or relax the sandbox option.

**Prevent →** Define required environment in units explicitly; never rely on login profiles.

</details>

### Q147. After switching an interface to a static IP with `nmcli`, the IP is set but the old DHCP address still appears too. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two connection profiles are active for the device, or `ipv4.method` is still `auto` with the static address added as extra.

**Diagnose →**
```bash
nmcli con show
nmcli -f ipv4.method,ipv4.addresses con show "System eth0"
ip addr show eth0
```

**Fix →** `nmcli con mod "System eth0" ipv4.method manual` and delete the duplicate profile (`nmcli con delete <name>`), then `nmcli con up "System eth0"`.

**Prevent →** One profile per interface, managed by automation.

</details>

### Q148. Clients get `ERR_CONNECTION_RESET` intermittently to a web server; `nstat` shows `TcpExtListenOverflows` rising. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The listen (accept) queue overflows during bursts because the app accepts too slowly or the backlog is too small.

**Diagnose →**
```bash
nstat -az TcpExtListenOverflows TcpExtListenDrops
ss -ltn 'sport = :443'          # Recv-Q vs Send-Q (backlog)
sysctl net.core.somaxconn
```

**Fix →** Raise `net.core.somaxconn` and the app's listen backlog (nginx `listen 443 backlog=4096`), add workers so accept keeps up.

**Prevent →** Graph listen overflows; load test with realistic bursts.

</details>

### Q149. A host can resolve and ping IPv4 but apps hang for a long time before connecting; `getent ahosts` shows IPv6 addresses first. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AAAA records are returned and the client tries IPv6 first, but there is no working IPv6 route, so it waits for timeout before falling back.

**Diagnose →**
```bash
getent ahosts api.example.com
ip -6 route
curl -6 -v https://api.example.com --max-time 5
```

**Fix →** Fix IPv6 connectivity, or prefer IPv4 in `/etc/gai.conf` (`precedence ::ffff:0:0/96 100`), or disable IPv6 on that interface if it's not used.

**Prevent →** Either support IPv6 properly or disable it consistently.

</details>

### Q150. `systemd-resolved` is running but lookups fail with `resolve call failed: All attempts to contact name servers or networks failed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** resolved has no upstream DNS servers configured for any link, or DNSSEC/DoT settings fail with the upstream.

**Diagnose →**
```bash
resolvectl status
resolvectl query example.com
grep -Ev '^#|^$' /etc/systemd/resolved.conf
```

**Fix →** Set upstream servers per link (via NetworkManager/netplan) or globally (`DNS=10.0.0.2` in `resolved.conf`), set `DNSSEC=no` or `allow-downgrade` if upstream doesn't support it, and `systemctl restart systemd-resolved`. Flush with `resolvectl flush-caches`.

**Prevent →** Include DNS checks in host monitoring.

</details>

## SSH, sudo and authentication

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. `ssh deploy@server01` fails with `deploy@server01: Permission denied (publickey).` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client offers the wrong key, the public key isn't in the right `authorized_keys`, or permissions on `~`, `~/.ssh` or `authorized_keys` are too open so sshd ignores them.

**Diagnose →**
```bash
ssh -v deploy@server01 2>&1 | grep -E 'Offering|Authentications|identity file'
# on the server (console or another account)
sudo ls -ld /home/deploy /home/deploy/.ssh /home/deploy/.ssh/authorized_keys
sudo journalctl -u sshd -n 30 | grep -i deploy
```

**Fix →**
```bash
sudo chmod go-w /home/deploy
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
```
Use the correct key: `ssh -i ~/.ssh/deploy_ed25519 deploy@server01`.

**Prevent →** Create keys with automation (Ansible `authorized_key`) which sets modes correctly.

</details>

### Q152. Permissions on `~/.ssh` are correct, but you still get `Permission denied (publickey)` and the server log says `Authentication refused: bad ownership or modes for directory /home/deploy`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The home directory itself is group- or world-writable (`StrictModes yes` checks every path component).

**Diagnose →**
```bash
sudo journalctl -u sshd | grep -i 'bad ownership'
namei -l /home/deploy/.ssh/authorized_keys
```

**Fix →** `sudo chmod 755 /home/deploy` (or `750`), owned by the user.

**Prevent →** Don't `chmod 777` home directories to "fix" sharing; use groups.

</details>

### Q153. Key auth fails only on RHEL with SELinux enforcing; the server logs show nothing unusual about modes. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `authorized_keys` has the wrong SELinux context (copied or created in an unusual way, or the home directory is in a non-standard path like `/data/home`).

**Diagnose →**
```bash
ls -Z /home/deploy/.ssh/authorized_keys      # expect ssh_home_t
sudo ausearch -m AVC -c sshd -ts recent
```

**Fix →** `sudo restorecon -Rv /home/deploy/.ssh`. For a custom home root: `semanage fcontext -a -e /home /data/home && restorecon -Rv /data/home`.

**Prevent →** Use standard home locations or register equivalence rules.

</details>

### Q154. After upgrading the client to a new OpenSSH, connections to an old appliance fail with `no matching host key type found. Their offer: ssh-rsa,ssh-dss`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** New OpenSSH (8.8+) disables `ssh-rsa` signatures (SHA-1) by default; the old server only offers those.

**Diagnose →**
```bash
ssh -vvv admin@appliance 2>&1 | grep -i 'host key'
ssh -Q key
```

**Fix →** Enable it only for that host in `~/.ssh/config`:
```
Host appliance
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```
Plan to upgrade the appliance.

**Prevent →** Scope legacy exceptions to specific hosts, never `Host *`.

</details>

### Q155. A user's RSA key stopped working against a new server: the server log says `userauth_pubkey: key type ssh-rsa not in PubkeyAcceptedAlgorithms`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The client signs with the deprecated `ssh-rsa` (SHA-1) algorithm (old client), or the server's crypto policy disallows it.

**Diagnose →**
```bash
ssh -V
ssh -v user@server 2>&1 | grep -i 'signature\|rsa'
update-crypto-policies --show            # RHEL server
```

**Fix →** Upgrade the client so it uses `rsa-sha2-256/512`, or better, generate an Ed25519 key: `ssh-keygen -t ed25519`.

**Prevent →** Standardise on Ed25519 keys.

</details>

### Q156. SSH login takes 20–30 seconds before the password or key prompt appears. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** sshd does a reverse DNS lookup of the client (`UseDNS yes`) and DNS times out, or GSSAPI authentication attempts time out.

**Diagnose →**
```bash
ssh -vvv user@server 2>&1 | grep -E 'debug1:.*(GSSAPI|Authentications)' 
grep -Ei 'UseDNS|GSSAPIAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*
```

**Fix →** Set `UseDNS no` and `GSSAPIAuthentication no` (if not using Kerberos) on the server; on the client `GSSAPIAuthentication no`. Fix DNS if it's broken generally.

**Prevent →** Keep sshd baseline config consistent.

</details>

### Q157. SSH fails immediately with `kex_exchange_identification: read: Connection reset by peer`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server closes the connection before the SSH banner: TCP wrappers/`hosts.deny`, fail2ban or a firewall resetting, `MaxStartups` exceeded (many unauthenticated connections), or sshd crashing per-connection.

**Diagnose →**
```bash
# server side
sudo journalctl -u sshd -n 50
sudo fail2ban-client status sshd 2>/dev/null
grep -i maxstartups /etc/ssh/sshd_config
cat /etc/hosts.deny 2>/dev/null
```

**Fix →** Unban the client IP (`fail2ban-client set sshd unbanip 1.2.3.4`), raise `MaxStartups 30:30:100` if under scanning load, fix the crash shown in logs.

**Prevent →** Restrict SSH exposure so scanners can't exhaust `MaxStartups`.

</details>

### Q158. `ssh` reports `ssh: connect to host server01 port 22: Connection refused` after you edited `sshd_config`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** sshd failed to restart because of a config syntax error, or it now listens on a different port.

**Diagnose →** (via console)
```bash
sudo sshd -t
systemctl status sshd
sudo ss -tlnp | grep sshd
```

**Fix →** Correct the reported line, `sshd -t`, then `systemctl restart sshd`.

**Prevent →** Always run `sshd -t` before restart and keep an open session while testing.

</details>

### Q159. An SSH key with a passphrase works interactively, but scripts fail with `Enter passphrase for key` prompts or `Permission denied` in automation. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No ssh-agent available in the non-interactive context.

**Diagnose →**
```bash
echo $SSH_AUTH_SOCK
ssh-add -l
```

**Fix →** For automation, use a dedicated passphrase-less key restricted on the server (`from=`, `command=`), or a CI secret loaded into an agent at job start (`eval $(ssh-agent) && ssh-add - <<< "$KEY"`).

**Prevent →** Separate human and machine keys.

</details>

### Q160. Agent forwarding (`ssh -A`) works to the bastion, but `git pull` on the next host says `Permission denied (publickey)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Forwarding is disabled on the target (`AllowAgentForwarding no`), you used `sudo` (which drops `SSH_AUTH_SOCK`), or forwarding wasn't requested on the second hop.

**Diagnose →**
```bash
echo $SSH_AUTH_SOCK; ssh-add -l      # on the inner host
sudo -E env | grep SSH_AUTH_SOCK
```

**Fix →** Use `ProxyJump` so keys never leave your laptop, and connect to the final host directly: `ssh -J bastion app01`. For git on the server, use deploy keys instead.

**Prevent →** Prefer `ProxyJump` over `-A`; forwarded agents can be abused by root on intermediate hosts.

</details>

### Q161. SSH certificate login fails with `error: Certificate invalid: name is not a listed principal`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The certificate's principals don't include the target username or any principal in `AuthorizedPrincipalsFile`.

**Diagnose →**
```bash
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub      # Principals, Valid
grep -E 'TrustedUserCAKeys|AuthorizedPrincipals' /etc/ssh/sshd_config
cat /etc/ssh/auth_principals/deploy 2>/dev/null
```

**Fix →** Reissue the certificate with the right principal (`ssh-keygen -s ca -I alice -n deploy,alice -V +8h key.pub`) or add the principal to the server's principals file.

**Prevent →** Map principals to roles centrally and test cert issuance in CI.

</details>

### Q162. SSH certificate login fails with `Certificate invalid: expired`, even though it was issued an hour ago with 8h validity. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Clock skew on the server (or CA) makes the cert look expired or not yet valid.

**Diagnose →**
```bash
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub | grep Valid
ssh server date -u 2>/dev/null; date -u
chronyc tracking
```

**Fix →** Fix time sync on the server/CA. Issue certs with a small backdated start (`-V -5m:+8h`) to tolerate minor skew.

**Prevent →** Monitor NTP offset on every host.

</details>

### Q163. `sudo` says `alice is not in the sudoers file. This incident will be reported.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user isn't in the admin group (`wheel` on RHEL, `sudo` on Debian/Ubuntu) and has no sudoers rule, or the session predates the group addition.

**Diagnose →**
```bash
id alice
sudo -l -U alice              # as an admin
grep -E '^%(wheel|sudo)' /etc/sudoers
```

**Fix →** `usermod -aG wheel alice` (or `sudo`), then have them log in again; or add a specific rule in `/etc/sudoers.d/`.

**Prevent →** Manage sudo via groups from central identity.

</details>

### Q164. `sudo` hangs for several seconds and prints `sudo: unable to resolve host app01: Name or service not known`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hostname was changed but isn't resolvable (missing from `/etc/hosts` and DNS).

**Diagnose →**
```bash
hostname
getent hosts "$(hostname)"
cat /etc/hosts
```

**Fix →** Add `127.0.1.1 app01` (Debian style) or the host's IP and FQDN to `/etc/hosts`, or fix DNS.

**Prevent →** Update `/etc/hosts` whenever the hostname changes (cloud-init `manage_etc_hosts: true`).

</details>

### Q165. After editing `/etc/sudoers` with a text editor, every `sudo` command fails with `sudo: parse error in /etc/sudoers near line 31` and nobody can become root. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Syntax error in sudoers; sudo refuses to run with a broken policy.

**Diagnose →**
```bash
pkexec visudo -c 2>/dev/null       # if polkit is available
```

**Fix →** Use `pkexec visudo` if polkit works; otherwise log in as root on the console, or reboot into rescue mode (`rd.break` / `init=/bin/bash`), remount rw and fix the line with `visudo -c -f /sysroot/etc/sudoers`.

**Prevent →** Only edit with `visudo` (it validates), and use drop-ins in `/etc/sudoers.d/` deployed with validation.

</details>

### Q166. A user's password login fails with `Your account has expired; please contact your system administrator`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The account expiry date (`chage -E`) has passed, set by policy or offboarding.

**Diagnose →**
```bash
sudo chage -l alice
```

**Fix →** If access is approved: `sudo chage -E -1 alice` (remove expiry) or set a new date.

**Prevent →** Notify users and managers before account expiry; tie expiry to HR data.

</details>

### Q167. A user gets `You are required to change your password immediately (administrator enforced)` and the change fails over SSH with `Authentication token manipulation error`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The password is forced to change (`chage -d 0`), and the update fails because `/etc/shadow` is not writable (read-only root, wrong SELinux label, immutable attribute) or pwquality rejects it.

**Diagnose →**
```bash
sudo chage -l alice
lsattr /etc/shadow; ls -lZ /etc/shadow
findmnt -no OPTIONS /
sudo journalctl -t passwd -n 20
```

**Fix →** Fix the shadow file issue (`chattr -i`, `restorecon`, remount rw), or set a password as root: `passwd alice`.

**Prevent →** Avoid immutable flags on auth files; monitor for read-only root.

</details>

### Q168. Users get locked out after a few wrong passwords and the lock persists: `pam_faillock(sshd:auth): Consecutive login failures for user alice account temporarily locked`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `pam_faillock` (RHEL 8/9, newer Ubuntu) locking after N failures; automated scripts with an old password can trigger it repeatedly.

**Diagnose →**
```bash
sudo faillock --user alice
grep -Ev '^#|^$' /etc/security/faillock.conf
```

**Fix →** `sudo faillock --user alice --reset`. Find and fix the source of failures (a script, a mounted share using old credentials).

**Prevent →** Set reasonable `deny` and `unlock_time` in `faillock.conf`; use keys for automation.

</details>

### Q169. LDAP/AD users can't log in: `id aduser` returns `no such user`, but local users are fine. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSSD down or offline, DNS can't find domain controllers, Kerberos clock skew, or the computer account in AD is broken.

**Diagnose →**
```bash
systemctl status sssd
sudo sssctl domain-status corp.example.com
dig -t SRV _ldap._tcp.corp.example.com
chronyc tracking
sudo journalctl -u sssd -n 50
```

**Fix →** Restart SSSD after fixing DNS/time; clear the cache (`sss_cache -E`); rejoin the domain if the machine account is broken (`realm leave` / `realm join`).

**Prevent →** Monitor SSSD domain status and time sync; keep a local break-glass admin.

</details>

### Q170. Kerberos-based SSH login fails with `Clock skew too great` in the logs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Server or client clock differs from the KDC by more than 5 minutes (the default).

**Diagnose →**
```bash
date -u; chronyc tracking
klist
```

**Fix →** Fix NTP sync on the affected host (`chronyc makestep`), then re-acquire tickets (`kinit`).

**Prevent →** Point all domain members at the domain controllers or the same NTP source.

</details>

### Q171. `su - appuser` returns `This account is currently not available.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The account's shell is `/sbin/nologin`, by design for service accounts.

**Diagnose →**
```bash
getent passwd appuser
```

**Fix →** Run commands as the user without changing their shell: `sudo -u appuser -s /bin/bash` or `sudo -u appuser <command>`. Only change the shell if there's a real need.

**Prevent →** Document how admins should act as service accounts.

</details>

### Q172. SFTP users in a chroot get disconnected immediately; the server log says `fatal: bad ownership or modes for chroot directory "/sftp/partner1"`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The chroot directory and every parent must be owned by root and not writable by group/others.

**Diagnose →**
```bash
namei -l /sftp/partner1
sudo journalctl -u sshd | grep -i chroot
```

**Fix →**
```bash
sudo chown root:root /sftp /sftp/partner1
sudo chmod 755 /sftp /sftp/partner1
sudo mkdir -p /sftp/partner1/upload && sudo chown partner1: /sftp/partner1/upload
```

**Prevent →** Template the chroot creation; users write only to subdirectories.

</details>

### Q173. SSH port forwarding fails with `channel 3: open failed: administratively prohibited: open failed`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The server disallows TCP forwarding (`AllowTcpForwarding no`, `PermitOpen` restrictions), or the key has `no-port-forwarding` / `restrict` options in `authorized_keys`.

**Diagnose →**
```bash
sudo sshd -T | grep -Ei 'allowtcpforwarding|permitopen'
grep -n 'no-port-forwarding\|restrict' ~/.ssh/authorized_keys
```

**Fix →** If approved, enable forwarding for that user/group via `Match` blocks, or add a `PermitOpen host:port` allowlist.

**Prevent →** Document which roles may tunnel; forwarding is often deliberately disabled.

</details>

### Q174. `ssh` with a FIDO2 key fails: `Confirm user presence for key ED25519-SK` never appears and you get `sign_and_send_pubkey: signing failed for ED25519-SK`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The security key isn't detected (USB/udev permissions), missing `libfido2`/middleware, or the key requires a PIN (`verify-required`) that wasn't supplied.

**Diagnose →**
```bash
ssh -vvv host 2>&1 | grep -iE 'sk|fido'
fido2-token -L 2>/dev/null
ls -l /dev/hidraw*
```

**Fix →** Install `libfido2`, fix udev rules so the user can access the hidraw device, touch the key when prompted, or enter the PIN.

**Prevent →** Document FIDO2 setup per OS; keep a backup key enrolled.

</details>

### Q175. Logins over SSH succeed, but the user lands in `/` with `Could not chdir to home directory /home/bob: No such file or directory`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The home directory wasn't created (user added without `-m`, or a directory-service user with no `mkhomedir`), or an NFS/autofs home share isn't mounted.

**Diagnose →**
```bash
getent passwd bob
ls -ld /home/bob
findmnt /home
```

**Fix →** Create it: `sudo mkhomedir_helper bob` or `install -d -m 700 -o bob -g bob /home/bob`. For directory users, enable `pam_mkhomedir` (`authselect enable-feature with-mkhomedir`, plus `oddjobd`).

**Prevent →** Use `useradd -m`, or enable mkhomedir for central accounts.

</details>

## Boot, time, cron and kernel

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. After a reboot the server drops to `You are in emergency mode. After logging in, type "journalctl -xb" to view system logs`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A required mount in `/etc/fstab` failed (wrong UUID, removed disk, typo, filesystem errors), so `local-fs.target` couldn't be reached.

**Diagnose →**
```bash
journalctl -xb | grep -Ei 'failed|dependency|mount'
systemctl --failed
cat /etc/fstab; lsblk -f
```

**Fix →** Fix or comment out the bad fstab line (`mount -o remount,rw /` first if root is read-only), run `systemctl daemon-reload && mount -a` to confirm, then `systemctl default` or reboot. If the filesystem is damaged, run `fsck`/`xfs_repair` on it.

**Prevent →** Use `nofail` for non-critical mounts and always run `mount -a` after editing fstab.

</details>

### Q177. Boot stops at `dracut-initqueue: Warning: dracut-initqueue timeout - starting timeout scripts` and then a `dracut:/#` shell. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The initramfs can't find the root device: wrong `root=` UUID on the kernel command line, missing storage driver in initramfs, LVM VG renamed, or a LUKS device not unlocked.

**Diagnose →** In the dracut shell:
```bash
cat /proc/cmdline
blkid
lvm vgscan; lvm lvs
ls /dev/mapper
```

**Fix →** If LVM is inactive: `lvm vgchange -ay` then `exit` to continue. Permanently: boot rescue media, chroot, fix `root=`/`rd.lvm.lv=` with `grubby`, and rebuild initramfs with `dracut -f --kver <version>`.

**Prevent →** Rebuild initramfs after storage changes and test reboots in a window.

</details>

### Q178. A server shows a `grub>` prompt instead of booting. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** GRUB can't find its config (`grub.cfg` missing or wrong partition), often after disk changes, a failed update or a resized/moved `/boot`.

**Diagnose →** At the prompt:
```
grub> ls
grub> ls (hd0,gpt2)/
```
Find the partition with `/boot/vmlinuz-*` or `/grub2/grub.cfg`.

**Fix →** Boot manually:
```
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-<ver> root=/dev/mapper/vg0-root ro
grub> initrd /initramfs-<ver>.img
grub> boot
```
Then regenerate: `grub2-mkconfig -o /boot/grub2/grub.cfg` (BIOS) or the EFI path, and reinstall GRUB if needed (`grub2-install /dev/sda` on BIOS).

**Prevent →** Keep a tested rescue procedure and don't touch `/boot` layout without a snapshot.

</details>

### Q179. The kernel panics at boot: `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing or broken initramfs for that kernel (an interrupted update, `/boot` full during install), so the kernel has no drivers to find root.

**Diagnose →** Boot an older kernel from the GRUB menu, then:
```bash
ls -l /boot/initramfs-* /boot/initrd.img-*
df -h /boot
rpm -q kernel || dpkg -l 'linux-image*'
```

**Fix →** Free space in `/boot`, then rebuild: `dracut -f --kver <newver>` (RHEL) or `update-initramfs -c -k <newver>` (Debian), or reinstall the kernel package.

**Prevent →** Keep `/boot` usage low (`installonly_limit`), and alert on it.

</details>

### Q180. Boot hangs for 90 seconds with `A start job is running for dev-disk-by\x2duuid-...device (1min 30s)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An fstab or crypttab entry (often swap or a data disk) references a device that no longer exists.

**Diagnose →**
```bash
journalctl -b | grep -i 'timed out waiting for device'
systemd-escape -u 'dev-disk-by\x2duuid-...'
blkid; cat /etc/fstab
```

**Fix →** Correct the UUID or remove the entry, add `nofail` (and `x-systemd.device-timeout=10s`) for optional devices.

**Prevent →** Update fstab whenever disks are replaced or reformatted.

</details>

### Q181. Logs from different servers are out of order and `timedatectl` shows `System clock synchronized: no` and `NTP service: inactive`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No time sync daemon running (chrony not installed/enabled, or conflicting `ntpd` and `chronyd`).

**Diagnose →**
```bash
timedatectl
systemctl status chronyd ntpd systemd-timesyncd 2>/dev/null | grep -E 'Loaded|Active'
```

**Fix →** `sudo dnf install -y chrony && sudo systemctl enable --now chronyd` (or `systemctl enable --now systemd-timesyncd` on minimal Ubuntu), then `timedatectl set-ntp true`. Run only one sync daemon.

**Prevent →** Include chrony in the base image and alert on unsynchronised hosts.

</details>

### Q182. `chronyc sources` shows all servers with `?` or `^?` and the clock is drifting. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** NTP servers unreachable: UDP 123 blocked by a firewall/security group, wrong server names, or DNS failing for pool names.

**Diagnose →**
```bash
chronyc sources -v
chronyc activity
getent hosts 2.pool.ntp.org
sudo tcpdump -ni any udp port 123 -c 10
```

**Fix →** Allow outbound UDP 123, or use an internal/cloud time source (`169.254.169.123` on AWS, `metadata.google.internal` on GCP). Restart chronyd.

**Prevent →** Use the cloud provider's local time service; monitor `chronyc tracking` offset.

</details>

### Q183. The clock is 10 minutes off and chrony is running, but it corrects only very slowly. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** chrony slews (gradually adjusts) the clock after startup, and only steps it during the first updates if `makestep` allows. Large offsets later take a long time to slew away.

**Diagnose →**
```bash
chronyc tracking            # System time offset
grep makestep /etc/chrony.conf
```

**Fix →** Step it now: `sudo chronyc makestep`. Keep `makestep 1.0 3` in `chrony.conf` so large offsets at boot get stepped.

**Prevent →** Investigate why it drifted (VM pause/resume, wrong clocksource, NTP blocked).

</details>

### Q184. A cron job hasn't run for days; nothing in its log, no errors. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The cron daemon isn't running or isn't enabled, the crontab file is invalid, or the job's own log is just not being written.

**Diagnose →**
```bash
systemctl status crond 2>/dev/null || systemctl status cron
journalctl -u crond -u cron --since "2 days ago" | tail
sudo crontab -l -u app
grep CRON /var/log/syslog 2>/dev/null | tail     # Debian/Ubuntu
```

**Fix →** `sudo systemctl enable --now crond` (RHEL) or `cron` (Debian). Fix crontab syntax errors reported in the journal.

**Prevent →** Use a dead-man's switch so missed runs alert, and include the cron service in health checks.

</details>

### Q185. A file you dropped into `/etc/cron.d/` is ignored. The cron log shows nothing about it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** On Debian/Ubuntu, `run-parts` naming rules: filenames with dots (e.g. `backup.cron`) are skipped. Also: missing user field, group/world-writable file, or not owned by root.

**Diagnose →**
```bash
ls -l /etc/cron.d/
cat /etc/cron.d/backup.cron
journalctl -u cron | grep -i 'bad\|insecure\|wrong'
```

**Fix →** Rename to `backup` (letters, digits, `-`, `_` only), `chmod 644`, `chown root:root`, and include the user column: `30 2 * * * root /usr/local/bin/backup.sh`.

**Prevent →** Lint cron files in config management.

</details>

### Q186. A crontab entry's last line never runs, but all other lines do. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file doesn't end with a newline; cron ignores an unterminated final line (common with files deployed by tools or edited by hand).

**Diagnose →**
```bash
tail -c1 /etc/cron.d/jobs | xxd      # should be 0a
```

**Fix →** Add a trailing newline: `echo >> /etc/cron.d/jobs`.

**Prevent →** Use templates that end with a newline; `crontab -e` usually handles this.

</details>

### Q187. A cron job runs at the wrong hour: scheduled for 02:00 but runs at 07:30. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The system timezone differs from what you assumed (server in UTC, you expected IST), or the timezone was changed and cron wasn't restarted.

**Diagnose →**
```bash
timedatectl | grep 'Time zone'
date; date -u
systemctl show crond -p ActiveEnterTimestamp 2>/dev/null
```

**Fix →** Write schedules in the server's timezone (UTC recommended), or set `CRON_TZ=Asia/Kolkata` at the top of the crontab where supported (cronie). Restart cron after timezone changes.

**Prevent →** Keep servers on UTC and document job times in UTC.

</details>

### Q188. A systemd timer is enabled but `systemctl list-timers` shows `n/a` for NEXT and it never fires. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The timer was enabled but not started, the `OnCalendar=` expression is invalid, or the matching `.service` doesn't exist.

**Diagnose →**
```bash
systemctl status backup.timer
systemd-analyze calendar 'Mon..Fri *-*-* 02:00:00'
systemctl cat backup.service
```

**Fix →** Correct the expression, `daemon-reload`, then `systemctl enable --now backup.timer`.

**Prevent →** Validate calendar expressions with `systemd-analyze calendar` before deploying.

</details>

### Q189. A daily cron job worked for months but now fails with `/bin/sh: 1: /opt/tools/report.sh: not found`, although the file exists. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The script's shebang points to an interpreter that's gone (e.g. `#!/usr/bin/python` after Python 2 removal), or it's a binary for another architecture or missing its dynamic loader. The "not found" refers to the interpreter, not the script.

**Diagnose →**
```bash
head -1 /opt/tools/report.sh
file /opt/tools/report.sh
ls -l "$(head -1 /opt/tools/report.sh | sed 's/^#!//; s/ .*//')"
```

**Fix →** Update the shebang (`#!/usr/bin/env python3`), install the interpreter, or rebuild the binary for this architecture.

**Prevent →** Use `env`-based shebangs and test scripts after OS upgrades.

</details>

### Q190. `dmesg` shows `INFO: task mysqld:2231 blocked for more than 120 seconds.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The hung task detector fired: a process stuck in D state for 2+ minutes, usually storage stalls (overloaded or failing disk, unresponsive SAN/EBS, NFS) or huge dirty page writeback.

**Diagnose →**
```bash
dmesg -T | grep -A 20 'blocked for more than'     # kernel stack shows where it waits
iostat -xz 1 5
cat /proc/pressure/io
grep -E 'Dirty|Writeback' /proc/meminfo
```

**Fix →** Resolve the storage bottleneck (faster volume, fix the failing device), tune dirty page limits, or fix the network storage.

**Prevent →** Alert on hung task messages and disk latency; don't run databases on network storage without performance guarantees.

</details>

### Q191. A server rebooted unexpectedly overnight. How do you find out why? 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kernel panic, OOM-induced watchdog, hardware fault, power loss, a scheduled automatic update reboot, or someone ran `reboot`.

**Diagnose →**
```bash
last -x | grep -E 'reboot|shutdown' | head
journalctl --list-boots
journalctl -b -1 -e                         # end of the previous boot
journalctl -b -1 | grep -Ei 'panic|watchdog|oom|reboot|shutdown'
ls /var/crash/ 2>/dev/null
```
Check the cloud provider's events and hardware logs (`ipmitool sel list`).

**Fix →** Depends on the cause found: clean shutdown messages point to a person or automation; an abrupt end of logs points to panic, power or hardware.

**Prevent →** Persistent journal, kdump, and monitoring of hardware events.

</details>

### Q192. `dmesg` shows `watchdog: BUG: soft lockup - CPU#3 stuck for 23s!` on a VM. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CPU didn't schedule for a long time: on VMs usually hypervisor contention (vCPU not run, host overloaded, live migration); on bare metal, a kernel or driver bug.

**Diagnose →**
```bash
dmesg -T | grep -A 30 'soft lockup'      # stack trace of what was running
mpstat 1 5                               # %steal
uname -r
```
Check hypervisor/cloud events for host maintenance.

**Fix →** For steal/hypervisor issues, move the VM or resize the host. For driver bugs, update the kernel/driver indicated in the stack.

**Prevent →** Avoid overcommitted hosts for latency-sensitive VMs; keep kernels current.

</details>

### Q193. `modprobe` fails with `modprobe: ERROR: could not insert 'vboxdrv': Key was rejected by service`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Secure Boot is enabled and the module isn't signed with a trusted key.

**Diagnose →**
```bash
mokutil --sb-state
modinfo vboxdrv | grep -i sig
dmesg | grep -i 'lockdown\|signature'
```

**Fix →** Sign the module and enroll the key with `mokutil --import`, or use the distro's signed module packages.

**Prevent →** Use DKMS with automatic signing configured for third-party modules.

</details>

### Q194. After a kernel update, an out-of-tree driver (e.g. NVIDIA or a storage driver) doesn't load: `modprobe: FATAL: Module nvidia not found in directory /lib/modules/6.x...`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module was built for the previous kernel and hasn't been rebuilt (DKMS not installed, build failed, or missing kernel headers).

**Diagnose →**
```bash
uname -r
dkms status
rpm -q kernel-devel-$(uname -r) || dpkg -l linux-headers-$(uname -r)
```

**Fix →** Install matching headers and rebuild: `dkms autoinstall -k $(uname -r)`, or boot the previous kernel until the vendor package supports the new one.

**Prevent →** Use DKMS or kmod packages, and lock kernel versions on hosts with critical out-of-tree drivers.

</details>

### Q195. A server suddenly remounts root read-only during the day and apps start failing with `Read-only file system`. The disk is an EBS/cloud volume. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The block device returned I/O errors or timed out (cloud volume impairment, NVMe timeouts), and ext4 with `errors=remount-ro` protected itself.

**Diagnose →**
```bash
dmesg -T | grep -Ei 'nvme|I/O error|EXT4-fs error|Remounting'
cat /sys/module/nvme_core/parameters/io_timeout
```
Check the cloud console for volume status checks.

**Fix →** Schedule a reboot, which runs fsck on the next mount; if the volume is impaired, restore from snapshot. Make sure the NVMe I/O timeout follows the cloud's guidance (e.g. a high `nvme_core.io_timeout` on AWS).

**Prevent →** Volume health alarms, snapshots, and redundant architecture so one volume isn't critical.

</details>

### Q196. `journalctl -b -1` says `Specifying boot ID or boot offset has no effect, no persistent journal was found`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** journald is in volatile mode (no `/var/log/journal`), so previous boots weren't saved.

**Diagnose →**
```bash
ls -ld /var/log/journal
grep -i '^Storage' /etc/systemd/journald.conf
```

**Fix →** For this incident, check `/var/log/messages` or `syslog` if rsyslog runs, or central logs. Then enable persistence: `mkdir -p /var/log/journal && systemctl restart systemd-journald`.

**Prevent →** Persistent journal in the base image.

</details>

### Q197. A VM boots very slowly, and `systemd-analyze blame` shows `NetworkManager-wait-online.service` taking 1m30s. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An interface configured for DHCP never gets a lease (an unused NIC), so wait-online waits for its timeout.

**Diagnose →**
```bash
systemd-analyze critical-chain
nmcli dev status
journalctl -b -u NetworkManager | grep -i dhcp
```

**Fix →** Disable autoconnect on unused interfaces (`nmcli con mod <name> connection.autoconnect no`), or mark them not required. Disable wait-online only if no service needs `network-online.target`.

**Prevent →** Keep network profiles in images minimal and accurate.

</details>

### Q198. After a power outage, the server boots but `/var` is mounted read-only and `dmesg` shows `EXT4-fs (dm-2): warning: mounting fs with errors, running e2fsck is recommended`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unclean shutdown left inconsistencies the journal couldn't resolve; ext4 detected errors.

**Diagnose →**
```bash
sudo tune2fs -l /dev/mapper/vg0-var | grep -Ei 'state|errors'
findmnt /var
dmesg -T | grep EXT4
```

**Fix →** Boot into rescue or emergency mode so `/var` isn't in use (`systemctl rescue`), unmount it, run `e2fsck -f /dev/mapper/vg0-var`, then reboot. Take a snapshot first if the data matters.

**Prevent →** UPS, storage with battery-backed cache, and monitoring of filesystem error state.

</details>

### Q199. `systemctl` reports `System has not been booted with systemd as init system (PID 1). Can't operate.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** You're inside a container, a chroot or WSL instance without systemd, where PID 1 is something else.

**Diagnose →**
```bash
ps -p 1 -o comm=
cat /proc/1/cgroup | head -3
```

**Fix →** In containers, run processes directly or use the image's entrypoint. In a chroot (rescue), use `systemctl --root=/mnt/sysroot enable foo` for offline changes. On WSL, enable systemd in `/etc/wsl.conf` (`[boot] systemd=true`).

**Prevent →** Don't rely on systemd in container images.

</details>

### Q200. The kernel logs `TCP: out of memory -- consider tuning tcp_mem` and network connections start failing on a busy proxy. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Total TCP buffer memory reached the `net.ipv4.tcp_mem` pressure/max limits (many connections with large buffers or slow clients), or orphaned sockets are piling up.

**Diagnose →**
```bash
cat /proc/net/sockstat             # TCP: inuse, orphan, mem (in pages)
sysctl net.ipv4.tcp_mem net.ipv4.tcp_max_orphans
ss -s
```

**Fix →** Raise `net.ipv4.tcp_mem` (in pages) in line with RAM, reduce per-socket buffer maximums if they are oversized, and set timeouts so slow or idle clients get closed.

**Prevent →** Graph `sockstat` memory and orphans; size proxies for peak connection counts.

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
