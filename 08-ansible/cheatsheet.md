# Ansible & Configuration Management: Cheatsheet

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

> Dense command and config reference grouped by task. Targets ansible-core 2.18+.

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

## Table of contents

- [Install and version](#install-and-version)
- [Ad-hoc commands](#ad-hoc-commands)
- [ansible-playbook flags](#ansible-playbook-flags)
- [Inventory](#inventory)
- [Variables and precedence](#variables-and-precedence)
- [Facts and magic variables](#facts-and-magic-variables)
- [Playbook skeleton](#playbook-skeleton)
- [Common modules](#common-modules)
- [Conditionals, loops, registers](#conditionals-loops-registers)
- [Handlers and flow control](#handlers-and-flow-control)
- [Error handling](#error-handling)
- [Jinja2](#jinja2)
- [Roles and collections](#roles-and-collections)
- [Vault](#vault)
- [Rolling deploys](#rolling-deploys)
- [Dynamic inventory (AWS)](#dynamic-inventory-aws)
- [ansible.cfg essentials](#ansiblecfg-essentials)
- [Performance](#performance)
- [Testing and linting](#testing-and-linting)
- [Debugging](#debugging)
- [Top 15 interview one-liners](#top-15-interview-one-liners)

---

## Install and version

```bash
python3 -m venv ~/.venvs/ansible && source ~/.venvs/ansible/bin/activate
pip install ansible-core                  # engine only; add collections as needed
pip install ansible                       # engine + curated community package
ansible --version                         # shows config file, modules path, python
ansible-config dump --only-changed        # settings that differ from defaults
ansible-doc -l | grep -i ufw              # find a module
ansible-doc ansible.builtin.template      # module docs, offline
ansible-galaxy collection list
```

## Ad-hoc commands

| Task | Command |
|------|---------|
| Connectivity test | `ansible all -m ansible.builtin.ping` |
| Run a command | `ansible web -m ansible.builtin.command -a "uptime"` |
| Shell features (pipes) | `ansible web -m ansible.builtin.shell -a "ps aux \| wc -l"` |
| Install a package | `ansible web -b -m ansible.builtin.apt -a "name=htop state=present update_cache=yes"` |
| Restart a service | `ansible web -b -m ansible.builtin.service -a "name=nginx state=restarted"` |
| Copy a file | `ansible web -b -m ansible.builtin.copy -a "src=./a.conf dest=/etc/a.conf mode=0644"` |
| Fetch a file | `ansible web -m ansible.builtin.fetch -a "src=/etc/os-release dest=./out/"` |
| Gather facts | `ansible web -m ansible.builtin.setup -a "filter=ansible_distribution*"` |
| Run as another user | `ansible web -b --become-user=postgres -m ansible.builtin.command -a "psql -c 'select 1'"` |
| Parallelism | `ansible all -f 50 -m ansible.builtin.ping` |
| One-line output | `ansible all -o -m ansible.builtin.ping` |
| Async long task | `ansible web -B 600 -P 5 -m ansible.builtin.command -a "/opt/long.sh"` |

## ansible-playbook flags

| Flag | Meaning |
|------|---------|
| `-i FILE` | Inventory file, directory or plugin config |
| `-l, --limit PATTERN` | Restrict hosts (`web-1`, `web:&prod`, `@retry.txt`) |
| `-t / --skip-tags` | Run or skip tags |
| `-e "k=v"`, `-e @vars.yml` | Extra vars (highest precedence) |
| `-C, --check` | Dry run |
| `-D, --diff` | Show file diffs |
| `-b, -K` | Become; ask for the become password |
| `-u USER`, `--private-key KEY` | Remote user and SSH key |
| `-f N` | Forks |
| `-v ... -vvvv` | Verbosity; `-vvv` shows SSH details, `-vvvv` adds connection debugging |
| `--syntax-check` | Parse only |
| `--list-tasks`, `--list-hosts`, `--list-tags` | Preview |
| `--start-at-task "name"` | Resume from a task |
| `--step` | Confirm each task |
| `--vault-id`, `--ask-vault-pass`, `--vault-password-file` | Vault access |
| `--flush-cache` | Clear fact cache |

```bash
ansible-playbook site.yml -i inventories/prod --limit web --check --diff
ansible-playbook deploy.yml -e target_version=v1.4.2 -e rolling_serial=25%
```

## Inventory

```ini
# hosts.ini
[web]
web-1 ansible_host=10.0.1.11
web-2 ansible_host=10.0.1.12
web[3:5].example.com

[db]
db-1 ansible_host=10.0.2.11 ansible_user=admin

[prod:children]
web
db

[prod:vars]
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# hosts.yml
all:
  children:
    web:
      hosts:
        web-1: { ansible_host: 10.0.1.11 }
        web-2: { ansible_host: 10.0.1.12 }
    db:
      hosts:
        db-1: { ansible_host: 10.0.2.11 }
```

| Connection variable | Purpose |
|---------------------|---------|
| `ansible_host` | Real address |
| `ansible_user`, `ansible_port` | SSH user and port |
| `ansible_ssh_private_key_file` | Key path |
| `ansible_ssh_common_args` | Extra ssh args (for example `-o ProxyJump=bastion`) |
| `ansible_become`, `ansible_become_method` | Escalation |
| `ansible_connection` | `ssh`, `local`, `winrm`, `docker`, `community.aws.aws_ssm` |
| `ansible_python_interpreter` | Python path on target |

```bash
ansible-inventory -i hosts.ini --graph
ansible-inventory -i hosts.ini --list | jq .
ansible-inventory -i hosts.ini --host web-1 --vars
```

Patterns: `all`, `web`, `web:db` (union), `web:&prod` (intersection), `web:!web-2` (exclusion), `~web-[0-9]+` (regex), `web[0]` (first).

## Variables and precedence

Weakest to strongest (condensed from the 22-level list):

1. Role `defaults/main.yml`
2. Inventory file or script group vars, then `group_vars/all`, then `group_vars/<group>`
3. Inventory `host_vars/<host>`
4. Play `vars`, `vars_files`, `vars_prompt`
5. Role `vars/main.yml` and block vars
6. Task `vars`, `set_fact`, `register`
7. `include_vars`
8. Extra vars `-e` (always wins)

```yaml
vars_files: [vars/common.yml]
vars:
  http_port: 80
tasks:
  - ansible.builtin.set_fact:
      full_url: "http://{{ inventory_hostname }}:{{ http_port }}"
```

Use `group_vars/<group>/` directories with several files (`vars.yml`, `vault.yml`).

## Facts and magic variables

| Variable | Meaning |
|----------|---------|
| `ansible_facts['distribution']`, `['os_family']`, `['distribution_major_version']` | OS identity |
| `ansible_facts['default_ipv4']['address']` | Primary IP |
| `ansible_facts['memtotal_mb']`, `['processor_vcpus']` | Resources |
| `inventory_hostname`, `inventory_hostname_short` | Inventory name |
| `groups['web']` | Hosts in a group |
| `group_names` | Groups of the current host |
| `hostvars['web-1']['var']` | Another host's vars |
| `play_hosts`, `ansible_play_hosts_all` | Hosts in the play |
| `ansible_check_mode`, `ansible_diff_mode` | Run mode flags |
| `playbook_dir`, `role_path` | Paths |
| `ansible_failed_task`, `ansible_failed_result` | Available in `rescue` |

## Playbook skeleton

```yaml
---
- name: Configure web tier
  hosts: web
  become: true
  gather_facts: true
  serial: 2
  max_fail_percentage: 20
  vars_files: [vars/main.yml]
  pre_tasks:
    - name: Update apt cache
      ansible.builtin.apt: { update_cache: true, cache_valid_time: 3600 }
  roles:
    - role: common
    - role: nginx
      vars: { nginx_port: 8080 }
      tags: nginx
  tasks:
    - name: Ensure nginx is running
      ansible.builtin.service: { name: nginx, state: started, enabled: true }
  post_tasks:
    - name: Smoke test
      ansible.builtin.uri: { url: "http://localhost/", status_code: 200 }
  handlers:
    - name: Reload nginx
      ansible.builtin.service: { name: nginx, state: reloaded }
```

Order of execution: `pre_tasks`, handlers triggered by them, `roles`, `tasks`, handlers, `post_tasks`, handlers.

## Common modules

| Area | Module | Key parameters |
|------|--------|----------------|
| Packages | `ansible.builtin.apt` | `name`, `state`, `update_cache`, `cache_valid_time`, `autoremove` |
| | `ansible.builtin.dnf`, `ansible.builtin.package` | `name`, `state` |
| | `ansible.builtin.pip` | `name`, `requirements`, `virtualenv`, `virtualenv_command` |
| Files | `ansible.builtin.copy` | `src`/`content`, `dest`, `owner`, `group`, `mode`, `validate`, `backup` |
| | `ansible.builtin.template` | `src`, `dest`, `mode`, `validate` |
| | `ansible.builtin.file` | `path`, `state` (directory, touch, absent, link), `mode`, `recurse` |
| | `ansible.builtin.lineinfile` | `path`, `regexp`, `line`, `state`, `validate` |
| | `ansible.builtin.blockinfile` | `path`, `block`, `marker` |
| | `ansible.builtin.replace` | `path`, `regexp`, `replace` |
| | `ansible.builtin.get_url`, `ansible.builtin.unarchive` | `url`, `checksum`, `dest`, `remote_src` |
| | `ansible.posix.synchronize` | `src`, `dest`, `delete`, `rsync_opts` |
| Services | `ansible.builtin.service`, `ansible.builtin.systemd_service` | `name`, `state`, `enabled`, `daemon_reload` |
| Users | `ansible.builtin.user`, `ansible.builtin.group` | `name`, `groups`, `append`, `shell`, `state`, `remove` |
| | `ansible.posix.authorized_key` | `user`, `key`, `exclusive` |
| Commands | `ansible.builtin.command`, `ansible.builtin.shell`, `ansible.builtin.script`, `ansible.builtin.raw` | `creates`, `removes`, `chdir` |
| Source control | `ansible.builtin.git` | `repo`, `dest`, `version`, `force`, `depth` |
| Network | `ansible.builtin.uri` | `url`, `method`, `status_code`, `body_format`, `return_content` |
| | `ansible.builtin.wait_for` | `port`, `host`, `delay`, `timeout`, `state` |
| | `community.general.ufw` | `rule`, `port`, `proto`, `state`, `policy` |
| System | `ansible.posix.sysctl` | `name`, `value`, `sysctl_set`, `reload` |
| | `ansible.builtin.cron`, `ansible.posix.mount`, `ansible.builtin.reboot` | |
| Flow | `ansible.builtin.debug`, `ansible.builtin.assert`, `ansible.builtin.fail`, `ansible.builtin.pause`, `ansible.builtin.meta` | |
| Containers | `community.docker.docker_container`, `community.docker.docker_compose_v2` | |
| AWS | `amazon.aws.ec2_instance`, `amazon.aws.s3_object`, `amazon.aws.ec2_instance_info` | |

## Conditionals, loops, registers

```yaml
- name: Only on Debian family
  ansible.builtin.apt: { name: apache2 }
  when: ansible_facts['os_family'] == "Debian"

- name: Multiple conditions
  ansible.builtin.debug: { msg: "prod web" }
  when:
    - "'web' in group_names"
    - env == "prod"

- name: Loop over dicts
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups | default([]) }}"
    append: true
  loop: "{{ team_users }}"
  loop_control:
    label: "{{ item.name }}"

- name: Capture and test a result
  ansible.builtin.command: systemctl is-active nginx
  register: nginx_state
  changed_when: false
  failed_when: nginx_state.rc not in [0, 3]

- name: Retry until healthy
  ansible.builtin.uri: { url: "http://localhost/health" }
  register: r
  until: r.status == 200
  retries: 10
  delay: 3
```

Loop helpers: `loop`, `loop_control` (`label`, `pause`, `index_var`, `loop_var`), `dict2items`, `subelements`, `product`, `zip`, `selectattr`, `map`.

## Handlers and flow control

```yaml
tasks:
  - ansible.builtin.template: { src: app.conf.j2, dest: /etc/app.conf }
    notify: Restart app
  - ansible.builtin.meta: flush_handlers   # run pending handlers now

handlers:
  - name: Restart app
    ansible.builtin.service: { name: app, state: restarted }
    listen: "restart web stack"
```

| Control | Use |
|---------|-----|
| `serial: 1`, `serial: [1, "30%", "100%"]` | Batch size |
| `max_fail_percentage: 0` | Abort if more than this share fails |
| `any_errors_fatal: true` | One failure stops all hosts |
| `run_once: true` | One host runs the task |
| `delegate_to: localhost` | Run elsewhere with the host's vars |
| `throttle: 2` | Cap concurrent hosts for a task |
| `strategy: free` | Hosts advance independently |
| `force_handlers: true` | Run handlers after failures |
| `meta: end_host`, `meta: end_play`, `meta: refresh_inventory` | Flow |
| `async: 600`, `poll: 0` | Fire and forget, check with `async_status` |
| `tags`, `when`, `become` on `block` | Apply to a group of tasks |

## Error handling

```yaml
- name: Safe config change
  block:
    - ansible.builtin.template: { src: a.j2, dest: /etc/a.conf, backup: true }
    - ansible.builtin.command: /usr/sbin/a-check
  rescue:
    - ansible.builtin.debug: { msg: "Failed: {{ ansible_failed_result.msg | default('n/a') }}" }
    - ansible.builtin.fail: { msg: "rolled back" }
  always:
    - ansible.builtin.debug: { msg: "done" }
```

`ignore_errors: true` hides a failure but continues; `ignore_unreachable: true` tolerates dead hosts; `failed_when`/`changed_when` redefine outcomes; `no_log: true` hides sensitive output.

## Jinja2

| Need | Syntax |
|------|--------|
| Variable | `{{ var }}` |
| Default | `{{ port \| default(80) }}`; `default(omit)` to drop a parameter |
| Required | `{{ var \| mandatory }}` |
| Loop | `{% for h in groups['web'] \| sort %}...{% endfor %}` |
| Condition | `{% if x is defined and x %}...{% elif %}...{% endif %}` |
| Join / split | `{{ list \| join(',') }}`, `{{ str.split(',') }}` |
| Dict to items | `{{ d \| dict2items }}` |
| Combine dicts | `{{ a \| combine(b, recursive=True) }}` |
| JSON / YAML out | `{{ d \| to_nice_json }}`, `{{ d \| to_nice_yaml }}` |
| Regex | `{{ s \| regex_replace('^v', '') }}`, `{{ s \| regex_search('\\d+') }}` |
| Hash / password | `{{ 'x' \| hash('sha256') }}`, `{{ pw \| password_hash('sha512') }}` |
| Type | `{{ n \| int }}`, `{{ s \| bool }}`, `{{ x is string }}` |
| Select / map | `{{ users \| selectattr('enabled') \| map(attribute='name') \| list }}` |
| Unique / sort | `{{ l \| unique \| sort }}` |
| Whitespace | `{%- ... -%}`, or `trim_blocks`/`lstrip_blocks` on the task |
| Lookups (controller side) | `lookup('env','HOME')`, `lookup('file','a.pub')`, `lookup('ansible.builtin.password','/tmp/p length=20')`, `query('ansible.builtin.items', list)` |
| Tests | `is defined`, `is none`, `is match('x.*')`, `is version('2.0','>=')`, `is changed`, `is failed` |
| Another host's value | `{{ hostvars[h]['ansible_host'] }}` |

```jinja
{% for h in groups['web'] | sort %}
server {{ h }} {{ hostvars[h]['ansible_host'] }}:{{ app_port }} check
{% endfor %}
```

## Roles and collections

```text
roles/nginx/
├── defaults/main.yml     # overridable variables (low precedence)
├── vars/main.yml         # constants (high precedence)
├── tasks/main.yml
├── handlers/main.yml
├── templates/*.j2
├── files/
├── meta/main.yml         # dependencies, platforms, argument_specs.yml
└── molecule/default/
```

```bash
ansible-galaxy role init nginx
ansible-galaxy role install geerlingguy.docker
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.general -p ./collections
ansible-galaxy collection build && ansible-galaxy collection publish *.tar.gz
```

```yaml
# requirements.yml
collections:
  - name: amazon.aws
    version: ">=9.0.0"
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
```

Role use: `roles:` (static, before tasks), `ansible.builtin.import_role` (static, at parse time), `ansible.builtin.include_role` (dynamic, at runtime, use with loops or conditions).

## Vault

```bash
ansible-vault create vault.yml
ansible-vault edit vault.yml
ansible-vault view vault.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey vault.yml
ansible-vault encrypt_string 's3cr3t' --name 'db_password'
ansible-vault encrypt_string --vault-id prod@.vault_prod 's3cr3t' --name 'db_password'
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass
ansible-playbook site.yml --vault-id dev@prompt --vault-id prod@.vault_prod
```

```ini
[defaults]
vault_password_file = .vault_pass
```

Pattern: `group_vars/prod/vars.yml` holds `db_password: "{{ vault_db_password }}"`, `group_vars/prod/vault.yml` (encrypted) holds `vault_db_password`. A `vault_password_file` that is executable is run as a script and its stdout is used as the password, which lets you fetch it from a secrets manager.

## Rolling deploys

```yaml
- hosts: web
  become: true
  serial: [1, "25%", "100%"]
  max_fail_percentage: 0
  tasks:
    - name: Drain
      ansible.builtin.file: { path: /var/www/maintenance.flag, state: touch, mode: "0644" }
    - name: Deploy
      ansible.builtin.git: { repo: "{{ repo }}", dest: /opt/app, version: "{{ ver }}" }
      notify: Restart app
    - ansible.builtin.meta: flush_handlers
    - name: Wait for the port
      ansible.builtin.wait_for: { port: 8000, timeout: 60 }
    - name: Health check
      ansible.builtin.uri: { url: "http://127.0.0.1/health", status_code: 200 }
      register: h
      until: h.status == 200
      retries: 10
      delay: 3
    - name: Undrain
      ansible.builtin.file: { path: /var/www/maintenance.flag, state: absent }
```

Add a `block/rescue` for rollback, `delegate_to: localhost` for load balancer API calls, `run_once: true` for DB migrations, and `throttle: 1` for risky steps.

## Dynamic inventory (AWS)

```yaml
# inventories/aws_ec2.yml  (must end in aws_ec2.yml)
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters:
  tag:Project: fleet
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
hostnames:
  - tag:Name
compose:
  ansible_host: public_ip_address     # or private_ip_address
cache: true
cache_plugin: jsonfile
cache_connection: /tmp/ansible_inventory_cache
cache_timeout: 300
```

```bash
pip install boto3 botocore
ansible-inventory -i inventories/aws_ec2.yml --graph
ansible-inventory -i inventories/aws_ec2.yml --host web-1
ansible-inventory -i inventories/aws_ec2.yml --list --flush-cache   # ignore and refresh the inventory cache
```

## ansible.cfg essentials

```ini
[defaults]
inventory           = inventories/aws_ec2.yml
remote_user         = ubuntu
forks               = 20
host_key_checking   = True
vault_password_file = .vault_pass
roles_path          = roles
collections_path    = collections
gathering           = smart
fact_caching        = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
callbacks_enabled   = ansible.posix.profile_tasks, ansible.posix.timer
interpreter_python  = auto_silent

[privilege_escalation]
become = True
become_method = sudo

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

Config lookup order: `ANSIBLE_CONFIG` env var, `./ansible.cfg`, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`. Every setting has an env var such as `ANSIBLE_FORKS`.

## Performance

| Lever | How |
|-------|-----|
| Fewer SSH round trips | `pipelining = True` (needs sudoers without `requiretty`) |
| Persistent connections | `ControlPersist=60s` |
| More parallelism | `forks`, `-f`; watch controller CPU and bastion limits |
| Skip facts | `gather_facts: false`, or `gather_subset: [min]` |
| Cache facts | `fact_caching = jsonfile` or `redis`, with `gathering = smart` |
| Batch work | `name: [a, b, c]` in package modules instead of loops |
| No lock-step | `strategy: free` for independent hosts |
| Long tasks | `async` plus `poll: 0` |
| Find the slow task | `ansible.posix.profile_tasks` |
| Only what changed | `--tags`, `--limit`, `check_mode` tasks |

## Testing and linting

```bash
ansible-playbook site.yml --syntax-check
ansible-lint --profile production
yamllint .
molecule init scenario -d docker           # inside a role
molecule test                              # full lifecycle
molecule converge && molecule verify       # iterate fast
molecule login                             # shell into the instance
molecule destroy
ansible-playbook site.yml --check --diff   # drift report
```

Molecule `test` sequence: dependency, cleanup, destroy, syntax, create, prepare, converge, idempotence, side_effect, verify, cleanup, destroy.

## Debugging

| Symptom | Command or setting |
|---------|--------------------|
| See SSH and module details | `ansible-playbook x.yml -vvv` |
| Keep remote module files | `ANSIBLE_KEEP_REMOTE_FILES=1` then inspect `~/.ansible/tmp` |
| Print a variable | `ansible.builtin.debug: var=name` |
| Resolve a host's variables | `ansible-inventory --host web-1 --vars` |
| Preview what runs | `--list-tasks --list-hosts --list-tags` |
| Resume | `--start-at-task "Install nginx"` |
| Step through | `--step` |
| Strategy debugger | `debugger: on_failed` on a task or play |
| Config that is active | `ansible-config dump --only-changed` |
| Which Python/collection | `ansible --version`, `ansible-galaxy collection list` |

```yaml
- name: Show what a module returned
  ansible.builtin.debug:
    var: result
    verbosity: 1
```

## Top 15 interview one-liners

1. **Ansible is agentless.** It pushes small Python modules over SSH, runs them, returns JSON and cleans up; nothing resident on the target except Python.
2. **Idempotent means safe to re-run.** Modules compare current and desired state and report `changed` only when they acted; `command` and `shell` need `creates`, `removes` or `changed_when`.
3. **Variable precedence in one line:** role defaults are weakest, inventory beats defaults, play vars beat inventory, and `-e` extra vars beat everything.
4. **Handlers run once, at the end of the play, only if notified by a changed task**; use `meta: flush_handlers` to run them early and `force_handlers: true` to run them after a failure.
5. **`vars` vs `defaults` in a role:** `defaults/` is meant to be overridden, `vars/` is high precedence and meant to stay constant.
6. **Template vs copy:** `template` renders Jinja2 on the controller using vars and facts; `copy` ships a static file or `content:`; both accept `validate:` to protect live config.
7. **`serial` limits the batch size** of hosts that run the whole play at once; combine with `max_fail_percentage: 0` to stop a bad release at the first failure.
8. **`delegate_to` runs a task on another host** while keeping the original host's variables; `run_once` executes it for one host only.
9. **Vault encrypts files or strings with AES256**; keep the password out of Git, use vault IDs per environment, and `no_log: true` for secret-handling tasks.
10. **Dynamic inventory:** plugins such as `amazon.aws.aws_ec2` query the cloud API and build groups from tags with `keyed_groups`, so the inventory never drifts from reality.
11. **`--check --diff` shows what would change** without changing it; modules that cannot predict (like `command`) are skipped, so check mode is a strong signal but not a guarantee.
12. **Ansible vs Terraform:** Terraform provisions and tracks infrastructure state; Ansible configures what runs on it and has no state file; they work well together.
13. **Push vs pull:** Ansible is push by default (`ansible-playbook`); `ansible-pull` flips it for autoscaling and firewalled nodes; Puppet and Chef are agent-pull by design.
14. **Speed levers:** pipelining, `ControlPersist`, more `forks`, `gather_facts: false`, fact caching, list arguments to package modules, `strategy: free`.
15. **Prove a role works with Molecule:** converge in a container, run converge again and assert zero changes (idempotence), then verify the real state with assertions.

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

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
