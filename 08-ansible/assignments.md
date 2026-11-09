# Ansible & Configuration Management: Assignments

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

> 25 graded exercises: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Each has an objective, tasks, acceptance criteria, collapsible hints and a bonus.

[⬅ Back to Ansible](./README.md) · [🏠 Home](../README.md)

## How to use these

You need `ansible-core` 2.18+ and two or three target machines. Cheap options: local VMs, Multipass or Vagrant boxes, Docker containers with SSH, or `t3.micro` EC2 instances. Keep every assignment in its own Git folder and commit when the acceptance criteria pass. Always use fully qualified module names (`ansible.builtin.copy`, not `copy`).

## Table of contents

- [🟢 Beginner](#-beginner): Assignments 1 to 10
- [🟡 Intermediate](#-intermediate): Assignments 11 to 20
- [🔴 Advanced](#-advanced): Assignments 21 to 25

---

# 🟢 Beginner

## Assignment 1. Install Ansible and reach a host 🟢

**Objective:** Get a working control node and prove connectivity.

**Tasks**
1. Create a Python virtualenv and install `ansible-core`.
2. Prepare one Linux target with SSH key login.
3. Create an inventory file `hosts.ini` with that target.
4. Run `ansible all -m ansible.builtin.ping`.
5. Run `ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'`.

**Expected outcome / acceptance criteria**
- `ping` returns `pong` and `changed: false`.
- You can name the distribution and version from the facts output.
- `ansible --version` shows a config file path or `None`, and you can explain why.

<details>
<summary>Hints</summary>

Set `remote_user` and `private_key_file` in `ansible.cfg` or in the inventory (`ansible_user`, `ansible_ssh_private_key_file`). `ansible all -m ping -vvv` shows the exact SSH command. If you get `Permission denied (publickey)`, test with plain `ssh -i key user@host` first.
</details>

**Bonus:** Make `ping` work through a bastion with `ansible_ssh_common_args='-o ProxyJump=bastion'`.

---

## Assignment 2. Build an inventory with groups 🟢

**Objective:** Understand groups, children groups and inventory inspection.

**Tasks**
1. Create an INI inventory with groups `web`, `db` and a parent group `prod:children` containing both.
2. Add a host alias with `ansible_host=` pointing to a real IP.
3. Convert the same inventory to YAML format.
4. Run `ansible-inventory --graph` and `ansible-inventory --list` on both versions.
5. Target groups with `ansible web -m ping` and `ansible 'prod:!db' -m ping`.

**Expected outcome / acceptance criteria**
- The INI and YAML inventories produce identical `--list` JSON (compare with `diff`).
- Patterns `web`, `prod`, `prod:!db` and `web:&prod` each select the hosts you predicted.

<details>
<summary>Hints</summary>

In INI, `[prod:children]` lists group names. In YAML, use `children:` under the group. The pattern operators are `:` (union), `:!` (exclude) and `:&` (intersection). `--list` output is easier to diff after piping through `jq -S .`.
</details>

**Bonus:** Add a numeric range host pattern such as `web[01:03].example.com` and verify it expands.

---

## Assignment 3. Ad-hoc commands for daily tasks 🟢

**Objective:** Use modules from the command line instead of SSH-ing in.

**Tasks**
1. Check uptime on all hosts with `ansible.builtin.command`.
2. Install `htop` with `ansible.builtin.apt` (use `--become`).
3. Copy a local file to `/tmp/hello.txt` with `ansible.builtin.copy`.
4. Restart `cron` using `ansible.builtin.service`.
5. Run the same install command twice and compare output.

**Expected outcome / acceptance criteria**
- First install shows `CHANGED`, the second shows `SUCCESS` with `changed: false`.
- You can explain why `-m command -a 'apt install htop'` is a worse choice than `-m apt`.

<details>
<summary>Hints</summary>

`ansible web -b -m ansible.builtin.apt -a "name=htop state=present update_cache=yes"`. The `-b` flag enables privilege escalation. Add `-o` for one-line output.
</details>

**Bonus:** Use `-m ansible.builtin.fetch` to pull `/etc/os-release` from every host into per-host folders.

---

## Assignment 4. Your first playbook 🟢

**Objective:** Describe a desired state in YAML.

**Tasks**
1. Write `web.yml` targeting group `web` with `become: true`.
2. Install `nginx`, ensure it is started and enabled.
3. Deploy a custom `index.html` with `ansible.builtin.copy` and its `content:` parameter.
4. Run with `--syntax-check`, then run it for real.
5. Fetch the page with `curl` from your workstation.

**Expected outcome / acceptance criteria**
- The page shows your custom text.
- `PLAY RECAP` shows `failed=0`.
- On a second run `changed=0`.

<details>
<summary>Hints</summary>

Playbooks are a list of plays, each with `name`, `hosts`, `tasks`. Use `ansible.builtin.service` with `state: started` and `enabled: true`. If curl cannot connect, check the cloud security group or `ufw status` on the host.
</details>

**Bonus:** Add a `ansible.builtin.uri` task that asserts the page returns 200 and contains your text.

---

## Assignment 5. Idempotency, check mode and diff 🟢

**Objective:** Learn to see changes before making them.

**Tasks**
1. Take your playbook from assignment 4.
2. Run with `--check --diff` after editing the HTML content.
3. Add a task using `ansible.builtin.shell: echo hi >> /tmp/log.txt` and observe that it reports `changed` on every run.
4. Fix it with `creates:`, or rewrite it with `ansible.builtin.lineinfile`.
5. Add a `command` task that only reads data and mark it `changed_when: false`.

**Expected outcome / acceptance criteria**
- `--check --diff` shows a unified diff and changes nothing on the host.
- Two consecutive real runs end with `changed=0`.
- You can state why `shell` and `command` are not idempotent by default.

<details>
<summary>Hints</summary>

`--diff` works with `copy`, `template`, `lineinfile`, `blockinfile`. `command` and `shell` are skipped in check mode unless you set `check_mode: false` on the task, which is what you want for read-only commands.
</details>

**Bonus:** Explain the output of a task marked `check_mode: false` when the whole play runs with `--check`.

---

## Assignment 6. Variables and precedence 🟢

**Objective:** Predict which variable value wins.

**Tasks**
1. Define `http_port: 80` in role-less play `vars:`.
2. Override it in `group_vars/web.yml` as `8080` and in `host_vars/<host>.yml` as `9090`.
3. Print the value on each host with `ansible.builtin.debug`.
4. Run again with `-e http_port=7000`.
5. Move the play-level definition to `vars_files:` and observe the new winner.

**Expected outcome / acceptance criteria**
- You can write the order you observed, from weakest to strongest, and it matches the docs.
- A one-paragraph note exists explaining why you should avoid defining the same variable in many places.

<details>
<summary>Hints</summary>

`ansible-inventory --host <name> --vars` shows inventory-sourced values only. Play vars beat inventory vars, and `-e` beats all. Role `defaults/` are the weakest, which is why they are the right place for overridable settings.
</details>

**Bonus:** Use `ansible.builtin.set_fact` and show that it outranks play vars, then explain the risk.

---

## Assignment 7. Loops and conditionals 🟢

**Objective:** Avoid copy-pasted tasks.

**Tasks**
1. Create three users from a list of dicts (`name`, `shell`, `groups`) with one task and `loop`.
2. Install a package list in a single `apt` task by passing a list to `name`.
3. Install `httpd` on RedHat family and `apache2` on Debian family using `when: ansible_facts['os_family'] == ...`.
4. Print each user's name with `loop_control: label: "{{ item.name }}"`.

**Expected outcome / acceptance criteria**
- Output shows labels instead of full dictionaries.
- On a Debian host the RedHat task shows `skipping`.
- Re-running produces no changes.

<details>
<summary>Hints</summary>

Facts are available as `ansible_facts['os_family']` (and the legacy `ansible_os_family`). Use `ansible.builtin.user` with `groups:` and `append: true` so you do not strip existing groups. Passing a list to `name:` is much faster than looping the package module.
</details>

**Bonus:** Use `with_items`-free syntax only, and `loop: "{{ users | selectattr('enabled') | list }}"` to filter the list.

---

## Assignment 8. Handlers 🟢

**Objective:** Restart services only when configuration changes.

**Tasks**
1. Template or copy an nginx config file and `notify: Reload nginx`.
2. Define the handler with `ansible.builtin.service: state=reloaded`.
3. Run twice and confirm the handler fires only the first time.
4. Make two tasks notify the same handler and confirm it runs once.
5. Add `ansible.builtin.meta: flush_handlers` before a health check.

**Expected outcome / acceptance criteria**
- Handler output appears only after a real change.
- Break a later task on purpose and observe that the pending handler does not run.
- Fix it with `force_handlers: true` on the play and show the difference.

<details>
<summary>Hints</summary>

Handlers run at the end of each play (or on `flush_handlers`) and de-duplicate by name. Names are matched exactly; notifying a handler that does not exist fails the play with an error, so read it carefully. Use `listen:` to group multiple handlers under one topic.
</details>

**Bonus:** Replace the notify chain with `listen: "restart web stack"` so two handlers run together.

---

## Assignment 9. First Jinja2 template 🟢

**Objective:** Generate files from variables and facts.

**Tasks**
1. Write `motd.j2` that prints hostname, OS, IP address, memory in MB and the Ansible-managed warning.
2. Deploy to `/etc/motd` with `ansible.builtin.template`, mode `0644`.
3. Add a `{% for %}` loop over a list of admin contacts.
4. Use `default()` for an optional variable.
5. Log into the host and read the result.

**Expected outcome / acceptance criteria**
- Output contains real fact values, not Jinja syntax.
- Missing optional variable falls back to the default without error.
- A second run reports `ok`.

<details>
<summary>Hints</summary>

Facts: `ansible_facts['hostname']`, `ansible_facts['distribution']`, `ansible_facts['default_ipv4']['address']`, `ansible_facts['memtotal_mb']`. Do not put timestamps in templates, or every run becomes `changed`. Use `{{ ansible_managed }}` only if you set it deliberately; otherwise write your own header comment.
</details>

**Bonus:** Add `trim_blocks`-friendly loops so the rendered output has no blank-line gaps.

---

## Assignment 10. Facts, register, tags and limits 🟢

**Objective:** Control what runs and use task results.

**Tasks**
1. Run a command with `register: result` and print `result.stdout_lines`.
2. Tag tasks with `packages`, `config` and `service`.
3. Run only `--tags config`, then `--skip-tags service`.
4. Use `--limit` to run on one host, and `--list-tasks` and `--list-hosts` to preview.
5. Set `gather_facts: false` on a play that does not need facts and compare run time.

**Expected outcome / acceptance criteria**
- Tag selection runs the expected subset (`--list-tasks --tags config` proves it).
- Timing of the no-facts play is measurably shorter (use `time`).

<details>
<summary>Hints</summary>

`register` captures `rc`, `stdout`, `stderr`, `changed`, `failed`. Tags can be applied at task, block, role or play level. Tag `always` runs unless explicitly skipped, and `never` runs only when requested.
</details>

**Bonus:** Enable the `profile_tasks` callback (`ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks`) to see task timings.

---

# 🟡 Intermediate

## Assignment 11. Refactor a playbook into a role 🟡

**Objective:** Make automation reusable.

**Tasks**
1. Scaffold a role with `ansible-galaxy role init nginx`.
2. Move tasks, handlers, templates and variables into the proper directories.
3. Put tunables in `defaults/main.yml`, constants in `vars/main.yml`.
4. Add `meta/main.yml` with supported platforms and a dependency on a `common` role.
5. Call the role from a short `site.yml`.

**Expected outcome / acceptance criteria**
- `site.yml` has fewer than 15 lines.
- Overriding a default from `group_vars` works with no role edits.
- `ansible-lint` passes with no errors.

<details>
<summary>Hints</summary>

Role variables in `vars/` have high precedence and are hard to override; use `defaults/` for anything a user might change. Prefix every role variable with the role name (`nginx_port`) to avoid collisions. Dependencies in `meta` run before the role.
</details>

**Bonus:** Add an `argument_specs.yml` under `meta/` so wrong variable types fail fast with a clear message.

---

## Assignment 12. Secrets with Ansible Vault 🟡

**Objective:** Keep secrets encrypted in Git.

**Tasks**
1. Create `group_vars/prod/vault.yml` with `ansible-vault create`, holding `vault_db_password`.
2. Reference it from `group_vars/prod/vars.yml` as `db_password`.
3. Encrypt a single value with `ansible-vault encrypt_string --name api_token`.
4. Use `--vault-id dev@prompt` and `--vault-id prod@.vault_prod` with separate passwords.
5. Template a config that uses the secret with `no_log: true`.
6. Rotate the password with `ansible-vault rekey`.

**Expected outcome / acceptance criteria**
- `git grep` finds no plaintext secret anywhere, and `.vault_pass*` is in `.gitignore`.
- The playbook runs unattended using the password file.
- Output never prints the secret, even with `-v`.

<details>
<summary>Hints</summary>

With `no_log: true` on a task, nothing is printed even on failure, which hurts debugging; enable it only on tasks that handle secrets. `ansible-vault view` and `edit` avoid leaving plaintext on disk. Rekey needs the old password; every file encrypted with it must be rekeyed.
</details>

**Bonus:** Write a pre-commit hook that blocks commits where a file named `vault.yml` lacks the `$ANSIBLE_VAULT` header.

---

## Assignment 13. Error handling: block, rescue, always 🟡

**Objective:** Make failures predictable.

**Tasks**
1. Write a block that deploys a config, validates it, and in `rescue` restores the backup.
2. Add an `always` section that records the result in `/var/log/deploy.log`.
3. Use `failed_when` to treat a specific command output as failure even when `rc == 0`.
4. Use `ignore_errors` once and replace it with `failed_when` or `rescue`.
5. Use `any_errors_fatal: true` on a play and show its effect across hosts.

**Expected outcome / acceptance criteria**
- A deliberately broken config leads to rollback, and the play recap shows `rescued=1`.
- The `always` task runs on both success and failure.
- You can explain why `ignore_errors` hides real problems.

<details>
<summary>Hints</summary>

Within `rescue`, the variables `ansible_failed_task` and `ansible_failed_result` describe the failure. A play with rescued tasks still ends `failed=0`. `block` can carry `become`, `when`, `tags` to avoid repetition.
</details>

**Bonus:** Add `retries`, `delay` and `until` to a flaky `uri` task and show the retry output.

---

## Assignment 14. Collections, requirements and ansible.cfg 🟡

**Objective:** Make a project reproducible.

**Tasks**
1. Create `requirements.yml` pinning `community.general`, `ansible.posix` and `amazon.aws` versions.
2. Install them into `./collections` using `ansible-galaxy collection install -r requirements.yml -p collections`.
3. Set `collections_path` in `ansible.cfg`.
4. Use a module from each collection (`community.general.ufw`, `ansible.posix.sysctl`).
5. Run `ansible-config dump --only-changed`.

**Expected outcome / acceptance criteria**
- A fresh clone plus one install command is enough to run the playbooks.
- `ansible-galaxy collection list` shows exact versions matching the pins.

<details>
<summary>Hints</summary>

Use `version: "==x.y.z"` for strict pins or `">=x,<y"` for ranges. Commit `requirements.yml`, not the downloaded `collections/` folder. `ansible-doc community.general.ufw` shows module docs offline.
</details>

**Bonus:** Add a role requirement from Galaxy using the `roles:` key in the same file.

---

## Assignment 15. Dynamic inventory on AWS 🟡

**Objective:** Stop maintaining host lists by hand.

**Tasks**
1. Launch three EC2 instances tagged `Env` and `Role`.
2. Write `aws_ec2.yml` for `amazon.aws.aws_ec2` with `regions`, `filters`, `keyed_groups`, `hostnames`, `compose`.
3. Run `ansible-inventory -i aws_ec2.yml --graph`.
4. Create a group via `groups:` with a Jinja condition (for example instances of type `t3.micro`).
5. Enable inventory caching with `cache: true` and `cache_plugin: jsonfile`.

**Expected outcome / acceptance criteria**
- Groups such as `role_web` and `env_prod` appear from tags alone.
- Terminating one instance and re-running removes it with no file edits.
- Cache hits are visible in timing and in the cache directory.

<details>
<summary>Hints</summary>

The file name must end with `aws_ec2.yml`. Install `boto3` and `botocore`. Credentials come from the standard AWS chain (env vars, profile, instance role). Use `compose: ansible_host: public_ip_address` for public access or `private_ip_address` from inside the VPC. Use `ansible-inventory --host <name>` to list available hostvars such as `tags`, `instance_type`.
</details>

**Bonus:** Use SSM Session Manager as the connection (`ansible_connection: community.aws.aws_ssm`) so no inbound port 22 is needed.

---

## Assignment 16. Config files from inventory data with Jinja2 🟡

**Objective:** Generate a load balancer config from the live host list.

**Tasks**
1. Write `haproxy.cfg.j2` that loops over `groups['web']` to create `server` lines.
2. Use `hostvars[host]['ansible_host']` for addresses and weights from a host variable with a default.
3. Add a conditional section only when `haproxy_enable_stats` is true.
4. Validate with `validate: haproxy -c -f %s`.
5. Add and remove a web host and confirm the template output changes.

**Expected outcome / acceptance criteria**
- The rendered file lists exactly the hosts in the group, ordered.
- Invalid config is never installed.
- A `diff` run shows only the added or removed server line.

<details>
<summary>Hints</summary>

`{% for h in groups['web'] | sort %}`. If the target group hosts were not contacted in this play, their facts are missing; use inventory variables or run a prior play with `gather_facts`. Use `{{- -}}` or `trim_blocks` for whitespace control.
</details>

**Bonus:** Write the template so it can also render an nginx upstream block using the same data via a variable switch.

---

## Assignment 17. Safe file edits: lineinfile, blockinfile, replace 🟡

**Objective:** Edit files you do not own, without breaking them.

**Tasks**
1. Harden `/etc/ssh/sshd_config` with `lineinfile` and `validate: sshd -t -f %s`.
2. Add a managed block to `/etc/hosts` with `blockinfile` and a custom marker.
3. Use `replace` to change a value across a file with a regex and `before:`/`after:`.
4. Use `backup: true` and find the backup file.
5. Prove each task is idempotent.

**Expected outcome / acceptance criteria**
- Runs 2 and 3 show `changed=0`.
- An invalid sshd value is rejected and the live file is untouched.
- You can justify when a `template` is the better tool.

<details>
<summary>Hints</summary>

`lineinfile` needs a `regexp` that matches both the commented and the active form (`'^#?\s*PermitRootLogin'`), otherwise it appends a new line on every run because the pattern never matches the existing one. In `blockinfile`, the marker must be unique per block, so use `{mark}` placeholders with distinct text. Keep a second SSH session open when changing sshd.
</details>

**Bonus:** Convert the sshd changes into a full template with a validated drop-in under `/etc/ssh/sshd_config.d/`.

---

## Assignment 18. Users, groups and SSH keys at scale 🟡

**Objective:** Manage access as data.

**Tasks**
1. Define a `team_users` list in `group_vars` with name, groups, state and public keys.
2. Create users with `ansible.builtin.user` and install keys with `ansible.posix.authorized_key` and `exclusive: true`.
3. Add a sudoers drop-in with `ansible.builtin.copy` and `validate: visudo -cf %s`.
4. Remove a user by setting `state: absent` and `remove: true`.
5. Lock the password for all service accounts.

**Expected outcome / acceptance criteria**
- Adding a key to the YAML list gives access on the next run; removing it revokes access.
- A malformed sudoers file never lands in `/etc/sudoers.d`.

<details>
<summary>Hints</summary>

`exclusive: true` removes keys not listed, so be careful not to remove your own automation key. Sudoers drop-ins must be mode `0440`. File names in `/etc/sudoers.d` must not contain dots or end with `~`.
</details>

**Bonus:** Use a `with_fileglob` or `lookup('file', ...)` to load keys from a `keys/` folder in the repo.

---

## Assignment 19. Containers with Ansible 🟡

**Objective:** Use Ansible to bootstrap Docker and run a workload.

**Tasks**
1. Install the Docker apt repository and `docker-ce` packages using modern keyring handling (`ansible.builtin.get_url` for the key into `/etc/apt/keyrings`, plus `ansible.builtin.apt_repository` with a `signed-by` option).
2. Add your user to the `docker` group.
3. Pull and run `nginx` using `community.docker.docker_container`.
4. Publish port 8080 and mount a templated config volume.
5. Update the config and recreate the container only when it changes.

**Expected outcome / acceptance criteria**
- `docker ps` shows the container with the expected port mapping.
- Second run: `changed=0`.
- Changing the template triggers one restart through a handler.

<details>
<summary>Hints</summary>

Install the Python `docker` library on the target (or use `ansible_python_interpreter` that has it). `community.docker.docker_container` is idempotent on image and parameters; `restart: true` or a handler forces a restart. Group membership takes effect only on a new login session.
</details>

**Bonus:** Replace the container with a `community.docker.docker_compose_v2` project.

---

## Assignment 20. Write a custom filter plugin and use lookups 🟡

**Objective:** Extend Jinja2 and reach outside the inventory.

**Tasks**
1. Create `filter_plugins/netutils.py` with a filter `to_cidr_hosts` that returns the host count for a prefix.
2. Use it inside a template and a debug task.
3. Use `lookup('env', 'HOME')`, `lookup('file', ...)` and `lookup('ansible.builtin.password', '/tmp/pw length=20')`.
4. Show that lookups run on the controller, not on the managed host.

**Expected outcome / acceptance criteria**
- `{{ 24 | to_cidr_hosts }}` renders 254.
- The password lookup returns the same value on the second run (it caches in the file).

<details>
<summary>Hints</summary>

A filter plugin is a class `FilterModule` with a `filters()` method returning a dict of name to callable. Put the file next to the playbook in `filter_plugins/`, or in a role. Lookups always execute on the control node; use `slurp` or `fetch` to read remote files.
</details>

**Bonus:** Add a unit test with `pytest` for the filter.

---

# 🔴 Advanced

## Assignment 21. Zero-downtime rolling deploy with rollback 🔴

**Objective:** Release safely across a fleet.

**Tasks**
1. Use three web servers behind a load balancer (nginx or an ALB).
2. Write a play with `serial: 1` (then `[1, "30%", "100%"]`) and `max_fail_percentage: 0`.
3. Drain the host (flag file, LB API via `delegate_to`, or target group deregistration), deploy a new version from Git, restart the service.
4. Gate on an `ansible.builtin.uri` health check with `retries` and `until`.
5. Add `block/rescue` that rolls back to the previous commit on failure and then fails the play.
6. Deploy a deliberately broken release and show that two hosts stay on the old version.

**Expected outcome / acceptance criteria**
- A continuous `curl` loop against the LB never gets a 5xx during a good release.
- The bad release stops after the first batch and the failed host is rolled back and re-added.
- The recap shows `rescued=1` on one host and nothing for the others.

<details>
<summary>Hints</summary>

`serial` accepts an integer, a percentage or a list for canary-then-ramp. `max_fail_percentage: 0` aborts as soon as any host in a batch fails. For an ALB, use `delegate_to: localhost` with `amazon.aws` or `community.aws` target group modules and wait for `draining` to finish. Save the previous commit before changing anything, or you have nothing to roll back to.
</details>

**Bonus:** Add `run_once: true` smoke tests after the last batch and a `throttle: 1` database migration step.

---

## Assignment 22. Test a role with Molecule 🔴

**Objective:** Prove a role is correct, idempotent and portable.

**Tasks**
1. Initialise a Molecule scenario with the Docker driver in your nginx role.
2. Test on two platforms (Ubuntu 24.04 and Rocky 9 or Debian 12).
3. Write `verify.yml` assertions: package present, service running, config content, HTTP 200, port listening.
4. Make the idempotence step fail on purpose with a bad `command` task, then fix it.
5. Add a second scenario that tests non-default variables.
6. Run in CI on pull requests.

**Expected outcome / acceptance criteria**
- `molecule test` passes on both platforms, and the logs show an `idempotence` step with zero changes.
- The failing task from step 4 breaks CI with a clear message.
- A matrix job runs both scenarios.

<details>
<summary>Hints</summary>

Use images built for systemd-in-Docker (for example the `geerlingguy/docker-*-ansible` family) with `privileged: true`, `cgroupns_mode: host`, and the cgroup volume mounted. Use `molecule converge` and `molecule login` while iterating, since a full test run recreates the container. The default verifier is Ansible itself; `testinfra` is an alternative with pytest.
</details>

**Bonus:** Add `ansible-lint` and `yamllint` as a pre-step and make the lint failure block the job.

---

## Assignment 23. Make a slow playbook fast 🔴

**Objective:** Measure first, then tune.

**Tasks**
1. Run a playbook on 20+ hosts (or 20 containers) and capture a baseline with `profile_tasks` and `time`.
2. Enable SSH pipelining and `ControlPersist` and re-measure.
3. Raise `forks`, set `gather_facts: false` or `gather_subset: [min]`, and enable fact caching with `jsonfile` or Redis.
4. Replace per-item package loops with list arguments.
5. Compare `strategy: linear` vs `strategy: free` and explain a case where `free` is unsafe.
6. Write a table of each change and its measured gain.

**Expected outcome / acceptance criteria**
- At least a 2x improvement on the same workload, backed by timing data.
- A clear note on the trade-off of each change (for example pipelining versus `requiretty`).

<details>
<summary>Hints</summary>

`ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks,ansible.posix.timer`. Pipelining lives under `[ssh_connection] pipelining = True`. A very high `forks` value can overload the controller or the SSH bastion. Look at `ANSIBLE_KEEP_REMOTE_FILES=1` when debugging, never in normal runs.
</details>

**Bonus:** Try `async` plus `poll: 0` and `async_status` for parallel long-running tasks on a single host.

---

## Assignment 24. Write a custom module with check mode 🔴

**Objective:** Understand what a module really is.

**Tasks**
1. Write `library/app_flag.py` that creates or removes a flag file and returns `changed` accurately.
2. Use `AnsibleModule` with `argument_spec`, `supports_check_mode=True` and `required_if`.
3. Return structured results (`path`, `state`, `previous`).
4. Add `DOCUMENTATION`, `EXAMPLES`, `RETURN` blocks so `ansible-doc app_flag` works.
5. Test with `ansible localhost -m app_flag` and in a playbook with `--check --diff`.
6. Write a pytest unit test using `AnsibleModule` patching, or a Molecule verify.

**Expected outcome / acceptance criteria**
- Running twice shows `changed` once, then `ok`.
- In check mode the file is not touched, but `changed` correctly predicts the result.
- `ansible-doc` renders your docs.

<details>
<summary>Hints</summary>

Use `module.exit_json(changed=..., **result)` and `module.fail_json(msg=...)`. In check mode, compute the change but return before performing it. Keep modules idempotent by comparing current and desired state before acting. Place the file in `library/` next to the playbook or in a collection's `plugins/modules/`.
</details>

**Bonus:** Package it in a collection with `galaxy.yml` and build it with `ansible-galaxy collection build`.

---

## Assignment 25. Full delivery pipeline for configuration code 🔴

**Objective:** Treat Ansible like software.

**Tasks**
1. Create a pipeline (GitHub Actions or GitLab CI) with stages: lint (`ansible-lint --profile production`, `yamllint`), Molecule test matrix, `--syntax-check`, and a `--check --diff` run against staging.
2. Store the vault password and SSH key in the CI secret store; load the key through an SSH agent step and delete the vault file after use.
3. Apply to staging automatically on merge, and to production only after a manual approval with `serial` rolling.
4. Add a nightly scheduled job that runs production with `--check --diff` and fails when `changed > 0` (drift report).
5. Build an execution environment image with `ansible-builder` containing pinned collections and Python deps, and run the playbook in it using `ansible-navigator` or in the CI container.

**Expected outcome / acceptance criteria**
- A pull request with a lint error or failing Molecule test cannot merge.
- Manual SSH edits on a staging host are reported by the nightly drift job.
- No secret appears in job logs (`no_log` and masked variables), and the pipeline file contains no credentials.

<details>
<summary>Hints</summary>

Parse the recap in the drift job (`grep -E 'changed=[1-9]'`), or use the JSON callback with `ANSIBLE_STDOUT_CALLBACK=json` and `jq`. Use environment-protected secrets for production only. Pin the EE base image and the collections to avoid surprise breakage. Prefer short-lived cloud credentials via OIDC rather than static keys.
</details>

**Bonus:** Add `ansible-pull` on one host as a comparison, and write down when pull beats push (large fleets, firewalled nodes, autoscaling at boot).

---

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
