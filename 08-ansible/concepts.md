# Ansible — Core Concepts

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

> Core concepts, internals, comparisons and "what happens when…" questions for Ansible and configuration management · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Fundamentals and Architecture](#fundamentals-and-architecture) — Q1–Q25
- [Inventory](#inventory) — Q26–Q50
- [Ad-hoc Commands, Modules and Idempotency](#ad-hoc-commands-modules-and-idempotency) — Q51–Q75
- [Playbooks, Plays, Tasks and Handlers](#playbooks-plays-tasks-and-handlers) — Q76–Q100
- [Variables, Facts and Precedence](#variables-facts-and-precedence) — Q101–Q125
- [Jinja2, Conditionals and Loops](#jinja2-conditionals-and-loops) — Q126–Q150
- [Roles, Collections and Galaxy](#roles-collections-and-galaxy) — Q151–Q175
- [Vault, Privilege Escalation and Error Handling](#vault-privilege-escalation-and-error-handling) — Q176–Q200

---

## Fundamentals and Architecture

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What is Ansible and what problem does it solve? 🟢

<details>
<summary><b>Answer</b></summary>

Ansible is an open-source automation engine for configuration management, application deployment, provisioning and orchestration. You describe the desired state of servers in YAML, and Ansible connects to them and makes them match.

The problem it solves is configuration drift and manual, snowflake-server administration. Instead of SSH-ing into 50 boxes and running commands by hand, you run one playbook that is reviewable in Git and repeatable on every host.

</details>

### Q2. What does "agentless" mean in Ansible? 🟢

<details>
<summary><b>Answer</b></summary>

No daemon or agent has to be installed on managed nodes. Ansible uses the transport the node already has: SSH for Linux/Unix, WinRM (or SSH) for Windows, and API connections for network devices and cloud services.

The practical wins are no agent to patch, no extra open port, and a node can be managed minutes after it boots. The trade-off is that the control node does the work over the network, so very large fleets need tuning (forks, pipelining) or a pull model such as `ansible-pull`.

</details>

### Q3. What are the requirements for the control node? 🟢

<details>
<summary><b>Answer</b></summary>

Any Linux or macOS machine (or WSL on Windows) with a supported Python 3 and `ansible-core` installed. Native Windows cannot be a control node.

```bash
python3 -m pip install --user ansible-core    # or: pip install ansible
ansible --version
```

Recent `ansible-core` releases (2.18/2.19) require Python 3.11+ on the controller, so older distros often need a venv or `pipx`.

</details>

### Q4. What does a managed node need? 🟢

<details>
<summary><b>Answer</b></summary>

For Linux: an SSH server, a user that can log in (ideally key-based), and a Python interpreter, because most modules are Python programs that run on the target. If the user is not root, it also needs `sudo` or another become method.

Without Python you can only use `raw` (and `script`) until you bootstrap it, for example with `ansible all -m raw -a "dnf install -y python3" --become`.

</details>

### Q5. Is Ansible push-based or pull-based? 🟢

<details>
<summary><b>Answer</b></summary>

Push by default: the control node initiates the connection and pushes tasks to the nodes when you run a playbook. There is no continuous reconciliation, so drift is only fixed when you run again.

Pull is possible with `ansible-pull`, which runs on each node, clones a Git repo and applies a local playbook, usually from cron or a systemd timer. In production we use it for autoscaled instances that must configure themselves at boot.

</details>

### Q6. Walk through what happens when you run `ansible-playbook site.yml`. 🟡

<details>
<summary><b>Answer</b></summary>

1. Ansible loads `ansible.cfg`, the inventory, and parses the playbook (variables, imports, roles).
2. For each play it resolves the host pattern against the inventory and applies `serial`/`forks` batching.
3. It gathers facts (unless `gather_facts: false`) by running the `setup` module on each host.
4. For each task, in order, it builds a module payload, copies it to a temp dir on the target, runs it with the discovered Python, reads the JSON result, and removes the temp files.
5. Handlers notified by changed tasks run at the end of the play (or at `meta: flush_handlers`).
6. The `PLAY RECAP` summarises ok/changed/failed/unreachable per host.

</details>

### Q7. What is the difference between `ansible`, `ansible-core` and the `ansible` community package? 🟡

<details>
<summary><b>Answer</b></summary>

`ansible-core` is the engine: the CLI tools, the playbook language, and a small set of built-in modules (`ansible.builtin.*`). The `ansible` package on PyPI is `ansible-core` plus a curated bundle of community collections (amazon.aws, community.general, kubernetes.core and so on).

In production we install `ansible-core` and pin only the collections we actually use in `requirements.yml`. This keeps the install small and upgrades predictable.

</details>

### Q8. In what order does Ansible look for `ansible.cfg`? 🟡

<details>
<summary><b>Answer</b></summary>

The first one found wins; files are not merged:

1. `ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` in the current directory
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

Individual settings can also be overridden by `ANSIBLE_*` environment variables or CLI flags. Check what is in effect with `ansible --version` (shows the config file) and `ansible-config dump --only-changed`.

A world-writable current directory is ignored for security, with a warning.

</details>

### Q9. How do you see which configuration settings are active and where they came from? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-config view                    # print the config file in use
ansible-config dump --only-changed     # settings that differ from defaults, with origin
ansible-config list | less             # every setting, env var and ini key
```

`dump` shows the source of each value (`env`, `ini`, `default`), which is the fastest way to find out why a setting is not taking effect.

</details>

### Q10. What are the common YAML pitfalls when writing playbooks? 🟡

<details>
<summary><b>Answer</b></summary>

- Indentation uses spaces only; tabs are a syntax error.
- A value starting with `{{` must be quoted: `msg: "{{ var }}"`, otherwise YAML reads it as a dict.
- Unquoted `yes/no/on/off` become booleans, and `0644` can become an octal-looking int. Quote file modes: `mode: "0644"`.
- A colon followed by a space inside a string needs quotes.
- Versions like `3.10` parse as the float 3.1 unless quoted.

`ansible-playbook --syntax-check` and `ansible-lint` catch most of these before a run.

</details>

### Q11. What does "declarative" mean in the context of Ansible, and is Ansible fully declarative? 🟡

<details>
<summary><b>Answer</b></summary>

Declarative means you state the end result ("package nginx is present, service is started") and the module works out what to do. Well-written modules check current state first and only change what differs.

Ansible is not purely declarative: playbooks execute top to bottom, and `shell`/`command` tasks are imperative. So it is a hybrid, declarative at the module level and procedural at the playbook level. This is different from Terraform or Puppet, which build a dependency graph.

</details>

### Q12. What are connection plugins? Name the common ones. 🟡

<details>
<summary><b>Answer</b></summary>

A connection plugin defines how Ansible talks to a host. It is selected with `ansible_connection`.

- `ssh` (default for Linux), `paramiko_ssh` (pure Python fallback)
- `local` — run on the control node itself
- `winrm`, `psrp` — Windows
- `docker`, `podman`, `kubectl` — execute inside containers
- `network_cli`, `netconf`, `httpapi` — network devices
- `community.aws.aws_ssm` — over AWS Systems Manager, no SSH needed

```ini
[web]
app1 ansible_connection=community.aws.aws_ssm
```

</details>

### Q13. How does SSH multiplexing (ControlPersist) speed up Ansible? 🔴

<details>
<summary><b>Answer</b></summary>

Every task normally needs an SSH connection. With `ControlMaster`/`ControlPersist`, the first connection to a host opens a master socket that stays alive in the background, and later tasks reuse it, skipping the TCP handshake, key exchange and authentication.

```ini
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path_dir = ~/.ansible/cp
```

These are already the defaults for the `ssh` plugin. Watch out for long socket paths: the Unix socket path limit (about 104 chars) causes `unix_listener: path too long` errors on deep home dirs or long hostnames, so keep `control_path_dir` short.

</details>

### Q14. How does Ansible actually execute a module on a remote host (AnsiballZ)? 🔴

<details>
<summary><b>Answer</b></summary>

For a normal task, Ansible builds a self-contained zip payload (AnsiballZ) that bundles the module code, its `module_utils` dependencies and the task arguments. It then:

1. Creates a temp directory under `~/.ansible/tmp` on the target.
2. Transfers the payload (SFTP/SCP, or piped over stdin with pipelining).
3. Runs it with the target's Python and parses the JSON it prints to stdout.
4. Deletes the temp dir.

Setting `ANSIBLE_KEEP_REMOTE_FILES=1` keeps the payload so you can `python AnsiballZ_xxx.py explode` it and debug a module locally. This is also why modules must print valid JSON and why stray output from shell profiles can break tasks.

</details>

### Q15. What is the `raw` module and when do you use it? 🟡

<details>
<summary><b>Answer</b></summary>

`raw` sends a command straight over SSH with no module payload, so it does not need Python on the target. It is not idempotent and has no `changed` detection.

Use it for bootstrapping (installing Python) and for devices that cannot run modules, such as some routers.

```yaml
- name: Bootstrap Python
  hosts: new_servers
  gather_facts: false
  become: true
  tasks:
    - ansible.builtin.raw: test -e /usr/bin/python3 || (apt-get update && apt-get install -y python3)
      changed_when: false
```

`gather_facts: false` matters here, since fact gathering itself needs Python.

</details>

### Q16. How do you find documentation for a module without leaving the terminal? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-doc ansible.builtin.copy         # full docs and examples
ansible-doc -s ansible.builtin.copy      # short snippet of parameters
ansible-doc -l | grep -i ec2             # list modules matching a name
ansible-doc -t lookup -l                 # list plugins of another type
```

The docs match the version you have installed, which is more reliable than a web page for a different release.

</details>

### Q17. List the main command-line tools that ship with Ansible and what each is for. 🟡

<details>
<summary><b>Answer</b></summary>

- `ansible` — run a single module ad hoc against hosts
- `ansible-playbook` — run playbooks
- `ansible-inventory` — display or export the parsed inventory
- `ansible-config` — view and dump configuration
- `ansible-doc` — module and plugin documentation
- `ansible-galaxy` — install roles and collections, scaffold roles
- `ansible-vault` — encrypt/decrypt secrets
- `ansible-console` — interactive REPL against an inventory
- `ansible-pull` — pull model, run a playbook from a Git repo locally
- `ansible-test` — test collections

</details>

### Q18. How do Ansible and Python versions interact between the controller and the targets? 🟡

<details>
<summary><b>Answer</b></summary>

The controller and the targets have separate Python requirements. The controller needs a recent Python (3.11+ for current ansible-core). The targets need a Python version that the installed ansible-core still supports for modules; each core release drops the oldest target versions.

So a very old target (for example CentOS 7 with Python 2.7 or 3.6) pins you to an older ansible-core, or forces you to install a newer Python on the target and set `ansible_python_interpreter`. Always check the support matrix when upgrading core.

</details>

### Q19. Can Ansible manage Windows hosts? How? 🟡

<details>
<summary><b>Answer</b></summary>

Yes, from a Linux control node. Windows hosts are reached over WinRM (or OpenSSH) and tasks run PowerShell-based modules from the `ansible.windows` and `community.windows` collections.

```ini
[win]
srv01 ansible_host=10.0.0.20

[win:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore   # lab only
ansible_user=Administrator
```

Modules are named `win_*` style (`ansible.windows.win_feature`, `win_service`, `win_copy`). Use `ansible.windows.win_ping` rather than `ping` to test.

</details>

### Q20. How does Ansible manage network devices that cannot run Python? 🔴

<details>
<summary><b>Answer</b></summary>

Network modules run on the control node, not the device. The control node opens a session with a network connection plugin (`ansible.netcommon.network_cli` over SSH, `netconf`, or `httpapi`), sends CLI commands or API calls, and parses the output.

```ini
[routers:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
ansible_user=admin
```

This is why `ansible_network_os` is mandatory and why network playbooks use platform collections like `cisco.ios`, `arista.eos` or `junipernetworks.junos`. Facts and idempotency depend on the resource modules (`ios_vlans`, `ios_interfaces`) rather than raw CLI.

</details>

### Q21. In what order are the parts of a play executed? 🟡

<details>
<summary><b>Answer</b></summary>

Within one play, in this order regardless of how they appear in the file:

1. `pre_tasks`
2. Handlers notified by pre_tasks
3. `roles` (and `tasks` that import roles)
4. `tasks`
5. Handlers notified by roles/tasks
6. `post_tasks`
7. Handlers notified by post_tasks

Role dependencies in `meta/main.yml` run before the role that depends on them. Facts are gathered before everything, as an implicit first step.

</details>

### Q22. What is the difference between parse-time (static) and run-time (dynamic) processing? 🔴

<details>
<summary><b>Answer</b></summary>

`import_*` statements (`import_tasks`, `import_role`, `import_playbook`) are static: Ansible expands them when the playbook is parsed, so tags, `when` and `--list-tasks` see every task. `include_*` statements (`include_tasks`, `include_role`) are dynamic: they are evaluated at run time, when the task is reached.

Consequences: variables defined at run time (loops, `set_fact`, registered results) can drive `include_tasks` but not `import_tasks` file names; `--list-tasks` and tags on an `include_tasks` do not apply to the included tasks unless you use `apply`; and a `when` on an `import_tasks` is copied to every imported task rather than evaluated once.

```yaml
- ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"   # dynamic, OK
- ansible.builtin.import_tasks: common.yml                              # static
```

</details>

### Q23. Can a playbook have multiple plays? Why would you do that? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. A playbook is a list of plays, and each play maps a host pattern to tasks. Multiple plays let you orchestrate across tiers in one run: configure the database, then app servers, then the load balancer, each with its own `hosts`, `become`, variables and `serial`.

```yaml
- hosts: db
  roles: [postgres]
- hosts: app
  roles: [app]
- hosts: lb
  roles: [haproxy]
```

Plays run sequentially. Data can pass between them through `hostvars` or `set_fact` on the hosts.

</details>

### Q24. What do the statuses in the PLAY RECAP mean? 🟢

<details>
<summary><b>Answer</b></summary>

- `ok` — task ran and made no change (or a fact/read-only task)
- `changed` — task modified something
- `unreachable` — SSH/connection failed for that host
- `failed` — a task returned failure
- `skipped` — a `when` condition was false
- `rescued` — a failure was caught by a `rescue` block
- `ignored` — a failure was ignored with `ignore_errors`

A second run of a well-written playbook should show `changed=0`. That is the quickest idempotency check.

</details>

### Q25. How does Ansible choose the Python interpreter on a target, and how do you override it? 🔴

<details>
<summary><b>Answer</b></summary>

By default `interpreter_python=auto`: Ansible runs a discovery script on the target that looks for known Python paths (`/usr/bin/python3`, `/usr/libexec/platform-python` and so on), picks the best one, and caches the result in the `discovered_interpreter_python` fact.

Override per host/group/play with `ansible_python_interpreter`, or globally in `ansible.cfg`:

```ini
[defaults]
interpreter_python = /usr/bin/python3
```

Set it explicitly when the module needs libraries from a specific venv (for example `boto3` in a virtualenv) or when auto discovery picks a Python that lacks `python3-apt`/`libselinux-python`.

</details>

---

## Inventory

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. What is an inventory and why does Ansible need one? 🟢

<details>
<summary><b>Answer</b></summary>

The inventory is the list of managed hosts, organised into groups, plus the variables attached to them. It answers "which machines does this play touch?" through host patterns like `hosts: web`.

It can be a static INI/YAML file, a directory of files, or a dynamic source such as the AWS EC2 plugin. Without an inventory, Ansible only knows the implicit `localhost`.

</details>

### Q27. What are the INI and YAML inventory formats and how do they differ? 🟢

<details>
<summary><b>Answer</b></summary>

Both describe the same thing. INI is compact; YAML is better for nested groups and typed variables (lists, dicts, booleans).

```ini
[web]
web1 ansible_host=10.0.1.11
web2 ansible_host=10.0.1.12

[web:vars]
http_port=8080
```

```yaml
all:
  children:
    web:
      hosts:
        web1: { ansible_host: 10.0.1.11 }
        web2: { ansible_host: 10.0.1.12 }
      vars:
        http_port: 8080
```

In INI, values in `key=value` are parsed as strings (or literals) with limited typing, which surprises people with `[group:vars]` lists.

</details>

### Q28. How do you create groups of groups? 🟢

<details>
<summary><b>Answer</b></summary>

Use the `:children` suffix in INI, or `children:` in YAML.

```ini
[web]
web1
web2

[db]
db1

[prod:children]
web
db
```

A play with `hosts: prod` hits all three hosts. Variables set on `prod` apply to all of them, and child group variables override parent group variables.

</details>

### Q29. Explain host patterns: `:`, `:&`, `:!` and wildcards. 🟡

<details>
<summary><b>Answer</b></summary>

- `web:db` — union (either group)
- `web:&prod` — intersection (in both)
- `web:!web3` — exclusion
- `web*` — glob on host or group names
- `~web\d+` — a regular expression, prefixed with `~`
- `web[0]` / `web[1:3]` — index/slice into a group

```bash
ansible 'prod:&web:!web3' -m ping
ansible-playbook site.yml --limit 'web:&eu'
```

Quote patterns, because the shell treats `!` and `&` specially.

</details>

### Q30. Which connection variables can you set per host in inventory? 🟢

<details>
<summary><b>Answer</b></summary>

The common ones:

- `ansible_host` — real address/DNS name if different from the inventory name
- `ansible_port`, `ansible_user`, `ansible_password`
- `ansible_ssh_private_key_file`
- `ansible_connection` — `ssh`, `local`, `winrm`, `docker`…
- `ansible_become`, `ansible_become_user`, `ansible_become_method`
- `ansible_python_interpreter`
- `ansible_ssh_common_args` — e.g. a ProxyJump through a bastion

```ini
db1 ansible_host=10.0.2.5 ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/prod.pem
```

</details>

### Q31. What are `group_vars` and `host_vars` directories? 🟡

<details>
<summary><b>Answer</b></summary>

Directories placed next to the inventory file (or the playbook) that hold variables by group or host name, instead of cluttering the inventory.

```text
inventory/
  hosts.ini
  group_vars/
    all.yml
    web.yml
    db/
      main.yml
      secrets.yml   # vault-encrypted
  host_vars/
    web1.yml
```

A group can use either `web.yml` or a `web/` directory whose files are all loaded. Splitting a vaulted `secrets.yml` from plain `main.yml` keeps diffs readable.

</details>

### Q32. How do you define a range of hosts in an inventory? 🟢

<details>
<summary><b>Answer</b></summary>

Use bracket ranges, numeric or alphabetic, with an optional stride and zero padding.

```ini
[web]
web[01:10].example.com
[dbs]
db-[a:c].example.com
[odd]
node[1:9:2]
```

The first expands to `web01` through `web10`. Check with `ansible-inventory --graph`.

</details>

### Q33. What is the implicit localhost and how does it behave? 🟡

<details>
<summary><b>Answer</b></summary>

If `localhost` is not in your inventory, Ansible still lets you target it with `hosts: localhost`. It uses `ansible_connection=local`, and the Python interpreter that runs Ansible (rather than SSH). It does not belong to `all` or any group, so `hosts: all` will not include it.

```yaml
- hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - ansible.builtin.debug: msg="control node tasks"
```

It is commonly used for cloud API calls, such as creating infrastructure before the real hosts exist.

</details>

### Q34. What are the default groups `all` and `ungrouped`? 🟢

<details>
<summary><b>Answer</b></summary>

`all` contains every host in the inventory. `ungrouped` contains every host that is not a member of any other group. Every host is always in `all` and in at least one more group, so a host with no groups ends up in `ungrouped`.

Variables in `group_vars/all.yml` apply to every host and are the lowest-priority inventory group level.

</details>

### Q35. How do you inspect the inventory Ansible actually parsed? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-inventory -i inventory/ --graph          # tree of groups and hosts
ansible-inventory -i inventory/ --graph --vars   # include variables
ansible-inventory -i inventory/ --list           # full JSON
ansible-inventory -i inventory/ --host web1      # merged vars for one host
ansible-inventory -i inventory/ --list --yaml
```

It shows the merged result, including dynamic sources, so it is the first thing to run when a host is not in the group you expect.

</details>

### Q36. What is a dynamic inventory? 🟡

<details>
<summary><b>Answer</b></summary>

An inventory generated at run time by querying an external source (a cloud API, CMDB, Kubernetes, vSphere) instead of a hand-edited file. In a cloud with autoscaling, a static file is wrong within hours, while a dynamic one reflects what exists now.

Today this is done with inventory plugins such as `amazon.aws.aws_ec2`, `azure.azcollection.azure_rm`, `google.cloud.gcp_compute`, or `kubernetes.core.k8s`. Older executable scripts that print JSON still work with `--list` and `--host` but are discouraged.

</details>

### Q37. What is the difference between an inventory plugin and an inventory script? 🟡

<details>
<summary><b>Answer</b></summary>

A script is an executable that must answer `--list` and `--host <name>` with JSON. It runs as a subprocess, with no built-in caching, no config validation and no Jinja2.

A plugin is a Python class loaded by Ansible, configured by a YAML file. It supports caching, `keyed_groups`, `compose`, `groups` expressions, filters and `strict` mode. Plugins are the supported path and are enabled in `[inventory] enable_plugins`.

</details>

### Q38. Show a minimal `aws_ec2` dynamic inventory configuration. 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  instance-state-name: running
  tag:Env: prod
compose:
  ansible_host: private_ip_address
```

```bash
ansible-galaxy collection install amazon.aws
pip install boto3 botocore
ansible-inventory -i inventory/aws_ec2.yml --graph
```

Credentials come from the normal AWS chain (environment variables, shared profile, or an instance role). The file name must end in `aws_ec2.yml` or `aws_ec2.yaml`, or the plugin refuses it.

</details>

### Q39. How do `keyed_groups`, `hostnames` and `compose` shape an `aws_ec2` inventory? 🔴

<details>
<summary><b>Answer</b></summary>

- `keyed_groups` builds groups from instance attributes, for example one group per tag value.
- `hostnames` chooses the inventory name, tried in order (`tag:Name`, `private-ip-address`, `dns-name`).
- `compose` creates host variables from Jinja2 expressions over the instance data.

```yaml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
keyed_groups:
  - key: tags.Role
    prefix: role         # group role_web, role_db
  - key: placement.availability_zone
    prefix: az
hostnames:
  - tag:Name
  - private-ip-address
compose:
  ansible_host: private_ip_address
  env: tags.Env | default('unknown')
```

Now `hosts: role_web:&az_ap_south_1a` works without anyone maintaining a list.

</details>

### Q40. Why does an `aws_ec2` inventory file sometimes get ignored with "did not meet aws_ec2 requirements"? 🟡

<details>
<summary><b>Answer</b></summary>

The plugin verifies the file name. It must end with `aws_ec2.yml` or `aws_ec2.yaml` (for example `prod.aws_ec2.yml`). Other causes: the `plugin:` key is missing or mis-spelled, or the plugin is not in `enable_plugins`.

Run with `-vvv` to see each plugin's reason for skipping the file, and use `ansible-inventory --list` to confirm parsing.

</details>

### Q41. Can you use more than one inventory source at a time? 🟡

<details>
<summary><b>Answer</b></summary>

Yes. Pass `-i` several times, or point `-i` at a directory, and Ansible merges all files in it (static files, dynamic plugin configs and scripts).

```bash
ansible-playbook -i inventory/static.ini -i inventory/aws_ec2.yml site.yml
ansible-playbook -i inventory/ site.yml
```

Group variables from `group_vars/` next to each source apply to the merged hosts. Hidden files and names ending in `.orig`, `.bak`, `.retry` are ignored in a directory source.

</details>

### Q42. How do you run a playbook against only a subset of hosts? 🟢

<details>
<summary><b>Answer</b></summary>

Use `--limit` (or `-l`), which intersects with the play's `hosts`.

```bash
ansible-playbook site.yml --limit web1
ansible-playbook site.yml --limit 'web:!web3'
ansible-playbook site.yml --limit @failed_hosts.txt
```

`--limit @file` reads host names from a file, useful to retry only failed hosts, which Ansible writes to a `.retry` file when `retry_files_enabled` is set (off by default).

</details>

### Q43. What does the `add_host` module do and when is it useful? 🔴

<details>
<summary><b>Answer</b></summary>

`add_host` adds a host (and optionally groups and variables) to the in-memory inventory during a run. It is lost when the playbook ends.

```yaml
- hosts: localhost
  tasks:
    - amazon.aws.ec2_instance:
        name: temp-build
        image_id: ami-0abc1234567890def
        instance_type: t3.micro
        wait: true
      register: ec2
    - ansible.builtin.add_host:
        name: "{{ ec2.instances[0].private_ip_address }}"
        groups: new_nodes
- hosts: new_nodes
  tasks:
    - ansible.builtin.ping:
```

It bridges provisioning and configuration in one playbook. Note it runs once per play, not per host, so loop over results explicitly.

</details>

### Q44. What does `group_by` do? 🔴

<details>
<summary><b>Answer</b></summary>

`group_by` creates groups at run time from the value of a variable or fact, so later plays can target them.

```yaml
- hosts: all
  tasks:
    - ansible.builtin.group_by:
        key: "os_{{ ansible_facts['distribution'] | lower }}"
- hosts: os_ubuntu
  tasks:
    - ansible.builtin.apt: { update_cache: true }
- hosts: os_rocky
  tasks:
    - ansible.builtin.dnf: { update_cache: true }
```

It avoids per-OS `when:` clauses on every task and keeps each OS family's play simple.

</details>

### Q45. If a host belongs to several groups that define the same variable, which one wins? 🔴

<details>
<summary><b>Answer</b></summary>

Rules for inventory group variables, from lowest to highest:

1. `all`
2. Parent groups
3. Child groups (a child beats its parent)
4. Host variables (beat any group)

For groups at the same level, they are merged in alphabetical order, so the last one alphabetically wins. You can override that with `ansible_group_priority` (higher wins, default 1), which is only honoured when set in the inventory source, not in `group_vars/` files.

```ini
[prod:vars]
ansible_group_priority=10
```

Relying on alphabetical order is fragile, so avoid putting the same variable in sibling groups.

</details>

### Q46. How do you target hosts with a regex pattern? 🟡

<details>
<summary><b>Answer</b></summary>

Start the pattern with `~`. It matches against host names or group names using Python regex.

```bash
ansible '~(web|app)[0-9]+\.prod' -m ping
ansible-playbook site.yml --limit '~db-eu-.*'
```

Remember to quote the pattern so the shell does not expand it. Combine with other operators, for example `~web[0-9]+:&prod`.

</details>

### Q47. How does inventory caching work and when would you enable it? 🔴

<details>
<summary><b>Answer</b></summary>

Dynamic inventory plugins can cache the API response so each run does not re-query the cloud, which matters with thousands of instances or API throttling.

```yaml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
cache: true
cache_plugin: ansible.builtin.jsonfile
cache_connection: /tmp/aws_inventory_cache
cache_timeout: 600
```

The trade-off is stale data: a new instance will not show up until the cache expires. Force a refresh with `ansible-inventory --flush-cache` (or by deleting the cache dir). Cache TTL should be shorter than your autoscaling churn.

</details>

### Q48. What is the `constructed` inventory plugin? 🔴

<details>
<summary><b>Answer</b></summary>

`ansible.builtin.constructed` builds extra groups and variables from the hosts discovered by other inventory sources, using Jinja2 conditions and host variables. It is placed in the same inventory directory and runs after the other sources.

```yaml
# inventory/zz_constructed.yml
plugin: ansible.builtin.constructed
strict: false
groups:
  prod_web: "env == 'prod' and 'web' in role"
keyed_groups:
  - key: ansible_facts.distribution
    prefix: os
compose:
  deploy_dir: "'/opt/' ~ app_name"
```

It lets one team own the cloud inventory while another layers logical grouping on top, without editing the plugin config.

</details>

### Q49. How do you reach hosts in a private subnet through a bastion host? 🟡

<details>
<summary><b>Answer</b></summary>

Set an SSH ProxyJump (or ProxyCommand) for those hosts. The cleanest way is as a group variable.

```ini
[private:vars]
ansible_ssh_common_args='-o ProxyJump=ec2-user@bastion.example.com'
```

Or put it in `~/.ssh/config` so plain `ssh` works the same way. In AWS, alternatives that need no bastion are SSM Session Manager with the `community.aws.aws_ssm` connection, or EC2 Instance Connect Endpoint.

</details>

### Q50. What is the difference between `inventory_hostname`, `ansible_hostname` and `ansible_host`? 🟡

<details>
<summary><b>Answer</b></summary>

- `inventory_hostname` — the name of the host as written in the inventory (alias).
- `ansible_host` — the address Ansible connects to; defaults to the inventory name.
- `ansible_hostname` (or `ansible_facts['hostname']`) — the machine's actual hostname, discovered by fact gathering.

They can all differ. A host named `web1` in inventory with `ansible_host=10.0.1.11` may report `ip-10-0-1-11` as its real hostname. Use `inventory_hostname` for logic about the inventory, and facts for what the machine says about itself.

</details>

---

## Ad-hoc Commands, Modules and Idempotency

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. What is an ad-hoc command and what is its syntax? 🟢

<details>
<summary><b>Answer</b></summary>

An ad-hoc command runs a single module against hosts from the command line, with nothing saved. Syntax: `ansible <pattern> -m <module> -a "<args>" [options]`.

```bash
ansible web -m ansible.builtin.ping
ansible web -m ansible.builtin.service -a "name=nginx state=restarted" -b
ansible all -a "uptime"                      # default module is command
ansible db -m ansible.builtin.setup -a "filter=ansible_mem*"
```

Good for one-off checks and emergencies. Anything you will do twice belongs in a playbook.

</details>

### Q52. What does `ansible all -m ping` actually test? 🟢

<details>
<summary><b>Answer</b></summary>

It is not an ICMP ping. The `ping` module logs in over SSH, runs a tiny Python module on the target, and returns `pong`. A `SUCCESS` therefore proves four things: network reachability, SSH authentication, a working Python interpreter, and that a temp directory can be written.

An `UNREACHABLE!` points to network/SSH/auth, while a module failure with "interpreter not found" points to Python.

</details>

### Q53. Give practical use cases for ad-hoc commands. 🟡

<details>
<summary><b>Answer</b></summary>

- Check connectivity or uptime across the fleet.
- Restart a service everywhere during an incident.
- Check disk or memory quickly: `ansible all -m shell -a "df -h /"`.
- Push a hotfix file: `ansible web -m copy -a "src=app.conf dest=/etc/app.conf" -b`.
- Gather facts of one host to inspect them: `-m setup`.
- Install a package on a few hosts.

Because these are not recorded in version control, we reserve them for diagnostics and break-glass fixes and then codify real changes in a playbook.

</details>

### Q54. What is the difference between the `command` and `shell` modules? 🟢

<details>
<summary><b>Answer</b></summary>

`command` runs the program directly, without a shell. Pipes, redirects, `&&`, globs and variable expansion like `$HOME` do not work. `shell` passes the string to `/bin/sh -c`, so all of those work.

```yaml
- ansible.builtin.command: /usr/bin/uptime
- ansible.builtin.shell: ps aux | grep nginx | wc -l
```

Prefer `command` because it is safer (no shell-injection surface) and more predictable. Use `shell` only when you truly need shell features. Both always report `changed` unless you add `creates`, `removes` or `changed_when`.

</details>

### Q55. Compare `raw`, `command`, `shell` and `script`. 🟡

<details>
<summary><b>Answer</b></summary>

- `command` — run an executable, no shell, needs Python on target.
- `shell` — run through `/bin/sh` (or `executable:`), needs Python.
- `raw` — SSH command only, no Python or module payload.
- `script` — copy a script from the control node and run it on the target; the script does not need to exist there beforehand.

```yaml
- ansible.builtin.script: scripts/check.sh --verbose
  args:
    creates: /var/tmp/check.done
```

All four are non-idempotent by default. Reach for a real module first.

</details>

### Q56. Why prefer a dedicated module over `shell`/`command`? 🟢

<details>
<summary><b>Answer</b></summary>

Modules check state before acting, report `changed` accurately, support check and diff mode, return structured data, handle platform differences and escape arguments safely. A `shell: apt install nginx` always reports changed and fails ugly on a RHEL host, whereas `ansible.builtin.package` works on both and does nothing the second time.

When no module exists, wrap the command with `creates`/`removes`/`changed_when` so it is still safe to re-run.

</details>

### Q57. What is idempotency and why does it matter? 🟢

<details>
<summary><b>Answer</b></summary>

An operation is idempotent if running it once or ten times leaves the system in the same final state. For Ansible this means a second run reports `changed=0`.

It matters because it lets you re-run playbooks safely after a partial failure, use them as drift correction, and run them in CI as a test. A playbook that is not idempotent (for example appending a line with `shell: echo >> file`) corrupts state over repeated runs.

</details>

### Q58. How do you make a `command` or `shell` task idempotent? 🟡

<details>
<summary><b>Answer</b></summary>

Tell Ansible when the task has nothing to do, and when it actually changed something.

```yaml
- name: Initialise the database once
  ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /var/lib/app/.initialised

- name: Apply migrations
  ansible.builtin.command: /opt/app/bin/migrate
  register: mig
  changed_when: "'Applying' in mig.stdout"
  failed_when: mig.rc != 0
```

`creates`/`removes` skip the task if a path exists/does not exist. `changed_when` turns an always-changed task into an accurate report. Use `when` with a prior `stat` or read-only command for more complex guards.

</details>

### Q59. When do you use `copy`, `template`, `lineinfile` and `blockinfile`? 🟡

<details>
<summary><b>Answer</b></summary>

- `copy` — place a static file or inline `content:` as-is.
- `template` — render a Jinja2 `.j2` file with variables, then place it. Default for config files.
- `lineinfile` — ensure one line is present/absent, optionally replacing a regex match.
- `blockinfile` — manage a marked multi-line block inside a file you do not fully own.

If you own the whole file, use `template`. Editing files in place with `lineinfile` becomes hard to reason about once a file has more than a few managed lines.

</details>

### Q60. What is the difference between `copy`, `synchronize` and `fetch`? 🟡

<details>
<summary><b>Answer</b></summary>

`copy` sends files from the control node to the target (or copies on the target with `remote_src: true`), and is fine for small files. `synchronize` wraps `rsync` (from `ansible.posix`) for large trees and deltas, and requires rsync on both ends. `fetch` goes the other way: it pulls files from targets to the control node.

```yaml
- ansible.posix.synchronize:
    src: build/
    dest: /var/www/app/
    delete: true
- ansible.builtin.fetch:
    src: /var/log/app.log
    dest: logs/
```

`fetch` stores files under `dest/<hostname>/path` by default so hosts do not overwrite each other.

</details>

### Q61. What are the common `lineinfile` mistakes? 🟡

<details>
<summary><b>Answer</b></summary>

- A `regexp` that is too loose replaces the wrong line, or one that does not match makes the module append a duplicate on each run.
- No `regexp`: only the exact `line` is checked, so changing the value later appends a second line.
- Without `backup: true` or `validate:`, a bad edit to `sshd_config` or `sudoers` locks you out.
- `state: present` with `insertafter` only applies when no match is found.

```yaml
- ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
    validate: '/usr/sbin/sshd -t -f %s'
```

</details>

### Q62. How do you install packages portably and keep apt fast? 🟡

<details>
<summary><b>Answer</b></summary>

`ansible.builtin.package` picks the right backend from facts, but package names often differ by OS, so you may still need per-family variables. Use `apt`/`dnf` directly when you need their options.

```yaml
- ansible.builtin.apt:
    name: [nginx, curl]
    state: present
    update_cache: true
    cache_valid_time: 3600
- ansible.builtin.dnf:
    name: nginx
    state: latest
```

`cache_valid_time` avoids `apt update` on every task. Pass `name` as a list, because one transaction is faster than a loop with one package per task. `state: latest` is not idempotent over time, so use `present` for repeatable builds.

</details>

### Q63. How do `service` and `systemd` modules differ? 🟡

<details>
<summary><b>Answer</b></summary>

`service` is a generic wrapper that picks systemd, SysV or others. `systemd_service` (alias `systemd`) is specific to systemd and adds `daemon_reload`, `masked`, `scope: user` and `enabled` control.

```yaml
- ansible.builtin.systemd_service:
    name: myapp
    state: restarted
    enabled: true
    daemon_reload: true
```

Run `daemon_reload` after dropping a new unit file, or systemd will keep using the old definition. In containers without systemd the modules fail, which is a common surprise in Molecule tests.

</details>

### Q64. What can the `file` module do? 🟢

<details>
<summary><b>Answer</b></summary>

The `file` module manages files, directories and links through the `state` parameter: `file` (verify permissions), `directory` (create), `absent` (delete), `touch`, `link` (symlink) and `hard`.

```yaml
- ansible.builtin.file:
    path: /opt/app/releases
    state: directory
    owner: app
    group: app
    mode: "0755"
    recurse: false
```

Always quote `mode`. `state: absent` on a directory deletes it recursively, so double check the path variable is not empty.

</details>

### Q65. How do you create a user with a hashed password using the `user` module? 🟡

<details>
<summary><b>Answer</b></summary>

The `password` parameter needs a hash, never plain text. Generate it with the `password_hash` filter.

```yaml
- ansible.builtin.user:
    name: deploy
    shell: /bin/bash
    groups: [sudo]
    append: true
    password: "{{ vault_deploy_password | password_hash('sha512', 'fixedsalt') }}"
    update_password: on_create
```

Without a fixed salt a new random hash is made each run and the task reports `changed` every time. `update_password: on_create` also avoids resetting passwords users changed. For SSH access, prefer `ansible.posix.authorized_key`.

</details>

### Q66. What is the difference between `get_url` and `uri`? 🟡

<details>
<summary><b>Answer</b></summary>

`get_url` downloads a file to the target, with optional checksum verification, and skips the download if the destination already matches. `uri` makes an HTTP request and returns the response, so it is for APIs, health checks and webhooks.

```yaml
- ansible.builtin.get_url:
    url: https://example.com/app-1.4.2.tar.gz
    dest: /tmp/app.tar.gz
    checksum: sha256:3b1c...e9
- ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
  register: health
  until: health.status == 200
  retries: 10
  delay: 3
```

</details>

### Q67. What is a FQCN and why use it? 🟡

<details>
<summary><b>Answer</b></summary>

A fully qualified collection name is `namespace.collection.module`, for example `ansible.builtin.copy` or `amazon.aws.ec2_instance`. Short names like `copy` still work for built-ins, but they resolve through a search path, which is ambiguous when two collections ship a module with the same name.

FQCNs make playbooks explicit, show which collection to install, and are enforced by `ansible-lint` (rule `fqcn`). Add the collections you rely on to `requirements.yml`.

</details>

### Q68. What does a module return, and how do you use it? 🟡

<details>
<summary><b>Answer</b></summary>

Every module returns JSON with common keys: `changed`, `failed`, `msg`, `rc`, `stdout`, `stderr`, plus module-specific data. Capture it with `register` and use it in `when`, `changed_when` or later tasks.

```yaml
- ansible.builtin.command: systemctl is-active nginx
  register: nginx_state
  failed_when: false
  changed_when: false
- ansible.builtin.debug:
    msg: "nginx is {{ nginx_state.stdout }} (rc={{ nginx_state.rc }})"
```

Run with `-v` to print the returned dictionary, and check the "Return Values" section of `ansible-doc` for module-specific keys.

</details>

### Q69. How does check mode work, and which modules cannot honour it? 🔴

<details>
<summary><b>Answer</b></summary>

`--check` asks each module to predict what it would change without changing it. A module must declare `supports_check_mode`. Modules that cannot predict, notably `command`, `shell` and `raw` (unless `creates`/`removes` allow a decision), are skipped, so later tasks that depend on their output may break or give wrong predictions.

Per-task control exists:

```yaml
- ansible.builtin.command: /opt/app/bin/version
  register: ver
  check_mode: false      # always run, even under --check
  changed_when: false
```

`check_mode: true` on a task forces a dry run for just that task. Combine with `--diff` to see file content changes. Check mode is a good guard, but never treat it as a guarantee of the real result.

</details>

### Q70. How would you write a simple custom module? 🔴

<details>
<summary><b>Answer</b></summary>

A module is a Python file (any language that outputs JSON works, but Python gets helpers) placed in `library/` next to the playbook or inside a collection's `plugins/modules/`.

```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(path=dict(type='str', required=True),
                           state=dict(type='str', default='present', choices=['present', 'absent'])),
        supports_check_mode=True)
    path = module.params['path']
    changed = False   # compare current and desired state here
    if module.check_mode:
        module.exit_json(changed=changed)
    # make the change only if needed
    module.exit_json(changed=changed, path=path)

if __name__ == '__main__':
    main()
```

Document it with `DOCUMENTATION`, `EXAMPLES` and `RETURN` strings so `ansible-doc` works, and test with `ansible-test sanity`. Always check state first so the module is idempotent.

</details>

### Q71. What is the difference between a module and an action plugin? 🔴

<details>
<summary><b>Answer</b></summary>

A module runs on the target host. An action plugin runs on the control node, before (or instead of) the module, and can do work locally. Many familiar modules are really action plugin plus module pairs.

`template` renders the Jinja2 file on the control node (action plugin) and then calls the `copy` module to transfer the result. `copy` and `unarchive` read local files in the same way. `include_tasks`, `debug`, `set_fact`, `assert` and `add_host` run entirely on the controller.

This is why template functions like `lookup` and variables from your inventory work inside templates, and why `ansible_facts` of the target are available only because they were collected and sent back earlier.

</details>

### Q72. What are the most useful options for ad-hoc and playbook runs (`-b`, `-K`, `-u`, `-f`, `-e`)? 🟢

<details>
<summary><b>Answer</b></summary>

- `-b` / `--become` — escalate privileges (sudo by default)
- `-K` — prompt for the become password
- `-u user` — remote user; `-k` asks for the SSH password
- `-f N` — forks, number of hosts handled in parallel (default 5)
- `-e "k=v"` or `-e @vars.yml` — extra variables (highest precedence)
- `-i` inventory, `-l` limit, `-t` tags, `-v` verbosity

```bash
ansible web -i inventory/ -b -u ubuntu -f 20 -m apt -a "upgrade=dist"
```

</details>

### Q73. Where do lookups run, and what plugin types exist in Ansible? 🔴

<details>
<summary><b>Answer</b></summary>

Lookup plugins always execute on the control node, and `lookup('file', 'x')` reads a file from the controller, not the target. This surprises people who expect it to read from the remote host. For remote data use `slurp`, `command` or `stat`.

Plugin types: `lookup`, `filter`, `test`, `callback`, `connection`, `inventory`, `strategy`, `vars`, `cache`, `become`, `action`, `shell`. A lookup is evaluated when the template is rendered, which may be many times if a variable is used repeatedly.

```yaml
db_password: "{{ lookup('ansible.builtin.env', 'DB_PASSWORD') }}"
motd: "{{ lookup('ansible.builtin.file', 'files/motd.txt') }}"
```

Use `query()` (or `wantlist=True`) when you need a list back.

</details>

### Q74. How does the `stat` module work with `register` to control flow? 🟡

<details>
<summary><b>Answer</b></summary>

`stat` returns facts about a path without changing it. The key is `stat.exists`, plus `isdir`, `mode`, `checksum` and so on.

```yaml
- ansible.builtin.stat:
    path: /etc/app/license.key
  register: lic

- ansible.builtin.fail:
    msg: "License missing"
  when: not lic.stat.exists
```

Prefer module-native idempotency (`creates`, state parameters) when it exists, and use `stat` for decisions that modules cannot make for you. Remember `stat` follows no symlinks unless `follow: true`.

</details>

### Q75. How do `async` and `poll` work for long-running tasks? 🔴

<details>
<summary><b>Answer</b></summary>

`async: N` lets a task run in the background for up to N seconds. `poll: M` makes Ansible check every M seconds and wait for completion, which keeps the SSH connection from timing out. With `poll: 0` it fires and forgets, and you later check with `async_status`.

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

This runs the same job on many hosts in parallel without being limited by `forks`. Fire-and-forget tasks cannot report failure unless you check them, and `async` does not work with `check_mode`.

</details>

---

## Playbooks, Plays, Tasks and Handlers

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. What is a playbook? 🟢

<details>
<summary><b>Answer</b></summary>

A playbook is a YAML file containing a list of plays. Each play maps a group of hosts to an ordered list of tasks, where each task calls one module with arguments. It is the reusable, version-controlled replacement for a sequence of ad-hoc commands.

```yaml
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present
```

</details>

### Q77. What are the main keys of a play? 🟢

<details>
<summary><b>Answer</b></summary>

- `name` — description shown in output
- `hosts` — pattern of target hosts (required)
- `become`, `become_user`, `remote_user`, `connection`
- `vars`, `vars_files`, `vars_prompt`, `environment`
- `gather_facts`, `serial`, `strategy`, `any_errors_fatal`
- `pre_tasks`, `roles`, `tasks`, `post_tasks`, `handlers`

Play-level keywords act as defaults that individual tasks and blocks can override.

</details>

### Q78. Why should every task have a `name`? 🟢

<details>
<summary><b>Answer</b></summary>

The name is what you see in the run output (`TASK [Install nginx]`), in logs, in AWX job views and in `--list-tasks`. Without one you get a module name and arguments, which is hard to read, and `--start-at-task` cannot target the task. Handler names are also how `notify` finds them.

`ansible-lint` flags unnamed tasks. Names should say what the task achieves, starting with a capital letter and a verb.

</details>

### Q79. Which playbook flags help you inspect and debug before a real run? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-playbook site.yml --syntax-check          # parse only
ansible-playbook site.yml --list-hosts            # which hosts match
ansible-playbook site.yml --list-tasks            # tasks that would run
ansible-playbook site.yml --check --diff          # dry run with diffs
ansible-playbook site.yml --step                  # confirm each task
ansible-playbook site.yml --start-at-task "Install nginx"
ansible-playbook site.yml -vvv                    # connection-level debug
```

`--list-tasks` combined with `--tags` shows exactly what a tag selection will run.

</details>

### Q80. What is a handler and how is it triggered? 🟢

<details>
<summary><b>Answer</b></summary>

A handler is a task that runs only when notified, typically to restart or reload a service after its config changed. A task notifies a handler with `notify`, and the notification only fires if that task reported `changed`.

```yaml
tasks:
  - ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reload nginx
handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

No config change means no needless restart, which is the whole point.

</details>

### Q81. What are the behaviours and gotchas of handlers? 🟡

<details>
<summary><b>Answer</b></summary>

- A handler runs once per play per host, even if notified by ten tasks.
- It runs at the end of the play section (pre_tasks, tasks, post_tasks each flush), not right after the notifying task.
- Handlers run in the order they are defined, not the order they were notified.
- The notification string must match the handler `name` (or one of its `listen` topics) exactly.
- A handler does not run if the notifying task did not change anything, which includes the case where a previous run changed the file but failed before the handler.

```yaml
handlers:
  - name: Restart web stack
    ansible.builtin.service: { name: nginx, state: restarted }
    listen: "restart web services"
```

`listen` lets many handlers respond to one topic.

</details>

### Q82. How do you run handlers immediately instead of at the end of the play? 🟡

<details>
<summary><b>Answer</b></summary>

Use the `meta: flush_handlers` task. All pending notifications run at that point.

```yaml
- ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
  notify: Restart app

- ansible.builtin.meta: flush_handlers

- ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
```

Use this when a following task depends on the service already running the new config, such as a health check or a database migration.

</details>

### Q83. What happens to pending handlers when a task fails? 🔴

<details>
<summary><b>Answer</b></summary>

By default, when a task fails on a host, that host is removed from the rest of the play and its already-notified handlers do not run. The result can be a changed config file with the service still running the old config, and the next run sees no change, so the handler never fires.

Fix with `force_handlers: true` in the play, `--force-handlers` on the CLI, or `ANSIBLE_FORCE_HANDLERS`. Another safe pattern is to avoid the problem: reload with `meta: flush_handlers` right after the config task. `unreachable` hosts still do not run handlers.

</details>

### Q84. How do tags work? 🟡

<details>
<summary><b>Answer</b></summary>

Tags label tasks, blocks, roles, plays or imports so you can run or skip a subset.

```yaml
- ansible.builtin.package: { name: nginx }
  tags: [packages, web]
- ansible.builtin.template: { src: a.j2, dest: /etc/a }
  tags: config
```

```bash
ansible-playbook site.yml --tags config
ansible-playbook site.yml --skip-tags packages
ansible-playbook site.yml --list-tags
```

Tags on a block or role apply to everything inside it. Tags on `include_tasks` apply only to the include itself unless you use `apply:`, whereas `import_tasks` passes tags down.

</details>

### Q85. What are the special tags `always`, `never`, `tagged`, `untagged` and `all`? 🟡

<details>
<summary><b>Answer</b></summary>

- `always` — runs even when you select other tags (unless `--skip-tags always`). Good for facts or setup that later tasks need.
- `never` — task is skipped unless you explicitly request it by a tag it has too, such as `[never, debug]`. Good for dangerous or rare tasks.
- `tagged` / `untagged` — as arguments to `--tags`, select any tagged or untagged tasks.
- `all` — the default, run everything except `never`.

```yaml
- ansible.builtin.command: /opt/app/wipe-cache
  tags: [never, wipe_cache]
```

</details>

### Q86. How do you compose playbooks, and what is `import_playbook`? 🟡

<details>
<summary><b>Answer</b></summary>

A top-level playbook can import other playbooks, each with its own plays, so you can keep `site.yml` as an entry point that composes per-tier playbooks.

```yaml
# site.yml
- import_playbook: common.yml
- import_playbook: db.yml
- import_playbook: web.yml
```

`import_playbook` is static and can only be used at the top level (not inside a play). Conditions and tags on it apply to every play inside. There is no dynamic `include_playbook` anymore.

</details>

### Q87. What does the `meta` module do? Name useful actions. 🔴

<details>
<summary><b>Answer</b></summary>

`meta` tasks influence Ansible's own execution rather than the host.

- `flush_handlers` — run pending handlers now
- `refresh_inventory` — reload inventory (after `add_host`/cloud changes)
- `clear_facts` / `clear_host_errors` — reset cached facts or failed state
- `end_play` — stop the whole play without failure
- `end_host` — stop this host only; `end_batch` — stop the current serial batch
- `reset_connection` — drop the connection so new group membership (for example after adding a user to `docker`) takes effect
- `noop` — do nothing

```yaml
- ansible.builtin.meta: end_play
  when: skip_all | default(false)
```

`meta` tasks cannot be used with `loop`, and `when` is evaluated per host.

</details>

### Q88. What is the `environment` keyword and when do you use it? 🟢

<details>
<summary><b>Answer</b></summary>

`environment` sets environment variables for the commands a task runs on the target, at task, block or play level. It is the clean way to set proxies or cloud region for a few tasks.

```yaml
- name: Install through proxy
  ansible.builtin.pip:
    name: boto3
  environment:
    https_proxy: "http://proxy.corp:3128"
    PIP_INDEX_URL: "https://pypi.corp/simple"
```

It only affects the module process on the target; it does not change the control node's environment.

</details>

### Q89. How does `vars_prompt` work? 🟡

<details>
<summary><b>Answer</b></summary>

At the start of a play Ansible asks the operator for values and stores them as variables.

```yaml
vars_prompt:
  - name: release_version
    prompt: "Version to deploy"
    private: false
    default: "latest"
  - name: db_password
    prompt: "DB password"
    private: true
```

Prompts do not appear in non-interactive runs like AWX or CI, where you must pass `-e` extras or use a survey instead. Extra vars override prompts, so a prompt is skipped when the variable is already given.

</details>

### Q90. How do you use `debug` to inspect variables without being noisy? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
- ansible.builtin.debug:
    var: ansible_facts['default_ipv4']['address']
- ansible.builtin.debug:
    msg: "Deploying {{ app_version }} to {{ inventory_hostname }}"
    verbosity: 2        # only show with -vv
```

`var:` prints the variable's value without needing `{{ }}`. `verbosity` keeps debug output out of normal runs. Do not debug secrets; use `no_log: true` on sensitive tasks instead.

</details>

### Q91. What is the difference between `assert` and `fail`? 🟡

<details>
<summary><b>Answer</b></summary>

`fail` aborts with a message (usually together with `when`). `assert` evaluates a list of conditions and fails if any is false, so it reads as validation.

```yaml
- ansible.builtin.assert:
    that:
      - app_version is defined
      - app_version is match('^[0-9]+\.[0-9]+\.[0-9]+$')
      - ansible_facts['memtotal_mb'] >= 2048
    fail_msg: "Need a semver app_version and at least 2 GB RAM"
    success_msg: "Preflight OK"
```

Put assertions at the top of a play to fail fast before anything changes.

</details>

### Q92. What do `wait_for` and `wait_for_connection` do? 🟡

<details>
<summary><b>Answer</b></summary>

`wait_for` waits for a condition on the target or on a port: a TCP port opening or closing, a file appearing, or a regex in a file. `wait_for_connection` waits until Ansible itself can connect and run a module, which is the right check after a reboot.

```yaml
- ansible.builtin.reboot:           # built-in, handles reconnect
    reboot_timeout: 600
- ansible.builtin.wait_for:
    host: 10.0.1.11
    port: 443
    timeout: 120
  delegate_to: localhost
```

Use `delegate_to: localhost` with `wait_for` when the port should be checked from the controller's viewpoint.

</details>

### Q93. How do `until`, `retries` and `delay` work? 🟡

<details>
<summary><b>Answer</b></summary>

`until` re-runs a task until the condition is true, up to `retries` times, waiting `delay` seconds between attempts. The default is 3 retries and 5 seconds.

```yaml
- ansible.builtin.uri:
    url: "https://{{ inventory_hostname }}/ready"
    validate_certs: false
  register: r
  until: r.status == 200
  retries: 30
  delay: 10
```

If the condition is never met, the task fails. Note that the task itself must not fail on a non-200, so use a module that returns the status (like `uri` with `status_code: [200, 503]`) rather than one that errors out.

</details>

### Q94. What does `run_once` do and what are its subtleties? 🔴

<details>
<summary><b>Answer</b></summary>

`run_once: true` runs the task on the first host of the current batch only. The result is applied (registered) to all hosts in the batch, so later tasks can read it from any host.

Subtleties:
- With `serial`, "first host of the batch" means it runs once per batch, not once per play.
- It does not mean "on the controller"; combine with `delegate_to: localhost` for that.
- If the first host is skipped or unreachable, the task may be skipped.

```yaml
- ansible.builtin.command: /opt/app/bin/migrate-db
  run_once: true
```

</details>

### Q95. What does the registered variable look like when a task uses `loop`? 🔴

<details>
<summary><b>Answer</b></summary>

The registered variable has a `results` list with one entry per item, each carrying the module's output plus `item`. The top-level `changed` is true if any item changed, and `failed` true if any failed.

```yaml
- ansible.builtin.command: "id {{ item }}"
  loop: [alice, bob, ghost]
  register: ids
  failed_when: false

- ansible.builtin.debug:
    msg: "{{ ids.results | selectattr('rc', 'ne', 0) | map(attribute='item') | list }}"
```

The example prints the users for whom `id` failed. Use `loop_control: label: "{{ item.name }}"` to keep output readable with large dicts.

</details>

### Q96. What are strategies and how do `linear`, `free` and `debug` differ? 🔴

<details>
<summary><b>Answer</b></summary>

A strategy controls how tasks are scheduled across hosts.

- `linear` (default): each task runs on all hosts in the batch and Ansible waits for the slowest before starting the next task.
- `free`: each host runs through the play at its own speed without waiting for others. Good when host speeds vary a lot and tasks are independent.
- `host_pinned`: like free but keeps a host on a worker until it finishes the play.
- `debug`: drops into an interactive debugger when a task fails.

```yaml
- hosts: web
  strategy: free
```

Output order becomes interleaved with `free`, and `run_once` and cross-host dependencies get harder to reason about.

</details>

### Q97. How do `any_errors_fatal` and `max_fail_percentage` control failure scope? 🔴

<details>
<summary><b>Answer</b></summary>

By default one host failing only removes that host; others carry on. `any_errors_fatal: true` makes any failure stop the entire play for every host after the current task, useful for coordinated changes. `max_fail_percentage` aborts the play once more than that share of hosts in a batch has failed.

```yaml
- hosts: web
  serial: 5
  max_fail_percentage: 20
  tasks: [...]
```

The percentage is checked per `serial` batch, and uses "greater than", so with batch size 5 and `max_fail_percentage: 20`, two failures abort but one does not. Without `serial`, the whole inventory is the single batch.

</details>

### Q98. What does `--diff` show and how does it combine with `--check`? 🟡

<details>
<summary><b>Answer</b></summary>

`--diff` prints before/after content for modules that support it: `template`, `copy`, `lineinfile`, `blockinfile`, `file` attributes. With `--check` you get a preview of exactly what would change without writing anything.

```bash
ansible-playbook site.yml --check --diff --limit web1
```

Set `diff: false` on tasks that handle secrets, otherwise passwords from a template show in the output (and `no_log: true` also hides the diff).

</details>

### Q99. How do relative paths resolve in a playbook? 🟡

<details>
<summary><b>Answer</b></summary>

Ansible searches by context. For roles, `copy`/`template`/`script` look in the role's `files/` or `templates/` first. For plain playbooks they look relative to the playbook directory, in `files/` or `templates/`. Variables files and `include_tasks` paths are relative to the including file.

Useful magic variables: `playbook_dir`, `role_path`, `inventory_dir`, `inventory_file`. Running from another working directory is fine, but `ansible.cfg` is looked up in the current directory, so use `ANSIBLE_CONFIG` in wrappers and CI.

</details>

### Q100. What does `gather_facts: false` do and when do you use it? 🟡

<details>
<summary><b>Answer</b></summary>

It skips the implicit `setup` task at the beginning of the play, saving several seconds per host. Use it when the play does not need facts: localhost API plays, simple restarts, or bootstrap plays before Python exists.

```yaml
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.setup:
        gather_subset: [min, network]
```

A play can gather a subset later. If you rely on `ansible_facts` while facts are off, the variable is undefined and the template fails.

</details>

---

## Variables, Facts and Precedence

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. Where can you define variables in Ansible? 🟢

<details>
<summary><b>Answer</b></summary>

- Inventory (host and group variables, `group_vars/`, `host_vars/`)
- Playbook: play `vars`, `vars_files`, `vars_prompt`, task and block `vars`
- Roles: `defaults/main.yml`, `vars/main.yml`
- At run time: `register`, `set_fact`, `include_vars`
- Command line: `-e key=value`, `-e @file.yml`
- Facts collected from the host, and environment via lookups

Pick one home for each variable, usually role `defaults` for tunables and `group_vars` for environment-specific values.

</details>

### Q102. How do you reference a variable, and when must you quote it? 🟢

<details>
<summary><b>Answer</b></summary>

Use Jinja2 syntax `{{ var }}`. If a YAML value *starts* with `{{`, the whole value must be quoted, or YAML will try to parse it as a dictionary. Inside `when:`, `failed_when:` and `until:` you write bare expressions with no braces.

```yaml
vars:
  greeting: "{{ prefix }}-world"        # quoted, required
tasks:
  - ansible.builtin.debug: { msg: "Hi {{ greeting }}" }
    when: prefix is defined             # no braces here
```

</details>

### Q103. Give a summary of variable precedence. 🟡

<details>
<summary><b>Answer</b></summary>

There are 22 levels. A practical summary from lowest to highest:

1. role `defaults`
2. inventory group vars, then host vars (`group_vars/all` < other groups < `host_vars`)
3. playbook `group_vars` and `host_vars`
4. host facts and cached `set_fact`
5. play `vars`, `vars_prompt`, `vars_files`
6. role `vars`, block vars, task vars
7. `include_vars`, `set_fact`, registered variables
8. role and include parameters
9. extra vars (`-e`)

Rule of thumb: the closer to the task, and the more explicit, the higher it wins. Extra vars always win.

</details>

### Q104. What are the lowest and highest precedence sources, and why does that matter? 🟡

<details>
<summary><b>Answer</b></summary>

Lowest is role `defaults/main.yml`, designed to be overridden by anything. Highest is extra vars (`-e`), which beat everything including `set_fact` and role `vars`.

This is why role defaults hold tunable settings, and why `-e` is a reliable override for a one-off run. It is also a trap in AWX: a job template's extra variables silently override your playbook logic, and an `-e` value cannot be changed from inside the play.

</details>

### Q105. What is the difference between a role's `defaults/` and `vars/`? 🟡

<details>
<summary><b>Answer</b></summary>

`defaults/main.yml` has the lowest precedence, so inventory, play or `-e` can override it. It is the role's public interface. `vars/main.yml` has high precedence (level 15) and is hard to override from inventory, so use it for internal constants (such as OS-specific package names) that users should not change.

```yaml
# roles/nginx/defaults/main.yml
nginx_worker_processes: auto
# roles/nginx/vars/Debian.yml
nginx_pkg: nginx-full
```

If users cannot override something that they reasonably need to, you put it in the wrong place.

</details>

### Q106. What are facts and how are they gathered? 🟢

<details>
<summary><b>Answer</b></summary>

Facts are data about a managed node: OS and version, IP addresses, CPU, memory, mounts, devices, environment. They are gathered by the `setup` module, run automatically at the start of each play unless `gather_facts: false`.

```bash
ansible web1 -m ansible.builtin.setup
ansible web1 -m ansible.builtin.setup -a "filter=ansible_distribution*"
```

Used in conditions and templates: `when: ansible_facts['os_family'] == 'Debian'`.

</details>

### Q107. What is the difference between `ansible_facts['x']` and the older `ansible_x` variables? 🟡

<details>
<summary><b>Answer</b></summary>

Facts are available in two forms. `ansible_facts['distribution']` is the namespaced form and is always present. The injected form `ansible_distribution` was created for backwards compatibility and is controlled by `inject_facts_as_vars` (default true, planned to change to false in a future release).

Use `ansible_facts[...]` in new code. The injected variables also have a higher precedence than inventory vars, which can cause confusing collisions.

</details>

### Q108. What are local (custom) facts? 🟡

<details>
<summary><b>Answer</b></summary>

Files ending in `.fact` in `/etc/ansible/facts.d/` on the target are read during fact gathering. They can be INI, JSON, or an executable that prints JSON. The data appears under `ansible_local`.

```ini
# /etc/ansible/facts.d/app.fact
[general]
version=1.4.2
role=api
```

```yaml
- ansible.builtin.debug:
    msg: "App version is {{ ansible_local['app']['general']['version'] }}"
```

Handy for letting a host report its own deployed version, and for deploying with a different `facts_path`.

</details>

### Q109. What is the difference between `register`, `set_fact` and `vars`? 🟡

<details>
<summary><b>Answer</b></summary>

- `vars` defines a value at a scope (play/block/task). It is lazily evaluated, only when used.
- `register` stores a task's result in a variable for the rest of the play (per host).
- `set_fact` creates a host variable at run time from an expression; it is evaluated immediately and persists for the host until the run ends.

```yaml
- ansible.builtin.command: cat /etc/version
  register: v
- ansible.builtin.set_fact:
    app_ver: "{{ v.stdout | trim }}"
```

Use `vars` for configuration, `register` to capture output, and `set_fact` to compute a value once and reuse it.

</details>

### Q110. What does `cacheable: true` on `set_fact` do? 🔴

<details>
<summary><b>Answer</b></summary>

By default a `set_fact` value lives only in memory for the current run. With `cacheable: true` it is also written to the fact cache (if one is configured), so the next run, or another playbook, can read it as a host fact.

```yaml
- ansible.builtin.set_fact:
    last_deployed: "{{ app_version }}"
    cacheable: true
```

In the current run it still has `set_fact` precedence; when loaded from cache in a later run it has host-fact precedence. It does nothing useful without `fact_caching` set up, and entries expire with `fact_caching_timeout`.

</details>

### Q111. Which magic variables are most useful? 🟡

<details>
<summary><b>Answer</b></summary>

- `inventory_hostname`, `inventory_hostname_short`
- `hostvars` — all variables of every host in the inventory
- `groups` — dict of group name to list of hosts
- `group_names` — groups the current host is in
- `ansible_play_hosts`, `ansible_play_batch` — hosts in the play / current serial batch
- `ansible_check_mode`, `ansible_diff_mode`, `ansible_version`
- `playbook_dir`, `role_path`, `ansible_run_tags`

```yaml
- debug: msg="{{ groups['db'] | length }} database hosts"
```

</details>

### Q112. How do you read another host's variables, and when does it fail? 🔴

<details>
<summary><b>Answer</b></summary>

Use `hostvars['hostname']['var']`, commonly to build config that mentions other nodes.

```yaml
upstream_servers: >-
  {{ groups['app'] | map('extract', hostvars, ['ansible_facts', 'default_ipv4', 'address']) | list }}
```

It fails with `undefined variable` for facts when that host has not been contacted in this run: facts only exist after fact gathering (or from the fact cache). Fix by running a short fact-gathering play against those hosts first (`gather_subset: network`) or enable cache with `gathering = smart`. Inventory and group variables are available without contacting the host.

</details>

### Q113. How do you pass variables on the command line? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-playbook site.yml -e "env=prod version=1.4.2"
ansible-playbook site.yml -e '{"packages": ["git", "vim"], "debug": true}'
ansible-playbook site.yml -e @vars/prod.yml
```

Key=value form always produces strings, so `-e "debug=false"` is the string `"false"`, which can still be truthy in some contexts. Use JSON or a file for booleans, numbers and lists. Extra vars have the highest precedence.

</details>

### Q114. What is the difference between `vars_files` and `include_vars`? 🟡

<details>
<summary><b>Answer</b></summary>

`vars_files` is a play keyword, loaded at play start (static), and the files must exist. `include_vars` is a task that loads a file or directory at run time, so the path can use facts and loops, and it has a higher precedence.

```yaml
- ansible.builtin.include_vars: "{{ ansible_facts['os_family'] }}.yml"
- ansible.builtin.include_vars:
    dir: vars/app
    extensions: [yml]
```

For an OS-specific file with a default fallback, combine it with `lookup('ansible.builtin.first_found', ...)`.

</details>

### Q115. How does fact caching work and how do you configure it? 🔴

<details>
<summary><b>Answer</b></summary>

Fact caching saves gathered facts between runs so Ansible can skip the `setup` step for hosts whose facts are still fresh.

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /var/cache/ansible/facts
fact_caching_timeout = 86400
```

`gathering = smart` gathers only when there is no cached entry. Cache plugins include `jsonfile`, `redis` and `memcached`. Redis suits shared AWX/CI runners. Risks: stale facts after an OS upgrade or IP change, and sensitive data stored in the cache, so protect the directory or the Redis instance.

</details>

### Q116. How do you gather only some facts to save time? 🟡

<details>
<summary><b>Answer</b></summary>

Use `gather_subset` to include or exclude categories, or `filter` to select keys.

```yaml
- hosts: all
  gather_facts: true
  gather_subset:
    - "!hardware"
    - "!all"
    - network
```

`min` is always collected. Hardware facts (disks, DMI) are the slowest on big hosts. You can also run `ansible.builtin.setup` with `gather_subset: min` as a task and only gather more if needed.

</details>

### Q117. How do you handle "variable is undefined" errors? 🟢

<details>
<summary><b>Answer</b></summary>

The default failure is `'foo' is undefined`. Provide a fallback with `default`, test with `is defined`, or fail early with `assert`.

```yaml
timeout: "{{ app_timeout | default(30) }}"
region: "{{ cloud_region | default(omit) }}"   # omit the parameter entirely
when: feature_flag | default(false) | bool
```

`default(omit)` removes the module parameter completely when the variable is not set. `default('', true)` also replaces empty strings and `None`.

</details>

### Q118. What are the variable scopes in Ansible? 🟡

<details>
<summary><b>Answer</b></summary>

- Global: config settings, environment, and `-e` extra vars
- Play: `vars`, `vars_files`, `roles`
- Host: inventory host and group vars, facts, registered variables and `set_fact`, which stay with a host across plays in the same run
- Block and task: `vars` set on them live only inside

Registered and `set_fact` values persist for the host for the rest of the run, including later plays. Task `vars` disappear after the task.

</details>

### Q119. How do you merge two dictionaries, and what happened to `hash_behaviour`? 🔴

<details>
<summary><b>Answer</b></summary>

By default, a higher-precedence dict replaces a lower one completely instead of merging. The `hash_behaviour = merge` setting changes that globally, but it is deprecated because it makes variables unpredictable.

Merge explicitly with the `combine` filter:

```yaml
app_config: "{{ app_config_defaults | combine(app_config_overrides | default({}), recursive=true) }}"
```

`recursive=true` merges nested dicts; lists are not merged and are replaced. The same pattern, a `*_defaults` dict plus a `*_overrides` dict, is a clean way to design role variables.

</details>

### Q120. What are the naming rules for variables, and why prefix role variables? 🟢

<details>
<summary><b>Answer</b></summary>

Names must be letters, digits and underscores, start with a letter or underscore, and must not be Python keywords or reserved names like `environment`, `name` or `hosts` used as play keywords. No dashes or dots.

Role variables share one global namespace per host, so prefix them with the role name (`nginx_port`, `postgres_version`) to avoid collisions. Variables meant to be private get a double prefix `__nginx_pkg`, which is the community convention.

</details>

### Q121. What causes "recursive loop detected in template string"? 🔴

<details>
<summary><b>Answer</b></summary>

Variables are templated lazily, when used. A variable that refers to itself therefore loops forever.

```yaml
path: "{{ path }}/bin"          # error: recursive loop detected
```

Fix by using a different name for the base value, or by using `set_fact` with a different target name so the right side is evaluated once.

```yaml
base_path: /opt/app
bin_path: "{{ base_path }}/bin"
```

It also happens indirectly with a role default `foo: "{{ foo | default('x') }}"`; use a differently named variable.

</details>

### Q122. How do registered variables behave across plays and hosts? 🟡

<details>
<summary><b>Answer</b></summary>

A registered variable is stored per host and stays available for the rest of the playbook run, including later plays on that host. It is not visible to other hosts directly, but you can read it through `hostvars['web1']['result']`.

It is lost when the run ends (unless `set_fact` with `cacheable`). A skipped task still registers a result with `skipped: true`, so check `result is skipped` or `result.skipped` before using `result.stdout`.

</details>

### Q123. What is the precedence relationship between `set_fact`, `include_vars`, task vars and role vars? 🔴

<details>
<summary><b>Answer</b></summary>

From lower to higher: role `vars` < block vars < task vars < `include_vars` < `set_fact`/registered < role parameters < include parameters < extra vars.

A practical implication: `set_fact` overrides an inventory variable for the rest of the run and cannot be undone by `group_vars`. Another: a variable set with `set_fact` will also beat a task `vars` value of the same name in a later task, which has led to many "why is it not changing" bugs. Use a different name or avoid `set_fact` for values that should remain overridable.

</details>

### Q124. Which facts do you use most often? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
ansible_facts['distribution']          # Ubuntu, Rocky
ansible_facts['distribution_version']
ansible_facts['os_family']             # Debian, RedHat
ansible_facts['default_ipv4']['address']
ansible_facts['hostname'] / ['fqdn']
ansible_facts['memtotal_mb']
ansible_facts['processor_vcpus']
ansible_facts['mounts']
ansible_facts['pkg_mgr']
ansible_facts['service_mgr']
```

Use `os_family` for package-manager decisions and `distribution` plus `distribution_major_version` for version-specific branches.

</details>

### Q125. How do you override a variable for one host only? 🟢

<details>
<summary><b>Answer</b></summary>

Put it in `host_vars/<hostname>.yml` (or inline in the inventory). Host variables beat group variables of any group, so this is the clean per-host exception.

```yaml
# inventory/host_vars/web3.yml
nginx_worker_processes: 8
```

For a one-off run, pass `-e` with `--limit web3`. Keep exceptions rare, because every host-specific override makes the fleet harder to reason about.

</details>

---

## Jinja2, Conditionals and Loops

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. What role does Jinja2 play in Ansible? 🟢

<details>
<summary><b>Answer</b></summary>

Jinja2 is the templating engine used everywhere Ansible evaluates an expression: variables in YAML (`"{{ var }}"`), conditions in `when`, and whole files rendered by the `template` module. Rendering happens on the control node before anything is sent to the target.

Because of that, templates can use inventory data, other hosts' facts through `hostvars`, lookups and filters, none of which exist on the managed node.

</details>

### Q127. What are the three Jinja2 delimiters? 🟢

<details>
<summary><b>Answer</b></summary>

- `{{ ... }}` — expression, prints a value
- `{% ... %}` — statement: `if`, `for`, `set`, `macro`
- `{# ... #}` — comment, not rendered

```jinja
{# nginx upstream #}
upstream app {
{% for h in groups['app'] %}
    server {{ hostvars[h]['ansible_host'] }}:8080;
{% endfor %}
}
```

</details>

### Q128. Show a template with a loop and a conditional, and how it is deployed. 🟢

<details>
<summary><b>Answer</b></summary>

```jinja
# {{ ansible_managed }}
server {
    listen {{ nginx_port | default(80) }};
    server_name {{ server_names | join(' ') }};
{% if nginx_tls_enabled | default(false) %}
    ssl_certificate {{ nginx_cert_path }};
{% endif %}
}
```

```yaml
- ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/site.conf
    owner: root
    mode: "0644"
  notify: Reload nginx
```

Templates live in the role's `templates/` directory. For files that are complete configs, add `validate:` (for example `validate: nginx -t -c %s`) so a broken render never replaces a working file.

</details>

### Q129. Which filters do you use every day? 🟢

<details>
<summary><b>Answer</b></summary>

```yaml
"{{ x | default('fallback') }}"
"{{ names | join(',') }}"
"{{ name | upper }}  {{ name | lower }}  {{ s | trim }}"
"{{ list | length }}"
"{{ data | to_json }}"  "{{ data | to_nice_yaml }}"
"{{ path | basename }}  {{ path | dirname }}"
"{{ pw | password_hash('sha512') }}"
"{{ value | int }}  {{ flag | bool }}"
"{{ secret | b64encode }}"
```

Run `ansible-doc -t filter -l` to list them. Anything not built in is available in collections such as `ansible.utils` and `community.general`.

</details>

### Q130. How do you transform lists and dicts with filters (`map`, `select`, `selectattr`, `unique`, `flatten`)? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
vars:
  users:
    - { name: alice, admin: true }
    - { name: bob,   admin: false }
admin_names: "{{ users | selectattr('admin') | map(attribute='name') | list }}"
all_groups:  "{{ [['a','b'], ['b','c']] | flatten | unique }}"
ports:       "{{ services | map(attribute='port') | select('gt', 1024) | list }}"
```

`select`/`reject` filter by a test, `selectattr`/`rejectattr` filter dicts by attribute, and `map` projects values. Most filter results are generators, so add `| list` to get a real list.

</details>

### Q131. How does `when` work and which operators can you use? 🟢

<details>
<summary><b>Answer</b></summary>

`when` takes a raw Jinja2 expression, written without braces, evaluated per host. If it is false the task is skipped.

```yaml
- ansible.builtin.dnf: { name: httpd }
  when: ansible_facts['os_family'] == 'RedHat'
- ansible.builtin.debug: msg="big box"
  when: ansible_facts['memtotal_mb'] > 8192
- ansible.builtin.debug: msg="prod"
  when: env in ['prod', 'dr'] and not maintenance | bool
```

Operators: `==`, `!=`, `>`, `<`, `in`, `not`, `and`, `or`, and tests like `is defined`.

</details>

### Q132. How do you combine several conditions? 🟢

<details>
<summary><b>Answer</b></summary>

Write them with `and`/`or`, or give `when` a list, which means AND.

```yaml
when:
  - ansible_facts['os_family'] == 'Debian'
  - ansible_facts['distribution_major_version'] | int >= 22
  - deploy_enabled | bool
```

For OR with grouping, use parentheses: `when: (a == 1 or b == 2) and c`.

</details>

### Q133. What are Jinja2 tests, and which ones are most useful? 🟢

<details>
<summary><b>Answer</b></summary>

Tests are used with `is` and return a boolean:

- `is defined` / `is undefined`
- `is none`, `is string`, `is mapping`, `is iterable`
- `is match('regex')`, `is search('regex')`
- `is success`, `is failed`, `is changed`, `is skipped` (on registered results)
- `is file`, `is directory` (about paths on the control node)
- `is version('x.y', '>=')`

```yaml
when: result is failed
when: myvar is not defined
```

</details>

### Q134. How do you compare software versions correctly in a condition? 🟡

<details>
<summary><b>Answer</b></summary>

Use the `version` test, not string comparison. Strings compare character by character, so `'9.10' < '9.9'`.

```yaml
when: ansible_facts['distribution_version'] is version('22.04', '>=')
when: app_version is version('2.0.0', 'lt', strict=true)
```

Operators include `<`, `<=`, `==`, `>=`, `>`, `!=`. `strict=true` enforces semantic versioning and fails on malformed input rather than guessing.

</details>

### Q135. What is the difference between `loop` and the older `with_*` keywords? 🟡

<details>
<summary><b>Answer</b></summary>

`loop` is the modern, uniform keyword that takes a list. `with_items`, `with_dict`, `with_fileglob` and friends are the older forms, implemented as lookups and with subtly different flattening behaviour: `with_items` flattens one level of nested lists, `loop` does not.

```yaml
- ansible.builtin.user: { name: "{{ item }}", state: present }
  loop: [alice, bob]
- ansible.builtin.user: { name: "{{ item }}" }
  with_items: [alice, bob]     # legacy equivalent
```

Convert with `loop` plus a filter (`dict2items`, `flatten`) when migrating.

</details>

### Q136. How do you loop over a dictionary? 🟡

<details>
<summary><b>Answer</b></summary>

Convert it into a list of key/value items with `dict2items`.

```yaml
vars:
  sysctl_settings:
    vm.swappiness: 10
    net.core.somaxconn: 4096
tasks:
  - ansible.posix.sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
      sysctl_set: true
    loop: "{{ sysctl_settings | dict2items }}"
```

The reverse is `items2dict`. Prefer a list of dicts in your data model when you need to loop in several places.

</details>

### Q137. What does `loop_control` provide? 🟡

<details>
<summary><b>Answer</b></summary>

- `label` — what is printed for each item (hides noisy dicts and secrets)
- `loop_var` — rename `item`, essential in nested includes
- `index_var` — expose the loop index
- `pause` — sleep between iterations
- `extended: true` — adds `ansible_loop` (first, last, length, previtem, nextitem)

```yaml
loop: "{{ users }}"
loop_control:
  label: "{{ item.name }}"
  pause: 2
```

</details>

### Q138. How do you do a nested loop (all combinations) or loop inside an included file? 🔴

<details>
<summary><b>Answer</b></summary>

Either build the cross product with the `product` filter, or loop over `include_tasks` and rename the loop variable in the inner loop.

```yaml
- ansible.builtin.debug: msg="{{ item.0 }} on {{ item.1 }}"
  loop: "{{ ['web', 'api'] | product([80, 443]) | list }}"
```

```yaml
- ansible.builtin.include_tasks: per_user.yml
  loop: "{{ users }}"
  loop_control:
    loop_var: user          # inner file can use its own item
```

Without `loop_var`, an inner loop reuses `item` and Ansible warns about the variable being overwritten.

</details>

### Q139. How does `when` interact with `loop`? 🟡

<details>
<summary><b>Answer</b></summary>

The condition is evaluated separately for each item, so `item` is usable in it, and items for which it is false are skipped individually.

```yaml
- ansible.builtin.debug: msg="{{ item.name }}"
  loop: "{{ users }}"
  when: item.active
```

The registered result still contains an entry for each skipped item. Filtering the list in `loop:` itself (`selectattr`) is cleaner and gives shorter output.

</details>

### Q140. How are booleans and strings treated in conditions? 🟡

<details>
<summary><b>Answer</b></summary>

Values set from YAML (`true`, `false`) are real booleans, but values from `-e key=value`, shell output or INI inventories are strings. A string like `"false"` is not a false value, so a condition on it may be wrong, and recent ansible-core releases (2.19+) raise an error for non-boolean conditionals instead of guessing.

Be explicit:

```yaml
when: deploy_enabled | bool
when: result.stdout | length > 0
when: mylist | length > 0         # not: when: mylist
```

`bool` converts `yes/true/1/on` (case-insensitive) to true.

</details>

### Q141. How do you control whitespace in Jinja2 templates? 🟡

<details>
<summary><b>Answer</b></summary>

Statement tags leave blank lines and indentation behind. Control it with:

- `{%- ... -%}` — a dash trims whitespace on that side
- `trim_blocks` (default `true` in the `template` module) removes the first newline after a block tag
- `lstrip_blocks: true` strips leading spaces before a block tag

```yaml
- ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    lstrip_blocks: true
```

Check output with `--diff`, and watch for YAML or Python-style files where indentation matters.

</details>

### Q142. What is `ansible_managed` and why put it in templates? 🟡

<details>
<summary><b>Answer</b></summary>

`{{ ansible_managed }}` renders a header string, by default `Ansible managed`, that you can customise in `ansible.cfg` (`ansible_managed = ...`). Putting it at the top of every templated file tells the next admin the file is generated, and that manual edits will be overwritten.

Avoid putting timestamps or hostnames in it. They change on every run and make the template task report `changed` every time.

</details>

### Q143. How do you generate a config section from inventory groups, for example an HAProxy backend? 🟡

<details>
<summary><b>Answer</b></summary>

Loop over `groups[...]` and read host data through `hostvars`.

```jinja
backend app
    balance roundrobin
{% for h in groups['app'] | sort %}
    server {{ h }} {{ hostvars[h]['ansible_host'] | default(h) }}:8080 check
{% endfor %}
```

Add a `sort` so the file does not reorder between runs. Make sure the load balancer play runs after (or gathers) facts for the app hosts when you depend on facts instead of inventory variables.

</details>

### Q144. How do `regex_replace` and `regex_search` work? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
"{{ 'web-prod-01' | regex_replace('-[0-9]+$', '') }}"          # web-prod
"{{ 'v1.4.2' | regex_replace('^v', '') }}"                      # 1.4.2
"{{ out.stdout | regex_search('version (\\d+\\.\\d+)', '\\1') }}"  # ['1.4']
"{{ lines | map('regex_search', '^ERROR.*') | select | list }}"
```

`regex_search` returns the match, or an empty string (or `None` when no match depending on version) if nothing matches, so guard with `| default`. With a capture group argument it returns a list. In YAML double-quoted strings, escape backslashes (`\\d`), or use single quotes.

</details>

### Q145. When do you use `json_query` versus `selectattr`/`map`? 🔴

<details>
<summary><b>Answer</b></summary>

`json_query` (from `community.general`) uses JMESPath for complex queries on nested JSON, such as cloud API results. It needs the `jmespath` Python library on the controller.

```yaml
running: "{{ ec2.instances | community.general.json_query('[?state.name==`running`].instance_id') }}"
```

Native filters (`selectattr`, `map`, `selectattr('x','equalto',y)`) need no extra dependency and are easier to debug. Use `json_query` when the structure is deeply nested or needs projection of several fields at once. Backticks mark literals in JMESPath, which is why quoting often goes wrong.

</details>

### Q146. How would you do IP address calculations in a template? 🔴

<details>
<summary><b>Answer</b></summary>

Use the `ansible.utils.ipaddr` and `ipmath`-style filters, which depend on the `netaddr` Python package on the controller.

```yaml
network: "{{ '10.20.30.40/24' | ansible.utils.ipaddr('network') }}"        # 10.20.30.0
prefix:  "{{ '10.20.30.40/24' | ansible.utils.ipaddr('prefix') }}"         # 24
valid:   "{{ addr | ansible.utils.ipaddr | bool }}"
nth:     "{{ '10.20.30.0/24' | ansible.utils.ipaddr('net') | ansible.utils.ipaddr('10') }}"
```

Alternatives are `ansible.utils.cidr_merge` and `ipsubnet`. Install with `pip install netaddr` and `ansible-galaxy collection install ansible.utils`.

</details>

### Q147. How do you write an inline conditional value? 🟡

<details>
<summary><b>Answer</b></summary>

Use the `ternary` filter or a Jinja2 inline `if`.

```yaml
log_level: "{{ (env == 'prod') | ternary('warn', 'debug') }}"
mode: "{{ 'secure' if tls_enabled | bool else 'plain' }}"
replicas: "{{ 3 if env == 'prod' else 1 }}"
```

`ternary` takes an optional third value for the `None` case. For several branches, a dictionary lookup is clearer: `{{ {'prod': 'warn', 'stage': 'info'}[env] | default('debug') }}`.

</details>

### Q148. How do you stop Ansible from evaluating `{{ }}` in a string, such as for Go or Helm templates? 🔴

<details>
<summary><b>Answer</b></summary>

Wrap the text in `{% raw %}...{% endraw %}` inside templates, or mark a YAML value as unsafe with `!unsafe` so it is never templated.

```yaml
- ansible.builtin.copy:
    dest: /etc/prometheus/alert.yml
    content: |
      {% raw %}
      summary: "{{ $labels.instance }} is down"
      {% endraw %}
- ansible.builtin.debug:
    msg: !unsafe "{{ not_rendered }}"
```

`!unsafe` is also how you protect values from untrusted sources (like command output or inventory strings from an API) from being templated, a real injection risk if that data contains `{{ }}`.

</details>

### Q149. How do you loop over files? 🟡

<details>
<summary><b>Answer</b></summary>

For files on the control node use the `fileglob` lookup. For files on the target use the `find` module and loop over its results.

```yaml
- ansible.builtin.copy:
    src: "{{ item }}"
    dest: /etc/app/conf.d/
  loop: "{{ lookup('ansible.builtin.fileglob', 'files/conf.d/*.conf', wantlist=true) }}"

- ansible.builtin.find: { paths: /var/log/app, patterns: '*.log', age: 14d }
  register: old
- ansible.builtin.file: { path: "{{ item.path }}", state: absent }
  loop: "{{ old.files }}"
```

For a whole tree, `synchronize` or `copy` with a directory `src` is faster than looping.

</details>

### Q150. How do `subelements` and `zip` help with nested data? 🔴

<details>
<summary><b>Answer</b></summary>

`subelements` iterates over a list of dicts and a list inside each of them, giving `item.0` (parent) and `item.1` (child). `zip` pairs lists element by element.

```yaml
users:
  - name: alice
    keys: ["ssh-ed25519 AAAA...", "ssh-ed25519 BBBB..."]
tasks:
  - ansible.posix.authorized_key:
      user: "{{ item.0.name }}"
      key: "{{ item.1 }}"
    loop: "{{ users | subelements('keys') }}"

  - ansible.builtin.debug: msg="{{ item.0 }} -> {{ item.1 }}"
    loop: "{{ names | zip(ips) | list }}"
```

`zip` stops at the shortest list; use `zip_longest` to pad.

</details>

---

## Roles, Collections and Galaxy

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. What is a role and why use one? 🟢

<details>
<summary><b>Answer</b></summary>

A role is a standard directory layout that packages tasks, handlers, templates, files and variables for one purpose, such as `nginx` or `postgres`. Playbooks then become a short list of roles applied to host groups.

Benefits: reuse across projects, a single place to test and version the logic, a documented variable interface, and smaller playbooks that read like an architecture diagram.

</details>

### Q152. What is the standard role directory structure? 🟢

<details>
<summary><b>Answer</b></summary>

```text
roles/nginx/
  tasks/main.yml       # entry point
  handlers/main.yml
  defaults/main.yml    # overridable variables, lowest precedence
  vars/main.yml        # internal constants, high precedence
  files/               # static files for copy
  templates/           # .j2 files for template
  meta/main.yml        # dependencies, galaxy metadata
  library/ module_utils/   # role-local plugins
  molecule/            # tests
```

Only `tasks/main.yml` is required; every other directory is optional. Ansible loads `main.yml` (or `.yaml`, `.json`) from each of them.

</details>

### Q153. How do you scaffold a new role? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-galaxy role init roles/nginx
```

This creates the directory skeleton with `README.md`, `meta/main.yml` and empty `main.yml` files. Inside a collection use `ansible-galaxy role init --init-path roles nginx`. Delete empty directories you do not use so the role stays readable. For a new collection use `ansible-galaxy collection init myorg.platform`.

</details>

### Q154. What are the ways to use a role in a play? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
- hosts: web
  roles:                      # static, runs after pre_tasks
    - common
    - { role: nginx, nginx_port: 8080 }
  tasks:
    - ansible.builtin.import_role: { name: app }    # static, in task order
    - ansible.builtin.include_role:                 # dynamic, can loop and use when
        name: monitoring
      when: monitoring_enabled | bool
```

`roles:` is the classic form. `import_role` and `include_role` let you place a role in the middle of a task list and run it conditionally. Dynamic `include_role` is evaluated at run time, so tags and `--list-tasks` do not see inside it without `apply`.

</details>

### Q155. How do role dependencies work? 🟡

<details>
<summary><b>Answer</b></summary>

List them in `meta/main.yml`. Ansible runs dependencies before the role itself, in order.

```yaml
# roles/app/meta/main.yml
dependencies:
  - role: common
  - role: java
    java_version: 17
```

Dependencies can pass parameters and are installed by `ansible-galaxy` when listed there with a source. Keep dependency chains shallow: hidden implicit roles make a play hard to reason about, so many teams prefer explicit ordering in the playbook.

</details>

### Q156. If a role is included twice in a play, does it run twice? 🔴

<details>
<summary><b>Answer</b></summary>

Not by default. A role runs only once per play, even if listed several times or pulled in as a dependency by several roles, unless the later reference has different parameters, or the role sets `allow_duplicates: true` in `meta/main.yml`.

```yaml
# roles/firewall_rule/meta/main.yml
allow_duplicates: true
```

```yaml
roles:
  - { role: firewall_rule, port: 80 }
  - { role: firewall_rule, port: 443 }    # runs because the params differ
```

This is a frequent cause of "my role did not run the second time" questions. Using `include_role` in a loop always runs.

</details>

### Q157. How do you pass parameters to a role? 🟡

<details>
<summary><b>Answer</b></summary>

Either set variables the role reads from `defaults/`, or pass them inline. Inline parameters have a very high precedence (level 20), above `set_fact` and below only extra vars.

```yaml
roles:
  - role: postgres
    vars:
      postgres_version: 16
      postgres_max_connections: 300
```

Use `group_vars` for environment values and inline parameters for role calls that differ within one playbook. Declare the interface with `argument_specs` so mistakes are caught early.

</details>

### Q158. How do you install roles from Galaxy and keep them reproducible? 🟢

<details>
<summary><b>Answer</b></summary>

List them in `requirements.yml` with pinned versions and install into the project.

```yaml
roles:
  - name: geerlingguy.docker
    version: 7.4.1
  - src: https://github.com/example-org/ansible-role-hardening.git
    scm: git
    version: v2.3.0
    name: hardening
```

```bash
ansible-galaxy role install -r requirements.yml -p roles/
```

Pinning a version or Git tag stops an upstream change from breaking your next run.

</details>

### Q159. What is an Ansible collection? 🟢

<details>
<summary><b>Answer</b></summary>

A collection is a distribution format that bundles modules, plugins, roles, playbooks and documentation under a `namespace.name`, such as `amazon.aws` or `community.docker`. It is versioned and installed independently of ansible-core, so cloud modules can release faster than the engine.

You reference content by FQCN, for example `community.docker.docker_container`, and install with `ansible-galaxy collection install`.

</details>

### Q160. How do you install collections reproducibly? 🟡

<details>
<summary><b>Answer</b></summary>

```yaml
# requirements.yml
collections:
  - name: amazon.aws
    version: ">=9.0.0,<10.0.0"
  - name: community.general
    version: 10.1.0
  - name: https://github.com/example-org/platform.git
    type: git
    version: main
```

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible-galaxy collection list
```

`-p ./collections` keeps dependencies in the project. Use exact pins in CI and a lock-style process (or an execution environment) for production.

</details>

### Q161. What is the difference between a role and a collection? 🟡

<details>
<summary><b>Answer</b></summary>

A role is one reusable unit of tasks and variables. A collection is a packaging format that can contain many roles plus modules, filters, lookups, inventory plugins and docs, with a namespace and version.

If you only need to share configuration logic, a role is enough. If you ship custom modules or plugins, or want one versioned artifact for your platform team's content, use a collection with the roles inside it. Galaxy standalone roles still work, but new content is expected to be published as collections.

</details>

### Q162. How does Ansible find collections, and what is the `collections` keyword? 🟡

<details>
<summary><b>Answer</b></summary>

Collections are searched in `collections_path` (`ANSIBLE_COLLECTIONS_PATH`), which defaults to `~/.ansible/collections` and `/usr/share/ansible/collections`, plus a `collections/` directory adjacent to the playbook (`collections/ansible_collections/<ns>/<name>`).

The play keyword `collections:` adds namespaces to the lookup so short names resolve:

```yaml
- hosts: all
  collections:
    - community.docker
  tasks:
    - docker_container: { name: web, image: nginx }
```

This is discouraged. FQCNs avoid ambiguity.

</details>

### Q163. What is inside a collection's directory layout? 🔴

<details>
<summary><b>Answer</b></summary>

```text
myorg/platform/
  galaxy.yml            # namespace, name, version, dependencies, authors
  README.md
  meta/runtime.yml      # requires_ansible, action groups, plugin routing
  plugins/
    modules/  filter/  lookup/  inventory/  module_utils/  action/
  roles/
  playbooks/
  docs/
  tests/
```

It installs under `ansible_collections/myorg/platform`. `meta/runtime.yml` sets the minimum ansible-core version and can redirect or deprecate plugin names. Plugins and roles are referenced as `myorg.platform.<name>`.

</details>

### Q164. How do you build and publish your own collection? 🔴

<details>
<summary><b>Answer</b></summary>

```bash
ansible-galaxy collection init myorg.platform
cd myorg/platform && ansible-galaxy collection build
# -> myorg-platform-1.0.0.tar.gz
ansible-galaxy collection publish myorg-platform-1.0.0.tar.gz \
  --server https://galaxy.example.com --api-key "$GALAXY_TOKEN"
```

Before publishing: bump `version` in `galaxy.yml`, run `ansible-test sanity`, `ansible-lint`, add a changelog, and set `build_ignore` to exclude test and CI files. For internal sharing, configure a private Galaxy (Automation Hub or Pulp) under `[galaxy_server.*]` in `ansible.cfg`.

</details>

### Q165. What is `argument_specs` for roles? 🔴

<details>
<summary><b>Answer</b></summary>

`meta/argument_specs.yml` declares a role's accepted variables with types, defaults, choices and required flags. Ansible validates the inputs before the role runs and fails with a clear message, and the spec also feeds documentation (`ansible-doc -t role`).

```yaml
argument_specs:
  main:
    short_description: Install and configure nginx
    options:
      nginx_port:
        type: int
        default: 80
      nginx_tls:
        type: bool
        default: false
      nginx_sites:
        type: list
        elements: dict
        required: true
```

This replaces hand-written `assert` blocks at the top of `tasks/main.yml`.

</details>

### Q166. Where do handlers go in a role, and how do tasks notify them? 🟢

<details>
<summary><b>Answer</b></summary>

Handlers live in `handlers/main.yml` inside the role. Tasks in the same role notify them by name. Role handlers are visible to the entire play, so a task in another role (or playbook) can notify them as well, which is why handler names should be unique, usually prefixed with the role name.

```yaml
# roles/nginx/handlers/main.yml
- name: nginx | reload
  ansible.builtin.service: { name: nginx, state: reloaded }
```

</details>

### Q167. How do you handle OS-specific logic inside a role? 🟡

<details>
<summary><b>Answer</b></summary>

Load OS-specific variables with `include_vars` and a first-found fallback, and split OS-specific tasks into separate files.

```yaml
- ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
        - default.yml
      paths: [vars]
- ansible.builtin.include_tasks: "install-{{ ansible_facts['os_family'] | lower }}.yml"
```

This keeps `tasks/main.yml` a short readable sequence, with OS differences in data files.

</details>

### Q168. Where does Ansible look for roles? 🟢

<details>
<summary><b>Answer</b></summary>

In order: the `roles/` directory next to the playbook, the `roles_path` setting (`ANSIBLE_ROLES_PATH`; defaults include `~/.ansible/roles` and `/etc/ansible/roles`), and the directory of the playbook itself. Collection roles are referenced by FQCN: `role: myorg.platform.nginx`.

```ini
[defaults]
roles_path = ./roles:~/.ansible/roles
```

`ansible-galaxy role list` shows what is installed and where.

</details>

### Q169. How do you pin and upgrade role and collection versions safely? 🟡

<details>
<summary><b>Answer</b></summary>

Pin in `requirements.yml` (exact tag for roles, range or exact for collections), commit it, and upgrade deliberately via a pull request that runs your tests.

```bash
ansible-galaxy collection install -r requirements.yml --upgrade
ansible-galaxy collection list --format yaml
```

Read the collection changelog for breaking changes (removed modules, renamed parameters) before bumping a major version, run the playbook in check mode against staging, and keep the lock of installed versions in CI logs for traceability.

</details>

### Q170. How do tags and `when` apply to roles? 🟡

<details>
<summary><b>Answer</b></summary>

With the `roles:` keyword, `tags` and `when` are applied to every task in the role. With `include_role`, they apply only to the include task itself unless you wrap with `apply`.

```yaml
roles:
  - { role: nginx, tags: [web], when: "'web' in group_names" }

tasks:
  - ansible.builtin.include_role:
      name: nginx
      apply:
        tags: [web]
    tags: [web]
```

The second form needs the tag on both the include and in `apply`, otherwise `--tags web` will skip the include and the inner tasks never run.

</details>

### Q171. How do you list and verify installed collections? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
ansible-galaxy collection list                       # all with versions and paths
ansible-galaxy collection list amazon.aws
ansible-galaxy collection verify amazon.aws          # compare to the server's checksums
```

`list` also reveals duplicates, when the same collection is installed in two paths and the wrong one wins. `verify` detects local modifications or a corrupted install. You can see which collection a module resolves to with `ansible-doc <fqcn>`.

</details>

### Q172. What does `meta/main.yml` contain for a role? 🟢

<details>
<summary><b>Answer</b></summary>

Galaxy metadata (author, description, license, `min_ansible_version`, supported platforms, `galaxy_tags`), `dependencies`, and `allow_duplicates`.

```yaml
galaxy_info:
  author: DevOps Shack
  description: Install and configure nginx
  license: MIT
  min_ansible_version: "2.15"
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
dependencies: []
```

Only the dependency and duplicate settings affect execution. The rest feeds Galaxy search.

</details>

### Q173. What are `tasks_from`, `vars_from` and `defaults_from` in `include_role`/`import_role`? 🟡

<details>
<summary><b>Answer</b></summary>

They let you run a role's alternate entry point instead of `main.yml`. A role can contain, for example, `tasks/install.yml` and `tasks/configure.yml`.

```yaml
- ansible.builtin.include_role:
    name: postgres
    tasks_from: backup
    vars_from: prod
```

This turns a role into a small library of operations (`install`, `backup`, `restore`) invoked from different playbooks, without splitting it into several roles. `handlers_from` does the same for handlers.

</details>

### Q174. What is an Execution Environment (EE)? 🔴

<details>
<summary><b>Answer</b></summary>

An EE is a container image that packages ansible-core (via `ansible-runner`), the collections, Python libraries and system packages your automation needs. It removes "works on my laptop" drift: the same image runs in a developer's terminal and in AWX/AAP.

```yaml
# execution-environment.yml
version: 3
images:
  base_image:
    name: quay.io/ansible/community-ee-base:latest
dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt
```

```bash
ansible-builder build -t registry.example.com/ee-platform:1.0 .
```

</details>

### Q175. What is `ansible-navigator` and how does it relate to `ansible-playbook`? 🔴

<details>
<summary><b>Answer</b></summary>

`ansible-navigator` is a TUI/CLI that runs playbooks (and `doc`, `inventory`, `collections` commands) inside an execution environment, with a browsable interface for results. It is the developer-side tool that matches how AAP runs jobs.

```bash
ansible-navigator run site.yml -i inventory/ -m stdout --eei registry.example.com/ee-platform:1.0
ansible-navigator run site.yml --ee false      # run without a container
```

`-m stdout` gives plain `ansible-playbook`-style output for CI. Use it to reproduce an AAP failure locally with the same image.

</details>

---

## Vault, Privilege Escalation and Error Handling

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. What is Ansible Vault? 🟢

<details>
<summary><b>Answer</b></summary>

Vault encrypts sensitive data, such as passwords, API keys and certificates, so it can be committed to Git next to the rest of your automation. Ansible decrypts it in memory at run time when it is given the vault password.

It protects secrets at rest in the repository. It is not a secrets manager: there is no rotation, access auditing or per-user policy, which is why larger teams combine it with HashiCorp Vault, AWS Secrets Manager or AAP credentials.

</details>

### Q177. Which `ansible-vault` commands do you use day to day? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-vault create group_vars/prod/vault.yml     # new encrypted file
ansible-vault encrypt secrets.yml                  # encrypt in place
ansible-vault view group_vars/prod/vault.yml
ansible-vault edit group_vars/prod/vault.yml       # opens $EDITOR, re-encrypts
ansible-vault decrypt secrets.yml                  # back to plain text (careful)
ansible-vault rekey group_vars/prod/vault.yml      # change the password
ansible-vault encrypt_string 's3cr3t' --name 'db_password'
```

Avoid `decrypt`: you risk committing the plain text file by mistake.

</details>

### Q178. How do you supply the vault password when running a playbook? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
```

The password file must have mode 600 and stay out of Git (`.gitignore`). If the file is executable, Ansible runs it and reads the password from stdout. That is how teams fetch the password from a secrets manager or keychain without storing it on disk.

</details>

### Q179. What is `encrypt_string` and when is it better than encrypting a whole file? 🟡

<details>
<summary><b>Answer</b></summary>

It encrypts a single value that you paste inline into a normal vars file, leaving the other keys readable.

```bash
ansible-vault encrypt_string --vault-id prod@prompt 'S3cret!' --name 'db_password'
```

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.2;AES256;prod
  6338343936...
```

Inline strings keep variable names visible, which helps grep and code review. The downsides: diffs on the encrypted block are opaque, and the data is easy to leak if copy-pasted into the wrong place. Whole-file encryption of a `vault.yml` is simpler to rotate.

</details>

### Q180. What are vault IDs and why use them? 🔴

<details>
<summary><b>Answer</b></summary>

A vault ID labels a vault password, so you can use different passwords per environment and supply several at once.

```bash
ansible-vault encrypt --encrypt-vault-id prod group_vars/prod/vault.yml
ansible-playbook site.yml \
  --vault-id dev@~/.vault_dev \
  --vault-id prod@prompt
```

The label is stored in the file header (`$ANSIBLE_VAULT;1.2;AES256;prod`). Ansible tries the matching ID first and then falls back to the other supplied passwords. It is not a security boundary on its own, but it lets a team give developers only the `dev` password while CI holds `prod`.

</details>

### Q181. How is Vault data encrypted and what does the file look like? 🔴

<details>
<summary><b>Answer</b></summary>

The file is a header line plus a hex-encoded payload: `$ANSIBLE_VAULT;1.1;AES256` (or `1.2` with a vault ID label). The format uses AES-256 in CTR mode with a key derived from your password using PBKDF2 and a random salt, and the ciphertext is authenticated with an HMAC-SHA256.

Consequences: the same plain text encrypts to different output each time (random salt), so re-encrypting causes noisy Git diffs; the strength depends entirely on the password's strength; and file names and variable structure (for whole-file encryption) are hidden, while inline `encrypt_string` leaves variable names visible.

</details>

### Q182. How do you rotate a vault password? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
ansible-vault rekey --vault-password-file old.txt \
  --new-vault-password-file new.txt \
  group_vars/prod/vault.yml roles/*/vars/vault.yml
```

Rekey every encrypted file with the same old password, then distribute the new password and update CI. `rekey` does not touch inline `!vault` strings inside plain files; those must be regenerated with `encrypt_string`. Rotating the password does not make previously leaked history safe, so rotate the underlying secrets as well.

</details>

### Q183. What does `no_log: true` do and why is it important? 🟢

<details>
<summary><b>Answer</b></summary>

It hides a task's arguments and results from console output, callbacks and logs, replacing them with a censored message. Use it for any task handling passwords, tokens or private keys.

```yaml
- ansible.builtin.user:
    name: deploy
    password: "{{ vault_deploy_password | password_hash('sha512', 'salt') }}"
  no_log: true
```

The trade-off is that failures become hard to debug. Temporarily disable it in a safe environment, or set `ANSIBLE_DISPLAY_ARGS_TO_STDOUT=false` and be careful that `-vvv` can still reveal data in some cases.

</details>

### Q184. What is privilege escalation (`become`)? 🟢

<details>
<summary><b>Answer</b></summary>

Ansible connects as an unprivileged user and then runs the task as another user, usually root, with `become`. Default method is `sudo`. Others include `su`, `doas`, `pbrun`, `runas` (Windows), and `machinectl`.

```yaml
- hosts: web
  become: true                 # default become_user is root
  tasks:
    - ansible.builtin.service: { name: nginx, state: restarted }
    - ansible.builtin.command: whoami
      become: true
      become_user: app          # run as another user
```

Set it at the narrowest scope: play, block or task. A task using `become: false` can override the play default.

</details>

### Q185. What are common problems when becoming a non-root user? 🔴

<details>
<summary><b>Answer</b></summary>

When Ansible becomes an unprivileged user such as `postgres`, that user must be able to read the temporary module files the connecting user created. If not, you see:

`Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user`

Fixes, in order of preference:
- Install the `acl` package on the target, so Ansible can use `setfacl` to grant access.
- Enable pipelining (`pipelining = True`), so no temp file is needed. It requires `requiretty` disabled in sudoers.
- As a last resort `allow_world_readable_tmpfiles = true`, which has security downsides.

Also remember `become_user` plus `sudo` needs a sudoers rule allowing that user, and `become_flags` can set `-H` or `-i` for the right environment.

</details>

### Q186. How do you provide the become (sudo) password? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-playbook site.yml --ask-become-pass     # or -K
```

```ini
[web:vars]
ansible_become_password={{ vault_sudo_password }}
```

The preferred production setup avoids passwords entirely: a dedicated automation user with key-based SSH and a `NOPASSWD` sudoers entry limited to what automation needs, or the cloud's default `ec2-user`/`ubuntu` that already has passwordless sudo.

</details>

### Q187. What is the difference between `ignore_errors` and `failed_when`? 🟡

<details>
<summary><b>Answer</b></summary>

`ignore_errors: true` lets the play continue even though the task failed; the failure is still recorded and shown in the recap as `ignored`. `failed_when` redefines what counts as a failure based on the result, so you can ignore a known harmless exit code or fail on a text pattern.

```yaml
- ansible.builtin.command: grep -q ERROR /var/log/app.log
  register: g
  failed_when: g.rc not in [0, 1]        # rc 1 = not found, fine
  changed_when: false
```

Prefer `failed_when` because `ignore_errors` hides real problems such as a missing binary or a bad argument, not just the one you expected.

</details>

### Q188. What are `changed_when` and `failed_when` used for? 🟡

<details>
<summary><b>Answer</b></summary>

They override Ansible's own judgement of a task result using conditions on the registered output.

```yaml
- ansible.builtin.command: /usr/local/bin/rotate-keys
  register: rot
  changed_when: "'rotated 0' not in rot.stdout"
  failed_when:
    - rot.rc != 0
    - "'already running' not in rot.stderr"
```

`changed_when: false` marks read-only commands as not changing anything, which keeps recaps clean and keeps handlers from firing. A list under `failed_when` is an AND.

</details>

### Q189. How do `block`, `rescue` and `always` work? 🟡

<details>
<summary><b>Answer</b></summary>

`block` groups tasks. If any task in it fails, tasks in `rescue` run. `always` runs afterwards, whether or not the block failed. It works like try/except/finally.

```yaml
- block:
    - ansible.builtin.command: /opt/app/deploy.sh
  rescue:
    - ansible.builtin.command: /opt/app/rollback.sh
  always:
    - ansible.builtin.service: { name: app, state: started }
```

If `rescue` succeeds, the host is counted as `rescued`, not failed, and the play continues. A block can also carry shared `when`, `become`, `vars` and `tags`.

</details>

### Q190. What does `ignore_unreachable` do? 🟡

<details>
<summary><b>Answer</b></summary>

By default an unreachable host (SSH down) is removed from the play. `ignore_unreachable: true` on a task or block lets the play keep that host in rotation for later tasks, which is useful around a reboot or a firewall change.

```yaml
- ansible.builtin.command: shutdown -r +1
  async: 1
  poll: 0
  ignore_unreachable: true
```

It is not the same as `ignore_errors`: unreachable and failed are separate states. The built-in `reboot` module handles most reboot cases without it.

</details>

### Q191. How do you inspect what failed in a `rescue` block? 🔴

<details>
<summary><b>Answer</b></summary>

Inside `rescue`, two variables describe the failure: `ansible_failed_task` (name, action and so on) and `ansible_failed_result` (the module's return data).

```yaml
rescue:
  - ansible.builtin.debug:
      msg: "Task '{{ ansible_failed_task.name }}' failed: {{ ansible_failed_result.msg | default('no msg') }}"
  - ansible.builtin.uri:
      url: "{{ slack_webhook }}"
      method: POST
      body_format: json
      body: { text: "Deploy failed on {{ inventory_hostname }}" }
```

Notify, roll back, then decide whether to let the play carry on or end it with a final `fail` (or `meta: end_host`) so the failure is not masked.

</details>

### Q192. How does `delegate_to` work and what are its subtleties? 🔴

<details>
<summary><b>Answer</b></summary>

`delegate_to` runs a task on a different host while keeping the original host's variable context (`inventory_hostname`, host vars, registered values). Typical uses: call a load balancer API from the controller, update a DNS record, or take a node out of rotation.

```yaml
- ansible.builtin.command: "/usr/local/bin/lb-disable {{ inventory_hostname }}"
  delegate_to: lb1
```

Subtleties: connection variables (such as `ansible_user`, `ansible_host`) come from the delegate host; `become` rules still apply; `delegate_facts: true` stores gathered facts on the delegate instead of the original host; and without `run_once`, the task is executed once per original host, so a delegated task in a 50-host play runs 50 times.

</details>

### Q193. How do `block` and `become` inheritance interact? 🟡

<details>
<summary><b>Answer</b></summary>

Keywords on a block (`become`, `become_user`, `when`, `vars`, `tags`, `environment`, `ignore_errors`) apply to every task inside it, and a task's own keyword overrides the inherited one. This keeps playbooks short:

```yaml
- hosts: db
  tasks:
    - become: true
      become_user: postgres
      when: pg_role == 'primary'
      block:
        - community.postgresql.postgresql_db: { name: app }
        - community.postgresql.postgresql_user: { name: app_rw, db: app }
```

A `when` on a block is evaluated for each task in it, so values set by an earlier task can change the result partway through.

</details>

### Q194. What do `ansible-lint` and `yamllint` check? 🟢

<details>
<summary><b>Answer</b></summary>

`yamllint` checks YAML style: indentation, line length, trailing spaces, truthy values. `ansible-lint` checks Ansible best practices: FQCN use, unnamed tasks, `command` used where a module exists, missing `changed_when`, risky file permissions, deprecated syntax.

```bash
ansible-lint site.yml roles/
ansible-lint --profile production
```

Profiles go from `min` to `production` and get stricter. Run both locally via pre-commit and in CI so style problems never reach review.

</details>

### Q195. What is the `vault_` prefix convention for secrets in `group_vars`? 🟡

<details>
<summary><b>Answer</b></summary>

Keep two files per group: a plain `vars.yml` that references secrets, and an encrypted `vault.yml` that holds the values with a `vault_` prefix.

```yaml
# group_vars/prod/vars.yml
db_password: "{{ vault_db_password }}"
# group_vars/prod/vault.yml   (encrypted)
vault_db_password: S3cret!
```

You can grep for `db_password` and see where it is used without decrypting anything, and diffs on `vars.yml` stay readable. The encrypted file only changes when a secret does.

</details>

### Q196. What is `ansible-pull` and when would you use it? 🟡

<details>
<summary><b>Answer</b></summary>

`ansible-pull` inverts the model: each node clones a Git repository and runs a playbook against itself (`localhost`), usually on a cron or systemd timer.

```bash
ansible-pull -U https://git.example.com/infra/config.git -C main \
  -i localhost, local.yml --only-if-changed
```

Good for autoscaled instances that bootstrap themselves (user-data), for nodes behind NAT with no inbound SSH, and for very large fleets where one controller would be a bottleneck. Costs: Ansible and credentials on every node, and no central view of results unless you add reporting.

</details>

### Q197. What are the exit codes of `ansible-playbook`? 🟡

<details>
<summary><b>Answer</b></summary>

- `0` — success
- `1` — error (generic)
- `2` — one or more hosts failed
- `3` — one or more hosts were unreachable
- `4` — parser error
- `5` — bad or incomplete options
- `99` — interrupted by the user
- `250` — unexpected error

CI jobs rely on these. Note that `ignore_errors` and rescued failures still return 0, and a run where every host is unreachable returns 3, not 2.

</details>

### Q198. What is `ansible-console`? 🟢

<details>
<summary><b>Answer</b></summary>

An interactive REPL where you run modules against an inventory group without retyping the pattern or options each time.

```bash
ansible-console -i inventory/ web --become
web (3)[f:5]# ping
web (3)[f:5]# service name=nginx state=restarted
web (3)[f:5]# forks 20
```

Handy when working through an incident on a group of hosts. It has tab completion and keeps the forks and become settings for the session.

</details>

### Q199. What are callback plugins and how do you use them? 🔴

<details>
<summary><b>Answer</b></summary>

Callback plugins receive events during a run (task start, result, play recap) and either change the output (stdout callbacks) or do something extra (notification callbacks). Common ones: `ansible.posix.profile_tasks` (per-task timings), `ansible.posix.timer`, `ansible.posix.json`, and custom Slack or logging callbacks.

```ini
[defaults]
stdout_callback = ansible.builtin.default
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer

[callback_default]
result_format = yaml
```

Exactly one stdout callback is active. `profile_tasks` is the first thing we enable when a playbook is slow, since it ranks the slowest tasks.

</details>

### Q200. How can the vault password be provided by a script, and why would you do it? 🟡

<details>
<summary><b>Answer</b></summary>

If the file passed to `--vault-password-file` is executable, Ansible runs it and uses its stdout as the password. The script can fetch the secret from AWS Secrets Manager, HashiCorp Vault or the OS keychain, so it never sits in a file on disk.

```bash
#!/usr/bin/env bash
# vault-pass.sh  (chmod +x)
aws secretsmanager get-secret-value --secret-id ansible/vault \
  --query SecretString --output text
```

```bash
ansible-playbook site.yml --vault-password-file ./vault-pass.sh
```

Scripts for `--vault-id` follow the same rule and are given the vault ID as `--vault-id` argument when the file name ends in `-client`.

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
