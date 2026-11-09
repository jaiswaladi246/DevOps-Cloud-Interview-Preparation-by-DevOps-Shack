# Linux — Scenario-Based Questions

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

> Real-world Linux situations and how to reason through them in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Users, groups and access control](#users-groups-and-access-control) — Q1–Q25
- [Storage, disks and filesystems](#storage-disks-and-filesystems) — Q26–Q50
- [Processes, services and systemd](#processes-services-and-systemd) — Q51–Q75
- [Scheduling, automation and scripting](#scheduling-automation-and-scripting) — Q76–Q100
- [Networking, SSH and remote access](#networking-ssh-and-remote-access) — Q101–Q125
- [Security, hardening and incident response](#security-hardening-and-incident-response) — Q126–Q150
- [Packages, kernel and boot](#packages-kernel-and-boot) — Q151–Q175
- [Migrations, performance and fleet operations](#migrations-performance-and-fleet-operations) — Q176–Q200

---

## Users, groups and access control

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You need to onboard 50 new developers onto a shared build server, each with their own account and SSH key. How do you do it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bulk, repeatable user creation with consistent groups and key-only login.

**What I'd do →**
1. Get a CSV of `username,pubkey` from the ticket.
2. Create a `developers` group once: `sudo groupadd developers`.
3. Loop over the file:
```bash
while IFS=, read -r u key; do
  sudo useradd -m -s /bin/bash -G developers "$u"
  sudo install -d -m 700 -o "$u" -g "$u" /home/$u/.ssh
  echo "$key" | sudo tee /home/$u/.ssh/authorized_keys >/dev/null
  sudo chown "$u:$u" /home/$u/.ssh/authorized_keys
  sudo chmod 600 /home/$u/.ssh/authorized_keys
done < users.csv
```
4. Lock passwords (`passwd -l`) since login is key-only, and verify with `getent group developers`.

**Why →** Scripted creation avoids typos and wrong permissions that break key auth.

**Interview tip →** Say that beyond a handful of servers you would do this with Ansible's `user` and `authorized_key` modules, or central identity (LDAP/SSSD).

</details>

### Q2. Three teams share one server and each needs a project directory only its members can write to, with new files staying group-owned. How do you set it up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic group-shared directory with the setgid bit.

**What I'd do →**
1. Create groups `team-a`, `team-b`, `team-c` and add members with `usermod -aG`.
2. Create directories and set ownership:
```bash
sudo mkdir -p /srv/projects/{team-a,team-b,team-c}
for t in team-a team-b team-c; do
  sudo chown root:$t /srv/projects/$t
  sudo chmod 2770 /srv/projects/$t
done
```
3. Set `umask 002` for those users (in `/etc/profile.d/`) so files are group-writable.

**Why →** The setgid bit (`2` in `2770`) makes new files inherit the directory's group instead of the creator's primary group.

**Interview tip →** Mention that users must log out and back in for new group membership to apply.

</details>

### Q3. A junior engineer needs to restart nginx but must not get full root. How do you grant that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least privilege through a narrow sudo rule.

**What I'd do →**
1. Create a drop-in with `sudo visudo -f /etc/sudoers.d/nginx-ops`:
```
%nginx-ops ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx, /usr/bin/systemctl status nginx
```
2. Add the user to `nginx-ops`.
3. Test: `sudo -l -U alice` and have them run `sudo systemctl reload nginx`.

**Why →** Full paths and exact arguments stop the rule from being used to run arbitrary commands. `visudo` validates syntax so you can't lock yourself out of sudo.

**Interview tip →** Point out that granting `sudo vim` or `sudo less` is effectively root because they can spawn a shell.

</details>

### Q4. An engineer leaves the company today. What do you do with their Linux account on 30 servers? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Revoke access immediately but keep data and audit trail.

**What I'd do →**
1. Lock the account and kill sessions:
```bash
sudo usermod -L -e 1 bob        # lock password, expire account
sudo pkill -KILL -u bob
sudo mv /home/bob/.ssh/authorized_keys /root/offboard-bob.keys
```
2. Search for their keys in other accounts: `grep -r "bob@" /home/*/.ssh/authorized_keys /root/.ssh/authorized_keys`.
3. Check `crontab -l -u bob` and running services owned by them.
4. After the retention period, archive the home dir and `userdel -r bob`.

**Why →** Locking first is reversible; deleting right away can break jobs or ownership that nobody knew depended on that user.

**Interview tip →** Say you do this from a central tool (Ansible, IdP/LDAP disable) so no server is missed.

</details>

### Q5. A service account for a backup tool must exist on every server but nobody should be able to log in as it. How do you create it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** System user with no shell, no password, no home clutter.

**What I'd do →**
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin backupsvc
sudo passwd -l backupsvc
getent passwd backupsvc
```
Give it ownership only of what it needs, for example `/var/backups/app`.

**Why →** `--system` uses a low UID range and no aging; `nologin` blocks interactive shells while still allowing the process to run as that user.

**Interview tip →** Mention systemd `DynamicUser=yes` as an alternative when the service needs no persistent identity.

</details>

### Q6. Your security team requires passwords to expire every 90 days for human users only. How do you implement it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Password aging policy, applied to existing and future users.

**What I'd do →**
1. For new users, set defaults in `/etc/login.defs`: `PASS_MAX_DAYS 90`, `PASS_MIN_DAYS 1`, `PASS_WARN_AGE 14`.
2. For existing humans (UID >= 1000):
```bash
awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd | xargs -I{} sudo chage -M 90 -m 1 -W 14 {}
chage -l alice
```
3. Exclude service accounts and key-only accounts where expiry would just break automation.

**Why →** `login.defs` only affects accounts created afterwards; `chage` fixes the existing ones.

**Interview tip →** Note that modern guidance (NIST 800-63B) prefers strong passwords plus MFA over forced rotation, but you implement what policy says.

</details>

### Q7. Two developers need read access to `/var/log/app/` without being added to the `adm` or root group. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fine-grained access beyond owner/group/other: use ACLs.

**What I'd do →**
```bash
sudo setfacl -R -m u:alice:rX,u:bob:rX /var/log/app
sudo setfacl -d -m u:alice:rX,u:bob:rX /var/log/app   # default ACL for new files
getfacl /var/log/app
```
Check that logrotate does not recreate files with modes that drop the ACL mask (`create 0640` keeps the mask usable).

**Why →** ACLs grant per-user rights without changing the file's owner or group, and default ACLs apply to files created later.

**Interview tip →** Mention the `+` in `ls -l` output as the sign that an ACL exists.

</details>

### Q8. You have 200 servers and user accounts are drifting between them. How do you centralise identity? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local `/etc/passwd` management does not scale; move to a directory service.

**What I'd do →**
1. Pick a source of truth: FreeIPA, Active Directory, or an LDAP behind your IdP.
2. Join servers with SSSD (`realm join` for AD/IPA) and use `sssd` in `/etc/nsswitch.conf` for `passwd` and `group`.
3. Store SSH public keys in the directory and serve them via `AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys`.
4. Control who may log in per host group with HBAC rules or `simple_allow_groups`.
5. Keep one local break-glass admin per server.

**Why →** One place to add, remove and audit users; offboarding becomes a single disable.

**Interview tip →** Mention SSSD caching so logins still work if the directory is briefly unreachable.

</details>

### Q9. A contractor needs SFTP access to upload files to one directory and nothing else. How do you set that up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chrooted SFTP-only account.

**What I'd do →**
1. `sudo useradd -s /usr/sbin/nologin -G sftponly contractor1`
2. Prepare the chroot (must be root-owned and not group-writable):
```bash
sudo mkdir -p /sftp/contractor1/upload
sudo chown root:root /sftp/contractor1 && sudo chmod 755 /sftp/contractor1
sudo chown contractor1:contractor1 /sftp/contractor1/upload
```
3. In `/etc/ssh/sshd_config`:
```
Match Group sftponly
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```
4. `sshd -t && systemctl reload sshd`, then test with `sftp`.

**Why →** `internal-sftp` needs no binaries inside the chroot, and `ForceCommand` blocks shell access.

**Interview tip →** The most common failure is chroot ownership; sshd refuses if the chroot path is writable by the user.

</details>

### Q10. Auditors want to know exactly which commands admins ran with sudo last month. How do you provide that going forward? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Command-level audit trail that survives on-box tampering.

**What I'd do →**
1. Enable sudo logging in `/etc/sudoers.d/logging`:
```
Defaults logfile=/var/log/sudo.log
Defaults log_input, log_output
Defaults iolog_dir=/var/log/sudo-io/%{user}
```
2. Add auditd rules for execve by admins: `-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -k root-cmds`.
3. Ship `/var/log/secure` or `auth.log`, sudo logs and audit logs to a central SIEM.
4. Answer queries with `ausearch -k root-cmds --start this-month` or in the SIEM.

**Why →** Local logs can be wiped by root; forwarding them off-box keeps them trustworthy.

**Interview tip →** `sudoreplay` can replay recorded sessions when `log_output` is on.

</details>

### Q11. Developers keep running `sudo su -` and doing everything as root. How do you change that without blocking their work? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared root sessions destroy accountability; replace with per-command sudo.

**What I'd do →**
1. List what they actually need (restart services, read logs, deploy) from sudo logs.
2. Write `Cmnd_Alias` rules for those commands in `/etc/sudoers.d/`.
3. Remove the broad `ALL` rule, and explicitly deny shells: `!/bin/su, !/usr/bin/su, !/bin/bash` (as defence, knowing deny lists are bypassable).
4. Give the team `journalctl` access via the `systemd-journal` group instead of sudo.
5. Communicate, roll out to one server, then the fleet.

**Why →** Every action gets tied to a named user in the logs.

**Interview tip →** Admit deny lists in sudo are weak; allow lists are the real control.

</details>

### Q12. A developer reports that their newly added group membership is not working even though `id alice` shows it. What do you explain and do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Group lists are loaded at login, so existing sessions don't see the change.

**What I'd do →**
1. Confirm: `id alice` (reads the database) vs `id` in their session (current process).
2. Have them log out and back in, or run `newgrp groupname` for a subshell.
3. For services running as that user, restart the service.

**Why →** The kernel stores supplementary groups in the process credentials when the session starts.

**Interview tip →** This is also why `systemctl restart` is needed after adding a service user to a group.

</details>

### Q13. You must give the monitoring agent permission to read every process's `/proc/<pid>/` info and bind to port 162 without running as root. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use Linux capabilities instead of root.

**What I'd do →**
1. In the systemd unit:
```ini
[Service]
User=monagent
AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
NoNewPrivileges=yes
```
2. `systemctl daemon-reload && systemctl restart monagent`.
3. Verify: `grep Cap /proc/$(pidof monagent)/status | xargs` and decode with `capsh --decode=...`.

**Why →** Capabilities split root into small privileges; the bounding set guarantees the process can never gain more.

**Interview tip →** Prefer `AmbientCapabilities` in systemd over `setcap` on the binary, because file caps get lost on package upgrades.

</details>

### Q14. You need a shared admin account for an emergency, but auditors forbid shared passwords. What do you set up? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Break-glass access that is controlled and audited.

**What I'd do →**
1. Create one local `breakglass` user per server with a long random password stored in a vault (HashiCorp Vault, 1Password) that requires approval to check out.
2. Rotate the password automatically after every checkout.
3. Disable its SSH key login; allow console or a dedicated bastion only.
4. Alert on any login: an auditd rule or a PAM `pam_exec` hook that notifies the SIEM.
5. Test it quarterly.

**Why →** You still have a way in when LDAP/SSO is down, while every use is traceable to a person.

**Interview tip →** Mention that SSH certificates with short TTLs from Vault are a modern alternative.

</details>

### Q15. A user's home directory is on a server with quotas, and one person filled the shared `/home`. How do you stop this happening again? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Enforce per-user disk quotas.

**What I'd do →**
1. For XFS, mount with `uquota` in `/etc/fstab` (`/dev/vg0/home /home xfs defaults,uquota 0 0`) and remount/reboot.
2. Set limits:
```bash
sudo xfs_quota -x -c 'limit bsoft=9g bhard=10g alice' /home
sudo xfs_quota -x -c 'report -h' /home
```
3. For ext4 use `usrquota`, then `quotacheck -cum /home`, `quotaon /home`, `setquota -u alice 9G 10G 0 0 /home`.

**Why →** Hard limits stop a single user from starving everyone; soft limits give a warning first.

**Interview tip →** XFS quota must be enabled at mount time; `remount` is not enough for the root filesystem.

</details>

### Q16. How would you make sure a user can only log in during business hours from the office network? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PAM access controls by time and origin.

**What I'd do →**
1. Time: add `account required pam_time.so` to `/etc/pam.d/sshd`, and in `/etc/security/time.conf`:
```
sshd;*;alice;Wk0900-1800
```
2. Origin: add `account required pam_access.so` and in `/etc/security/access.conf`:
```
+ : alice : 10.20.0.0/16
- : alice : ALL
```
3. Back it up with `Match User alice` + `AllowUsers alice@10.20.*` in sshd for defence in depth.

**Why →** PAM account modules decide whether an authenticated user may log in.

**Interview tip →** Always test PAM changes with a second root session open.

</details>

### Q17. A team wants a shared Git deploy user on the server, but you need to know who deployed. How do you design it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared account, individual identity.

**What I'd do →**
1. Put each person's key in `~deploy/.ssh/authorized_keys` with a forced command and env tag:
```
command="/usr/local/bin/deploy-wrapper alice",no-port-forwarding,no-pty ssh-ed25519 AAAA... alice
```
2. The wrapper logs `alice` plus `$SSH_ORIGINAL_COMMAND` to syslog and runs only allowed actions.
3. sshd also logs the key fingerprint per login (`LogLevel VERBOSE`).

**Why →** You keep a single deploy account but every action is tied to a key owner.

**Interview tip →** Better long-term answer: deploy from CI with a dedicated identity, not humans over SSH.

</details>

### Q18. The security team wants all interactive shell sessions to time out after 15 minutes idle. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Idle shell and SSH timeouts.

**What I'd do →**
1. Shell: create `/etc/profile.d/tmout.sh`:
```bash
TMOUT=900
readonly TMOUT
export TMOUT
```
2. SSH: in `sshd_config` set `ClientAliveInterval 300` and `ClientAliveCountMax 3`, then reload.

**Why →** `TMOUT` closes idle bash shells; `readonly` stops users from unsetting it. Client-alive settings drop dead SSH connections.

**Interview tip →** Note `ClientAlive*` detects unresponsive clients, not idle users; `TMOUT` handles idleness.

</details>

### Q19. You suspect someone added an extra UID 0 account. How do you check and respond? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hidden root accounts are a classic backdoor.

**What I'd do →**
```bash
awk -F: '$3==0 {print $1}' /etc/passwd
awk -F: '$2=="" {print $1}' /etc/shadow      # empty passwords
ls -l /etc/passwd /etc/shadow; rpm -V setup 2>/dev/null || debsums -c 2>/dev/null
lastlog | head; last -F | head
```
If anything besides `root` shows UID 0, treat the host as compromised: isolate, preserve evidence, and investigate rather than just deleting the line.

**Why →** An attacker with root can add accounts; removing the symptom leaves the way they got in open.

**Interview tip →** Mention monitoring `/etc/passwd` with auditd (`-w /etc/passwd -p wa -k identity`) or file integrity tools like AIDE.

</details>

### Q20. A developer accidentally ran `chmod -R 777 /var/www`. How do you restore sane permissions? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reset directories and files separately.

**What I'd do →**
```bash
sudo chown -R www-data:www-data /var/www
sudo find /var/www -type d -exec chmod 755 {} +
sudo find /var/www -type f -exec chmod 644 {} +
```
Then re-apply exceptions such as writable upload dirs (`775`) or executable scripts, and check config files holding secrets (`640`).

**Why →** Directories need `x` to be entered; files usually should not be executable.

**Interview tip →** If the files came from a package, `rpm --setperms pkg` or reinstalling restores the original modes.

</details>

### Q21. You need to rename a user from `jsmith` to `john.smith` on a server without breaking their files. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Change login name and home path, keep UID.

**What I'd do →**
1. Make sure they are logged out: `pkill -u jsmith`.
2. Run:
```bash
sudo usermod -l john.smith -d /home/john.smith -m jsmith
sudo groupmod -n john.smith jsmith
```
3. Check crontabs (`/var/spool/cron/jsmith` is renamed by some distros, not all), sudoers entries and any config referencing the old name.

**Why →** Files are owned by UID, not name, so keeping the UID means ownership stays correct.

**Interview tip →** Mention that NFS-shared homes need the same change on every host or the UID mapping breaks.

</details>

### Q22. A new compliance rule says no user may reuse their last five passwords and passwords need 14+ characters. How do you enforce it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PAM password quality and history.

**What I'd do →**
1. In `/etc/security/pwquality.conf`: `minlen = 14`, `minclass = 3`, `dictcheck = 1`.
2. For history, use `pam_pwhistory`. On RHEL 9 use authselect: `authselect enable-feature with-pwhistory` and set `remember = 5` in `/etc/security/pwhistory.conf`.
3. On Debian/Ubuntu add to `/etc/pam.d/common-password`: `password required pam_pwhistory.so remember=5 use_authtok` before `pam_unix`.
4. Test with `passwd` as a normal user.

**Why →** pwquality checks strength at change time; pwhistory stores previous hashes in `/etc/security/opasswd`.

**Interview tip →** On RHEL, editing `/etc/pam.d/system-auth` directly gets overwritten by authselect.

</details>

### Q23. You have to give a CI runner user the right to run Docker but you are worried about security. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Membership in the `docker` group is root-equivalent.

**What I'd do →**
1. Prefer rootless Docker or Podman for the runner user: `dockerd-rootless-setuptool.sh install`.
2. If rootful Docker is required, run the runner on a dedicated, disposable VM that has no other secrets.
3. Never mount `/var/run/docker.sock` into untrusted job containers.
4. Consider building with BuildKit rootless or Kaniko/Buildah.

**Why →** Anyone in `docker` can run `docker run -v /:/host --privileged` and own the host.

**Interview tip →** Say clearly "docker group = root", interviewers look for that.

</details>

### Q24. A team needs some files to be impossible to delete even by root accidentally, such as a critical config. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Extended file attributes.

**What I'd do →**
```bash
sudo chattr +i /etc/resolv.conf     # immutable
sudo chattr +a /var/log/audit.trail # append-only
lsattr /etc/resolv.conf
```
Remove with `chattr -i` before planned changes.

**Why →** The immutable flag blocks write, rename, delete and link even for root, until someone removes it with `CAP_LINUX_IMMUTABLE`.

**Interview tip →** It protects against mistakes and tools rewriting files, not against an attacker with root, who can just run `chattr -i`.

</details>

### Q25. How would you design user and access management for a 500-server fleet with no shared passwords and fast offboarding? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scale, auditability and instant revocation.

**What I'd do →**
1. Identity from the IdP (Okta/Entra) synced to FreeIPA/AD or used directly by an access proxy.
2. SSH with short-lived certificates (Vault SSH CA, Teleport, or `step-ca`): servers trust `TrustedUserCAKeys`, certs last 8 hours.
3. Role-based principals mapped to sudo rules (`AuthorizedPrincipalsFile`).
4. All access through a bastion or proxy with session recording.
5. Only one break-glass local account, vaulted.

**Why →** Disable the user in the IdP and they lose access everywhere within hours, with no key cleanup.

**Interview tip →** Contrast it with `authorized_keys` sprawl, which is the failure mode most companies are trying to escape.

</details>

## Storage, disks and filesystems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. You get paged at 3 AM: the root filesystem on a production server is at 98% and climbing. Walk through your response. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stop the bleeding first, then find the root cause in daylight.

**What I'd do →**
1. Find what is growing:
```bash
df -h /
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail
sudo du -xh --max-depth=1 /var/log | sort -h | tail
```
2. Quick safe wins: `journalctl --vacuum-size=500M`, remove rotated logs (`*.gz`), `apt-get clean` / `dnf clean all`.
3. If a single log is huge and still open, truncate it rather than delete: `sudo truncate -s 0 /var/log/app/app.log`.
4. Once stable, fix the cause (missing logrotate, debug logging left on, runaway dump files).

**Why →** `du -x` stays on one filesystem so mounted volumes don't confuse you, and truncating keeps the app's file handle valid.

**Interview tip →** Say "I buy time first, then write a ticket for the root cause and an alert at 80%."

</details>

### Q27. A new 200 GB EBS volume is attached to an EC2 instance. How do you make it usable and persistent across reboots? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partition (optional), format, mount, fstab by UUID.

**What I'd do →**
```bash
lsblk                                  # find it, e.g. /dev/nvme1n1
sudo mkfs.xfs /dev/nvme1n1
sudo mkdir -p /data
sudo blkid /dev/nvme1n1                # copy the UUID
echo 'UUID=<uuid> /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a && df -h /data
```

**Why →** Device names like `/dev/nvme1n1` can change between boots; UUIDs don't. `nofail` keeps the instance booting if the volume is detached.

**Interview tip →** Always run `mount -a` after editing fstab, before rebooting, to catch mistakes.

</details>

### Q28. The `/var` LVM logical volume is nearly full and there is free space in the volume group. How do you grow it online? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** LVM lets you grow without downtime.

**What I'd do →**
```bash
sudo vgs                                   # confirm VFree
sudo lvextend -r -L +20G /dev/vg0/var      # -r resizes the filesystem too
df -h /var
```
If you prefer separate steps: `xfs_growfs /var` for XFS, `resize2fs /dev/vg0/var` for ext4.

**Why →** Both XFS and ext4 can grow while mounted.

**Interview tip →** Mention that XFS can't shrink, so you grow in sensible increments rather than all at once.

</details>

### Q29. The volume group has no free space. You've been given a new disk. How do you add it to grow `/var`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Add a physical volume to the VG, then extend.

**What I'd do →**
```bash
lsblk                                  # new disk /dev/sdc
sudo pvcreate /dev/sdc
sudo vgextend vg0 /dev/sdc
sudo lvextend -r -l +100%FREE /dev/vg0/var
sudo pvs; sudo vgs; df -h /var
```

**Why →** The LV spans both PVs transparently.

**Interview tip →** Flag the risk: a linear LV across two disks fails if either disk fails, so on bare metal put RAID underneath.

</details>

### Q30. A cloud VM's root disk was enlarged from 30 GB to 80 GB in the console, but `df` still shows 30 GB. What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The block device grew, but the partition and filesystem did not.

**What I'd do →**
```bash
lsblk                                  # disk 80G, partition still 30G
sudo growpart /dev/nvme0n1 1           # from cloud-guest-utils / cloud-utils-growpart
sudo xfs_growfs /                      # XFS
# or: sudo resize2fs /dev/nvme0n1p1    # ext4
df -h /
```
If root is on LVM: `growpart`, then `pvresize /dev/nvme0n1p3`, then `lvextend -r`.

**Why →** Three layers must each be resized: disk, partition, filesystem.

**Interview tip →** Note that cloud-init's `growpart` module does this automatically at boot on many images.

</details>

### Q31. You need to move a busy MySQL data directory from the root disk to a new, faster volume. Plan it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Short downtime is unavoidable for a consistent copy unless you replicate.

**What I'd do →**
1. Pre-sync while running: `rsync -aHAX /var/lib/mysql/ /mnt/newvol/mysql/`.
2. Stop MySQL, run a final `rsync -aHAX --delete` (fast, only changes).
3. Mount the new volume at `/var/lib/mysql` via fstab (better than changing `datadir`).
4. Fix SELinux labels if enforcing: `restorecon -Rv /var/lib/mysql`.
5. Start MySQL, check the error log and run a test query. Keep the old copy for a few days.

**Why →** Two-pass rsync keeps downtime to the size of the delta, and mounting at the original path avoids config and SELinux changes.

**Interview tip →** For zero downtime, mention building a replica on the new volume and failing over.

</details>

### Q32. A team wants a shared directory mounted on 10 app servers. How do you set up NFS? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** NFS server export plus client mounts.

**What I'd do →**
1. Server:
```bash
sudo dnf install -y nfs-utils    # or apt install nfs-kernel-server
sudo mkdir -p /exports/shared
echo '/exports/shared 10.0.1.0/24(rw,sync,root_squash)' | sudo tee -a /etc/exports
sudo systemctl enable --now nfs-server
sudo exportfs -rav
```
2. Open firewall for NFSv4 (TCP 2049).
3. Clients: `10.0.1.10:/exports/shared /mnt/shared nfs4 defaults,_netdev,nofail 0 0` in fstab, then `mount -a`.

**Why →** `root_squash` maps client root to `nobody`, so root on a client can't own the share. `_netdev` waits for the network at boot.

**Interview tip →** Say UIDs must match across clients (or use NFSv4 idmapping) otherwise ownership looks wrong.

</details>

### Q33. An NFS server will be down for maintenance and clients must not hang. How do you prepare the clients? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hard mounts block processes in D state until the server returns.

**What I'd do →**
1. Before maintenance, stop the apps that use the mount and `umount /mnt/shared` on clients.
2. Long term, use autofs so the mount happens on access and expires when idle:
```
# /etc/auto.master.d/shared.autofs
/mnt  /etc/auto.shared  --timeout=300
# /etc/auto.shared
shared  -fstype=nfs4,rw  10.0.1.10:/exports/shared
```
3. Only use `soft,timeo=` for read-only, non-critical data, because soft mounts can corrupt writes.

**Why →** Hard mounts protect data integrity; the price is hanging processes when the server is gone.

**Interview tip →** Showing you know that `soft` is a data-integrity trade-off scores points.

</details>

### Q34. You need to free a disk that is mounted, but `umount` says "target is busy". What do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some process has a file open or a cwd on that mount.

**What I'd do →**
```bash
sudo fuser -vm /data
sudo lsof +f -- /data
```
Stop those services cleanly (or ask users to `cd` out), then `umount /data`. Last resort: `umount -l /data` (lazy) and make sure nothing writes afterwards.

**Why →** Killing a database to unmount is worse than stopping it properly.

**Interview tip →** Mention that a shell sitting in the directory, often your own, is the classic culprit.

</details>

### Q35. A batch job writes millions of tiny files per day. Which filesystem and options would you pick? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inode density and metadata performance matter more than throughput.

**What I'd do →**
1. Prefer XFS: inodes are allocated dynamically, so you don't run out early.
2. If ext4 is required, format with more inodes: `mkfs.ext4 -i 4096 /dev/sdb` or `-N <count>`.
3. Mount with `noatime` to avoid a metadata write on every read.
4. Use a directory hashing scheme (`ab/cd/abcdef...`) so no single directory holds millions of entries.
5. Reconsider the design: object storage or packing files into archives is often better.

**Why →** ext4 fixes the inode count at format time; tiny files exhaust inodes before blocks.

**Interview tip →** Show you'd monitor `df -i` alongside `df -h`.

</details>

### Q36. You must take consistent backups of a filesystem that is constantly being written to. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use point-in-time snapshots.

**What I'd do →**
1. On LVM:
```bash
sudo lvcreate -s -n data_snap -L 10G /dev/vg0/data
sudo mount -o ro,nouuid /dev/vg0/data_snap /mnt/snap   # nouuid needed for XFS
sudo tar -czf /backup/data-$(date +%F).tar.gz -C /mnt/snap .
sudo umount /mnt/snap && sudo lvremove -y /dev/vg0/data_snap
```
2. For databases, flush or use the DB's own tool first (`mysqldump --single-transaction`, `pg_basebackup`), or `fsfreeze` around the snapshot.
3. In the cloud, use EBS snapshots with application-consistent hooks.

**Why →** A snapshot gives a frozen view; copying a live directory produces a torn backup.

**Interview tip →** Mention that an LVM snapshot becomes invalid if its COW space fills, so size it for the change rate.

</details>

### Q37. A developer asks for a 4 GB swap on a server with no free partitions. How do you add it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a swap file.

**What I'd do →**
```bash
sudo fallocate -l 4G /swapfile        # on XFS/ext4; use dd on btrfs or if fallocate fails
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
swapon --show; free -h
```

**Why →** `chmod 600` stops users reading memory pages written to swap.

**Interview tip →** Mention `vm.swappiness` and that some workloads (Kubernetes nodes historically, latency-sensitive DBs) avoid swap on purpose.

</details>

### Q38. You need software RAID 1 across two disks on a bare-metal server. How do you build and monitor it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** mdadm mirror.

**What I'd do →**
```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat
sudo mkfs.xfs /dev/md0
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf   # /etc/mdadm/mdadm.conf on Debian
sudo dracut -f      # or update-initramfs -u
```
Enable alerts: set `MAILADDR ops@example.com` in mdadm.conf and run `mdmonitor`.

**Why →** Saving the array config and rebuilding initramfs makes the array assemble at boot with the same name.

**Interview tip →** "RAID is not a backup" is the line interviewers want to hear.

</details>

### Q39. A disk in your RAID 1 array failed. How do you replace it without downtime? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Degraded mirror; swap and resync.

**What I'd do →**
```bash
cat /proc/mdstat                         # [U_] means degraded
sudo mdadm /dev/md0 --fail /dev/sdc --remove /dev/sdc
# physically replace the disk (hot-swap bay)
sudo sfdisk -d /dev/sdb | sudo sfdisk /dev/sdc   # copy partition table if partitioned
sudo mdadm /dev/md0 --add /dev/sdc
watch cat /proc/mdstat
```
If the array is also the boot disk, reinstall the bootloader on the new disk.

**Why →** The array keeps serving from the healthy disk while it rebuilds.

**Interview tip →** Check `smartctl -a` on the surviving disk; a second failure during rebuild is the real risk.

</details>

### Q40. Your application logs fill disks on 100 servers. Design a log retention approach. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Local retention short, central retention long.

**What I'd do →**
1. logrotate config per app:
```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 500M
}
```
2. Cap journald: `SystemMaxUse=1G` in `/etc/systemd/journald.conf`.
3. Ship logs to a central system (Loki, Elasticsearch, CloudWatch) with an agent like Fluent Bit or Vector.
4. Put `/var/log` on its own LV so it can't fill root.

**Why →** Local disks are for a few days of debugging; central storage gives search and long retention.

**Interview tip →** Explain `copytruncate` vs signalling the app (`postrotate`) and when each is safer.

</details>

### Q41. You need to encrypt a data disk on a Linux server holding customer data. How do you do it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** LUKS encryption at rest, with a plan for unlocking at boot.

**What I'd do →**
```bash
sudo cryptsetup luksFormat --type luks2 /dev/sdb
sudo cryptsetup open /dev/sdb securedata
sudo mkfs.xfs /dev/mapper/securedata
```
1. Add to `/etc/crypttab`: `securedata UUID=<luks-uuid> none luks` and mount via fstab.
2. For unattended boot, bind to TPM2 (`systemd-cryptenroll --tpm2-device=auto /dev/sdb`) or network-bound Clevis/Tang.
3. Back up the LUKS header: `cryptsetup luksHeaderBackup /dev/sdb --header-backup-file hdr.img`.

**Why →** If the header is lost, the data is gone forever; TPM/Tang avoids a human typing a passphrase.

**Interview tip →** In the cloud, mention provider-managed encryption (EBS + KMS) as usually enough unless policy requires OS-level keys.

</details>

### Q42. A team wants a filesystem mounted at `/tmp` that clears on reboot and can't run binaries. How do you set it up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** tmpfs with hardened mount options.

**What I'd do →**
1. `sudo systemctl enable tmp.mount` on distros that ship it, or add to fstab:
```
tmpfs /tmp tmpfs defaults,size=2G,nodev,nosuid,noexec,mode=1777 0 0
```
2. `sudo mount -o remount /tmp` (or reboot) and verify with `findmnt /tmp`.
3. Test installers and tools that unpack to `/tmp` and run from there; set `TMPDIR` for them if `noexec` breaks them.

**Why →** `noexec` blocks a common attacker trick of dropping binaries in `/tmp`; `1777` keeps the sticky bit so users can't delete each other's files.

**Interview tip →** Mention the sticky bit by name.

</details>

### Q43. Your company wants nightly backups of `/etc`, `/home` and `/var/www` to a remote server, retained for 30 days. Design it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deduplicated, encrypted, versioned backups.

**What I'd do →**
1. Use restic or borg rather than plain tar:
```bash
export RESTIC_REPOSITORY=sftp:backup@backup01:/srv/restic/web01
export RESTIC_PASSWORD_FILE=/root/.restic-pass
restic backup /etc /home /var/www --exclude-caches
restic forget --keep-daily 30 --prune
```
2. Run from a systemd timer, log to the journal, alert on failure.
3. Do a monthly restore test to a scratch directory.

**Why →** Dedup keeps storage small, encryption protects the remote copy, and `forget` handles retention.

**Interview tip →** "Backups you haven't restored are hopes, not backups."

</details>

### Q44. A disk is showing reallocated sectors in SMART. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The disk is failing; plan replacement before it dies.

**What I'd do →**
```bash
sudo smartctl -a /dev/sda | grep -Ei 'Reallocated|Pending|Uncorrectable|overall'
sudo smartctl -t long /dev/sda
```
1. Confirm backups are current and restorable.
2. If in RAID, schedule a disk swap. If not, migrate data now.
3. Watch `dmesg` for I/O errors.

**Why →** A rising reallocated or pending count predicts failure.

**Interview tip →** Mention running `smartd` so this alert arrives automatically, not when you happen to look.

</details>

### Q45. You need to check and repair an ext4 filesystem that reported errors. How do you do it safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Never run fsck on a mounted read-write filesystem.

**What I'd do →**
1. Unmount the filesystem (for root, boot into rescue mode or schedule a check at boot).
2. Take a snapshot or image first if the data matters (`dd` or a cloud snapshot).
3. Run:
```bash
sudo e2fsck -f -y /dev/vg0/data
```
4. For XFS use `xfs_repair /dev/vg0/data` (use `-n` first for a dry run).
5. Check `lost+found` for recovered files afterwards.

**Why →** Repairing a mounted filesystem can corrupt it further.

**Interview tip →** Mention that `tune2fs -l` shows the last check time and mount count.

</details>

### Q46. A partner sends you an ISO image you need to inspect. How do you look at its contents without burning it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Loop mount.

**What I'd do →**
```bash
sudo mkdir -p /mnt/iso
sudo mount -o loop,ro image.iso /mnt/iso
ls /mnt/iso
sudo umount /mnt/iso
```

**Why →** The loop device presents a file as a block device; `ro` avoids accidental modification.

**Interview tip →** `losetup -a` lists active loop devices, which helps clean up leftovers.

</details>

### Q47. A server needs a large shared scratch space across two disks for speed, and data loss is acceptable. What do you build? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Striping for throughput, no redundancy.

**What I'd do →**
```bash
sudo pvcreate /dev/sdb /dev/sdc
sudo vgcreate scratch /dev/sdb /dev/sdc
sudo lvcreate -i 2 -I 256k -l 100%FREE -n fast scratch
sudo mkfs.xfs /dev/scratch/fast
```
Mount with `noatime` and label it clearly as non-persistent.

**Why →** A striped LV (`-i 2`) spreads I/O over both disks; losing either disk loses everything, which is acceptable for scratch.

**Interview tip →** Compare with `mdadm --level=0`; both work, LVM is easier to resize.

</details>

### Q48. You must migrate a filesystem from ext4 to XFS on a live system. What is your plan? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** There is no in-place conversion; you copy data to a new filesystem.

**What I'd do →**
1. Create a new LV or disk and format XFS.
2. Pre-copy with `rsync -aHAXS --numeric-ids /data/ /newdata/`.
3. Stop writers, final rsync with `--delete`.
4. Swap mount points in fstab (by UUID) and mount.
5. Restore SELinux contexts, start services, verify, keep the old volume read-only for rollback.

**Why →** `-H` keeps hardlinks, `-A` ACLs, `-X` xattrs (SELinux labels), `-S` sparse files: all things plain `cp` can lose.

**Interview tip →** Mention that XFS can't shrink, so size it right.

</details>

### Q49. Users complain that `ls` in a directory with 3 million files takes forever. What would you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Huge flat directories are slow to list and sort.

**What I'd do →**
1. Short term: avoid sorting and stat: `ls -f` or `find . -maxdepth 1 -printf '%f\n' | head`.
2. Restructure into hashed subdirectories (`a1/b2/file`), done by the app.
3. Move processed files to dated archive dirs or object storage.
4. Make sure `dir_index` is on for ext4 (`tune2fs -l | grep features`).

**Why →** `ls` reads and sorts every entry and, with colour or `-l`, stats each one.

**Interview tip →** Mention that `rm *` fails with "Argument list too long"; use `find -delete`.

</details>

### Q50. How would you decide disk layout (partitions/LVs) for a new production Linux server? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Isolation, growth room and security options.

**What I'd do →**
1. Small `/boot` (1 GB) and `/boot/efi`.
2. LVM for everything else, leaving free space in the VG.
3. Separate LVs: `/`, `/var`, `/var/log`, `/var/log/audit`, `/home`, `/tmp`, and the app data volume.
4. Mount options: `nodev,nosuid,noexec` on `/tmp` and `/var/tmp`; `nodev` on `/home`.
5. In the cloud, keep data on separate volumes so instances can be rebuilt.

**Why →** A runaway log can't fill root, and CIS benchmarks expect these separations.

**Interview tip →** Mention leaving unallocated VG space so you can extend whichever LV grows.

</details>

## Processes, services and systemd

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Your team has a Python API that someone starts with `nohup python app.py &`. Make it a proper service running as a non-root user. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Turn an ad-hoc process into a managed, restartable systemd unit.

**What I'd do →**
1. `sudo useradd --system --shell /usr/sbin/nologin --home /opt/api api`
2. Create `/etc/systemd/system/api.service`:
```ini
[Unit]
Description=Orders API
After=network-online.target
Wants=network-online.target

[Service]
User=api
Group=api
WorkingDirectory=/opt/api
EnvironmentFile=/etc/api/api.env
ExecStart=/opt/api/venv/bin/gunicorn -b 0.0.0.0:8000 app:app
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
3. `sudo systemctl daemon-reload && sudo systemctl enable --now api`
4. Check `systemctl status api` and `journalctl -u api -f`.

**Why →** systemd restarts on crash, starts at boot, captures logs and runs it as a locked-down user.

**Interview tip →** Mention `Restart=on-failure` vs `always`, and that `ExecStart` needs an absolute path.

</details>

### Q52. The service you created must bind to port 443, but it runs as a non-root user. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ports below 1024 need `CAP_NET_BIND_SERVICE`.

**What I'd do →**
Add to the `[Service]` section:
```ini
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```
Then `daemon-reload` and restart. Alternatives: put nginx in front on 443 and proxy to a high port, or use socket activation.

**Why →** You grant one privilege instead of running the whole app as root.

**Interview tip →** Mention `sysctl net.ipv4.ip_unprivileged_port_start` exists but it is a host-wide change.

</details>

### Q53. You need to change one setting (memory limit) in a vendor-supplied unit file without editing it. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a drop-in override.

**What I'd do →**
```bash
sudo systemctl edit vendor-agent
```
Add:
```ini
[Service]
MemoryMax=512M
```
Save; systemd writes `/etc/systemd/system/vendor-agent.service.d/override.conf` and reloads. Restart the service and check with `systemctl show vendor-agent -p MemoryMax` or `systemctl cat vendor-agent`.

**Why →** Package upgrades overwrite `/usr/lib/systemd/system/*.service`, but never your drop-ins.

**Interview tip →** To clear a list setting like `ExecStart`, write an empty `ExecStart=` line first in the drop-in.

</details>

### Q54. A service depends on a database on the same host and fails at boot because the DB isn't ready. How do you fix the ordering? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordering vs requirement vs readiness.

**What I'd do →**
1. In the app unit:
```ini
[Unit]
Requires=postgresql.service
After=postgresql.service
```
2. If "started" isn't the same as "accepting connections", add a readiness check:
```ini
ExecStartPre=/usr/bin/pg_isready -h localhost -t 30
```
3. Make the app itself retry connections; that's the real fix.

**Why →** `After=` only orders; `Requires=`/`Wants=` pulls the unit in. Neither waits for the DB to be ready unless the DB unit uses `Type=notify`.

**Interview tip →** Explain the difference between `Wants` (soft) and `Requires` (hard).

</details>

### Q55. A service keeps crashing and systemd stops restarting it after a few attempts. How do you tune this? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Start rate limiting is kicking in.

**What I'd do →**
1. Look: `systemctl status app` shows "Start request repeated too quickly".
2. Find the cause in `journalctl -u app -b`.
3. Tune if appropriate:
```ini
[Unit]
StartLimitIntervalSec=300
StartLimitBurst=10

[Service]
Restart=on-failure
RestartSec=10
```
4. Reset after fixing: `systemctl reset-failed app`.

**Why →** Rate limiting prevents a crash loop from burning CPU. Tuning gives transient issues time to clear, but it doesn't hide real bugs.

**Interview tip →** Mention that `StartLimit*` belongs in `[Unit]` on current systemd.

</details>

### Q56. A long-running app must reload its config without dropping connections. How do you wire this into systemd? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reload instead of restart.

**What I'd do →**
```ini
[Service]
ExecStart=/usr/local/bin/myapp --config /etc/myapp.yml
ExecReload=/bin/kill -HUP $MAINPID
```
Then `systemctl reload myapp`. Validate the config first in `ExecReload` if the app supports it, for example nginx uses `nginx -t` before the signal.

**Why →** The app re-reads config on SIGHUP while keeping its sockets open.

**Interview tip →** `systemctl reload-or-restart` is handy in automation when you're not sure a unit supports reload.

</details>

### Q57. A script needs to run once at boot after networking, then exit. How do you do it with systemd? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A oneshot unit.

**What I'd do →**
```ini
[Unit]
Description=Register node at boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/register-node.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable it with `systemctl enable register-node`.

**Why →** `Type=oneshot` makes systemd wait for completion; `RemainAfterExit` keeps it "active" so dependents see it as done.

**Interview tip →** Prefer this over `/etc/rc.local`, which many distros no longer run by default.

</details>

### Q58. Ops wants to limit a noisy batch service to 2 CPUs and 4 GB RAM on a shared host. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** cgroup v2 limits via systemd.

**What I'd do →**
```bash
sudo systemctl set-property batch.service CPUQuota=200% MemoryMax=4G MemoryHigh=3.5G
systemctl show batch -p CPUQuotaPerSecUSec -p MemoryMax
systemd-cgtop
```
For ad-hoc jobs: `systemd-run --scope -p CPUQuota=100% -p MemoryMax=2G ./job.sh`.

**Why →** `CPUQuota=200%` equals two full CPUs; `MemoryHigh` throttles before `MemoryMax` triggers the OOM killer.

**Interview tip →** `set-property` persists by writing a drop-in; add `--runtime` for a temporary change.

</details>

### Q59. A security review says your services run with too many privileges. Which systemd hardening options would you add? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sandbox the service using systemd's built-in options.

**What I'd do →**
```ini
[Service]
User=app
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/app /var/log/app
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
CapabilityBoundingSet=
```
Score it with `systemd-analyze security app.service` and iterate until the app still works.

**Why →** If the app is exploited, the attacker lands in a read-only, capability-free box.

**Interview tip →** Mention `systemd-analyze security` by name; it shows you've done this for real.

</details>

### Q60. You need to run a nightly job as a specific user only while they're not logged in, and it should start even if they never log in. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** User-level systemd services need lingering.

**What I'd do →**
```bash
sudo loginctl enable-linger builder
sudo -iu builder
mkdir -p ~/.config/systemd/user
# create job.service and job.timer there
systemctl --user daemon-reload
systemctl --user enable --now job.timer
```

**Why →** Without lingering, the user's systemd manager only runs while they have a session, so user units stop at logout.

**Interview tip →** This is also how rootless Podman containers keep running after logout.

</details>

### Q61. An app forks into a background daemon itself. systemd thinks it died right after start. How do you configure the unit? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Wrong service `Type`.

**What I'd do →**
1. Best: run the app in the foreground (most have a `--foreground` or `-D` flag) and keep `Type=simple` or `exec`.
2. If it must fork:
```ini
[Service]
Type=forking
PIDFile=/run/myapp/myapp.pid
ExecStart=/usr/sbin/myapp -d
```

**Why →** With `Type=simple`, the parent exits after forking and systemd considers the service stopped and may kill the child.

**Interview tip →** Mention `Type=notify` for apps that signal readiness with `sd_notify`.

</details>

### Q62. You need to stop a service from being started by anyone, even as a dependency, during an investigation. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `disable` isn't enough; use `mask`.

**What I'd do →**
```bash
sudo systemctl stop cups
sudo systemctl mask cups
systemctl status cups     # shows "masked"
# later
sudo systemctl unmask cups
```

**Why →** Mask links the unit to `/dev/null`, so neither manual starts nor dependencies can activate it. `disable` only removes boot-time links.

**Interview tip →** Remember to mask the `.socket` too for socket-activated services.

</details>

### Q63. A process is using 100% CPU on a production box. What's your approach before killing it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Understand it before you kill it; it might be important.

**What I'd do →**
```bash
top -c                       # or htop; find PID and command
ps -o pid,ppid,user,etime,cmd -p <PID>
sudo ls -l /proc/<PID>/cwd /proc/<PID>/exe
sudo strace -p <PID> -f -c   # syscalls summary, Ctrl+C after a few seconds
```
If it's legitimate but should be less greedy: `renice +10 -p <PID>`. If runaway: `kill -TERM`, then `kill -KILL` if it ignores TERM.

**Why →** SIGTERM lets the app clean up; SIGKILL can leave locks and half-written files.

**Interview tip →** Grab a thread dump (`jstack`, `py-spy dump`) before killing for the postmortem.

</details>

### Q64. A legacy app has no systemd unit and must survive SSH logout while you build one. What quick options do you have? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Detach the process from the terminal for now.

**What I'd do →**
1. `tmux new -s legacy` then run it; detach with `Ctrl+b d`.
2. Or a transient systemd unit, which is better:
```bash
sudo systemd-run --unit=legacy-app --uid=appuser /opt/legacy/run.sh
journalctl -u legacy-app -f
```
3. `nohup ./run.sh > out.log 2>&1 &` as a last resort.

**Why →** `systemd-run` gives logging and control (`systemctl stop legacy-app`) without writing a file.

**Interview tip →** Mention SIGHUP: closing the terminal sends it, which is why plain `&` processes die.

</details>

### Q65. You need two instances of the same worker service with different config on one host. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** systemd template units.

**What I'd do →**
Create `/etc/systemd/system/worker@.service`:
```ini
[Service]
User=worker
EnvironmentFile=/etc/worker/%i.env
ExecStart=/opt/worker/bin/worker --queue %i
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Then:
```bash
sudo systemctl enable --now worker@emails worker@reports
systemctl status 'worker@*'
```

**Why →** `%i` is replaced with the instance name, so one file serves any number of instances.

**Interview tip →** Templates are also how `getty@tty1` and `openvpn@client` work.

</details>

### Q66. A service should only start when a request arrives on its port, to save memory. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Socket activation.

**What I'd do →**
1. `/etc/systemd/system/tool.socket`:
```ini
[Socket]
ListenStream=9000

[Install]
WantedBy=sockets.target
```
2. `tool.service` with `ExecStart=/usr/local/bin/tool` and the app reading the inherited socket (`LISTEN_FDS`).
3. `systemctl enable --now tool.socket`.

**Why →** systemd holds the port; the first connection starts the service, and no connection is lost during startup.

**Interview tip →** The app must support receiving the socket from systemd; not every app does.

</details>

### Q67. Developers want to see the logs of only their service from the last hour, without sudo. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** journald access for non-root users.

**What I'd do →**
```bash
sudo usermod -aG systemd-journal alice
# then alice runs:
journalctl -u api --since "1 hour ago" --no-pager
journalctl -u api -p err -f
```

**Why →** Members of `systemd-journal` (and `adm` on Debian/Ubuntu) can read the system journal.

**Interview tip →** Mention `--since`, `-p` priority and `-o json` for scripts.

</details>

### Q68. The journal is lost on every reboot and you need logs from before a crash. What do you change? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** journald is in volatile mode.

**What I'd do →**
```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo sed -i 's/^#\?Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
journalctl --list-boots
```
Afterwards use `journalctl -b -1 -e` for the previous boot.

**Why →** With `Storage=auto`, journald persists only if `/var/log/journal` exists.

**Interview tip →** Pair it with `SystemMaxUse=` so persistent logs don't fill the disk.

</details>

### Q69. A Java service takes 3 minutes to shut down and systemd kills it after 90 seconds, corrupting state. Fix it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default stop timeout is too short.

**What I'd do →**
```ini
[Service]
TimeoutStopSec=300
KillSignal=SIGTERM
KillMode=mixed
```
Then investigate why shutdown takes 3 minutes (draining connections, flushing caches).

**Why →** systemd sends `KillSignal`, waits `TimeoutStopSec`, then sends SIGKILL. `KillMode=mixed` sends SIGTERM only to the main process and SIGKILL to leftovers at the end.

**Interview tip →** Mention that `DefaultTimeoutStopSec` is set globally in `/etc/systemd/system.conf`.

</details>

### Q70. You need to temporarily run a command with a lower I/O priority so a backup doesn't hurt the database. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CPU and I/O niceness, or cgroup I/O weights.

**What I'd do →**
```bash
nice -n 19 ionice -c3 tar -czf /backup/data.tgz /data
# or with cgroups:
sudo systemd-run --scope -p IOWeight=10 -p CPUWeight=10 tar -czf /backup/data.tgz /data
```

**Why →** `ionice -c3` (idle class) only gets disk time when nobody else wants it; cgroup weights work on cgroup v2 with any I/O scheduler that supports them.

**Interview tip →** Mention that `ionice` classes depend on the I/O scheduler; `IOWeight` is the more modern approach.

</details>

### Q71. A daemon writes a PID file and ops scripts use it, but after crashes the stale PID points to a different process. How do you fix the design? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** PID files are racy; let the supervisor track the process.

**What I'd do →**
1. Run the daemon in the foreground under systemd (`Type=simple`/`notify`), which tracks it by cgroup.
2. Scripts use `systemctl is-active app`, `systemctl kill app` or `systemctl show -p MainPID app`.
3. If PID files must stay, put them in `/run` (cleared at boot) via `RuntimeDirectory=app`.

**Why →** cgroup tracking can't confuse processes, and `/run` is a tmpfs, so stale files vanish on reboot.

**Interview tip →** "Let systemd own process lifecycle" is the senior answer.

</details>

### Q72. You have to schedule a maintenance reboot of a host but want running services to stop cleanly in the right order. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use systemd shutdown, not a hard reset.

**What I'd do →**
```bash
sudo shutdown -r 02:00 "Kernel patching"
# check / cancel
shutdown --show 2>/dev/null; sudo shutdown -c
```
Drain the host from load balancers first, and make sure units have correct `After=` ordering, since systemd stops units in reverse start order.

**Why →** A clean shutdown runs `ExecStop` and lets databases flush.

**Interview tip →** Mention draining traffic before rebooting; that's what separates ops from just commands.

</details>

### Q73. An app needs environment variables containing secrets. How do you give them to a systemd service safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Avoid secrets in unit files, which are world-readable.

**What I'd do →**
1. Better: use systemd credentials:
```ini
[Service]
LoadCredential=db_password:/etc/app/secrets/db_password
ExecStart=/opt/app/bin/app
```
The app reads `$CREDENTIALS_DIRECTORY/db_password`. With `systemd-creds encrypt`, use `LoadCredentialEncrypted=` (TPM-bound).
2. Acceptable: `EnvironmentFile=/etc/app/app.env` with `chmod 600 root:root`.
3. Best for fleets: fetch from Vault or a cloud secret manager at start.

**Why →** `Environment=` lines show up in `systemctl show`, and environment variables leak to child processes and `/proc/<pid>/environ`.

**Interview tip →** Mention that `systemctl show` reveals `Environment=` to any user.

</details>

### Q74. A cron-started process spawns children that keep running after the parent is killed. How do you clean this up properly? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Orphaned process trees; you need group or cgroup-wide termination.

**What I'd do →**
1. Now: find the process group/session and kill it:
```bash
ps -eo pid,pgid,sid,cmd | grep job
kill -TERM -- -<PGID>
```
2. Long term: run the job as a systemd service/timer, so `systemctl stop` kills the whole cgroup (`KillMode=control-group`, default).

**Why →** Signals sent to one PID don't reach its children; cgroups contain every descendant.

**Interview tip →** The minus sign in `kill -- -PGID` targets the process group, a detail interviewers like.

</details>

### Q75. You must find which service is listening on an unexpected port 8443 on a server. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Map port to process, then process to package or unit.

**What I'd do →**
```bash
sudo ss -tlnp 'sport = :8443'
sudo lsof -i :8443
systemctl status <PID>          # shows the owning unit
rpm -qf /proc/<PID>/exe 2>/dev/null || dpkg -S "$(readlink /proc/<PID>/exe)"
```

**Why →** `systemctl status <PID>` finds the unit from the cgroup, telling you who installed and runs it.

**Interview tip →** If the owner is unknown and not in any package, treat it as a possible compromise.

</details>

## Scheduling, automation and scripting

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. A database dump must run every night at 1:30 AM. How do you schedule it with cron so it is reliable? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cron entry plus everything cron doesn't give you: environment, logging, failure alerts.

**What I'd do →**
1. Write a script with `set -euo pipefail` and absolute paths.
2. Add a system cron file `/etc/cron.d/db-dump`:
```
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=ops@example.com
30 1 * * * postgres /usr/local/bin/db-dump.sh >> /var/log/db-dump.log 2>&1
```
3. Test by running the script as that user: `sudo -u postgres /usr/local/bin/db-dump.sh`.

**Why →** Cron runs with a minimal `PATH` and no login profile, which is the number-one reason jobs work by hand but fail in cron.

**Interview tip →** Mention `/etc/cron.d` files include a user column, unlike `crontab -e`.

</details>

### Q77. Convert that nightly cron job to a systemd timer. Why would you? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timers add logging, dependencies and catch-up runs.

**What I'd do →**
`/etc/systemd/system/db-dump.service`:
```ini
[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/db-dump.sh
```
`/etc/systemd/system/db-dump.timer`:
```ini
[Timer]
OnCalendar=*-*-* 01:30:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```
`systemctl enable --now db-dump.timer`, check with `systemctl list-timers`.

**Why →** Output goes to the journal, `Persistent=true` runs a missed job after downtime, and resource limits and `OnFailure=` alerts are available.

**Interview tip →** Validate calendar expressions with `systemd-analyze calendar '*-*-* 01:30:00'`.

</details>

### Q78. A cron job runs every 5 minutes, but sometimes takes 8 minutes and two copies overlap, corrupting data. How do you prevent that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need a lock so only one instance runs.

**What I'd do →**
```
*/5 * * * * app flock -n /run/lock/sync.lock /usr/local/bin/sync.sh
```
Or inside the script:
```bash
exec 9>/run/lock/sync.lock
flock -n 9 || { echo "already running"; exit 0; }
```

**Why →** `flock` uses a kernel lock that is released automatically when the process dies, unlike a hand-made lock file that goes stale.

**Interview tip →** A systemd timer also won't start a second copy while the service is still active.

</details>

### Q79. You have 300 servers running the same cron job at midnight, and they hammer a shared API. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Thundering herd; spread the start times.

**What I'd do →**
1. With systemd timers: `RandomizedDelaySec=1800` (and `FixedRandomDelay=true` for a stable per-host offset).
2. With cron, add a random sleep: `0 0 * * * sleep $((RANDOM % 1800)); /usr/local/bin/job.sh` (needs `SHELL=/bin/bash`).
3. Or derive the minute from the hostname in your config management (Ansible `random` filter with `seed=inventory_hostname`).

**Why →** Jitter turns a spike into a flat load.

**Interview tip →** Mention that `$RANDOM` is a bash feature and cron's default shell is `/bin/sh`.

</details>

### Q80. A job must run on the last day of every month. Cron has no "last day" field. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use a date check or a systemd calendar expression.

**What I'd do →**
- Cron: run on days 28–31 and check tomorrow's date:
```
0 23 28-31 * * [ "$(date -d tomorrow +\%d)" = "01" ] && /usr/local/bin/month-end.sh
```
- systemd timer: `OnCalendar=*-*~01 23:00:00` (the `~` means counting from the end of the month).

**Why →** `%` is special in crontab and must be escaped as `\%`.

**Interview tip →** The escaped `%` is a classic "why doesn't my cron work" gotcha.

</details>

### Q81. You inherited a server and need to know every scheduled job on it. Where do you look? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jobs hide in many places.

**What I'd do →**
```bash
for u in $(cut -d: -f1 /etc/passwd); do sudo crontab -l -u "$u" 2>/dev/null | sed "s/^/$u: /"; done
ls -l /etc/crontab /etc/cron.d/ /etc/cron.{hourly,daily,weekly,monthly}/
systemctl list-timers --all
atq
ls /var/spool/cron/ /var/spool/cron/crontabs/ 2>/dev/null
```

**Why →** User crontabs, system crontabs, run-parts directories, systemd timers and `at` jobs are all separate.

**Interview tip →** Mentioning systemd timers and `anacron` shows you know modern distros.

</details>

### Q82. A one-off data fix must run at 9 PM tonight, just once. How do you schedule it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One-shot scheduling.

**What I'd do →**
```bash
echo "/usr/local/bin/fix-data.sh >> /tmp/fix.log 2>&1" | at 21:00
atq
at -c <job>      # see the full job
atrm <job>       # cancel
```
Or with systemd: `systemd-run --on-calendar='2026-11-09 21:00' /usr/local/bin/fix-data.sh`.

**Why →** No need to add then remember to remove a cron line.

**Interview tip →** `atd` must be running; check `systemctl status atd`.

</details>

### Q83. A laptop-style VM is often off at night, so the daily cleanup job never runs. How do you make sure it runs once a day anyway? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Machines that aren't always on need catch-up scheduling.

**What I'd do →**
- Put the script in `/etc/cron.daily/`, which anacron runs once a day when the machine is up.
- Or use a systemd timer with `OnCalendar=daily` and `Persistent=true`.

**Why →** Plain cron only runs at the exact minute; if the host is off, the run is skipped.

**Interview tip →** `/var/spool/anacron/` holds the last-run timestamps.

</details>

### Q84. Write a safe bash script header for a production maintenance script. What goes in it and why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fail fast, clean up, log.

**What I'd do →**
```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
trap 'echo "ERROR line $LINENO: $BASH_COMMAND" >&2' ERR
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT
log() { logger -t maint "$*"; echo "$(date -Is) $*"; }
```

**Why →** `-e` exits on error, `-u` catches unset variables, `pipefail` catches failures inside pipes, the ERR trap tells you where it died, and the EXIT trap always cleans up temp files.

**Interview tip →** Mention `shellcheck` in CI for every script.

</details>

### Q85. A cleanup script must delete files older than 14 days in `/data/exports`, but someone once deleted `/` by mistake with a similar script. How do you write it safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Guard against empty variables and wrong paths.

**What I'd do →**
```bash
#!/usr/bin/env bash
set -euo pipefail
target="${EXPORT_DIR:?EXPORT_DIR not set}"
[[ "$target" == /data/exports* ]] || { echo "refusing: $target"; exit 1; }
find "$target" -xdev -type f -mtime +14 -print -delete
```
Run once with `-print` only (no `-delete`) to review.

**Why →** `${VAR:?}` aborts if empty, so `rm -rf "$VAR/"` can never become `rm -rf /`. `-xdev` stops `find` crossing into other mounts.

**Interview tip →** Mention the famous Steam bug (`rm -rf "$STEAMROOT/"*` with an empty variable).

</details>

### Q86. You need to run the same command on 40 servers right now and you don't have Ansible set up. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Parallel SSH, carefully.

**What I'd do →**
```bash
# hosts.txt has one host per line
parallel-ssh -h hosts.txt -i -t 30 'uptime'           # pssh package
# or plain bash + xargs
xargs -P 10 -I{} ssh -o BatchMode=yes -o ConnectTimeout=5 {} 'uptime' < hosts.txt
```
Run on one host first, then a small batch, then the rest.

**Why →** `BatchMode=yes` stops it hanging on password prompts; limiting parallelism avoids overloading shared services.

**Interview tip →** Say you'd move it into an Ansible ad-hoc command (`ansible all -m shell -a uptime`) as soon as possible.

</details>

### Q87. Your team writes shell scripts that parse `ls` output. Why is that a problem and what do you suggest? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Filenames can have spaces, newlines and globs.

**What I'd do →**
```bash
# bad
for f in $(ls /data); do ...; done
# good
for f in /data/*; do [[ -e "$f" ]] || continue; ...; done
# or for recursion
find /data -type f -print0 | while IFS= read -r -d '' f; do ...; done
```

**Why →** `ls` output is for humans; word-splitting breaks on spaces and newlines. Globs and `-print0` are safe.

**Interview tip →** Always quote variables: `"$f"`.

</details>

### Q88. A script must read a config file with `KEY=VALUE` lines, but you don't want to `source` it because it could contain commands. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Parse instead of executing.

**What I'd do →**
```bash
declare -A cfg
while IFS='=' read -r key value; do
  [[ "$key" =~ ^[A-Z_][A-Z0-9_]*$ ]] || continue
  cfg["$key"]="$value"
done < /etc/myapp/settings.conf
echo "${cfg[DB_HOST]}"
```

**Why →** `source` runs whatever is in the file with the script's privileges, which is dangerous if the file is writable by others.

**Interview tip →** Mention checking the file owner and permissions before trusting it.

</details>

### Q89. A cron job's output fills root's mailbox with thousands of messages. How do you handle cron output properly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cron mails any output; redirect deliberately.

**What I'd do →**
1. Send normal output to a log and only errors to mail:
```
MAILTO=ops@example.com
*/10 * * * * app /usr/local/bin/job.sh > /var/log/job.log 2>&1 || echo "job failed"
```
2. Or log to syslog: `job.sh 2>&1 | logger -t job`.
3. Clean the backlog: `> /var/spool/mail/root`.

**Why →** Silent success, loud failure is the right pattern.

**Interview tip →** Better still: report to a dead-man's switch (Healthchecks.io, Cronitor) so missed runs alert too.

</details>

### Q90. How would you get alerted if a critical nightly job silently stops running altogether? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failure alerts don't fire if the job never starts. You need a heartbeat.

**What I'd do →**
1. At the end of a successful run, ping a dead-man's switch:
```bash
/usr/local/bin/backup.sh && curl -fsS -m 10 --retry 3 https://hc.example.com/ping/<uuid>
```
2. Or push a metric `job_last_success_timestamp` to Prometheus Pushgateway / node_exporter textfile collector, and alert if `time() - job_last_success_timestamp > 90000`.
3. For systemd timers, add `OnFailure=notify@%n.service`.

**Why →** The absence of a signal is what you alert on.

**Interview tip →** "Alert on missing success, not just on failure" is a senior-level answer.

</details>

### Q91. A script must process a 50 GB log file line by line quickly. What tools would you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Avoid bash loops for big data; use stream tools.

**What I'd do →**
```bash
LC_ALL=C grep -F ' 500 ' access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head
# split into chunks and process in parallel
split -n l/8 access.log part_ && ls part_* | parallel -j8 "grep -c ERROR {}"
```

**Why →** `while read` in bash is very slow on large files; `grep`, `awk`, `sort` are optimised C. `LC_ALL=C` skips locale processing for big speedups.

**Interview tip →** Mention `zgrep`/`zcat` for compressed logs instead of unpacking.

</details>

### Q92. Your team wants every server to automatically install security updates. How do you set it up and stay safe? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automate patching without surprise outages.

**What I'd do →**
- Ubuntu/Debian: `apt install unattended-upgrades` and enable security origins in `/etc/apt/apt.conf.d/50unattended-upgrades`.
- RHEL/Rocky: `dnf install dnf-automatic`, set `upgrade_type = security` and `apply_updates = yes` in `/etc/dnf/automatic.conf`, then `systemctl enable --now dnf-automatic.timer`.
- Roll out in rings: canary hosts first, then the fleet a day later. Control reboots separately.

**Why →** Security-only updates have low regression risk; rings catch the rest.

**Interview tip →** Mention `needs-restarting -r` (RHEL) or `/var/run/reboot-required` (Ubuntu) for reboot detection.

</details>

### Q93. You must run a job only on the primary node of a two-node cluster, but both nodes have the same cron. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Make the job check leadership before acting.

**What I'd do →**
1. If there's a floating VIP (keepalived/Pacemaker), check it:
```bash
ip -o addr show | grep -q '10.0.0.100/' || exit 0
```
2. Or take a distributed lock (Consul `consul lock`, etcd lease, a DB advisory lock `SELECT pg_try_advisory_lock(42)`).
3. Better: make it a cluster resource in Pacemaker, or a Kubernetes CronJob with `concurrencyPolicy: Forbid`.

**Why →** Duplicated runs cause double emails, double billing and similar issues.

**Interview tip →** Mention making jobs idempotent so a double run is harmless anyway.

</details>

### Q94. A script you wrote works for you but fails when run by `sudo` with "command not found". Why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** sudo resets `PATH` to `secure_path`.

**What I'd do →**
```bash
sudo grep secure_path /etc/sudoers
sudo env | grep PATH
```
Use absolute paths in scripts, or install tools into a directory in `secure_path` (`/usr/local/bin`).

**Why →** `secure_path` stops a user-controlled `PATH` from tricking root into running a malicious binary.

**Interview tip →** Don't "fix" it with `sudo env PATH=$PATH`; that defeats the protection.

</details>

### Q95. Your team wants scripts that work on both Ubuntu and RHEL hosts. How do you detect the distro reliably? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use `/etc/os-release`, the standard.

**What I'd do →**
```bash
. /etc/os-release
case "$ID" in
  ubuntu|debian) pkg="apt-get install -y" ;;
  rhel|rocky|almalinux|centos|fedora) pkg="dnf install -y" ;;
  *) case "$ID_LIKE" in *rhel*|*fedora*) pkg="dnf install -y";; *debian*) pkg="apt-get install -y";; esac ;;
esac
```

**Why →** `/etc/os-release` exists on every systemd distro; `ID_LIKE` covers derivatives.

**Interview tip →** Say past a few branches you'd let Ansible facts (`ansible_os_family`) handle it.

</details>

### Q96. A cron job runs a script that calls `rsync` to another host over SSH, but in cron it hangs forever. What's going on and how do you design it properly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No SSH agent and no TTY in cron; SSH is waiting for a password or host key confirmation.

**What I'd do →**
1. Use a dedicated key without a passphrase, restricted on the remote side (`command=`, `from=`).
2. Pre-populate `known_hosts`.
3. Make SSH fail fast:
```bash
rsync -a -e "ssh -i /home/sync/.ssh/id_ed25519 -o BatchMode=yes -o ConnectTimeout=10" /data/ sync@backup:/data/
```
4. Wrap with `timeout 2h` so it can't hang forever.

**Why →** `BatchMode=yes` turns a hidden prompt into an immediate error you can see in logs.

**Interview tip →** Mention `timeout` as a general safety net for every scheduled job.

</details>

### Q97. You need to rotate a custom application's log every hour, not daily. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** logrotate runs daily by default; it needs an hourly trigger.

**What I'd do →**
1. Config `/etc/logrotate.hourly.d/app`:
```
/var/log/app/app.log {
    hourly
    rotate 48
    compress
    missingok
    postrotate
        systemctl kill -s HUP app.service
    endscript
}
```
2. A timer or `/etc/cron.hourly/logrotate-hourly` running `logrotate /etc/logrotate.hourly.d/app`.
3. Test with `logrotate -d` (debug) and `-f` (force).

**Why →** The `hourly` directive only works if logrotate itself is invoked every hour.

**Interview tip →** Mention that `postrotate` signalling is safer than `copytruncate`, which can lose lines.

</details>

### Q98. Ops asks for a script that waits until a remote service is up before continuing a deployment. How do you write it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Retry loop with a timeout.

**What I'd do →**
```bash
for i in $(seq 1 30); do
  if curl -fsS -o /dev/null --max-time 3 http://10.0.1.20:8080/healthz; then
    echo "up after $i tries"; exit 0
  fi
  sleep 5
done
echo "service not ready after 150s" >&2; exit 1
```

**Why →** A bounded loop avoids hanging a pipeline forever and gives a clear failure.

**Interview tip →** `timeout 150 bash -c 'until curl -fs ...; do sleep 5; done'` is a compact alternative.

</details>

### Q99. Your team keeps writing the same server bootstrap steps by hand. How would you automate first-boot configuration for new cloud VMs? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use cloud-init plus config management, and bake images.

**What I'd do →**
```yaml
#cloud-config
users:
  - name: ops
    groups: wheel
    ssh_authorized_keys: [ "ssh-ed25519 AAAA... ops" ]
package_update: true
packages: [chrony, auditd, amazon-cloudwatch-agent]
runcmd:
  - [ systemctl, enable, --now, chronyd ]
  - [ sh, -c, "curl -fsSL https://config.internal/bootstrap.sh | bash" ]
```
Bake the slow parts into a golden image with Packer; let cloud-init do only host-specific bits.

**Why →** Faster boot, consistent results, fewer moving parts at scale.

**Interview tip →** Check `cloud-init status --long` and `/var/log/cloud-init-output.log` when it goes wrong.

</details>

### Q100. A scheduled job must only run if the previous job in a chain succeeded. How do you build job dependencies on a single host? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Chain with systemd dependencies instead of guessing times.

**What I'd do →**
- `extract.service` (oneshot) with a timer.
- `transform.service` with:
```ini
[Unit]
Requires=extract.service
After=extract.service
```
- Or in `extract.service`: `ExecStartPost=/usr/bin/systemctl start --no-block transform.service` (only runs on success), plus `OnFailure=alert@%n.service`.

**Why →** Time-based chains ("extract at 1:00, transform at 2:00") break the first time extract runs long.

**Interview tip →** Above a few steps, say you'd move it to Airflow or a CI pipeline for visibility.

</details>

## Networking, SSH and remote access

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. A key belonging to a departed engineer may be in `authorized_keys` on 300 servers. How do you rotate SSH keys across the fleet? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Add new keys first, verify, then remove old ones, so nobody is locked out.

**What I'd do →**
1. Generate new keys: `ssh-keygen -t ed25519 -C "ops-2026-11" -f ~/.ssh/ops_2026`.
2. Push them with Ansible:
```yaml
- ansible.posix.authorized_key:
    user: ops
    key: "{{ lookup('file', 'keys/ops_2026.pub') }}"
    state: present
```
3. Test login with the new key on every host (`ansible all -m ping --private-key ~/.ssh/ops_2026`).
4. Remove old keys with `state: absent`, or manage the whole file with `exclusive: true`.
5. Grep the fleet for unknown keys and investigate.

**Why →** Add-verify-remove avoids lockouts; `exclusive` makes the file match your source of truth.

**Interview tip →** End with "and this is why we moved to SSH certificates with short TTLs".

</details>

### Q102. You need to reach a database on a private subnet from your laptop through a bastion. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSH jump host plus local port forwarding.

**What I'd do →**
```bash
ssh -J ops@bastion.example.com -L 5433:db.internal:5432 ops@app01.internal -N
psql -h localhost -p 5433 -U app
```
Or permanently in `~/.ssh/config`:
```
Host app01
    HostName app01.internal
    User ops
    ProxyJump bastion.example.com
    LocalForward 5433 db.internal:5432
```

**Why →** `-J`/`ProxyJump` tunnels the SSH connection through the bastion end-to-end; `-L` forwards a local port to the target.

**Interview tip →** Mention that in AWS you'd often use SSM Session Manager port forwarding instead of a bastion.

</details>

### Q103. Your security team wants SSH hardened on all servers. What changes do you make? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Disable weak paths, limit who can log in.

**What I'd do →**
`/etc/ssh/sshd_config.d/10-hardening.conf`:
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups ssh-users
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
```
Then `sudo sshd -t && sudo systemctl reload sshd`, and test from a second session before closing the first.

**Why →** Keys plus no root plus a group allowlist removes the most common brute-force and misuse paths.

**Interview tip →** Always keep an existing session open while changing sshd; that's the lockout insurance.

</details>

### Q104. A new server needs a static IP, gateway and DNS on RHEL 9. How do you configure it persistently? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** NetworkManager via `nmcli`.

**What I'd do →**
```bash
nmcli con show
sudo nmcli con mod "System eth0" ipv4.method manual \
  ipv4.addresses 10.0.1.25/24 ipv4.gateway 10.0.1.1 \
  ipv4.dns "10.0.0.2 10.0.0.3" ipv4.dns-search example.internal
sudo nmcli con up "System eth0"
ip addr show eth0; ip route; resolvectl status 2>/dev/null || cat /etc/resolv.conf
```

**Why →** Edits through `nmcli` persist in the connection profile; editing `/etc/resolv.conf` directly gets overwritten.

**Interview tip →** On Ubuntu say Netplan (`/etc/netplan/*.yaml` + `netplan try`), which rolls back if you lose connectivity.

</details>

### Q105. You're changing the network config of a remote server over SSH and could lose access. How do you protect yourself? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Have an automatic rollback.

**What I'd do →**
1. Ubuntu: `sudo netplan try --timeout 120` reverts unless you confirm.
2. Generic: schedule a revert before applying:
```bash
sudo cp /etc/nftables.conf /root/nftables.conf.bak
echo "cp /root/nftables.conf.bak /etc/nftables.conf && systemctl restart nftables" | sudo at now + 5 minutes
# apply change, test, then:
sudo atrm <job>
```
3. Make sure out-of-band console access works (cloud serial console, iDRAC/iLO).

**Why →** If you lose the connection, the revert runs without you.

**Interview tip →** Run changes inside `tmux` so a dropped SSH session doesn't kill a half-finished command.

</details>

### Q106. A web server should accept HTTP/HTTPS from anywhere but SSH only from the office. Configure the firewall. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Host firewall with an allowlist for SSH.

**What I'd do →**
firewalld:
```bash
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.0/24" service name="ssh" accept'
sudo firewall-cmd --reload && sudo firewall-cmd --list-all
```
ufw: `ufw allow 80,443/tcp && ufw allow from 203.0.113.0/24 to any port 22 proto tcp && ufw enable`.

**Why →** Narrow SSH exposure removes most brute-force noise.

**Interview tip →** In the cloud, do the same in the security group and keep the host firewall as a second layer.

</details>

### Q107. Your app servers need to reach the internet for updates but must not be reachable from it. How do you set it up on a Linux gateway? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A NAT gateway with forwarding and masquerade.

**What I'd do →**
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/90-forward.conf && sudo sysctl --system
sudo nft add table ip nat
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add rule ip nat postrouting oifname "eth0" ip saddr 10.0.2.0/24 masquerade
```
Add forward rules that allow `10.0.2.0/24 -> eth0` and only established/related traffic back. Save with `nft list ruleset > /etc/nftables.conf`.

**Why →** Outbound connections are translated; unsolicited inbound has no NAT mapping and is dropped.

**Interview tip →** In AWS, say you'd use a managed NAT Gateway rather than run this yourself.

</details>

### Q108. You need a server to have two network interfaces bonded for redundancy. How do you configure it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Link bonding (active-backup unless the switch supports LACP).

**What I'd do →**
```bash
sudo nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
sudo nmcli con add type ethernet slave-type bond con-name bond0-p1 ifname eno1 master bond0
sudo nmcli con add type ethernet slave-type bond con-name bond0-p2 ifname eno2 master bond0
sudo nmcli con mod bond0 ipv4.method manual ipv4.addresses 10.0.1.30/24 ipv4.gateway 10.0.1.1
sudo nmcli con up bond0
cat /proc/net/bonding/bond0
```

**Why →** active-backup needs no switch config; `mode=802.3ad` (LACP) gives aggregate bandwidth but needs switch support.

**Interview tip →** Test failover by pulling one cable or `ip link set eno1 down`.

</details>

### Q109. A developer needs to access an internal web UI on a remote server that only listens on 127.0.0.1:3000. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** SSH local port forward to localhost on the remote side.

**What I'd do →**
```bash
ssh -L 3000:127.0.0.1:3000 dev@server01 -N
# then open http://localhost:3000 locally
```

**Why →** The service stays unexposed; access is only through authenticated SSH.

**Interview tip →** Mention `-D 1080` (dynamic SOCKS proxy) when you need many internal services at once.

</details>

### Q110. You need to copy a 500 GB directory between two servers on a 1 Gbps link as fast as possible. What do you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Throughput and resumability matter.

**What I'd do →**
```bash
rsync -aHAX --info=progress2 --partial -e "ssh -T -c aes128-gcm@openssh.com -o Compression=no" /data/ dst:/data/
```
For many small files, parallelise: `ls /data | xargs -P4 -I{} rsync -aHAX /data/{} dst:/data/`. On a trusted network, `tar | mbuffer | nc` avoids encryption overhead.

**Why →** rsync resumes and only sends differences; a fast AES-GCM cipher with hardware support keeps CPU from becoming the bottleneck.

**Interview tip →** Calculate first: 500 GB at ~110 MB/s is roughly 80 minutes, so set expectations.

</details>

### Q111. Your app needs to resolve internal names like `db.internal` that aren't in public DNS. What options do you have on the host? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Point the resolver at internal DNS, or use split DNS.

**What I'd do →**
1. Best: configure the internal DNS servers via DHCP or NetworkManager (`ipv4.dns`).
2. With systemd-resolved, route only the internal domain to internal DNS:
```bash
sudo resolvectl dns eth1 10.0.0.2
sudo resolvectl domain eth1 '~internal'
```
(persist in NetworkManager or a `.network` file).
3. Quick fix for one or two hosts: `/etc/hosts`.

**Why →** Split DNS sends `*.internal` to internal servers and everything else to the normal resolver.

**Interview tip →** Mention `/etc/nsswitch.conf` controls whether `/etc/hosts` or DNS is consulted first.

</details>

### Q112. You must open SSH on a non-standard port 2222 on a RHEL server with SELinux enforcing. What steps are needed? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** sshd config, SELinux port label and firewall all need changing.

**What I'd do →**
```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
echo 'Port 2222' | sudo tee /etc/ssh/sshd_config.d/20-port.conf
sudo firewall-cmd --permanent --add-port=2222/tcp && sudo firewall-cmd --reload
sudo sshd -t && sudo systemctl restart sshd
ss -tlnp | grep 2222
```

**Why →** SELinux only lets sshd bind to ports labelled `ssh_port_t`; without it, sshd fails to start.

**Interview tip →** Say a non-standard port reduces log noise but isn't real security.

</details>

### Q113. Users connecting to a new server get "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED". The server was rebuilt. How do you handle this at scale? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** New host keys; you need a trusted way to distribute them.

**What I'd do →**
1. Confirm the rebuild is real (out-of-band check of the fingerprint: `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` on the console).
2. Remove the stale entry: `ssh-keygen -R host.example.com`.
3. Long term, sign host keys with an SSH CA and have clients trust it:
```
# client ~/.ssh/known_hosts
@cert-authority *.example.com ssh-ed25519 AAAA...CA
```
Servers use `HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub`.

**Why →** With host certificates, rebuilt servers just get a new signed cert and clients never see warnings or need TOFU.

**Interview tip →** Never tell people to use `StrictHostKeyChecking=no`; that's exactly how MITM gets through.

</details>

### Q114. You need to measure real network throughput between two data centre servers before a migration. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Measure bandwidth, latency and loss.

**What I'd do →**
```bash
# on server B
iperf3 -s
# on server A
iperf3 -c serverB -t 30 -P 4
iperf3 -c serverB -u -b 500M        # UDP, shows loss and jitter
mtr -rwc 100 serverB
```

**Why →** `-P 4` uses parallel streams, which shows what the link can do when one TCP stream is limited by latency.

**Interview tip →** Mention the bandwidth-delay product; high-latency links need bigger TCP windows.

</details>

### Q115. A service must be reachable on a virtual IP that fails over between two Linux load balancers. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** VRRP with keepalived.

**What I'd do →**
`/etc/keepalived/keepalived.conf` on the primary:
```
vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication { auth_type PASS auth_pass s3cr3t }
    virtual_ipaddress { 10.0.1.100/24 }
    track_script { chk_haproxy }
}
```
Backup node: `state BACKUP`, `priority 100`. Allow VRRP (protocol 112) in the firewall.

**Why →** If HAProxy dies, the priority drops and the VIP moves; neighbours learn the new MAC via gratuitous ARP.

**Interview tip →** In most clouds VRRP multicast doesn't work, so use the cloud LB or an API-driven failover.

</details>

### Q116. You need to capture traffic on a server to prove whether a client's requests are even arriving. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** tcpdump with a tight filter.

**What I'd do →**
```bash
sudo tcpdump -i any -nn 'tcp port 8080 and host 198.51.100.7' -c 200
sudo tcpdump -i eth0 -nn -s0 -w /tmp/cap.pcap 'port 8080'   # open in Wireshark later
```
Look for SYN without SYN-ACK (app or firewall), or SYN followed by RST (nothing listening).

**Why →** Packets on the wire are the ground truth when app logs and the client disagree.

**Interview tip →** Filter tightly on busy servers or the capture itself causes load.

</details>

### Q117. Your team uses password-protected SSH keys but automation needs to SSH without prompts. How do you balance the two? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Humans and machines need different keys.

**What I'd do →**
1. Humans: passphrase-protected keys with `ssh-agent`, or FIDO2 keys (`ssh-keygen -t ed25519-sk`).
2. Automation: separate key per system, no passphrase, stored in a secret manager, restricted on the server:
```
from="10.0.5.0/24",command="/usr/local/bin/backup-receive",no-pty,no-port-forwarding ssh-ed25519 AAAA... ci-backup
```
3. Rotate machine keys on a schedule.

**Why →** A leaked automation key can only do one thing from one network.

**Interview tip →** Never reuse a person's key in CI.

</details>

### Q118. You need to forward port 80 on a Linux host to a container or VM on 10.0.3.5:8080. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** DNAT plus forwarding.

**What I'd do →**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo nft add table ip nat
sudo nft add chain ip nat prerouting '{ type nat hook prerouting priority -100; }'
sudo nft add rule ip nat prerouting iifname "eth0" tcp dport 80 dnat to 10.0.3.5:8080
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add rule ip nat postrouting ip daddr 10.0.3.5 masquerade
```
Persist in `/etc/nftables.conf` and make `ip_forward` permanent in `/etc/sysctl.d/`.

**Why →** DNAT rewrites the destination; masquerade makes return traffic come back through this host.

**Interview tip →** Mention that locally generated traffic hits the `output` hook, not `prerouting`, so testing from the same host needs an extra rule.

</details>

### Q119. Large file uploads through a VPN tunnel hang while small requests work. You're asked to fix it on the Linux router. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic MTU/PMTUD problem; tunnel overhead shrinks the usable MTU.

**What I'd do →**
1. Confirm: `ping -M do -s 1472 remote` fails, a smaller size works.
2. Clamp TCP MSS on the router:
```bash
sudo nft add rule inet filter forward tcp flags syn tcp option maxseg size set rt mtu
```
3. Or lower the tunnel interface MTU (e.g. 1400) and make sure ICMP "fragmentation needed" isn't blocked.

**Why →** Big packets exceed the tunnel MTU and get dropped silently when ICMP is filtered.

**Interview tip →** "Small works, big hangs" should immediately make you think MTU.

</details>

### Q120. Your SSH sessions to a remote server drop after a few minutes idle through a corporate firewall. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A stateful firewall times out idle connections.

**What I'd do →**
Client side `~/.ssh/config`:
```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```
Or server side: `ClientAliveInterval 60`.

**Why →** Keepalive packets keep the connection entry alive in the firewall's state table.

**Interview tip →** Use `tmux` or `mosh` for flaky links so your work survives drops.

</details>

### Q121. You need a quick internal HTTP proxy so isolated hosts can download packages. What would you set up? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Forward proxy with an allowlist.

**What I'd do →**
1. Install Squid on a host with internet; in `squid.conf`:
```
acl internal src 10.0.0.0/8
acl repos dstdomain .ubuntu.com .rockylinux.org .docker.io
http_access allow internal repos
http_access deny all
http_port 3128
```
2. On clients: `proxy=http://proxy.internal:3128` in `/etc/dnf/dnf.conf` or `Acquire::http::Proxy "http://proxy.internal:3128";` in apt config.

**Why →** You control and log egress without giving hosts full internet.

**Interview tip →** For scale, a local mirror or Artifactory/Nexus proxy repo is better than a generic proxy.

</details>

### Q122. You need SSH access to a server behind NAT with no inbound ports open. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The server must call out; use a reverse tunnel or an agent.

**What I'd do →**
1. Reverse tunnel from the server to a relay:
```bash
autossh -M 0 -N -o ServerAliveInterval=30 -R 2222:localhost:22 tunnel@relay.example.com
```
Then from the relay: `ssh -p 2222 user@localhost`.
2. Run it as a systemd service so it restarts.
3. Production-grade: an outbound-only agent (AWS SSM, Teleport, Tailscale).

**Why →** NAT allows outbound connections; the tunnel carries traffic back in.

**Interview tip →** Mention locking down the relay with `GatewayPorts no` so the forwarded port isn't public.

</details>

### Q123. A team wants to know who is logged in and from where on a server right now and historically. Which commands? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Session accounting files plus auth logs.

**What I'd do →**
```bash
w                                  # current sessions and what they're running
who -a
last -F -n 20                      # history from /var/log/wtmp
lastb | head                       # failed logins from /var/log/btmp
journalctl -u sshd --since today | grep Accepted
```

**Why →** `wtmp` holds logins, `btmp` failed attempts and the journal has the key fingerprint used.

**Interview tip →** Note that an attacker with root can edit these files, so ship them off-box.

</details>

### Q124. Your application opens thousands of outbound connections per second to an API and starts failing with "Cannot assign requested address". How do you tune the host? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral port exhaustion from many short-lived connections in TIME_WAIT.

**What I'd do →**
1. Confirm: `ss -tan state time-wait | wc -l` and `sysctl net.ipv4.ip_local_port_range`.
2. Fix in the app first: use connection pooling and keep-alive.
3. Tune:
```bash
sudo sysctl -w net.ipv4.ip_local_port_range="10240 65535"
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
```
4. Spread across more destination IPs or source IPs if needed.

**Why →** Each connection needs a unique 4-tuple; with one destination IP:port you only have the ephemeral port range to vary.

**Interview tip →** Never mention `tcp_tw_recycle`; it was removed in Linux 4.12 because it broke NAT clients.

</details>

### Q125. You must make sure a server's hostname and FQDN are set correctly before joining it to a domain. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static hostname plus resolvable FQDN.

**What I'd do →**
```bash
sudo hostnamectl set-hostname app01.corp.example.com
hostnamectl
hostname -f
getent hosts app01.corp.example.com
```
Add a `/etc/hosts` entry if DNS isn't set up yet. On cloud images, set `preserve_hostname: true` in cloud-init so it isn't reset.

**Why →** Kerberos and many domain tools fail if the FQDN doesn't resolve.

**Interview tip →** Cloud-init resetting the hostname on reboot is a common surprise.

</details>

## Security, hardening and incident response

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. Monitoring shows a web server making outbound connections to an unknown IP on port 4444 at 2 AM. You suspect it is compromised. What do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Contain, preserve evidence, then investigate. Don't "clean" a live box and put it back.

**What I'd do →**
1. Contain: pull it out of the load balancer and restrict network at the security group/firewall to your forensic IPs only. Don't power it off yet (memory is evidence).
2. Capture volatile data:
```bash
ss -tanp > /tmp/ir/conns.txt
ps auxwwf > /tmp/ir/ps.txt
ls -l /proc/*/exe 2>/dev/null | grep deleted > /tmp/ir/deleted-exe.txt
last -F > /tmp/ir/last.txt
```
3. Snapshot the disk (EBS snapshot) for offline forensics.
4. Check persistence: crontabs, systemd units, `~/.ssh/authorized_keys`, `/etc/ld.so.preload`, new users.
5. Rebuild from a known-good image, rotate every credential the host had, and patch the entry point.

**Why →** You can't trust a compromised kernel or binaries; rebuilding is the only clean state.

**Interview tip →** Say "isolate, preserve, investigate, rebuild, rotate", and that you'd involve the security team immediately.

</details>

### Q127. After the compromise you need to find how the attacker got in. Where do you look? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timeline reconstruction from logs and the disk snapshot.

**What I'd do →**
1. Auth logs: `grep -E 'Accepted|Failed' /var/log/secure*` (or `auth.log`), especially unusual IPs or keys.
2. Web logs around the first bad timestamp: strange POSTs, webshell paths, `../` traversal, `${jndi:`.
3. Recently modified files on the mounted snapshot:
```bash
find /mnt/evidence -xdev -newermt "2026-11-08 00:00" ! -newermt "2026-11-09 03:00" -type f -ls
```
4. Package integrity: `rpm -Va` or `debsums -c` against the snapshot root.
5. Audit logs: `ausearch -ts 11/08/2026 -k exec`.

**Why →** The first malicious event tells you the vulnerability to fix fleet-wide.

**Interview tip →** Use central logs as the source of truth, since local ones may have been tampered with.

</details>

### Q128. Your CISO asks you to harden a fresh Linux server baseline. What's on your list? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Follow a benchmark and automate it.

**What I'd do →**
1. Patch and enable automatic security updates.
2. SSH: keys only, no root, allowlisted groups.
3. Remove unneeded packages and services (`systemctl list-unit-files --state=enabled`).
4. Host firewall default-deny inbound.
5. SELinux/AppArmor enforcing.
6. auditd plus log forwarding; time sync with chrony.
7. Mount options (`nodev,nosuid,noexec` on `/tmp`), sysctl hardening.
8. Scan with OpenSCAP: `oscap xccdf eval --profile cis --report report.html /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml`.

**Why →** CIS gives an auditable standard; OpenSCAP proves compliance.

**Interview tip →** Say you bake the hardened baseline into the golden image, not apply it by hand.

</details>

### Q129. A developer says "SELinux keeps breaking my app, can we just disable it?" How do you respond and what do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fix the policy issue, don't remove the protection.

**What I'd do →**
1. Find the denial:
```bash
sudo ausearch -m AVC -ts recent
sudo sealert -a /var/log/audit/audit.log      # setroubleshoot
```
2. Usually it's a label or boolean: `restorecon -Rv /srv/app`, `semanage fcontext -a -t httpd_sys_content_t '/srv/app(/.*)?'`, `setsebool -P httpd_can_network_connect on`.
3. Only if nothing fits, build a small local module: `audit2allow -M myapp` and review it before loading.
4. If needed during debugging, make only that domain permissive: `semanage permissive -a httpd_t`.

**Why →** SELinux contains exploits; most "breakage" is a missing label after files were moved.

**Interview tip →** "setenforce 0 is a diagnostic, not a fix" is the line to say.

</details>

### Q130. You need to know when anyone modifies `/etc/sudoers`, `/etc/passwd` or SSH config. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** File watch rules with auditd, shipped centrally.

**What I'd do →**
`/etc/audit/rules.d/50-identity.rules`:
```
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers
-w /etc/ssh/sshd_config -p wa -k sshd
```
Load with `sudo augenrules --load`, query with `ausearch -k sudoers -i`.

**Why →** auditd records who (auid survives sudo), when and with which process.

**Interview tip →** Mention `auid` (login UID), which shows the original user even after `sudo su -`.

</details>

### Q131. A critical OpenSSL CVE is announced. How do you find and patch affected servers quickly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inventory, patch, restart the right processes.

**What I'd do →**
1. Find versions fleet-wide: `ansible all -m shell -a "rpm -q openssl || dpkg -l openssl"`.
2. Patch: `dnf update openssl` / `apt-get install --only-upgrade openssl libssl3`.
3. Find processes still using the old library:
```bash
sudo lsof -n | grep -E 'libssl.*(deleted)'
sudo needs-restarting -s    # RHEL
sudo needrestart -r l       # Debian/Ubuntu
```
4. Restart those services or reboot.
5. Check containers and statically linked binaries separately.

**Why →** Updating the package doesn't fix processes that already loaded the old library into memory.

**Interview tip →** The "(deleted)" library trick shows real experience.

</details>

### Q132. How do you make sure only signed, approved packages can be installed on your servers? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** GPG verification plus controlled repositories.

**What I'd do →**
1. Keep `gpgcheck=1` (and `repo_gpgcheck=1` where supported) in every `.repo` file; for apt, use `signed-by=` keyrings.
2. Point servers only at an internal mirror (Satellite, Pulp, Artifactory) with approved content.
3. Block direct internet egress from servers.
4. Alert on `rpm -qa --qf '%{NAME} %{SIGPGP:pgpsig}\n'` showing unsigned packages.

**Why →** Signature checks stop tampered packages; internal mirrors control what is approved.

**Interview tip →** Mention apt's move away from `apt-key` to per-repo `signed-by` keyrings.

</details>

### Q133. A pen test reports your servers accept weak SSH ciphers and old key exchange algorithms. How do you fix it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Restrict algorithms in sshd or system crypto policy.

**What I'd do →**
- RHEL: `sudo update-crypto-policies --set FUTURE` (or a custom policy module), then restart sshd.
- Others, in an sshd drop-in:
```
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512
```
- Verify: `ssh -Q kex` and scan with `ssh-audit host`.

**Why →** Removing CBC ciphers, SHA-1 MACs and weak DH groups closes the findings.

**Interview tip →** Check old clients (legacy Java libraries, network gear) before rollout.

</details>

### Q134. You find a crypto-miner using 100% CPU under the `www-data` user. Beyond killing it, what do you do? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** The web app was exploited; the miner is just the symptom.

**What I'd do →**
1. Capture `ps`, `ls -l /proc/<PID>/exe`, `cat /proc/<PID>/cmdline`, `ss -tnp` before killing.
2. Find persistence: `crontab -l -u www-data`, `/tmp`, `/var/tmp`, `/dev/shm`, systemd user units, `.bashrc`.
3. Find the entry point in the web logs (RCE, upload of a webshell).
4. Rebuild the server, patch the app, rotate secrets the app could read.
5. Add prevention: `noexec` on `/tmp` and `/dev/shm`, egress filtering, and a systemd sandbox for the web service.

**Why →** If you only kill the process, the cron entry restarts it in a minute.

**Interview tip →** Mention egress restriction; miners and reverse shells need outbound access.

</details>

### Q135. Compliance requires that every server's file system be checked for unexpected changes daily. What do you implement? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** File integrity monitoring.

**What I'd do →**
```bash
sudo dnf install -y aide
sudo aide --init
sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
sudo aide --check
```
Schedule `aide --check` daily with a timer, send reports centrally, and re-baseline after approved changes (`aide --update`).

**Why →** AIDE hashes binaries and configs and reports differences.

**Interview tip →** Store the baseline DB off-host or read-only, or an attacker just updates it.

</details>

### Q136. You need to restrict which kernel modules can load, for example blocking USB storage on servers. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** modprobe blacklist plus install override.

**What I'd do →**
`/etc/modprobe.d/disable-usb-storage.conf`:
```
blacklist usb-storage
install usb-storage /bin/false
```
Unload if loaded: `sudo modprobe -r usb-storage`; rebuild initramfs if needed; check with `lsmod | grep usb_storage`.

**Why →** `blacklist` alone only stops auto-loading by alias; `install ... /bin/false` blocks explicit loads too.

**Interview tip →** Mention CIS also recommends disabling rare filesystems like `cramfs` and `udf` this way.

</details>

### Q137. An auditor asks you to prove which ports are open on a server and why. How do you produce that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Show listening sockets and map them to services and firewall rules.

**What I'd do →**
```bash
sudo ss -tulpn
sudo firewall-cmd --list-all       # or: sudo nft list ruleset / sudo ufw status verbose
nmap -sT -p- server01              # from another host, what's actually reachable
```
Document each port with its owning service and business reason; close anything without one.

**Why →** `ss` shows what listens; the external `nmap` shows what is exposed after firewalls.

**Interview tip →** Distinguish "listening on 127.0.0.1" from "listening on 0.0.0.0".

</details>

### Q138. A developer accidentally committed a private SSH key used on production servers to a public GitHub repo. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Treat the key as compromised immediately; deleting the commit isn't enough.

**What I'd do →**
1. Remove the public key from every `authorized_keys` now (Ansible `authorized_key state=absent`).
2. Issue a new key pair for the legitimate use.
3. Check auth logs for that key's fingerprint from unknown IPs (`ssh-keygen -lf key.pub`, then grep the fingerprint in logs with `LogLevel VERBOSE`).
4. Purge from git history (`git filter-repo`) and enable secret scanning / push protection.

**Why →** Bots scrape GitHub for keys within minutes.

**Interview tip →** "Revoke first, clean up history second."

</details>

### Q139. You need to apply kernel network hardening settings to all servers. Which sysctls and how? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Persistent sysctl drop-in.

**What I'd do →**
`/etc/sysctl.d/60-net-hardening.conf`:
```
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
```
Apply with `sudo sysctl --system`.

**Why →** These stop spoofing, ICMP redirect attacks and SYN floods, and hide kernel pointers.

**Interview tip →** Don't disable `ip_forward` on Kubernetes nodes or routers; know your host's role.

</details>

### Q140. You need to give a vendor temporary root-level access to debug an appliance VM for one day. How do you control it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Time-boxed, recorded, revocable access.

**What I'd do →**
1. Create a user with an expiry: `useradd -e 2026-11-10 vendor1`.
2. Grant sudo only via a dated drop-in and remove it with an `at` job: `echo "rm /etc/sudoers.d/vendor1; usermod -L vendor1" | at 18:00 tomorrow`.
3. Enable session recording (`tlog`, sudo `log_output`, or bastion recording).
4. Allow their source IP only on the firewall.
5. Review recordings afterwards and rotate any secrets they could see.

**Why →** Access that removes itself can't be forgotten.

**Interview tip →** Better: SSH certificates with an 8-hour validity, issued just for them.

</details>

### Q141. How do you protect a Linux server from SSH brute-force attempts flooding the logs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reduce exposure, then rate-limit.

**What I'd do →**
1. Disable password auth (brute force becomes pointless).
2. Restrict SSH by source IP in the firewall or security group.
3. Install fail2ban:
```
# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 5
bantime = 1h
```
`systemctl enable --now fail2ban`; check `fail2ban-client status sshd`.

**Why →** Keys-only stops the attack from ever succeeding; fail2ban cuts the noise.

**Interview tip →** Say exposure reduction (no public SSH, use SSM/bastion) beats fail2ban.

</details>

### Q142. A web app runs as `www-data` and you want it unable to read anything outside `/var/www`, even if exploited. What options do you have? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mandatory access control or systemd sandboxing.

**What I'd do →**
1. systemd: `ProtectSystem=strict`, `ProtectHome=yes`, `ReadWritePaths=/var/www/uploads`, `TemporaryFileSystem=/etc:ro` with `BindReadOnlyPaths=` for needed files, or `InaccessiblePaths=/etc/ssh /root`.
2. SELinux: keep the default `httpd_t` domain and label only allowed content `httpd_sys_content_t`.
3. AppArmor (Ubuntu): profile in `/etc/apparmor.d/` with `/var/www/** r,` and `aa-enforce`.

**Why →** Even with code execution, the process is fenced by the kernel.

**Interview tip →** Mention starting AppArmor profiles in complain mode (`aa-complain`) and tuning from logs.

</details>

### Q143. Your organisation must ensure that root's login and every privileged command is attributable. What design do you propose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No direct root logins; personal accounts plus sudo plus audit.

**What I'd do →**
1. `PermitRootLogin no`, root password locked or vaulted.
2. Personal accounts from central identity; sudo via groups.
3. `Defaults log_output` and auditd rules on `execve` with `auid>=1000`.
4. `pam_loginuid` so the auid survives `sudo su`.
5. Forward everything to a SIEM with immutable storage.

**Why →** Every root action maps back to a human.

**Interview tip →** Mention that `auid` is set at login and can't be changed by the user afterwards.

</details>

### Q144. You must scan all servers for known-vulnerable packages weekly. What do you use? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automated vulnerability scanning.

**What I'd do →**
- RHEL: `dnf updateinfo list --security` or OpenSCAP with OVAL content.
- Ubuntu: `pro security-status` or OVAL via `oscap`.
- Fleet tools: AWS Inspector, Wazuh, Qualys, or Trivy for root filesystems (`trivy rootfs /`).
Report to a dashboard with SLA by severity (critical in 7 days, etc.).

**Why →** Patching without measurement drifts; scanning gives an auditable backlog.

**Interview tip →** Mention prioritising by exploitability (CISA KEV list), not just CVSS score.

</details>

### Q145. A server holds secrets in `/etc/app/config.yml` readable by everyone. How do you fix and prevent this? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tighten permissions now, move secrets out long term.

**What I'd do →**
```bash
sudo chown root:app /etc/app/config.yml
sudo chmod 640 /etc/app/config.yml
sudo find /etc -type f -perm -o+r -name '*.yml' -exec grep -l -i 'password' {} +
```
Rotate the exposed secrets, then move them to Vault or a cloud secret manager and enforce modes in config management.

**Why →** Any local user or exploited service could read world-readable secrets.

**Interview tip →** Always rotate after exposure; fixing permissions doesn't un-leak a secret.

</details>

### Q146. Your team must find all SUID/SGID binaries on servers and remove unnecessary ones. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** SUID binaries are privilege escalation targets.

**What I'd do →**
```bash
sudo find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -exec ls -l {} \; 2>/dev/null
```
Compare against a known-good baseline per distro. Remove SUID where not needed: `chmod u-s /usr/bin/<tool>`. Make sure nothing suid lives in `/home`, `/tmp` or `/opt` from unknown sources, and mount those with `nosuid`.

**Why →** A new SUID root binary in an odd place is a classic backdoor.

**Interview tip →** Mention GTFOBins as the reference for abusable binaries.

</details>

### Q147. Ransomware hit a file server and encrypted shares. As the Linux admin, how do you recover? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Contain, identify scope, restore from clean backups, close the entry point.

**What I'd do →**
1. Disconnect the share and the infected clients; stop Samba/NFS.
2. Identify the start time from file mtimes and logs.
3. Verify backups/snapshots are clean and from before that time (immutable or offline copies).
4. Rebuild the server if it was compromised itself, restore data, rescan.
5. Rotate credentials and fix the access path (often a compromised user's SMB credentials).

**Why →** Restoring onto an infected host or from infected backups just re-encrypts.

**Interview tip →** Mention immutable backups (S3 Object Lock, read-only snapshots) as the real defence.

</details>

### Q148. How would you securely wipe a decommissioned server's disks? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Depends on media and encryption.

**What I'd do →**
1. If disks were LUKS-encrypted: destroy the key slots: `cryptsetup luksErase /dev/sdb` (crypto-erase).
2. SSD/NVMe: use the drive's secure erase: `nvme format /dev/nvme0n1 --ses=1` or `blkdiscard --secure` where supported.
3. HDD: `shred -n 1 -z /dev/sdb` or `dd if=/dev/zero`.
4. Document serial numbers and the method for audit.

**Why →** Overwriting doesn't reliably reach all SSD cells because of wear levelling; crypto-erase or firmware erase does.

**Interview tip →** Cloud volumes are crypto-erased by the provider when deleted.

</details>

### Q149. A user reports their account was used to log in from another country. How do you investigate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Possible credential or key theft.

**What I'd do →**
1. Lock the account and kill sessions: `usermod -L user; pkill -u user`.
2. Check logins: `last -F -i user`, `journalctl _COMM=sshd | grep user` for IP and key fingerprint.
3. Review `.bash_history`, `authorized_keys` (new keys?), crontab and files they touched.
4. Rotate their keys/passwords and check other systems they access.
5. Add MFA or geo/IP restrictions.

**Why →** The fingerprint tells you which key was used, which narrows down where it leaked.

**Interview tip →** `LogLevel VERBOSE` in sshd is what makes fingerprints appear in logs.

</details>

### Q150. You need to prove to auditors that your servers meet CIS Level 1. How do you produce evidence repeatedly? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automated scanning and remediation as code.

**What I'd do →**
1. Scan with OpenSCAP and the SCAP Security Guide profile, saving reports:
```bash
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results results.xml --report report.html /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```
2. Remediate using the generated Ansible playbook (`oscap xccdf generate fix --fix-type ansible`).
3. Bake compliance into the golden image and run scans on a schedule, storing reports centrally.
4. Document justified exceptions.

**Why →** Auditors want consistent, timestamped evidence, not screenshots.

**Interview tip →** Mention that some CIS controls break apps, so you test and record exceptions rather than applying blindly.

</details>

## Packages, kernel and boot

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. You must upgrade the kernel on 50 production servers and be able to roll back if the new kernel misbehaves. Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Kernels install side by side, so rollback is choosing the old one at boot. The risk is in the reboot itself.

**What I'd do →**
1. Test the new kernel on staging and a canary host for a few days.
2. Make sure old kernels are kept: `installonly_limit=3` in `/etc/dnf/dnf.conf` (apt keeps the previous one by default).
3. Roll out in batches: drain the host, `dnf update kernel` / `apt install linux-image-...`, reboot, check `uname -r`, health checks, re-add to the LB.
4. Rollback on a host:
```bash
sudo grubby --info=ALL | grep -E '^(index|kernel)'
sudo grubby --set-default /boot/vmlinuz-<old-version>
sudo reboot
```
5. Have console access ready in case the new kernel doesn't boot at all; pick the old entry from the GRUB menu.

**Why →** Batches limit the blast radius; keeping old kernels makes rollback a reboot, not a reinstall.

**Interview tip →** Mention `grub2-reboot <entry>` for a one-time boot into the new kernel, falling back automatically on the next reboot if it hangs and gets power-cycled.

</details>

### Q152. You need to pin a package version so routine updates don't upgrade it (e.g. Docker or PostgreSQL). How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Version locking per package manager.

**What I'd do →**
- RHEL: `sudo dnf install 'dnf-command(versionlock)'` then `sudo dnf versionlock add docker-ce` (`dnf versionlock list`, `delete` to remove).
- Debian/Ubuntu: `sudo apt-mark hold docker-ce` (`apt-mark showhold`, `unhold`).

**Why →** Uncontrolled major upgrades of stateful software cause outages.

**Interview tip →** Remember to unhold for security patches; a forgotten hold is technical debt.

</details>

### Q153. A package update broke an application. How do you roll it back? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Downgrade to the previous version.

**What I'd do →**
- RHEL:
```bash
sudo dnf history list
sudo dnf history info <ID>
sudo dnf history undo <ID>
# or: sudo dnf downgrade nginx-1.24.0-1.el9
```
- Debian/Ubuntu: `apt list -a nginx` then `sudo apt install nginx=1.24.0-2ubuntu1`, then `apt-mark hold nginx`.

**Why →** `dnf history undo` reverses the whole transaction including dependencies.

**Interview tip →** The previous version must still be in the repo or cache, which is why internal mirrors keep old versions.

</details>

### Q154. Your servers can't reach the internet but need packages. How do you provide repositories? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Internal mirror.

**What I'd do →**
1. On a host with internet, sync repos: `dnf reposync --repoid=baseos --download-metadata -p /srv/repos` (or `apt-mirror`/`aptly` for Debian).
2. Serve `/srv/repos` with nginx.
3. Point clients at it in `/etc/yum.repos.d/internal.repo` with `gpgcheck=1`.
4. Schedule syncs and keep dated snapshots for reproducibility.

**Why →** Faster, controlled, works offline, and lets you freeze a known-good repo state.

**Interview tip →** Mention Pulp, Satellite, Artifactory or Nexus as production-grade options.

</details>

### Q155. You need to compile and install software from source on a server that also uses packages. How do you avoid making a mess? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Keep source builds separate from package-managed paths.

**What I'd do →**
1. Build in a separate directory and install to `/opt/<name>-<version>` with `./configure --prefix=/opt/tool-2.1`.
2. Symlink `/opt/tool -> /opt/tool-2.1` for easy upgrades.
3. Better: package it (`fpm`, `rpmbuild`) so it can be removed cleanly and deployed via repo.
4. Don't install compilers on production; build in CI or a build container.

**Why →** `make install` into `/usr` collides with package files and can't be cleanly uninstalled.

**Interview tip →** Mention `checkinstall` or `fpm` for quick packaging.

</details>

### Q156. You need to change kernel boot parameters (e.g. add `transparent_hugepage=never`) persistently. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Edit the kernel command line and regenerate the bootloader config.

**What I'd do →**
- RHEL 8/9: `sudo grubby --update-kernel=ALL --args="transparent_hugepage=never"`.
- Debian/Ubuntu: add to `GRUB_CMDLINE_LINUX` in `/etc/default/grub` and run `sudo update-grub`.
- Reboot, verify with `cat /proc/cmdline` and `cat /sys/kernel/mm/transparent_hugepage/enabled`.

**Why →** Some settings must be applied at boot; editing `grub.cfg` directly gets overwritten.

**Interview tip →** Explain why DBs like MongoDB/Redis recommend disabling THP (latency spikes from compaction).

</details>

### Q157. A server must apply kernel security patches without rebooting. What are your options? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Live patching.

**What I'd do →**
- RHEL: `kpatch` (`dnf install kpatch-patch-$(uname -r | ...)`, `kpatch list`).
- Ubuntu: Livepatch via `pro enable livepatch`, `canonical-livepatch status`.
- Still plan regular reboots, because live patches cover only selected critical fixes.

**Why →** Live patching replaces vulnerable functions in the running kernel, buying time until a maintenance window.

**Interview tip →** The senior answer: design for reboots (redundancy, draining) so you don't depend on live patching.

</details>

### Q158. You need to upgrade a server from RHEL 8 to RHEL 9 (or Ubuntu 22.04 to 24.04). In-place or rebuild? Plan it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Major upgrades are risky; rebuilding is often cleaner.

**What I'd do →**
1. Preferred: build a new RHEL 9 server from code (Terraform + Ansible), migrate data, swap traffic.
2. If in-place is required:
   - RHEL: `leapp preupgrade`, fix inhibitors in `/var/log/leapp/leapp-report.txt`, then `leapp upgrade` and reboot.
   - Ubuntu: `sudo do-release-upgrade` after full update.
3. Snapshot the disk before starting; test on a clone first.
4. Verify services, SELinux contexts and custom repos after.

**Why →** Rebuilds give a clean, reproducible host; in-place carries old cruft but keeps the IP and identity.

**Interview tip →** Saying "snapshot first, test on a clone" is what interviewers want.

</details>

### Q159. A new server must load a third-party kernel module (e.g. a storage driver) on boot. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Install the module package, configure autoload.

**What I'd do →**
```bash
sudo modprobe vendor_drv
lsmod | grep vendor_drv
echo vendor_drv | sudo tee /etc/modules-load.d/vendor_drv.conf
echo 'options vendor_drv queue_depth=64' | sudo tee /etc/modprobe.d/vendor_drv.conf
```
Prefer DKMS or kmod packages so the module rebuilds for new kernels.

**Why →** `/etc/modules-load.d/` is read by `systemd-modules-load` at boot.

**Interview tip →** With Secure Boot, unsigned modules won't load; mention MOK signing.

</details>

### Q160. You must make sure a server boots into text mode only, no GUI, and reduce boot time. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Change the default systemd target and find slow units.

**What I'd do →**
```bash
sudo systemctl set-default multi-user.target
systemctl get-default
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain
```
Disable unneeded units with `systemctl disable --now <unit>`.

**Why →** Targets replaced runlevels; `multi-user.target` is the old runlevel 3.

**Interview tip →** `NetworkManager-wait-online` and `cloud-init` are common slow units.

</details>

### Q161. You've forgotten the root password of a VM with console access only. How do you regain access? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interrupt boot and get a shell before login.

**What I'd do →**
1. At GRUB press `e`, append `rd.break` (RHEL) or `init=/bin/bash` (others) to the `linux` line, boot with `Ctrl+x`.
2. RHEL with `rd.break`:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel      # SELinux relabel
exit; exit
```
3. With `init=/bin/bash`: `mount -o remount,rw /`, `passwd`, then `exec /sbin/init`.

**Why →** The `.autorelabel` step fixes the SELinux label on `/etc/shadow`, or nobody can log in.

**Interview tip →** Mention that this is why physical/console access equals root, and GRUB passwords exist.

</details>

### Q162. A cloud instance won't boot after a change and you only have the cloud console. How do you repair it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use the serial console, or mount the root volume on a rescue instance.

**What I'd do →**
1. Read the boot log: EC2 "Get system log" / serial console.
2. If serial console access works, boot to rescue/emergency and fix there.
3. Otherwise: stop the instance, detach the root volume, attach to a rescue instance in the same AZ:
```bash
lsblk
sudo mount -o nouuid /dev/nvme1n1p1 /mnt/rescue     # nouuid for XFS clones
sudo vi /mnt/rescue/etc/fstab                         # fix the bad change
sudo umount /mnt/rescue
```
4. Reattach as root device and start.

**Why →** You can't use GRUB interactively without a console, so you fix the filesystem from another machine.

**Interview tip →** Snapshot the volume before touching it.

</details>

### Q163. You need to rebuild the initramfs after adding a storage driver required for boot. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Boot-critical drivers must be in the initramfs.

**What I'd do →**
- RHEL: `echo 'add_drivers+=" vendor_drv "' | sudo tee /etc/dracut.conf.d/vendor.conf` then `sudo dracut -f --kver $(uname -r)`.
- Debian/Ubuntu: add `vendor_drv` to `/etc/initramfs-tools/modules` then `sudo update-initramfs -u -k all`.
- Check: `lsinitrd | grep vendor_drv` or `lsinitramfs /boot/initrd.img-$(uname -r) | grep vendor_drv`.

**Why →** If the root disk driver isn't in the initramfs, the kernel can't find root and drops to a dracut shell.

**Interview tip →** Also needed when moving a VM to hardware with different storage controllers.

</details>

### Q164. You want to keep `/boot` from filling with old kernels. How do you manage that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Limit installed kernels.

**What I'd do →**
- RHEL: `installonly_limit=3` in `/etc/dnf/dnf.conf`, clean up with `sudo dnf remove --oldinstallonly --setopt installonly_limit=2 kernel`.
- Ubuntu: `sudo apt autoremove --purge`.
- Check usage: `df -h /boot` and `rpm -q kernel` / `dpkg -l 'linux-image*'`.

**Why →** A full `/boot` makes the next kernel install fail half-way.

**Interview tip →** Never remove the running kernel (`uname -r`).

</details>

### Q165. You need to find which package owns a file, and which files a package installed. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Package database queries.

**What I'd do →**
```bash
rpm -qf /usr/sbin/sshd          # owner (RHEL)
rpm -ql openssh-server          # files
dnf provides '*/bin/dig'        # which package would provide it
dpkg -S /usr/sbin/sshd          # owner (Debian)
dpkg -L openssh-server
apt-file search bin/dig
```

**Why →** Essential when you need to reinstall a damaged file or find what provides a missing command.

**Interview tip →** `rpm -V pkg` / `debsums pkg` verify the files against the package checksums.

</details>

### Q166. A developer needs a newer Python than the distro offers, without breaking system tools. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Never replace the system Python.

**What I'd do →**
1. RHEL 9: install AppStream versions side by side: `dnf install python3.12`.
2. Ubuntu: deadsnakes PPA or `uv python install 3.13`.
3. Use virtual environments per app (`python3.12 -m venv /opt/app/venv`).
4. Or ship the app in a container.

**Why →** Tools like `dnf` and many system scripts depend on the platform Python; overwriting `/usr/bin/python3` breaks them.

**Interview tip →** Mention PEP 668 "externally managed environment" errors on new distros and that they're intentional.

</details>

### Q167. You need to tune kernel parameters for a high-traffic web server. Which ones and why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Connection backlog, file handles and buffers.

**What I'd do →**
`/etc/sysctl.d/70-web.conf`:
```
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_fin_timeout = 15
net.core.netdev_max_backlog = 16384
fs.file-max = 2097152
net.ipv4.tcp_congestion_control = bbr
```
Plus `LimitNOFILE=` in the service unit and the app's own listen backlog. Apply with `sysctl --system` and measure before and after.

**Why →** Defaults suit general workloads; bursts overflow small backlogs and drop SYNs.

**Interview tip →** Say "measure first" (`nstat`, `ss -s`, listen queue overflows in `netstat -s`), never copy-paste tuning blindly.

</details>

### Q168. You need to install a set of security tools on 100 servers with mixed distros consistently. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config management with distro-aware package lists.

**What I'd do →**
```yaml
- hosts: all
  become: true
  tasks:
    - ansible.builtin.package:
        name: "{{ item }}"
        state: present
      loop: "{{ sec_packages[ansible_os_family] }}"
  vars:
    sec_packages:
      RedHat: [audit, aide, fail2ban]
      Debian: [auditd, aide, fail2ban]
```

**Why →** Package names differ across families (`audit` vs `auditd`); the generic `package` module picks the right manager.

**Interview tip →** Mention idempotence: run it twice, second run reports no changes.

</details>

### Q169. A server shows "kernel: BUG" and occasional hangs. You want a crash dump next time it happens. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Configure kdump to capture vmcore on panic.

**What I'd do →**
```bash
sudo dnf install -y kexec-tools
sudo grubby --update-kernel=ALL --args="crashkernel=auto"   # or an explicit size like 512M
sudo systemctl enable --now kdump
sudo kdumpctl status
```
Also panic on soft lockups/hung tasks so they produce dumps: `kernel.softlockup_panic=1`, `kernel.hung_task_panic=1` (carefully). Analyse with `crash` and debuginfo, or send to the vendor.

**Why →** Without a vmcore you're guessing; with it, support can find the faulty driver.

**Interview tip →** Test it in a lab with `echo c > /proc/sysrq-trigger`, never on production.

</details>

### Q170. You need to install packages on a server temporarily for debugging, and remove them all afterwards. How do you keep it clean? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Track the transaction so you can undo it.

**What I'd do →**
- RHEL: install, then `dnf history undo last` (or the transaction ID) when done.
- Ubuntu: `apt install strace tcpdump`, then `apt purge strace tcpdump && apt autoremove`.
- Best: use a debug container sharing namespaces (`nsenter`, or `toolbox` on Fedora/RHEL) so the host stays clean.

**Why →** Debug tools left on production widen the attack surface and drift from the baseline.

**Interview tip →** Mention immutable hosts where you never install anything, only redeploy.

</details>

### Q171. You must ensure a server boots with Secure Boot enabled and still loads your custom kernel module. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sign the module with a key enrolled in MOK.

**What I'd do →**
```bash
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 3650 -subj "/CN=Org Module Signing/"
sudo mokutil --import MOK.der        # set a one-time password, reboot, enroll in MOK manager
sudo /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 MOK.priv MOK.der vendor_drv.ko
mokutil --sb-state
```
DKMS can sign automatically with configured keys.

**Why →** With Secure Boot, the kernel lockdown rejects unsigned modules ("Key was rejected by service").

**Interview tip →** Keep the signing private key off the servers.

</details>

### Q172. You need to schedule regular patch windows across 400 servers with minimal user impact. Design the process. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rings, automation, health checks, reporting.

**What I'd do →**
1. Group hosts: canary (5%), ring 1, ring 2, with HA pairs split across rings.
2. Per host, an Ansible playbook with `serial: 10%`: drain from LB, update, reboot if needed, wait for health checks, undrain.
3. Stop the rollout automatically when failures pass a threshold (`max_fail_percentage`).
4. Freeze repos to a dated snapshot so every ring gets the same packages.
5. Report compliance (patched/unpatched) to a dashboard.

**Why →** Same packages everywhere, limited blast radius, automatic stop.

**Interview tip →** Mentioning repo snapshots for consistency across rings sets you apart.

</details>

### Q173. A newly installed service isn't enabled at boot on Debian but is on RHEL. Why might that be, and how do you standardise? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Distro presets differ.

**What I'd do →**
1. Debian/Ubuntu usually enable and start services on install; RHEL doesn't, by preset policy.
2. Check: `systemctl is-enabled svc` and `systemctl preset svc`.
3. Standardise in config management: always state `enabled: true` and `state: started` explicitly.

**Why →** Relying on distro defaults causes surprises after a reboot.

**Interview tip →** Mention preset files under `/usr/lib/systemd/system-preset/`.

</details>

### Q174. You need to roll back a whole server to its state before a risky change, not just one package. What can you use? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Filesystem or volume snapshots.

**What I'd do →**
1. Cloud: snapshot the volume (or AMI) before the change; restore by creating a volume from it.
2. LVM: `lvcreate -s -n root_pre -L 10G /dev/vg0/root`; to roll back, `lvconvert --merge /dev/vg0/root_pre` and reboot.
3. Btrfs/Snapper (SUSE, Fedora): `snapper create`, then `snapper rollback`.
4. For VMs: hypervisor snapshot, deleted after the change succeeds.

**Why →** One step returns OS, config and packages to a known state.

**Interview tip →** Snapshots aren't backups; delete them after the change because they hurt performance and fill.

</details>

### Q175. You need to verify that no system binaries were tampered with after a suspected incident, without trusting the server's own tools. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Verify against package checksums from a trusted environment.

**What I'd do →**
1. Snapshot the disk and mount it read-only on a clean forensic host.
2. Verify using the clean host's `rpm` against the evidence root:
```bash
sudo rpm --root /mnt/evidence -Va | grep -E '^..5'    # checksum changed
```
For Debian: `debsums -r /mnt/evidence -c` or compare hashes with packages downloaded from the mirror.
3. Check `/etc/ld.so.preload` and loaded kernel modules in the memory capture.

**Why →** A rootkit can make the compromised host's `rpm` and `ls` lie.

**Interview tip →** Note that the package DB itself could be altered; downloaded packages from the official mirror are the true reference.

</details>

## Migrations, performance and fleet operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. You must migrate 2 TB of application data from an old server to a new one with less than 5 minutes of downtime. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Copy most of the data while live, then a short final sync during a freeze.

**What I'd do →**
1. Initial copy while the app runs (may take hours):
```bash
rsync -aHAXS --numeric-ids --info=progress2 /srv/data/ new:/srv/data/
```
2. Repeat the same rsync a few times; each pass gets faster.
3. Lower DNS TTL a day earlier.
4. Maintenance window: stop the app, final `rsync ... --delete`, start the app on the new server, switch DNS/LB.
5. Keep the old server read-only for rollback for a few days.

**Why →** The final pass only moves what changed since the last pass, so downtime is minutes.

**Interview tip →** Mention testing the cutover once end-to-end with a dry run (`rsync -n`) and measuring how long the final pass takes.

</details>

### Q177. The new server has different UIDs for the same users than the old one. What breaks during migration and how do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Files are owned by number; mismatched UIDs give files to the wrong user.

**What I'd do →**
1. Compare: `getent passwd app` on both hosts.
2. Option A: create users on the new host with the same UIDs/GIDs before copying (`useradd -u 1050 -g 1050 app`), then rsync with `--numeric-ids`.
3. Option B: copy and remap: `rsync --usermap=1050:2001 --groupmap=1050:2001` or fix afterwards with `find / -xdev -uid 1050 -exec chown -h 2001 {} +`.

**Why →** Consistent UIDs across a fleet (from central identity) avoid this entirely.

**Interview tip →** This is the same root cause as NFS ownership showing as `nobody` or wrong names.

</details>

### Q178. A server is "slow" according to users. What is your first 60 seconds of investigation? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check the four resources quickly: CPU, memory, disk, network.

**What I'd do →**
```bash
uptime                       # load trend
dmesg -T | tail              # OOM, I/O errors
vmstat 1 5                   # r, b, si/so, wa
mpstat -P ALL 1 3            # per-CPU
iostat -xz 1 3               # %util, await
free -h
sar -n DEV 1 3               # network throughput
top -c
```

**Why →** This quickly tells you which resource is saturated, so you dig in the right place.

**Interview tip →** Name the USE method (Utilisation, Saturation, Errors) for each resource.

</details>

### Q179. Your team must capacity-plan for a server whose disk grows 4% per week. How do you predict when it fills? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use historical metrics and alert on trends, not thresholds.

**What I'd do →**
1. Collect disk usage with node_exporter into Prometheus.
2. Alert when the forecast says the disk will fill within a week:
```
predict_linear(node_filesystem_avail_bytes{mountpoint="/data"}[7d], 7*24*3600) < 0
```
3. Plan the expansion before the date, and investigate why growth is 4% weekly (retention?).

**Why →** A fixed 90% threshold alerts too late on fast-growing disks and too early on stable ones.

**Interview tip →** `predict_linear` is a great specific detail to mention.

</details>

### Q180. You need to replace a physical server with a VM without changing its IP or hostname. Plan the cutover. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build the new host with a temporary IP, then take over the identity.

**What I'd do →**
1. Build the VM with a temporary IP, install and configure via Ansible.
2. Sync data (two-pass rsync).
3. Window: stop services on the old server, final sync, shut the old server down (or change its IP).
4. Reconfigure the VM with the original IP and hostname, restart networking.
5. Clear stale ARP on the gateway if needed: `arping -U -I eth0 10.0.1.50` (gratuitous ARP).
6. Run smoke tests; keep the old box powered off but intact.

**Why →** Clients keep working without DNS or config changes.

**Interview tip →** Mention SSH host keys: copy the old ones over or users get a "host key changed" warning.

</details>

### Q181. Memory usage on a server shows 95% used but the app is fine. A manager is worried. How do you explain and verify? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Linux uses free memory for page cache.

**What I'd do →**
```bash
free -h            # look at "available", not "free"
grep -E 'MemAvailable|Cached|Dirty' /proc/meminfo
vmstat 1 5         # si/so near 0 = no swapping pressure
```
Show that "available" is large and swap activity is zero.

**Why →** Cache is given back instantly when apps need it; unused RAM is wasted RAM.

**Interview tip →** Mention PSI (`cat /proc/pressure/memory`) as the real signal of memory pressure.

</details>

### Q182. You need to set up central logging for 200 Linux servers. Design it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Agents on hosts, a pipeline, and a searchable store.

**What I'd do →**
1. Agent per host: Fluent Bit/Vector reading journald and `/var/log/*`, or rsyslog forwarding over TLS:
```
*.* @@(o)logs.internal:6514
```
2. Buffer on disk at the agent so outages don't lose logs.
3. Store in Loki, Elasticsearch/OpenSearch or a cloud service; retention per log class.
4. Add labels: host, env, service.
5. Alerts on key events (auth failures, OOM kills, disk errors).

**Why →** Searchable history across hosts, and logs survive the host.

**Interview tip →** Mention that `@@` in rsyslog means TCP and `@` means UDP.

</details>

### Q183. A batch server is constantly swapping and the job is 10× slower. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The working set doesn't fit in RAM.

**What I'd do →**
1. Confirm: `vmstat 1` (`si`/`so` high), `sar -B` (major faults), `smem -s swap` or `grep VmSwap /proc/*/status`.
2. Short term: reduce concurrency or memory settings (JVM `-Xmx`, worker count).
3. Lower `vm.swappiness` (e.g. 10) if cache is pushing app pages out.
4. Long term: add RAM / bigger instance, or fix the job to stream data instead of loading it all.

**Why →** Swapping replaces nanosecond memory access with millisecond disk access.

**Interview tip →** Say you'd use cgroups (`MemoryHigh`) to isolate the batch job from other services.

</details>

### Q184. You must benchmark a new storage volume before moving a database onto it. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Measure IOPS, throughput and latency with a DB-like pattern.

**What I'd do →**
```bash
fio --name=randrw --filename=/mnt/new/fio.test --size=10G --direct=1 \
    --rw=randrw --rwmixread=70 --bs=8k --iodepth=32 --numjobs=4 \
    --ioengine=libaio --runtime=120 --time_based --group_reporting
fio --name=seqread --filename=/mnt/new/fio.test --direct=1 --rw=read --bs=1M --iodepth=16 --runtime=60 --time_based
```
Look at IOPS, bandwidth and p99 latency (`clat percentiles`). Compare with the current volume using the same job.

**Why →** `--direct=1` bypasses the page cache so you measure the disk, not RAM.

**Interview tip →** Note that cloud volumes may have burst credits; run long enough to see sustained performance.

</details>

### Q185. A legacy server nobody understands must be documented before it is migrated. How do you discover what it does? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inventory services, connections, jobs and data.

**What I'd do →**
```bash
systemctl list-units --type=service --state=running
ss -tulpn                                    # listening
ss -tnp state established                    # who it talks to
crontab -l; ls /etc/cron.d; systemctl list-timers
df -h; lsblk; cat /etc/fstab; showmount -e 2>/dev/null
rpm -qa --last | head -50 || ls -lt /var/lib/dpkg/info | head
ls -la /opt /srv /usr/local/bin
```
Watch connections over days (or flow logs) to catch weekly/monthly traffic.

**Why →** Hidden cron jobs and rare inbound clients are what break after a migration.

**Interview tip →** Mention leaving it powered off but intact for a while after cutover ("scream test").

</details>

### Q186. You need to make 300 servers' configuration consistent and prevent drift. What approach do you propose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Declarative configuration management with regular enforcement.

**What I'd do →**
1. Codify everything in Ansible roles in Git (users, packages, sysctl, sshd, agents).
2. Run it on a schedule (AWX/Ansible Automation Platform, or `ansible-pull` via timer) in check mode to report drift, and apply mode to fix.
3. Changes go through pull requests and CI (`ansible-lint`, Molecule tests).
4. Long term, immutable images: bake with Packer, replace instead of patching.

**Why →** Hand-made changes get overwritten or flagged, so the fleet matches Git.

**Interview tip →** Mention `--check --diff` to show exactly what would change.

</details>

### Q187. An application has a memory leak and gets slower until restarted weekly. While devs fix it, how do you reduce impact? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Contain the leak and restart in a controlled way.

**What I'd do →**
1. Cap memory with `MemoryMax=` and `Restart=on-failure` so it restarts cleanly instead of pushing the host into swap.
2. Schedule a rolling restart across instances during low traffic (systemd timer running `systemctl restart app`, one host at a time).
3. Graph RSS over time (`ps -o rss= -p PID` into metrics) to give devs data.

**Why →** Controlled restarts beat random OOM kills of other processes.

**Interview tip →** Be clear it's a mitigation, not a fix; give devs heap dumps.

</details>

### Q188. Your team wants to know which process is causing heavy disk I/O on a server. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-process I/O accounting.

**What I'd do →**
```bash
sudo iotop -oPa          # only processes doing I/O, accumulated
pidstat -d 1 5           # kB_rd/s, kB_wr/s per process
sudo cat /proc/<PID>/io
```

**Why →** `iostat` shows the disk is busy; `iotop` and `pidstat -d` show who is making it busy.

**Interview tip →** Mention that `iotop` needs `CONFIG_TASK_IO_ACCOUNTING` and root.

</details>

### Q189. You need to measure where an application spends CPU time in production without restarting it. How? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Sampling profiler with low overhead.

**What I'd do →**
```bash
sudo perf top -p <PID>
sudo perf record -F 99 -g -p <PID> -- sleep 30
sudo perf report --stdio | head -50
```
Generate a flame graph (`perf script | stackcollapse-perf.pl | flamegraph.pl > out.svg`). For runtimes: `py-spy record -p PID`, `async-profiler` for Java. eBPF tools: `profile-bpfcc -F 99 30`.

**Why →** Sampling at 99 Hz has negligible overhead and shows hot code paths.

**Interview tip →** Mention that missing symbols (no frame pointers / no debuginfo) make stacks unreadable.

</details>

### Q190. A service must handle a 10× traffic spike for a sale event next week. As the Linux admin, what do you prepare on the hosts? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Remove host-level limits before they become the bottleneck.

**What I'd do →**
1. Load-test to find the first limit.
2. Raise `LimitNOFILE` for the service, `net.core.somaxconn`, `tcp_max_syn_backlog`, ephemeral port range.
3. Check conntrack size if using NAT/firewalls: `net.netfilter.nf_conntrack_max`.
4. Ensure log volume won't fill disks (rotate by size, ship off-host).
5. Pre-scale instances and warm caches; confirm autoscaling and alerts.

**Why →** Under 10× load, kernel tables (fds, conntrack, backlog) overflow before CPU does.

**Interview tip →** Conntrack table full ("nf_conntrack: table full, dropping packet") is a classic surprise at peak.

</details>

### Q191. You need to give developers a reproducible Linux environment matching production for testing. What options do you propose? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same OS, same config, disposable.

**What I'd do →**
1. Containers from the same base image as production for app testing.
2. Vagrant or cloud VMs built from the same Packer image and Ansible roles for OS-level testing.
3. Ephemeral environments created from Terraform per branch.

**Why →** "Works on my machine" usually means a different OS, library or config.

**Interview tip →** Mention that the same Ansible code must build both prod and dev.

</details>

### Q192. You need to decommission a server safely. What's your checklist? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Make sure nothing depends on it, then remove all traces.

**What I'd do →**
1. Check active connections over days (`ss -tn`, flow logs) and DNS references.
2. Remove from LB, monitoring and backup jobs; stop services and leave powered off for a period (scream test).
3. Take a final backup/snapshot as policy requires.
4. Remove DNS records, config management inventory, firewall rules, certificates and secrets it used.
5. Wipe disks or delete volumes; record it in the CMDB.

**Why →** Orphaned DNS, firewall rules and credentials are security risks after the host is gone.

**Interview tip →** Mention dangling DNS records leading to subdomain takeover.

</details>

### Q193. Two servers must keep a directory in sync both ways for a simple active-active setup. What do you recommend? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bidirectional file sync is hard; conflicts are the main risk.

**What I'd do →**
1. Prefer removing the need: shared storage (NFS, EFS), object storage, or one writer.
2. If files are rarely edited on both sides: `unison` with a conflict policy, or `lsyncd` one-way per direction for separate subdirs.
3. For real active-active: a clustered filesystem (GlusterFS, CephFS) or DRBD with a cluster FS (complex).

**Why →** Two `rsync` crons in opposite directions will eventually delete or overwrite new files.

**Interview tip →** Saying "I'd avoid bidirectional sync" and explaining why is the senior answer.

</details>

### Q194. You need to move a service from one data centre to the cloud with a database that can't have long downtime. Outline the Linux side. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Replicate the data continuously and switch over at the end.

**What I'd do →**
1. Build cloud hosts from code; set up VPN or Direct Connect.
2. Set up DB replication from the DC primary to a cloud replica (native streaming replication or AWS DMS).
3. Sync static files with repeated rsync or `lsyncd`.
4. Lower DNS TTL; run read traffic tests against the cloud.
5. Cutover: stop writes, wait for replica lag to reach 0, promote the cloud replica, switch DNS, start apps.
6. Keep reverse replication or the old primary as a rollback path.

**Why →** Downtime becomes the time to promote and switch DNS, not the time to copy data.

**Interview tip →** Mention verifying replica lag and checksums before promoting.

</details>

### Q195. Your servers' clocks drift and logs from different hosts don't line up. How do you set up time sync properly? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** chrony with reliable sources.

**What I'd do →**
`/etc/chrony.conf`:
```
server 169.254.169.123 prefer iburst     # AWS Time Sync (or your internal NTP)
pool 2.pool.ntp.org iburst
makestep 1.0 3
rtcsync
```
`systemctl enable --now chronyd`, then check `chronyc tracking` and `chronyc sources -v`. Set every server to UTC: `timedatectl set-timezone UTC`.

**Why →** Consistent time matters for log correlation, TLS, Kerberos and distributed databases.

**Interview tip →** UTC on servers, local time only in dashboards.

</details>

### Q196. A team wants containers on a Linux host but you need to keep the host OS minimal and stable. How do you set up the host? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Container host baseline.

**What I'd do →**
1. Minimal OS install (or a container-optimised OS such as Bottlerocket, Flatcar or Fedora CoreOS).
2. Install only the runtime (containerd/Docker or Podman), monitoring and logging agents.
3. Put `/var/lib/containerd` or `/var/lib/docker` on its own volume.
4. cgroup v2, SELinux/AppArmor enforcing, `user namespaces` or rootless where possible.
5. Automate OS updates with reboots via a drain process.

**Why →** Fewer packages mean fewer patches and less drift; separate storage stops image bloat filling root.

**Interview tip →** Mention immutable OS options and why they suit container hosts.

</details>

### Q197. You need to find which servers in the fleet still run an old OS version scheduled for end of life. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inventory from facts.

**What I'd do →**
```bash
ansible all -m setup -a 'filter=ansible_distribution*' --tree /tmp/facts
ansible all -m shell -a 'grep -E "^(ID|VERSION_ID)=" /etc/os-release'
```
Or query your CMDB, cloud inventory (SSM Inventory) or monitoring labels. Then build a migration plan per server with owners and dates.

**Why →** EOL OS versions stop receiving security fixes.

**Interview tip →** Mention specific dates you know, e.g. CentOS 7 EOL June 2024, Ubuntu 20.04 standard support ended May 2025.

</details>

### Q198. You need to reduce boot-to-ready time for autoscaling instances from 6 minutes to under 1 minute. What do you change? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Move work from boot time to image build time.

**What I'd do →**
1. Measure: `systemd-analyze blame`, `cloud-init analyze blame`.
2. Bake packages, agents and app artefacts into the image with Packer; no `yum update` at boot.
3. Cloud-init only for instance-specific config (hostname, secrets fetch).
4. Disable unneeded units; avoid `NetworkManager-wait-online` delays where possible.
5. Pre-warm: use warm pools (AWS ASG) for very fast scale-out.

**Why →** Package installs and config runs at boot are the main delay.

**Interview tip →** "Bake, don't fry" is a well-known phrase for this.

</details>

### Q199. Your org wants to move from pets (hand-built servers) to cattle. What does that mean on Linux hosts and how do you start? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Servers become disposable and identical; state lives elsewhere.

**What I'd do →**
1. Externalise state: databases on managed services or dedicated volumes, logs shipped, files in object storage.
2. Codify the OS: Packer images + Ansible + cloud-init.
3. Deploy through autoscaling groups; replace instead of SSHing in to fix.
4. Disable direct SSH by default; use SSM or break-glass only.
5. Start with one stateless tier, prove it, expand.

**Why →** Any host can die without data loss or a hero fixing it at 3 AM.

**Interview tip →** Pick one service and describe the before/after; concrete beats theory.

</details>

### Q200. After a major outage caused by a manual Linux change, management asks you to prevent a repeat. What do you put in place? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Process and tooling, not blame.

**What I'd do →**
1. Blameless postmortem with a timeline and contributing factors.
2. All changes through code (Ansible/Terraform) with peer review and CI checks.
3. Staged rollout (canary first) and automatic rollback.
4. Restrict ad-hoc root via sudo rules and recorded sessions.
5. Better monitoring for the failure mode that was missed, and runbooks.

**Why →** Humans make mistakes; systems should make them hard to make and quick to detect.

**Interview tip →** Use the word "blameless" and give a concrete action item.

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
