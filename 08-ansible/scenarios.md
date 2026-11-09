# Ansible — Scenario-Based Questions

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

> Real-world Ansible scenarios: what you would do, why, and how to explain it in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Inventory and Targeting](#inventory-and-targeting) — Q1–Q25
- [Playbook Design, Handlers and Idempotency](#playbook-design-handlers-and-idempotency) — Q26–Q50
- [Variables, Facts and Templates](#variables-facts-and-templates) — Q51–Q75
- [Roles, Collections and Project Structure](#roles-collections-and-project-structure) — Q76–Q100
- [Rolling Updates, Delegation and Orchestration](#rolling-updates-delegation-and-orchestration) — Q101–Q125
- [Security: Vault, Become and Compliance](#security-vault-become-and-compliance) — Q126–Q150
- [Performance, Testing and CI/CD](#performance-testing-and-cicd) — Q151–Q175
- [AWX, Cloud and Large-Scale Operations](#awx-cloud-and-large-scale-operations) — Q176–Q200

---

## Inventory and Targeting

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. You are handed a fresh Ansible project and 10 servers. How do you make sure Ansible can reach all of them before writing any playbook? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** No playbook yet; the first risk is connectivity, not logic.

**What I'd do →**
1. Write an inventory with the hosts and `ansible_user`/key settings.
2. Run `ansible all -i inventory.ini -m ping`.
3. For failures, rerun with `-vvv` on one host.

```bash
ansible all -i inventory.ini -m ansible.builtin.ping
ansible web01 -i inventory.ini -m ansible.builtin.ping -vvv
```

**Why →** `ping` tests SSH login, Python on the target and module execution in one shot, so a pong means real playbooks will run.

**Interview tip →** Say that `ping` is not ICMP; it is a full Ansible round trip.

</details>

### Q2. Your team keeps prod and staging hosts in one inventory file and someone ran a playbook on prod by mistake. How do you prevent it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Environment separation is weak; the blast radius is controlled by a human typing the right `-l`.

**What I'd do →**
1. Split into `inventories/staging/` and `inventories/prod/`, each with its own `hosts` and `group_vars/`.
2. Make staging the default in `ansible.cfg` (`inventory = inventories/staging`).
3. Require `-i inventories/prod` explicitly, and add a pre-task guard for prod.
4. In CI, only the prod pipeline job has the prod credentials.

```yaml
- hosts: all
  pre_tasks:
    - ansible.builtin.pause:
        prompt: "Type PROD to continue"
      register: confirm
      when: env_name == 'prod'
    - ansible.builtin.assert:
        that: confirm.user_input == 'PROD'
      when: env_name == 'prod'
```

**Why →** Separate inventories mean separate variables and separate credentials, so a typo cannot cross environments.

**Interview tip →** Mention that `group_vars` per environment also prevents prod secrets leaking into staging runs.

</details>

### Q3. You need to run a playbook on only the web servers in Mumbai that are not in the canary group. How? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A set-algebra host pattern: intersect two groups, exclude a third.

**What I'd do →**
```bash
ansible-playbook site.yml --limit 'web:&mumbai:!canary'
ansible-playbook site.yml --limit 'web:&mumbai:!canary' --list-hosts
```
Always run `--list-hosts` first to see what the pattern resolves to.

**Why →** `:&` is intersection and `:!` is exclusion; quoting prevents the shell from eating `!` and `&`.

**Interview tip →** Know that `--limit` narrows the play's `hosts:`; it cannot add hosts the play never targeted.

</details>

### Q4. Servers are autoscaled in AWS and IPs change daily. How do you keep the inventory accurate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A static file will always be stale; the source of truth is the cloud API.

**What I'd do →**
1. Use the `amazon.aws.aws_ec2` inventory plugin.
2. Group by tags with `keyed_groups`, and use private IPs as the connection address.
3. Enable caching so repeated runs are quick.

```yaml
# inventories/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters:
  tag:Env: prod
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
compose:
  ansible_host: private_ip_address
cache: true
cache_timeout: 300
```

**Why →** Instances join groups automatically through tags, so new nodes get configured without editing any file.

**Interview tip →** The file name must end in `aws_ec2.yml` or `aws_ec2.yaml` for the plugin to accept it.

</details>

### Q5. A host is reachable by SSH on port 2222 with a different user from all others. How do you model that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One exception to otherwise uniform connection settings.

**What I'd do →** Set host-level connection variables on that host only.

```ini
[web]
web01
web02
legacy01 ansible_port=2222 ansible_user=admin ansible_ssh_private_key_file=~/.ssh/legacy.pem
```

**Why →** Inventory-level variables beat group defaults for that host while leaving everyone else untouched.

**Interview tip →** If the exception grows to several settings, move it to `host_vars/legacy01.yml`.

</details>

### Q6. You must configure servers in a private subnet that is only reachable through a bastion. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No direct route from the control node; SSH has to hop.

**What I'd do →**
1. Set a ProxyJump in the inventory group vars.
2. Keep the bastion's key and the private hosts' key separate.

```yaml
# group_vars/private.yml
ansible_ssh_common_args: >-
  -o ProxyJump=ec2-user@bastion.example.com
  -o StrictHostKeyChecking=accept-new
```
Test with `ansible private -m ping`.

**Why →** Ansible shells out to OpenSSH, so any native SSH feature works with `ansible_ssh_common_args`.

**Interview tip →** Mention SSM Session Manager as the bastion-less alternative in AWS (`aws_ssm` connection plugin).

</details>

### Q7. You want to run a task on every host in the `db` group and also on localhost in the same playbook. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two audiences in one run, with different connections.

**What I'd do →** Use two plays, or `delegate_to: localhost` for a single task.

```yaml
- hosts: db
  tasks:
    - ansible.builtin.dnf: { name: postgresql-server, state: present }

- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - ansible.builtin.debug: { msg: "DB tier configured" }
```

**Why →** Plays are the unit of host selection; mixing hosts inside one play is what `delegate_to` is for.

**Interview tip →** Implicit localhost is not in the `all` group, so it is not matched by `hosts: all`.

</details>

### Q8. You do not know which of 300 servers still run the vulnerable OpenSSL version. How do you find out fast? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** An audit task: read-only, wide, and urgent.

**What I'd do →**
1. Use an ad-hoc command or `package_facts` with a high fork count.
2. Collect output to a file.

```bash
ansible all -f 50 -m ansible.builtin.shell -a "openssl version" -o | tee openssl-audit.txt
ansible all -m ansible.builtin.package_facts -a "manager=auto" --tree /tmp/facts
```
`-o` gives one line per host, which is greppable.

**Why →** Ad-hoc is the right tool for read-only questions; no playbook needed.

**Interview tip →** For recurring audits, convert it to a playbook that writes a CSV with `lineinfile` or `copy` on the controller.

</details>

### Q9. A teammate asks for a group that contains all hosts that have more than 16 GB RAM. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Group membership depends on facts, which exist only at run time.

**What I'd do →** Use `group_by` after fact gathering.

```yaml
- hosts: all
  tasks:
    - ansible.builtin.group_by:
        key: "bigmem_{{ ansible_memtotal_mb > 16384 }}"

- hosts: bigmem_True
  tasks:
    - ansible.builtin.debug: { msg: "large host" }
```

**Why →** `group_by` builds groups in memory for the rest of the run; inventory files cannot know facts in advance.

**Interview tip →** Alternatively use the `constructed` inventory plugin with cached facts for a persistent version.

</details>

### Q10. You inherited a 400-line INI inventory with repeated variables. How do you clean it up without breaking anything? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Refactor with a safety net: the effective variables per host must not change.

**What I'd do →**
1. Snapshot: `ansible-inventory -i old --list > before.json`.
2. Move shared variables into `group_vars/` and per-host ones into `host_vars/`.
3. Snapshot again and diff.

```bash
ansible-inventory -i inventory --list > after.json
diff <(jq -S . before.json) <(jq -S . after.json)
```

**Why →** `ansible-inventory --list` shows merged hostvars, so an empty diff proves the refactor is behaviour-neutral.

**Interview tip →** Use `--graph --vars` for a quick human-readable view of the group tree.

</details>

### Q11. You need to add a newly provisioned EC2 instance to the inventory in the same playbook that creates it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The host does not exist at inventory-parse time.

**What I'd do →** Create it, wait for SSH, then `add_host`.

```yaml
- hosts: localhost
  tasks:
    - amazon.aws.ec2_instance: { name: app1, image_id: ami-0abc, instance_type: t3.micro, wait: true }
      register: ec2
    - ansible.builtin.add_host:
        name: "{{ ec2.instances[0].private_ip_address }}"
        groups: new_app
    - ansible.builtin.wait_for:
        host: "{{ ec2.instances[0].private_ip_address }}"
        port: 22
        timeout: 300

- hosts: new_app
  tasks:
    - ansible.builtin.ping:
```

**Why →** `add_host` mutates the in-memory inventory, and a later play can target the new group.

**Interview tip →** Alternatively call `meta: refresh_inventory` when using a dynamic inventory plugin.

</details>

### Q12. Two groups define the same variable `app_port` with different values and a host is in both. Which wins and how do you control it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same-precedence group variables with a merge conflict.

**What I'd do →**
1. Know the rule: at the same level, groups are merged in alphabetical order and the last one wins, unless `ansible_group_priority` is set.
2. Set `ansible_group_priority` in the group that should win (higher wins).
3. Better, avoid the conflict by defining the variable at one level.

```ini
# inventory source (not group_vars)
[canary:vars]
ansible_group_priority=20
```

**Why →** Relying on alphabetical order is fragile; an explicit priority documents intent.

**Interview tip →** `ansible_group_priority` only works when set in the inventory source, not in `group_vars/` files, and `ansible-inventory --host` confirms the winner.

</details>

### Q13. Your inventory has hundreds of hosts named `web001` through `web300`. How do you write it compactly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repetition that ranges solve.

**What I'd do →**
```ini
[web]
web[001:300].example.com

[db]
db-[a:c].example.com
```
In YAML inventory you can use the same bracket ranges as the host key.

**Why →** Ranges expand at parse time, and zero padding is preserved when you write `001`.

**Interview tip →** Verify with `ansible web --list-hosts | wc -l`.

</details>

### Q14. A playbook must target different hosts depending on a variable passed at the command line. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The target is a parameter, for example a CI job input.

**What I'd do →** Template `hosts:` from an extra var, with a safe default.

```yaml
- hosts: "{{ target | default('none') }}"
  gather_facts: false
  tasks:
    - ansible.builtin.ping:
```
```bash
ansible-playbook deploy.yml -e target=web:\&mumbai
```

**Why →** Extra vars are available when the play's hosts are resolved; defaulting to `none` means a forgotten variable does nothing instead of hitting `all`.

**Interview tip →** Contrast with `--limit`, which narrows rather than defines.

</details>

### Q15. You manage Windows and Linux servers from one controller. How do you structure the inventory? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two connection types, two sets of modules.

**What I'd do →** Separate groups with connection variables in `group_vars`.

```yaml
# group_vars/windows.yml
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
ansible_port: 5986
```
```yaml
# group_vars/linux.yml
ansible_connection: ssh
```
Use `ansible.windows.*` modules for the Windows group, and `ansible_connection: psrp` or SSH if WinRM is not allowed.

**Why →** Keeping connection settings at group level lets playbooks stay platform-neutral where possible.

**Interview tip →** Install `pywinrm` on the controller and mention that certificate validation should not be ignored outside labs.

</details>

### Q16. Your playbook should skip hosts that are currently in maintenance, tracked in a CMDB. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** An external system decides membership; the inventory must reflect it.

**What I'd do →**
1. Write (or use) a dynamic inventory script/plugin that queries the CMDB and puts maintenance hosts into a `maintenance` group.
2. Exclude it in every run with a default limit, or enforce it in the play.

```yaml
- hosts: "all:!maintenance"
  roles: [baseline]
```

**Why →** Excluding by group in the play makes the rule impossible to forget at the command line.

**Interview tip →** Dynamic scripts must support `--list` and `--host <name>` and print JSON.

</details>

### Q17. How would you run the same playbook against a single host to test, without editing the playbook? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A safe canary run.

**What I'd do →**
```bash
ansible-playbook site.yml --limit web01 --check --diff
ansible-playbook site.yml --limit web01
```

**Why →** `--limit` confines the blast radius, `--check --diff` previews the changes, and then the real run on one host tells you whether to proceed.

**Interview tip →** Say you start with one host, then 10%, then everyone: that is `serial` in a production rollout.

</details>

### Q18. A host appears in `ansible-inventory --list` but is not matched by `hosts: webservers`. What do you check? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A pattern or group-name mismatch.

**What I'd do →**
1. `ansible-inventory --graph` to see the real group tree.
2. `ansible webservers --list-hosts` to test the pattern.
3. Check for typos, case (`Webservers` differs from `webservers`), and for groups with hyphens (use underscores).

**Why →** Group names are case-sensitive, and a host can be present in `all` but in no matching group.

**Interview tip →** Ansible warns "Could not match supplied host pattern" for a mistyped group; never ignore it.

</details>

### Q19. You want a group of hosts derived from a variable, like every host with `env: prod` in its host_vars. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Persistent grouping derived from variables, not facts.

**What I'd do →** Use the `constructed` inventory plugin alongside the static inventory.

```yaml
# inventories/constructed.yml
plugin: ansible.builtin.constructed
strict: false
keyed_groups:
  - key: env
    prefix: env
groups:
  prod_web: env == 'prod' and 'web' in group_names
```

**Why →** The plugin builds groups from hostvars at parse time, so every command sees the same groups.

**Interview tip →** Put it in the inventory directory; it is loaded after the other sources.

</details>

### Q20. A colleague hard-coded IP addresses in playbooks. How do you remove them? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data is baked into code.

**What I'd do →**
1. Put addresses in inventory variables (`ansible_host`, `db_host`).
2. Refer to them with `hostvars` or group lookups in templates.

```yaml
db_host: "{{ hostvars[groups['db'][0]].ansible_host }}"
```

**Why →** Moving data to inventory means the same playbook works in every environment.

**Interview tip →** Prefer inventory hostnames and DNS where you can; IPs only as `ansible_host`.

</details>

### Q21. You must apply a change to hosts in two regions with different SSH keys. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credentials are per group.

**What I'd do →** Group per region and set the key in `group_vars`.

```yaml
# group_vars/eu.yml
ansible_ssh_private_key_file: ~/.ssh/eu-prod.pem
# group_vars/us.yml
ansible_ssh_private_key_file: ~/.ssh/us-prod.pem
```

**Why →** The play stays identical; only the connection details vary by group.

**Interview tip →** In AWX, use separate machine credentials per inventory source instead of file paths.

</details>

### Q22. Your inventory plugin returns hosts, but variables from `group_vars` next to the playbook are ignored. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `group_vars` are loaded relative to the inventory source or to the playbook, and the location matters.

**What I'd do →**
1. Remember: `group_vars/` is searched next to the inventory file and next to the playbook.
2. If the inventory is a plugin config in `inventories/prod/aws_ec2.yml`, put `group_vars/` in `inventories/prod/`.
3. Check with `ansible-inventory -i inventories/prod --host <h>`.

**Why →** Playbook-adjacent group_vars load for any inventory; inventory-adjacent ones load only for that inventory's directory.

**Interview tip →** When both exist, playbook-adjacent group_vars win over inventory-adjacent ones.

</details>

### Q23. You need to send a one-off command to 500 hosts quickly without overloading the jump host. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** High parallelism meets a bottleneck.

**What I'd do →**
```bash
ansible all -m ansible.builtin.command -a "uptime" -f 20
```
and enable SSH pipelining and ControlPersist in `ansible.cfg`.

**Why →** `-f` caps concurrent connections (default 5); tune it to what the bastion's `MaxStartups` and CPU allow.

**Interview tip →** Mention that the default `forks = 5` is deliberately conservative.

</details>

### Q24. Hosts are renamed but keep the same IP, and facts caching still shows old names. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stale fact cache keyed by `inventory_hostname`.

**What I'd do →**
1. Clear the cache entry: delete the file/key for the old name, or run `ansible-playbook ... --flush-cache`.
2. Re-gather facts: `ansible renamed -m ansible.builtin.setup`.
3. Reduce `fact_caching_timeout` where names change often.

**Why →** The cache key is the inventory name, so renamed hosts leave orphaned entries and new names start empty.

**Interview tip →** Cached facts speed up playbooks that use `gather_facts: smart`, but trade freshness.

</details>

### Q25. A new engineer asks how you know which inventory variables actually apply to a host. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Debugging variable merging.

**What I'd do →**
```bash
ansible-inventory -i inventories/prod --host web01
ansible web01 -m ansible.builtin.debug -a "var=hostvars[inventory_hostname]"
```

**Why →** The first prints inventory-derived variables after merging; the second shows what a running play sees, including facts and extra vars.

**Interview tip →** Mention `--vars` with `--graph` to show variables per group.

</details>

---

## Playbook Design, Handlers and Idempotency

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Your playbook reports `changed` on every run for a task that runs a shell command. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `command` and `shell` always report changed because Ansible cannot know the outcome.

**What I'd do →**
1. Prefer a real module.
2. Otherwise add `creates:`/`removes:` or an explicit `changed_when`.

```yaml
- ansible.builtin.command: /opt/app/bin/migrate
  register: mig
  changed_when: "'Applied' in mig.stdout"
```

**Why →** Honest change reporting is what makes idempotency checks and handlers trustworthy.

**Interview tip →** A second run of a healthy playbook should show `changed=0`.

</details>

### Q27. Nginx must reload only when its config file actually changes. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic handler use case.

**What I'd do →**
```yaml
tasks:
  - ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      validate: nginx -t -c %s
    notify: Reload nginx
handlers:
  - name: Reload nginx
    ansible.builtin.service: { name: nginx, state: reloaded }
```

**Why →** Handlers run once at the end of the play, only if a notifying task reported `changed`; `validate` stops a broken file from being installed.

**Interview tip →** Use `meta: flush_handlers` if later tasks depend on the reload.

</details>

### Q28. A task fails halfway and its handler never ran, but the config file was changed. The next run shows no change and the service never reloads. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Handlers are skipped for hosts that fail before the end of the play, and the next run sees the file as already correct.

**What I'd do →**
1. Set `force_handlers: true` on the play (or `--force-handlers`) so notified handlers run despite later failures.
2. Or flush handlers right after the change.
3. Repair now by running the handler's action once.

```yaml
- hosts: web
  force_handlers: true
```

**Why →** Notifications are in memory only; if the run dies, they are lost.

**Interview tip →** This is the best-known handler gotcha; naming it shows real experience.

</details>

### Q29. You need three tasks to notify one handler, and want it to run once. Is that automatic? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several changes, one restart.

**What I'd do →** Just add `notify: Restart app` to each task.

**Why →** A handler runs once per play per host no matter how many tasks notified it. Use `listen:` to let one event trigger several handlers.

```yaml
handlers:
  - name: Restart app
    ansible.builtin.systemd: { name: app, state: restarted }
    listen: "app config changed"
```

**Interview tip →** Notify by `listen` topic to decouple task names from handler names.

</details>

### Q30. You need to deploy a binary only if it is missing or its checksum differs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Make a download idempotent.

**What I'd do →**
```yaml
- ansible.builtin.get_url:
    url: "https://releases.example.com/app-{{ app_version }}.tar.gz"
    dest: /opt/app/app-{{ app_version }}.tar.gz
    checksum: "sha256:{{ app_sha256 }}"
    mode: "0644"
```

**Why →** `get_url` with `checksum` downloads only when the destination is absent or differs, and verifies integrity.

**Interview tip →** Versioned file names give you free rollbacks.

</details>

### Q31. How do you safely edit one line in `/etc/ssh/sshd_config` and be sure SSH keeps working? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A change that can lock you out of every server.

**What I'd do →**
```yaml
- ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    validate: '/usr/sbin/sshd -t -f %s'
    backup: true
  notify: Reload sshd
```
Test on one host first with `--limit` and keep an existing session open.

**Why →** `validate` rejects a broken file, and `reload` keeps current connections alive.

**Interview tip →** Prefer dropping a file in `sshd_config.d/` where supported.

</details>

### Q32. A task should run only on Debian-family hosts and a different one on RedHat. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** OS-conditional logic.

**What I'd do →**
```yaml
- ansible.builtin.apt: { name: nginx, state: present }
  when: ansible_facts['os_family'] == 'Debian'
- ansible.builtin.dnf: { name: nginx, state: present }
  when: ansible_facts['os_family'] == 'RedHat'
```
Or just use `ansible.builtin.package` with a vars map for names that differ.

**Why →** Facts give a stable OS signal; `package` is the generic wrapper.

**Interview tip →** Use `ansible_facts['...']` rather than injected `ansible_os_family` names.

</details>

### Q33. You need to wait for an application to listen on port 8080 after a restart before continuing. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Races between restart and the next step.

**What I'd do →**
```yaml
- ansible.builtin.wait_for:
    port: 8080
    delay: 2
    timeout: 60
- ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
  register: r
  retries: 10
  delay: 3
  until: r.status == 200
```

**Why →** Port open does not mean healthy; the `uri` retry loop proves the app answers.

**Interview tip →** `wait_for` can also wait for a file, a string in a file, or a port to drain.

</details>

### Q34. You want to run a task even in check mode, for example to read a version. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check mode skips commands, which breaks later logic.

**What I'd do →**
```yaml
- ansible.builtin.command: cat /opt/app/VERSION
  register: ver
  check_mode: false
  changed_when: false
```

**Why →** `check_mode: false` forces a real run for a read-only task; `changed_when: false` keeps the report clean.

**Interview tip →** Use `check_mode: true` on a task to force it to be dry-run even in a normal run.

</details>

### Q35. A step is allowed to fail, but you still need to react to the failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Not every failure is fatal.

**What I'd do →** Use `block/rescue/always`, or `ignore_errors` plus `register` for simple cases.

```yaml
- block:
    - ansible.builtin.command: /opt/app/migrate
  rescue:
    - ansible.builtin.command: /opt/app/rollback
    - ansible.builtin.fail: { msg: "migration failed, rolled back" }
  always:
    - ansible.builtin.debug: { msg: "cleanup done" }
```

**Why →** `rescue` runs on failure, `always` runs regardless, and re-failing keeps the run honest.

**Interview tip →** `ignore_errors` hides failures; `failed_when` redefines them. Pick intentionally.

</details>

### Q36. A command returns exit code 2 for "nothing to do", which Ansible counts as failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tool semantics differ from Ansible's default `rc != 0` rule.

**What I'd do →**
```yaml
- ansible.builtin.command: /usr/local/bin/sync-tool
  register: r
  failed_when: r.rc not in [0, 2]
  changed_when: r.rc == 0
```

**Why →** `failed_when` and `changed_when` let you encode the tool's real contract.

**Interview tip →** Prefer an official module; custom conditions are the fallback.

</details>

### Q37. You must ensure a playbook can be re-run after any failure without cleaning up manually. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Idempotent and restartable design.

**What I'd do →**
1. Use state-based modules everywhere (`state: present`).
2. Guard commands with `creates:` or a `stat` check.
3. Write temp files to a path that is cleaned in `always`.
4. Use `--start-at-task` or tags only for resuming, never as a requirement.
5. Prove it: run twice and require `changed=0` in CI.

**Why →** If a half-applied state breaks re-runs, the playbook encodes steps rather than a desired state.

**Interview tip →** "Run it twice" is the single best idempotency test.

</details>

### Q38. How do you run a playbook against a host without making any change but see what would differ? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pre-change review.

**What I'd do →**
```bash
ansible-playbook site.yml --check --diff --limit web01
```

**Why →** `--check` predicts changes where modules support it and `--diff` shows file differences. Tasks using `command`/`shell` are skipped in check mode unless they set `check_mode: false`.

**Interview tip →** Check mode is a prediction, not a guarantee, especially when later tasks depend on earlier ones.

</details>

### Q39. You need a task to run once per play, not once per host, such as sending a Slack notification. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A play-wide side effect.

**What I'd do →**
```yaml
- ansible.builtin.uri:
    url: "{{ slack_webhook }}"
    method: POST
    body_format: json
    body: { text: "Deploy finished" }
  run_once: true
  delegate_to: localhost
```

**Why →** `run_once` executes on the first host in the batch; `delegate_to: localhost` makes the call from the controller.

**Interview tip →** With `serial`, `run_once` fires once per batch; use a condition on `ansible_play_hosts` if you need once overall.

</details>

### Q40. A handler needs to run in the middle of a play, before the next set of tasks. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordering dependency on a handler.

**What I'd do →**
```yaml
- ansible.builtin.template: { src: app.conf.j2, dest: /etc/app.conf }
  notify: Restart app
- ansible.builtin.meta: flush_handlers
- ansible.builtin.uri: { url: http://localhost:8080/health }
```

**Why →** `flush_handlers` runs pending handlers immediately instead of waiting for the end of the play.

**Interview tip →** Handlers also flush at the end of each of `pre_tasks`, `roles/tasks` and `post_tasks` sections.

</details>

### Q41. You want to loop over a list of users and create each with its own groups. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A data-driven loop.

**What I'd do →**
```yaml
vars:
  team:
    - { name: asha, groups: [sudo, docker] }
    - { name: ravi, groups: [docker] }
tasks:
  - ansible.builtin.user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
      append: true
    loop: "{{ team }}"
    loop_control:
      label: "{{ item.name }}"
```

**Why →** `loop_control.label` keeps output short and avoids leaking big dicts into logs.

**Interview tip →** `append: true` is vital, or you remove existing group memberships.

</details>

### Q42. You need to retry a flaky task, like a package mirror, up to five times. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Transient network failures.

**What I'd do →**
```yaml
- ansible.builtin.apt: { name: nginx, update_cache: true }
  register: r
  retries: 5
  delay: 10
  until: r is succeeded
```

**Why →** `until` re-evaluates after each attempt; `delay` avoids hammering the mirror.

**Interview tip →** Retries mask real problems if overused; log and alert on repeated retries.

</details>

### Q43. A deployment must stop on the first host that fails and not touch the rest. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fail-fast rollout.

**What I'd do →**
```yaml
- hosts: web
  serial: 1
  max_fail_percentage: 0
  any_errors_fatal: true
```

**Why →** `serial: 1` isolates risk, and `any_errors_fatal` aborts the play for all hosts when any task fails.

**Interview tip →** Without `serial`, `max_fail_percentage` is evaluated across the whole play, so it behaves differently.

</details>

### Q44. You want to keep a backup of every config file Ansible replaces. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safety net for rollbacks.

**What I'd do →**
```yaml
- ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    backup: true
```

**Why →** The module saves a timestamped copy next to the original, and returns `backup_file`.

**Interview tip →** Backups pile up; long-term, rely on Git for the source and keep backups for emergencies.

</details>

### Q45. Your playbook has 80 tasks and a failure at task 63. You fixed the cause. How do you resume efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Avoid re-running everything.

**What I'd do →**
```bash
ansible-playbook site.yml --start-at-task "Deploy application"
ansible-playbook site.yml --limit @site.retry   # retry file if enabled
```
If the playbook is idempotent, re-running the whole thing is also safe.

**Why →** `--start-at-task` skips earlier tasks (and their registered vars), so use it only when later tasks do not depend on skipped ones.

**Interview tip →** Retry files are disabled by default; use `--limit` with failed hosts from the recap.

</details>

### Q46. How would you ensure a package is at an exact version and never upgraded accidentally? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Version pinning.

**What I'd do →**
```yaml
- ansible.builtin.apt:
    name: nginx=1.24.0-2ubuntu7
    state: present
    allow_downgrade: true
- ansible.builtin.dpkg_selections:
    name: nginx
    selection: hold
```

**Why →** The version string pins the install and `hold` blocks upgrades from `apt upgrade`.

**Interview tip →** On RHEL use `dnf` with `name: nginx-1.24.0` and the `versionlock` plugin.

</details>

### Q47. You need a task to run only the first time a server is built. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One-time bootstrap.

**What I'd do →** Use a marker file with `creates`.

```yaml
- ansible.builtin.command: /opt/bootstrap.sh
  args:
    creates: /var/lib/bootstrap.done
- ansible.builtin.file:
    path: /var/lib/bootstrap.done
    state: touch
    modification_time: preserve
    access_time: preserve
```

**Why →** `creates` makes the task skip once the marker exists.

**Interview tip →** Better still, make bootstrap itself idempotent so a marker is optional.

</details>

### Q48. A task prints a password in the logs. How do you stop that? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret exposure in output.

**What I'd do →**
```yaml
- ansible.builtin.user:
    name: svc
    password: "{{ svc_pw | password_hash('sha512') }}"
  no_log: true
```

**Why →** `no_log: true` hides the task's arguments and results, including from `-v` output.

**Interview tip →** It also hides errors; toggle it off briefly in a safe environment to debug.

</details>

### Q49. How would you delay a restart until a maintenance window? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Changes should apply now but disruptive actions later.

**What I'd do →**
1. Let the config deploy normally and write a flag file or notify a handler that only sets a "restart pending" marker.
2. A scheduled AWX job (or cron on the controller) runs a play with `tags: restart` inside the window.
3. Report pending restarts using `needs-restarting -r` or `/var/run/reboot-required`.

**Why →** Separating configuration from disruption keeps deploys safe and auditable.

**Interview tip →** Mention AWX schedules with a survey-less, tag-limited template.

</details>

### Q50. Your teammate used `command: systemctl restart app` in tasks. What do you say in review? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-idempotent and reports changed every run.

**What I'd do →** Replace with a module and a handler.

```yaml
handlers:
  - name: Restart app
    ansible.builtin.systemd:
      name: app
      state: restarted
      daemon_reload: true
```

**Why →** Modules report real state; handlers restart only when something changed.

**Interview tip →** Use `state: started` plus `enabled: true` in a normal task for the "ensure running" part.

</details>

---

## Variables, Facts and Templates

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. A variable you set in `group_vars` is being overridden and you do not know where. How do you find the source? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Precedence conflict across up to 22 levels.

**What I'd do →**
1. Print the final value: `ansible web01 -m debug -a "var=app_port"`.
2. Search the repo: `grep -rn "app_port" .`.
3. Check the usual suspects: `-e` extra vars, `set_fact`, role `vars/` (higher than `group_vars`), play `vars:`, and `host_vars`.
4. Run with `-vvv` and look at where each var file loads.

**Why →** Role `vars/main.yml`, `set_fact`, `include_vars` and `-e` all outrank `group_vars`.

**Interview tip →** Say you keep tunable defaults in role `defaults/` precisely because it has the lowest precedence.

</details>

### Q52. You need different values per environment without copying playbooks. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One code, many configurations.

**What I'd do →** One inventory per environment, each with `group_vars/all.yml` and group files.

```text
inventories/
  staging/group_vars/all.yml   # app_replicas: 2
  prod/group_vars/all.yml      # app_replicas: 6
```
Run `ansible-playbook -i inventories/prod site.yml`.

**Why →** Environment data lives with the inventory; the playbook stays generic.

**Interview tip →** Avoid `if env == 'prod'` branches in tasks; use variables.

</details>

### Q53. You must render an Nginx config with one `server` block per site from a list. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Template with a loop.

**What I'd do →**
```jinja
{# templates/sites.conf.j2 #}
{% for s in sites %}
server {
    listen {{ s.port | default(80) }};
    server_name {{ s.name }};
    root {{ s.root }};
}
{% endfor %}
```
```yaml
- ansible.builtin.template:
    src: sites.conf.j2
    dest: /etc/nginx/conf.d/sites.conf
    validate: nginx -t -c /etc/nginx/nginx.conf
```

**Why →** Data in YAML, structure in the template; adding a site is a one-line variable change.

**Interview tip →** Use `{%- -%}` whitespace control or `trim_blocks` to keep output tidy.

</details>

### Q54. A template needs the IPs of all hosts in the `db` group, for example for a replication config. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-host data inside a template.

**What I'd do →**
```jinja
{% for h in groups['db'] %}
host replication repl {{ hostvars[h].ansible_default_ipv4.address }}/32 md5
{% endfor %}
```
Make sure facts for the `db` hosts exist, either by gathering in an earlier play or via fact caching.

**Why →** `hostvars` exposes other hosts' variables and facts, but only if gathered in this run or cached.

**Interview tip →** Facts missing causes `'ansible_default_ipv4' is undefined`; gather facts for db hosts first.

</details>

### Q55. Nothing should be rendered unless a mandatory variable is supplied. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fail early on missing input.

**What I'd do →**
```yaml
- ansible.builtin.assert:
    that:
      - db_password is defined
      - db_password | length >= 12
    fail_msg: "db_password must be set and 12+ chars"
```
and use `{{ db_host | mandatory }}` in templates.

**Why →** A clear early failure beats a half-rendered config.

**Interview tip →** `assert` in `pre_tasks` or `tasks/validate.yml` of a role is common.

</details>

### Q56. Per-host values, like `server_id` for MySQL, must be unique and stable. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Generate unique but deterministic values.

**What I'd do →** Either set them in `host_vars/`, or derive from inventory position.

```yaml
mysql_server_id: "{{ groups['db'].index(inventory_hostname) + 1 }}"
```

**Why →** Index in the group is stable as long as hosts are only appended. If you reorder, IDs change, so explicit `host_vars` is safer for databases.

**Interview tip →** Call out the trade-off between convenience and stability.

</details>

### Q57. A fact you need is not collected by default, such as a custom application version. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Extend facts.

**What I'd do →** Drop a script or JSON into `/etc/ansible/facts.d/` on the target.

```yaml
- ansible.builtin.copy:
    dest: /etc/ansible/facts.d/app.fact
    content: '{"version": "{{ app_version }}"}'
    mode: "0644"
- ansible.builtin.setup:
    filter: ansible_local
```
Then use `ansible_local.app.version`.

**Why →** Files ending in `.fact` (JSON, INI or an executable printing JSON) become `ansible_local`.

**Interview tip →** Re-run `setup` after creating the file to refresh within the same play.

</details>

### Q58. Gathering facts takes minutes on 500 hosts and you only need the OS family. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fact gathering is overkill.

**What I'd do →**
```yaml
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.setup:
        gather_subset: [min]
        filter: ansible_os_family
```
Or set `gather_subset = min` in `ansible.cfg`, and enable fact caching.

**Why →** The full subset runs hardware and network probes you rarely need.

**Interview tip →** `gather_facts: false` on plays that never use facts is free speed.

</details>

### Q59. You have to merge two dictionaries, with one overriding the other, such as defaults plus per-site settings. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Deep merge of structured data.

**What I'd do →**
```yaml
app_config: "{{ app_defaults | combine(app_overrides, recursive=True) }}"
```

**Why →** `combine` merges dicts; without `recursive=True` nested keys are replaced wholesale.

**Interview tip →** For lists use `+` or `union`, and `unique` to dedupe.

</details>

### Q60. Passwords in a template must not be world readable. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** File permission hygiene.

**What I'd do →**
```yaml
- ansible.builtin.template:
    src: db.env.j2
    dest: /etc/app/db.env
    owner: app
    group: app
    mode: "0600"
```

**Why →** Quote the mode as a string; an unquoted `0600` is read by YAML as octal in some contexts and as an int in others.

**Interview tip →** Always set `mode` explicitly on secrets.

</details>

### Q61. You want a variable value computed once from a command and reused through the play. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Store results for reuse.

**What I'd do →**
```yaml
- ansible.builtin.command: git -C /opt/app rev-parse --short HEAD
  register: git_rev
  changed_when: false
- ansible.builtin.set_fact:
    release: "{{ git_rev.stdout }}"
    cacheable: true
```

**Why →** `register` lives for the play on that host; `set_fact` with `cacheable` can persist into the fact cache.

**Interview tip →** `set_fact` has high precedence and can shadow inventory variables.

</details>

### Q62. A list of strings in a variable needs to be passed as a comma-separated argument. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Basic filter use.

**What I'd do →**
```yaml
servers: [a, b, c]
line: "{{ servers | join(',') }}"
```

**Why →** `join` is a Jinja filter; others you will use daily are `default`, `map`, `select`, `regex_replace`, `to_nice_json`.

**Interview tip →** Know `map(attribute='name') | list` for extracting a field from a list of dicts.

</details>

### Q63. Your template renders `True` in a config that expects `true`. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Python vs JSON/YAML boolean casing.

**What I'd do →**
```jinja
enabled: {{ feature_x | bool | lower }}
{{ app_settings | to_nice_json }}
```

**Why →** Jinja prints Python's `True`; `to_json`, `to_nice_yaml` or `| lower` produce the target-format spelling.

**Interview tip →** Serialise whole structures with `to_nice_json` rather than hand-assembling braces.

</details>

### Q64. A template for a config file must not be overwritten by Ansible if an admin edited it, but must exist. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Seed once, never clobber.

**What I'd do →**
```yaml
- ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    force: false
```

**Why →** With `force: false` the file is only created if it does not exist.

**Interview tip →** Explain the drift trade-off: this is intentional for admin-owned files; otherwise own the file fully.

</details>

### Q65. A cluster needs a per-host list of all peers except itself. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Peer lists in templates.

**What I'd do →**
```jinja
{% for h in groups['cluster'] if h != inventory_hostname %}
peer {{ hostvars[h].ansible_host | default(h) }}
{% endfor %}
```
Or in YAML: `{{ groups['cluster'] | difference([inventory_hostname]) }}`.

**Why →** `difference` is clean and testable; the loop-with-`if` form is fine inside templates.

**Interview tip →** `ansible_play_hosts` is better than `groups[...]` when you want only hosts still alive in this run.

</details>

### Q66. You need to pull a secret from AWS Secrets Manager at run time rather than keep it in Git. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dynamic secret lookup.

**What I'd do →**
```yaml
db_password: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/db/password', region='ap-south-1') }}"
```
Mark consuming tasks `no_log: true`, and give the controller an IAM role with `secretsmanager:GetSecretValue` on that ARN only.

**Why →** Lookups run on the controller, so targets never need AWS permissions.

**Interview tip →** Compare with Vault (file-based), HashiCorp Vault lookup and AWX credential injection.

</details>

### Q67. Different servers need different values but the template has dozens of `if` blocks. How do you simplify? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Template complexity is a smell.

**What I'd do →**
1. Push decisions into variables (`group_vars`), not template branching.
2. Use a dictionary lookup by OS or role instead of if/elif chains.
3. Split into several templates if the files genuinely differ.

```yaml
pkg: "{{ {'Debian':'apache2','RedHat':'httpd'}[ansible_facts['os_family']] }}"
```

**Why →** Data-driven config is easier to review and test.

**Interview tip →** Templates should format data, not decide policy.

</details>

### Q68. Your play uses `with_items` and you have been asked to modernise. What changes? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Legacy loop syntax.

**What I'd do →** Replace `with_items: "{{ list }}"` by `loop: "{{ list }}"`. For nested or flattened lists use filters.

```yaml
loop: "{{ users | product(groups) | list }}"      # was with_nested
loop: "{{ dict | dict2items }}"                   # was with_dict
```

**Why →** `loop` is the recommended keyword; `with_*` still works but hides what is happening.

**Interview tip →** `with_items` flattens one level automatically; `loop` does not, so use `| flatten(1)`.

</details>

### Q69. A variable contains Jinja that should not be evaluated at definition time, like a string for another system. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Escaping templating.

**What I'd do →**
```yaml
prom_label: "{% raw %}{{ $labels.instance }}{% endraw %}"
motd: !unsafe "Welcome {{ user }}"
```

**Why →** `{% raw %}` stops Jinja from evaluating the braces, and `!unsafe` marks a YAML string as never templated.

**Interview tip →** Prometheus alert and Grafana templates inside Ansible templates are the classic example.

</details>

### Q70. How do you detect that a registered result was skipped or failed and branch on it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use result tests rather than raw fields.

**What I'd do →**
```yaml
- ansible.builtin.command: grep -q ready /tmp/state
  register: r
  failed_when: false
- ansible.builtin.debug: { msg: "not ready" }
  when: r is failed
- ansible.builtin.debug: { msg: "changed" }
  when: r is changed
```

**Why →** `is failed`, `is changed`, `is skipped` and `is succeeded` are readable and robust.

**Interview tip →** Avoid the legacy `r|failed` filter syntax.

</details>

### Q71. You want to read a YAML file on the controller and use its contents as variables. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** External data source.

**What I'd do →**
```yaml
- ansible.builtin.include_vars:
    file: "vars/{{ env_name }}.yml"
    name: env_cfg
- ansible.builtin.debug: { var: env_cfg.api_url }
```
Or `lookup('ansible.builtin.file', ...) | from_yaml`.

**Why →** `include_vars` loads at run time and `name:` namespaces it to avoid precedence surprises.

**Interview tip →** `include_vars` outranks `group_vars` and play vars (it's a "role/include vars" level).

</details>

### Q72. A play uses a variable that is defined only for some hosts. How do you handle the rest? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Optional variable.

**What I'd do →**
```yaml
- ansible.builtin.debug:
    msg: "{{ proxy_url | default(omit) }}"
- ansible.builtin.apt: { name: curl }
  environment:
    http_proxy: "{{ proxy_url | default('') }}"
  when: proxy_url is defined
```

**Why →** `default(omit)` drops a module parameter entirely; `is defined` guards a task.

**Interview tip →** `default('')` with empty string does not trigger for variables set to empty; use `default(x, true)` for that.

</details>

### Q73. You must generate a config containing a random password once and then keep it stable on later runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Idempotent random data.

**What I'd do →**
```yaml
app_secret: "{{ lookup('ansible.builtin.password', '/secure/creds/' ~ inventory_hostname ~ '.txt length=32 chars=ascii_letters,digits') }}"
```

**Why →** The `password` lookup creates the file on the first call and reads it back later, so the value stays the same. It runs on the controller, so protect that directory.

**Interview tip →** Better for production: generate in a secrets manager and look it up.

</details>

### Q74. A teammate puts `ansible_python_interpreter` in every play. Where should it live? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Connection config duplicated.

**What I'd do →** Move it to inventory (`group_vars/all.yml` or host vars), or set `interpreter_python = auto_silent` in `ansible.cfg`.

```yaml
# group_vars/legacy.yml
ansible_python_interpreter: /usr/bin/python3.9
```

**Why →** It is a property of the host, not the play.

**Interview tip →** Modern Ansible discovers Python automatically (`auto`).

</details>

### Q75. A Jinja expression builds a path from several variables and sometimes yields a double slash. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** String concatenation sloppiness.

**What I'd do →**
```yaml
dest: "{{ base_dir | trim('/') }}/{{ app }}"
dest2: "{{ [base_dir, app] | join('/') | regex_replace('/+', '/') }}"
dest3: "{{ (base_dir ~ '/' ~ app) | path_join }}"
```

**Why →** `path_join` (Ansible 2.10+) normalises separators; plain concatenation does not.

**Interview tip →** Use `~` for string concatenation; `+` can fail on mixed types.

</details>

---

## Roles, Collections and Project Structure

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. You have a 600-line playbook that sets up web, db and monitoring. How do you restructure it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A monolith that is hard to reuse and test.

**What I'd do →**
1. Create roles: `roles/common`, `roles/nginx`, `roles/postgres`, `roles/node_exporter`.
2. Make `site.yml` import small playbooks, such as `web.yml` and `db.yml`.
3. Move values to `defaults/main.yml` and inventory.

```yaml
- hosts: web
  roles: [common, nginx, node_exporter]
```

**Why →** Roles give reuse, a standard layout and a natural unit for Molecule tests.

**Interview tip →** Keep playbooks as thin mappings of hosts to roles.

</details>

### Q77. Where do you put a variable users of your role are expected to override? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Public vs internal role variables.

**What I'd do →** `defaults/main.yml` for the public interface, `vars/main.yml` for internal constants.

```yaml
# roles/nginx/defaults/main.yml
nginx_worker_processes: auto
nginx_listen_port: 80
```

**Why →** Defaults have the lowest precedence, so any inventory or play var wins; `vars/` is high precedence and hard to override.

**Interview tip →** Prefix variables with the role name to avoid collisions.

</details>

### Q78. Two roles need to run in a fixed order, such as `common` before `nginx`. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Dependency between roles.

**What I'd do →** Declare it in `meta/main.yml`.

```yaml
# roles/nginx/meta/main.yml
dependencies:
  - role: common
```

**Why →** Dependencies run first, and by default only once per play even if several roles list them.

**Interview tip →** Prefer explicit ordering in the playbook when the dependency is policy rather than a hard requirement.

</details>

### Q79. A role is used twice with different parameters in one play, but only runs once. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Role de-duplication.

**What I'd do →** Set `allow_duplicates: true` in the role's `meta/main.yml`, or use `include_role` with vars.

```yaml
# meta/main.yml
allow_duplicates: true
```
```yaml
- ansible.builtin.include_role: { name: vhost, vars: { site: a.example.com } }
- ansible.builtin.include_role: { name: vhost, vars: { site: b.example.com } }
```

**Why →** Roles with identical parameters are deduplicated by default.

**Interview tip →** A loop over `include_role` is another common pattern.

</details>

### Q80. You want to share roles across several repos and teams. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Distribution and versioning.

**What I'd do →**
1. Package roles in a collection (`namespace.collection`) or keep one repo per role.
2. Pin versions in `requirements.yml`.

```yaml
collections:
  - name: acme.platform
    version: 1.4.2
    source: https://galaxy.internal.example.com
roles:
  - src: git+https://git.example.com/ansible/nginx.git
    version: v2.1.0
    name: acme.nginx
```
Install with `ansible-galaxy install -r requirements.yml`.

**Why →** Pinned versions give reproducible builds.

**Interview tip →** Use a private Galaxy/Automation Hub or Artifactory for internal content.

</details>

### Q81. How do you pin Ansible itself so a CI run today behaves like one last month? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tool-version drift.

**What I'd do →**
1. Pin `ansible-core` and Python dependencies in `requirements.txt` (and lock hashes).
2. Pin collections in `requirements.yml`.
3. Better, bake both into an Execution Environment image or a container used by CI.

```text
ansible-core==2.17.*
```

**Why →** Module behaviour changes between versions; a floating install makes failures irreproducible.

**Interview tip →** Say you upgrade deliberately on a schedule with a test run.

</details>

### Q82. A role installs a package and writes a config, but must support Ubuntu and Rocky Linux. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** OS-specific variables.

**What I'd do →**
```yaml
# tasks/main.yml
- ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
        - default.yml
      paths: [vars]
- ansible.builtin.package: { name: "{{ nginx_pkg }}", state: present }
```

**Why →** The data differs per OS and the tasks stay the same.

**Interview tip →** This `first_found` pattern is what well-known community roles use.

</details>

### Q83. Your role's tasks directory has grown to a single 400-line file. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Poor readability.

**What I'd do →** Split into files and include them.

```yaml
# tasks/main.yml
- ansible.builtin.import_tasks: install.yml
- ansible.builtin.import_tasks: configure.yml
- ansible.builtin.import_tasks: service.yml
```
Tag each import so users can run only `configure`.

**Why →** Small files with a clear name map to the lifecycle and are easy to review.

**Interview tip →** `import_tasks` is static (parsed upfront); `include_tasks` is dynamic (needed for loops or runtime filenames).

</details>

### Q84. When would you pick `include_tasks` over `import_tasks`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static vs dynamic reuse.

**What I'd do →** Use `include_tasks` when the file name depends on a variable, when you loop over the include, or when you need `when` applied once at runtime. Use `import_tasks` when you want tags and `--list-tasks` to see everything.

```yaml
- ansible.builtin.include_tasks: "{{ ansible_facts['os_family'] }}.yml"
```

**Why →** Imports are expanded before the run; includes are evaluated as the play reaches them, so tags on the include apply only to the include itself.

**Interview tip →** `--list-tasks` does not show tasks inside dynamic includes.

</details>

### Q85. A role should work on Ansible 2.15 to 2.18 and you must declare that. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Metadata for consumers and Galaxy.

**What I'd do →** Fill `meta/main.yml`, and for collections set `requires_ansible` in `meta/runtime.yml`.

```yaml
galaxy_info:
  role_name: nginx
  min_ansible_version: "2.15"
  platforms:
    - { name: Ubuntu, versions: [jammy, noble] }
```
```yaml
# collection meta/runtime.yml
requires_ansible: ">=2.15.0,<2.19"
```

**Why →** Consumers get a clear error instead of an obscure failure, and CI can test the matrix.

**Interview tip →** Test the declared matrix in CI, otherwise the range is wishful.

</details>

### Q86. You have to ship a custom module and filter with your role. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Role-local plugins.

**What I'd do →** Put them in the role: `roles/myrole/library/my_module.py`, `roles/myrole/filter_plugins/my_filters.py`. They become available to that role and anything after it in the play.

For wider reuse, put them in a collection under `plugins/modules` and `plugins/filter`, then call them by FQCN.

**Why →** Collections give namespacing, versioning and documentation hooks.

**Interview tip →** Mention `ansible-doc` and `DOCUMENTATION` strings for modules.

</details>

### Q87. A teammate wants to publish an internal collection. What goes in it and how do you build it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Packaging for sharing.

**What I'd do →**
```bash
ansible-galaxy collection init acme.platform
# add roles/, plugins/, playbooks/, galaxy.yml
ansible-galaxy collection build
ansible-galaxy collection publish acme-platform-1.4.2.tar.gz --server internal
```
Add `galaxy.yml` metadata with `namespace`, `name`, `version`, `dependencies`.

**Why →** A tarball is versioned and installable anywhere.

**Interview tip →** Lint with `ansible-lint` and sanity-test with `ansible-test sanity` before publishing.

</details>

### Q88. Your project needs a standard layout. What do you show a new hire? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Project structure.

**What I'd do →**
```text
ansible.cfg
requirements.yml
site.yml
inventories/{staging,prod}/{hosts,group_vars/,host_vars/}
playbooks/
roles/
collections/
molecule/           # per role
```

**Why →** Predictable placement means anyone can find variables and roles quickly.

**Interview tip →** Mention one repo, one `ansible.cfg` checked in, and `.gitignore` for vault password files.

</details>

### Q89. A role has a task that should run only if the user opts in, e.g. install the monitoring agent. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Feature toggle.

**What I'd do →**
```yaml
# defaults/main.yml
app_enable_monitoring: false
# tasks/main.yml
- ansible.builtin.import_tasks: monitoring.yml
  when: app_enable_monitoring | bool
```

**Why →** Boolean flags in defaults give a documented switch.

**Interview tip →** `| bool` guards against the string `"false"` passed with `-e`.

</details>

### Q90. How do you test that a role does not break when its variable `x` is missing? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Validate input handling.

**What I'd do →** Add an `argument_specs` file so Ansible validates inputs before the role runs.

```yaml
# roles/nginx/meta/argument_specs.yml
argument_specs:
  main:
    options:
      nginx_listen_port: { type: int, default: 80 }
      nginx_sites: { type: list, elements: dict, required: true }
```

**Why →** Missing or wrongly typed variables fail early with a readable message and documented defaults.

**Interview tip →** `ansible-doc -t role` can display those specs.

</details>

### Q91. You need to upgrade a community role without surprising changes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Third-party update risk.

**What I'd do →**
1. Read the changelog and diff old vs new tag.
2. Bump the pin in a branch.
3. Run Molecule/`--check --diff` against staging.
4. Roll to prod via the normal pipeline.

**Why →** Roles change defaults over time; a pinned, reviewed upgrade avoids silent behaviour shifts.

**Interview tip →** Fork when upstream is abandoned, but track it.

</details>

### Q92. Roles in your repo access each other's internal variables. How do you reduce coupling? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tight coupling.

**What I'd do →**
1. Define a clear public interface in `defaults/` and `argument_specs`.
2. Pass data in via role parameters, not by reading another role's vars.
3. Share common facts via a `common` role or inventory variables.

**Why →** Roles that only depend on declared inputs are replaceable and testable alone.

**Interview tip →** Say "a role is a function: inputs in defaults, effects on the host".

</details>

### Q93. A role must run a task on another host, such as registering with a load balancer. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Delegation inside a role.

**What I'd do →**
```yaml
- community.general.haproxy:
    backend: web
    host: "{{ inventory_hostname }}"
    state: enabled
  delegate_to: "{{ groups['lb'][0] }}"
```

**Why →** Variables still resolve for the original host, while the action runs on the LB.

**Interview tip →** Be aware `ansible_*` connection vars come from the delegate host in newer versions; check with `-vvv`.

</details>

### Q94. How do you apply only part of a role, for instance only its configuration? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Selective execution.

**What I'd do →** Tag tasks and use `--tags`.

```yaml
- ansible.builtin.template: { src: app.conf.j2, dest: /etc/app.conf }
  tags: [config]
```
```bash
ansible-playbook site.yml --tags config
```

**Why →** Tags let you apply a slice without editing code.

**Interview tip →** Tags on `import_tasks` apply to every task inside; on `include_tasks` only to the include.

</details>

### Q95. Migrating a legacy pre-2.10 playbook that uses short module names to modern Ansible. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Short names such as `yum:` or `ec2:` break or are ambiguous.

**What I'd do →**
1. Run `ansible-lint` and let it flag `fqcn`.
2. Use `ansible-lint --fix` for mechanical changes.
3. Install the collections they now live in (`amazon.aws`, `community.general`).
4. Replace removed modules (`ec2` becomes `ec2_instance`).

**Why →** FQCNs make the source of every module explicit.

**Interview tip →** Core builtins are `ansible.builtin.*`.

</details>

### Q96. You are asked to make a role idempotent and prove it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Evidence required.

**What I'd do →** Use Molecule's default scenario, which runs `converge` then `idempotence`.

```bash
molecule test
```
The idempotence step fails if the second converge reports any `changed` task.

**Why →** It automates "run twice, expect zero changes".

**Interview tip →** Fix offenders with `changed_when` or a proper module, not by disabling the check.

</details>

### Q97. A role requires a handler from another role. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-role notification.

**What I'd do →** Handlers are global to the play, so a task can notify by name. But this couples roles; better to use `listen` topics.

```yaml
# role A
notify: "reload web stack"
# role B handler
- name: Reload nginx
  ansible.builtin.service: { name: nginx, state: reloaded }
  listen: "reload web stack"
```

**Why →** A topic name is a stable contract; handler names are not.

**Interview tip →** Include roles dynamically with `include_role` and handlers become visible only after that point.

</details>

### Q98. You need to run roles in a loop for each tenant. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-tenancy.

**What I'd do →**
```yaml
- ansible.builtin.include_role:
    name: tenant
  vars:
    tenant_name: "{{ item.name }}"
    tenant_port: "{{ item.port }}"
  loop: "{{ tenants }}"
  loop_control: { label: "{{ item.name }}" }
```

**Why →** `include_role` supports loops; `roles:` does not.

**Interview tip →** Use `allow_duplicates` or unique params so tasks aren't skipped.

</details>

### Q99. How would you review a pull request that adds a new role? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Quality gate.

**What I'd do →** Check: FQCNs, `ansible-lint` clean, idempotent (Molecule), `defaults` documented, no secrets, `no_log` where needed, handlers used instead of restarts, `become` scoped narrowly, README with variables, tags where useful, supports declared OS list.

**Why →** A checklist avoids opinion-based review.

**Interview tip →** Mention automating the checklist in CI.

</details>

### Q100. A role repo and a playbook repo each carry their own `ansible.cfg`. Which one wins? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config lookup order.

**What I'd do →** Know the order: `ANSIBLE_CONFIG` env var, then `./ansible.cfg` in the current directory, then `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`. Only one file is used; they do not merge. Check with `ansible --version` (shows config file) and `ansible-config dump --only-changed`.

**Why →** The working directory at invocation decides; world-writable directories are ignored for security.

**Interview tip →** Mention the "ignoring ansible.cfg in a world writable directory" warning.

</details>

---

## Rolling Updates, Delegation and Orchestration

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. You must deploy a new app version to 40 web servers behind a load balancer with zero downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling update: take a few nodes out, update, verify, put back.

**What I'd do →**
1. Use `serial` to limit the batch size.
2. Drain each node from the LB with `delegate_to`.
3. Deploy, health check, re-enable.

```yaml
- hosts: web
  serial: [1, "10%", "25%"]
  max_fail_percentage: 10
  pre_tasks:
    - community.general.haproxy: { backend: web, host: "{{ inventory_hostname }}", state: disabled, wait: true }
      delegate_to: lb01
  roles: [app]
  post_tasks:
    - ansible.builtin.uri: { url: "http://{{ inventory_hostname }}:8080/health" }
      register: h
      until: h.status == 200
      retries: 15
      delay: 4
    - community.general.haproxy: { backend: web, host: "{{ inventory_hostname }}", state: enabled }
      delegate_to: lb01
```

**Why →** The canary-first `serial` list limits risk, and the health gate stops the roll before it spreads.

**Interview tip →** Mention `max_fail_percentage` and `any_errors_fatal` as stop conditions.

</details>

### Q102. What does `serial: [1, 5, "50%"]` do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Escalating batches.

**What I'd do →** Explain: first batch is 1 host, the next is 5, then 50% of the total in each remaining batch until all hosts are done.

**Why →** You pay a small cost to learn early whether the release is bad.

**Interview tip →** `serial` batches run the whole play to completion on one batch before starting the next.

</details>

### Q103. During a rolling update, 3 of 10 hosts in a batch fail. Does the play continue? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Failure thresholds.

**What I'd do →** Check `max_fail_percentage`. Default behaviour: failed hosts drop out and the rest continue. If the failure percentage in a batch exceeds `max_fail_percentage`, the play stops for all hosts. `any_errors_fatal: true` stops everything on the first failure.

**Why →** Thresholds are evaluated per batch when `serial` is used.

**Interview tip →** A percentage of 0 means "any failure aborts".

</details>

### Q104. You must restart database replicas one at a time but never the primary first. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ordering matters and roles differ.

**What I'd do →** Two plays: replicas with `serial: 1`, then the primary after a controlled failover.

```yaml
- hosts: db_replicas
  serial: 1
  tasks:
    - ansible.builtin.include_role: { name: pg_upgrade }
    - ansible.builtin.command: pg_isready
      register: r
      until: r.rc == 0
      retries: 20
      delay: 5

- hosts: db_primary
  tasks:
    - ansible.builtin.command: patronictl switchover --force --candidate "{{ new_primary }}"
      run_once: true
    - ansible.builtin.include_role: { name: pg_upgrade }
```

**Why →** Ansible plays run in order, so order is explicit and each stage is gated.

**Interview tip →** Check replication lag before moving on.

</details>

### Q105. You need a task to run on the LB for every web host, without logging into each web host. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `delegate_to` use.

**What I'd do →** Use `delegate_to: lb01` on the task; it runs on the LB once per web host, with `inventory_hostname` still the web host.

**Why →** Useful for registering and deregistering backends, DNS updates and monitoring downtimes.

**Interview tip →** Add `run_once: true` only if the action should happen a single time.

</details>

### Q106. A shell command must run on localhost for each remote host, e.g. a CLI call to a cloud API. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Controller-side call per host.

**What I'd do →**
```yaml
- ansible.builtin.command: "aws ec2 create-tags --resources {{ instance_id }} --tags Key=Patched,Value=true"
  delegate_to: localhost
  become: false
```

**Why →** `become: false` stops it from trying sudo on the controller.

**Interview tip →** Prefer the collection module over the CLI.

</details>

### Q107. A deployment should run strictly one host at a time, but fact gathering should not be slow. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `serial: 1` gathers facts per batch, increasing wall time.

**What I'd do →** Gather facts in a first play for all hosts, then run the serial play with `gather_facts: false`, or with cached facts.

```yaml
- hosts: web
  gather_facts: true
  tasks: []
- hosts: web
  serial: 1
  gather_facts: false
```

**Why →** Facts are collected in parallel once, not 40 times sequentially.

**Interview tip →** Facts live in memory for the whole run, so the second play still sees them.

</details>

### Q108. Hosts need rebooting after a kernel patch, one at a time, and the play must wait for them to return. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe reboot.

**What I'd do →**
```yaml
- hosts: all
  serial: 1
  tasks:
    - ansible.builtin.reboot:
        reboot_timeout: 900
        post_reboot_delay: 20
        test_command: systemctl is-system-running --wait
```

**Why →** The `reboot` module handles the disconnect, polls until SSH returns and runs a test command.

**Interview tip →** Never use `shell: reboot` with async; the module exists for this.

</details>

### Q109. You need to check a condition on all hosts first and only then change any of them. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** All-or-nothing gate.

**What I'd do →** Use a preflight play with `any_errors_fatal`.

```yaml
- hosts: db
  any_errors_fatal: true
  tasks:
    - ansible.builtin.assert:
        that: ansible_facts['mounts'] | selectattr('mount','equalto','/') | map(attribute='size_available') | first > 5000000000
        fail_msg: "Less than 5 GB free on /"
- hosts: db
  roles: [upgrade]
```

**Why →** Failing in the first play means no host has changed yet.

**Interview tip →** A strong answer includes a rollback strategy, not only a gate.

</details>

### Q110. How do you run a long job (a 45-minute backup) on many hosts without holding SSH sessions? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Long tasks and connection timeouts.

**What I'd do →**
```yaml
- ansible.builtin.command: /opt/backup/run.sh
  async: 3600
  poll: 0
  register: job
- ansible.builtin.async_status:
    jid: "{{ job.ansible_job_id }}"
  register: res
  until: res.finished
  retries: 120
  delay: 30
```

**Why →** `poll: 0` fires and forgets, and `async_status` checks later, so no session is held open between polls.

**Interview tip →** `async` tasks do not work with check mode and some connection types.

</details>

### Q111. You must run a play on hosts in a certain order, such as db, then app, then web. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tier ordering.

**What I'd do →** Use several plays in one playbook in that order.

```yaml
- import_playbook: db.yml
- import_playbook: app.yml
- import_playbook: web.yml
```

**Why →** Plays execute strictly in file order; `order: sorted|reverse_sorted|shuffle` only controls host order inside a play.

**Interview tip →** Put `import_playbook` at the top level only.

</details>

### Q112. A task on host A needs data produced by host B in the same play. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cross-host data flow.

**What I'd do →** Register on B, read with `hostvars` from A, and make sure B's task ran first (strategy `linear` guarantees that per task).

```yaml
- hosts: db_primary
  tasks:
    - ansible.builtin.command: pg_current_wal_lsn
      register: lsn
- hosts: db_replica
  tasks:
    - ansible.builtin.debug: { msg: "{{ hostvars[groups['db_primary'][0]].lsn.stdout }}" }
```

**Why →** `hostvars` exposes registered variables from earlier plays in the same run.

**Interview tip →** In a single play with `free` strategy ordering is not guaranteed.

</details>

### Q113. What is the `free` strategy and when would you use it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hosts have very different speeds.

**What I'd do →** Set `strategy: free` on the play so each host proceeds as fast as it can instead of waiting for the slowest host at each task.

**Why →** Great for independent long-running work like package updates; bad when later tasks depend on all hosts finishing earlier steps.

**Interview tip →** Output is interleaved, so logs are harder to read.

</details>

### Q114. Two engineers run the same playbook against production at the same time. How do you prevent it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Concurrency control.

**What I'd do →** Run Ansible only through AWX or a CI job with a concurrency group, so only one job per inventory runs. As a fallback, take a lock via a task on localhost:

```yaml
- ansible.builtin.command: flock -n /tmp/deploy.lock true
  delegate_to: localhost
  run_once: true
```
In AWX, disable "Allow Simultaneous Runs" on the template.

**Why →** Concurrent runs cause races on the same hosts and unpredictable handler behaviour.

**Interview tip →** CI "concurrency" or AWX job slicing are the cleaner answers.

</details>

### Q115. How would you do a blue/green deployment with Ansible? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Two identical stacks, switch traffic at the end.

**What I'd do →**
1. Determine the idle colour from a variable or the LB state.
2. Deploy and smoke test the idle stack.
3. Switch the LB target group or DNS.
4. Keep the old stack for the rollback window, then destroy.

```yaml
- hosts: "{{ idle_color }}"
  roles: [app]
- hosts: localhost
  tasks:
    - amazon.aws.elb_application_lb:
        name: app-alb
        listeners:
          - Protocol: HTTP
            Port: 80
            DefaultActions: [{ Type: forward, TargetGroupName: "tg-{{ idle_color }}" }]
        state: present
```

**Why →** Cutover is one atomic change, and rollback is flipping it back.

**Interview tip →** Compare with rolling for cost and database migration complexity.

</details>

### Q116. A handler restarts a service but the restart takes 30 seconds and later tasks fail on connection refused. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Race after restart.

**What I'd do →** Flush handlers, then wait for readiness.

```yaml
- ansible.builtin.meta: flush_handlers
- ansible.builtin.wait_for: { port: 5432, timeout: 120 }
```

**Why →** Handlers normally run at the end of the section, so you must flush before dependent tasks.

**Interview tip →** Use health-check endpoints rather than only port checks.

</details>

### Q117. You must update a cluster where only one node can be down at a time and you must know the quorum is healthy before the next node. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Stateful clusters (etcd, Kafka, Elasticsearch).

**What I'd do →**
1. `serial: 1`.
2. Pre-check cluster health from the node or via delegation.
3. Update, restart, wait for rejoin.
4. Post-check health; fail the play otherwise.

```yaml
- ansible.builtin.uri:
    url: "http://{{ inventory_hostname }}:9200/_cluster/health"
    return_content: true
  register: ch
  until: ch.json.status == 'green'
  retries: 60
  delay: 10
```

**Why →** The gate prevents a second node going down while the cluster is still recovering.

**Interview tip →** Mention disabling shard allocation first for Elasticsearch.

</details>

### Q118. A playbook should send a notification when it starts and when it finishes, even if it fails. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Lifecycle notifications.

**What I'd do →** Wrap the work in `block/always`, or use callback plugins. A simple pattern:

```yaml
- hosts: web
  tasks:
    - block:
        - ansible.builtin.include_role: { name: app }
      always:
        - community.general.slack:
            token: "{{ slack_token }}"
            msg: "Deploy {{ 'FAILED' if ansible_failed_task is defined else 'ok' }} on {{ inventory_hostname }}"
          delegate_to: localhost
```

**Why →** `always` runs after both success and failure, though not if the host is unreachable.

**Interview tip →** AWX has native notification templates for start/success/fail.

</details>

### Q119. You need to throttle a task so only 2 hosts hit the artifact server at once, while others run in parallel. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Per-task concurrency.

**What I'd do →**
```yaml
- ansible.builtin.get_url: { url: "{{ artifact_url }}", dest: /opt/app.tar.gz }
  throttle: 2
```

**Why →** `throttle` limits workers for that task only, without changing forks or `serial`.

**Interview tip →** `throttle` is also useful for rate-limited APIs.

</details>

### Q120. After a deployment you want to run smoke tests from the controller against each host. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Post-deploy verification.

**What I'd do →**
```yaml
- ansible.builtin.uri:
    url: "https://{{ inventory_hostname }}/health"
    validate_certs: true
    status_code: 200
  delegate_to: localhost
```

**Why →** The controller tests the same path users take; failures stop the batch.

**Interview tip →** Fail the play on a failed smoke test to prevent promoting a broken release.

</details>

### Q121. Your rolling update stopped halfway because of a failure. How do you finish safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Partial rollout, mixed versions.

**What I'd do →**
1. Identify updated vs pending hosts from the PLAY RECAP or a version check.
2. Fix the root cause.
3. Re-run limited to the remaining hosts.
4. Confirm the final version everywhere.

```bash
ansible web -m ansible.builtin.command -a "/opt/app/bin/app --version" -o
ansible-playbook deploy.yml --limit @/tmp/pending.txt
```

**Why →** An idempotent play skips hosts that are already correct.

**Interview tip →** Design for mixed-version tolerance during rollouts.

</details>

### Q122. A play must act on a host that is unreachable at the start but may come back soon. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unreachable hosts are dropped from the play.

**What I'd do →** Use `wait_for_connection` early, and `meta: clear_host_errors` to re-include hosts that failed earlier.

```yaml
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.wait_for_connection: { timeout: 300 }
    - ansible.builtin.setup:
```

**Why →** Hosts marked unreachable stay out for the rest of the play; waiting before facts gathering prevents that.

**Interview tip →** `meta: clear_host_errors` resets failed state for the play.

</details>

### Q123. You want one task to run only on the last host of a batch. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Batch-end action.

**What I'd do →**
```yaml
- ansible.builtin.debug: { msg: "last in batch" }
  when: inventory_hostname == ansible_play_batch[-1]
```

**Why →** `ansible_play_batch` lists the hosts in the current serial batch, and `ansible_play_hosts` lists all active hosts in the play.

**Interview tip →** `run_once` combined with `serial` is the more common route.

</details>

### Q124. How do you orchestrate steps across tiers where each depends on the previous, like provision, configure, deploy, test? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-stage workflow.

**What I'd do →** Build a top-level playbook that imports stage playbooks. In AWX, express it as a workflow job template with success/failure edges and approvals.

```yaml
- import_playbook: provision.yml
- import_playbook: configure.yml
- import_playbook: deploy.yml
- import_playbook: verify.yml
```

**Why →** Each stage is independently runnable and testable, while the workflow gives order and approval gates.

**Interview tip →** Mention approval nodes for production.

</details>

### Q125. You need to patch hosts in a specific order, lowest-traffic first, and the order is stored as a variable. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Custom host ordering that `order:` cannot express.

**What I'd do →** Define explicit numeric tiers per host and run one play per tier, or give `serial` a fixed ordering through group membership.

```yaml
# host_vars: patch_wave: 1 / 2 / 3
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.group_by:
        key: "wave_{{ patch_wave }}"
- import_playbook: patch.yml
  vars: { wave_group: wave_1 }
- import_playbook: patch.yml
  vars: { wave_group: wave_2 }
```
with `hosts: "{{ wave_group }}"` and `serial` inside `patch.yml`.

**Why →** Play `order` only supports `inventory`, `reverse_inventory`, `sorted`, `reverse_sorted` and `shuffle`; anything else needs explicit grouping.

**Interview tip →** If you cannot do it simply, ask whether the ordering is truly required.

</details>

---

## Security: Vault, Become and Compliance

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. Database passwords are sitting in plain text in `group_vars`. How do you fix it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets in Git.

**What I'd do →**
1. Encrypt the file: `ansible-vault encrypt group_vars/prod/vault.yml`.
2. Keep a plain `vars.yml` that references `vault_` variables.
3. Pass the password via `--vault-password-file` or `--vault-id`.
4. Rotate any password that was ever committed in plain text.

```yaml
# group_vars/prod/vars.yml
db_password: "{{ vault_db_password }}"
```

**Why →** The prefix pattern keeps names greppable while values stay encrypted.

**Interview tip →** Rotating is non-negotiable; Git history still holds the old value.

</details>

### Q127. You need separate vault passwords for staging and prod. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Different trust domains.

**What I'd do →** Use vault IDs.

```bash
ansible-vault encrypt --vault-id staging@prompt group_vars/staging/vault.yml
ansible-vault encrypt --vault-id prod@~/.vault_prod group_vars/prod/vault.yml
ansible-playbook site.yml --vault-id staging@~/.vault_stg --vault-id prod@~/.vault_prod
```

**Why →** Each file's header carries its label, and Ansible tries the matching password, so developers can hold staging only.

**Interview tip →** Production vault password lives in CI/AWX credentials, not on laptops.

</details>

### Q128. How do you change the vault password for all encrypted files? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rotation after someone leaves.

**What I'd do →**
```bash
ansible-vault rekey --vault-id prod@old_pw_file --new-vault-id prod@new_pw_file group_vars/prod/*.yml
```
Then rotate the underlying secrets too, because the ex-employee may have read them.

**Why →** Rekey protects future access to the files, not secrets already seen.

**Interview tip →** Say both: rekey the vault and rotate the secrets.

</details>

### Q129. A CI pipeline needs the vault password without storing it in the repo. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secure delivery to automation.

**What I'd do →** Store it as a masked CI secret and write it to a temp file or use a script that prints it from a secret manager.

```bash
echo "$VAULT_PASS" > "$RUNNER_TEMP/vp" && chmod 600 "$RUNNER_TEMP/vp"
ansible-playbook site.yml --vault-password-file "$RUNNER_TEMP/vp"
shred -u "$RUNNER_TEMP/vp"
```
A cleaner method: `--vault-password-file ./vault-pass.sh` where the script fetches from AWS Secrets Manager.

**Why →** If the file is executable, Ansible runs it and uses stdout.

**Interview tip →** Never echo the secret in logs; mask it.

</details>

### Q130. You want to see the value inside an encrypted file without writing a decrypted copy to disk. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inspecting a secret safely.

**What I'd do →**
```bash
ansible-vault view group_vars/prod/vault.yml
ansible-vault edit group_vars/prod/vault.yml
```

**Why →** `view` shows it in the pager, and `edit` uses a temp file then re-encrypts.

**Interview tip →** Set `EDITOR` and be careful with shell history for `encrypt_string`.

</details>

### Q131. Only one value in a vars file is secret and you want it in the same file as plain values. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inline encryption.

**What I'd do →**
```bash
ansible-vault encrypt_string --vault-id prod@prompt 's3cr3t' --name 'db_password'
```
Paste the resulting `!vault |` block into the YAML.

**Why →** It keeps diffs readable for non-secret lines.

**Interview tip →** Whole-file encryption hides structure from reviewers; inline shows the keys. Choose by team preference.

</details>

### Q132. A task needs root on the target but your user has only limited sudo rights. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least-privilege become.

**What I'd do →**
1. Scope `become` to tasks that need it, not the whole play.
2. Ask ops to give a sudoers drop-in for specific commands if possible.
3. Use `become_user` for service accounts.

```yaml
- ansible.builtin.command: systemctl restart app
  become: true
- ansible.builtin.command: /opt/app/bin/report
  become: true
  become_user: appsvc
```

**Why →** Minimal escalation reduces blast radius and audit scope.

**Interview tip →** `NOPASSWD` for a service account over a fixed command list is the production norm.

</details>

### Q133. `become_user` to an unprivileged account fails with "Failed to set permissions on the temporary files". 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both users are unprivileged, so the temp module file isn't readable by `become_user`.

**What I'd do →** Enable pipelining (`pipelining = True`), which avoids temp files, or install `acl` so Ansible can use `setfacl`, or set `allow_world_readable_tmpfiles = true` (less secure).

```ini
[ssh_connection]
pipelining = True
```

**Why →** Without pipelining Ansible writes the module as login user and needs the become user to read it.

**Interview tip →** Pipelining requires `requiretty` to be off in sudoers.

</details>

### Q134. You must prove to auditors which changes were made and by whom. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Audit trail.

**What I'd do →**
1. Run everything through AWX/CI so each job records user, inventory, commit, and output.
2. Forward job events to SIEM.
3. Use named SSH users, not shared ones, and sudo logging on targets.
4. Enable `log_path` or callbacks (`json`, `ara`) on the controller.

**Why →** Git tells what was intended; the job log tells what ran.

**Interview tip →** ARA Records Ansible gives a searchable run history.

</details>

### Q135. You are asked to enforce a CIS baseline on 300 Linux hosts and show drift reports. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compliance as code.

**What I'd do →**
1. Use a maintained role (for example, the CIS role set) or write controls as tasks tagged by control ID.
2. Run in `--check --diff` mode on a schedule to report drift without changing anything.
3. Run in enforce mode on a change window.
4. Export results to a report.

```bash
ansible-playbook cis.yml --check --diff --tags level1 | tee drift-$(date +%F).log
```

**Why →** Check mode as a scheduled audit turns Ansible into a drift detector.

**Interview tip →** Be honest that check mode has gaps; pair with OpenSCAP scans.

</details>

### Q136. A playbook sets `ansible_ssh_pass` in plain text for a Cisco device. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hard-coded password.

**What I'd do →** Move the credential to Vault or an AWX credential and reference it via a variable. Prefer key-based auth for SSH.

```yaml
ansible_password: "{{ vault_cisco_password }}"
```

**Why →** Any value in version control must be encrypted, and logs must not show it (`no_log`).

**Interview tip →** AWX credentials inject into jobs without being visible to users.

</details>

### Q137. How do you disable root SSH login and password authentication across the fleet safely? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** High-risk hardening change.

**What I'd do →**
1. Confirm key-based sudo access works for the Ansible user on a canary.
2. Apply with `validate` and a `reload` handler.
3. Open a second SSH session before the change.
4. Roll with `serial`.

```yaml
- ansible.builtin.copy:
    dest: /etc/ssh/sshd_config.d/10-hardening.conf
    content: |
      PermitRootLogin no
      PasswordAuthentication no
    mode: "0600"
    validate: /usr/sbin/sshd -t -f %s
  notify: Reload sshd
```

**Why →** A mistake here removes your own access.

**Interview tip →** Note that `validate` with a drop-in file only validates syntax of the fragment; test the full config when possible.

</details>

### Q138. A sudo password is required on the targets. How does Ansible get it without prompting each time? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interactive become password.

**What I'd do →** Use `-K` (`--ask-become-pass`) interactively, or supply `ansible_become_password` from Vault in unattended runs. Better, use passwordless sudo restricted to needed commands for the automation user.

**Why →** Prompting breaks automation; plain text breaks security; vault or sudoers rules are the compromise.

**Interview tip →** AWX stores it in a credential.

</details>

### Q139. A teammate committed `.vault_pass` to Git. What now? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Leaked master secret.

**What I'd do →**
1. Treat every vault encrypted with it as compromised.
2. Rekey vaults with a new password and rotate the secrets inside.
3. Remove the file from history if policy requires (`git filter-repo`), but assume it is already exposed.
4. Add it to `.gitignore` and a pre-commit secret scanner.

**Why →** Deleting the file does not delete history or clones.

**Interview tip →** Say "rotate first, clean history second".

</details>

### Q140. You need to store an SSH private key that Ansible deploys to hosts. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret file distribution.

**What I'd do →** Encrypt the whole key file with vault and use `copy` with `content`.

```yaml
- ansible.builtin.copy:
    content: "{{ vault_deploy_key }}\n"
    dest: /home/deploy/.ssh/id_ed25519
    owner: deploy
    mode: "0600"
  no_log: true
```

**Why →** `no_log` avoids exposing the key in output and diffs.

**Interview tip →** Use `--diff` carefully; it can print file contents. Use `no_log` or `diff: false`.

</details>

### Q141. Running `--diff` on a template leaks a secret into the CI log. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Diff output reveals file content.

**What I'd do →** Disable diff on that task.

```yaml
- ansible.builtin.template:
    src: db.env.j2
    dest: /etc/app/db.env
    mode: "0600"
  diff: false
  no_log: true
```

**Why →** The `diff: false` task keyword suppresses diff output for the task.

**Interview tip →** Both keywords exist: `diff` shows content differences, `no_log` hides module args and result.

</details>

### Q142. A new hire needs access to staging vault only. How do you set that up? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least-privilege secrets.

**What I'd do →** Give them only the staging vault ID password and use separate vault files per environment so the prod files remain unreadable.

**Why →** The prod password is never shared, so prod files stay opaque to them.

**Interview tip →** Combine with separate inventories and CI roles.

</details>

### Q143. How do you ensure an unattended playbook run cannot execute arbitrary commands via extra vars? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** User input reaching `command`/`shell` is code execution.

**What I'd do →**
1. Validate inputs with `assert` and `argument_specs`.
2. Quote with `| quote` in shell strings.
3. Avoid `shell` when a module exists.
4. In AWX, use surveys with typed fields and disallow ad hoc `-e` for non-admins ("prompt on launch" off).

```yaml
- ansible.builtin.shell: "rm -rf {{ cleanup_dir | quote }}"
```

**Why →** Unvalidated variables into shell lines are injection vectors.

**Interview tip →** Mention `!unsafe` for data that must not template.

</details>

### Q144. You want to use HashiCorp Vault instead of Ansible Vault. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Central secrets management with dynamic secrets.

**What I'd do →**
```yaml
db_password: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=kv/data/prod/db:password auth_method=approle role_id=... secret_id=... url=https://vault.example.com:8200') }}"
```
Use AppRole/JWT or cloud auth, short TTLs, and `no_log`. Install `hvac` on the controller.

**Why →** Central policy, audit and rotation instead of files in Git.

**Interview tip →** AWX can inject credentials via credential lookup plugins for Vault.

</details>

### Q145. You discover that `become` tasks leave module temp files in the user's home. How do you reduce that exposure? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Temporary files and permissions.

**What I'd do →** Enable pipelining to avoid writing modules, set `remote_tmp` to a protected location, and make sure `ansible_remote_tmp` is not world-readable.

```ini
[defaults]
remote_tmp = /tmp/.ansible-${USER}/tmp
[ssh_connection]
pipelining = True
```

**Why →** Pipelining streams the module over stdin and leaves nothing on disk (except for async).

**Interview tip →** Pipelining plus sudo requires `!requiretty`.

</details>

### Q146. How do you make sure a password given to `user` module is stored hashed? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The `user` module expects a hash in `password:`.

**What I'd do →**
```yaml
- ansible.builtin.user:
    name: svc
    password: "{{ vault_svc_pw | password_hash('sha512', 'fixedsalt') }}"
    update_password: on_create
```

**Why →** A fixed salt keeps the hash stable between runs, or `update_password: on_create` prevents repeated changes.

**Interview tip →** Plain text in `password:` would set a literal (invalid) hash.

</details>

### Q147. Compliance requires that all package installs come from an internal mirror. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Controlled supply chain.

**What I'd do →** Manage repo files with Ansible, remove others, and verify GPG.

```yaml
- ansible.builtin.yum_repository:
    name: internal-base
    description: Internal base
    baseurl: https://mirror.example.com/rhel9/baseos/
    gpgcheck: true
    gpgkey: https://mirror.example.com/RPM-GPG-KEY-internal
- ansible.builtin.file: { path: /etc/yum.repos.d/rocky.repo, state: absent }
```

**Why →** Ansible enforces the repo list on every run, so drift is corrected.

**Interview tip →** Add a firewall rule blocking outbound internet to back it up.

</details>

### Q148. An engineer asks if `ansible_become_pass` in an inventory is acceptable. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret in inventory.

**What I'd do →** Only if the value is vaulted or injected from a credential store; plain text is not acceptable.

**Why →** Inventory is checked in and widely readable.

**Interview tip →** Offer the alternative: sudoers `NOPASSWD` for the automation account with limited commands.

</details>

### Q149. You must rotate the SSH key used by Ansible across all hosts without losing access. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Key rotation with rollback safety.

**What I'd do →**
1. Add the new public key (keep the old one).
2. Verify login with the new key on every host.
3. Switch the controller/AWX credential.
4. Remove the old key.

```yaml
- ansible.posix.authorized_key:
    user: ansible
    key: "{{ lookup('file', 'new_ansible.pub') }}"
    state: present
# later
- ansible.posix.authorized_key:
    user: ansible
    key: "{{ lookup('file', 'old_ansible.pub') }}"
    state: absent
```

**Why →** Overlap prevents lockouts.

**Interview tip →** Use `exclusive: true` only after you are sure.

</details>

### Q150. How would you scan your Ansible repo for security problems before merge? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shift-left.

**What I'd do →** In CI run `ansible-lint` (with the production profile), `yamllint`, a secret scanner such as gitleaks, and `ansible-playbook --syntax-check`. Fail the build on findings.

```bash
ansible-lint --profile production
gitleaks detect --source .
```

**Why →** Cheap automated gates catch plain-text secrets and bad practices before they reach a server.

**Interview tip →** `ansible-lint` profiles go from `min` to `production`.

</details>

---

## Performance, Testing and CI/CD

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. A playbook that touches 200 hosts takes 40 minutes. Where do you start tuning? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unknown bottleneck; measure first.

**What I'd do →**
1. Enable timing callbacks: `callbacks_enabled = profile_tasks, timer` in `ansible.cfg`.
2. Raise `forks` (for example 50), enable `pipelining`, and SSH `ControlPersist`.
3. Turn off or reduce fact gathering and add fact caching.
4. Replace loops of package installs with one `name: [list]` call.
5. Use `async` for long independent tasks and `strategy: free` where safe.

```ini
[defaults]
forks = 50
gathering = smart
callbacks_enabled = profile_tasks, timer
[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

**Why →** Most time is connection overhead and slow tasks; profiling points at the few that matter.

**Interview tip →** Say "measure, then tune" and name `profile_tasks`.

</details>

### Q152. You install 15 packages with a loop and it is slow. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One module call per package means 15 round trips and 15 package-manager runs.

**What I'd do →**
```yaml
- ansible.builtin.apt:
    name: [git, curl, htop, jq, unzip]
    state: present
    update_cache: true
    cache_valid_time: 3600
```

**Why →** The package manager resolves everything in one transaction.

**Interview tip →** `cache_valid_time` avoids repeated `apt update`.

</details>

### Q153. How would you test a role in CI before it reaches production? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Automated role testing.

**What I'd do →** Use Molecule with a container driver (Docker/Podman).

```yaml
# molecule/default/molecule.yml
driver: { name: docker }
platforms:
  - name: ubuntu24
    image: geerlingguy/docker-ubuntu2404-ansible
    pre_build_image: true
    command: ""
    privileged: true
provisioner: { name: ansible }
verifier: { name: ansible }
```
```bash
molecule test      # create, converge, idempotence, verify, destroy
```

**Why →** The sequence proves the role applies, is idempotent, and meets assertions.

**Interview tip →** Add a matrix of OS images to check portability.

</details>

### Q154. What would a Molecule `verify.yml` check for an Nginx role? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Assertions on the end state.

**What I'd do →**
```yaml
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.service_facts:
    - ansible.builtin.assert:
        that: "ansible_facts.services['nginx.service'].state == 'running'"
    - ansible.builtin.uri:
        url: http://localhost
        status_code: 200
```

**Why →** Assert behaviour (running, listening, responding), not implementation details.

**Interview tip →** Testinfra (Python) is the older verifier option.

</details>

### Q155. Molecule's idempotence step fails. How do you find the offending task? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some task changes on the second run.

**What I'd do →**
1. Read the log: Molecule prints the tasks that changed.
2. Run `molecule converge` twice and look at the second output for `changed:`.
3. Fix with real modules, `changed_when`, or `creates`.

**Why →** Typical causes: `command`/`shell` without guards, `lineinfile` with a bad regexp, templates containing timestamps.

**Interview tip →** Never "fix" by skipping the idempotence step.

</details>

### Q156. Design a CI pipeline for an Ansible repo. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** End-to-end quality gates.

**What I'd do →**
1. Lint: `yamllint`, `ansible-lint`.
2. Syntax: `ansible-playbook --syntax-check` for each playbook.
3. Unit/integration: Molecule per changed role.
4. Dry run on staging: `--check --diff`.
5. Deploy to staging; smoke tests.
6. Manual approval; deploy to prod with `serial`.

```mermaid
flowchart LR
  A["Commit"] --> B["Lint"] --> C["Syntax check"] --> D["Molecule"] --> E["Staging check mode"] --> F["Staging deploy"] --> G["Approval"] --> H["Prod rolling deploy"]
```

**Why →** Cheaper checks run first; expensive ones only after they pass.

**Interview tip →** Run only changed roles in Molecule to keep the pipeline fast.

</details>

### Q157. `--check` shows no change but the real run changes things. Why? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check mode is a prediction.

**What I'd do →** Look for tasks that depend on earlier skipped tasks' results, `command`/`shell` tasks skipped in check mode (so registered values are empty), and modules without check support.

**Why →** If task B reads a file task A would have created, B sees the old state in check mode.

**Interview tip →** Use `check_mode: false` only for read-only tasks, and test in a throwaway environment for full confidence.

</details>

### Q158. How do you lint playbooks and fail the build only on serious issues? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gradual lint adoption.

**What I'd do →**
```yaml
# .ansible-lint
profile: moderate
skip_list: [yaml[line-length]]
warn_list: [experimental]
```
```bash
ansible-lint
```

**Why →** Profiles raise the bar over time (`min`, `basic`, `moderate`, `safety`, `shared`, `production`).

**Interview tip →** Use `# noqa rule-name` sparingly with a comment on why.

</details>

### Q159. How do you test a playbook that talks to real cloud APIs without paying for resources? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cloud provisioning tests are slow and costly.

**What I'd do →**
1. Separate provisioning from configuration; test configuration with containers (Molecule).
2. Test provisioning against a sandbox account with tagged, auto-expiring resources.
3. Use LocalStack for supported AWS services via `endpoint_url`.
4. Always run a `destroy` in `always`.

**Why →** Config logic is where most bugs live and it tests cheaply in containers.

**Interview tip →** Tag all test resources and run a nightly janitor.

</details>

### Q160. A pipeline step must fail if a run had any `changed` tasks (drift detection). 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Use Ansible as a drift checker.

**What I'd do →** Run in check mode and parse the recap, or use the JSON callback.

```bash
ANSIBLE_STDOUT_CALLBACK=json ansible-playbook site.yml --check > out.json
jq '[.stats[] | .changed] | add' out.json
```
Fail when the sum is greater than zero.

**Why →** In check mode `changed` means "would change", which is drift.

**Interview tip →** Schedule it in AWX nightly with notifications.

</details>

### Q161. Fact gathering is the slowest part of a nightly run. How do you cache facts? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated expensive probing.

**What I'd do →**
```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /var/cache/ansible/facts
fact_caching_timeout = 86400
```
Use Redis for a shared cache across controllers.

**Why →** `smart` gathers only when no valid cache exists.

**Interview tip →** Don't cache facts for fast-changing values like free memory in decisions.

</details>

### Q162. You want an Ansible run to execute on the target host itself, pulling config from Git, instead of pushing from a controller. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Pull model for autoscaled or firewalled hosts.

**What I'd do →** Use `ansible-pull` from cron or a systemd timer, or from user-data at boot.

```bash
ansible-pull -U https://git.example.com/infra/ansible.git -C main -i localhost, local.yml
```

**Why →** Hosts configure themselves and scale without a central push.

**Interview tip →** You lose central visibility, so report results to a webhook or use AWX callbacks.

</details>

### Q163. Your controller runs out of memory with `forks=200`. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each fork is a Python process with the playbook loaded.

**What I'd do →**
1. Lower forks to what memory allows (roughly 100–150 MB per fork for big runs).
2. Split the inventory with AWX job slicing, or run multiple smaller jobs.
3. Reduce the data in hostvars; avoid huge registered results.
4. Use `no_log` or `loop_control.label` to avoid giant output.

**Why →** Memory scales with forks and the size of inventory data.

**Interview tip →** AWX job slicing is the scalable answer.

</details>

### Q164. You want a nightly job that runs a safe subset of tasks: health checks only. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tagged read-only tasks.

**What I'd do →**
```yaml
- ansible.builtin.command: df -h /
  changed_when: false
  tags: [healthcheck, never]
```
```bash
ansible-playbook site.yml --tags healthcheck
```

**Why →** The special tag `never` keeps tasks out of normal runs unless you ask for their other tag.

**Interview tip →** `always` runs every time; `never` runs only if explicitly selected.

</details>

### Q165. A teammate asks if tags are enough to protect production from partial runs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tags are convenience, not safety.

**What I'd do →** Explain they are easy to misuse (a missing tag skips a needed task). For safety, use separate playbooks or workflow nodes, limits enforced in AWX, and approval gates.

**Why →** `--tags` can leave dependencies unexecuted, such as a package install skipped before a config task.

**Interview tip →** Use `always` tags for prerequisites like fact gathering and variable loading.

</details>

### Q166. How do you run a quick syntax and dry run check on all playbooks locally before pushing? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast local feedback.

**What I'd do →**
```bash
ansible-playbook playbooks/*.yml --syntax-check
ansible-lint
yamllint .
```
Add them as a `pre-commit` hook.

**Why →** Syntax check catches parse errors; lint catches bad practice.

**Interview tip →** `--syntax-check` doesn't validate module arguments; Molecule and check mode do.

</details>

### Q167. A job must produce machine-readable reports for a dashboard. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Structured output.

**What I'd do →** Use the `json` or `junit` callback, or write a CSV from the play.

```bash
ANSIBLE_STDOUT_CALLBACK=yaml ansible-playbook site.yml
ANSIBLE_CALLBACKS_ENABLED=junit JUNIT_OUTPUT_DIR=./reports ansible-playbook site.yml
```

**Why →** JUnit XML plugs into CI test reports natively.

**Interview tip →** AWX exposes job events through its REST API too.

</details>

### Q168. How would you validate Jinja templates without a real server? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Template testing.

**What I'd do →** Render with `ansible localhost -m template` or `ansible.builtin.template` into `/tmp` in Molecule, then check the output.

```bash
ansible localhost -m ansible.builtin.template \
  -a "src=templates/app.conf.j2 dest=/tmp/app.conf" -e @tests/vars.yml
```
Run the target tool's own validator (`nginx -t`) on the rendered file in Molecule.

**Why →** You catch undefined variables and syntax early.

**Interview tip →** Keep test vars files in the role's `molecule/` folder.

</details>

### Q169. Playbook runs against 1000 hosts are failing intermittently with SSH timeouts. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Likely concurrency overload on the control node or network.

**What I'd do →**
1. Check forks vs. bastion `MaxStartups` and controller CPU.
2. Set `timeout = 30` and `retries = 3` under `[ssh_connection]`.
3. Reduce forks, enable ControlPersist and pipelining.
4. Use `serial` to limit batch size.

**Why →** Too many simultaneous handshakes drop connections.

**Interview tip →** Distinguish "unreachable" (connection) from "failed" (task) in the recap.

</details>

### Q170. How do you keep a task from running when invoked with `--check`, because it only makes sense for real runs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Check-mode-aware logic.

**What I'd do →**
```yaml
- ansible.builtin.command: /opt/app/send-metrics
  when: not ansible_check_mode
```

**Why →** `ansible_check_mode` is a special variable that is true during `--check`.

**Interview tip →** `ansible_diff_mode` is the equivalent for `--diff`.

</details>

### Q171. You need to confirm a role runs on both Ubuntu 22.04 and Rocky 9. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Multi-platform testing.

**What I'd do →** Add platforms to `molecule.yml` and pass the image via env var so CI can matrix it.

```yaml
platforms:
  - name: instance
    image: "${MOLECULE_DISTRO:-ubuntu2404}"
```
In CI run a job matrix over `ubuntu2204`, `rockylinux9`.

**Why →** Parallel CI jobs per distro keep wall time low.

**Interview tip →** Use images with systemd for service tests.

</details>

### Q172. How do you promote the same tested change through dev, staging and prod using Git? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Release process.

**What I'd do →** Trunk-based: merge to main, pipeline runs on staging automatically, a tag or manual approval triggers prod with the same commit and pinned requirements.

**Why →** The artefact (repo + pinned deps / EE image) is identical across environments; only inventory differs.

**Interview tip →** Avoid per-environment branches; they drift.

</details>

### Q173. Developers complain that the Ansible toolchain differs across laptops and CI. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Reproducibility.

**What I'd do →** Build an Execution Environment with `ansible-builder`, and run it locally with `ansible-navigator`.

```yaml
# execution-environment.yml
version: 3
images:
  base_image: { name: quay.io/ansible/ansible-runner:latest }
dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt
```
```bash
ansible-builder build -t acme/ee:1.0
ansible-navigator run site.yml --eei acme/ee:1.0 -m stdout
```

**Why →** Same image in AWX, CI and laptops means same behaviour.

**Interview tip →** EEs replaced Tower's virtualenvs.

</details>

### Q174. A task intermittently shows `changed` due to timestamps in a templated file. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Non-deterministic template output.

**What I'd do →** Remove `{{ ansible_date_time.iso8601 }}` or similar from templates, or use `ansible_managed` (a static string).

```jinja
# {{ ansible_managed }}
```

**Why →** Any value that changes per run makes the file differ each time.

**Interview tip →** Idempotency means output depends only on inputs.

</details>

### Q175. You are asked to estimate the effect of a change on production without applying it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe preview.

**What I'd do →**
```bash
ansible-playbook site.yml -i inventories/prod --check --diff --limit web01,db01
```
Review changes, then run on a canary.

**Why →** Check plus diff shows concrete deltas, though not everything.

**Interview tip →** Say the preview is advisory; a staged rollout is the real safety.

</details>

---

## AWX, Cloud and Large-Scale Operations

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. Your company wants a self-service UI for running playbooks with RBAC. What do you propose? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Central execution with access control.

**What I'd do →** Deploy AWX (or Red Hat Ansible Automation Platform). Model: Organization, Project (Git), Inventory, Credentials, Job Template, then RBAC roles (Execute, Admin, Read) per team.

**Why →** Users can run approved automation without seeing credentials, and every run is logged.

**Interview tip →** AWX is the upstream of AAP's automation controller.

</details>

### Q177. How do you run AWX on Kubernetes? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Modern AWX deployment.

**What I'd do →** Install the AWX Operator, then create an `AWX` custom resource.

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata: { name: awx }
spec:
  service_type: ClusterIP
  ingress_type: ingress
  hostname: awx.example.com
```
Job pods run in the cluster through container groups.

**Why →** The operator manages Postgres, upgrades and the task/web/ee components.

**Interview tip →** The old docker-compose installer is gone.

</details>

### Q178. A job template should prompt for the target limit and a version, with validation. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Safe user input in AWX.

**What I'd do →** Enable "Prompt on launch" for limit and add a Survey with typed fields (multiple choice for version).

**Why →** A survey constrains values, unlike free-form extra vars.

**Interview tip →** Survey answers become extra vars, which have the highest precedence.

</details>

### Q179. You need provisioning followed by configuration followed by an approval and then deploy. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** AWX workflow.

**What I'd do →** Build a Workflow Job Template with nodes: provision, configure, approval, deploy; use "on success" and "on failure" edges, and a notification on failure.

**Why →** Visual flow, approvals and branching without custom scripting.

**Interview tip →** Workflows can pass data between nodes with `set_stats`.

</details>

### Q180. A dynamic inventory source in AWX syncs slowly and stale hosts remain. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Inventory sync settings.

**What I'd do →** Enable "Overwrite" and "Update on launch", set a cache timeout, and set a schedule.

**Why →** Overwrite removes hosts no longer in the source; update-on-launch keeps data fresh per job.

**Interview tip →** Use `source_vars` to apply the same `keyed_groups` as the CLI plugin.

</details>

### Q181. You manage 5,000 servers and a single job takes hours. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs horizontal scaling.

**What I'd do →**
1. Enable Job Slicing (for example 10 slices) so the inventory is split into parallel jobs.
2. Add execution nodes or instance groups.
3. Tune forks per template.
4. Use fact caching and skip unnecessary gathering.

**Why →** Slices run on different nodes in parallel with isolated memory.

**Interview tip →** Slicing divides hosts, not tasks, so plays must be host-independent.

</details>

### Q182. How do you let a team run jobs in a private network without exposing the controller to it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Remote execution.

**What I'd do →** Use Receptor mesh with an execution node (or hop node) inside the private network; assign it to an instance group and attach that to the inventory or template.

**Why →** The control plane stays central while jobs execute locally to the targets.

**Interview tip →** Automation mesh replaced isolated nodes.

</details>

### Q183. A job template needs the same playbook but different credentials per environment. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Credential separation.

**What I'd do →** Create separate machine credentials (and vault credentials) and templates per environment, or allow credential prompt-on-launch with RBAC.

**Why →** Teams only get Use permission on the credentials for their environment.

**Interview tip →** Credentials are never shown after saving, even to admins.

</details>

### Q184. You must provision AWS infrastructure with Ansible. How would you structure it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cloud provisioning.

**What I'd do →**
1. Use `amazon.aws` modules (`ec2_instance`, `ec2_security_group`, `elb_application_lb`).
2. Run on `localhost` with `connection: local`.
3. Auth with an IAM role or environment credentials, not keys in the repo.
4. Make every call idempotent by naming and tagging.

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - amazon.aws.ec2_instance:
        name: app-01
        image_id: "{{ ami }}"
        instance_type: t3.small
        vpc_subnet_id: "{{ subnet }}"
        tags: { Env: prod, Role: app }
        state: running
```

**Why →** Tags identify existing resources, so reruns do not create duplicates.

**Interview tip →** Be ready to say Terraform provisions, Ansible configures, and why.

</details>

### Q185. Terraform creates instances and Ansible must configure them. How do you hand over? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tool boundary.

**What I'd do →** Tag instances in Terraform and use `aws_ec2` dynamic inventory in Ansible, or export a Terraform output to an inventory file. Trigger Ansible after `apply` in CI.

**Why →** Tags decouple the tools; no static file to sync.

**Interview tip →** `cloud-init` can bootstrap a minimal Ansible user and Python.

</details>

### Q186. You need to rotate TLS certificates on 200 hosts without downtime. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Rolling cert update.

**What I'd do →** Distribute with `copy` (vaulted or fetched from a secret store), validate, reload service (not restart), `serial: 10%`, and verify with `openssl s_client` or `uri`.

```yaml
- ansible.builtin.copy:
    src: "{{ cert_src }}"
    dest: /etc/ssl/app.pem
    mode: "0640"
  notify: Reload nginx
```

**Why →** Reload swaps certs without dropping connections.

**Interview tip →** Better: automate renewal (ACME) and let Ansible only distribute.

</details>

### Q187. How do you manage Kubernetes resources with Ansible? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ansible and k8s.

**What I'd do →** Use `kubernetes.core.k8s` and `kubernetes.core.helm` modules from localhost.

```yaml
- kubernetes.core.helm:
    name: ingress
    chart_ref: ingress-nginx/ingress-nginx
    release_namespace: ingress
    create_namespace: true
    values: { controller: { replicaCount: 3 } }
```

**Why →** Good for bootstrapping clusters and glue; GitOps tools (Argo CD) own the steady state.

**Interview tip →** Install the `kubernetes` Python package on the controller.

</details>

### Q188. A 2 AM incident: a bad config went to all web servers. How do you roll back with Ansible? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast, controlled rollback.

**What I'd do →**
1. Check out the last known good commit/tag.
2. Run the same playbook with `serial` and the previous version variable.
3. Verify health; notify.
4. Fix forward later through the normal pipeline.

```bash
git checkout v1.8.3
ansible-playbook deploy.yml -e app_version=1.8.3 --limit web
```

**Why →** Config as code means rollback is a redeploy of an older commit.

**Interview tip →** Keep previous release directories on the host for instant symlink rollback.

</details>

### Q189. How do you onboard 300 existing hand-built servers into Ansible management safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Brownfield adoption.

**What I'd do →**
1. Start read-only: inventory and facts audit.
2. Write roles to match reality, run in `--check --diff` to find drift.
3. Fix intentional drift in code; fix accidental drift in hosts.
4. Roll enforce mode out by tier.

**Why →** Jumping straight to enforce can break unknown snowflakes.

**Interview tip →** Mention "converge to code, not code to servers" decisions made per diff.

</details>

### Q190. How would you use Ansible to patch Linux servers monthly? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Patch management.

**What I'd do →**
```yaml
- hosts: all
  serial: "20%"
  tasks:
    - ansible.builtin.dnf: { name: "*", state: latest, security: true }
      register: upd
    - ansible.builtin.reboot:
      when: upd is changed
```
Schedule in AWX, patch non-prod a week earlier, and skip hosts in maintenance.

**Why →** Rings and batches catch issues early.

**Interview tip →** `needs-restarting -r` decides if a reboot is really required.

</details>

### Q191. A playbook should run daily and email results only on change or failure. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Schedules and notifications.

**What I'd do →** AWX schedule plus a notification template on "failure" and "success" with message templates. Without AWX use cron and the `mail` module in a final play based on a registered changed count.

**Why →** Notify on exception keeps noise low.

**Interview tip →** Use separate templates for audit and enforce jobs.

</details>

### Q192. You need to run a playbook against devices that only support SSH CLI, such as switches. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Network automation.

**What I'd do →** Use `ansible_connection: ansible.netcommon.network_cli`, `ansible_network_os`, and platform collections.

```yaml
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios
ansible_become: true
ansible_become_method: enable
```
Use resource modules (for example `cisco.ios.ios_vlans`) for idempotent config.

**Why →** Network modules run on the controller, not the device.

**Interview tip →** Use `state: gathered` and `rendered` for audit workflows.

</details>

### Q193. An ops team wants Ansible results visible in a dashboard of what changed where. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Run history and reporting.

**What I'd do →** Use AWX job history and its API, or install ARA Records Ansible as a callback for non-AWX runs.

**Why →** Searchable run data helps audits and incident review.

**Interview tip →** Stream AWX logs to an external aggregator (Splunk/ELK) via logging settings.

</details>

### Q194. How do you manage per-host differences for hundreds of hosts without hundreds of files? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scalable variable design.

**What I'd do →** Define attributes (role, tier, region) as groups or tags, and derive values via group_vars and `keyed_groups`. Use host_vars only for true exceptions.

**Why →** Fewer files, more consistent fleet, easier audit.

**Interview tip →** "Cattle not pets": hosts are described by attributes.

</details>

### Q195. A play needs to destroy a test environment on schedule to save money. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scheduled teardown.

**What I'd do →**
```yaml
- amazon.aws.ec2_instance:
    filters: { "tag:Env": test, instance-state-name: running }
    state: absent
```
Schedule the job in AWX nightly, and use a protection tag to exempt instances.

**Why →** Filters make it declarative and repeatable.

**Interview tip →** Dry-run with `--check` and require approval for anything tagged prod.

</details>

### Q196. How do you roll out Ansible changes to the automation platform itself safely? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Operating the controller.

**What I'd do →** Keep AWX/EE versions pinned, upgrade a staging AWX first, take Postgres backups before operator upgrades, test job templates after upgrade.

**Why →** The platform is a high-privilege system with credentials to everything.

**Interview tip →** Restrict who can edit projects and credentials; treat AWX as production.

</details>

### Q197. A developer wants to run playbooks directly from their laptop against prod. Your policy? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Change control.

**What I'd do →** Allow laptops for dev/staging and `--check`; require AWX/CI for prod with approvals so credentials stay centralised and runs are logged.

**Why →** Personal machines are unaudited and credentials leak.

**Interview tip →** Offer break-glass access with logging.

</details>

### Q198. A new region must be bootstrapped with the exact same stack as an existing one. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeatable environment creation.

**What I'd do →** Create a new inventory directory (`inventories/prod-eu/`) with its group_vars, and run the same site playbook; infrastructure comes from the same Terraform module with different variables.

**Why →** A parameterised codebase reproduces identically.

**Interview tip →** Differences should be only variables and inventory.

</details>

### Q199. How do you decide between a push model with AWX and `ansible-pull` for a fleet of 10,000 edge devices? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Scale and connectivity trade-off.

**What I'd do →** Edge devices behind NAT and intermittently connected favour pull: they run `ansible-pull` on a timer. For centrally controlled data-centre fleets, push through AWX with mesh nodes. A hybrid is common: pull for baseline, push for urgent change.

**Why →** Pull scales without inbound access but loses real-time orchestration and central logging.

**Interview tip →** Add a signed Git tag or commit verification for pull.

</details>

### Q200. Explain to a manager why you chose Ansible and what its limits are. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Business justification.

**What I'd do →** Say: agentless, readable YAML, large module ecosystem, good for configuration, orchestration and glue, low learning curve. Limits: not a state-tracking provisioner like Terraform, slower at massive scale than agent-based tools, and large inventories need AWX or tuning.

**Why →** Choosing the tool for the job and naming its limits builds trust.

**Interview tip →** Pair Ansible with Terraform and GitOps rather than defending it for everything.

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
