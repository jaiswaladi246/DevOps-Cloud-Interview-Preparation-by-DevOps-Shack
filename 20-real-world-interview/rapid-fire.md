# Real-World Interview — Rapid-Fire Round

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

> Short, crisp questions across every DevOps tool: the round where interviewers fire 20 questions in 10 minutes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Real-World Interview](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Linux and shell](#linux-and-shell) — Q1–Q25
- [Git](#git) — Q26–Q40
- [Networking](#networking) — Q41–Q58
- [Docker](#docker) — Q59–Q80
- [Kubernetes](#kubernetes) — Q81–Q108
- [CI/CD](#cicd) — Q109–Q122
- [Terraform and Ansible](#terraform-and-ansible) — Q123–Q148
- [AWS](#aws) — Q149–Q170
- [Monitoring and security](#monitoring-and-security) — Q171–Q188
- [Mixed bag](#mixed-bag) — Q189–Q200

> **How to play this round.** Answer in one breath: the fact first, then a few words on why. If you know a command, say it. If you don't know, say what you'd check. Bluffing in a rapid-fire round is easy to spot.

---

## Linux and shell

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. What does exit code 0 mean, and how do you see the last command's exit code? 🟢

<details>
<summary><b>Answer</b></summary>

0 means success. Anything from 1 to 255 means failure, and the meaning depends on the program.

```bash
echo $?
```

</details>

### Q2. Hard link vs soft link? 🟢

<details>
<summary><b>Answer</b></summary>

A hard link is another name for the same inode. It survives deletion of the original but can't cross filesystems or point to directories. A symlink is a separate file holding a path, and it breaks if the target is removed.

```bash
ln file hard; ln -s /path/file soft
```

</details>

### Q3. What do the permissions `755` and `640` mean? 🟢

<details>
<summary><b>Answer</b></summary>

`755` = owner rwx, group r-x, others r-x. `640` = owner rw-, group r--, others nothing. Each digit is read (4) + write (2) + execute (1).

</details>

### Q4. How do you find which process is using the most CPU right now? 🟢

<details>
<summary><b>Answer</b></summary>

Run `top` (press `P` to sort by CPU) or `htop`, or use a one-shot command.

```bash
ps -eo pid,comm,%cpu --sort=-%cpu | head -5
```

</details>

### Q5. What is a zombie process? 🟡

<details>
<summary><b>Answer</b></summary>

A process that has exited but whose parent hasn't called `wait()` to read its exit status. It uses no CPU or memory, only a PID table entry. You fix it by fixing or killing the parent, not the zombie.

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
```

</details>

### Q6. SIGTERM vs SIGKILL? 🟢

<details>
<summary><b>Answer</b></summary>

SIGTERM (15) asks the process to exit and can be caught for graceful shutdown. SIGKILL (9) is handled by the kernel, can't be caught, and ends the process immediately with no cleanup.

</details>

### Q7. What does `load average: 4.00 2.00 1.00` tell you? 🟡

<details>
<summary><b>Answer</b></summary>

The average number of runnable plus uninterruptible (D state) tasks over 1, 5 and 15 minutes. Load is rising. Whether 4 is high depends on core count (`nproc`): on 4 cores it means fully busy.

</details>

### Q8. How do you check free memory, and why is "free" often low on a healthy server? 🟡

<details>
<summary><b>Answer</b></summary>

Run `free -h` and read the **available** column. Linux uses spare RAM for page cache, which it gives back on demand, so low "free" is normal.

</details>

### Q9. How do you follow a systemd service's logs live? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
journalctl -u nginx -f
journalctl -u nginx --since "10 min ago" -p err
```

</details>

### Q10. What is an inode, and what happens when inodes run out? 🟡

<details>
<summary><b>Answer</b></summary>

An inode stores a file's metadata (owner, permissions, block pointers) but not its name. When inodes run out you can't create files even if `df -h` shows free space. Millions of tiny files are the usual cause.

```bash
df -i
```

</details>

### Q11. `>` vs `>>` vs `2>&1`? 🟡

<details>
<summary><b>Answer</b></summary>

`>` overwrites, `>>` appends, and `2>&1` sends stderr to wherever stdout currently points. Order matters: `cmd > out.log 2>&1` captures both streams, while `cmd 2>&1 > out.log` does not.

</details>

### Q12. How do you run a command that survives logout? 🟢

<details>
<summary><b>Answer</b></summary>

For a quick job, use `nohup cmd &` or a `tmux`/`screen` session. For anything long-lived, make it a systemd service.

</details>

### Q13. What's in `/proc`? 🔴

<details>
<summary><b>Answer</b></summary>

A virtual filesystem the kernel generates live: per-process info (`/proc/<pid>/status`, `fd/`, `environ`) and system info (`/proc/meminfo`, `/proc/cpuinfo`, `/proc/loadavg`). Nothing in it is stored on disk.

</details>

### Q14. How do you make a systemd service start on boot and restart on failure? 🟡

<details>
<summary><b>Answer</b></summary>

Set `Restart=on-failure` (with `RestartSec=5`) in `[Service]` and `WantedBy=multi-user.target` in `[Install]`, then run:

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now myapp
```

</details>

### Q15. What does `sudo !!` do? 🟢

<details>
<summary><b>Answer</b></summary>

Reruns the previous command with sudo. `!!` is Bash history expansion of the last command.

</details>

### Q16. How do you see open files and network connections of a process? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
lsof -p <pid>
ls -l /proc/<pid>/fd
ss -tnp | grep <pid>
```

</details>

### Q17. What is the OOM killer? 🟡

<details>
<summary><b>Answer</b></summary>

The kernel mechanism that kills a process when memory is exhausted. It picks the victim by `oom_score` (memory use adjusted by `oom_score_adj`). Kills show up in `dmesg -T | grep -i oom` or `journalctl -k`.

</details>

### Q18. What does `chmod u+s` (setuid) do, and why is it risky? 🔴

<details>
<summary><b>Answer</b></summary>

The executable runs with the file owner's privileges (root for `/usr/bin/passwd`). A bug in a setuid-root binary is a privilege escalation path, so audit them regularly.

```bash
find / -perm -4000 -type f 2>/dev/null
```

</details>

### Q19. What are `/etc/passwd` and `/etc/shadow`? 🟢

<details>
<summary><b>Answer</b></summary>

`/etc/passwd` lists users (name, UID, GID, home, shell) and is world-readable. `/etc/shadow` holds password hashes and ageing, readable by root only.

</details>

### Q20. How do you schedule a script every day at 2:30 AM with cron? 🟢

<details>
<summary><b>Answer</b></summary>

Add this with `crontab -e`. Fields are minute, hour, day of month, month, day of week:

```bash
30 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

</details>

### Q21. What does `set -x` do? 🟢

<details>
<summary><b>Answer</b></summary>

It prints each command, after expansion, before running it. It's the quickest way to debug a Bash script. Turn it off with `set +x`.

</details>

### Q22. What is a file descriptor, and what are 0, 1 and 2? 🟡

<details>
<summary><b>Answer</b></summary>

An integer handle to an open file, socket or pipe. 0 is stdin, 1 is stdout, 2 is stderr. Per-process limits are shown by `ulimit -n`, and "Too many open files" means that limit was hit.

</details>

### Q23. What is the difference between a process and a thread? 🟡

<details>
<summary><b>Answer</b></summary>

A process has its own memory space. Threads inside a process share memory and file descriptors. Linux schedules both as tasks; threads are created by `clone()` with shared address space.

</details>

### Q24. What are cgroups and namespaces? 🔴

<details>
<summary><b>Answer</b></summary>

Namespaces isolate what a process can **see** (PID, network, mount, UTS, IPC, user, cgroup). cgroups limit what it can **use** (CPU, memory, I/O, PIDs). Together they are what a container is.

</details>

### Q25. How do you find which package a file belongs to? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
dpkg -S /usr/bin/curl        # Debian/Ubuntu
rpm -qf /usr/bin/curl        # RHEL/Rocky/Amazon Linux
```

</details>

## Git

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `git fetch` vs `git pull`? 🟢

<details>
<summary><b>Answer</b></summary>

`fetch` downloads remote commits and updates remote-tracking branches without touching your work. `pull` is `fetch` plus `merge` (or rebase, with `--rebase`) into your current branch.

</details>

### Q27. Merge vs rebase in one line each? 🟡

<details>
<summary><b>Answer</b></summary>

Merge keeps history as it happened and adds a merge commit. Rebase replays your commits on top of the target for a linear history but rewrites commit SHAs, so never rebase shared branches.

</details>

### Q28. How do you undo the last commit but keep the changes? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
git reset --soft HEAD~1     # changes stay staged
git reset HEAD~1            # changes stay, unstaged (--mixed)
```

</details>

### Q29. `git reset` vs `git revert`? 🟡

<details>
<summary><b>Answer</b></summary>

`reset` moves the branch pointer back and rewrites history, so use it on local work only. `revert` adds a new commit that undoes an old one, which makes it safe on shared branches like main.

</details>

### Q30. How do you recover a commit after a bad `reset --hard`? 🟡

<details>
<summary><b>Answer</b></summary>

The reflog keeps where HEAD has been, by default for about 90 days for reachable entries.

```bash
git reflog
git reset --hard HEAD@{2}
```

</details>

### Q31. What does `git stash` do? 🟢

<details>
<summary><b>Answer</b></summary>

It shelves uncommitted changes and cleans the working tree. Get them back with `git stash pop`, and use `git stash -u` to include untracked files.

</details>

### Q32. What is `git cherry-pick`? 🟢

<details>
<summary><b>Answer</b></summary>

It applies the change from a specific commit onto your current branch as a new commit. The common use is moving a hotfix from main to a release branch.

```bash
git cherry-pick <sha>
```

</details>

### Q33. How do you find which commit introduced a bug? 🟡

<details>
<summary><b>Answer</b></summary>

`git bisect` binary-searches history. `git bisect run` automates it with a test script.

```bash
git bisect start; git bisect bad; git bisect good v1.4.0
git bisect run ./test.sh
```

</details>

### Q34. A secret was committed and pushed. What do you do first? 🔴

<details>
<summary><b>Answer</b></summary>

**Rotate the secret first**, because it is already compromised. Then purge it from history with `git filter-repo` and force-push, and ask GitHub support to clear cached views if needed.

</details>

### Q35. What does `git push --force-with-lease` protect against? 🟡

<details>
<summary><b>Answer</b></summary>

It refuses the force-push if the remote branch has moved since you last fetched, so you don't wipe a teammate's commits by accident.

</details>

### Q36. Annotated vs lightweight tag? 🟡

<details>
<summary><b>Answer</b></summary>

An annotated tag is a full object with tagger, date and message, and it can be signed; use it for releases. A lightweight tag is just a named pointer to a commit.

```bash
git tag -a v1.2.0 -m "Release 1.2.0"
```

</details>

### Q37. What is a detached HEAD? 🟡

<details>
<summary><b>Answer</b></summary>

HEAD points straight at a commit instead of a branch, for example after `git checkout <sha>` or a tag. New commits there are lost unless you create a branch: `git switch -c fix-branch`.

</details>

### Q38. What is `.gitignore` and why doesn't it ignore a file already tracked? 🟡

<details>
<summary><b>Answer</b></summary>

`.gitignore` only affects untracked files. To stop tracking a file that's already committed, remove it from the index:

```bash
git rm --cached secrets.env
```

</details>

### Q39. Squash merge: what is it, and what's the trade-off? 🔴

<details>
<summary><b>Answer</b></summary>

It combines all PR commits into one commit on main. You get a clean history and easy reverts, but you lose the individual commits, and branches built on the PR branch need a rebase.

</details>

### Q40. What does `git log --oneline --graph --all` show? 🟢

<details>
<summary><b>Answer</b></summary>

A compact ASCII graph of every branch, with one line per commit. It's the fastest way to see how branches diverged before a merge or rebase.

</details>

## Networking

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q41. TCP vs UDP? 🟢

<details>
<summary><b>Answer</b></summary>

TCP is connection-oriented, reliable and ordered, with a handshake and retransmits (HTTP/1.1, HTTP/2, SSH). UDP is connectionless with no delivery guarantee, but low overhead (DNS, video, QUIC/HTTP/3).

</details>

### Q42. Describe the TCP three-way handshake. 🟢

<details>
<summary><b>Answer</b></summary>

The client sends SYN, the server replies SYN-ACK, and the client sends ACK. Only then does data flow.

</details>

### Q43. What happens when you type a URL in the browser, in 4 steps? 🟡

<details>
<summary><b>Answer</b></summary>

1. A DNS lookup resolves the hostname to an IP.
2. A TCP connection is opened (or QUIC for HTTP/3).
3. The TLS handshake runs: certificate check and key exchange.
4. The HTTP request is sent, the response comes back, and the browser renders it and fetches the page's other resources.

</details>

### Q44. A record vs CNAME vs ALIAS? 🟡

<details>
<summary><b>Answer</b></summary>

A maps a name to an IPv4 address (AAAA for IPv6). CNAME maps a name to another name and isn't allowed at the zone apex. Route 53 ALIAS maps a name to an AWS resource and does work at the apex.

</details>

### Q45. What is a /24 and how many usable hosts does it have? 🟡

<details>
<summary><b>Answer</b></summary>

A /24 has 256 addresses. Normally 254 are usable (network and broadcast addresses are reserved). In AWS it's 251, because AWS reserves 5 per subnet.

</details>

### Q46. What is the difference between ports 80, 443, 22, 53 and 3306? 🟢

<details>
<summary><b>Answer</b></summary>

HTTP, HTTPS, SSH, DNS (UDP and TCP) and MySQL respectively. Also worth knowing: 5432 PostgreSQL, 6379 Redis, 6443 Kubernetes API, 2379 etcd.

</details>

### Q47. How do you test whether a TCP port is open from a server? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
nc -vz db.internal 5432
curl -v telnet://db.internal:5432
```

</details>

### Q48. "Connection refused" vs "connection timed out"? 🟡

<details>
<summary><b>Answer</b></summary>

Refused means the host replied with a RST: it's reachable, but nothing is listening (or a firewall sent a reject). Timed out means the packets were dropped silently, which usually points to a firewall, security group or routing problem.

</details>

### Q49. How do you see the route a packet takes? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
traceroute example.com
mtr -rwc 50 example.com    # continuous, shows per-hop loss
```

</details>

### Q50. What is NAT, and why do private subnets need a NAT gateway? 🟡

<details>
<summary><b>Answer</b></summary>

NAT rewrites private source IPs to a public one. Instances without public IPs reach the internet (for patches or APIs) through a NAT gateway, while inbound connections from the internet stay impossible.

</details>

### Q51. Layer 4 vs Layer 7 load balancer? 🟡

<details>
<summary><b>Answer</b></summary>

L4 routes on IP and port (TCP/UDP) and is fast and protocol-agnostic, like AWS NLB. L7 understands HTTP, so it can route on host, path or headers and terminate TLS, like AWS ALB or NGINX.

</details>

### Q52. Forward proxy vs reverse proxy? 🟡

<details>
<summary><b>Answer</b></summary>

A forward proxy acts for clients reaching the internet (egress filtering, caching). A reverse proxy sits in front of servers for incoming traffic (load balancing, TLS termination, caching), like NGINX or Envoy.

</details>

### Q53. How do you query a specific DNS server and see TTL? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
dig @8.8.8.8 api.example.com +noall +answer
```

The second column of the answer is the remaining TTL in seconds.

</details>

### Q54. What does TLS actually give you? 🟡

<details>
<summary><b>Answer</b></summary>

Confidentiality (encryption), integrity (tamper detection) and server authentication (a certificate chain to a trusted CA). With mTLS, the client is authenticated too.

</details>

### Q55. What is MTU, and what symptom does an MTU mismatch cause? 🔴

<details>
<summary><b>Answer</b></summary>

MTU is the largest packet a link carries (1500 bytes on Ethernet; up to 9001 jumbo inside an AWS VPC). With a mismatch, small requests work but large transfers or TLS handshakes hang, especially over VPNs and tunnels where ICMP "fragmentation needed" messages are blocked.

</details>

### Q56. What is the difference between a security group and a NACL? 🔴

<details>
<summary><b>Answer</b></summary>

A security group is stateful, attached to an ENI or instance, and has allow rules only. A NACL is stateless, applies at the subnet level, has allow and deny rules evaluated in number order, and needs return traffic (ephemeral ports) allowed explicitly.

</details>

### Q57. What is SNI? 🔴

<details>
<summary><b>Answer</b></summary>

Server Name Indication: the client sends the hostname in the TLS ClientHello so one IP can serve many certificates. Without it you get the server's default certificate.

```bash
openssl s_client -connect 1.2.3.4:443 -servername app.example.com
```

</details>

### Q58. What is anycast? 🔴

<details>
<summary><b>Answer</b></summary>

The same IP is announced from many locations, and BGP routes each user to the nearest one. CDNs, public DNS like 1.1.1.1, and AWS Global Accelerator use it.

</details>

## Docker

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q59. Image vs container? 🟢

<details>
<summary><b>Answer</b></summary>

An image is a read-only, layered template. A container is a running (or stopped) instance of it, with a thin writable layer on top.

</details>

### Q60. CMD vs ENTRYPOINT? 🟡

<details>
<summary><b>Answer</b></summary>

ENTRYPOINT is the fixed executable. CMD gives default arguments that `docker run image <args>` replaces. Together: `ENTRYPOINT ["python","app.py"]` with `CMD ["--port","8000"]`.

</details>

### Q61. COPY vs ADD? 🟡

<details>
<summary><b>Answer</b></summary>

Prefer COPY, which simply copies files. ADD also extracts local tar archives and can fetch URLs, and that hidden behaviour is why COPY is the default choice.

</details>

### Q62. How do you see why an image is so big? 🟡

<details>
<summary><b>Answer</b></summary>

`docker history` shows the size of each layer. `dive` lets you explore the files in each layer interactively.

```bash
docker history --no-trunc myapp:1.0
dive myapp:1.0
```

</details>

### Q63. What's a dangling image, and how do you clean up disk? 🟢

<details>
<summary><b>Answer</b></summary>

A dangling image is an untagged layer left over from rebuilds.

```bash
docker image prune            # dangling only
docker system prune -a --volumes   # everything unused; careful
docker system df
```

</details>

### Q64. Bind mount vs named volume? 🟢

<details>
<summary><b>Answer</b></summary>

A bind mount maps a host path into the container, which suits development. A named volume is managed by Docker under `/var/lib/docker/volumes`, portable, and the right choice for persistent data.

</details>

### Q65. How do you get a shell in a running container? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker exec -it api sh     # or bash, if the image has it
```

</details>

### Q66. What does `EXPOSE` actually do? 🟡

<details>
<summary><b>Answer</b></summary>

Nothing at the network level. It's documentation and metadata. You still need `-p 8080:80` to publish, or `-P` to publish exposed ports to random host ports.

</details>

### Q67. Why run containers as non-root? 🟢

<details>
<summary><b>Answer</b></summary>

If the app is compromised, root in the container is much closer to root on the host (kernel bugs, mounted sockets). Add `USER 10001` in the Dockerfile.

</details>

### Q68. What restart policies does Docker have? 🟡

<details>
<summary><b>Answer</b></summary>

`no` (the default), `on-failure[:N]`, `always`, and `unless-stopped`, which is like always but stays down after you stop it manually.

```bash
docker run -d --restart unless-stopped nginx
```

</details>

### Q69. What is the default Docker network driver, and why use a user-defined bridge? 🟡

<details>
<summary><b>Answer</b></summary>

The default driver is `bridge`. A user-defined bridge adds built-in DNS, so containers resolve each other by name, and better isolation.

```bash
docker network create appnet
```

</details>

### Q70. How do you limit a container's memory and CPU? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker run --memory 512m --cpus 1.5 myapp
docker stats
```

</details>

### Q71. Why is exit code 137 so common? 🔴

<details>
<summary><b>Answer</b></summary>

137 = 128 + 9 (SIGKILL). It usually means the OOM killer hit the memory limit, or the process ignored SIGTERM and was force-killed after the stop timeout.

</details>

### Q72. What does a multi-stage build achieve? 🟡

<details>
<summary><b>Answer</b></summary>

You build in a heavy stage (compilers, SDKs) and copy only the artifacts into a slim final stage, so the image is smaller and has a smaller attack surface.

</details>

### Q73. What is the Docker build cache rule that breaks most builds? 🟡

<details>
<summary><b>Answer</b></summary>

Once a layer changes, every layer after it rebuilds. Put rarely changing steps (dependency install) before frequently changing ones (`COPY . .`).

</details>

### Q74. containerd vs Docker? 🔴

<details>
<summary><b>Answer</b></summary>

containerd is the container runtime daemon that manages images and container lifecycle, using runc underneath. Docker Engine is a developer-facing layer (CLI, build, networking) built on top of containerd. Kubernetes talks to containerd directly through CRI.

</details>

### Q75. What does `docker inspect` give you? Show one useful format string. 🟡

<details>
<summary><b>Answer</b></summary>

The full JSON config and state of a container or image.

```bash
docker inspect -f '{{.State.Status}} {{.NetworkSettings.IPAddress}}' api
```

</details>

### Q76. How do you copy a file out of a container? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
docker cp api:/app/logs/error.log ./error.log
```

</details>

### Q77. What is distroless, and what's the catch? 🔴

<details>
<summary><b>Answer</b></summary>

Images with only the app and its runtime: no shell and no package manager. That means fewer CVEs and less attack surface. The catch is you can't `exec` a shell to debug; use `kubectl debug` with an ephemeral container instead.

</details>

### Q78. How do you pass secrets to a container at runtime safely? 🟡

<details>
<summary><b>Answer</b></summary>

Mount them as files (Docker or Kubernetes secrets) or fetch them from a secrets manager at start. Environment variables work but leak through `docker inspect`, crash dumps and child processes. Never bake secrets into the image.

</details>

### Q79. What does `docker compose up -d --build` do? 🟢

<details>
<summary><b>Answer</b></summary>

It rebuilds images for services with a `build:` section, then creates or recreates changed containers in the background.

</details>

### Q80. What is the OCI? 🔴

<details>
<summary><b>Answer</b></summary>

The Open Container Initiative, which defines the image spec, runtime spec and distribution spec. Because of it, images built by Docker, Buildah or Kaniko run on any compliant runtime (containerd, CRI-O) and push to any compliant registry.

</details>

## Kubernetes

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q81. Pod vs Deployment? 🟢

<details>
<summary><b>Answer</b></summary>

A Pod is one or more containers sharing a network namespace and volumes, and it is mortal. A Deployment manages ReplicaSets to keep N pods running and handles rolling updates and rollbacks.

</details>

### Q82. Name the control plane components. 🟡

<details>
<summary><b>Answer</b></summary>

kube-apiserver (front door), etcd (state store), kube-scheduler (places pods), kube-controller-manager (reconcile loops), and cloud-controller-manager (cloud integration). On every node run kubelet, kube-proxy (or an eBPF replacement) and the container runtime.

</details>

### Q83. ClusterIP vs NodePort vs LoadBalancer? 🟢

<details>
<summary><b>Answer</b></summary>

ClusterIP is a virtual IP reachable inside the cluster only. NodePort adds a port (30000–32767 by default) on every node. LoadBalancer adds a cloud load balancer in front of the NodePort or pod IPs.

</details>

### Q84. Liveness vs readiness vs startup probe? 🟡

<details>
<summary><b>Answer</b></summary>

A failing liveness probe restarts the container. A failing readiness probe removes the pod from Service endpoints. A startup probe holds off the other two until the app has booted.

</details>

### Q85. Requests vs limits? 🟡

<details>
<summary><b>Answer</b></summary>

Requests are what the scheduler reserves for placement. Limits are the hard cap: going over the CPU limit gets the container throttled, and going over the memory limit gets it OOMKilled.

</details>

### Q86. What are the QoS classes? 🔴

<details>
<summary><b>Answer</b></summary>

Guaranteed (requests equal limits for every container), Burstable (some requests set) and BestEffort (none set). Under memory pressure, BestEffort pods are evicted first and Guaranteed pods last.

</details>

### Q87. How do you roll back a Deployment? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl rollout history deploy/api
kubectl rollout undo deploy/api --to-revision=3
```

</details>

### Q88. ConfigMap vs Secret? 🟡

<details>
<summary><b>Answer</b></summary>

Both hold key-value config. Secrets are base64-encoded (not encrypted by default), can be encrypted at rest with a KMS provider, and are access-controlled separately. Never treat base64 as security.

</details>

### Q89. StatefulSet vs Deployment? 🟡

<details>
<summary><b>Answer</b></summary>

A StatefulSet gives stable pod names (`db-0`, `db-1`), stable DNS through a headless Service, per-pod PVCs, and ordered rollout. Use it for databases and clustered stateful apps.

</details>

### Q90. What is a DaemonSet used for? 🟢

<details>
<summary><b>Answer</b></summary>

It runs one pod on every node (or on selected nodes). Typical uses are log shippers (Fluent Bit), node exporters, CNI agents and security agents.

</details>

### Q91. How do you get a shell in a pod, and debug one that has no shell? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
kubectl exec -it api-xyz -- sh
kubectl debug -it api-xyz --image=busybox:1.36 --target=api   # ephemeral container
```

</details>

### Q92. Taints/tolerations vs node affinity? 🟡

<details>
<summary><b>Answer</b></summary>

A taint repels pods from a node, and a toleration lets a pod ignore that taint. Affinity attracts pods to nodes. To dedicate nodes to a workload you need both.

</details>

### Q93. What does `kubectl apply` do differently from `kubectl create`? 🔴

<details>
<summary><b>Answer</b></summary>

`create` is imperative and fails if the object exists. `apply` is declarative: it creates or updates by three-way merge against the last-applied config, so it can be run repeatedly.

</details>

### Q94. What is a PV, a PVC and a StorageClass? 🟡

<details>
<summary><b>Answer</b></summary>

A PV is a piece of storage. A PVC is a pod's request for storage. A StorageClass tells the CSI driver how to provision PVs dynamically (for example `gp3` EBS) when a PVC asks.

</details>

### Q95. What happens when you delete a pod managed by a Deployment? 🟢

<details>
<summary><b>Answer</b></summary>

The ReplicaSet controller sees fewer pods than desired and creates a replacement right away. To actually remove pods, scale the Deployment down.

</details>

### Q96. How does a Service find its pods? 🔴

<details>
<summary><b>Answer</b></summary>

Through its label selector. The EndpointSlice controller tracks the IPs of Ready pods that match, and kube-proxy (iptables or IPVS) or an eBPF data plane routes the ClusterIP to them.

</details>

### Q97. What is etcd, and why back it up? 🔴

<details>
<summary><b>Answer</b></summary>

etcd is the Raft-based key-value store that holds all cluster state. Lose it without a backup and you lose the cluster's objects. On managed Kubernetes (EKS, GKE) the provider runs it for you.

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db
```

</details>

### Q98. What is an Ingress vs the Gateway API? 🔴

<details>
<summary><b>Answer</b></summary>

Ingress is the older L7 routing API, limited and extended through annotations. Gateway API is its successor (GatewayClass, Gateway, HTTPRoute), with role separation, header and weight routing, and portable features.

</details>

### Q99. What is a PodDisruptionBudget? 🟡

<details>
<summary><b>Answer</b></summary>

It limits how many pods of an app can be down at once from **voluntary** disruptions such as drains and upgrades (`minAvailable` or `maxUnavailable`). It doesn't protect against node crashes.

</details>

### Q100. What is a CRD and an Operator? 🔴

<details>
<summary><b>Answer</b></summary>

A CRD adds a new resource type to the API, like `Certificate` or `Rollout`. An Operator is a controller that reconciles those resources, encoding human operations knowledge (backups, failover) in code.

</details>

### Q101. How do you see resource usage of pods and nodes? 🟢

<details>
<summary><b>Answer</b></summary>

These need metrics-server installed.

```bash
kubectl top pods -A --sort-by=memory
kubectl top nodes
```

</details>

### Q102. What does `kubectl drain` do? 🟡

<details>
<summary><b>Answer</b></summary>

It cordons the node (no new pods) and evicts the existing pods while respecting PDBs. It's used before maintenance or an upgrade.

```bash
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
```

</details>

### Q103. What's the difference between HPA, VPA and Cluster Autoscaler/Karpenter? 🔴

<details>
<summary><b>Answer</b></summary>

HPA changes the pod count. VPA changes pod requests. Cluster Autoscaler or Karpenter changes the node count when pods can't be scheduled.

</details>

### Q104. What is a headless Service? 🔴

<details>
<summary><b>Answer</b></summary>

A Service with `clusterIP: None`. DNS returns the individual pod IPs instead of one virtual IP. StatefulSets use it to give pods stable names like `db-0.db`.

</details>

### Q105. How do you switch namespace for your current context? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl config set-context --current --namespace=staging
```

</details>

### Q106. What does `imagePullPolicy: IfNotPresent` mean, and what's the default? 🔴

<details>
<summary><b>Answer</b></summary>

`IfNotPresent` pulls only if the image isn't already on the node. The default is `Always` when the tag is `:latest` or omitted, and `IfNotPresent` for any other tag.

</details>

### Q107. What are Pod Security Standards? 🔴

<details>
<summary><b>Answer</b></summary>

Three built-in profiles, privileged, baseline and restricted, enforced per namespace by the Pod Security Admission controller through labels. It replaced PodSecurityPolicy, which was removed in Kubernetes 1.25.

```bash
kubectl label ns prod pod-security.kubernetes.io/enforce=restricted
```

</details>

### Q108. How do you get all pods with their node and IP in one command? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
kubectl get pods -A -o wide
```

</details>

## CI/CD

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q109. CI vs continuous delivery vs continuous deployment? 🟢

<details>
<summary><b>Answer</b></summary>

CI means every merge is built and tested automatically. Continuous delivery means every passing build is releasable, with a human deciding when. Continuous deployment means every passing build goes to production automatically.

</details>

### Q110. What is an artifact, and why "build once, deploy many"? 🟡

<details>
<summary><b>Answer</b></summary>

An artifact is the built output (an image, JAR or zip). Promoting the same immutable artifact through every environment means production runs exactly what was tested. Rebuilding per environment breaks that guarantee.

</details>

### Q111. What are the DORA metrics? 🟡

<details>
<summary><b>Answer</b></summary>

Deployment frequency, lead time for changes, change failure rate, and time to restore service. They measure delivery speed and stability together.

</details>

### Q112. Blue/green vs canary vs rolling? 🟡

<details>
<summary><b>Answer</b></summary>

Rolling replaces instances gradually. Blue/green runs two full environments and switches all traffic at once, which gives instant rollback. Canary sends a small percentage of traffic to the new version and increases it while metrics stay healthy.

</details>

### Q113. What is GitOps in one sentence? 🟢

<details>
<summary><b>Answer</b></summary>

Git is the single source of truth for desired state, and an in-cluster agent (Argo CD or Flux) pulls and continuously reconciles the cluster to match it.

</details>

### Q114. Jenkins declarative vs scripted pipeline? 🟡

<details>
<summary><b>Answer</b></summary>

Declarative uses the structured `pipeline { stages { ... } }` syntax with validation, `post`, `when` and `options`. Scripted is free-form Groovy (`node { }`). Prefer declarative and drop into `script { }` only when you need logic.

</details>

### Q115. What does `needs:` do in GitHub Actions? 🟢

<details>
<summary><b>Answer</b></summary>

It makes a job wait for other jobs and lets it read their outputs through `needs.<job>.outputs`. Without `needs`, jobs run in parallel.

</details>

### Q116. How do you keep secrets out of CI logs? 🟡

<details>
<summary><b>Answer</b></summary>

Use the platform's secret store (values are masked automatically), use OIDC instead of long-lived keys, never `echo` secrets or run `set -x` around them, and add `::add-mask::` for values generated at runtime.

</details>

### Q117. Why pin GitHub Actions to a commit SHA? 🔴

<details>
<summary><b>Answer</b></summary>

Tags like `@v4` can be moved by the action's owner or by an attacker who compromises it. A full SHA is immutable, which protects against supply-chain attacks such as the 2025 tj-actions/changed-files compromise.

</details>

### Q118. What is a merge queue? 🟡

<details>
<summary><b>Answer</b></summary>

It tests each PR against the latest main plus the PRs queued ahead of it before merging, so main never breaks from two PRs that pass separately but conflict together.

</details>

### Q119. What is a flaky test, and how do you handle it? 🟡

<details>
<summary><b>Answer</b></summary>

A test that passes and fails on the same code. Quarantine it (mark it as non-blocking and track it), fix the root cause (timing, shared state, ordering), and don't hide it with blind retries.

</details>

### Q120. Where should database migrations run in a pipeline? 🔴

<details>
<summary><b>Answer</b></summary>

As a separate, single-run step before the new app version rolls out (a Kubernetes Job or Helm pre-upgrade hook), and backward compatible with the old version through expand/contract. Don't run them in every replica's startup.

</details>

### Q121. What's the purpose of Argo CD's sync and self-heal? 🟡

<details>
<summary><b>Answer</b></summary>

Sync applies what's in Git. Self-heal reverts manual changes made in the cluster back to the Git state. With prune enabled, resources deleted from Git are deleted from the cluster.

</details>

### Q122. What is shift-left security in CI? 🟡

<details>
<summary><b>Answer</b></summary>

Running security checks early in the PR (secret scanning, SAST, dependency/SCA scanning, IaC scanning, image scanning) so issues are fixed before merge, when they are cheapest to fix.

</details>

## Terraform and Ansible

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q123. What is Terraform state, and why is it needed? 🟡

<details>
<summary><b>Answer</b></summary>

State maps resources in your code to real resource IDs and stores their attributes. Terraform uses it to plan diffs. Keep it remote, locked and encrypted, because it can contain secrets.

</details>

### Q124. `terraform plan` vs `apply` vs `refresh`? 🟡

<details>
<summary><b>Answer</b></summary>

`plan` shows the changes. `apply` executes them. Refresh updates state from the real world; the standalone command is deprecated, so use `terraform apply -refresh-only`.

</details>

### Q125. What does `terraform init` do? 🟢

<details>
<summary><b>Answer</b></summary>

It configures the backend, downloads providers (recording them in `.terraform.lock.hcl`) and downloads modules. Run it again after changing the backend, providers or module sources.

</details>

### Q126. `count` vs `for_each`? 🟡

<details>
<summary><b>Answer</b></summary>

`count` indexes resources by number, so removing a middle item shifts the others and recreates them. `for_each` keys resources by a map or set key, so the rest stay stable. Prefer `for_each` for collections.

</details>

### Q127. What is drift, and how do you detect it? 🟡

<details>
<summary><b>Answer</b></summary>

Drift is when real infrastructure has been changed outside Terraform. Detect it with a scheduled `terraform plan -detailed-exitcode`, where exit code 2 means there are changes.

</details>

### Q128. How do you target or replace a single resource? 🟡

<details>
<summary><b>Answer</b></summary>

`-replace` supersedes `terraform taint`. Use `-target` only in emergencies.

```bash
terraform apply -replace="aws_instance.web"
terraform plan -target="module.vpc"
```

</details>

### Q129. Variable vs local vs output? 🟢

<details>
<summary><b>Answer</b></summary>

A variable is an input to a module. A local is a computed value used inside a module. An output exposes a value to the caller or the CLI.

</details>

### Q130. Why commit `.terraform.lock.hcl`? 🔴

<details>
<summary><b>Answer</b></summary>

It pins exact provider versions and their checksums, so every teammate and CI run uses the same provider builds. Upgrade on purpose with `terraform init -upgrade`.

</details>

### Q131. What does `sensitive = true` do, and what doesn't it do? 🔴

<details>
<summary><b>Answer</b></summary>

It hides the value in plan and apply output. The value is **still stored in plain text in state**, so protect the state bucket with encryption and IAM. Terraform 1.10+ ephemeral values and resources can keep some secrets out of state entirely.

</details>

### Q132. What's a provider alias used for? 🔴

<details>
<summary><b>Answer</b></summary>

It configures several instances of the same provider, such as two regions or two accounts. For example, ACM certificates for CloudFront must be in us-east-1:

```hcl
provider "aws" {
  alias  = "use1"
  region = "us-east-1"
}
```

</details>

### Q133. How do you upgrade a module version safely? 🟡

<details>
<summary><b>Answer</b></summary>

Pin `version = "~> 5.1"`, bump it in a PR, run `terraform init -upgrade` and plan, read the changelog for breaking changes, and roll out environment by environment.

</details>

### Q134. What is Terragrunt for? 🔴

<details>
<summary><b>Answer</b></summary>

It's a thin wrapper that keeps backend and provider configuration DRY across many environments and accounts, handles dependencies between stacks, and runs commands across many stacks (`terragrunt run-all plan`).

</details>

### Q135. Is Ansible agent-based? What does it need on the target? 🟢

<details>
<summary><b>Answer</b></summary>

No, it's agentless. It needs SSH access (WinRM for Windows) and Python on the target for most modules.

</details>

### Q136. What does idempotent mean in Ansible? 🟢

<details>
<summary><b>Answer</b></summary>

Running the playbook again produces the same end state and reports `changed=0` when nothing needed to change. Modules check current state before acting.

</details>

### Q137. Ansible `command` vs `shell` module? 🟡

<details>
<summary><b>Answer</b></summary>

`command` runs without a shell, so pipes, redirects and `$VARS` don't work, which also makes it safer. `shell` runs through `/bin/sh`. Prefer a dedicated module over either one.

</details>

### Q138. What are handlers? 🟢

<details>
<summary><b>Answer</b></summary>

Tasks that run only when another task notifies them with `changed`, once, at the end of the play. The typical use is restarting a service after a config change.

</details>

### Q139. Where do you put variables in Ansible, lowest to highest priority, briefly? 🔴

<details>
<summary><b>Answer</b></summary>

Role `defaults/` (lowest), then inventory and `group_vars`/`host_vars`, then play `vars`, then role `vars/`, then `set_fact`/registered vars, and finally `-e` extra vars, which always win.

</details>

### Q140. How do you dry-run a playbook and see what would change? 🟢

<details>
<summary><b>Answer</b></summary>

```bash
ansible-playbook site.yml --check --diff
```

</details>

### Q141. What is `ansible-galaxy` used for? 🟢

<details>
<summary><b>Answer</b></summary>

It installs roles and collections, and scaffolds new roles.

```bash
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role init myrole
```

</details>

### Q142. What does `serial` do in a play? 🟡

<details>
<summary><b>Answer</b></summary>

It runs the play on a batch of hosts at a time (`serial: 2` or `serial: "25%"`) for rolling updates. Combine it with `max_fail_percentage` to stop a bad rollout early.

</details>

### Q143. What is `delegate_to`? 🔴

<details>
<summary><b>Answer</b></summary>

It runs a task on a different host than the current target, but in that target's context. A common use is calling a load balancer API from `localhost` to deregister the current host.

</details>

### Q144. How do you speed up a slow Ansible run? 🔴

<details>
<summary><b>Answer</b></summary>

Raise `forks`, enable SSH pipelining (`pipelining = True`), use `gather_facts: false` or fact caching, use `strategy: free` where ordering doesn't matter, and pass lists to package modules instead of looping.

</details>

### Q145. Ansible vs Terraform: when do you use each? 🟡

<details>
<summary><b>Answer</b></summary>

Terraform provisions infrastructure declaratively and keeps state (VPCs, clusters, databases). Ansible configures what runs on machines, procedurally and without state (packages, files, services). They're often used together.

</details>

### Q146. What is a Jinja2 filter? Give two examples. 🟡

<details>
<summary><b>Answer</b></summary>

A filter transforms a value inside a template. For example `{{ name | upper }}`, and `{{ port | default(8080) }}`, which supplies a fallback when the variable is undefined.

</details>

### Q147. What does `become` do? 🟢

<details>
<summary><b>Answer</b></summary>

It escalates privilege for a task or play, using sudo by default. Use `become_user` to switch to a user other than root.

</details>

### Q148. `import_tasks` vs `include_tasks`? 🔴

<details>
<summary><b>Answer</b></summary>

`import_*` is static and processed at parse time, so tags and `when` apply to every imported task. `include_*` is dynamic and evaluated at runtime, so it can be used with loops and variable file names.

</details>

## AWS

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q149. Region vs Availability Zone? 🟢

<details>
<summary><b>Answer</b></summary>

A region is a geographic area (for example ap-south-1, Mumbai). An AZ is one or more isolated data centres inside a region. Spread across at least 2 AZs, ideally 3, for high availability.

</details>

### Q150. IAM user vs role? 🟡

<details>
<summary><b>Answer</b></summary>

A user has long-term credentials and represents a person or legacy system. A role is assumed for short-lived credentials by services, EC2, pods or federated users. Prefer roles everywhere.

</details>

### Q151. What is the order of IAM policy evaluation? 🔴

<details>
<summary><b>Answer</b></summary>

An explicit Deny anywhere always wins. Otherwise access needs an explicit Allow, and also has to be permitted by every applicable boundary (SCPs, permission boundaries, session policies, and resource policies where they apply). The default is implicit deny.

</details>

### Q152. S3 storage classes in one line each? 🟡

<details>
<summary><b>Answer</b></summary>

Standard is for hot data. Intelligent-Tiering moves objects automatically based on access. Standard-IA and One Zone-IA are for infrequent access. Glacier Instant, Flexible and Deep Archive are for archives, with retrieval from milliseconds up to hours.

</details>

### Q153. What is a VPC endpoint, and why use one? 🟡

<details>
<summary><b>Answer</b></summary>

It gives private access to AWS services without the internet or a NAT gateway. Gateway endpoints (S3, DynamoDB) are free. Interface endpoints (PrivateLink) are ENIs that cost per hour. They cut NAT data charges and keep traffic private.

</details>

### Q154. ALB vs NLB? 🟡

<details>
<summary><b>Answer</b></summary>

ALB works at L7 (HTTP/HTTPS) with path and host routing, WAF and OIDC auth. NLB works at L4 (TCP/UDP/TLS) with very high throughput, static IPs, and it preserves the client IP.

</details>

### Q155. What does an Auto Scaling Group need to replace unhealthy instances properly? 🟡

<details>
<summary><b>Answer</b></summary>

A launch template, subnets across AZs, and `health_check_type = ELB` so failing load balancer health checks trigger replacement, not just EC2 status checks. Add a grace period for boot time.

</details>

### Q156. RDS Multi-AZ vs read replica? 🔴

<details>
<summary><b>Answer</b></summary>

Multi-AZ is a synchronous standby for automatic failover, and it isn't readable in the classic setup. A read replica uses asynchronous replication to scale reads and can be promoted, including across regions, for DR.

</details>

### Q157. What is IMDSv2, and why enforce it? 🔴

<details>
<summary><b>Answer</b></summary>

The session-token version of the instance metadata service: a PUT for a token first, then requests that carry the token. It blocks SSRF attacks that steal instance role credentials through simple GETs.

```bash
TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
```

</details>

### Q158. How do you find who deleted a resource? 🟢

<details>
<summary><b>Answer</b></summary>

CloudTrail. Search the event history (90 days by default) for `Delete*` events on that resource, or query a trail stored in S3 with Athena or CloudTrail Lake.

</details>

### Q159. What is an SCP? 🔴

<details>
<summary><b>Answer</b></summary>

A Service Control Policy in AWS Organizations. It sets the maximum permissions for accounts in an OU but grants nothing itself. Example: deny all regions except ap-south-1, or deny leaving the organization.

</details>

### Q160. EBS vs EFS vs S3? 🟡

<details>
<summary><b>Answer</b></summary>

EBS is block storage attached to one instance in one AZ. EFS is managed NFS shared across instances and AZs. S3 is object storage over HTTP with effectively unlimited capacity.

</details>

### Q161. How do pods on EKS get AWS permissions without keys? 🔴

<details>
<summary><b>Answer</b></summary>

IRSA (an OIDC provider plus a service account annotated with a role ARN) or EKS Pod Identity (the agent add-on plus an association). Each maps a Kubernetes service account to an IAM role that issues short-lived credentials.

</details>

### Q162. What are the main EC2 purchase options? 🟡

<details>
<summary><b>Answer</b></summary>

On-Demand, Savings Plans and Reserved Instances (commit for 1 or 3 years for a big discount), Spot (up to about 90% off but can be reclaimed with a 2-minute warning), and Dedicated Hosts.

</details>

### Q163. Lambda cold start: what is it, and how do you reduce it? 🟡

<details>
<summary><b>Answer</b></summary>

The extra latency when a new execution environment is initialised. Reduce it with smaller packages, lighter runtimes, initialising clients outside the handler, provisioned concurrency, or SnapStart (Java, and since 2024 Python and .NET).

</details>

### Q164. SQS standard vs FIFO? 🔴

<details>
<summary><b>Answer</b></summary>

Standard has nearly unlimited throughput, at-least-once delivery and best-effort ordering. FIFO gives ordering within a message group and exactly-once processing via deduplication, with lower throughput limits.

</details>

### Q165. What is a NAT gateway's biggest hidden cost? 🔴

<details>
<summary><b>Answer</b></summary>

The per-GB data processing charge on top of the hourly charge. Traffic to S3, ECR or DynamoDB through NAT adds up fast. Use gateway and interface VPC endpoints instead.

</details>

### Q166. CloudWatch Logs vs metrics vs alarms? 🟢

<details>
<summary><b>Answer</b></summary>

Logs are raw events. Metrics are numeric time series (and metric filters can derive them from logs). Alarms watch a metric against a threshold and trigger SNS, Auto Scaling or other actions.

</details>

### Q167. How does Route 53 failover routing work? 🟡

<details>
<summary><b>Answer</b></summary>

A primary record is tied to a health check and a secondary record serves when the check fails. Use a low TTL so clients follow the change quickly.

</details>

### Q168. What is AWS Systems Manager Session Manager good for? 🟢

<details>
<summary><b>Answer</b></summary>

Shell access to instances without SSH keys, bastions or open port 22, controlled by IAM and logged to CloudWatch or S3. It needs the SSM agent and an instance role.

</details>

### Q169. Secrets Manager vs Parameter Store? 🟡

<details>
<summary><b>Answer</b></summary>

Secrets Manager has built-in rotation and cross-region replication, and costs per secret. Parameter Store SecureString is cheaper (standard tier free), good for config and simple secrets, but has no native rotation.

</details>

### Q170. What is the AWS shared responsibility model in one line? 🟢

<details>
<summary><b>Answer</b></summary>

AWS secures the cloud itself (hardware, facilities, the managed service layer). You secure what you put in it (IAM, data, OS patching on EC2, network config, encryption choices).

</details>

## Monitoring and security

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q171. Monitoring vs observability? 🟢

<details>
<summary><b>Answer</b></summary>

Monitoring watches known failure modes with predefined metrics and alerts. Observability is being able to ask new questions about unknown failures, using rich metrics, logs and traces with correlated context.

</details>

### Q172. What are the three pillars of observability? 🟢

<details>
<summary><b>Answer</b></summary>

Metrics (cheap numeric trends), logs (detailed events) and traces (one request's path across services). Profiles are increasingly counted as a fourth.

</details>

### Q173. SLI vs SLO vs SLA? 🟡

<details>
<summary><b>Answer</b></summary>

An SLI is the measurement (for example the percentage of successful requests). An SLO is the internal target (99.9% over 30 days). An SLA is the contractual promise to customers, with penalties, and is usually looser than the SLO.

</details>

### Q174. What are the four golden signals? 🟡

<details>
<summary><b>Answer</b></summary>

Latency, traffic, errors and saturation, from the Google SRE book. RED (Rate, Errors, Duration) applies them to services and USE (Utilisation, Saturation, Errors) to resources.

</details>

### Q175. Prometheus: pull or push, and how do short-lived jobs report? 🟡

<details>
<summary><b>Answer</b></summary>

Prometheus pulls, scraping `/metrics`. Short-lived batch jobs push to a Pushgateway, which Prometheus then scrapes.

</details>

### Q176. Counter vs gauge vs histogram? 🔴

<details>
<summary><b>Answer</b></summary>

A counter only goes up (use `rate()` on it). A gauge goes up and down (memory, queue depth). A histogram counts observations into buckets, so you can compute percentiles with `histogram_quantile()`.

</details>

### Q177. Write the PromQL for the 5xx error ratio over 5 minutes. 🟡

<details>
<summary><b>Answer</b></summary>

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

</details>

### Q178. Write PromQL for p99 latency from a histogram. 🔴

<details>
<summary><b>Answer</b></summary>

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

Keep `le` in the `by` clause, or the quantile can't be computed.

</details>

### Q179. What is high cardinality, and why does it hurt? 🔴

<details>
<summary><b>Answer</b></summary>

Labels with many unique values (user ID, request ID, full URL) create a separate time series for every combination. That blows up memory and storage in Prometheus. Put those values in logs or traces, not in metric labels.

</details>

### Q180. What makes a good alert? 🟡

<details>
<summary><b>Answer</b></summary>

It's actionable, urgent, tied to user impact (symptoms such as SLO burn rate, not causes like CPU at 80%), and links a runbook. If nobody needs to act now, make it a ticket or a dashboard, not a page.

</details>

### Q181. What is OpenTelemetry? 🟡

<details>
<summary><b>Answer</b></summary>

A vendor-neutral CNCF standard of APIs, SDKs and the Collector for generating and exporting traces, metrics and logs. You instrument once and send the data to any backend (Tempo, Jaeger, Datadog and others).

</details>

### Q182. What is the principle of least privilege? 🟢

<details>
<summary><b>Answer</b></summary>

Give every identity (human, service, pipeline) only the permissions it needs, only for as long as it needs them. It limits the damage a leaked credential can do.

</details>

### Q183. SAST vs DAST vs SCA? 🟡

<details>
<summary><b>Answer</b></summary>

SAST scans source code for bugs (Semgrep, SonarQube). DAST attacks the running app (OWASP ZAP). SCA finds vulnerable dependencies (Trivy, Snyk, Dependabot).

</details>

### Q184. What is an SBOM? 🟡

<details>
<summary><b>Answer</b></summary>

A Software Bill of Materials: a machine-readable list of every component and version in an artifact (SPDX or CycloneDX). It lets you answer "are we affected by this CVE?" in minutes.

```bash
syft myapp:1.0 -o cyclonedx-json > sbom.json
```

</details>

### Q185. How do you sign and verify a container image? 🔴

<details>
<summary><b>Answer</b></summary>

With Sigstore cosign. Keyless signing uses OIDC identity from CI. Enforce verification at admission with Kyverno or policy-controller.

```bash
cosign sign myrepo/app@sha256:<digest>
cosign verify myrepo/app@sha256:<digest> --certificate-identity-regexp '.*' --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

</details>

### Q186. Encryption at rest vs in transit? 🟢

<details>
<summary><b>Answer</b></summary>

At rest protects stored data (KMS-encrypted EBS, S3, RDS). In transit protects data moving over the network (TLS, mTLS). Production needs both.

</details>

### Q187. What does a WAF protect against? 🟡

<details>
<summary><b>Answer</b></summary>

L7 attacks such as SQL injection, XSS, bad bots and request floods (through rate-based rules), placed in front of an ALB, CloudFront or API Gateway. It doesn't replace fixing the app.

</details>

### Q188. What is zero trust in one line? 🔴

<details>
<summary><b>Answer</b></summary>

Never trust based on network location. Authenticate and authorise every request using identity and device posture, encrypt everywhere, and give least-privilege access per resource.

</details>

## Mixed bag

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q189. What is the difference between horizontal and vertical scaling? 🟢

<details>
<summary><b>Answer</b></summary>

Horizontal scaling adds more instances and needs a stateless design, but has no hard ceiling. Vertical scaling makes one instance bigger; it's simple but limited, and often needs a restart.

</details>

### Q190. Stateless vs stateful service? 🟡

<details>
<summary><b>Answer</b></summary>

A stateless service keeps no client data between requests, so any replica can serve any request and scaling is easy. A stateful service keeps data (databases, sessions on local disk), so it needs stable identity and storage.

</details>

### Q191. What is idempotency, and why does it matter for retries? 🔴

<details>
<summary><b>Answer</b></summary>

Performing an operation twice has the same effect as performing it once. Retries after timeouts are only safe for idempotent operations, so use idempotency keys for things like payments.

</details>

### Q192. What is a circuit breaker? 🔴

<details>
<summary><b>Answer</b></summary>

After repeated failures to a dependency, it "opens" and fails fast for a cool-down period instead of piling up slow calls, then lets a few trial requests through (half-open). It prevents cascading failures.

</details>

### Q193. What is a post-mortem, and what makes it "blameless"? 🟢

<details>
<summary><b>Answer</b></summary>

A written review after an incident: timeline, impact, root causes, what went well, and action items. Blameless means it focuses on system and process gaps, not on who made the mistake, so people report honestly.

</details>

### Q194. What does "cattle, not pets" mean? 🟢

<details>
<summary><b>Answer</b></summary>

Servers are identical, disposable units built from code, replaced rather than repaired, and never hand-tuned. Immutable images and autoscaling groups are the practice.

</details>

### Q195. What is toil, in SRE terms? 🟡

<details>
<summary><b>Answer</b></summary>

Manual, repetitive, automatable work with no lasting value that grows with service size. SRE practice caps it (Google targets under 50% of time) and automates it away.

</details>

### Q196. What is the 12-factor app rule about config? 🟡

<details>
<summary><b>Answer</b></summary>

Keep config that varies between deploys in the environment, not in code. That's what lets the same build artifact run in dev, staging and production.

</details>

### Q197. What's an error budget? 🟡

<details>
<summary><b>Answer</b></summary>

100% minus the SLO. At 99.9% over 30 days that's about 43 minutes of allowed unreliability. While budget remains, ship features; when it's spent, prioritise reliability work.

</details>

### Q198. Mean time to detect vs mean time to recover: which do you improve first? 🔴

<details>
<summary><b>Answer</b></summary>

Usually MTTD. You can't recover from what you haven't noticed, and good SLO-based alerting often cuts total impact more than faster fixes. Then cut MTTR with fast rollbacks and feature flags.

</details>

### Q199. What would you check first if "the site is slow" with no other information? 🔴

<details>
<summary><b>Answer</b></summary>

Scope and timing first: is it everyone or one region, and did anything change (deploys, config, traffic)? Then the golden-signal dashboards from the edge inward (CDN/LB latency, then app p99, then DB and dependencies), and traces for one slow request.

</details>

### Q200. If you could automate only one thing on a new team, what would it be? 🔴

<details>
<summary><b>Answer</b></summary>

A reliable, fast path to production: CI on every PR plus one-click (or automatic) deploy and rollback. It speeds up every other improvement and reduces risk on every change. Back it with a concrete metric such as lead time.

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
