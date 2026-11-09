# Linux — Production & Best Practices

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

> Hardening, patching, performance tuning, logging, capacity, backups and operational practices for Linux fleets · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Linux](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [SSH & Access Hardening](#ssh--access-hardening) — Q1–Q25
- [Firewalls, MAC & Compliance](#firewalls-mac--compliance) — Q26–Q45
- [Patching, Automation & Golden Images](#patching-automation--golden-images) — Q46–Q65
- [Performance & the USE Method](#performance--the-use-method) — Q66–Q100
- [Logs, Limits & High-Traffic Tuning](#logs-limits--high-traffic-tuning) — Q101–Q130
- [Disk, Capacity & Backups](#disk-capacity--backups) — Q131–Q150
- [User Lifecycle, Auditing & Time Sync](#user-lifecycle-auditing--time-sync) — Q151–Q170
- [Rapid-Fire Commands](#rapid-fire-commands) — Q171–Q200

---

## SSH & Access Hardening

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What are the first sshd_config changes you make on a new production server? 🟢

<details>
<summary><b>Answer</b></summary>

```
# /etc/ssh/sshd_config.d/10-hardening.conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowGroups ssh-users
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
sudo sshd -t && sudo systemctl reload ssh   # 'sshd' on RHEL
```

Keep your current session open and test a new login before closing it.

</details>

### Q2. Why use a drop-in file in /etc/ssh/sshd_config.d/ instead of editing sshd_config? 🟡

<details>
<summary><b>Answer</b></summary>

Package upgrades may ship a new `sshd_config` and prompt or create `.rpmnew`/`.dpkg-dist` files; drop-ins are never touched. Configuration management also manages one small file cleanly. Note that sshd uses the **first** value it reads for most keywords, and the main file `Include`s the drop-ins near the top, so drop-ins win. On Ubuntu cloud images, `50-cloud-init.conf` may set `PasswordAuthentication yes`, so name yours with a lower number.

</details>

### Q3. Why disable SSH password authentication entirely? 🟢

<details>
<summary><b>Answer</b></summary>

Every internet-facing SSH port receives constant brute-force and credential-stuffing attempts. With only key (or certificate) authentication, guessing is infeasible, and stolen passwords from other breaches are useless. It also removes the need for complex password policies on servers. Check effective settings with `sudo sshd -T | grep -E 'passwordauth|permitroot'`.

</details>

### Q4. Should you change the SSH port from 22? 🟡

<details>
<summary><b>Answer</b></summary>

It cuts log noise from automated scanners but is not real security; targeted scans find it in seconds. The real controls are: keys only, no root login, restricting source IPs with security groups or a firewall, and ideally not exposing SSH at all (bastion, VPN, AWS SSM Session Manager, or a zero-trust access proxy). If you do change it on SELinux systems you must also run `semanage port -a -t ssh_port_t -p tcp 2222`.

</details>

### Q5. What are SSH certificates and why do large fleets use them? 🔴

<details>
<summary><b>Answer</b></summary>

Instead of copying public keys to every `authorized_keys`, a CA signs user keys with an identity, allowed principals and an expiry. Servers trust the CA once:

```
# sshd_config
TrustedUserCAKeys /etc/ssh/user_ca.pub
AuthorizedPrincipalsFile /etc/ssh/principals/%u
```

```bash
ssh-keygen -s user_ca -I alice@corp -n ops -V +8h id_ed25519.pub
```

Short-lived certificates (hours) mean offboarding is automatic and no key sprawl exists. Host certificates also remove the "unknown host" prompt. Tools like Vault SSH, Teleport and step-ca automate issuance.

</details>

### Q6. How do you restrict what a specific SSH key can do? 🔴

<details>
<summary><b>Answer</b></summary>

Prefix options in `authorized_keys`:

```
from="10.0.0.0/16",command="/usr/local/bin/backup-receive",restrict ssh-ed25519 AAAA... backup@ci
```

`restrict` disables port, agent and X11 forwarding and PTY allocation; `command=` forces a single command regardless of what the client asks; `from=` limits source addresses. This is how we give CI or backup systems narrowly scoped access.

</details>

### Q7. How do you apply different sshd rules for different users or networks? 🟡

<details>
<summary><b>Answer</b></summary>

Use `Match` blocks at the end of the config:

```
Match Group sftp-only
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no

Match Address 10.0.0.0/8
    PasswordAuthentication no
```

The chroot directory must be owned by root and not group/other-writable. Test a user's effective settings with `sudo sshd -T -C user=bob,host=x,addr=10.1.2.3`.

</details>

### Q8. What is fail2ban and how does it work? 🟢

<details>
<summary><b>Answer</b></summary>

fail2ban watches logs (files or the journal) for failure patterns defined in filters, and when an IP exceeds `maxretry` within `findtime`, it runs an action, usually adding a firewall ban for `bantime`.

```ini
# /etc/fail2ban/jail.local
[sshd]
enabled  = true
backend  = systemd
maxretry = 5
findtime = 10m
bantime  = 1h
```

```bash
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 203.0.113.7
```

</details>

### Q9. What are the limits of fail2ban? 🟡

<details>
<summary><b>Answer</b></summary>

It is reactive and per-host: distributed attacks from thousands of IPs each trying a couple of times slip under `maxretry`. IPv6 makes per-address bans weak. Behind a load balancer it sees the LB's IP unless the real client IP is logged. It also adds no value if password auth is already disabled, beyond reducing noise. Edge controls (security groups, WAF, rate limits) and key-only auth matter more. Always add your bastion and monitoring IPs to `ignoreip`.

</details>

### Q10. How do you enforce MFA for SSH? 🔴

<details>
<summary><b>Answer</b></summary>

Options: FIDO2 hardware-backed keys (`ssh-keygen -t ed25519-sk`), where the key requires a physical touch; or PAM with TOTP (`pam_google_authenticator`) combined with keys:

```
AuthenticationMethods publickey,keyboard-interactive
KbdInteractiveAuthentication yes
UsePAM yes
```

and `auth required pam_google_authenticator.so` in `/etc/pam.d/sshd`. At scale we prefer an access proxy (Teleport, SSM, Boundary) that does SSO + MFA centrally and issues short-lived certs.

</details>

### Q11. Why use AWS SSM Session Manager instead of SSH? 🟡

<details>
<summary><b>Answer</b></summary>

The SSM agent makes an outbound connection to AWS, so no inbound port 22, no bastion and no SSH keys are needed. Access is controlled with IAM, every session can be logged to CloudWatch or S3, and you can still tunnel ports or run `ssh` over it (`ProxyCommand aws ssm start-session ...`). It needs the agent installed, an instance profile with `AmazonSSMManagedInstanceCore`, and network reach to the SSM endpoints (VPC endpoints in private subnets).

</details>

### Q12. How do you harden sudo in production? 🟡

<details>
<summary><b>Answer</b></summary>

- Use groups and files in `/etc/sudoers.d/`, managed by config management and validated with `visudo -cf`.
- Avoid blanket `NOPASSWD: ALL` for humans; scope service accounts to exact commands.
- `Defaults use_pty` and `Defaults logfile=/var/log/sudo.log`, or I/O logging (`log_output`) for sensitive hosts.
- `Defaults timestamp_timeout=5`.
- Ship sudo logs centrally.
- Never allow commands that spawn shells (`vi`, `less`, `find`, `awk`, `python`); use `sudoedit` for file edits.

</details>

### Q13. Why is `sudoedit` safer than `sudo vim /etc/file`? 🟡

<details>
<summary><b>Answer</b></summary>

`sudo vim` runs the editor as root, and from vim you can run `:!bash` to get a root shell, defeating a narrowly scoped sudo rule. `sudoedit /etc/nginx/nginx.conf` copies the file to a temp file, runs your editor **as you**, then copies it back as root. Grant it in sudoers with `%web sudoedit /etc/nginx/*.conf`.

</details>

### Q14. How do you limit brute-force at the PAM level? 🟡

<details>
<summary><b>Answer</b></summary>

`pam_faillock` (RHEL 8+, also Ubuntu 22.04+) locks accounts after repeated failures.

```
# /etc/security/faillock.conf
deny = 5
unlock_time = 900
fail_interval = 900
```

On RHEL enable with `authselect enable-feature with-faillock`. Check and reset with `faillock --user alice` and `faillock --user alice --reset`. Be careful: this can be used to lock out a known admin account, so combine it with SSH keys.

</details>

### Q15. What password policy settings do you configure on Linux? 🟢

<details>
<summary><b>Answer</b></summary>

`/etc/security/pwquality.conf` for strength (`minlen = 14`, `dcredit`, `ucredit`, `dictcheck = 1`), password history with `pam_pwhistory` (`remember = 5`), and ageing defaults in `/etc/login.defs` (`PASS_MAX_DAYS`, `PASS_WARN_AGE`). Modern guidance (NIST 800-63B) favours long passphrases and breach checks over forced periodic rotation, but many compliance frameworks still require ageing.

</details>

### Q16. How do you protect /tmp and other shared mounts? 🟡

<details>
<summary><b>Answer</b></summary>

Mount `/tmp`, `/var/tmp` and `/dev/shm` with `nodev,nosuid,noexec` so attackers cannot run dropped binaries or SUID files from there.

```
tmpfs  /dev/shm  tmpfs  defaults,nodev,nosuid,noexec  0 0
```

On systemd use `systemctl enable tmp.mount` with an override for options. Caveat: some installers and package scripts execute from `/tmp`, so test, and point them at `TMPDIR=/var/lib/app/tmp` if needed.

</details>

### Q17. How do you sandbox a systemd service? 🔴

<details>
<summary><b>Answer</b></summary>

```ini
[Service]
User=api
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/api /var/log/api
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
SystemCallFilter=@system-service
```

`systemd-analyze security api.service` scores the exposure and lists which directives are missing.

</details>

### Q18. What is DynamicUser= in systemd? 🔴

<details>
<summary><b>Answer</b></summary>

`DynamicUser=yes` allocates a transient UID/GID when the service starts and releases it when it stops, so no permanent account exists to be abused. It implies `ProtectSystem=strict`, `ProtectHome=read-only`, `PrivateTmp=yes` and `NoNewPrivileges=yes`. Persistent data goes in `StateDirectory=api` (`/var/lib/api`), `LogsDirectory=`, `CacheDirectory=`, which systemd chowns automatically. Great for stateless helpers and exporters.

</details>

### Q19. How do you remove unnecessary attack surface on a server? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ss -tulnp                                     # what is listening?
systemctl list-unit-files --state=enabled     # what starts at boot?
sudo systemctl disable --now cups avahi-daemon rpcbind
sudo apt purge telnet rsh-client              # or dnf remove
```

Start from a minimal image, install only what the role needs, and bind internal services (Redis, metrics, admin UIs) to `127.0.0.1` or a private interface.

</details>

### Q20. How do you set a login banner and why? 🟢

<details>
<summary><b>Answer</b></summary>

Legal/compliance teams want an authorized-use warning before login.

```
# sshd_config
Banner /etc/issue.net
```

Put the text in `/etc/issue.net` (pre-login over SSH), `/etc/issue` (console) and `/etc/motd` (post-login). Never reveal OS or kernel versions in banners; CIS checks that `\v`, `\r`, `\m`, `\s` escapes are absent.

</details>

### Q21. How do you detect unauthorized SSH keys across a fleet? 🔴

<details>
<summary><b>Answer</b></summary>

Manage `authorized_keys` only through config management (Ansible `authorized_key` with `exclusive: true`) or move to SSH certificates or `AuthorizedKeysCommand` that fetches keys from a central directory. Then periodically audit:

```bash
sudo find / -xdev -name authorized_keys -o -name authorized_keys2 2>/dev/null \
  | xargs -r sudo ssh-keygen -lf
```

Compare fingerprints with the approved list and alert on drift. Watch the files with auditd too.

</details>

### Q22. What is `AuthorizedKeysCommand` used for? 🟡

<details>
<summary><b>Answer</b></summary>

sshd runs a program to fetch a user's public keys at login instead of (or in addition to) reading files.

```
AuthorizedKeysCommand /usr/local/bin/fetch-keys %u
AuthorizedKeysCommandUser nobody
```

The program prints keys to stdout, e.g. from LDAP/SSSD (`sss_ssh_authorizedkeys`), IAM, or an internal API. Revoking access centrally takes effect on the next login without touching servers. The command must be root-owned and not writable by others.

</details>

### Q23. Root SSH is disabled. How do admins still do root tasks and keep accountability? 🟢

<details>
<summary><b>Answer</b></summary>

Each admin logs in with a personal account and escalates with `sudo`. Every command is then tied to a named user in `/var/log/auth.log` / `/var/log/secure` and the journal, and can be shipped to a SIEM. Shared root passwords are locked (`passwd -l root`) or stored in a vault for break-glass console access only.

</details>

### Q24. What does "break-glass access" mean for Linux servers? 🟡

<details>
<summary><b>Answer</b></summary>

An emergency path for when normal access (SSO, LDAP, bastion, SSM) is down. Typical design: a local emergency account or root password stored in a vault with dual-control checkout, usable only on the serial/cloud console, with alerting whenever it is used and mandatory rotation afterwards. Test it quarterly; an untested break-glass process fails exactly when you need it.

</details>

### Q25. How do you keep sshd itself secure over time? 🟡

<details>
<summary><b>Answer</b></summary>

Patch OpenSSH promptly (regreSSHion CVE-2024-6387 was a remote unauthenticated RCE in sshd), remove weak algorithms, and audit regularly:

```bash
ssh -V
sudo sshd -T | grep -E '^(ciphers|macs|kexalgorithms)'
ssh-audit server.example.com    # third-party scanner
```

Also restrict SSH at the network layer so a future zero-day is not reachable from the internet.

</details>

---

## Firewalls, MAC & Compliance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. How do ufw, firewalld, nftables and iptables relate? 🟢

<details>
<summary><b>Answer</b></summary>

The kernel packet filter is **netfilter**. `iptables` (legacy) and `nftables` are the low-level rule interfaces; on modern distros `iptables` is often `iptables-nft`, a compatibility layer over nftables. **ufw** (Ubuntu) and **firewalld** (RHEL) are friendlier front-ends that generate nftables/iptables rules. Use one front-end per host; mixing them leads to rules overwriting each other.

</details>

### Q27. Configure ufw for a web server with SSH only from the office. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 203.0.113.0/24 to any port 22 proto tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
sudo ufw status numbered
```

Add the SSH rule **before** `enable`, or you lock yourself out of a remote box.

</details>

### Q28. Explain firewalld zones and runtime vs permanent configuration. 🟡

<details>
<summary><b>Answer</b></summary>

Zones are trust levels bound to interfaces or source networks (`public`, `internal`, `trusted`, `drop`). Changes are **runtime** by default and lost on reload; `--permanent` writes config but does not apply until reload.

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-source=10.0.0.0/16 --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all --zone=public
```

`firewall-cmd --runtime-to-permanent` saves tested runtime rules.

</details>

### Q29. Write a minimal nftables ruleset for a server. 🔴

<details>
<summary><b>Answer</b></summary>

```
#!/usr/sbin/nft -f
flush ruleset
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    ct state invalid drop
    iif lo accept
    meta l4proto { icmp, ipv6-icmp } accept
    tcp dport 22 ip saddr 10.0.0.0/8 accept
    tcp dport { 80, 443 } accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output priority 0; policy accept; }
}
```

```bash
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
sudo systemctl enable nftables
```

`inet` covers IPv4 and IPv6 in one table.

</details>

### Q30. Why are nftables preferred over iptables at scale? 🔴

<details>
<summary><b>Answer</b></summary>

nftables updates rulesets atomically (no window with half-applied rules), handles IPv4/IPv6 in one `inet` table, and has native sets and maps with O(1) lookups, so blocking 100k IPs is one rule with a set instead of 100k linear rules. iptables evaluates rules linearly and rewrites whole tables on each change, which became a bottleneck for kube-proxy with thousands of Services (hence kube-proxy's nftables and IPVS modes).

</details>

### Q31. How do you avoid locking yourself out when changing remote firewall rules? 🟡

<details>
<summary><b>Answer</b></summary>

- Schedule an automatic rollback first: `echo "nft -f /etc/nftables.conf.bak" | sudo at now + 5 minutes`, then cancel with `atrm` once you confirm access.
- Use `firewall-cmd` runtime changes (lost on reload) before making them permanent.
- Keep an established SSH session open (`ct state established accept` keeps it alive).
- Have console access (cloud serial console, iLO/iDRAC) ready.

</details>

### Q32. Host firewall vs cloud security groups: do you need both? 🟡

<details>
<summary><b>Answer</b></summary>

Security groups / NSGs are the primary control in the cloud: stateful, enforced outside the VM (a compromised host cannot disable them), and managed with Terraform. A host firewall adds defence in depth, protects against misconfigured security groups, and can filter between processes on the same subnet. Many teams keep host firewalls minimal on cloud VMs and strict on bare metal. In Kubernetes nodes, avoid host firewalls that fight with CNI/kube-proxy rules.

</details>

### Q33. What is SELinux, in practical terms? 🟢

<details>
<summary><b>Answer</b></summary>

SELinux is mandatory access control on RHEL/Fedora: every process and file has a **label** (context), and a policy says which process types may access which object types. Even if nginx runs as root and is exploited, the `httpd_t` domain cannot read `/home` or `/etc/shadow`. Modes: `enforcing`, `permissive` (log only), `disabled`.

```bash
getenforce; sestatus
ls -Z /var/www/html; ps -eZ | grep nginx
```

</details>

### Q34. An app gets "Permission denied" only with SELinux enforcing. How do you fix it properly? 🟡

<details>
<summary><b>Answer</b></summary>

Do not disable SELinux. Find the denial and apply the right fix:

```bash
sudo ausearch -m AVC -ts recent
sudo sealert -a /var/log/audit/audit.log           # setroubleshoot explains + suggests
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/www(/.*)?'
sudo restorecon -Rv /srv/www                        # wrong file label
sudo setsebool -P httpd_can_network_connect on      # needed boolean
sudo semanage port -a -t http_port_t -p tcp 8081    # non-standard port
```

Only as a last resort build a local module with `audit2allow -M`, and review what it allows.

</details>

### Q35. Why use `restorecon` instead of `chcon`? 🟡

<details>
<summary><b>Answer</b></summary>

`chcon` changes a label directly, but the change is lost on the next relabel (`restorecon`, `fixfiles`, or a full `/.autorelabel`). The persistent approach is to define the rule in policy with `semanage fcontext -a` and then apply it with `restorecon`. Files moved with `mv` keep their original label (copies get the destination's default), a classic cause of AVC denials.

</details>

### Q36. What is AppArmor and how does it compare to SELinux? 🟡

<details>
<summary><b>Answer</b></summary>

AppArmor (Ubuntu, Debian, SUSE) confines programs with **path-based** profiles in `/etc/apparmor.d/`, which are easier to read and write. SELinux is **label-based**, more granular, and covers the whole system including MLS. Both are LSMs.

```bash
sudo aa-status
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx    # log only
sudo aa-enforce  /etc/apparmor.d/usr.sbin.nginx
sudo aa-logprof                                     # build rules from logs
```

</details>

### Q37. What are CIS Benchmarks and how do you apply them? 🟢

<details>
<summary><b>Answer</b></summary>

CIS publishes distro-specific hardening guides with Level 1 (practical baseline) and Level 2 (defence in depth, may break things) controls: partitions, services, sysctls, SSH, auditd, permissions, logging. We apply them through code, never by hand: Ansible roles (e.g. ansible-lockdown), Ubuntu Security Guide (`usg`), or CIS-hardened marketplace images, and scan with OpenSCAP or CIS-CAT. Document justified exceptions instead of blindly applying every control.

</details>

### Q38. How do you scan a host for compliance with OpenSCAP? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sudo dnf install openscap-scanner scap-security-guide
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml   # list profiles
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results results.xml --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

The HTML report shows pass/fail per rule with remediation; `oscap xccdf generate fix --fix-type ansible` produces a playbook. Run it in CI against golden images.

</details>

### Q39. Which sysctl settings belong in a security baseline? 🟡

<details>
<summary><b>Answer</b></summary>

```
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.suid_dumpable = 0
```

Do not set `ip_forward=0` on Kubernetes or Docker hosts.

</details>

### Q40. How do you detect file tampering on critical system files? 🔴

<details>
<summary><b>Answer</b></summary>

Use file integrity monitoring: AIDE builds a database of hashes and attributes and reports changes.

```bash
sudo aide --init && sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
sudo aide --check
```

Also verify package-owned files: `rpm -Va` or `debsums -c`. Store the baseline off-host (an attacker with root can update a local DB), and in cloud fleets prefer immutable images where drift itself is the alert.

</details>

### Q41. How do you find world-writable files and other risky permissions? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo find / -xdev -type f -perm -0002 -ls 2>/dev/null           # world-writable files
sudo find / -xdev -type d -perm -0002 ! -perm -1000 -ls 2>/dev/null   # world-writable dirs without sticky bit
sudo find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null   # SUID/SGID
```

Run these in scheduled compliance scans and compare against a known-good baseline.

</details>

### Q42. How do you secure a Linux host against known vulnerable kernels and packages? 🟢

<details>
<summary><b>Answer</b></summary>

Subscribe to distro security advisories (USN, RHSA), run vulnerability scanners (Amazon Inspector, Qualys, Trivy for hosts/images, `dnf updateinfo list --security`), patch on a defined SLA by severity, and track kernel versions fleet-wide. Remember a kernel update is only effective after a reboot (or livepatch).

</details>

### Q43. Why should you never set SELinux to disabled, even temporarily? 🟡

<details>
<summary><b>Answer</b></summary>

Booting with SELinux disabled means files created during that time get **no labels**. When you re-enable it, those files cause denials everywhere until a full relabel (`touch /.autorelabel && reboot`), which can take a long time on large filesystems. Use `setenforce 0` (permissive) for short debugging instead, or make a single domain permissive: `semanage permissive -a httpd_t`. On RHEL 9, `SELINUX=disabled` in `/etc/selinux/config` no longer fully disables SELinux (it only skips loading the policy); a full disable needs `selinux=0` on the kernel command line.

</details>

### Q44. How do you protect secrets on a Linux server? 🟡

<details>
<summary><b>Answer</b></summary>

- Do not bake secrets into images or commit them to repos; fetch at runtime from Vault, AWS Secrets Manager/SSM, or use systemd credentials (`LoadCredential=`).
- File permissions `0600`/`0640` owned by the service user, never world-readable.
- Avoid passing secrets as command-line arguments (visible in `ps` and `/proc/<pid>/cmdline`); environment variables are visible in `/proc/<pid>/environ` to root and the same user.
- Use instance roles instead of long-lived cloud keys.

</details>

### Q45. What is kernel lockdown and when would you enable it? 🔴

<details>
<summary><b>Answer</b></summary>

Lockdown (an LSM) restricts root from modifying the running kernel: no unsigned modules, no `/dev/mem` or `/dev/kmem` writes, no kexec of unsigned kernels, restricted BPF and perf in `confidentiality` mode. It is enabled automatically with Secure Boot on many distros (`integrity` mode). Check `cat /sys/kernel/security/lockdown`. It hardens against rootkits but can break tools like some eBPF tracers and out-of-tree drivers.

</details>

---
## Patching, Automation & Golden Images

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q46. What does a good Linux patching strategy look like? 🟢

<details>
<summary><b>Answer</b></summary>

- **Inventory**: know every host, OS version and kernel.
- **SLAs by severity**: e.g. critical in 72 hours, high in 14 days, the rest monthly.
- **Rings**: dev → staging → a production canary → the rest of production.
- **Automation**: Ansible, AWS SSM Patch Manager, or image rebuilds; no hand patching.
- **Reboots**: scheduled and rolling, drained from load balancers first.
- **Verification**: health checks and a rollback plan (snapshot or previous image).
- **Reporting**: compliance dashboard of pending updates and reboot-required hosts.

</details>

### Q47. In-place patching vs immutable image replacement: which do you prefer? 🟡

<details>
<summary><b>Answer</b></summary>

For cloud fleets behind load balancers or in autoscaling groups, we rebuild a patched golden image weekly and roll instances (immutable). Every host is then identical, tested, and rollback means using the previous AMI. In-place patching suits long-lived stateful servers (databases, on-prem, licensed boxes) where replacement is costly, done with rolling automation and snapshots. Most organizations run both.

</details>

### Q48. How do you know whether a server needs a reboot after patching? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required.pkgs   # Debian/Ubuntu
sudo dnf needs-restarting -r      # RHEL: exit code 1 = reboot needed
sudo dnf needs-restarting -s      # services that need a restart
sudo needrestart -r l             # Debian: list services using old libraries
```

Also compare `uname -r` with the newest installed kernel.

</details>

### Q49. How do you configure automatic security updates? 🟢

<details>
<summary><b>Answer</b></summary>

Ubuntu/Debian: `unattended-upgrades`.

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
# /etc/apt/apt.conf.d/50unattended-upgrades: Allowed-Origins, Automatic-Reboot "false"
```

RHEL: `dnf-automatic` with `upgrade_type = security` and `apply_updates = yes` in `/etc/dnf/automatic.conf`, then `systemctl enable --now dnf-automatic.timer`. On critical hosts we auto-download but apply in a maintenance window.

</details>

### Q50. What is kernel live patching? 🟡

<details>
<summary><b>Answer</b></summary>

Livepatch replaces vulnerable kernel functions in memory without a reboot, using ftrace-based redirection. Offerings: Canonical Livepatch (Ubuntu Pro), RHEL `kpatch`, SUSE kGraft/klp, KernelCare. It closes the window for critical CVEs on hosts that cannot reboot quickly, but it is not a substitute for eventually rebooting into the patched kernel; patches only cover selected CVEs.

```bash
sudo canonical-livepatch status
kpatch list
```

</details>

### Q51. How do you roll kernel updates across a 200-node fleet safely? 🔴

<details>
<summary><b>Answer</b></summary>

1. Patch and reboot dev/staging, run the test suite and soak for a few days.
2. Canary: a few production nodes in each AZ; watch error rates, latency, dmesg.
3. Roll in batches (Ansible `serial: 10%`, `max_fail_percentage: 0`): drain from LB or `kubectl drain`, patch, reboot, wait for health checks, re-enable.
4. Stop automatically on failure.
5. Keep the previous kernel installed for GRUB fallback; `grub2-set-default` or `grubby --set-default` to roll back.

</details>

### Q52. What is a golden image? 🟢

<details>
<summary><b>Answer</b></summary>

A pre-built, hardened, patched machine image (AMI, Azure image, VMware template) containing the OS baseline: CIS hardening, monitoring and log agents, security tools, internal CA certificates, and sometimes the app runtime. Every server launches from it, so boot is fast and identical. Images are versioned, scanned and promoted through environments just like application artifacts.

</details>

### Q53. How do you build golden images with Packer? 🟡

<details>
<summary><b>Answer</b></summary>

```hcl
source "amazon-ebs" "base" {
  region        = "ap-south-1"
  instance_type = "t3.small"
  ssh_username  = "ubuntu"
  ami_name      = "base-ubuntu24-{{timestamp}}"
  source_ami_filter {
    filters     = { name = "ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" }
    owners      = ["099720109477"]
    most_recent = true
  }
}
build {
  sources = ["source.amazon-ebs.base"]
  provisioner "ansible" { playbook_file = "./harden.yml" }
}
```

```bash
packer init . && packer validate . && packer build .
```

Run it in CI on a schedule, then scan and test the result before promoting.

</details>

### Q54. What should go into the image vs into boot-time configuration? 🟡

<details>
<summary><b>Answer</b></summary>

**Bake** what is slow, stable and common: OS patches, hardening, agents, runtimes, large dependencies. **Configure at boot** (cloud-init/user-data, SSM, config management) what is environment-specific or secret: hostnames, environment name, secrets fetched from a vault, service discovery endpoints. Never bake secrets, SSH host keys, or machine IDs into the image.

</details>

### Q55. What must you clean up before capturing an image? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sudo cloud-init clean --logs --machine-id   # reset cloud-init state and machine-id
sudo truncate -s0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*              # regenerated on first boot
sudo rm -rf /tmp/* /var/tmp/* /root/.ssh/authorized_keys /home/*/.bash_history
sudo journalctl --rotate && sudo journalctl --vacuum-time=1s
```

Duplicate machine-ids break DHCP client IDs, journald and some monitoring agents; duplicate SSH host keys make every server look identical to an attacker.

</details>

### Q56. What is cloud-init and what does it do at first boot? 🟢

<details>
<summary><b>Answer</b></summary>

cloud-init reads instance metadata and user-data on first boot and configures the instance: hostname, users and SSH keys, package installs, files, and arbitrary commands.

```yaml
#cloud-config
package_update: true
packages: [nginx]
write_files:
  - path: /etc/app/env
    content: "ENV=prod\n"
runcmd:
  - systemctl enable --now nginx
```

Debug with `cloud-init status --long` and `/var/log/cloud-init-output.log`.

</details>

### Q57. Why use configuration management (Ansible) even with golden images? 🟡

<details>
<summary><b>Answer</b></summary>

The same Ansible roles build the image (Packer provisioner) and keep long-lived servers converged, so one source of truth exists. It handles things images cannot: rotating certificates, adding users fleet-wide, emergency config changes, and drift remediation on stateful hosts. Running playbooks in `--check --diff` mode on a schedule also detects drift.

</details>

### Q58. What is configuration drift and how do you detect it? 🟡

<details>
<summary><b>Answer</b></summary>

Drift is when a server's real state diverges from its declared state, usually from manual hotfixes. Detect it with scheduled `ansible-playbook --check --diff`, AIDE / package verification (`rpm -Va`), or compliance scanners. Prevent it by removing interactive root access, making changes only through pipelines, and replacing rather than repairing instances.

</details>

### Q59. How do you make an Ansible task idempotent? 🟡

<details>
<summary><b>Answer</b></summary>

Use modules that describe desired state (`package`, `template`, `lineinfile`, `systemd`, `user`) instead of `shell`/`command`. When you must run a command, guard it with `creates:`/`removes:` or `changed_when:`/`when:`.

```yaml
- name: Initialize app DB once
  ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /var/lib/app/.initialized
```

Run twice; the second run must report `changed=0`.

</details>

### Q60. How do you handle a critical zero-day (e.g. OpenSSH or glibc) across the fleet in hours? 🔴

<details>
<summary><b>Answer</b></summary>

1. Assess exposure: which versions are affected, which hosts are internet-facing (inventory + scanner data).
2. Mitigate immediately where possible: restrict network access, config workarounds (e.g. `LoginGraceTime 0` was the regreSSHion mitigation).
3. Patch in fast rings: canary → internet-facing → internal, with automation (SSM Run Command / Ansible) and restarts of affected services or reboots for kernel/glibc.
4. Verify with package version queries fleet-wide, and rebuild golden images so new instances are not vulnerable.
5. Write a post-incident review on time-to-patch.

</details>

### Q61. Why pin repositories to snapshots or internal mirrors? 🔴

<details>
<summary><b>Answer</b></summary>

Public repos change daily, so two servers patched an hour apart can end up with different versions, and a bad upstream package can hit production unexpectedly. An internal mirror or dated snapshot (Pulp, Satellite, Artifactory, aptly, `snapshot.ubuntu.com`) means dev, staging and prod install exactly the same tested set, works without internet access, and allows controlled promotion between environments.

</details>

### Q62. How do you avoid apt/dnf prompts in automation? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get -y \
  -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" upgrade
sudo dnf -y upgrade
```

Use `apt-get` rather than `apt` in scripts; `apt` warns that its CLI is not stable for scripting. `--force-confold` keeps your modified config files when packages ship new ones.

</details>

### Q63. How do you handle "Could not get lock /var/lib/dpkg/lock-frontend" in automation? 🟡

<details>
<summary><b>Answer</b></summary>

Another apt process holds the lock, often `unattended-upgrades` or cloud-init on a fresh instance. Do not delete lock files. Wait for it:

```bash
cloud-init status --wait
while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 5; done
```

apt-get also supports `-o DPkg::Lock::Timeout=300`. In images, disable automatic upgrades on first boot if your pipeline handles patching.

</details>

### Q64. What is the role of an inventory/CMDB in Linux operations? 🟢

<details>
<summary><b>Answer</b></summary>

You cannot patch, secure or monitor servers you do not know about. A dynamic inventory (Ansible `aws_ec2` plugin, cloud tags, Consul) or CMDB records host, role, environment, owner, OS version and kernel. Patching rings, alerts and compliance reports are all driven from those tags.

</details>

### Q65. How do you plan an OS major-version upgrade (e.g. Ubuntu 22.04 → 24.04)? 🔴

<details>
<summary><b>Answer</b></summary>

Prefer replacement over in-place: build a new golden image on the new release, run the app test suite, and roll new instances in while draining old ones. Check for breaking changes (OpenSSL 3, Python version, cgroup v2, iptables→nftables, removed packages, sshd defaults). If in-place is unavoidable (`do-release-upgrade`, or `leapp` for RHEL 8→9), snapshot first, run the pre-upgrade report (`leapp preupgrade`), and upgrade one canary at a time.

</details>

---
## Performance & the USE Method

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q66. What is the USE method? 🟢

<details>
<summary><b>Answer</b></summary>

Brendan Gregg's checklist: for every resource (CPU, memory, disk, network, plus controllers and locks) check **U**tilization (how busy), **S**aturation (queued work it cannot service yet), and **E**rrors. Walking through it systematically finds bottlenecks faster than guessing, and it tells you which tool to run for each cell.

</details>

### Q67. Map the USE method to concrete Linux commands. 🟡

<details>
<summary><b>Answer</b></summary>

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat -P ALL 1` (%usr+%sys) | `vmstat 1` r > nCPU, PSI cpu | `perf stat` / MCE in dmesg |
| Memory | `free -m`, `vmstat` | `si/so`, PSI memory, OOM kills | `dmesg` (EDAC) |
| Disk | `iostat -xz 1` %util | `aqu-sz`, `r_await/w_await` | `smartctl -a`, dmesg I/O errors |
| Network | `sar -n DEV 1` | `ss -s`, drops, retransmits (`nstat`) | `ip -s link`, `ethtool -S` |

</details>

### Q68. What is your "first 60 seconds" checklist on a slow server? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
uptime                 # load trend
dmesg -T | tail        # OOM, I/O errors, throttling
vmstat 1 5             # run queue, swap, iowait
mpstat -P ALL 1 3      # one hot CPU?
pidstat 1 3            # which processes use CPU
iostat -xz 1 3         # disk latency and utilization
free -m
sar -n DEV 1 3         # network throughput
sar -n TCP,ETCP 1 3    # connection rate, retransmits
top                    # overall picture
```

This covers every USE cell before diving deeper.

</details>

### Q69. How do you read vmstat output? 🟡

<details>
<summary><b>Answer</b></summary>

```
procs ---memory--- --swap-- --io-- -system-- ------cpu-----
 r  b  swpd  free  si  so  bi  bo  in  cs us sy id wa st
```

- `r`: runnable tasks; consistently above CPU count = CPU saturation.
- `b`: blocked in uninterruptible sleep (I/O).
- `si`/`so`: swap in/out per second; non-zero sustained = memory pressure.
- `bi`/`bo`: blocks read/written.
- `cs`: context switches; a sudden jump suggests lock contention or too many threads.
- `us sy id wa st`: CPU breakdown.

Ignore the first line; it is the average since boot.

</details>

### Q70. How do you read `iostat -xz 1`? 🟡

<details>
<summary><b>Answer</b></summary>

Key columns per device:

- `r/s`, `w/s`, `rkB/s`, `wkB/s`: IOPS and throughput.
- `r_await`, `w_await`: average ms per I/O including queue time; the latency the app feels.
- `aqu-sz`: average queue length; saturation signal.
- `%util`: time the device was busy. Near 100% matters on a single spinning disk, but NVMe and cloud volumes process many I/Os in parallel, so they can show 100% and still have headroom; trust await and the volume's IOPS limits instead.

`-z` hides idle devices.

</details>

### Q71. What is sar and why install sysstat on every server? 🟢

<details>
<summary><b>Answer</b></summary>

sysstat collects system metrics every 10 minutes (configurable) into `/var/log/sysstat/` or `/var/log/sa/`, so you can look at **history**: what CPU, memory, disk and network looked like at 03:14 last night when the alert fired, even without a monitoring stack.

```bash
sudo systemctl enable --now sysstat
sar -u -f /var/log/sa/sa15 -s 03:00:00 -e 03:30:00
sar -r       # memory
sar -d -p    # disks
sar -n DEV   # network
```

</details>

### Q72. top vs htop vs atop: when do you use each? 🟢

<details>
<summary><b>Answer</b></summary>

- **top**: always installed; press `1` per-CPU, `P`/`M` sort by CPU/memory, `H` threads, `c` full command.
- **htop**: friendlier, tree view (`F5`), per-core bars, easy filtering and killing.
- **atop**: records system and per-process metrics to disk at intervals, including disk and network per process, so you can replay what happened during an incident (`atop -r /var/log/atop/atop_20261108`).

</details>

### Q73. One CPU core is at 100% while the others are idle. What does it suggest? 🟡

<details>
<summary><b>Answer</b></summary>

A single-threaded bottleneck: a single-threaded app (Redis, Node.js event loop, a Python process with the GIL), a hot lock, or interrupt handling pinned to one core (`si`/`hi` high on that core).

```bash
mpstat -P ALL 1
pidstat -t -p <pid> 1          # which thread
cat /proc/interrupts | grep eth0
```

Fixes: run more worker processes, spread NIC queues/IRQs (irqbalance, RSS/RPS), or scale horizontally.

</details>

### Q74. How do you find what code is burning CPU with perf? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
sudo perf top -p <pid>                        # live hottest functions
sudo perf record -F 99 -g -p <pid> -- sleep 30
sudo perf report --stdio | head -50
sudo perf stat -p <pid> -- sleep 10           # cycles, IPC, cache misses
```

For flame graphs: `perf script | stackcollapse-perf.pl | flamegraph.pl > cpu.svg`. JIT runtimes need symbol maps (Java `-XX:+PreserveFramePointer` plus perf-map-agent, Node `--perf-basic-prof`). Sampling at 99 Hz keeps overhead low enough for production.

</details>

### Q75. What are eBPF tools and why are they valuable in production? 🔴

<details>
<summary><b>Answer</b></summary>

eBPF runs verified, sandboxed programs in the kernel with low overhead, so you can trace production safely. bcc and bpftrace tools:

```bash
sudo execsnoop-bpfcc            # every new process (short-lived ones top misses)
sudo opensnoop-bpfcc -p <pid>   # files opened
sudo biolatency-bpfcc           # disk I/O latency histogram
sudo tcpretrans-bpfcc           # TCP retransmits with addresses
sudo runqlat-bpfcc              # CPU scheduler queue latency
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'
```

Tool names have a `-bpfcc` suffix on Debian/Ubuntu.

</details>

### Q76. How do you find which process is doing the most disk I/O? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo iotop -oPa          # only active processes, accumulated
pidstat -d 1             # kB_rd/s, kB_wr/s, iodelay per process
sudo biotop-bpfcc        # per-process block I/O via eBPF
```

`iotop` needs `CONFIG_TASK_IO_ACCOUNTING` and, on newer kernels, `delayacct` enabled (`sysctl kernel.task_delayacct=1`) for the IO% column.

</details>

### Q77. How do you tell whether an app is CPU-bound, I/O-bound or waiting on something else? 🟡

<details>
<summary><b>Answer</b></summary>

Look at the process's state over time:

```bash
pidstat -u -d -w -p <pid> 1    # CPU, I/O, context switches
top -H -p <pid>                # threads in R (CPU) vs S/D
```

Mostly `R` with high %CPU = CPU-bound. Mostly `D` with I/O wait = disk-bound. Mostly `S` with low CPU = waiting on network, locks or downstream services; then look at `strace -c`, application metrics, or `offcputime-bpfcc` to see where it sleeps.

</details>

### Q78. What causes high %sy (system CPU) and how do you investigate? 🔴

<details>
<summary><b>Answer</b></summary>

The kernel is doing a lot of work: very high syscall rates (tiny reads/writes, polling), context switching, page faults and memory compaction, network packet processing, or lock contention in the kernel.

```bash
sudo perf top                   # look for kernel symbols
strace -c -f -p <pid>           # syscall counts
vmstat 1                        # cs and in columns
sar -B 1                        # paging and faults
```

Common fixes: larger I/O buffers, batching, epoll instead of busy polling, disabling THP defrag.

</details>

### Q79. How do you check for and interpret CPU steal time on cloud VMs? 🟡

<details>
<summary><b>Answer</b></summary>

`%st` in `top`, `vmstat` or `mpstat`. Sustained steal above a few percent means the hypervisor is not giving you the CPU you are paying for. On AWS burstable instances (T3/T4g), running out of CPU credits shows as massive steal and throttling; check the `CPUCreditBalance` metric. Fix: unlimited mode, a larger or non-burstable instance type, or dedicated hosts.

</details>

### Q80. How do you determine if a server needs more RAM? 🟡

<details>
<summary><b>Answer</b></summary>

Not by low "free" memory. Signals of real memory shortage:

- `MemAvailable` consistently small relative to working set
- sustained `si`/`so` in `vmstat`, rising major faults (`sar -B` `majflt/s`)
- memory PSI (`/proc/pressure/memory`) `some avg60` rising
- OOM kills in `dmesg`
- page cache too small for the hot data (database read latency climbing)

</details>

### Q81. Should production servers run with swap? 🔴

<details>
<summary><b>Answer</b></summary>

It depends on the workload. A small swap with low swappiness lets the kernel page out truly idle anonymous memory and gives a softer failure mode than instant OOM kills. Latency-critical services prefer predictable OOM behaviour to slow thrashing. Kubernetes historically required swap off; NodeSwap (`LimitedSwap`) is beta since 1.28 and requires cgroup v2. Databases like Elasticsearch recommend disabling swap or locking memory (`bootstrap.memory_lock`). Whatever you choose, alert on swap **activity**, not swap usage.

</details>

### Q82. How do you diagnose a memory leak in a long-running process? 🔴

<details>
<summary><b>Answer</b></summary>

1. Confirm growth over time: `pidstat -r -p <pid> 60` or RSS from monitoring; a sawtooth that returns to baseline is GC, a steady ramp is a leak.
2. Check whether it is heap (`AnonPages`, `smaps_rollup` Anonymous), file mappings, or kernel memory (`slabtop` for `SUnreclaim`).
3. Use language tools: Java heap dumps (`jcmd <pid> GC.heap_dump`), Go `pprof`, Python `tracemalloc`, native `valgrind`/`heaptrack`, or `memleak-bpfcc -p <pid>`.
4. Mitigate while fixing with `MemoryMax=` and restarts, so the leak cannot take down the host.

</details>

### Q83. What are disk I/O schedulers and which should you use? 🔴

<details>
<summary><b>Answer</b></summary>

With blk-mq, the options are `none`, `mq-deadline`, `bfq` and `kyber`.

```bash
cat /sys/block/nvme0n1/queue/scheduler     # [none] mq-deadline
echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler
```

NVMe and cloud virtual disks usually perform best with `none` (the device does its own scheduling). `mq-deadline` is a good default for SATA SSDs and HDDs. `bfq` targets interactive desktops. Persist with a udev rule.

</details>

### Q84. How do you benchmark disk performance correctly? 🔴

<details>
<summary><b>Answer</b></summary>

Use `fio` with a workload that matches the application, not `dd`:

```bash
fio --name=randrw --filename=/data/fio.test --size=4G --direct=1 \
    --rw=randrw --rwmixread=70 --bs=4k --iodepth=32 --numjobs=4 \
    --ioengine=libaio --runtime=60 --time_based --group_reporting
```

`--direct=1` bypasses the page cache. Report IOPS, throughput **and** latency percentiles (p99). On cloud volumes, newly restored EBS volumes need initialization first, and gp3/io2 provisioned IOPS and throughput caps dominate results.

</details>

### Q85. What do dirty page writeback settings control? 🔴

<details>
<summary><b>Answer</b></summary>

- `vm.dirty_background_ratio` / `_bytes`: when background flushing starts.
- `vm.dirty_ratio` / `_bytes`: when writers are forced to flush synchronously (stall).
- `vm.dirty_expire_centisecs`: age before dirty data must be written.

On servers with lots of RAM, percentage-based defaults let gigabytes of dirty data accumulate, then a huge flush stalls I/O for seconds. Setting byte values (e.g. `vm.dirty_background_bytes=268435456`, `vm.dirty_bytes=1073741824`) gives smoother writeback. Watch `Dirty:` and `Writeback:` in `/proc/meminfo`.

</details>

### Q86. How do you check network interface errors and drops? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
ip -s link show eth0             # RX/TX errors, dropped, overruns
ethtool -S eth0 | grep -iE 'drop|err|miss|fifo'
nstat -az | grep -iE 'retrans|drop|overflow'
netstat -s | grep -i -E 'listen|overflow'
```

On AWS ENA, `ethtool -S` exposes `bw_in_allowance_exceeded`, `pps_allowance_exceeded` and `conntrack_allowance_exceeded`, which reveal instance-level network limits that are invisible elsewhere.

</details>

### Q87. How do you measure network throughput and latency between two hosts? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
iperf3 -s                                  # on server
iperf3 -c 10.0.1.20 -P 4 -t 30             # client, 4 parallel streams
iperf3 -c 10.0.1.20 -u -b 1G               # UDP with loss/jitter
ping -c 20 10.0.1.20                       # RTT
mtr -rwc 100 api.example.com               # per-hop loss and latency
```

</details>

### Q88. What is a performance baseline and why do you need one? 🟢

<details>
<summary><b>Answer</b></summary>

A record of what "normal" looks like for each service: CPU, memory, I/O latency, request rate, p99 latency, connections, at different times of day and week. Without it, you cannot tell whether 70% CPU or 5 ms disk latency during an incident is abnormal. Collect continuously with node_exporter/Prometheus or a cloud agent, and keep sysstat locally.

</details>

### Q89. What does tuned do on RHEL? 🟡

<details>
<summary><b>Answer</b></summary>

`tuned` applies profile-based tuning (CPU governor, sysctls, disk scheduler, THP) to match the workload.

```bash
sudo tuned-adm list
sudo tuned-adm active
sudo tuned-adm profile throughput-performance     # or latency-performance, virtual-guest
sudo tuned-adm recommend
```

Custom profiles in `/etc/tuned/<name>/tuned.conf` can inherit (`include=`) and override. Be aware it can override your own sysctl files.

</details>

### Q90. What CPU frequency and power settings matter for latency-sensitive servers? 🔴

<details>
<summary><b>Answer</b></summary>

Power-saving governors and deep C-states add wake-up latency. For low-latency systems on bare metal: `performance` governor, and limit C-states.

```bash
cpupower frequency-info
sudo cpupower frequency-set -g performance
sudo cpupower idle-info
```

Or `tuned-adm profile latency-performance`. On most cloud VMs the hypervisor controls this, so it has little effect; choose instance types instead.

</details>

### Q91. What is CPU pinning/isolation and when do you use it? 🔴

<details>
<summary><b>Answer</b></summary>

Pinning binds a process to specific cores so its cache stays warm and it never competes with noisy processes; isolation removes cores from the general scheduler.

```bash
taskset -cp 2-5 <pid>
# kernel cmdline: isolcpus=2-5 nohz_full=2-5 rcu_nocbs=2-5
```

systemd: `CPUAffinity=2-5`. Kubernetes: the CPU Manager `static` policy with Guaranteed pods. Used for trading systems, packet processing (DPDK) and real-time workloads.

</details>

### Q92. How do you profile why an application is slow but not using CPU? 🔴

<details>
<summary><b>Answer</b></summary>

It is blocked off-CPU: on locks, disk, network or sleeps. Use off-CPU analysis:

```bash
sudo offcputime-bpfcc -p <pid> -f 30 > offcpu.stacks   # folded stacks for flame graph
sudo strace -f -T -p <pid> -e trace=network,read,write   # time per syscall
sudo tcplife-bpfcc                                       # connection durations
```

Then correlate with downstream latency (database slow log, upstream timings, connection pool waits).

</details>

### Q93. What is irqbalance and when should you tune IRQ affinity? 🔴

<details>
<summary><b>Answer</b></summary>

`irqbalance` distributes hardware interrupts across CPUs. On high packet-rate servers (load balancers, proxies), one CPU handling all NIC interrupts saturates with `%si`. Multi-queue NICs with RSS spread queues across cores:

```bash
ethtool -l eth0                     # channels/queues
sudo ethtool -L eth0 combined 8
grep eth0 /proc/interrupts
```

For single-queue virtual NICs, enable RPS (`/sys/class/net/eth0/queues/rx-0/rps_cpus`) to spread processing in software.

</details>

### Q94. How do you capture system state during an intermittent performance issue? 🟡

<details>
<summary><b>Answer</b></summary>

Record continuously so the evidence exists when it happens: atop or sysstat at 1-minute (or finer) intervals, perf/eBPF triggered by a threshold, and application metrics. A simple trigger script:

```bash
while sleep 5; do
  if [ "$(cut -d' ' -f1 /proc/loadavg | cut -d. -f1)" -gt 20 ]; then
    { date; top -b -n1 -H | head -50; ps -eo stat,pid,cmd | awk '$1~/D/'; } >> /var/tmp/spike.log
  fi
done
```

</details>

### Q95. What Linux metrics would you alert on for every server? 🟢

<details>
<summary><b>Answer</b></summary>

- Disk usage and inode usage per filesystem (with predictive alerts)
- Memory available and OOM kill events
- CPU saturation (run queue / PSI) rather than raw utilization
- Disk latency (await) and errors
- Network errors, drops and retransmits
- Host down / node_exporter scrape failures
- Clock offset (time sync)
- Failed systemd units, reboot-required age
- Certificate expiry for services on the host

</details>

### Q96. How do you correctly read CPU usage of a process on a multi-core machine? 🟢

<details>
<summary><b>Answer</b></summary>

`top` and `ps` show per-process %CPU relative to **one** core by default, so a process using four cores fully shows 400%. Press `I` in `top` to toggle Irix mode off, which divides by the number of CPUs. `ps %cpu` is averaged over the whole process lifetime, not the current rate; use `pidstat 1` or `top` for current usage.

</details>

### Q97. How do too many threads or processes hurt performance? 🟡

<details>
<summary><b>Answer</b></summary>

Each thread costs memory (stack, often 8 MiB virtual) and scheduler time. When runnable threads far exceed cores, context switching (`cs` in vmstat, `pidstat -w`) and cache thrashing waste CPU, and lock contention rises. Symptoms: high `%sy`, high `cs`, throughput flat or dropping while CPU climbs. Fix with bounded worker pools sized to cores and I/O behaviour.

</details>

### Q98. How do cgroup limits show up when debugging performance inside containers? 🔴

<details>
<summary><b>Answer</b></summary>

Tools like `top` and `free` inside a container often show **host** totals, not the container's limits. Read the cgroup files instead:

```bash
cat /sys/fs/cgroup/memory.max /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/cpu.max
grep -E 'nr_throttled|throttled_usec' /sys/fs/cgroup/cpu.stat
cat /sys/fs/cgroup/memory.events      # oom_kill count
```

Runtimes that size thread pools or heaps from host CPU/RAM (older JVMs, Node, Go before GOMAXPROCS awareness) misbehave under tight limits.

</details>

### Q99. How do you approach a "server is slow" ticket without a clear symptom? 🟢

<details>
<summary><b>Answer</b></summary>

Clarify first: what is slow (a page, an API, SSH itself?), since when, for everyone or some users, and what changed (deploys, traffic, patches). Then check the resource USE checklist on the host and compare with the baseline, check dependencies (DB, DNS, downstream APIs), and work from evidence. "Slow" is often a downstream dependency or a network issue, not the host.

</details>

### Q100. What is the difference between latency, throughput and utilization, and why does high utilization hurt latency? 🔴

<details>
<summary><b>Answer</b></summary>

Throughput is work per second, latency is time per operation, utilization is the fraction of time a resource is busy. By queueing theory, as utilization approaches 100%, queue length and wait time grow non-linearly (for a simple M/M/1 queue, response time ∝ 1 / (1 − utilization)). At 50% busy a request barely waits; at 90% it waits ~10x the service time. That is why we target 60–70% sustained on latency-sensitive resources and scale before saturation.

</details>

---
## Logs, Limits & High-Traffic Tuning

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. How does log management work on a modern Linux server? 🟢

<details>
<summary><b>Answer</b></summary>

Services log to stdout/stderr (captured by journald) or to files under `/var/log`. journald can forward to rsyslog, which writes classic files (`/var/log/syslog`, `/var/log/messages`, `auth.log`/`secure`) and can forward remotely. An agent (Fluent Bit, Vector, the CloudWatch agent, Elastic Agent) ships logs to a central store (Loki, Elasticsearch/OpenSearch, CloudWatch, Splunk). logrotate and journald size limits keep local disks from filling.

</details>

### Q102. Why centralize logs instead of reading them on each server? 🟢

<details>
<summary><b>Answer</b></summary>

Instances in autoscaling groups disappear with their logs; incidents span many hosts; attackers with root delete local logs; and searching 200 servers with SSH and grep does not scale. Central logging gives retention, search, correlation, alerting and access control without granting shell access.

</details>

### Q103. Write a logrotate config for an application log. 🟢

<details>
<summary><b>Answer</b></summary>

```
# /etc/logrotate.d/orders-api
/var/log/orders-api/*.log {
    daily
    rotate 14
    maxsize 500M
    compress
    delaycompress
    missingok
    notifempty
    create 0640 api adm
    sharedscripts
    postrotate
        systemctl kill -s USR1 orders-api.service >/dev/null 2>&1 || true
    endscript
}
```

```bash
sudo logrotate -d /etc/logrotate.d/orders-api   # dry run
sudo logrotate -f /etc/logrotate.d/orders-api   # force
```

</details>

### Q104. What is the difference between `copytruncate` and `create` + postrotate signal? 🟡

<details>
<summary><b>Answer</b></summary>

With `create`, logrotate renames the file and creates a new one; the app keeps writing to the old (renamed) inode until told to reopen (SIGHUP/USR1, `nginx -s reopen`). With `copytruncate`, logrotate copies the file then truncates the original in place, so the app needs no signal, but lines written between copy and truncate are **lost**, and apps not using `O_APPEND` leave a sparse file with a hole. Prefer reopen signals; use copytruncate only for apps that cannot reopen logs.

</details>

### Q105. Logs rotated but disk usage did not drop. Why? 🟡

<details>
<summary><b>Answer</b></summary>

The process still holds the old (renamed or deleted) file open. `lsof +L1` shows deleted files still open; for renamed files, `lsof -p <pid> | grep log` shows it writing to `app.log.1`. The postrotate step is missing or failed to signal the app. Fix the config, then signal or restart the service.

</details>

### Q106. How often does logrotate run, and why does `maxsize` sometimes not trigger in time? 🟡

<details>
<summary><b>Answer</b></summary>

logrotate is run once a day by `logrotate.timer` (systemd) or `/etc/cron.daily/logrotate`. Size options are only evaluated when it runs, so a log can grow far past `maxsize` between runs. For chatty services, run logrotate hourly for that config (a dedicated timer calling `logrotate /etc/logrotate.d/app` with its own state file `-s`), and set `hourly` in the config, or fix verbosity at the source.

</details>

### Q107. How do you limit journald disk usage? 🟡

<details>
<summary><b>Answer</b></summary>

```ini
# /etc/systemd/journald.conf.d/size.conf
[Journal]
SystemMaxUse=2G
SystemKeepFree=4G
MaxRetentionSec=14day
RateLimitIntervalSec=30s
RateLimitBurst=10000
```

```bash
sudo systemctl restart systemd-journald
journalctl --disk-usage
sudo journalctl --vacuum-size=1G
```

</details>

### Q108. journald shows "Suppressed N messages from unit". What does it mean? 🟡

<details>
<summary><b>Answer</b></summary>

journald's rate limiter dropped messages because a unit exceeded `RateLimitBurst` messages within `RateLimitIntervalSec` (defaults 10000 in 30 s). You are losing logs. Fix the noisy service's log level, raise the limits globally, or per unit with `LogRateLimitIntervalSec=` / `LogRateLimitBurst=` in the service file.

</details>

### Q109. How do you forward logs to a remote syslog server with rsyslog? 🟡

<details>
<summary><b>Answer</b></summary>

```
# /etc/rsyslog.d/90-forward.conf
*.* action(type="omfwd" target="logs.internal" port="514" protocol="tcp"
           queue.type="LinkedList" queue.filename="fwdq"
           queue.maxDiskSpace="1g" queue.saveOnShutdown="on"
           action.resumeRetryCount="-1")
```

The disk-assisted queue buffers messages if the collector is down. Use TCP (or RELP/TLS for reliability and encryption) rather than UDP for anything important.

</details>

### Q110. What is structured logging and why does it matter for Linux services? 🟡

<details>
<summary><b>Answer</b></summary>

Emitting logs as JSON (or key=value) with consistent fields (timestamp in UTC ISO-8601, level, service, request/trace ID, message) makes them machine-parseable without fragile regexes. Central systems can then filter, aggregate and correlate across services. journald itself is structured: `journalctl -o json` exposes fields like `_SYSTEMD_UNIT`, `_PID`, `PRIORITY`.

</details>

### Q111. What should you never put in logs? 🟢

<details>
<summary><b>Answer</b></summary>

Passwords, API keys, tokens, session cookies, private keys, full card numbers, and unnecessary personal data. Logs are copied to many systems with broad access and long retention, so they become a data breach. Mask or drop sensitive fields at the source, add redaction in the shipping pipeline as a second layer, and restrict log file permissions (`0640`).

</details>

### Q112. What are file descriptor limits, and where are they set? 🟢

<details>
<summary><b>Answer</b></summary>

Each process can open only so many files and sockets. There is a per-process soft and hard limit (`RLIMIT_NOFILE`) and a system-wide maximum.

```bash
ulimit -n; ulimit -Hn                 # current shell soft/hard
cat /proc/<pid>/limits | grep 'open files'
cat /proc/sys/fs/file-max             # system-wide max
cat /proc/sys/fs/file-nr              # allocated, unused, max
```

Hitting the limit gives `EMFILE: Too many open files`.

</details>

### Q113. You raised nofile in limits.conf but the service still has 1024. Why? 🟡

<details>
<summary><b>Answer</b></summary>

`/etc/security/limits.conf` is applied by `pam_limits` to **PAM login sessions** (SSH, su, sudo). systemd services do not go through PAM, so they ignore it. Set the limit in the unit:

```ini
[Service]
LimitNOFILE=65536
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart api
grep 'open files' /proc/$(systemctl show -p MainPID --value api)/limits
```

For a global default for all services, set `DefaultLimitNOFILE=` in `/etc/systemd/system.conf`.

</details>

### Q114. Show correct limits.conf syntax. 🟢

<details>
<summary><b>Answer</b></summary>

```
# /etc/security/limits.d/90-app.conf
# <domain>  <type>  <item>   <value>
appuser     soft    nofile   65536
appuser     hard    nofile   131072
@devops     soft    nproc    4096
*           hard    core     0
```

Changes apply to new login sessions only. The soft limit cannot exceed the hard limit; a user can raise their own soft limit up to hard with `ulimit -n`.

</details>

### Q115. How do you find which process is leaking file descriptors? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
for p in /proc/[0-9]*; do echo "$(ls $p/fd 2>/dev/null | wc -l) $(cat $p/comm 2>/dev/null) ${p#/proc/}"; done | sort -rn | head
sudo lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn   # by type
sudo ls -l /proc/<pid>/fd | grep -c socket
```

A count that grows forever, especially sockets in CLOSE_WAIT, means the app is not closing connections. Raising the limit only buys time.

</details>

### Q116. What does `net.core.somaxconn` control? 🟡

<details>
<summary><b>Answer</b></summary>

The maximum length of a listening socket's **accept queue** (fully established connections waiting for the app to `accept()`). The app's `listen(backlog)` is capped at this value. Default is 4096 on kernels 5.4+ (128 before). When it overflows, the kernel drops connections; `nstat -az TcpExtListenOverflows` and `ss -lnt` (Recv-Q vs Send-Q on LISTEN sockets) show it. Raise both the sysctl and the app's backlog (nginx `listen 443 backlog=8192`).

</details>

### Q117. What is `tcp_max_syn_backlog`, and how do SYN cookies relate? 🔴

<details>
<summary><b>Answer</b></summary>

`net.ipv4.tcp_max_syn_backlog` sizes the queue of half-open connections (SYN received, waiting for the final ACK). Under a SYN flood or huge connection bursts it fills; with `net.ipv4.tcp_syncookies=1` the kernel then answers with SYN cookies (encoding state in the sequence number) instead of dropping, so legitimate clients still connect. Logs show `possible SYN flooding on port 443. Sending cookies.` On busy front-ends, raise the backlog and keep syncookies on.

</details>

### Q118. What sysctl settings do you tune on a high-traffic web/proxy server? 🔴

<details>
<summary><b>Answer</b></summary>

```
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 16384
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
fs.file-max = 2097152
net.netfilter.nf_conntrack_max = 1048576
```

Change one thing at a time, measure, and keep it in `/etc/sysctl.d/` under config management. Do not copy lists blindly; defaults are good for most servers.

</details>

### Q119. A proxy runs out of ephemeral ports. How do you fix it? 🔴

<details>
<summary><b>Answer</b></summary>

Symptoms: `connect: Cannot assign requested address` (EADDRNOTAVAIL), tens of thousands of TIME-WAIT sockets to the same upstream IP:port. Each outgoing connection needs a unique (src IP, src port, dst IP, dst port) tuple. Fixes:

1. Reuse connections: HTTP keepalive to upstreams (nginx `keepalive 64;` in the upstream block).
2. Widen `net.ipv4.ip_local_port_range` to `1024 65535`.
3. `net.ipv4.tcp_tw_reuse=1` (safe for outgoing connections).
4. Add more source IPs or upstream addresses.

Never use `tcp_tw_recycle`; it broke clients behind NAT and was removed in kernel 4.12.

</details>

### Q120. What is conntrack and why can it drop packets on busy servers? 🔴

<details>
<summary><b>Answer</b></summary>

netfilter's connection tracking keeps an entry per flow for stateful firewalling and NAT (Docker, Kubernetes, firewalld all use it). When the table is full, new connections are dropped and dmesg shows `nf_conntrack: table full, dropping packet`.

```bash
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
sudo conntrack -S
sudo sysctl -w net.netfilter.nf_conntrack_max=1048576
```

Also consider reducing timeouts (`nf_conntrack_tcp_timeout_established`) or exempting high-volume traffic with `notrack` rules.

</details>

### Q121. What does `net.core.netdev_max_backlog` do? 🔴

<details>
<summary><b>Answer</b></summary>

It is the maximum number of packets queued per CPU when the NIC receives packets faster than the kernel can process them (softirq input queue). Overflows show in the second column of `/proc/net/softnet_stat` (per CPU, hex). Raise it on high packet-rate hosts, together with `net.core.netdev_budget` and spreading load across CPUs (RSS/RPS).

</details>

### Q122. How do you size `vm.max_map_count` and why do Elasticsearch/OpenSearch need it? 🟡

<details>
<summary><b>Answer</b></summary>

`vm.max_map_count` limits the number of memory mappings a process may have (default 65530). Elasticsearch/OpenSearch mmap many index files and refuse to start in production mode below 262144: `max virtual memory areas vm.max_map_count [65530] is too low`.

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-es.conf && sudo sysctl --system
```

</details>

### Q123. What does `fs.inotify.max_user_watches` do and when do you raise it? 🟡

<details>
<summary><b>Answer</b></summary>

It limits how many files/directories a user can watch with inotify. Tools that watch large trees (IDEs, webpack, Filebeat, kubelet on nodes with many pods, Promtail) hit it with `ENOSPC: System limit for number of file watchers reached` or "too many open files" from inotify. Raise `fs.inotify.max_user_watches` (e.g. 524288) and `fs.inotify.max_user_instances` (e.g. 1024).

</details>

### Q124. What is `kernel.pid_max` and `TasksMax`, and why do they matter? 🔴

<details>
<summary><b>Answer</b></summary>

`kernel.pid_max` caps PIDs (threads count too); default is 32768 on small systems and up to 4194304 on large 64-bit ones. systemd also limits tasks per unit (`TasksMax=`, default 15% of the kernel limit via `DefaultTasksMax`), and per user slice. Thread-heavy apps (JVMs with thousands of threads) fail with `fork: Resource temporarily unavailable` or `unable to create native thread` long before CPU or memory is exhausted. Check `systemctl show api -p TasksCurrent -p TasksMax`.

</details>

### Q125. How do you validate that a sysctl tuning change actually helped? 🟡

<details>
<summary><b>Answer</b></summary>

Define the metric first (p99 latency, connection errors, ListenOverflows, retransmits), capture a baseline under representative load, change one parameter, re-run the same load test (wrk, k6, vegeta), and compare. Roll to a canary in production and watch real metrics before rolling out. Record the reason next to the setting in the config file.

</details>

### Q126. What TCP keepalive settings matter for long-lived connections through NAT/LBs? 🔴

<details>
<summary><b>Answer</b></summary>

Cloud NAT gateways and load balancers drop idle flows silently (AWS NAT Gateway after 350 s idle; ALB/NLB have their own idle timeouts). The Linux default `net.ipv4.tcp_keepalive_time=7200` (2 hours) is far too long, so idle DB connections appear alive but are dead. Lower it (e.g. 300 s with `tcp_keepalive_intvl=30`, `tcp_keepalive_probes=5`), and make sure the app enables `SO_KEEPALIVE` (most DB drivers have an option) or uses application-level pings.

</details>

### Q127. What is BBR and when would you enable it? 🔴

<details>
<summary><b>Answer</b></summary>

BBR is a TCP congestion control algorithm from Google that models bottleneck bandwidth and RTT instead of reacting to packet loss like CUBIC (the default). It improves throughput on high-latency or lossy paths, such as serving users over the internet or cross-region transfers.

```
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

```bash
sysctl net.ipv4.tcp_available_congestion_control
```

Test first: within a data centre the gains are small.

</details>

### Q128. How do you protect a host from fork bombs and runaway process counts? 🟡

<details>
<summary><b>Answer</b></summary>

Limit processes per user (`nproc` in limits.conf for login users), per service (`TasksMax=` in systemd units), and for containers (`pids.max` via `--pids-limit` or Kubernetes pod PID limits). systemd's default `TasksMax` on user slices already prevents a single user's fork bomb from taking down the host.

</details>

### Q129. How do you keep core dumps from filling disks while still capturing them? 🔴

<details>
<summary><b>Answer</b></summary>

Use systemd-coredump with size limits:

```ini
# /etc/systemd/coredump.conf.d/limits.conf
[Coredump]
Storage=external
Compress=yes
ProcessSizeMax=2G
ExternalSizeMax=2G
MaxUse=10G
```

```bash
coredumpctl list
coredumpctl info <pid>
coredumpctl debug <pid>      # opens gdb
```

Core dumps can contain secrets from memory, so restrict access, and disable them on hosts handling sensitive data (`fs.suid_dumpable=0`, `* hard core 0`).

</details>

### Q130. What are the risks of setting `vm.swappiness=0` or `vm.overcommit_memory=2` blindly? 🟡

<details>
<summary><b>Answer</b></summary>

`swappiness=0` on modern kernels does not disable swap, but strongly avoids it until memory is nearly exhausted, which can make OOM kills arrive suddenly instead of degrading gracefully. `overcommit_memory=2` with a low `overcommit_ratio` makes `malloc`/`fork` fail early, and many runtimes (JVM, Redis `BGSAVE` fork, Chrome) reserve large virtual ranges they never touch, so they fail to start even with plenty of free RAM. Tune these per workload with testing.

</details>

---
## Disk, Capacity & Backups

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q131. How do you lay out filesystems on a production server? 🟡

<details>
<summary><b>Answer</b></summary>

Separate data with different growth patterns so one cannot fill another: root OS volume, `/var/log` (runaway logs), `/var/lib/<app>` or `/data` on its own volume (databases, can grow and be snapshotted independently), `/tmp` as tmpfs or separate with `noexec,nosuid,nodev`. Use LVM or separate cloud volumes so each can grow online. CIS also recommends separate `/var`, `/var/tmp`, `/var/log/audit` and `/home`.

</details>

### Q132. How do you do capacity planning for disk? 🟡

<details>
<summary><b>Answer</b></summary>

Track usage over time per filesystem (Prometheus `node_filesystem_avail_bytes`), compute growth rate, and forecast days-until-full. Alert on prediction, not only thresholds:

```
predict_linear(node_filesystem_avail_bytes{mountpoint="/data"}[6h], 4*3600) < 0
```

Plan expansions when forecast hits ~70–80%, include inodes, account for growth spikes (backups, reindexing, log bursts), and review quarterly with application owners.

</details>

### Q133. What disk usage alert thresholds do you use? 🟢

<details>
<summary><b>Answer</b></summary>

Warning around 80%, critical around 90%, plus a "will be full in < 4 hours" predictive alert, which catches fast growth that static thresholds miss. Big volumes need absolute free-space thresholds too (10% of 10 TiB is still 1 TiB free). Always alert on inode usage as well.

</details>

### Q134. A disk is 100% full in production. What do you do first? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
df -h; df -i
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail
sudo lsof +L1                                  # deleted-but-open files
sudo journalctl --vacuum-size=500M
sudo find /var/log -name '*.gz' -mtime +7 -delete
```

Free space safely first (old rotated logs, caches, package caches `apt clean`/`dnf clean all`), truncate rather than delete open logs (`: > file`), then grow the volume and fix the root cause (rotation, retention, alerting).

</details>

### Q135. How do you clean up disk space safely on Docker hosts? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
docker system df
docker image prune -a --filter "until=168h"
docker container prune
docker builder prune --keep-storage 10GB
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail
```

Prevent container log growth with `/etc/docker/daemon.json`: `"log-driver": "json-file", "log-opts": {"max-size": "50m", "max-file": "3"}`. Avoid `docker volume prune` unless you are certain no data volumes matter.

</details>

### Q136. What is the 3-2-1 backup rule? 🟢

<details>
<summary><b>Answer</b></summary>

Keep **3** copies of data, on **2** different media/storage types, with **1** off-site. The modern extension is 3-2-1-1-0: one copy immutable or air-gapped (protects against ransomware and compromised admin accounts) and **0** errors in verified restore tests.

</details>

### Q137. How do you use rsync for backups? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
rsync -aHAX --numeric-ids --delete --info=progress2 \
  /srv/data/ backup@nas:/backups/web1/data/
```

`-a` preserves perms, times, symlinks; `-H` hard links; `-A` ACLs; `-X` xattrs (SELinux labels); `--numeric-ids` avoids UID remapping; `--delete` mirrors deletions. A mirror alone is not a backup: a deletion or ransomware encryption is replicated too. Use `--link-dest` snapshots or a versioned tool.

</details>

### Q138. How does `rsync --link-dest` create space-efficient snapshots? 🔴

<details>
<summary><b>Answer</b></summary>

Each run creates a new dated directory; unchanged files are **hard-linked** to the previous snapshot instead of copied, so every snapshot looks like a full backup but only changed files use space.

```bash
today=$(date +%F)
rsync -aH --delete --link-dest=/backups/web1/latest /srv/data/ /backups/web1/$today/
ln -sfn /backups/web1/$today /backups/web1/latest
```

Deleting an old snapshot directory only frees files no other snapshot links to.

</details>

### Q139. Which backup tools do you use beyond rsync, and why? 🟡

<details>
<summary><b>Answer</b></summary>

Deduplicating, encrypted, versioned tools: **restic**, **BorgBackup**, **Kopia**. They chunk files, deduplicate across snapshots, encrypt client-side, support retention policies and verification, and restic/Kopia write directly to S3-compatible storage.

```bash
restic -r s3:s3.amazonaws.com/acme-backups/web1 backup /srv/data
restic -r ... forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune
restic -r ... check
```

</details>

### Q140. Filesystem-level vs block-level (snapshot) backups: trade-offs? 🟡

<details>
<summary><b>Answer</b></summary>

File-level (rsync, restic) gives granular restores of single files and cross-platform portability but is slow on millions of small files. Block-level snapshots (EBS snapshots, LVM, ZFS, SAN) capture an entire volume quickly and incrementally, and restore a whole server fast, but single-file restores need mounting a restored volume. Many teams use snapshots for fast disaster recovery and file/app-level backups for granular recovery and off-site copies.

</details>

### Q141. How do you take a consistent snapshot of a running database volume? 🔴

<details>
<summary><b>Answer</b></summary>

A plain snapshot is crash-consistent at best (like pulling the power). For application consistency:

1. Flush and pause writes: `FLUSH TABLES WITH READ LOCK` (MySQL), `pg_backup_start()` (PostgreSQL 15+), or `fsfreeze -f /data` at the filesystem level.
2. Take the snapshot (EBS, LVM) immediately.
3. Release: `UNLOCK TABLES` / `pg_backup_stop()` / `fsfreeze -u /data`.

Multi-volume databases need multi-volume crash-consistent snapshots (EBS supports this). Usually native tools (`pg_basebackup` + WAL archiving, Percona XtraBackup) are better for point-in-time recovery.

</details>

### Q142. What is the difference between RPO and RTO for server backups? 🟢

<details>
<summary><b>Answer</b></summary>

**RPO** (recovery point objective) is how much data loss is acceptable, which sets backup frequency: hourly backups mean up to 1 hour lost. **RTO** (recovery time objective) is how long recovery may take, which drives the method: restoring 5 TB from object storage takes hours; switching to a replica takes minutes. Define them per system with the business, then design and **test** to meet them.

</details>

### Q143. Why must backups be tested, and how? 🟢

<details>
<summary><b>Answer</b></summary>

An untested backup is a hope, not a backup: jobs fail silently, encryption keys get lost, and restores take far longer than assumed. Automate restore tests: regularly restore to a scratch host, check integrity (checksums, `restic check --read-data-subset`), start the application or database against restored data, run sanity queries, and record the time taken versus RTO. Alert when the last successful backup is older than expected.

</details>

### Q144. How do you protect backups from ransomware or a compromised admin account? 🔴

<details>
<summary><b>Answer</b></summary>

- Immutable storage: S3 Object Lock (compliance mode), immutable Azure blobs, WORM appliances.
- Separate credentials and accounts: the backup target lives in a different cloud account with write-only (append) access from servers; servers cannot delete.
- Pull-based backups where the backup server initiates.
- Encryption with keys stored separately.
- Monitoring for unusual deletions or sudden change rates in backup sizes.

</details>

### Q145. How do you schedule and monitor backup jobs properly? 🟡

<details>
<summary><b>Answer</b></summary>

Run them as systemd timers (with `Persistent=true` and `OnFailure=` to trigger an alert unit) or a central backup scheduler. Emit metrics or a heartbeat on success (push a timestamp to Prometheus Pushgateway or a dead-man's-switch service like Healthchecks), and alert when no successful run happened within the expected window. Alerting only on failures misses jobs that never ran.

</details>

### Q146. How do you plan swap, memory and CPU capacity for a new service? 🟡

<details>
<summary><b>Answer</b></summary>

Load-test the service to find per-request CPU and memory cost and the saturation point, then size for peak traffic plus headroom (often target ~60–70% at peak), add N+1 capacity so losing one node or AZ still handles peak, and validate with production metrics after launch. Revisit after major releases. Right-size downward too; overprovisioning is a cost problem.

</details>

### Q147. How do you monitor disk health on bare metal? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo smartctl -H /dev/sda          # overall health
sudo smartctl -a /dev/sda          # reallocated/pending sectors, CRC errors
sudo nvme smart-log /dev/nvme0     # percentage_used, media_errors
cat /proc/mdstat                   # RAID state
```

Run `smartd` for continuous monitoring with email/alerts, export metrics (smartctl_exporter), and watch dmesg for I/O errors. Rising reallocated or pending sectors mean replace the disk proactively.

</details>

### Q148. What is TRIM/discard and how should it be configured? 🔴

<details>
<summary><b>Answer</b></summary>

TRIM tells SSDs (and thin-provisioned storage) which blocks are no longer in use, preserving performance and reclaiming space on thin volumes. Mounting with `discard` issues TRIM synchronously on every delete, which can add latency. The common practice is periodic batched TRIM:

```bash
sudo systemctl enable --now fstrim.timer   # weekly
sudo fstrim -av
```

For LVM thin pools, also set `issue_discards`/`thin_pool_discards` appropriately.

</details>

### Q149. How do you migrate data to a new disk with minimal downtime? 🔴

<details>
<summary><b>Answer</b></summary>

With LVM, move extents online with no downtime:

```bash
sudo pvcreate /dev/nvme2n1
sudo vgextend vgdata /dev/nvme2n1
sudo pvmove /dev/nvme1n1 /dev/nvme2n1     # online, resumable
sudo vgreduce vgdata /dev/nvme1n1
```

Without LVM: initial `rsync -aHAX` while live, then a short downtime window: stop the app, a final `rsync --delete` (fast, only changes), switch the mount, start the app.

</details>

### Q150. What is your retention policy design for backups? 🟡

<details>
<summary><b>Answer</b></summary>

Grandfather-father-son: e.g. 7 daily, 4 weekly, 12 monthly, sometimes yearly for compliance. Align with RPO, legal/regulatory requirements (and data deletion obligations like GDPR), and cost, moving older backups to cheaper storage classes (S3 Glacier) with lifecycle rules. Document it and enforce it automatically (`restic forget --keep-*`, S3 lifecycle), not by hand.

</details>

---

## User Lifecycle, Auditing & Time Sync

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. How do you manage Linux users across hundreds of servers? 🟢

<details>
<summary><b>Answer</b></summary>

Never create human accounts by hand per server. Options: central identity (FreeIPA, Active Directory or LDAP via SSSD) so accounts and groups live in one directory; SSH certificates from an SSO-backed CA; or an access platform (Teleport, SSM). Config management can manage local accounts for small fleets, but central identity makes onboarding and offboarding a single action.

</details>

### Q152. What is SSSD? 🟡

<details>
<summary><b>Answer</b></summary>

The System Security Services Daemon connects Linux to identity providers (AD, FreeIPA, LDAP, Kerberos). It provides NSS (users/groups via `getent`) and PAM authentication, caches credentials for offline login, and can serve SSH public keys from the directory.

```bash
sudo realm join --user=admin corp.example.com   # AD join via realmd
id alice@corp.example.com
sudo sss_cache -E                                # clear cache after directory changes
```

</details>

### Q153. What should happen in the offboarding process for an engineer with server access? 🟡

<details>
<summary><b>Answer</b></summary>

1. Disable the identity in the central directory/SSO (revokes SSH certificates, VPN, consoles at once).
2. Remove their public keys from any `authorized_keys`, and from CI/deploy systems.
3. Rotate shared secrets they could see (break-glass passwords, API keys, service account keys).
4. Expire local accounts: `usermod -e 1 -L alice`, then archive and remove after a retention period.
5. Reassign file ownership and cron jobs; check for scheduled tasks and running processes (`pgrep -u alice`).
6. Record the action for audit.

</details>

### Q154. How do you safely remove a local user? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo pkill -KILL -u bob                    # end their processes
sudo crontab -r -u bob                     # remove crontab
sudo tar czf /root/archive/bob-home.tgz /home/bob
sudo userdel -r bob                        # remove user, home and mail spool
sudo find / -xdev -nouser -ls 2>/dev/null  # leftover files elsewhere
```

On Debian `deluser --remove-home bob` is the friendlier wrapper.

</details>

### Q155. How should service accounts be created and governed? 🟡

<details>
<summary><b>Answer</b></summary>

One account per service, system UID range, no login shell, no password, home under `/var/lib/<svc>` or none, owning only its own files, and least-privilege sudo (ideally none). Prefer systemd `DynamicUser=` where possible. Track owner and purpose in config management, and review periodically for unused accounts.

```bash
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/orders -M orders
```

</details>

### Q156. How do you review who has access to a server? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
getent passwd | awk -F: '$3>=1000 && $7 !~ /nologin|false/'   # interactive users
getent group sudo wheel                                         # admins
sudo grep -rv '^#' /etc/sudoers /etc/sudoers.d/ | grep -v '^\s*$'
sudo find /home /root -name authorized_keys -exec wc -l {} +
lastlog | grep -v 'Never logged in'
last -n 20
```

Automate this as a periodic access review report for compliance.

</details>

### Q157. What is auditd and what is it used for? 🟢

<details>
<summary><b>Answer</b></summary>

auditd is the userspace daemon for the kernel audit subsystem. It records security-relevant events (file access, syscalls, logins, privilege changes, SELinux denials) to `/var/log/audit/audit.log` according to rules. It is required by PCI-DSS, HIPAA, SOC 2 and CIS controls to answer "who did what, when" on a host.

```bash
sudo auditctl -s       # status
sudo auditctl -l       # loaded rules
```

</details>

### Q158. Write auditd rules to watch sensitive files and privileged commands. 🟡

<details>
<summary><b>Answer</b></summary>

```
# /etc/audit/rules.d/50-custom.rules
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers
-w /etc/ssh/sshd_config -p wa -k sshd
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=4294967295 -k root_cmds
-e 2
```

```bash
sudo augenrules --load
```

`-p wa` = write and attribute changes; `-k` tags events for searching; `-e 2` makes rules immutable until reboot.

</details>

### Q159. How do you search and report on audit logs? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo ausearch -k identity -ts today -i          # -i interprets UIDs and syscalls
sudo ausearch -m USER_LOGIN --success no -ts recent
sudo ausearch -ua alice -ts yesterday
sudo aureport --summary
sudo aureport -au --failed                      # failed authentications
sudo aureport -x --summary                      # executables
```

</details>

### Q160. What is the audit UID (auid / loginuid) and why is it important? 🔴

<details>
<summary><b>Answer</b></summary>

The auid is set at login by `pam_loginuid` and **never changes** for that session, even after `sudo` or `su`. So when alice runs `sudo -i` and then deletes a file as root, audit records `uid=0 auid=1001(alice)`. That is what gives accountability for root actions. auid `4294967295` (-1, "unset") means the process was not started from a login session (daemons).

</details>

### Q161. What are the performance risks of auditd and how do you control them? 🔴

<details>
<summary><b>Answer</b></summary>

Broad syscall rules (e.g. auditing every `execve` or `open` system-wide) on busy servers generate huge volumes and can consume significant CPU. If the backlog fills, the kernel either drops events or, with `-f 2`, panics. Control it:

- Scope rules with filters (`-F auid>=1000`, specific paths, `-F exit=-EACCES`).
- Tune `-b` (backlog) and `--backlog_wait_time`.
- Watch `auditctl -s` for `lost` and `backlog`.
- Ship logs off-host (audisp plugins, Fluent Bit) and rotate (`max_log_file`, `num_logs` in `auditd.conf`).

</details>

### Q162. How do you send audit events to a central SIEM? 🟡

<details>
<summary><b>Answer</b></summary>

Options: the `audisp-remote` plugin to a central auditd, the `syslog` audisp plugin to rsyslog and then to the SIEM, or a log shipper (Fluent Bit, Elastic Agent auditd integration, Wazuh agent) reading `audit.log`. Some teams replace auditd with go-audit or Falco/Tetragon (eBPF) for richer, structured events. The point is that root on the host cannot quietly erase the record.

</details>

### Q163. How do you record full terminal sessions for privileged access? 🔴

<details>
<summary><b>Answer</b></summary>

- sudo I/O logging: `Defaults log_output` with `Defaults!/usr/bin/sudoreplay !log_output`; replay with `sudoreplay -l` and `sudoreplay <id>`.
- `tlog` (RHEL, integrates with SSSD) records sessions to the journal for central playback.
- Access proxies (Teleport, AWS SSM session logging) record at the gateway, which is tamper-resistant from the host.

Tell users their sessions are recorded (banner, policy), and protect the recordings because they may contain secrets.

</details>

### Q164. Why is accurate time critical on Linux servers? 🟢

<details>
<summary><b>Answer</b></summary>

TLS certificate validation, Kerberos (fails beyond 5 minutes skew), distributed databases and consensus systems (etcd, Cassandra), log correlation across hosts, token expiry (JWT, AWS SigV4 rejects requests with large skew), cron schedules and audit evidence all depend on it. Clock drift causes confusing failures that look unrelated to time.

</details>

### Q165. What is chrony and how do you configure it? 🟢

<details>
<summary><b>Answer</b></summary>

chrony is the default NTP implementation on RHEL and many modern distros; it syncs fast after boot and handles intermittent connectivity and VMs well.

```
# /etc/chrony.conf
server 169.254.169.123 prefer iburst minpoll 4 maxpoll 4   # AWS Time Sync
pool 2.pool.ntp.org iburst
makestep 1.0 3
rtcsync
```

```bash
sudo systemctl enable --now chronyd     # 'chrony' on Debian/Ubuntu
```

</details>

### Q166. How do you check that time sync is healthy? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
chronyc tracking        # System time offset, Leap status: Normal
chronyc sources -v      # '^*' marks the selected source
chronyc sourcestats
timedatectl             # "System clock synchronized: yes"
```

Alert when the offset exceeds a threshold (e.g. 100 ms) or no source is reachable. node_exporter exposes `node_timex_offset_seconds` and `node_timex_sync_status`.

</details>

### Q167. What does `makestep` do, and why not always step the clock? 🔴

<details>
<summary><b>Answer</b></summary>

Normally chrony **slews** the clock (speeds it up or slows it slightly) so time stays monotonic. `makestep 1.0 3` allows a jump (step) if the offset is larger than 1 s, but only during the first 3 updates after start, which fixes big offsets at boot. Stepping during runtime can make time go backwards, breaking timeouts, ordering in logs and databases. Force a manual step only when needed: `chronyc makestep`.

</details>

### Q168. Which timezone should production servers use? 🟢

<details>
<summary><b>Answer</b></summary>

UTC. It has no daylight-saving jumps (which can run cron jobs twice or skip them), makes logs across regions directly comparable, and avoids confusion during incidents. Convert to local time in dashboards.

```bash
sudo timedatectl set-timezone UTC
timedatectl
```

</details>

### Q169. How do you handle leap seconds and time smearing? 🔴

<details>
<summary><b>Answer</b></summary>

A leap second inserts 23:59:60, which historically triggered kernel and application bugs. Cloud providers (AWS, Google) **smear** it, adjusting clocks gradually over hours, so their NTP servers never show the leap. Never mix smeared and non-smeared NTP sources in the same fleet, or hosts disagree by up to a second during the smear window. With chrony, `leapsecmode slew` and `maxslewrate` let you smear locally. The CGPM has resolved to stop leap seconds by 2035, but configs still matter until then.

</details>

### Q170. How do you set up an internal NTP hierarchy for an isolated network? 🔴

<details>
<summary><b>Answer</b></summary>

Run two or three internal chrony servers that sync to reliable upstreams (GPS/PTP appliances, cloud time service or public pools through a controlled egress) and serve the network:

```
# on internal NTP servers
pool time.cloudflare.com iburst
allow 10.0.0.0/8
local stratum 10      # keep serving if upstream is lost
```

Clients point at all internal servers (at least 3 for falseticker detection). Monitor offset between them. For microsecond needs (finance, telecom), use PTP (`ptp4l`, `phc2sys`) with hardware timestamping.

</details>

---
## Rapid-Fire Commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q171. Which process is listening on port 8080? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo lsof -iTCP:8080 -sTCP:LISTEN
sudo fuser -v 8080/tcp
```

</details>

### Q172. Show the top 10 memory-consuming processes. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ps -eo pid,user,rss,comm --sort=-rss | head -11
```

RSS is in KiB.

</details>

### Q173. Find files larger than 1 GiB on the root filesystem only. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo find / -xdev -type f -size +1G -exec ls -lh {} + 2>/dev/null
```

`-xdev` keeps it on one filesystem.

</details>

### Q174. Show failed systemd units. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
systemctl --failed
systemctl list-units --state=failed --no-legend
```

</details>

### Q175. Show the last 10 reboots and shutdowns. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
last -x reboot shutdown | head -10
journalctl --list-boots | tail -10
```

</details>

### Q176. Show failed SSH login attempts by source IP. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo journalctl -u ssh -u sshd --since today | grep 'Failed password' \
  | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
sudo lastb | head        # from /var/log/btmp
```

</details>

### Q177. Test whether a remote TCP port is reachable without telnet or nc. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
timeout 3 bash -c '</dev/tcp/db.internal/5432' && echo open || echo closed
curl -v telnet://db.internal:5432 --max-time 3
```

`/dev/tcp` is a bash feature, so it works in minimal images.

</details>

### Q178. Check when a TLS certificate on a remote server expires. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null \
  | openssl x509 -noout -enddate -subject
openssl x509 -in /etc/ssl/certs/app.pem -noout -checkend 2592000   # expires within 30 days?
```

</details>

### Q179. Show all listening services with their systemd unit. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
for pid in $(sudo ss -tlnpH | grep -oP 'pid=\K[0-9]+' | sort -u); do
  printf '%s %s\n' "$pid" "$(ps -o unit= -p "$pid")"
done
```

`ps -o unit=` prints the systemd unit owning a PID.

</details>

### Q180. Count HTTP status codes in an nginx access log. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

Field 9 is the status code in the default `combined` format.

</details>

### Q181. Watch a command's output refresh every 2 seconds. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
watch -n 2 -d 'ss -s; df -h /data'
```

`-d` highlights changes between refreshes.

</details>

### Q182. Show which package version is installed and which is available. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
apt-cache policy openssl       # Debian/Ubuntu: Installed vs Candidate
dnf list openssl --showduplicates   # RHEL
rpm -q openssl
```

</details>

### Q183. List security updates pending. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo dnf updateinfo list --security        # RHEL
sudo dnf upgrade --security                # apply only security
apt list --upgradable 2>/dev/null | grep -i security   # Debian/Ubuntu
sudo pro security-status                   # Ubuntu Pro view
```

</details>

### Q184. See the full command line and start time of a PID. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ps -o pid,lstart,etime,args -p 1234
```

`lstart` is the exact start timestamp; `etime` is elapsed time.

</details>

### Q185. Show the 10 directories using the most space under /var. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo du -xh /var 2>/dev/null | sort -rh | head -10
ncdu -x /var        # interactive, if installed
```

</details>

### Q186. Show a unit's resolved configuration including defaults. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
systemctl show nginx -p Restart -p LimitNOFILE -p MemoryMax -p User
systemctl cat nginx             # file plus drop-ins as written
```

</details>

### Q187. Show memory pressure and CPU pressure right now. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
cat /proc/pressure/memory /proc/pressure/cpu /proc/pressure/io
```

Rising `some avg10` values indicate saturation before it shows elsewhere.

</details>

### Q188. Show TCP retransmission counters system-wide. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
nstat -az TcpRetransSegs TcpOutSegs
sar -n ETCP 1 5          # retrans/s
```

A retransmit rate above ~1% of segments sent points to network loss or congestion.

</details>

### Q189. Generate a random 32-character password. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
openssl rand -base64 24
tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32; echo
```

</details>

### Q190. Copy a large directory between servers fast, preserving everything, resumable. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
rsync -aHAX --partial --info=progress2 -e 'ssh -T -c aes128-gcm@openssh.com' /data/ dst:/data/
```

`--partial` keeps partially transferred files so a rerun resumes.

</details>

### Q191. Check which kernel will boot next on RHEL. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo grubby --default-kernel
sudo grubby --info=ALL | grep -E '^(index|kernel)'
sudo grubby --set-default /boot/vmlinuz-<version>    # pin a specific kernel
```

</details>

### Q192. Show open network connections of a single process. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
sudo lsof -a -p 1234 -i
sudo ss -tnp | grep 'pid=1234,'
```

</details>

### Q193. Measure DNS resolution time from the server. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
dig api.example.com | grep 'Query time'
curl -o /dev/null -s -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} total=%{time_total}\n' https://api.example.com
```

</details>

### Q194. Trace which files a command opens as it starts. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
strace -f -e trace=openat,stat,access -o /tmp/trace.txt ./app
grep -E 'ENOENT|EACCES' /tmp/trace.txt | head
```

Great for "config file not found" or "permission denied" mysteries.

</details>

### Q195. Run a command with lowest CPU and I/O priority. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
nice -n 19 ionice -c3 tar czf /backup/data.tgz /data
```

</details>

### Q196. Show per-CPU softirq activity to spot network processing hot spots. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
watch -d -n1 'grep -E "NET_RX|NET_TX" /proc/softirqs'
mpstat -P ALL 1 | awk '$NF ~ /[0-9]/ {print $3, $(NF-4)}'   # %soft per CPU
```

One CPU with most `NET_RX` means RSS/RPS or IRQ affinity needs tuning.

</details>

### Q197. Freeze and unfreeze a filesystem for a snapshot. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo fsfreeze -f /data
# take EBS/LVM snapshot here
sudo fsfreeze -u /data
```

Writers block while frozen, so keep the window to seconds and always unfreeze in a `trap`.

</details>

### Q198. Find recently modified files in /etc during an incident. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
sudo find /etc -xdev -type f -mmin -120 -printf '%TY-%Tm-%Td %TH:%TM %p\n' | sort
```

Compare with config management state, package verification (`rpm -Va`, `debsums -c`) and audit logs.

</details>

### Q199. Show which cgroup and slice a process belongs to and its limits. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
cat /proc/1234/cgroup                 # 0::/system.slice/api.service
systemctl status 1234                 # unit owning the PID
cat /sys/fs/cgroup/system.slice/api.service/{memory.max,cpu.max,pids.max}
```

</details>

### Q200. Gather a quick system snapshot to attach to an incident ticket. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
{
  date -u; hostnamectl; uptime
  free -m; df -hT; df -i
  vmstat 1 5; iostat -xz 1 3
  ps -eo pid,ppid,user,stat,%cpu,%mem,etime,args --sort=-%cpu | head -25
  ss -s; systemctl --failed
  journalctl -p err -b --since "-1h" --no-pager | tail -100
  dmesg -T | tail -50
} > /var/tmp/snapshot-$(hostname)-$(date +%s).txt 2>&1
```

For deeper collection use `sos report` (RHEL) or `sosreport` on Ubuntu.

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
