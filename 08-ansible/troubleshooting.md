# Ansible — Troubleshooting

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

> Realistic Ansible failures with exact error text: likely causes, how to diagnose, how to fix and how to prevent them · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Connection and SSH Failures](#connection-and-ssh-failures) — Q1–Q25
- [Privilege Escalation and Python Interpreter Issues](#privilege-escalation-and-python-interpreter-issues) — Q26–Q50
- [YAML, Syntax and Jinja2 Errors](#yaml-syntax-and-jinja2-errors) — Q51–Q75
- [Variables, Facts and Precedence Problems](#variables-facts-and-precedence-problems) — Q76–Q100
- [Modules, Idempotency and Handlers](#modules-idempotency-and-handlers) — Q101–Q125
- [Roles, Collections and Galaxy](#roles-collections-and-galaxy) — Q126–Q150
- [Vault, Performance and Execution Behaviour](#vault-performance-and-execution-behaviour) — Q151–Q175
- [Cloud, AWX, Windows and Molecule](#cloud-awx-windows-and-molecule) — Q176–Q200

---

## Connection and SSH Failures

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: Permission denied (publickey,password)."}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong user, wrong or missing private key, key not in `authorized_keys`, or the server disallows the auth method.

**Diagnose →**
```bash
ansible web01 -m ping -vvv
ssh -i ~/.ssh/id_ed25519 -v ec2-user@web01
```
`-vvv` shows the exact `ssh` command Ansible built, including `-o User=` and `IdentityFile`.

**Fix →** Set `ansible_user` and `ansible_ssh_private_key_file` in inventory, or pass `-u` and `--private-key`. Add the public key to `~/.ssh/authorized_keys` with correct permissions (`700` on `.ssh`, `600` on the file).

**Prevent →** Standardise one automation user and key per environment, and test with `ansible all -m ping` in CI.

</details>

### Q2. `UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: ssh: connect to host 10.0.1.15 port 22: Connection timed out"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Security group or firewall blocking port 22, host down, wrong IP, or no route (private subnet without bastion).

**Diagnose →**
```bash
nc -vz 10.0.1.15 22
traceroute -T -p 22 10.0.1.15
ansible-inventory --host web01 | grep ansible_host
```

**Fix →** Open TCP 22 from the controller or bastion, correct `ansible_host`, or add `ansible_ssh_common_args: '-o ProxyJump=bastion'`.

**Prevent →** Keep network access for the control node in IaC, and use dynamic inventory with private IPs.

</details>

### Q3. `UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: ssh: connect to host web01 port 22: Connection refused"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `sshd` is not running, listens on another port, or the host is still booting.

**Diagnose →**
```bash
nc -vz web01 22
ssh -p 2222 user@web01          # test an alternate port
# on the host console: systemctl status sshd; ss -tlnp | grep ssh
```

**Fix →** Start `sshd`, or set `ansible_port` in inventory. After provisioning use `wait_for_connection` or `wait_for` on port 22.

**Prevent →** Add a wait step in provisioning playbooks.

</details>

### Q4. `Host key verification failed.` after rebuilding a server 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host was recreated with a new host key but `known_hosts` has the old one.

**Diagnose →** `ssh-keygen -F web01` and compare the fingerprint.

**Fix →**
```bash
ssh-keygen -R web01
ssh-keyscan -H web01 >> ~/.ssh/known_hosts
```
For short-lived hosts: `ansible_ssh_common_args: '-o StrictHostKeyChecking=accept-new'`.

**Prevent →** Prefer `accept-new` over disabling checks, and publish host keys via SSHFP or your CA. Avoid `host_key_checking = False` in production.

</details>

### Q5. The playbook hangs for ever at "Gathering Facts" on one host 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH waiting for a prompt (new host key confirmation or password), a slow DNS lookup on the target, `sudo` prompting for a password, or a broken shell profile printing output.

**Diagnose →**
```bash
ansible web01 -m setup -vvv
ssh -v web01 true
```
Check `UseDNS` in `sshd_config` and `/etc/profile` or `.bashrc` for interactive commands.

**Fix →** Disable `UseDNS`, make profiles non-interactive (`[[ $- == *i* ]] || return`), set `timeout = 30`, and use `ansible_ssh_common_args` with `-o BatchMode=yes`.

**Prevent →** Use `BatchMode` and a connect timeout in `ansible.cfg`.

</details>

### Q6. `Failed to connect to the host via ssh: Too many authentication failures` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ssh-agent` holds many keys and the server hits `MaxAuthTries` before reaching the right one.

**Diagnose →** `ssh-add -l`, then `ssh -vvv host` to see keys offered.

**Fix →**
```ini
[ssh_connection]
ssh_args = -o IdentitiesOnly=yes
```
and set `ansible_ssh_private_key_file` explicitly.

**Prevent →** Use `IdentitiesOnly=yes` and one key per environment.

</details>

### Q7. `fatal: [web01]: UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: Shared connection to web01 closed."}` mid-run 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The SSH ControlPersist socket is stale, the connection dropped mid-run (NAT idle timeout), or the host rebooted.

**Diagnose →** Look for `ControlPath` sockets in `~/.ansible/cp/` and test `ssh -O check host`.

**Fix →**
```bash
rm -rf ~/.ansible/cp/*
```
and set keepalives: `ssh_args = -o ServerAliveInterval=30 -o ServerAliveCountMax=5 -o ControlMaster=auto -o ControlPersist=60s`.

**Prevent →** Use keepalives on long runs, and the `reboot` module for restarts.

</details>

### Q8. `unix_listener: path "/home/user/.ansible/cp/xxxx.yyyy" too long for Unix domain socket` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The ControlPath is longer than the 104-108 byte socket path limit, common with long hostnames or deep home directories.

**Diagnose →** Check `control_path_dir` and the hostnames in use.

**Fix →**
```ini
[ssh_connection]
control_path_dir = /tmp/.ansible-cp
# or
control_path = %(directory)s/%%C
```

**Prevent →** Use the hashed `%C` token (default in modern versions) and a short directory.

</details>

### Q9. `Failed to connect to the host via ssh: ... Permission denied` only when the playbook runs from CI, not from my laptop 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CI runner lacks the key or `ssh-agent`, key file permissions are too open, or the user name differs.

**Diagnose →**
```bash
ls -l $KEY_FILE            # must be 600
ssh -i $KEY_FILE -o BatchMode=yes -vv user@host true
```

**Fix →** Write the key from a secret to a file with `chmod 600`, and pass `--private-key`. Set `ANSIBLE_REMOTE_USER` explicitly.

**Prevent →** Use the same container image locally and in CI.

</details>

### Q10. `ERROR! Specified inventory, inventory directory, or script does not exist` / `[WARNING]: provided hosts list is empty, only localhost is available` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong `-i` path, wrong working directory, or `inventory =` missing from `ansible.cfg`; a trailing comma is needed for a one-off host list.

**Diagnose →**
```bash
ansible --version | grep 'config file'
ansible-inventory -i inventory.ini --graph
```

**Fix →** Fix the path, or use `-i 'web01,'` (with the comma) for a single host.

**Prevent →** Set `inventory` in a checked-in `ansible.cfg`.

</details>

### Q11. `[WARNING]: Could not match supplied host pattern, ignoring: webservers` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Group does not exist in the loaded inventory, typo or case mismatch, or the dynamic inventory returned nothing.

**Diagnose →**
```bash
ansible-inventory -i inventory --graph
ansible webservers --list-hosts
```

**Fix →** Correct the group name or fix the inventory source.

**Prevent →** In CI run `ansible-playbook ... --list-hosts` first and fail the job when the host count is zero.

</details>

### Q12. `skipping: no hosts matched` and the play does nothing 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `hosts:` pattern empty, `--limit` excludes everything, or the host list resolved to zero because of `:&` logic.

**Diagnose →** `ansible-playbook site.yml --list-hosts`, then remove `--limit` to compare.

**Fix →** Correct the pattern or limit; remember that a limit pattern can only narrow what the play targets.

**Prevent →** Print `--list-hosts` in pipelines before deploying.

</details>

### Q13. `Timeout (12s) waiting for privilege escalation prompt` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `sudo` is asking for a password in a different form than expected, `requiretty` is set, a slow PAM or DNS lookup delays the prompt, or `become_method` does not match the host.

**Diagnose →**
```bash
ssh host 'sudo -n true'
ansible web01 -b -m ping -vvv
```
Check `/etc/sudoers` and `/var/log/auth.log`.

**Fix →** Increase `timeout`, fix DNS (`/etc/hosts` with the hostname), set `ansible_become_method`, or use passwordless sudo for the automation user.

**Prevent →** Make sure the hostname resolves locally on every target.

</details>

### Q14. `msg: "Invalid/incorrect password: Sorry, try again."` when using `-K` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typed the login password instead of the sudo password, the account has no sudo rights, or the password differs per host.

**Diagnose →** `ssh -t host sudo -v` and enter the password manually.

**Fix →** Provide the right password via `-K` or `ansible_become_password` (vaulted).

**Prevent →** Use passwordless sudo with limited commands for the automation account.

</details>

### Q15. SSH works from the shell but Ansible says `UNREACHABLE` with `Could not resolve hostname` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inventory name is not resolvable DNS and `ansible_host` is not set; shell uses `~/.ssh/config` aliases but the inventory uses a different name.

**Diagnose →**
```bash
getent hosts web01
ansible-inventory --host web01 | jq .ansible_host
```

**Fix →** Set `ansible_host: 10.0.1.15`, or add the alias to DNS. Ansible honours `~/.ssh/config` too, but only for the name it connects with.

**Prevent →** Keep `ansible_host` in inventory (or compose it dynamically).

</details>

### Q16. `Failed to connect to the host via ssh: Bad owner or permissions on /home/user/.ssh/config` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH rejects config or key files that are group/world writable or owned by someone else.

**Diagnose →** `ls -la ~/.ssh` and `ssh -vvv host`.

**Fix →**
```bash
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/id_*
chown -R "$USER": ~/.ssh
```

**Prevent →** Do not mount `.ssh` from shared volumes with loose permissions into containers.

</details>

### Q17. `kex_exchange_identification: read: Connection reset by peer` during large runs 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many parallel handshakes exceeding sshd `MaxStartups`, fail2ban banning the controller, or a bastion with connection limits.

**Diagnose →** On the target: `journalctl -u sshd | grep -i 'drop\|exceeded'` and `fail2ban-client status sshd`.

**Fix →** Lower `forks`, enable ControlPersist so handshakes are reused, whitelist the controller in fail2ban, raise `MaxStartups 100:30:200`.

**Prevent →** Size forks to the weakest hop in the path.

</details>

### Q18. `UNREACHABLE! => {"msg": "Failed to connect ... Connection closed by 10.0.1.15 port 22"}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `AllowUsers` or `AllowGroups` excludes the user, TCP wrappers, fail2ban, or unsupported key algorithm (for example ssh-rsa disabled on newer OpenSSH).

**Diagnose →**
```bash
ssh -vvv user@10.0.1.15
# server side
journalctl -u sshd -n 50
```

**Fix →** Add the user to `AllowUsers`, or use an Ed25519 key. For old targets only, `-o PubkeyAcceptedAlgorithms=+ssh-rsa` as a stopgap.

**Prevent →** Keep sshd hardening in a role that always includes the automation user.

</details>

### Q19. `ssh: connect to host ... port 22: Network is unreachable` from inside a container controller 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Container network has no route to the target subnet, VPN not available in the container, or IPv6/IPv4 mismatch.

**Diagnose →** `docker exec ctl ip route`, `ping`/`nc` from inside the container.

**Fix →** Run with `--network host` or attach to the right network, mount VPN config, or use a bastion in the inventory.

**Prevent →** Document the network model for the execution environment.

</details>

### Q20. Playbook works on the first run, later runs fail with `Connection reset by peer` right after a handler restarts `sshd` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `restarted` kills the connection Ansible is using; or a config error prevents sshd from coming back.

**Diagnose →** Check `systemctl status sshd` from the console; verify with `sshd -t`.

**Fix →** Use `state: reloaded`, validate before installing the config, and add `meta: reset_connection` after the change.

```yaml
- ansible.builtin.meta: reset_connection
```

**Prevent →** `validate: sshd -t -f %s` on every sshd config task.

</details>

### Q21. `fatal: [web01]: FAILED! => {"msg": "to use the 'ssh' connection type with passwords, you must install the sshpass program"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Password SSH is requested (`ansible_password` or `-k`) but `sshpass` is missing on the controller.

**Diagnose →** `which sshpass`.

**Fix →** `apt install sshpass` or `dnf install sshpass`, or switch to key auth, which is preferred.

**Prevent →** Use keys; reserve passwords for first-time bootstrapping.

</details>

### Q22. After a reboot task: `fatal: [web01]: FAILED! => {"msg": "Timed out waiting for last boot time check (timeout=600)"}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host takes longer than the timeout to boot, networking or SSH does not come up, or `boot_time_command` fails.

**Diagnose →** Console log, `ansible web01 -m ping` repeatedly, check cloud-init and fsck delays.

**Fix →**
```yaml
- ansible.builtin.reboot:
    reboot_timeout: 1800
    connect_timeout: 10
    post_reboot_delay: 30
```
Fix the slow-boot cause (large fsck, network wait).

**Prevent →** Test reboots in staging and set realistic timeouts.

</details>

### Q23. `fatal: [localhost]: UNREACHABLE! Failed to connect to the host via ssh: Permission denied` when targeting localhost 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A host named `localhost` in your own inventory overrides the implicit local connection and Ansible tries SSH.

**Diagnose →** `ansible-inventory --host localhost` and look for `ansible_connection`.

**Fix →**
```ini
localhost ansible_connection=local
```
or use `connection: local` in the play.

**Prevent →** Always set `connection: local` for control-node plays.

</details>

### Q24. `Failed to connect to the host via ssh: channel 0: open failed: administratively prohibited` when using a bastion 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bastion's sshd has `AllowTcpForwarding no` or `PermitOpen` restrictions, which block ProxyJump or ProxyCommand with `-W`.

**Diagnose →** `ssh -vvv -J bastion target` and inspect the bastion's `sshd_config`.

**Fix →** Allow TCP forwarding for the automation user on the bastion (`AllowTcpForwarding yes` or a `Match User` block), or use SSM Session Manager.

**Prevent →** Include the jump requirements in the bastion build.

</details>

### Q25. `ansible-playbook` prints `[WARNING]: Platform linux on host web01 is using the discovered Python interpreter ... but future installation of another Python interpreter could change the meaning` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interpreter auto-discovery is working but warns it may change.

**Diagnose →** `ansible web01 -m setup -a 'filter=ansible_python*'`.

**Fix →** Pin it per host/group (`ansible_python_interpreter=/usr/bin/python3`), or set `interpreter_python = auto_silent` in `ansible.cfg`.

**Prevent →** Set the interpreter in `group_vars/all.yml`.

</details>

---

## Privilege Escalation and Python Interpreter Issues

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `fatal: [web01]: FAILED! => {"msg": "Missing sudo password"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `become: true` is set but the user needs a password and none was supplied.

**Diagnose →** `ssh host 'sudo -n true'` returns "a password is required".

**Fix →** Run with `-K`, set `ansible_become_password` from Vault, or configure `NOPASSWD` for the automation user.

**Prevent →** Provision a sudoers drop-in at bootstrap: `ansible ALL=(ALL) NOPASSWD: ALL` (or a restricted command list).

</details>

### Q27. `sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No password supplied and `sudo` wants a TTY; sometimes `requiretty` in sudoers.

**Diagnose →** `sudo -l -U ansible` and check for `Defaults requiretty`.

**Fix →** Provide the password, or give `NOPASSWD`. Remove `requiretty` (`Defaults:ansible !requiretty`), especially with pipelining.

**Prevent →** Bake the sudoers entry into your base image.

</details>

### Q28. `sudo: sorry, you must have a tty to run sudo` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `Defaults requiretty` in `/etc/sudoers` (older RHEL/CentOS) and pipelining enabled.

**Diagnose →** `grep requiretty /etc/sudoers /etc/sudoers.d/*`.

**Fix →**
```text
Defaults:ansible !requiretty
```
or disable pipelining.

**Prevent →** Use `visudo -cf` validation when deploying sudoers files.

</details>

### Q29. `FAILED! => {"msg": "Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user (rc: 1, err: chmod: invalid mode: 'A+user:app:rx:allow')"}` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both the login user and `become_user` are unprivileged and the system can't use ACLs (`setfacl` missing or no ACL support on the filesystem).

**Diagnose →** `which setfacl`, `mount | grep acl`.

**Fix →** Install `acl`, enable pipelining (`pipelining = True`), or as a last resort `allow_world_readable_tmpfiles = True`.

**Prevent →** Include `acl` in the base image and enable pipelining.

</details>

### Q30. `"msg": "/bin/sh: 1: /usr/bin/python: not found"` or `"The module failed to execute correctly, you probably need to set the interpreter."` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Target has no Python, or only `python3` at a different path.

**Diagnose →** `ansible web01 -m raw -a 'which python3 python'`.

**Fix →** Bootstrap Python with `raw`, or set `ansible_python_interpreter=/usr/bin/python3`.

```yaml
- ansible.builtin.raw: test -e /usr/bin/python3 || (apt-get update && apt-get install -y python3)
  changed_when: false
```

**Prevent →** Bake Python in the image, or cloud-init it.

</details>

### Q31. `ModuleNotFoundError: No module named 'apt'` / `Could not import python modules: apt, apt_pkg. Please install python3-apt package.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ansible is using a Python interpreter (often a virtualenv or a custom `/usr/local/bin/python3`) that lacks the system `python3-apt`.

**Diagnose →**
```bash
ansible web01 -m setup -a 'filter=ansible_python_interpreter'
ansible web01 -m raw -a '/usr/bin/python3 -c "import apt"'
```

**Fix →** Point to `/usr/bin/python3`, or install `python3-apt` for that interpreter.

**Prevent →** Do not override the interpreter with a venv for system-package tasks.

</details>

### Q32. `FAILED! => {"msg": "The Python 2 yum module is needed for this module"}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Using the `yum` module on a host whose default Python is 3 but `python3-dnf`/`yum` bindings are missing, or on a modern host where `dnf` should be used.

**Diagnose →** `rpm -q python3-dnf`.

**Fix →** Use `ansible.builtin.dnf` (or `package`), and install `python3-dnf`.

**Prevent →** Use `ansible.builtin.package` for portability.

</details>

### Q33. `Unable to encode the result: "AnsiballZ" ... SyntaxError: invalid syntax` when targeting an old CentOS 6 host 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Python on the target is too old for the Ansible version's modules (current `ansible-core` needs Python 3.8+ on targets; Python 2.7 support is gone).

**Diagnose →** `ansible web01 -m raw -a 'python --version'` and `ansible --version`.

**Fix →** Install a newer Python on the host, or use an older `ansible-core` supported for that target, or upgrade the OS.

**Prevent →** Track the target Python support matrix on every Ansible upgrade.

</details>

### Q34. `fatal: [web01]: FAILED! => {"msg": "Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!"}` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SELinux is enabled and the Python SELinux bindings are not available to the interpreter in use.

**Diagnose →** `getenforce`, `rpm -q python3-libselinux`.

**Fix →**
```bash
ansible web01 -b -m ansible.builtin.dnf -a "name=python3-libselinux state=present"
```
Use the system interpreter (not a venv).

**Prevent →** Include `python3-libselinux` in the base image.

</details>

### Q35. `become_method` `su` fails: `Timeout (12s) waiting for privilege escalation prompt` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `su` prompt localised or different, wrong root password, or PAM restrictions.

**Diagnose →** `ansible web01 -b --become-method su -K -m ping -vvv`.

**Fix →** Provide the correct `ansible_become_password`, set `LANG=C` through `ansible_shell_executable`/`environment`, or switch to `sudo`.

**Prevent →** Standardise on sudo.

</details>

### Q36. `FAILED! => {"msg": "Incorrect sudo password"}` for one host only 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host-specific password differs from the rest or account locked after bad attempts.

**Diagnose →** `ssh -t host sudo -v`; check `faillock` or `pam_tally2`.

**Fix →** Set the host's own `ansible_become_password` in `host_vars`, reset the faillock: `faillock --user ansible --reset`.

**Prevent →** Use passwordless sudo and consistent accounts.

</details>

### Q37. `chown: invalid user: 'appuser'` when running the `file` or `copy` module 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user does not exist yet; ordering issue.

**Diagnose →** `getent passwd appuser`.

**Fix →** Create the user earlier in the play (the `user` module before `copy`).

**Prevent →** Put identities first in role task order.

</details>

### Q38. `failed to resolve remote temporary directory from ansible-tmp-...: 'mkdir -p ... ' failed: Permission denied` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Home directory not writable or full disk, read-only filesystem, or `remote_tmp` points to an unwritable location.

**Diagnose →** `df -h ~ /tmp`, `touch ~/.ansible/tmp/x`, check mount flags.

**Fix →** Fix permissions or free disk. Set `remote_tmp = /tmp/.ansible-${USER}/tmp` in `ansible.cfg` when homes are read-only.

**Prevent →** Alert on disk usage and use `remote_tmp` for locked-down hosts.

</details>

### Q39. `Authentication or permission failure. In some cases, you may have been able to authenticate and did not have permissions on the target directory.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Same family as Q38: the remote user cannot create `~/.ansible/tmp`; or SSH auth truly failed.

**Diagnose →** Run with `-vvv`; check the full "mkdir" command and its stderr; `ls -ld ~ ~/.ansible`.

**Fix →** Correct home directory ownership (`chown -R user: ~`), or set `remote_tmp`.

**Prevent →** Do not run Ansible as a user whose home is root-owned.

</details>

### Q40. `The conditional check 'ansible_distribution == "Ubuntu"' failed. The error was: error while evaluating conditional: 'ansible_distribution' is undefined` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `gather_facts: false`, facts failed to gather, or injected facts disabled (`inject_facts_as_vars = False`).

**Diagnose →** `ansible web01 -m setup -a 'filter=ansible_distribution'`.

**Fix →** Enable fact gathering or run `setup`; use `ansible_facts['distribution']`.

**Prevent →** Always use the `ansible_facts[...]` form.

</details>

### Q41. `Shared connection to web01 closed.` and the command output is truncated or garbled after `become` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interactive shell startup scripts, banners, or MOTD printing on non-interactive logins, or a custom login shell.

**Diagnose →** `ssh web01 true | xxd | head` to look for extra output; check `/etc/profile.d`, `.bashrc`, `.bash_profile`.

**Fix →** Remove output from shell profiles for the automation user (guard with an interactive check) and disable banners. Keep the user's shell as `/bin/bash`.

**Prevent →** Lint the base image for profile noise.

</details>

### Q42. `The module failed to execute correctly, you probably need to set the interpreter.` with `/usr/bin/env: 'python': No such file or directory` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Only `python3` exists; module shebang or discovery found no `python`.

**Diagnose →** `ansible web01 -m raw -a 'command -v python python3'`.

**Fix →** `-e ansible_python_interpreter=/usr/bin/python3`, or install Python.

**Prevent →** Use `interpreter_python = auto` and Python 3 images.

</details>

### Q43. `sudo: /etc/sudoers.d/ansible: syntax error near line 1` and now nobody can sudo 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A broken sudoers file deployed without validation.

**Diagnose →** From a root console: `visudo -c`.

**Fix →** Remove or fix the broken file as root (cloud console or rescue mode). Then:

```yaml
- ansible.builtin.copy:
    dest: /etc/sudoers.d/ansible
    content: "ansible ALL=(ALL) NOPASSWD: ALL\n"
    mode: "0440"
    validate: /usr/sbin/visudo -cf %s
```

**Prevent →** Always use `validate` for sudoers and sshd files.

</details>

### Q44. `FAILED! => {"msg": "Failed to get information on remote file (/path): MODULE FAILURE ... Permission denied"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task runs as an unprivileged user and cannot read the path.

**Diagnose →** `ansible web01 -m stat -a path=/path -b` vs. without `-b`.

**Fix →** Add `become: true` to the task (or the required group access).

**Prevent →** Set `become` at the play level when most tasks need root.

</details>

### Q45. `unable to resolve ... ansible_python_interpreter` after switching from Python 2 to 3 on Ubuntu 20.04 hosts: `/usr/bin/python: No such file or directory` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An old inventory still pins `ansible_python_interpreter=/usr/bin/python`.

**Diagnose →** `grep -rn ansible_python_interpreter inventories group_vars host_vars`.

**Fix →** Remove the pin or change it to `/usr/bin/python3`.

**Prevent →** Prefer auto discovery.

</details>

### Q46. Ansible runs, but `become_user: postgres` tasks fail with `could not change directory to "/root": Permission denied` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `psql` runs from the login user's working directory, which the target user cannot access.

**Diagnose →** Run `sudo -u postgres psql -c 'select 1'` from `/root`.

**Fix →**
```yaml
- community.postgresql.postgresql_db: { name: app }
  become: true
  become_user: postgres
  vars:
    ansible_remote_tmp: /tmp/.ansible-postgres
```
or use `chdir: /tmp` on `command` tasks.

**Prevent →** Use `chdir` for commands and community modules for DB work.

</details>

### Q47. `Failed to lock apt for exclusive operation` or `Could not get lock /var/lib/dpkg/lock-frontend` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `unattended-upgrades`, cloud-init or another apt process holds the lock, often right after boot.

**Diagnose →** `fuser -v /var/lib/dpkg/lock-frontend`, `ps aux | grep -E 'apt|dpkg'`.

**Fix →**
```yaml
- ansible.builtin.apt:
    name: nginx
    lock_timeout: 300
```
Or wait for cloud-init: `cloud-init status --wait`.

**Prevent →** Use `lock_timeout` and disable unattended upgrades on build/bootstrap windows.

</details>

### Q48. `E: Unable to locate package nginx` on a fresh Ubuntu host 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The apt cache has never been updated.

**Diagnose →** `apt-cache policy nginx`.

**Fix →**
```yaml
- ansible.builtin.apt:
    name: nginx
    update_cache: true
    cache_valid_time: 3600
```

**Prevent →** Always use `update_cache` with `cache_valid_time` in bootstrap roles.

</details>

### Q49. `No package matching 'httpd' is available` on Ubuntu 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Package name differs by distro (`apache2` on Debian family).

**Diagnose →** `ansible web01 -m setup -a 'filter=ansible_os_family'`.

**Fix →** Use a vars map per OS family and the `package` module.

**Prevent →** Define distro-specific names in `vars/<os_family>.yml`.

</details>

### Q50. `fatal: [web01]: FAILED! => {"msg": "Could not find the requested service nginx: host"}` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The service/unit does not exist (package not installed, or typo) or the init system is not detected (container without systemd).

**Diagnose →** `systemctl list-unit-files | grep nginx`; inside a container check PID 1.

**Fix →** Install the package first; use `ansible.builtin.systemd` explicitly. In containers use a systemd-enabled image or skip service tasks.

**Prevent →** Order tasks install → configure → service; test with Molecule on systemd images.

</details>

---

## YAML, Syntax and Jinja2 Errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. `Syntax Error while loading YAML. mapping values are not allowed in this context` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A colon followed by a space inside an unquoted value (`msg: Status: OK`), or a key indented under a scalar instead of a mapping.

**Diagnose →**
```bash
ansible-playbook site.yml --syntax-check
yamllint site.yml
```
The error prints the file, line and column with a caret under the offending character. The real mistake is often on the line above.

**Fix →** Quote the value: `msg: "Status: OK"`, or fix the indentation so the key belongs to a mapping.

**Prevent →** Run `yamllint` and `ansible-lint` as a pre-commit hook. Quote any string that contains `: `, `#`, `{` or starts with `*`, `&`, `@`.

</details>

### Q52. `found character '\t' that cannot start any token` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tab character. YAML only allows spaces for indentation. This usually comes from an editor that inserts tabs, or from copy-pasting out of a terminal or a wiki.

**Diagnose →**
```bash
grep -nP '\t' roles/web/tasks/main.yml
cat -A roles/web/tasks/main.yml | grep '\^I'
```

**Fix →** Replace the tabs with spaces: `sed -i 's/\t/  /g' file.yml`, then re-check the indentation.

**Prevent →** Add an `.editorconfig` with `indent_style = space` for `*.yml`, and run `yamllint` in CI.

</details>

### Q53. `ERROR! conflicting action statements: ansible.builtin.copy, dest` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module arguments are dedented to the task level, so Ansible sees `copy` and `dest` as two separate keys and tries to treat both as modules. A misspelled task keyword (`notifiy:`, `whn:`) gives the same error, because Ansible treats unknown keys as possible actions.

**Diagnose →** Look at the task the error points to:
```yaml
- name: Copy config
  ansible.builtin.copy:
  src: app.conf        # wrong: same level as the module
  dest: /etc/app.conf
```

**Fix →** Indent the arguments under the module, and fix any misspelled keywords.

**Prevent →** `ansible-lint` catches both problems. Editor YAML schemas (the Red Hat Ansible VS Code extension) flag them as you type.

</details>

### Q54. Ansible says `This one looks easy to fix. It seems that there is a value started with a quote, and the YAML parser is expecting to see the line ended with the same kind of quote.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value starts with a quote but has text after the closing quote:
```yaml
cmd: "/opt/app/bin/start" --port 8080
```

**Diagnose →** `--syntax-check` shows the line. Look for a quote that closes partway through the value.

**Fix →** Quote the whole value: `cmd: "/opt/app/bin/start --port 8080"`, or use single quotes on the outside when the value contains double quotes.

**Prevent →** For long commands, use the YAML folded style (`>-`) or the `argv:` list form of `command`, which avoids quoting completely.

</details>

### Q55. `We could be wrong, but this one looks like it might be an issue with missing quotes. Always quote template expression brackets when they start a value.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A value starts with `{{`. YAML reads `{` as the start of an inline dictionary.
```yaml
dest: {{ app_dir }}/config.yml     # broken
```

**Fix →**
```yaml
dest: "{{ app_dir }}/config.yml"
```

**Prevent →** The rule is simple: if a value starts with `{{`, wrap the whole value in quotes. `ansible-lint` and `yamllint` both catch this.

</details>

### Q56. `ERROR! A malformed block was encountered while loading tasks: {...} should be a list or None but is <class 'ansible.parsing.yaml.objects.AnsibleMapping'>` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A tasks file (`tasks/main.yml`, or a file loaded with `include_tasks`/`import_tasks`) holds a mapping at the top level instead of a list. This usually means the leading `- ` on the first task is missing. A related mistake is putting a whole play (`hosts:`, `tasks:`) into a file that is included as tasks.

**Diagnose →** Open the file the error names. Its first line must be `- name: ...`, not `name: ...`.

**Fix →** Turn the content into a list of tasks. If the file is really a play, load it with `import_playbook` at the playbook level instead of `import_tasks`.

**Prevent →** Keep a clear split: `playbooks/*.yml` hold plays, and `tasks/*.yml` hold task lists only.

</details>

### Q57. `ERROR! 'tsks' is not a valid attribute for a Play` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A typo in a play-level keyword (`tsks`, `handler`, `var`, `gather_fact`), or a task keyword such as `become_pass` used at play level.

**Diagnose →** The error names the bad key. Compare it with the valid play keywords: `ansible-doc -t keyword -l`.

**Fix →** Correct the spelling. For example, use `tasks`, `handlers`, `vars` and `gather_facts`.

**Prevent →** `ansible-lint` with schema validation catches these before they reach a run.

</details>

### Q58. `template error while templating string: expected token 'end of print statement', got 'name'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Invalid Jinja inside `{{ }}`. Common causes are a space in a variable name (`{{ user name }}`), a hyphen (`{{ app-port }}` is read as subtraction), or two expressions with no operator between them.

**Diagnose →** Test the expression on its own:
```bash
ansible localhost -m debug -a 'msg={{ user_name | upper }}' -e user_name=bob
```

**Fix →** Use valid identifiers (`user_name`), use bracket notation for keys with special characters (`{{ item['app-port'] }}`), and join strings with `~`.

**Prevent →** Name variables in `snake_case`. `ansible-lint` rule `var-naming` enforces this.

</details>

### Q59. `The task includes an option with an undefined variable. The error was: 'dict object' has no attribute 'stdout'` after a looped `register` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** When a task runs with `loop`, the registered variable does not have `stdout`. It has a `results` list with one entry per item.

**Diagnose →**
```yaml
- ansible.builtin.debug:
    var: check_out
```
You will see `results: [ {item: ..., stdout: ...}, ... ]`.

**Fix →**
```yaml
- ansible.builtin.debug:
    msg: "{{ check_out.results | map(attribute='stdout') | list }}"
```

**Prevent →** Always `debug` a registered variable once while you develop. Name looped registers in the plural (`checks`) so readers expect a list.

</details>

### Q60. `'item' is undefined` even though the task has a `loop` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `loop:` is indented under the module arguments instead of at task level. Ansible renders the module arguments first, so `{{ item }}` is undefined. If that did not fail, you would see `Unsupported parameters for (ansible.builtin.apt) module: loop`.

**Diagnose →**
```yaml
- ansible.builtin.apt:
    name: "{{ item }}"
    loop: [git, curl]     # wrong level
```

**Fix →** Dedent `loop:` so it lines up with `ansible.builtin.apt:`. Better still, pass the list directly with `name: [git, curl]`, which is faster too.

**Prevent →** `loop`, `when`, `register`, `notify` and `become` are task keywords. They always sit at the same indentation as the module name.

</details>

### Q61. `[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: {{ env }} == 'prod'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `when:`, `failed_when:`, `changed_when:` and `until:` are already Jinja expressions. Wrapping them in `{{ }}` templates the expression twice. Newer ansible-core releases are stricter, so this is a time bomb.

**Fix →**
```yaml
when: env == 'prod'
```
To build a variable name dynamically, use `lookup('vars', 'port_' ~ env)` or `hostvars[inventory_hostname]['port_' ~ env]`, not nested braces.

**Prevent →** `ansible-lint` rule `no-jinja-when` flags this.

</details>

### Q62. `The conditional check 'result.rc == 0' failed. The error was: error while evaluating conditional (result.rc == 0): 'dict object' has no attribute 'rc'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task that registered `result` was skipped (its `when` was false, or it was a `command` running in `--check` mode). A skipped result has `skipped: true` and no `rc`.

**Diagnose →** `debug: var=result` and look for `"skipped": true` or `"msg": "Command would have run if not in check mode"`.

**Fix →**
```yaml
when: result is not skipped and result.rc == 0
# or
when: (result.rc | default(1)) == 0
```
For read-only commands that must run in check mode, add `check_mode: false`.

**Prevent →** Use the test plugins `is succeeded`, `is failed`, `is skipped` and `is changed` instead of raw attributes.

</details>

### Q63. `template error while templating string: No filter named 'json_query'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `json_query` is part of `community.general`, not ansible-core. With a minimal `ansible-core` install, that collection is not present. When the collection is present but the Python library is missing, you get `You need to install "jmespath" prior to running json_query filter` instead.

**Diagnose →**
```bash
ansible-galaxy collection list community.general
python3 -c 'import jmespath'      # on the controller
```

**Fix →** Install `community.general` and `pip install jmespath` into the controller's Python, and call the filter by its FQCN, `community.general.json_query`. Often the cleaner fix is to drop JMESPath and use core filters: `selectattr`, `map(attribute=...)` and `items2dict`.

**Prevent →** Pin collections in `collections/requirements.yml` and Python dependencies in `requirements.txt`, and build both into your execution environment.

</details>

### Q64. A `when: enable_tls` task runs even though the job was started with `-e enable_tls=false` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-e key=value` always passes a **string**. The string `"false"` is not empty, so Jinja treats it as truthy. On ansible-core 2.19+ this is an error instead: `Conditionals must have a boolean result.`

**Diagnose →**
```yaml
- ansible.builtin.debug:
    msg: "{{ enable_tls | type_debug }}"   # prints AnsibleUnicode / str
```

**Fix →**
```bash
ansible-playbook site.yml -e '{"enable_tls": false}'   # JSON keeps the type
```
or in the play: `when: enable_tls | bool`.

**Prevent →** Use `| bool` on every flag that can come from `-e`, a survey or an environment variable, and declare flag types in `meta/argument_specs.yml` so bad input fails early.

</details>

### Q65. `recursive loop detected in template string: {{ app_port | default(8080) }}` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A variable is defined in terms of itself, for example `app_port: "{{ app_port | default(8080) }}"` in `vars:` or `group_vars`. Ansible evaluates variables lazily, so the reference never resolves.

**Diagnose →** Search for the self-reference: `grep -rn 'app_port:.*app_port' .`

**Fix →** Use two names, so that the defaults layer supplies the base value:
```yaml
# roles/app/defaults/main.yml
app_port: 8080
# a computed value gets its own name
app_listen: "0.0.0.0:{{ app_port }}"
```
If you must "add to" a list or dict, build a new variable from the old one (`app_pkgs_final: "{{ app_pkgs + extra_pkgs }}"`).

**Prevent →** Treat variables as immutable bindings, not assignments. Defaults belong in `defaults/main.yml`, not in self-referencing expressions.

</details>

### Q66. `template error while templating string: unexpected char '$' at 18` when deploying Prometheus alert rules 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file contains its own `{{ }}` syntax (Prometheus `{{ $labels.instance }}`, Go templates, Helm charts, GitHub Actions `${{ }}`), and Ansible tries to render it as Jinja.

**Fix →** Choose one of these:
```jinja
{% raw %}summary: "{{ $labels.instance }} is down"{% endraw %}
```
- Use `copy` instead of `template` if the file has no Ansible variables at all.
- Change the delimiters for this one file by putting a header on the first line: `#jinja2: variable_start_string:'[[', variable_end_string:']]'`.
- Mark a variable value as `!unsafe` so it is never templated.

**Prevent →** Keep third-party template files under `files/`, not `templates/`.

</details>

### Q67. `Unexpected end of template. Jinja was looking for the following tags: 'endif' or 'else' or 'elif'. The innermost block that needs to be closed is 'if'.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A missing `{% endif %}` or `{% endfor %}` in a `.j2` file, or an `{% if %}` hidden inside a comment line of the target format.

**Diagnose →** The error names the template. Count the blocks quickly:
```bash
grep -cE '\{%-? *if ' templates/nginx.conf.j2
grep -cE '\{%-? *endif' templates/nginx.conf.j2
```

**Fix →** Close every block. Indent nested blocks inside the template so they are easy to see.

**Prevent →** Render templates in CI with a test playbook (`template` with `delegate_to: localhost` into a temporary directory), or with `ansible-playbook --check --diff` in Molecule.

</details>

### Q68. The rendered Nginx config has random blank lines and broken indentation 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Jinja keeps the newline after each `{% %}` tag and the spaces in front of it. With the `template` module's defaults (`trim_blocks: true`, `lstrip_blocks: false`), indented block tags still leave stray whitespace behind.

**Fix →**
```yaml
- ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    lstrip_blocks: true
```
Or use whitespace control in the template: `{%- for s in servers %}` / `{% endfor -%}`.

**Prevent →** Add `#jinja2: lstrip_blocks: True` as the first line of complex templates, and review the `--diff` output in pull requests.

</details>

### Q69. File permissions come out as `--w----r-T` after setting `mode: 644` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An unquoted `644` is the **decimal** integer 644, which is octal `1204`: sticky bit, owner write, other read. An unquoted `0644` happens to work because YAML 1.1 reads it as octal, but that is fragile.

**Diagnose →** `stat -c '%a %A' /etc/app.conf` prints `1204 --w----r-T`.

**Fix →**
```yaml
mode: "0644"       # or the symbolic form: "u=rw,g=r,o=r"
```

**Prevent →** Always quote modes. `ansible-lint` rules `risky-octal` and `risky-file-permissions` catch this.

</details>

### Q70. A package pinned as `version: 1.10` installs `1.1` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** YAML parses `1.10` as the float `1.1`. Similar traps: `NO` (country code Norway) becomes `false`, `on`/`yes` become `true`, and `012` becomes octal `10`.

**Diagnose →** `debug: msg="{{ app_version }} {{ app_version | type_debug }}"` prints `1.1 float`.

**Fix →** Quote it: `app_version: "1.10"`. Do the same for version numbers, ZIP codes, phone numbers and any string made of digits.

**Prevent →** Enable `yamllint` rule `truthy` with `allowed-values: ['true', 'false']`, and quote every value that is not meant to be a number.

</details>

### Q71. `did not find expected '-' indicator while parsing a block collection` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Items in a list have mixed indentation, or a key line was inserted between list items at the wrong level:
```yaml
tasks:
  - name: one
    ansible.builtin.ping:
   - name: two        # one space off
```

**Diagnose →** The error points to the line where the parser expected a `-`. Compare the column of every `- ` in that list.

**Fix →** Line up all list items at the same column.

**Prevent →** Use 2-space indentation consistently, and run `yamllint -d '{rules: {indentation: {spaces: 2}}}'`.

</details>

### Q72. `ERROR! no module/action detected in task.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The task has a `name`, `when` or `tags` but no module, usually because the module line was deleted or commented out. Older versions said `no action detected in task. This often indicates a misspelled module name, or incorrect module path.`

**Diagnose →** Open the task that the error points to and check it has exactly one module key.

**Fix →** Add the module back, or delete the leftover task. For a no-op placeholder, use `ansible.builtin.meta: noop`.

**Prevent →** Do not leave half-commented tasks in the main branch. Review diffs of task files carefully.

</details>

### Q73. CI fails on ansible-lint with `yaml[truthy]`, `fqcn[action-core]`, `name[missing]` and `no-changed-when` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The lint profile was raised (for example to `production`), or ansible-lint was upgraded and added rules.

**Diagnose →**
```bash
ansible-lint --version
ansible-lint -p        # parseable output, one violation per line
```

**Fix →**
- `yaml[truthy]`: use `true`/`false` instead of `yes`/`no`.
- `fqcn[action-core]`: write `ansible.builtin.copy` instead of `copy`.
- `name[missing]`: give every task a name.
- `no-changed-when`: add `changed_when` to `command` and `shell` tasks.

For a large legacy codebase, run `ansible-lint --fix` first, then review the diff.

**Prevent →** Pin the ansible-lint version in `requirements-dev.txt`, and set the `profile` in `.ansible-lint`. Use `--generate-ignore` to accept the existing violations and stop new ones.

</details>

### Q74. `Unexpected templating type error occurred on ({{ base_port + port_offset }}): unsupported operand type(s) for +: 'int' and 'str'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One of the operands is a string. It came from `-e port_offset=10`, an AWX survey text field, a `lookup('env', ...)` or `stdout`.

**Fix →**
```yaml
listen_port: "{{ base_port | int + port_offset | int }}"
```
For lists, `+` needs a list on both sides: `"{{ list_a + [single_item] }}"`.

**Prevent →** Cast at the boundary where data enters the playbook (`| int`, `| bool`, `| from_json`), and declare `type: int` in `meta/argument_specs.yml`.

</details>

### Q75. `ansible-playbook --syntax-check` passes, but the run fails immediately on an undefined variable 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--syntax-check` only parses YAML and play structure. It does not render templates, resolve variables, run dynamic includes or check that modules exist.

**Diagnose →**
```bash
ansible-playbook site.yml --check --diff -l staging-web01
ansible-inventory -i inventories/prod --host web01 --vars
```

**Fix →** Define the variable where it belongs: role `defaults/` for sensible defaults, or `group_vars` for environment values. Fail early with an assert:
```yaml
- ansible.builtin.assert:
    that: [db_host is defined, db_host | length > 0]
    fail_msg: "db_host must be set in group_vars"
```

**Prevent →** Use a layered CI: `yamllint` → `ansible-lint` → `--syntax-check` → Molecule converge. Add `meta/argument_specs.yml` for required role inputs.

</details>

---

## Variables, Facts and Precedence Problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Variables in `group_vars/webservers.yml` are silently ignored 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ansible only loads `group_vars/` and `host_vars/` from two places: next to the **inventory** source and next to the **playbook**. A `group_vars/` folder anywhere else (repo root, `roles/`) is not read. A filename that does not match the group name exactly (`webserver.yml` against group `webservers`) is also skipped.

**Diagnose →**
```bash
ansible-inventory -i inventories/prod/hosts.yml --host web01 --vars
ansible-inventory -i inventories/prod/hosts.yml --graph
```
If the variable is not shown, Ansible never loaded it.

**Fix →** Move the folder to `inventories/prod/group_vars/webservers.yml`, or next to the playbook. Make the filename match the group name.

**Prevent →** Use the standard layout of one inventory directory per environment, each with its own `group_vars/` and `host_vars/`.

</details>

### Q77. You set `nginx_port: 8443` in inventory `group_vars`, but the role still uses `80` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role defines `nginx_port` in `roles/nginx/vars/main.yml`. Role `vars/` are higher in precedence than inventory `group_vars` and `host_vars`. Only `defaults/main.yml` is lower.

**Diagnose →**
```bash
grep -rn 'nginx_port' roles/nginx/
```

**Fix →** Move tunable values to `roles/nginx/defaults/main.yml`. Keep `vars/main.yml` for internal constants that callers must never change, such as package names per OS.

**Prevent →** Use this convention in code review: anything a user may override goes in `defaults/`, prefixed with the role name.

</details>

### Q78. `set_fact: release_tag=v2` runs, but later tasks still see `v1` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `release_tag` was also passed with `-e` (or comes from an AWX survey or extra vars). Extra vars have the **highest** precedence and override `set_fact`, `register` and everything else.

**Diagnose →** Check the job command line and the AWX extra vars or survey. `debug: var=release_tag` straight after the `set_fact` still shows `v1`.

**Fix →** Use a different name for the computed value (`release_tag_effective`), or stop passing the key as an extra var.

**Prevent →** Reserve extra vars for true run-time inputs, and never reuse those names for internal facts.

</details>

### Q79. `'ansible.vars.hostvars.HostVarsVars object' has no attribute 'ansible_default_ipv4'` when templating `hostvars['db01']` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Facts for `db01` were never gathered in this run. Either `db01` is not in the current play, or `--limit web*` excluded it, so only inventory variables exist for it.

**Diagnose →**
```yaml
- ansible.builtin.debug:
    msg: "{{ hostvars['db01'].keys() | list }}"
```

**Fix →** Gather facts for the database hosts first:
```yaml
- hosts: dbservers
  gather_facts: true
  tasks: []
- hosts: webservers
  tasks: ...
```
Alternatively, use `ansible.builtin.setup` with `delegate_to: "{{ item }}"` and `delegate_facts: true` over `groups['dbservers']`, or turn on a fact cache.

**Prevent →** Put stable addresses in inventory variables (`db_private_ip`) instead of relying on another host's facts.

</details>

### Q80. `'ansible_eth0' is undefined` on new cloud instances 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interface names depend on the hardware and OS: `ens5` on Nitro EC2, `enp0s3` on VirtualBox, `eth0` on older images. Hard-coding `eth0` breaks when the instance type changes.

**Diagnose →** `ansible web01 -m setup -a 'filter=ansible_interfaces'`

**Fix →** Use the interface-independent facts:
```yaml
bind_ip: "{{ ansible_facts['default_ipv4']['address'] }}"
```

**Prevent →** Never reference a fact named after an interface in a role. Use `default_ipv4`, `all_ipv4_addresses`, or inventory variables.

</details>

### Q81. Facts like `ansible_hostname` became undefined after someone changed `ansible.cfg` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `inject_facts_as_vars = False` was set. Facts are then only available under `ansible_facts`, not as top-level `ansible_*` variables. Recent ansible-core releases also warn about relying on the injected names.

**Diagnose →** `ansible-config dump --only-changed | grep -i inject`

**Fix →** Update the code to the namespaced form:
```yaml
"{{ ansible_facts['hostname'] }}"   # not ansible_hostname
"{{ ansible_facts['os_family'] }}"  # not ansible_os_family
```

**Prevent →** Write all new roles with `ansible_facts[...]`. It is future-proof and does not collide with your own variables.

</details>

### Q82. `web01` is in both `prod` and `eu` groups, and `ntp_server` has a different value on every run you check 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both groups define `ntp_server`. For groups at the same level, Ansible merges in **alphabetical** order, so the last one wins (`prod` beats `eu`). A child group beats its parent. A teammate adding a group named `zz_*` changes the result without anyone noticing.

**Diagnose →**
```bash
ansible-inventory --host web01 --vars | jq .ntp_server
ansible-inventory --graph web01
```

**Fix →** Give one owner to each variable. To make the order explicit, use `ansible_group_priority` (higher wins; set it in inventory, not in `group_vars`):
```yaml
eu:
  vars:
    ansible_group_priority: 10
```

**Prevent →** Design groups along separate dimensions (environment, region, role), and never define the same variable in two dimensions.

</details>

### Q83. `group_vars/prod.yml` defines `app_config: {log_level: warn}` and the other keys from `group_vars/all.yml` disappear 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Dictionaries are **replaced**, not merged. The default `hash_behaviour` is `replace`, so the more specific `app_config` replaces the whole dictionary.

**Fix →** Use separate names for each layer and merge them explicitly:
```yaml
# all.yml
app_config_defaults: {log_level: info, workers: 4, port: 8080}
# prod.yml
app_config_overrides: {log_level: warn}
# role
app_config: "{{ app_config_defaults | combine(app_config_overrides | default({}), recursive=true) }}"
```

**Prevent →** Do not set `hash_behaviour = merge` globally. It is deprecated in spirit and changes behaviour everywhere. Prefer flat variables for values that need overriding.

</details>

### Q84. With `gather_facts: false`, a template renders the host's old IP address from last week 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fact caching is on (`fact_caching = jsonfile` or `redis`) with a long `fact_caching_timeout`. The host was rebuilt, but the cache still serves the old facts. `set_fact: ... cacheable: true` values also persist across runs.

**Diagnose →**
```bash
ansible-config dump --only-changed | grep -i fact_caching
ls -l ~/.ansible/fact_cache/web01   # jsonfile backend
```

**Fix →** Run once with `--flush-cache`, or delete that host's cache entry. Then gather fresh facts with `ansible.builtin.setup` for rebuilt hosts.

**Prevent →** Set a sensible `fact_caching_timeout` (for example 3600–86400 seconds). In immutable or auto-scaled fleets, prefer `gathering = smart` with a short timeout, and use `cacheable` only for values that are safe to keep.

</details>

### Q85. A Jenkins job hangs forever at the start of `ansible-playbook` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The play uses `vars_prompt`, or the run passes `--ask-become-pass` or `--ask-vault-pass`, and there is no terminal to answer the prompt.

**Diagnose →** Run the same command locally and see whether it prompts. `grep -rn vars_prompt playbooks/`.

**Fix →** Supply the values without prompting. A `vars_prompt` variable is skipped if it is passed with `-e`. Use `--vault-password-file` or a script, and put the become password in a vaulted `ansible_become_password`.

**Prevent →** Make playbooks non-interactive by default, and add a CI timeout so a hang fails quickly.

</details>

### Q86. `Could not find or access 'prod.yml' Searched in: .../roles/app/vars/prod.yml ...` from `include_vars` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Relative paths in `include_vars` are resolved against the role's `vars/` directory or the playbook's directory, not your current shell directory. The file may also be named after a value that does not exist (`{{ env }}.yml` with `env` unset or misspelled).

**Diagnose →** The error lists every path it searched. Compare them with where the file actually is.

**Fix →** Put the file in `roles/app/vars/`, or use `"{{ playbook_dir }}/vars/{{ env }}.yml"`. For optional files, use the `first_found` lookup with a fallback:
```yaml
- ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files: ["{{ env }}.yml", default.yml]
      paths: ["{{ role_path }}/vars"]
```

**Prevent →** Validate `env` against an allowed list with `assert` before you use it in a path.

</details>

### Q87. `host_vars/web01.yml` is ignored for one host 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The host's inventory name is `web01.example.com` (or an IP address), and the `host_vars` filename has to match the **inventory hostname** exactly, not `ansible_host` or the short name.

**Diagnose →** `ansible-inventory --list | jq '._meta.hostvars | keys'`

**Fix →** Rename the file to `host_vars/web01.example.com.yml`, or change the inventory to use a short alias with `ansible_host` set:
```ini
web01 ansible_host=web01.example.com
```

**Prevent →** Use short, stable aliases as inventory names, and keep FQDNs and IPs in `ansible_host`.

</details>

### Q88. `[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Group names contain hyphens or dots (`web-servers`, `eu-west-1`). Group names must be valid Python identifiers to be safely usable in Jinja. This is common with dynamic inventory groups built from tags or regions.

**Fix →** Rename the groups to `web_servers`. In dynamic inventories, rely on the plugins' default sanitising (`TRANSFORM_INVALID_GROUP_CHARS`) or set `keyed_groups` with a `separator: "_"`.

**Prevent →** Use `snake_case` group names. In templates, access groups as `groups['web_servers']`.

</details>

### Q89. The generated `upstream {}` block contains `web01` instead of IPs, and Nginx cannot resolve the names 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The template uses `inventory_hostname`, which is the inventory alias. The address to connect to is `ansible_host`, and the real interface IP is a fact.

**Fix →**
```jinja
{% for h in groups['app'] %}
server {{ hostvars[h]['ansible_host'] | default(h) }}:8080;
{% endfor %}
```
Or use `hostvars[h]['ansible_facts']['default_ipv4']['address']` if facts for those hosts have been gathered.

**Prevent →** Use `inventory_hostname` for labels and logs, and `ansible_host` or IP facts for network addresses.

</details>

### Q90. A `delegate_to: localhost` task fails with `/usr/bin/python3.9: No such file or directory` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `localhost` is defined explicitly in the inventory, so it belongs to `all` and inherits `ansible_python_interpreter: /usr/bin/python3.9` from `group_vars/all.yml`. That path exists on the targets, but not on the controller or in the execution environment.

**Diagnose →** `ansible-inventory --host localhost --vars | grep python`

**Fix →**
```yaml
# host_vars/localhost.yml
ansible_connection: local
ansible_python_interpreter: "{{ ansible_playbook_python }}"
```
Or remove `localhost` from the inventory and rely on implicit localhost, which already uses `ansible_playbook_python`.

**Prevent →** Set the interpreter per OS group (`rhel8`, `ubuntu`), not in `all`.

</details>

### Q91. A database migration with `run_once: true` ran four times in a rolling deploy with `serial: 25%` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `run_once` means once **per batch**. With `serial`, each batch is a separate pass through the play, so the task runs once per batch.

**Fix →** Move one-time steps into their own play before the rolling play:
```yaml
- name: Migrate once
  hosts: app[0]
  tasks:
    - ansible.builtin.command: /opt/app/bin/migrate
- name: Rolling deploy
  hosts: app
  serial: "25%"
  roles: [app]
```
Or guard the task with `when: inventory_hostname == ansible_play_hosts_all[0]`.

**Prevent →** Make migrations idempotent (track a version in a migrations table), so that a repeated run is harmless anyway.

</details>

### Q92. `ERROR! Invalid extra vars data supplied. 'vars/prod.yml' could not be made into a dictionary` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file path was passed without `@`. `-e vars/prod.yml` is read as an inline string, not as a file.

**Fix →**
```bash
ansible-playbook site.yml -e @vars/prod.yml
```
Also check that the file's top level is a mapping, not a list.

**Prevent →** Wrap the call in a Makefile or CI template so the flags stay consistent.

</details>

### Q93. `{{ proxy_url | default('http://proxy:3128') }}` still renders as an empty string 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `default` only fires when the variable is **undefined**. Here it is defined but empty (`proxy_url: ""` or `null` from an AWX survey or a `group_vars` placeholder).

**Fix →** Pass `true` as the second argument, so falsy values also fall back to the default:
```yaml
"{{ proxy_url | default('http://proxy:3128', true) }}"
```

**Prevent →** Do not define placeholder empty values in `group_vars`. Leave the variable undefined and keep defaults in the role.

</details>

### Q94. A config file ends up containing `__omit_place_holder__3f1c9e...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `omit` was used inside a larger string or a nested structure:
```yaml
extra_args: "--timeout={{ timeout | default(omit) }}"
```
`omit` only works when it is the **entire value** of a module parameter. Anywhere else, its placeholder string leaks into the output.

**Fix →** Build the value conditionally:
```yaml
extra_args: "{{ ('--timeout=' ~ timeout) if timeout is defined else '' }}"
```
For dictionaries, filter out the unwanted keys (`dict2items | rejectattr(...) | items2dict`) instead of putting `omit` inside them.

**Prevent →** Use `default(omit)` only at the top level of module arguments. Search rendered outputs in CI for `__omit_place_holder__`.

</details>

### Q95. `'ansible_mounts' is undefined` after a "speed up facts" change 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `gather_subset` was reduced (for example to `['!all', 'min']` or `['!hardware']`), and mounts, memory and devices are part of the `hardware` subset.

**Diagnose →** `ansible web01 -m setup -a 'gather_subset=!all,min' | grep -c mounts` returns 0.

**Fix →** Add back what the play needs:
```yaml
gather_facts: true
gather_subset: [min, hardware]
```
Or call `ansible.builtin.setup` with that subset right before the task that needs it.

**Prevent →** Document which facts each role relies on, and test roles with the same `gather_subset` that production uses.

</details>

### Q96. A custom fact in `/etc/ansible/facts.d/app.fact` does not appear under `ansible_local` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- The file does not end in `.fact`.
- The file is a static JSON or INI file but has the executable bit set, so Ansible tries to **run** it.
- An executable fact prints something that is not JSON.
- Facts were gathered **before** the task that created the file.

**Diagnose →**
```bash
ansible web01 -m setup -a 'filter=ansible_local'
ls -l /etc/ansible/facts.d/; /etc/ansible/facts.d/app.fact | jq .   # if executable
```
A broken file shows an error string under `ansible_local.app` instead of a dictionary.

**Fix →** Use `mode: "0644"` for static facts and `"0755"` for scripts that print JSON. After deploying the file, refresh facts with `ansible.builtin.setup: filter=ansible_local`.

**Prevent →** Ship custom facts in one role, together with a refresh task.

</details>

### Q97. `{{ ansible_env.HOME }}` resolves to `/root`, and files land in the wrong home directory 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Facts were gathered with `become: true` at play level, so the environment facts describe root. `ansible_env` is a snapshot from fact gathering, not from the current task's user.

**Fix →** Look up the home directory of the user you actually mean:
```yaml
- ansible.builtin.getent:
    database: passwd
    key: deploy
- ansible.builtin.set_fact:
    deploy_home: "{{ ansible_facts.getent_passwd['deploy'][4] }}"
```
Or use `"~deploy"` paths where the module expands `~`.

**Prevent →** Do not depend on `ansible_env` for paths. Make the user's home an explicit role variable.

</details>

### Q98. `could not locate file in lookup: /etc/app/settings.json` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Lookups (`file`, `template`, `env`, `pipe`) always run on the **controller**. The file exists on the remote host, not on the machine running Ansible.

**Fix →** Read remote files with a module:
```yaml
- ansible.builtin.slurp:
    src: /etc/app/settings.json
  register: raw
- ansible.builtin.set_fact:
    settings: "{{ raw.content | b64decode | from_json }}"
```
Use `ansible.builtin.fetch` to copy the file back to the controller.

**Prevent →** Remember the rule: lookups and filters run locally, modules run remotely. The same applies to `lookup('env', 'HOME')`, which returns the controller's environment.

</details>

### Q99. `nginx_worker_processes` is `8` on one host, and nobody knows where it is set 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It is defined in one of more than 20 precedence layers: a role default, a role var, `group_vars`, `host_vars`, inventory, `set_fact`, `include_vars`, or extra vars.

**Diagnose →**
```bash
ansible-inventory -i inventories/prod --host web07 --vars | jq .nginx_worker_processes
grep -rn 'nginx_worker_processes' inventories/ roles/ playbooks/
ansible -i inventories/prod web07 -m ansible.builtin.debug -a var=nginx_worker_processes
```
If inventory does not have it, check `set_fact`, `include_vars` and the AWX extra vars and surveys.

**Fix →** Remove the stray definition, and keep the variable in one intended layer.

**Prevent →** Prefix role variables with the role name, keep overrides in `group_vars` only, and check for duplicate definitions in code review.

</details>

### Q100. `[WARNING]: TASK: app : Configure vhost: The loop variable 'item' is already in use. You should set the `loop_var` value in the `loop_control` option for the task to something else to avoid variable collisions and unexpected behavior.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An outer loop runs `include_tasks` or `include_role`, and the included file loops again. Both use `item`, so the inner loop shadows the outer one.

**Fix →**
```yaml
- ansible.builtin.include_tasks: vhost.yml
  loop: "{{ sites }}"
  loop_control:
    loop_var: site
```
Inside `vhost.yml`, refer to `site.name` and keep `item` for the inner loop.

**Prevent →** Always set `loop_var` when looping over `include_*`. It also makes the included files easier to read.

</details>

---

## Modules, Idempotency and Handlers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `ERROR! The requested handler 'Restart nginx' was not found in either the main handlers list nor in the listening handlers list` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The name in `notify:` does not match the handler's `name:` exactly. The match is case- and space-sensitive (`restart nginx` against `Restart nginx`). Other causes: the handler lives in a different role that is not part of this play, or the role is loaded with `include_role` after the notify fires.

**Diagnose →**
```bash
grep -rn -A1 'handlers' roles/nginx/handlers/main.yml
grep -rn 'notify' roles/ | grep -i nginx
```

**Fix →** Make the names identical. Better, decouple them with `listen`:
```yaml
# handlers/main.yml
- name: Restart nginx service
  ansible.builtin.service: {name: nginx, state: restarted}
  listen: nginx config changed
# task
notify: nginx config changed
```

**Prevent →** Use `listen` topics across roles. Molecule converge catches a missing handler immediately.

</details>

### Q102. A handler named `restart {{ app_service }}` is never found when notified as `restart myapp` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Handler names are templated once, when handlers are loaded, using only the variables available at that point (play vars, role defaults and vars, extra vars). If `app_service` comes from inventory, `set_fact` or `include_vars`, the name does not render the way you expect, and the notify does not match.

**Fix →** Give the handler a static name, template the module arguments instead, and use `listen`:
```yaml
- name: Restart application
  ansible.builtin.systemd:
    name: "{{ app_service }}"
    state: restarted
  listen: app restart needed
```

**Prevent →** Never template handler names or `notify` strings. Keep names static and parameters dynamic.

</details>

### Q103. A `command` task reports `changed` on every run, and the Molecule idempotence check fails 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `command` and `shell` cannot know whether they changed anything, so they always report `changed: true`.

**Fix →** Choose the option that fits:
```yaml
# 1. Run only if an artifact is missing
- ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /var/lib/app/.initialised

# 2. Read-only commands
- ansible.builtin.command: /opt/app/bin/app --version
  register: v
  changed_when: false

# 3. Decide from the output
- ansible.builtin.command: /opt/app/bin/migrate
  register: m
  changed_when: "'No migrations to apply' not in m.stdout"
```

**Prevent →** Prefer a real module when one exists. `ansible-lint` rule `no-changed-when` flags these tasks.

</details>

### Q104. `curl ... | tar xz` fails on the host but the `shell` task reports success 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without `pipefail`, a pipeline returns the exit code of its **last** command. `curl` can fail (404, DNS) while `tar` exits 0 on empty input, or the other way round.

**Fix →**
```yaml
- ansible.builtin.shell: |
    set -o pipefail
    curl -fsSL https://example.com/app.tgz | tar xz -C /opt/app
  args:
    executable: /bin/bash
    creates: /opt/app/bin/app
```
`/bin/sh` on Debian is `dash`, which only supports `pipefail` in recent versions, so set `executable: /bin/bash`. Even better, split the work into `get_url` and `unarchive`.

**Prevent →** `ansible-lint` rule `risky-shell-pipe` flags pipelines without `pipefail`. Use `curl -f` so HTTP errors give a non-zero exit code.

</details>

### Q105. `lineinfile` appends the same line again on every run 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `regexp` does not match the line that `line` writes. For example, the regexp is `^PermitRootLogin yes`, the line written is `PermitRootLogin no`, so the next run finds no match and appends again. Without `regexp`, the comparison is on the exact line, so differences in whitespace also cause duplicates.

**Diagnose →** `grep -n PermitRootLogin /etc/ssh/sshd_config` shows several copies.

**Fix →** The regexp must match both the old and the new form:
```yaml
- ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?\s*PermitRootLogin\s'
    line: PermitRootLogin no
    validate: /usr/sbin/sshd -t -f %s
```
Then remove the existing duplicates once.

**Prevent →** For anything more than a line or two, template the whole file or use `*.d/` drop-in files.

</details>

### Q106. `lineinfile` with `backrefs: true` reports `ok` but the line was never added 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** This is how `backrefs` works: if `regexp` matches nothing, the file is left **unchanged**, and no line is appended.

**Fix →** Use two tasks (one to replace if present, one to add if missing), or drop `backrefs` and write the complete line:
```yaml
- ansible.builtin.lineinfile:
    path: /etc/default/grub
    regexp: '^GRUB_CMDLINE_LINUX='
    line: 'GRUB_CMDLINE_LINUX="quiet audit=1"'
```

**Prevent →** Only use `backrefs` when you are editing part of a line that you know already exists. Assert that it exists first if it matters.

</details>

### Q107. A `template` task reports `changed` and restarts the service on every run 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- The template contains something that changes on every run: `{{ ansible_date_time.iso8601 }}`, `now()`, a random value, or `{{ groups['web'] }}` in non-deterministic order.
- The file is modified after deployment by the application or another tool.
- `owner`/`group`/`mode` differ from what the package resets on update.

**Diagnose →**
```bash
ansible-playbook site.yml -l web01 --check --diff --tags nginx
```
The diff shows exactly which line changes.

**Fix →** Remove the timestamps (use `{{ ansible_managed }}` without a date), sort lists (`| sort`), and seed random values (`random(seed=inventory_hostname)`).

**Prevent →** Run each role twice in Molecule. A second run must show `changed=0`.

</details>

### Q108. `"msg": "Destination directory /etc/myapp does not exist"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `copy` and `template` do not create parent directories.

**Fix →**
```yaml
- ansible.builtin.file:
    path: /etc/myapp
    state: directory
    owner: root
    mode: "0755"
- ansible.builtin.template:
    src: app.yml.j2
    dest: /etc/myapp/app.yml
```

**Prevent →** Order tasks as directories, then files, then services.

</details>

### Q109. `Could not find or access 'nginx.conf.j2' Searched in: .../roles/nginx/templates/nginx.conf.j2 ...` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file sits in the wrong folder (for example `roles/nginx/files/`), the case differs, or `src` includes the folder name (`templates/nginx.conf.j2`), which some layouts tolerate but which breaks when the task is moved.

**Diagnose →** The error lists every path Ansible searched. Compare them with `find roles/nginx -name 'nginx.conf*'`.

**Fix →** Put templates in `roles/<role>/templates/` and static files in `roles/<role>/files/`, and refer to them by filename only.

**Prevent →** Use `ansible-galaxy role init` to create roles with the standard directory layout.

</details>

### Q110. `get_url` fails with `Request failed: <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate>` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A corporate TLS-intercepting proxy or an internal CA (Artifactory, Nexus) whose root certificate is not in the target host's trust store. It can also be an old CA bundle on an old OS.

**Diagnose →**
```bash
curl -v https://artifactory.internal/app.tgz
openssl s_client -connect artifactory.internal:443 -showcerts </dev/null
```

**Fix →** Install the CA on the hosts:
```yaml
- ansible.builtin.copy:
    src: corp-root-ca.crt
    dest: /usr/local/share/ca-certificates/corp-root-ca.crt
  notify: update ca trust       # update-ca-certificates / update-ca-trust
```
Use `ca_path:` on `get_url` and `uri` for one-off cases. Use `validate_certs: false` only in a lab, never in production.

**Prevent →** Make CA trust part of the base role that runs first on every host.

</details>

### Q111. `unarchive` fails with `Failed to find handler for "/tmp/app.tar.gz". Make sure the required command to extract the file is installed.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The target is missing the tool needed to extract the archive (`unzip` for `.zip`, `tar` and `gtar`, `zstd`, `xz`), or the file is not a valid archive at all (an HTML error page saved as `.tar.gz`).

**Diagnose →** `file /tmp/app.tar.gz` and `which unzip tar` on the target.

**Fix →** Install the extractor first (`package: name=unzip`). Download with `get_url` and a `checksum:`, so a bad download fails before `unarchive`. Remember that `remote_src: true` is needed when the archive is already on the target.

**Prevent →** List archive tools in the base role, and always verify checksums of downloaded artifacts.

</details>

### Q112. `template` fails with `"msg": "failed to validate"` and `nginx: [emerg] unknown directive "proxy_passs"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `validate:` worked as designed: the rendered file failed the config test, so Ansible kept the old file in place. The bug is in the template or its variables.

**Diagnose →** The `stderr` in the result shows the validator output. Render the template locally to inspect it.

**Fix →** Fix the template. Keep the validation in place:
```yaml
- ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s
  notify: Reload nginx
```
For included snippets that cannot be tested alone, validate the whole config in a separate `command: nginx -t` before the handler runs.

**Prevent →** Use `validate` on every config file that has a checker: `sshd -t -f %s`, `visudo -cf %s`, `nginx -t`, `haproxy -c -f %s`.

</details>

### Q113. `Unable to start service myapp: Job for myapp.service failed because the control process exited with error code.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The application itself fails to start (bad config, port in use, missing environment), or the unit file was changed but systemd was never reloaded, so it still runs the old definition.

**Diagnose →**
```bash
ansible web01 -b -m command -a 'systemctl status myapp --no-pager'
ansible web01 -b -m command -a 'journalctl -u myapp -n 50 --no-pager'
```

**Fix →** Fix the application error shown in the journal. After changing a unit file, reload systemd:
```yaml
- ansible.builtin.systemd:
    name: myapp
    state: started
    enabled: true
    daemon_reload: true
```

**Prevent →** Notify a handler with `daemon_reload: true` whenever a unit file changes, and add a health check (`uri` or `wait_for`) after starting.

</details>

### Q114. `uri` fails with `Status code was 401 and not [200]: HTTP Error 401: Unauthorized` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No credentials were sent, the token is wrong or expired, or the server expects preemptive Basic auth and `uri` waits for a 401 challenge that never comes.

**Fix →**
```yaml
- ansible.builtin.uri:
    url: https://api.internal/v1/deploy
    method: POST
    url_username: "{{ api_user }}"
    url_password: "{{ api_pass }}"
    force_basic_auth: true
    body_format: json
    body: {version: "{{ app_version }}"}
    status_code: [200, 201]
  no_log: true
```
For bearer tokens, use `headers: {Authorization: "Bearer {{ token }}"}`.

**Prevent →** List every acceptable `status_code` explicitly, and keep tokens in Vault with `no_log`.

</details>

### Q115. `wait_for` fails with `Timeout when waiting for 127.0.0.1:8080` although the app is up 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The application listens on another interface (only the private IP, or only `::1`), on a different port, or takes longer to start than `timeout` allows. If the task has `delegate_to: localhost`, it checks the controller, not the host.

**Diagnose →** `ss -lntp | grep 8080` on the host.

**Fix →**
```yaml
- ansible.builtin.wait_for:
    host: "{{ ansible_host }}"
    port: 8080
    timeout: 120
  delegate_to: localhost      # only if you mean "reachable from outside"
```
For HTTP applications, prefer `uri` with `until`/`retries` against a health endpoint.

**Prevent →** Make the bind address an explicit role variable, and check health rather than just whether a port is open.

</details>

### Q116. `"msg": "async task did not complete within the requested time - 300s"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The job really took longer than `async: 300` (a large backup, an OS upgrade), or it hangs waiting for input (`apt` prompting about a config file, a script waiting on stdin).

**Diagnose →** Log in and look at the process. The async job's output is in `~/.ansible_async/<jid>` of the become user.

**Fix →** Fire the job and check on it separately, with a realistic upper bound:
```yaml
- ansible.builtin.command: /opt/backup/run.sh
  async: 3600
  poll: 0
  register: job
- ansible.builtin.async_status:
    jid: "{{ job.ansible_job_id }}"
  register: st
  until: st.finished
  retries: 120
  delay: 30
```
Make commands non-interactive (`DEBIAN_FRONTEND=noninteractive`, `-y`, `</dev/null`).

**Prevent →** Use async for anything that runs longer than the SSH keepalive window, and clean up with `async_status: mode=cleanup`.

</details>

### Q117. `FAILED - RETRYING: [web01]: Wait for app health (5 retries left).` ends in failure although the app is healthy 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `until` condition is wrong: it checks a key that does not exist, compares a string with an integer, or the task is missing `register`. Another cause is that `uri` itself fails on a non-200 code and never gets to evaluate `until` the way you expect.

**Fix →**
```yaml
- ansible.builtin.uri:
    url: "http://{{ ansible_host }}:8080/health"
    status_code: [200, 503]
    return_content: true
  register: health
  until: health.status == 200 and (health.json.status | default('')) == 'UP'
  retries: 20
  delay: 6
```

**Prevent →** Run the task once with `-vvv` and copy the exact result keys into your condition.

</details>

### Q118. `--check` fails with `Could not find the requested service nginx` on a fresh host 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** In check mode, the package install is only **simulated**. The next task tries to manage a service whose unit file does not exist yet, so it fails.

**Fix →** Let dependent tasks tolerate check mode:
```yaml
- ansible.builtin.service:
    name: nginx
    state: started
  ignore_errors: "{{ ansible_check_mode }}"
```
Or skip them with `when: not ansible_check_mode`.

**Prevent →** Check mode is most useful against hosts that are already configured, for drift detection. Do not expect a clean dry run on a fresh host.

</details>

### Q119. CI logs show database passwords in plain text after adding `--diff` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--diff` prints the before and after content of templated files, including secrets that were rendered from Vault. `no_log` hides the task result, but teams often forget it on templates.

**Fix →** Rotate the leaked credentials. Then turn off the diff for sensitive files:
```yaml
- ansible.builtin.template:
    src: db.conf.j2
    dest: /etc/app/db.conf
    mode: "0600"
  diff: false
  no_log: true
```
Scrub or expire the CI job logs.

**Prevent →** Keep secrets in separate files rendered with `diff: false`, or load them at runtime from a secret manager. Audit for tasks that render vaulted variables without `no_log`.

</details>

### Q120. A task fails with `"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result"` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `no_log` also hides the error message, so you cannot see why the task failed.

**Fix →** Make `no_log` switchable for debugging, and run the debug pass only on a safe environment:
```yaml
no_log: "{{ not (debug_secrets | default(false) | bool) }}"
```
Then run `-e debug_secrets=true -l staging-host`. Alternatively, reproduce the call by hand on the host.

**Prevent →** Put secret-handling tasks in small, separate tasks, so that `no_log` hides as little as possible. Validate inputs with `assert` before the sensitive task.

</details>

### Q121. `file: path=/var/www state=directory owner=www-data recurse=true` takes minutes and shows `changed` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `recurse: true` walks every file under the directory. On large trees (uploads, `node_modules`) this is slow, and any file the application created with a different owner is reported as changed.

**Fix →** Set ownership on the top-level directory only, and let the application create files with the right user (umask, a systemd `User=`). If a one-off recursive fix is needed, run it once behind a flag instead of on every run.

**Prevent →** Avoid `recurse: true` in roles that run regularly. Deploy release directories that already have the correct ownership.

</details>

### Q122. The `user` module reports `changed` on every run for the `password` parameter 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `password_hash('sha512')` generates a **random salt** each run, so the hash is always different from the one in `/etc/shadow`.

**Fix →** Use a stable salt per host, or only set the password when the user is created:
```yaml
- ansible.builtin.user:
    name: deploy
    password: "{{ deploy_password | password_hash('sha512', deploy_salt) }}"
    update_password: on_create
  no_log: true
```
Here `deploy_salt` is a fixed vaulted string of 16 characters or fewer (`[./a-zA-Z0-9]`).

**Prevent →** Prefer SSH keys and disable passwords for service accounts. When passwords are required, store the pre-computed hash in Vault.

</details>

### Q123. `ignore_errors: true` is set, but the play still aborts on that task 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ignore_errors` only covers a task that **ran and returned failed**. It does not cover unreachable hosts, undefined variables in the task's arguments, syntax errors or a missing module.

**Fix →**
- For unreachable hosts, use `ignore_unreachable: true`.
- For undefined variables, add a `default()`, or use `block`/`rescue`, which catches most task-level errors:
```yaml
- block:
    - ansible.builtin.include_tasks: optional.yml
  rescue:
    - ansible.builtin.debug:
        msg: "optional step failed: {{ ansible_failed_result.msg | default('n/a') }}"
```

**Prevent →** Prefer explicit `failed_when` conditions over blanket `ignore_errors`. `ansible-lint` flags `ignore-errors` usage.

</details>

### Q124. `failed_when` with two conditions never fails although `stderr` contains `ERROR` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A list under `failed_when` is combined with **AND**. All items must be true for the task to fail.
```yaml
failed_when:
  - result.rc != 0
  - "'ERROR' in result.stderr"      # both must be true
```

**Fix →** Use `or` explicitly:
```yaml
failed_when: result.rc != 0 or 'ERROR' in result.stderr
```

**Prevent →** Write mixed conditions on one line with clear `and`/`or` and parentheses. Add a Molecule case that forces the failure.

</details>

### Q125. Two `blockinfile` tasks on the same file keep overwriting each other 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Both tasks use the default marker `# {mark} ANSIBLE MANAGED BLOCK`, so the second task replaces the block that the first one wrote, and the next run swaps them back.

**Fix →**
```yaml
- ansible.builtin.blockinfile:
    path: /etc/hosts
    marker: "# {mark} ANSIBLE MANAGED: internal services"
    block: |
      10.0.1.10 db.internal
```
Give each block its own unique marker.

**Prevent →** Always set `marker` explicitly. For more than one or two blocks, template the file or use drop-in directories.

</details>

---

## Roles, Collections and Galaxy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `ERROR! the role 'nginx' was not found in /home/ci/project/playbooks/roles:/home/ci/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:/home/ci/project/playbooks` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role was never installed in CI (`requirements.yml` was not run), the playbook moved into a subfolder so `roles/` is no longer next to it, or `roles_path` in `ansible.cfg` is not loaded.

**Diagnose →** The error prints the exact search path. Run `ansible --version` to see which `ansible.cfg` is in use, and `ansible-config dump | grep -i roles_path`.

**Fix →**
```ini
# ansible.cfg at repo root
[defaults]
roles_path = ./roles:./galaxy_roles
```
```bash
ansible-galaxy role install -r requirements.yml -p galaxy_roles
```

**Prevent →** Pin role sources in `requirements.yml`, and run the install step in every pipeline or bake it into the execution environment.

</details>

### Q127. `ERROR! couldn't resolve module/action 'community.docker.docker_container'. This often indicates a misspelling, missing collection, or incorrect module path.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The collection is not installed for the Python and Ansible being used. Typical situations: a minimal `ansible-core` install, a different virtualenv, or an execution environment that does not include it. A typo in the FQCN gives the same error.

**Diagnose →**
```bash
ansible-galaxy collection list | grep -i docker
ansible-doc community.docker.docker_container | head -5
which ansible-playbook && ansible --version
```

**Fix →**
```yaml
# collections/requirements.yml
collections:
  - name: community.docker
    version: ">=3.10.0,<5.0.0"
```
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

**Prevent →** Keep a version-pinned `collections/requirements.yml` in the repo, and use it in CI, in AWX projects and in `ansible-builder` definitions.

</details>

### Q128. `[WARNING]: The specified collections path '/opt/collections' is not part of the configured Ansible collections paths ... The installed collection will not be picked up in an Ansible run` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The collection was installed with `-p /opt/collections`, but `collections_path` in `ansible.cfg` (or `ANSIBLE_COLLECTIONS_PATH`) does not include that folder.

**Fix →**
```ini
[defaults]
collections_path = ./collections:~/.ansible/collections:/usr/share/ansible/collections
```
Or install into a `collections/` folder next to the playbook, which Ansible always searches.

**Prevent →** Use one documented install path per project, and check with `ansible-galaxy collection list` in CI.

</details>

### Q129. `Unsupported parameters for (community.general.ufw) module: comment. Supported parameters include: ...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The parameter was added in a newer collection version than the one installed. The bundled `ansible` package may carry an older `community.general` than the docs you are reading. An indentation mistake that puts a task keyword under the module gives the same error.

**Diagnose →**
```bash
ansible-galaxy collection list community.general
ansible-doc community.general.ufw | grep -A3 comment
```

**Fix →** Upgrade the collection (`ansible-galaxy collection install community.general --upgrade`, or bump the version pin), or drop the parameter.

**Prevent →** Read the docs matching your installed version (`ansible-doc` is always correct), and pin collection versions.

</details>

### Q130. `ERROR! Failed to resolve the requested dependencies map. Could not satisfy the following requirements: * community.general:>=12.0.0 (direct request)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The version range does not exist on the configured Galaxy server, two requirements conflict (one collection pins `<10`, another needs `>=12`), or an internal mirror has not synced yet.

**Diagnose →**
```bash
ansible-galaxy collection install -r requirements.yml -vvv
ansible-galaxy collection list
```
Look at which server each version was looked up on.

**Fix →** Relax or align the version ranges, or point `server_list` at a source that has the version. Use `--force-with-deps` only after you understand the conflict.

**Prevent →** Update collection versions together in one pull request, and test the install in CI against the same Galaxy or Automation Hub servers production uses.

</details>

### Q131. `ERROR! Unknown error when attempting to call Galaxy at 'https://galaxy.ansible.com/api/': <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed ...>` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A corporate TLS-intercepting proxy, or a controller Python that does not trust the proxy's CA. It can also be a missing `HTTPS_PROXY` on a locked-down network.

**Fix →** Add the corporate CA to the controller's trust store. Python uses certifi or the OS bundle depending on the install, so set `SSL_CERT_FILE` or `REQUESTS_CA_BUNDLE` if needed. Export `HTTPS_PROXY`. On restricted networks, mirror collections into a private Automation Hub or Galaxy NG. Use `--ignore-certs` only for a one-off test.

**Prevent →** Build execution environments with the CA and the collections baked in, so runtime runs never call Galaxy.

</details>

### Q132. Installing from Private Automation Hub fails with `HTTP Code: 401, Message: Unauthorized` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The token is missing or expired, the URL points at the wrong repository (`published` or `validated`), or the server is not listed in `server_list`.

**Fix →**
```ini
[galaxy]
server_list = private_hub, release_galaxy

[galaxy_server.private_hub]
url = https://hub.internal/api/galaxy/content/published/
token = <set via env instead>

[galaxy_server.release_galaxy]
url = https://galaxy.ansible.com/
```
Supply the token through `ANSIBLE_GALAXY_SERVER_PRIVATE_HUB_TOKEN` from the CI secret store.

**Prevent →** Use long-lived service tokens stored in a secret manager, never in the committed `ansible.cfg`.

</details>

### Q133. A role installed from Git is not found by `roles: [nginx]` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Without a `name:`, the role takes the repository name (`ansible-role-nginx`), so `nginx` does not match.

**Diagnose →** `ansible-galaxy role list -p galaxy_roles`

**Fix →**
```yaml
# requirements.yml
roles:
  - name: nginx
    src: git+https://github.com/acme/ansible-role-nginx.git
    version: v2.3.1
```

**Prevent →** Always set `name` and pin `version` to a tag or commit in `requirements.yml`.

</details>

### Q134. A role listed in `meta/main.yml` dependencies runs only once, even though two roles need it with different parameters 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ansible deduplicates role dependencies. A role runs once per play unless it is called with **different parameters** or the role sets `allow_duplicates: true`. The opposite surprise also happens: a dependency runs twice because its parameters differ slightly.

**Fix →** If running it several times is intended, add this to the dependency role:
```yaml
# roles/firewall_rule/meta/main.yml
allow_duplicates: true
```
Otherwise, call it explicitly with `include_role` where it is needed.

**Prevent →** Keep `meta` dependencies for true prerequisites. Make orchestration visible in the playbook instead of hiding it in dependency chains.

</details>

### Q135. `--tags nginx` skips every task inside a role loaded with `include_role`, even though the include is tagged 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tags on a **dynamic** `include_role` or `include_tasks` apply only to the include statement itself, not to the tasks it loads. The include runs, but the tasks inside do not carry the `nginx` tag, so they are filtered out.

**Fix →**
```yaml
- ansible.builtin.include_role:
    name: nginx
    apply:
      tags: [nginx]
  tags: [nginx]
```
Or use `import_role`, which is static and inherits tags to every task.

**Prevent →** Use the rule: `import_*` when you need tags and `--list-tasks`, `include_*` when you need loops or runtime conditions. Check with `ansible-playbook site.yml --tags nginx --list-tasks`.

</details>

### Q136. Half the tasks of an imported role are skipped after the first task runs 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `when` on `import_role` or `import_tasks` is copied onto **every task** inside, and evaluated again for each one. If the role sets the variable used in the condition (for example `when: not app_installed`, and a task sets `app_installed: true`), the remaining tasks are skipped.

**Fix →** Use a dynamic include so the condition is evaluated once:
```yaml
- ansible.builtin.include_role:
    name: app
  when: not app_installed
```
Or use a condition variable that the role does not change.

**Prevent →** Do not let a role change the variables that gate it. Remember: imports copy the condition onto each task, includes evaluate it once.

</details>

### Q137. Role B picks up `listen_port: 9000` from role A, even though role B has its own default 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By default (`private_role_vars = False`), the `vars/` and `defaults/` of a role imported via `roles:` or `import_role` become visible to the rest of the play. Role A's `vars/main.yml` value outranks role B's `defaults/`.

**Fix →** Prefix every role variable with its role name (`rolea_listen_port`, `roleb_listen_port`). If that is not possible, set:
```ini
[defaults]
private_role_vars = True
```
or use `include_role` with `public: false` (the default for `include_role`).

**Prevent →** Enforce `var-naming[no-role-prefix]` in `ansible-lint`.

</details>

### Q138. The play stops at `Validating arguments against arg spec main` with `missing required arguments: nginx_server_name` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role has `meta/argument_specs.yml`, which marks `nginx_server_name` as `required: true`, and the caller did not provide it. This is the check doing its job.

**Fix →** Pass the value through `group_vars`, `vars:` on the role, or `include_role`. If the argument should be optional, give it a `default` in both `argument_specs.yml` and `defaults/main.yml`.
```yaml
argument_specs:
  main:
    options:
      nginx_server_name:
        type: str
        required: true
        description: Public hostname for the vhost
```

**Prevent →** Keep the specs and defaults in sync. `ansible-doc -t role -r roles nginx` shows the documented interface.

</details>

### Q139. `ERROR! [DEPRECATED]: ansible.builtin.include has been removed. Use include_tasks or import_tasks instead.` after upgrading ansible-core 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The bare `include:` keyword was removed in ansible-core 2.16. Old playbooks and Galaxy roles still use it.

**Diagnose →** `grep -rnE '^\s*-?\s*include:' playbooks/ roles/`

**Fix →** Use `ansible.builtin.import_tasks` for static use (tags and `--list-tasks` work), `ansible.builtin.include_tasks` for loops or dynamic filenames, and `ansible.builtin.import_playbook` at play level.

**Prevent →** Before upgrading ansible-core, run the new version in CI with `ansible-lint` and treat deprecation warnings as failures.

</details>

### Q140. `[WARNING]: Collection community.general does not support Ansible version 2.15.13` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The installed collection declares `requires_ansible: ">=2.16.0"` in `meta/runtime.yml`, but the controller or execution environment runs an older ansible-core. The modules may still load, but they can fail in subtle ways.

**Diagnose →** `ansible --version`, `ansible-galaxy collection list community.general`

**Fix →** Upgrade ansible-core, or pin the collection to the last version that supports your core version.

**Prevent →** Pin ansible-core and the collections together in one execution environment definition, and upgrade them as a set.

</details>

### Q141. `Failed to import the required Python library (psycopg2) on db01's Python /usr/bin/python3. Please read the module documentation and install it in the appropriate location.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Collection modules run on the **target** host and need their Python dependencies installed there, for the interpreter Ansible uses. It may be installed for a different Python (`python3.11` against `/usr/bin/python3`).

**Fix →**
```yaml
- ansible.builtin.package:
    name: python3-psycopg2
    state: present
```
Install it before any `community.postgresql` task, or set `ansible_python_interpreter` to the Python that has the library.

**Prevent →** Give each role a "dependencies" task at the top that installs the module libraries it needs.

</details>

### Q142. `[WARNING]: Ansible is being run in a world writable directory (/vagrant), ignoring it as an ansible.cfg source.` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The project folder is world-writable (Vagrant shared folders, WSL `/mnt/c`, a mounted volume in CI). For security, Ansible refuses to load `ansible.cfg` from it, so `roles_path`, `inventory` and other settings are silently lost.

**Fix →** Remove world-write permission (`chmod o-w .`), mount with proper permissions (for WSL, `metadata` mount options), or point to the file explicitly with `export ANSIBLE_CONFIG=$PWD/ansible.cfg`.

**Prevent →** Check `ansible --version` for the line `config file = ...` at the start of CI jobs.

</details>

### Q143. `ansible-galaxy collection build` rejects the collection name `acme.web-tools` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Collection namespaces and names may only contain lowercase letters, digits and underscores, and cannot start with a digit or an underscore. Hyphens are not allowed.

**Fix →** In `galaxy.yml`, set `namespace: acme` and `name: web_tools`, and refer to content as `acme.web_tools.<module>`.

**Prevent →** Scaffold with `ansible-galaxy collection init acme.web_tools`, which validates the name.

</details>

### Q144. A role fails on Rocky Linux 9 with `Could not find or access 'Rocky.yml'` from an OS-specific include 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role includes `{{ ansible_facts['distribution'] }}.yml`, and only `RedHat.yml`, `CentOS.yml` and `Ubuntu.yml` exist.

**Fix →** Fall back from the most specific file to the most generic:
```yaml
- ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['distribution'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
        - default.yml
      paths: ["{{ role_path }}/vars"]
```

**Prevent →** Key OS differences on `os_family` first, and test new distributions in the Molecule platform matrix.

</details>

### Q145. A role fix is merged, but `ansible-galaxy role install -r requirements.yml` keeps running the old code 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The role is already installed, and `ansible-galaxy` skips it with `- nginx is already installed, skipping.` unless the version changed or you force the install. Pinning a moving branch makes this worse.

**Fix →** Bump the `version:` to the new tag, or run `ansible-galaxy role install -r requirements.yml --force`.

**Prevent →** Pin roles to immutable tags or commits. In CI, install into a clean directory on every run.

</details>

### Q146. A task under `tasks:` installs a prerequisite, but the role that needs it fails first 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Play order is fixed: `pre_tasks` → their handlers → `roles` → `tasks` → their handlers → `post_tasks`. Everything in `roles:` runs **before** `tasks:`, whatever order the file is written in.

**Fix →** Move the prerequisite to `pre_tasks`, or place the role exactly where you need it with `import_role` inside `tasks:`.

**Prevent →** In a play, use either `roles:` or `tasks:` with `import_role`, not both, so the order you read is the order that runs.

</details>

### Q147. Collection installs started failing signature verification after `--keyring` was enabled in CI 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The keyring is missing the publisher's public key (for example, the Red Hat or internal Automation Hub signing key), the keyring path in the CI image is wrong, or the server does not serve signatures for some collections. `GALAXY_REQUIRED_VALID_SIGNATURE_COUNT` then fails the install.

**Diagnose →**
```bash
gpg --no-default-keyring --keyring ~/.ansible/pubring.kbx --list-keys
ansible-galaxy collection verify acme.web_tools --keyring ~/.ansible/pubring.kbx -vvv
```

**Fix →** Import the correct signing key into the keyring that the CI image uses. Set `GALAXY_REQUIRED_VALID_SIGNATURE_COUNT=+1` only for sources that sign everything. Use `--ignore-signature-status-code` only for specific known status codes, never as a blanket bypass.

**Prevent →** Bake keys and the verification settings into the execution environment build, and verify during the build, not at runtime.

</details>

### Q148. A custom collection module fails with `Could not find imported module support code for ansible_collections.acme.app.plugins.modules.app_config. Looked for (['ansible.module_utils.app_helpers', ...])` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The module imports its helper the old, non-collection way (`from ansible.module_utils.app_helpers import ...`). Inside a collection, module_utils must be imported by their full collection path, so that AnsiballZ can bundle them.

**Fix →**
```python
from ansible_collections.acme.app.plugins.module_utils.app_helpers import render_config
```
Make sure the file exists at `plugins/module_utils/app_helpers.py`, and rebuild and reinstall the collection.

**Prevent →** Run `ansible-test sanity` and unit tests for every collection before publishing. They catch broken imports.

</details>

### Q149. `template error while templating string: No filter named 'to_slug'` for your team's own filter 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- The plugin lives in `filter_plugins/` next to an old playbook location, but the playbook moved.
- The plugin is in a role's `filter_plugins/`, and that role is not part of this play.
- The plugin is in a collection but is called without its FQCN.
- The Python file has a syntax error, so the plugin silently fails to load (visible with `-vvvv`).

**Diagnose →**
```bash
ansible-doc -t filter -l | grep -i slug
ANSIBLE_FILTER_PLUGINS=./filter_plugins ansible localhost -m debug -a "msg={{ 'A B' | to_slug }}" -vvvv
```

**Fix →** Ship custom plugins in a collection and call them as `acme.utils.to_slug`, or set `filter_plugins = ./plugins/filter` in `ansible.cfg`.

**Prevent →** Keep custom plugins in a versioned collection with tests (`ansible-test units`).

</details>

### Q150. An outdated copy of the `common` role runs instead of the one in the repo 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Roles are found by name, and the first match wins. If the playbook moved and the project's `roles/` is no longer next to it, or `roles_path` lists `~/.ansible/roles` first, a stale `common` installed months ago shadows the repo copy.

**Diagnose →**
```yaml
- ansible.builtin.debug:
    var: role_path        # inside the role: shows which copy is used
```
`ls ~/.ansible/roles /etc/ansible/roles` for leftover copies.

**Fix →** Remove the stale copies, put the project's role directories first in `roles_path`, and give internal roles unique names, or better, ship them in a collection (`acme.platform.common`) so the FQCN removes the ambiguity.

**Prevent →** Run CI in clean containers or execution environments with no user-level role directories.

</details>

---

## Vault, Performance and Execution Behaviour

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. `ERROR! Attempting to decrypt but no vault secrets found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The playbook loads a vaulted file or an `!vault` variable, and no password was supplied: no `--ask-vault-pass`, `--vault-password-file` or `--vault-id`, and no `vault_password_file` in `ansible.cfg`.

**Fix →**
```bash
ansible-playbook site.yml --vault-id prod@~/.secrets/prod.pass
# or in CI
ANSIBLE_VAULT_PASSWORD_FILE=/run/secrets/vault_pass ansible-playbook site.yml
```

**Prevent →** Set `vault_identity_list` in the project `ansible.cfg`, pointing at password files that CI and developers create outside the repo.

</details>

### Q152. `ERROR! Decryption failed (no vault secrets were found that could decrypt) on inventories/prod/group_vars/all/vault.yml` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A password was supplied but it is the wrong one, for example the staging password used for a prod file. Another cause is a password file with a trailing newline or extra characters. Ansible strips one trailing newline, but not `\r\n` from Windows editors.

**Diagnose →**
```bash
head -1 inventories/prod/group_vars/all/vault.yml   # $ANSIBLE_VAULT;1.2;AES256;prod
ansible-vault view --vault-id prod@prompt inventories/prod/group_vars/all/vault.yml
od -c ~/.secrets/prod.pass | tail -2               # look for \r
```

**Fix →** Use the correct password for that environment, and remove the stray `\r` with `dos2unix`.

**Prevent →** Use one vault ID per environment, with IDs that match the inventory directories.

</details>

### Q153. Both `--vault-id dev@dev.pass` and `--vault-id prod@prompt` are passed, and prod files still fail to decrypt 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The vault ID label is only a hint. Ansible tries every supplied secret, so a failure means none of them is right. The file was probably encrypted with a different password than the one now called "prod", for example by someone who forgot `--encrypt-vault-id`. With `vault_id_match = True`, only the secret whose label matches the file header is tried.

**Diagnose →** `head -1` on the file shows the label it was encrypted with. If it has no label (`1.1` format), it was encrypted without an ID.

**Fix →** Re-key the file to the correct ID:
```bash
ansible-vault rekey --vault-id old@prompt --new-vault-id prod@prompt file.yml
```

**Prevent →** Set `encrypt_vault_id = prod` in each environment's `ansible.cfg`, or wrap encryption in a script that always passes `--encrypt-vault-id`.

</details>

### Q154. `Vault format unhexlify error: Odd-length string` after pasting an `encrypt_string` value 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `!vault |` block was damaged when pasted: lines re-wrapped, a line dropped, inconsistent indentation, or the trailing digits cut off. `Non-hexadecimal digit found` points to the same problem.

**Fix →** Generate the value again and paste it as one block:
```bash
ansible-vault encrypt_string --vault-id prod@prompt --stdin-name db_password
```
```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.2;AES256;prod
  6233...   # every line indented the same, no edits
```
Check with `ansible localhost -m debug -a var=db_password -e @file.yml --vault-id prod@prompt`.

**Prevent →** For many secrets, prefer whole-file encryption (`vault.yml`), which cannot be mangled by YAML indentation.

</details>

### Q155. `Problem running vault password script /home/ci/.vault_pass ([Errno 8] Exec format error). If this is not a script, remove the executable bit from the file.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** If a vault password file is executable, Ansible **runs** it and uses its stdout as the password. A plain text file that ended up `chmod +x` (common when copied from Windows or by a CI secret mount) is executed and fails.

**Fix →** For a plain password file, run `chmod 600 ~/.vault_pass`. For a real script (for example one fetching from AWS Secrets Manager), give it a shebang, make it print only the password, and exit 0:
```bash
#!/usr/bin/env bash
aws secretsmanager get-secret-value --secret-id ansible/vault/prod \
  --query SecretString --output text
```
Scripts named `*-client` (for example `vault-pass-client.py`) also receive `--vault-id <label>`.

**Prevent →** Check the file mode in CI, and keep the password retrieval logic in a reviewed script.

</details>

### Q156. Every change to `vault.yml` produces an unreadable diff and constant merge conflicts 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Encrypted files change completely on every edit, because each encryption uses a new salt. Two people editing the same vault file at the same time always conflict.

**Fix →** Get readable local diffs with a textconv:
```bash
echo '*vault*.yml diff=ansible-vault' >> .gitattributes
git config diff.ansible-vault.textconv "ansible-vault view"
```
To resolve a conflict, view both versions, merge the plaintext by hand, and encrypt again.

**Prevent →** Keep vault files small and split them per environment and per role. Consider an external secret manager for secrets that change often.

</details>

### Q157. Vaulted variables are undefined, although the file decrypts fine with `ansible-vault view` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The vault file is not where Ansible loads variables from. It sits at `group_vars/prod/secrets/vault.yml` (too deep is fine), or at `vars/vault.yml` (never loaded automatically), or it is named after a group that does not exist.

**Fix →** Use the standard pattern:
```text
inventories/prod/group_vars/all/vars.yml    # db_password: "{{ vault_db_password }}"
inventories/prod/group_vars/all/vault.yml   # vault_db_password: ...  (encrypted)
```
Or load it explicitly with `vars_files`.

**Prevent →** Use the `vault_` prefix and indirection pattern, so `grep` can always find where a secret is used.

</details>

### Q158. On 300 hosts, every task seems to run in batches of five 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default `forks` is **5**. Ansible only talks to five hosts at a time per task.

**Diagnose →** `ansible-config dump | grep -i forks`

**Fix →**
```ini
[defaults]
forks = 50
```
Size this to the controller's CPU and memory. Every fork is a Python process.

**Prevent →** Set `forks` in the project's `ansible.cfg`, and measure with `callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer`.

</details>

### Q159. `pipelining = True` was turned on, but runs are no faster 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The setting is not applied (wrong section or wrong config file), the connection plugin does not use it, or a `become` method that needs a tty disables it. Some modules, such as `copy` and `template`, still need file transfers.

**Diagnose →**
```bash
ansible-config dump --only-changed | grep -i pipelining
ansible web01 -m ping -vvvv | grep -cE 'PUT|sftp'
```
With pipelining active, simple modules show no `PUT` of the AnsiballZ file.

**Fix →** Put the setting in the `[ssh_connection]` (or `[connection]`) section of the `ansible.cfg` that is actually in use, and make sure `requiretty` is not set in sudoers.

**Prevent →** Benchmark changes with `profile_tasks` before and after, instead of guessing.

</details>

### Q160. `ERROR! A worker was found in a dead state` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A fork process died, usually because the Linux OOM killer stopped it. Typical triggers are high `forks`, huge registered results (a `find` across `/`, `slurp` of large files), large fact sets, or a container controller with a low memory limit.

**Diagnose →**
```bash
dmesg -T | grep -i -E 'killed process|oom'
kubectl describe pod <awx-job-pod> | grep -i OOMKilled     # AWX
```

**Fix →** Lower `forks`, avoid registering big outputs (filter on the target, use `changed_when` and short `stdout`), use `gather_subset` to cut facts, and give the controller or AWX job pod more memory.

**Prevent →** Set memory requests and limits for execution nodes based on measured peaks, and watch controller memory during large runs.

</details>

### Q161. With `strategy: free`, a task that reads `hostvars['db01'].db_ready` fails on some runs 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `free` lets each host run ahead on its own, so a web host can reach its task before `db01` has set `db_ready`. With the default `linear` strategy, every host finishes task N before any host starts task N+1.

**Fix →** Split the work into separate plays (database first, then web), or keep the play `linear`. Use `free` only for plays whose hosts do not depend on each other.

**Prevent →** Treat cross-host dependencies (`hostvars`, `delegate_to` another inventory host, `run_once` results) as a sign that `free` is not appropriate.

</details>

### Q162. The rolling deploy stops after the second batch with `NO MORE HOSTS LEFT` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Every host in the batch failed, or `max_fail_percentage` was exceeded, or `any_errors_fatal: true` was triggered. Ansible then ends the play, and later batches never start. That is intentional.

**Diagnose →** Scroll up to the first `fatal:` in the failing batch, and check the `PLAY RECAP` for `failed=` and `unreachable=`.

**Fix →** Fix the cause on the failed hosts, then continue with only the hosts that are not yet updated:
```bash
ansible-playbook deploy.yml --limit 'app:!app-0[1-4]'
```

**Prevent →** Add a health check at the end of each batch so a bad release stops early. Write deploys so that a re-run on already-updated hosts is a no-op.

</details>

### Q163. A run failed, but there is no `.retry` file to re-run only the failed hosts 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `retry_files_enabled` has defaulted to `False` since Ansible 2.8.

**Fix →**
```ini
[defaults]
retry_files_enabled = True
retry_files_save_path = ~/.ansible/retry
```
Then run `ansible-playbook site.yml --limit @~/.ansible/retry/site.retry`. Alternatively, copy the failed hosts from the recap into `--limit`.

**Prevent →** In AWX or CI, use the "relaunch on failed hosts" feature, or make plays idempotent so a full re-run is safe.

</details>

### Q164. `--start-at-task "Deploy app"` fails with `'artifact_url' is undefined` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--start-at-task` skips everything before that task, including the `set_fact`, `register` or `include_vars` tasks that define later variables.

**Fix →** Tag the tasks that define variables with `always`, so they run even when skipped over:
```yaml
- ansible.builtin.set_fact:
    artifact_url: "{{ repo_url }}/{{ app_version }}/app.tgz"
  tags: [always]
```
Alternatively, use `--tags deploy` with well-tagged roles instead of `--start-at-task`.

**Prevent →** Compute derived variables in `vars` or `defaults` (evaluated lazily) instead of in `set_fact` where possible.

</details>

### Q165. `--tags deploy` also ran a cleanup task tagged `always` and deleted the release directory 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tasks tagged `always` run for any `--tags` selection, and are skipped only with `--skip-tags always`. The `never` tag works the opposite way: those tasks run only when their other tag is requested explicitly.

**Fix →** Remove `always` from destructive tasks, and give them `never` plus an explicit tag:
```yaml
- ansible.builtin.file:
    path: /opt/app/releases/old
    state: absent
  tags: [never, cleanup]
```
Now it only runs with `--tags cleanup`.

**Prevent →** Keep `always` for fact-setting and validation tasks only. Review `--list-tasks --tags <x>` output before relying on a tag.

</details>

### Q166. After an upgrade: `ERROR! Your Ansible version (2.17.x) is too recent. The most recent version supported by Mitogen for Ansible is ...` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The Mitogen strategy plugin is enabled (`strategy = mitogen_linear`) and its release does not support the new ansible-core yet. Mitogen hooks into private internals, so it often lags behind core releases.

**Fix →** Upgrade Mitogen to a release that supports your ansible-core, or remove `strategy_plugins` and `strategy = mitogen_*` from `ansible.cfg` and make up the speed with `pipelining`, `ControlPersist`, higher `forks` and fewer tasks.

**Prevent →** Treat Mitogen as an optional accelerator. Pin it together with ansible-core, and test upgrades in CI before rolling them to controllers.

</details>

### Q167. A role that runs `include_tasks` in a loop over 3,000 firewall rules takes 40 minutes 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Each loop iteration of `include_tasks` is processed per host: templating, a task object for each item, and a separate module run with its own SSH round trip. With thousands of items, the overhead dominates.

**Fix →** Push the loop down to the target:
- Template a full ruleset file (`nftables.conf`, iptables-restore format) once, and apply it with a handler.
- Use modules that accept lists, such as `package` with a list of names.
- If you must loop, loop over a module directly instead of over `include_tasks`.

**Prevent →** Profile with `profile_tasks`, and treat any task with more than a few hundred iterations as a sign the design needs to change.

</details>

### Q168. `ERROR! Invalid callback for stdout specified: yaml` after rebuilding the controller 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `stdout_callback = yaml` relied on the `community.general.yaml` callback, which is missing in a minimal ansible-core install, and is deprecated in favour of the built-in result format option.

**Fix →**
```ini
[defaults]
stdout_callback = ansible.builtin.default
callback_result_format = yaml
```

**Prevent →** Prefer built-in options over collection callbacks for core behaviour, and keep `ansible.cfg` in the repo, reviewed like code.

</details>

### Q169. A setting in your `ansible.cfg` has no effect 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another config file wins. Ansible uses the **first** file it finds and never merges them: `ANSIBLE_CONFIG`, then `./ansible.cfg`, then `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`. An environment variable such as `ANSIBLE_FORKS` overrides the file. The setting may also be in the wrong `[section]`.

**Diagnose →**
```bash
ansible --version | grep 'config file'
ansible-config dump --only-changed
env | grep ^ANSIBLE_
```

**Fix →** Run from the project root, or export `ANSIBLE_CONFIG`. Move the key into the section that `ansible-config list` documents for it.

**Prevent →** Start CI jobs by printing `ansible --version` and `ansible-config dump --only-changed`.

</details>

### Q170. `ERROR! Unexpected Exception, this is probably a bug: cannot import name 'environmentfilter' from 'jinja2'` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An old Ansible (2.9 or 2.10) runs with Jinja2 3.1 or newer, which removed `environmentfilter`. This usually happens after `pip install -U` pulled new dependencies into a shared environment.

**Diagnose →** `ansible --version` shows the Python path. Then run `<that python> -m pip show jinja2 ansible-core`.

**Fix →** The real fix is to upgrade to a supported ansible-core. As a stopgap, pin `jinja2<3.1` in that virtualenv.

**Prevent →** Run Ansible from a dedicated virtualenv or execution environment with locked dependencies (`pip-compile`, an EE image tag), never from the system Python.

</details>

### Q171. `ERROR: Ansible requires the locale encoding to be UTF-8; Detected ISO8859-1.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Since ansible-core 2.14, the controller refuses to run under a non-UTF-8 locale. This is common in minimal containers, cron jobs and CI runners with `LANG` unset.

**Fix →**
```bash
export LANG=C.UTF-8 LC_ALL=C.UTF-8
```
Set these in the Dockerfile `ENV`, in the CI job environment, or in the cron entry.

**Prevent →** Set the locale in your controller and execution environment images.

</details>

### Q172. Packages got installed on the CI runner, and every "host" in the output reports the runner's hostname 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `connection: local` (play level) or `ansible_connection: local` (in `group_vars/all`) was copied from an API-only playbook. Every task now runs on the controller once per inventory host.

**Diagnose →**
```bash
ansible-inventory --host web01 --vars | grep ansible_connection
grep -rn 'connection: local' playbooks/
```

**Fix →** Remove the setting, and for API tasks use `delegate_to: localhost` (or `hosts: localhost`) only where it is needed. Then clean up the runner.

**Prevent →** Run CI on ephemeral runners, and add an `assert` that `ansible_facts['hostname']` matches the expected pattern in sensitive roles.

</details>

### Q173. An `uri` call to the CMDB returns `HTTP Error 429: Too Many Requests` for most hosts 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** With `forks: 50`, fifty hosts hit the API at the same moment, and the API rate-limits.

**Fix →**
```yaml
- ansible.builtin.uri:
    url: https://cmdb.internal/api/hosts/{{ inventory_hostname }}
  delegate_to: localhost
  throttle: 5
  register: cmdb
  until: cmdb.status != 429
  retries: 5
  delay: 10
```
`throttle` limits only this task, without slowing the rest of the play.

**Prevent →** Use `throttle` for shared external dependencies (APIs, license servers, package mirrors), or make one bulk API call with `run_once`.

</details>

### Q174. Copying a 4 GB artifact with `copy` takes 20 minutes per host and sometimes times out 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `copy` checksums the file on both sides and pushes it from the controller over SSH to every host, so the controller's uplink becomes the bottleneck.

**Fix →** Have the targets pull the artifact instead:
```yaml
- ansible.builtin.get_url:
    url: "https://artifacts.internal/app/{{ app_version }}/app.tar.gz"
    dest: /opt/app/app.tar.gz
    checksum: "sha256:{{ app_sha256 }}"
```
For directory trees, use `ansible.posix.synchronize`, which needs `rsync` on both ends.

**Prevent →** Store artifacts in a repository (S3, Artifactory, Nexus) close to the hosts. Do not ship large binaries from the controller.

</details>

### Q175. Runs on the controller start failing with `[Errno 28] No space left on device` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Something fills the controller's disk over time: a growing `log_path` file, a `jsonfile` fact cache for thousands of hosts, `fetch` tasks that download files on every run, or leftover `~/.ansible/tmp` directories from killed runs.

**Diagnose →**
```bash
df -h ~ /tmp
du -sh ~/.ansible/* /var/log/ansible.log 2>/dev/null
```

**Fix →** Clean up and rotate: add a `logrotate` rule for the Ansible log, prune the fact cache, and set `fetch` destinations under a directory that is cleaned. Remove stale `~/.ansible/tmp/ansible-local-*` directories.

**Prevent →** Run controllers ephemerally (CI containers, AWX execution environments), or monitor disk usage on long-lived controllers.

</details>

---

## Cloud, AWX, Windows and Molecule

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. `Failed to import the required Python library (botocore and boto3) on web01's Python /usr/bin/python3.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWS modules (`amazon.aws.*`) run wherever the task runs. Here the task ran on the managed host `web01`, which has no boto3. Normally AWS API calls should run on the controller, with `hosts: localhost` or `delegate_to: localhost`. When they do run there, the controller's Python (or virtualenv, or EE) may be missing boto3.

**Diagnose →** The message names the host and the interpreter. Check with `"{{ ansible_playbook_python }}" -c 'import boto3, botocore'`.

**Fix →**
```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - amazon.aws.ec2_instance_info:
        region: ap-south-1
        filters: {"tag:Env": prod}
```
Then `pip install boto3 botocore` into the controller environment.

**Prevent →** Build boto3 into the execution environment (`requirements.txt` in `ansible-builder`), and keep API plays on `localhost`.

</details>

### Q177. `[WARNING]: * Failed to parse inventory/aws.yml with auto plugin: inventory source 'inventory/aws.yml' could not be verified by inventory plugin 'amazon.aws.aws_ec2'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `aws_ec2` plugin only accepts files whose names end in `aws_ec2.yml` or `aws_ec2.yaml`. It can also fail if the collection is not installed, or if `enable_plugins` in `ansible.cfg` excludes `auto` and `amazon.aws.aws_ec2`.

**Fix →** Rename the file to `inventory/prod.aws_ec2.yml`:
```yaml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters:
  instance-state-name: running
  "tag:Env": prod
```
Test it with `ansible-inventory -i inventory/prod.aws_ec2.yml --graph`.

**Prevent →** Use the naming convention `<env>.aws_ec2.yml`, and check the inventory with `ansible-inventory --graph` in CI.

</details>

### Q178. The `aws_ec2` inventory parses without errors but returns zero hosts 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- The wrong `regions`.
- A tag filter with the wrong case (`tag:env` against `tag:Env`).
- The wrong AWS account or profile.
- Instances that are stopped while the filter asks for `running`.
- A stale inventory cache (`cache: true`).

**Diagnose →**
```bash
aws sts get-caller-identity
aws ec2 describe-instances --region ap-south-1 --filters "Name=tag:Env,Values=prod" --query 'Reservations[].Instances[].InstanceId'
ansible-inventory -i prod.aws_ec2.yml --list -vvv | head -50
```

**Fix →** Align the filters with what the AWS CLI returns, set `profile:` or the environment credentials for the right account, and refresh with `--flush-cache` if caching is on.

**Prevent →** Standardise tag keys (for example `Env`, `Role`) enforced by Terraform or AWS Config, and alert when a key group comes back empty.

</details>

### Q179. `Failed to describe instances: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation` or `(ExpiredToken)` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The IAM identity lacks `ec2:DescribeInstances` (and `ec2:DescribeRegions` if `regions` is not set), an SSO or STS session expired, or the controller is picking up stale keys from `~/.aws/credentials` before its instance role.

**Diagnose →**
```bash
aws sts get-caller-identity
env | grep ^AWS_
```

**Fix →** Grant read-only EC2 describe permissions to the automation role, refresh the session (`aws sso login --profile prod`), or remove the stale static keys so the instance or IRSA role is used.

**Prevent →** In CI and AWX, use short-lived role credentials (OIDC to AWS, instance profiles, IRSA) instead of static access keys.

</details>

### Q180. Dynamic inventory finds the instances, but every host is `UNREACHABLE` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** By default, the plugin names hosts by public DNS. Private-subnet instances have no public DNS, or the controller cannot route to it. Ansible then tries to connect to an address that does not work.

**Diagnose →** `ansible-inventory -i prod.aws_ec2.yml --host <name> | jq '.ansible_host, .private_ip_address, .public_dns_name'`

**Fix →**
```yaml
hostnames:
  - tag:Name
  - private-ip-address
compose:
  ansible_host: private_ip_address
```
Run from inside the VPC, or go through a bastion (`ProxyJump`) or SSM.

**Prevent →** Use readable names for hosts (`tag:Name`) and keep connection addresses in `compose`.

</details>

### Q181. Using the SSM connection plugin fails with `SessionManagerPlugin is not found` or hangs on file transfer 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- The AWS `session-manager-plugin` binary is not installed on the controller or in the EE.
- No `ansible_aws_ssm_bucket_name` is set. The plugin moves module files through S3, so a bucket is required.
- The instance role lacks `AmazonSSMManagedInstanceCore` or access to that bucket.
- The bucket is in a different region than the instance.

**Diagnose →**
```bash
session-manager-plugin --version
aws ssm describe-instance-information --query 'InstanceInformationList[].InstanceId'
```

**Fix →**
```yaml
ansible_connection: amazon.aws.aws_ssm     # community.aws.aws_ssm on older collections
ansible_aws_ssm_region: ap-south-1
ansible_aws_ssm_bucket_name: acme-ansible-ssm-ap-south-1
```
Install the plugin into the EE image, and grant the instance role `s3:GetObject` and `s3:PutObject` on the bucket.

**Prevent →** Provision the bucket, the IAM roles and the SSM agent with Terraform as part of the base image or module.

</details>

### Q182. `winrm or requests is not installed: No module named 'winrm'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The controller's Python lacks `pywinrm`. Windows hosts run modules over WinRM, and the client library is on the controller side.

**Fix →**
```bash
pip install "pywinrm>=0.4"            # add [kerberos] or [credssp] extras as needed
```
Install it into the same environment `ansible-playbook` uses (check `ansible --version` for the Python path) or into the EE.

**Prevent →** Keep `pywinrm` (and `pypsrp` if you use `psrp`) in the EE's `requirements.txt`.

</details>

### Q183. `ntlm: HTTPSConnectionPool(host='win01', port=5986): Max retries exceeded with url: /wsman (Caused by SSLError(SSLCertVerificationError(...)))` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The WinRM HTTPS listener uses a self-signed certificate, or a certificate from an internal CA that the controller does not trust, or a certificate whose CN does not match the name you connect with.

**Fix →** In production, issue listener certificates from the internal CA (through AD CS auto-enrolment) and trust that CA:
```yaml
ansible_winrm_ca_trust_path: /etc/pki/tls/certs/corp-root-ca.pem
```
In a lab only, use `ansible_winrm_server_cert_validation: ignore`.

**Prevent →** Automate listener certificate provisioning with GPO or a bootstrap script, and connect by FQDN matching the certificate.

</details>

### Q184. `ntlm: the specified credentials were rejected by the server` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The wrong username format, a wrong or expired password, a locked account, or the user is not in a group allowed to use WinRM (local Administrators or Remote Management Users). With `basic` transport, only **local** accounts work, and only over HTTPS or with `AllowUnencrypted`.

**Diagnose →** Test from another Windows host: `Test-WSMan win01 -Credential (Get-Credential) -Authentication Negotiate`.

**Fix →** Use `ansible_user: svc_ansible@CORP.EXAMPLE.COM` with Kerberos, or `CORP\svc_ansible` with NTLM. Unlock the account or reset the password, and add it to the right local group.

**Prevent →** Use a dedicated domain service account managed by GPO, with its password stored in Vault or an AWX credential and rotated regularly.

</details>

### Q185. `kerberos: authGSSClientStep() failed: (('Unspecified GSS failure. Minor code may provide more information', 851968), ('Server not found in Kerberos database', -1765328377))` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Kerberos needs the service principal `HTTP/<fqdn>`. Connecting by IP address or short name, a broken reverse DNS entry, or a host in a different realm means the KDC cannot find the principal.

**Diagnose →**
```bash
kinit svc_ansible@CORP.EXAMPLE.COM && klist
kvno HTTP/win01.corp.example.com
host win01.corp.example.com; host <its IP>
```

**Fix →** Set `ansible_host` to the FQDN. Fix DNS (forward and reverse). Make sure `/etc/krb5.conf` has the realm in **uppercase**, with correct `kdc` and `domain_realm` mappings.

**Prevent →** Use the FQDN in inventory for Windows hosts, and add a CI test that runs `kinit` and a `win_ping` against one known host.

</details>

### Q186. `The powershell shell family is incompatible with the sudo become plugin` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `become: true` with the default `sudo` method was inherited by Windows hosts, usually from `group_vars/all` or play-level settings shared with Linux.

**Fix →**
```yaml
# group_vars/windows.yml
ansible_connection: winrm
ansible_shell_type: powershell
ansible_become_method: runas
ansible_become_user: SYSTEM      # only where elevation is really needed
```
Better, do not set `become` for Windows plays at all unless a task needs it.

**Prevent →** Keep separate plays and group variables for Linux and Windows, and do not put connection or become settings in `all`.

</details>

### Q187. A Windows task that copies from `\\fileserver\share` fails with `Access is denied`, although the same account works over RDP 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** This is the Kerberos and NTLM **double-hop** problem. The WinRM session cannot pass your credentials on to a second server.

**Fix →** Choose one of these:
- Kerberos with delegation: `ansible_winrm_kerberos_delegation: true`, with a forwardable ticket (`kinit -f`).
- `become: true`, `become_method: runas`, and `become_user: CORP\svc_ansible` with a password, which creates a fresh logon that has network credentials.
- CredSSP transport. It works, but it sends reusable credentials, so avoid it in high-security environments.

**Prevent →** Stage files from an HTTP artifact store with `win_get_url` instead of SMB shares, so no double hop is needed.

</details>

### Q188. A playbook works locally but fails in AWX with `couldn't resolve module/action 'community.mysql.mysql_db'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWX runs jobs inside an **execution environment** container, not with your laptop's collections. The EE does not include `community.mysql`, and the project has no `collections/requirements.yml` for AWX to install during project sync.

**Fix →** Choose one of these:
- Add `collections/requirements.yml` to the project. AWX installs it on project sync if collection installs are allowed.
- Build a custom EE:
```yaml
# execution-environment.yml (ansible-builder v3)
version: 3
images:
  base_image:
    name: quay.io/ansible/awx-ee:latest
dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt
```
```bash
ansible-builder build -t registry.internal/acme-ee:1.4.0
```

**Prevent →** Test locally with `ansible-navigator run --eei registry.internal/acme-ee:1.4.0` so local runs match AWX.

</details>

### Q189. An AWX project sync fails while installing collections from Private Automation Hub, though Git checkout succeeds 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AWX installs `collections/requirements.yml` using the **Galaxy / Automation Hub credentials attached to the Organization**, in the order they are listed. The hub credential is missing, out of order, or holds an expired token, so the install gets a 401 or falls back to public Galaxy, which does not have the internal collection.

**Diagnose →** Open the project update job output and look for the `ansible-galaxy collection install` step and the server URL it called.

**Fix →** Create a credential of type "Ansible Galaxy/Automation Hub API Token" with the hub URL and token, attach it to the Organization **above** public Galaxy, and sync again.

**Prevent →** Use a service token with a known expiry and rotation reminder. Better, pre-bake internal collections into the EE.

</details>

### Q190. AWX jobs stay in `Pending` and never start 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →**
- No instance in the job's instance group has free capacity (forks × concurrent jobs).
- The job is waiting for a dependent project or inventory update that is itself stuck.
- With container groups on Kubernetes, the job pod cannot be scheduled (resource requests, node selectors, quota) or cannot pull its EE image.

**Diagnose →**
```bash
kubectl -n awx get pods | grep automation-job
kubectl -n awx describe pod <automation-job-pod>     # events: Insufficient cpu, ImagePullBackOff
kubectl -n awx logs deploy/awx-task -c awx-task | tail -50
```
In the UI, check Instance Groups for capacity and the job's "Explanation" field.

**Fix →** Add capacity or adjust the job template's instance group, fix the stuck dependency job, and right-size the pod requests or fix the image pull secret.

**Prevent →** Monitor capacity and pending time, and set sensible job template forks so one job cannot consume an instance's whole capacity.

</details>

### Q191. An AWX inventory synced from the Git project shows the hosts, but none of the `group_vars` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The inventory source points at a file whose `group_vars/` is in a different directory, or the hosts were added manually or by another source type. AWX imports `group_vars` and `host_vars` only when they sit next to the inventory file or directory selected in a "Sourced from a Project" source.

**Diagnose →** Open the inventory source and check the "Inventory file" path. Compare it with the repository layout. Look at the variables of a group in the AWX UI.

**Fix →** Point the source at `inventories/prod/` (the directory) or at a file inside it, so the adjacent `group_vars/` is picked up. Enable "Overwrite" and "Overwrite variables" if stale values remain, and sync again.

**Prevent →** Keep the same inventory directory layout for CLI and AWX. Variables next to the playbook still load at run time, but variables next to the inventory need the right source path.

</details>

### Q192. An AWX job fails with `Decryption failed (no vault secrets were found that could decrypt)` although a Vault credential is attached 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The project mixes vault IDs (`dev`, `prod`), and the attached credential has no Vault Identifier, or a different one, from the label in the file header. Another cause is that the credential holds a different password than the one used to encrypt.

**Diagnose →** `head -1` on the vault file in the repository shows the label it was encrypted with. Compare it with the credential's "Vault Identifier" field.

**Fix →** Create one Vault credential per ID (`Vault Identifier: prod`) and attach all the ones the job needs. AWX allows several Vault credentials as long as their IDs differ.

**Prevent →** Use one vault ID per environment, the same ones in the CLI and AWX, and document them in the repo README.

</details>

### Q193. AWX jobs fail before starting with `Error: initializing source docker://registry.internal/acme-ee:1.4.0: ... unauthorized: authentication required` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The EE lives in a private registry, and the Execution Environment object in AWX has no Container Registry credential, or the credential has expired. On container groups, the namespace may also lack an `imagePullSecret`.

**Fix →** Create a "Container Registry" credential and attach it to the Execution Environment in AWX. For container groups, add `imagePullSecrets` to the pod spec override. Set the pull policy to "Missing" for pinned tags.

**Prevent →** Use immutable EE tags (`1.4.0`, not `latest`), robot or service accounts for the registry, and a test job that runs after every EE release.

</details>

### Q194. `ansible-navigator run site.yml` fails with `Permission denied (publickey)`, while `ansible-playbook site.yml` works 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Navigator runs the playbook inside an EE container. Your `~/.ssh` keys are not visible there, and the SSH agent socket is only passed through if `SSH_AUTH_SOCK` is set and the agent holds the key. Custom key paths in inventory point at host paths that do not exist in the container.

**Diagnose →**
```bash
ssh-add -l
ansible-navigator run site.yml -m stdout -vvv 2>&1 | grep -i identity
```

**Fix →** Start the agent and add the key (`eval "$(ssh-agent)"; ssh-add ~/.ssh/id_ed25519`), or mount the key through the navigator settings:
```yaml
ansible-navigator:
  execution-environment:
    volume-mounts:
      - src: ~/.ssh
        dest: /home/runner/.ssh
        options: ro
```
Use `--ee false` to confirm the problem is container-specific.

**Prevent →** Commit a project `ansible-navigator.yml` so every developer gets the same mounts and EE image.

</details>

### Q195. `molecule create` fails with `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker is not running, the user is not in the `docker` group (on Linux), or the machine only has Podman and the scenario uses the Docker driver.

**Diagnose →** `docker info`, `id -nG | grep docker`, `systemctl status docker`.

**Fix →** Start Docker and add the user to the group (`sudo usermod -aG docker $USER`, then log in again), or switch the scenario to Podman (`driver: name: podman`) with `molecule-plugins[podman]`.

**Prevent →** Document the driver in the repo and provide a dev container with Docker and Molecule preinstalled.

</details>

### Q196. Molecule converge fails on service tasks with `System has not been booted with systemd as init system (PID 1). Can't operate.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The test container runs a shell or `sleep` as PID 1, not systemd. On cgroup v2 hosts (Ubuntu 22.04+ runners, recent Docker Desktop), systemd containers also need the right cgroup setup.

**Fix →**
```yaml
platforms:
  - name: rocky9
    image: geerlingguy/docker-rockylinux9-ansible:latest
    command: /usr/sbin/init
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    pre_build_image: true
```

**Prevent →** Use images built for systemd testing, and keep a known-good `molecule.yml` template that every role copies.

</details>

### Q197. `Failed to find driver docker. Please ensure that the driver is correctly installed.` after upgrading Molecule 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The drivers were split out of Molecule core. The old `molecule-docker` package is replaced by `molecule-plugins[docker]`, and a fresh virtualenv has only the built-in `default` (delegated) driver.

**Diagnose →** `molecule drivers`, `pip list | grep -i molecule`.

**Fix →**
```bash
pip install "molecule" "molecule-plugins[docker]" docker
```
Also install the `community.docker` collection, which the driver's create and destroy playbooks use.

**Prevent →** Pin `molecule`, `molecule-plugins`, `ansible-core` and `ansible-lint` in `requirements-dev.txt`.

</details>

### Q198. `molecule verify` fails with `ModuleNotFoundError: No module named 'testinfra'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The scenario sets `verifier: name: testinfra`, but `pytest-testinfra` is not installed. Since Molecule 3, the default verifier is **Ansible** (`verify.yml`), so old scenarios often still expect testinfra.

**Fix →** Run `pip install pytest-testinfra`, or move to the Ansible verifier:
```yaml
verifier:
  name: ansible
```
with assertions in `molecule/default/verify.yml` (`uri`, `command` with `changed_when: false`, `assert`).

**Prevent →** Standardise one verifier across all roles, and pin its dependencies.

</details>

### Q199. Molecule passes locally but fails in GitHub Actions at `Gathering Facts` with `/bin/sh: 1: python3: not found` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The CI scenario uses a bare official image (`ubuntu:24.04`, `debian:12`) with no Python. Locally, a cached test image with Python was used.

**Fix →** Use an image that has Python and systemd (`pre_build_image: true`), or bootstrap in `prepare.yml` before facts are gathered:
```yaml
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.raw: test -e /usr/bin/python3 || (apt-get update && apt-get install -y python3)
      changed_when: false
```

**Prevent →** Pin test images by tag or digest in `molecule.yml`, and run CI and local tests with the same images.

</details>

### Q200. Raising `ansible_winrm_operation_timeout_sec` breaks every Windows host with `read_timeout_sec must exceed operation_timeout_sec, and both must be non-zero` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** pywinrm requires the HTTP read timeout to be **greater** than the WinRM operation timeout. The defaults are 30 and 20 seconds. Someone raised only the operation timeout to deal with slow tasks.

**Fix →** Raise both, keeping the read timeout higher:
```yaml
ansible_winrm_operation_timeout_sec: 60
ansible_winrm_read_timeout_sec: 90
```
For long-running work (Windows Updates, large installers), do not keep inflating timeouts. Use the module's own options (for example `ansible.windows.win_updates` with `reboot: true`) or `async`/`poll`.

**Prevent →** Keep WinRM tuning in `group_vars/windows.yml` with a comment explaining it, and test changes against one Windows host before rolling them out.

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
