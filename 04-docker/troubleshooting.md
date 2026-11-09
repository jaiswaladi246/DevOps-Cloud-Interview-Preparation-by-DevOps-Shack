# Docker — Troubleshooting

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

> Realistic Docker symptoms with exact error text, diagnosis commands and fixes · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Docker](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Daemon, install and permissions](#daemon-install-and-permissions) — Q1–Q25
- [Build failures and cache problems](#build-failures-and-cache-problems) — Q26–Q50
- [Containers that exit, crash or misbehave](#containers-that-exit-crash-or-misbehave) — Q51–Q75
- [Networking, DNS and Compose](#networking-dns-and-compose) — Q76–Q100
- [Storage, volumes and disk space](#storage-volumes-and-disk-space) — Q101–Q125
- [Registry, pull and push errors](#registry-pull-and-push-errors) — Q126–Q150
- [Resources, health checks, logging and restarts](#resources-health-checks-logging-and-restarts) — Q151–Q175
- [Security, rootless, Podman and containerd](#security-rootless-podman-and-containerd) — Q176–Q200

---

## Daemon, install and permissions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** dockerd is stopped, crashed, not installed, or `DOCKER_HOST`/context points elsewhere.

**Diagnose →**
```bash
systemctl status docker
journalctl -u docker --no-pager -n 50
echo $DOCKER_HOST; docker context ls
```

**Fix →** `sudo systemctl enable --now docker`. Unset a stale `DOCKER_HOST` or `docker context use default`. On Docker Desktop start the app.

**Prevent →** Enable the service at boot and monitor it.

</details>

### Q2. `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** User isn't in the `docker` group (socket is `root:docker 0660`) or group membership not refreshed.

**Diagnose →**
```bash
ls -l /var/run/docker.sock
id -nG
```

**Fix →** `sudo usermod -aG docker $USER` then `newgrp docker` or re-login. Do not `chmod 666` the socket.

**Prevent →** Remember the group is root-equivalent; prefer rootless for developers.

</details>

### Q3. dockerd fails to start after editing `daemon.json`: `unable to configure the Docker daemon with file /etc/docker/daemon.json` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Invalid JSON, unknown key, or a conflict with a flag in the systemd unit (`the following directives are specified both as a flag and in the configuration file: hosts`).

**Diagnose →**
```bash
dockerd --validate --config-file /etc/docker/daemon.json
jq . /etc/docker/daemon.json
journalctl -xeu docker
```

**Fix →** Correct JSON; for the `hosts` conflict create a systemd drop-in that clears `ExecStart` flags: `ExecStart=` then `ExecStart=/usr/bin/dockerd`.

**Prevent →** Validate config in CI before rollout.

</details>

### Q4. `docker: command not found` right after installing Docker Desktop on macOS. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Desktop not started, CLI symlinks not created, or PATH missing `/usr/local/bin`.

**Diagnose →** `ls -l /usr/local/bin/docker`, `which docker`, check Desktop settings for "Install CLI tools".

**Fix →** Start Docker Desktop, accept system privileges to create symlinks, or add `~/.docker/bin` to PATH.

**Prevent →** Standardise installs via MDM.

</details>

### Q5. `docker info` shows `WARNING: bridge-nf-call-iptables is disabled` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `br_netfilter` module not loaded or sysctl off.

**Diagnose →** `lsmod | grep br_netfilter; sysctl net.bridge.bridge-nf-call-iptables`

**Fix →**
```bash
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf
echo 'net.bridge.bridge-nf-call-iptables=1' | sudo tee /etc/sysctl.d/99-docker.conf
sudo sysctl --system
```

**Prevent →** Bake into the host image.

</details>

### Q6. Docker service flaps with `failed to start daemon: Error initializing network controller: ... iptables failed` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** iptables binary/alternatives mismatch (nft vs legacy), a firewall tool flushing chains, or missing kernel modules.

**Diagnose →**
```bash
iptables --version
update-alternatives --display iptables
journalctl -u docker | grep -i iptables
```

**Fix →** Align iptables backend (`update-alternatives --set iptables /usr/sbin/iptables-legacy` on older Docker, or use nft consistently on newer), reload modules, restart docker after firewalld.

**Prevent →** Restart Docker after firewall reloads; use `DOCKER-USER` for rules.

</details>

### Q7. `docker: Error response from daemon: Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io: no such host` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS failure on the host, proxy needed, or offline.

**Diagnose →**
```bash
getent hosts registry-1.docker.io
resolvectl status
curl -I https://registry-1.docker.io/v2/
```

**Fix →** Repair host DNS, or configure the daemon proxy via systemd drop-in (`HTTPS_PROXY`), then `systemctl daemon-reload && systemctl restart docker`.

**Prevent →** Use an internal mirror.

</details>

### Q8. After upgrade: `Error response from daemon: client version 1.43 is too new. Maximum supported API version is 1.41` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** CLI newer than a remote daemon.

**Diagnose →** `docker version`

**Fix →** Upgrade the daemon, or temporarily `export DOCKER_API_VERSION=1.41`.

**Prevent →** Keep CLI and engine versions aligned across hosts.

</details>

### Q9. `Failed to start docker.service: Unit docker.service is masked.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service was masked (manually or by a package/Snap change).

**Diagnose →** `systemctl status docker; systemctl is-enabled docker docker.socket`

**Fix →** `sudo systemctl unmask docker docker.socket && sudo systemctl enable --now docker`.

**Prevent →** Avoid mixing Snap and apt Docker installs.

</details>

### Q10. `docker` works with sudo but not in a systemd service running as a different user. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service user not in the docker group; supplementary groups not applied, or `ProtectHome`/sandboxing blocks the socket.

**Diagnose →** `systemctl show svc -p User -p SupplementaryGroups`; `sudo -u svcuser docker ps`.

**Fix →** Add `SupplementaryGroups=docker` to the unit, restart it.

**Prevent →** Use rootless Docker for service users where practical.

</details>

### Q11. Docker Desktop on Windows: `Docker Desktop - WSL kernel version too low` or WSL 2 not starting. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Outdated WSL kernel, virtualization disabled in BIOS, or Hyper-V/VirtualMachinePlatform features off.

**Diagnose →** `wsl --status`, `wsl --version`, Task Manager shows virtualization state.

**Fix →** `wsl --update`, enable "Virtual Machine Platform" and "Windows Subsystem for Linux", enable virtualization in BIOS, reboot.

**Prevent →** Keep WSL updated with Windows updates.

</details>

### Q12. `docker: Got permission denied` when running `docker compose` in CI under a non-root runner. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Runner user lacks socket access.

**Diagnose →** `id`, `ls -l /var/run/docker.sock`.

**Fix →** Add the runner user to `docker` group and restart the runner service (groups load at login/start).

**Prevent →** Use ephemeral runners with a configured image.

</details>

### Q13. Docker daemon takes 5+ minutes to start with many containers. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Huge number of containers/images/log files, slow disk, `live-restore` off with many containers to restore, corrupted metadata.

**Diagnose →**
```bash
journalctl -u docker | grep -E 'Loading containers|Restoring'
docker system df
ls /var/lib/docker/containers | wc -l
```

**Fix →** Remove dead containers, prune, move to faster disk, enable `live-restore`.

**Prevent →** Garbage-collect regularly and cap container churn.

</details>

### Q14. `docker ps` hangs forever. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** dockerd or containerd deadlocked, a stuck container/shim, or exhausted resources (PIDs, FDs, disk).

**Diagnose →**
```bash
timeout 10 docker version
systemctl status containerd docker
kill -SIGUSR1 $(pidof dockerd); journalctl -u docker | tail   # goroutine dump
df -h; ps aux | grep -c defunct
```

**Fix →** Restart containerd then docker; kill the stuck shim; reboot if kernel is wedged (D-state processes).

**Prevent →** Alert on daemon latency and node pressure.

</details>

### Q15. `Error response from daemon: driver failed programming external connectivity on endpoint ... iptables: No chain/target/match by that name` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The `DOCKER` iptables chain was flushed (firewalld/ufw reload, `iptables -F`).

**Diagnose →** `iptables -t nat -L DOCKER -n` (chain missing).

**Fix →** `sudo systemctl restart docker` to recreate chains.

**Prevent →** Do not flush iptables on Docker hosts; reload firewalld then restart Docker, or put custom rules in `DOCKER-USER`.

</details>

### Q16. `docker-compose: command not found` but `docker compose` is expected. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Compose v1 (`docker-compose`) is removed; v2 is a CLI plugin invoked as `docker compose`.

**Diagnose →** `docker compose version`; `ls /usr/libexec/docker/cli-plugins`.

**Fix →** Install `docker-compose-plugin` package. For legacy scripts add a shim `docker-compose` that execs `docker compose`.

**Prevent →** Update scripts to the v2 syntax.

</details>

### Q17. `docker compose up` fails: `unknown shorthand flag: 'd' in -d` or `docker: 'compose' is not a docker command.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Compose plugin missing from this installation (e.g., distro `docker.io` package).

**Diagnose →** `docker --help | grep compose`; `dpkg -l | grep compose`.

**Fix →** Install `docker-compose-plugin` from Docker's repo, or the `docker-compose-v2` package on Ubuntu.

**Prevent →** Install from the official Docker apt/yum repo.

</details>

### Q18. `Error response from daemon: OCI runtime create failed: runc create failed: unable to start container process: ... permission denied` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Entrypoint file not executable, AppArmor/SELinux denial, `noexec` mount on the path, or a wrong `USER` lacking access.

**Diagnose →**
```bash
docker run --rm --entrypoint ls image -l /entrypoint.sh
dmesg | grep -i -E 'apparmor|denied'
mount | grep noexec
```

**Fix →** `chmod +x` (or `COPY --chmod=0755`), use a path not on a noexec mount, fix the MAC policy/labels.

**Prevent →** Test images in an environment matching production security settings.

</details>

### Q19. `docker version` shows server `Docker Desktop` but a script expects Linux engine features such as `host` networking. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker Desktop runs the engine in a VM; host networking and bind mount paths refer to the VM, not the laptop.

**Diagnose →** `docker info | grep -E 'Operating System|Context'`; `docker context ls`.

**Fix →** Enable "host networking" in Desktop settings (supported on recent versions) or publish ports; use `host.docker.internal`.

**Prevent →** Document Desktop vs Linux differences and test in CI on Linux.

</details>

### Q20. `Error: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong context path or `-f` path, filename case, or `.dockerignore` excluding the Dockerfile.

**Diagnose →** `ls -la`; `docker build -f path/Dockerfile path/`.

**Fix →** Pass correct context and `-f`; the last arg is the context, not the Dockerfile.

**Prevent →** Use `docker buildx bake` or Compose `build.context`/`dockerfile` fields.

</details>

### Q21. After reboot, containers don't come back though `--restart always` was set. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** docker service not enabled at boot, containers were created before the policy was set, or they were stopped manually with `unless-stopped`.

**Diagnose →**
```bash
systemctl is-enabled docker
docker inspect -f '{{.HostConfig.RestartPolicy}}' c
journalctl -u docker -b | grep -i restart
```

**Fix →** `systemctl enable docker`; `docker update --restart unless-stopped c`.

**Prevent →** Define policies in Compose files and test a reboot.

</details>

### Q22. `Error response from daemon: Conflict. The container name "/web" is already in use by container ...` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** An old (possibly stopped) container has that name.

**Diagnose →** `docker ps -a --filter name=web`

**Fix →** `docker rm -f web` or choose another name; use `--rm` for ephemeral containers.

**Prevent →** Let Compose name containers or use `--rm`.

</details>

### Q23. `docker login` prints `Error saving credentials: error storing credentials - err: exit status 1, out: ... The name org.freedesktop.secrets was not provided` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `credsStore` set to `secretservice` or `desktop` in `~/.docker/config.json` but no keyring/secret service is running (headless server).

**Diagnose →** `cat ~/.docker/config.json`

**Fix →** Install and use `pass` with `docker-credential-pass`, or remove `credsStore` to store base64 creds (less secure), or use a credential helper for your cloud (`ecr-login`).

**Prevent →** Use short-lived tokens/OIDC in CI.

</details>

### Q24. `docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: ... cgroup` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** cgroup driver mismatch (cgroupfs vs systemd), cgroup v2 not delegated, or rootless without delegation.

**Diagnose →**
```bash
docker info | grep -i 'cgroup'
cat /sys/fs/cgroup/cgroup.controllers
journalctl -u containerd | tail
```

**Fix →** Align on `systemd` cgroup driver (`"exec-opts": ["native.cgroupdriver=systemd"]`), enable delegation for rootless (`/etc/systemd/system/user@.service.d/delegate.conf`).

**Prevent →** Use consistent runtime configuration across Kubernetes/Docker nodes.

</details>

### Q25. `docker context use remote` then `Cannot connect to the Docker daemon at ssh://...` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SSH key/host-key problems, remote user lacks docker access, or `docker` binary absent on the remote PATH for non-interactive shells.

**Diagnose →**
```bash
ssh user@host docker ps
docker --context remote info
ssh -v user@host true
```

**Fix →** Add the key to ssh-agent, add the user to docker group, ensure PATH includes docker in `~/.bashrc` (non-interactive part).

**Prevent →** Use `ControlMaster` in ssh config for speed and reliability.

</details>

---

## Build failures and cache problems

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. `failed to solve: process "/bin/sh -c apt-get install -y curl" did not complete successfully: exit code: 100` with `E: Unable to locate package curl` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Package index missing because `apt-get update` is in a separate cached layer or was never run (images ship without lists).

**Diagnose →** Re-run with `docker build --progress=plain --no-cache .`.

**Fix →** `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*`.

**Prevent →** Always chain update and install.

</details>

### Q27. `E: Failed to fetch http://deb.debian.org/...  404  Not Found` during build of an old image. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Base release is EOL and moved to archive (e.g., Debian buster, Ubuntu EOL), or a stale cached layer.

**Diagnose →** `cat /etc/os-release` in the image; check the release EOL date.

**Fix →** Upgrade base (`bookworm`), or point sources to `archive.debian.org` as a stopgap.

**Prevent →** Track base EOL dates; use Renovate.

</details>

### Q28. `COPY failed: file not found in build context or excluded by .dockerignore: stat app.jar: file does not exist` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File not built yet, path relative to wrong context, or `.dockerignore` excludes it (`*.jar`, `target`).

**Diagnose →**
```bash
cat .dockerignore
docker build --progress=plain .
ls target/
```

**Fix →** Build artifact first or use multi-stage, fix path, add `!target/app.jar` exception.

**Prevent →** Use multi-stage builds so artifacts come from a prior stage.

</details>

### Q29. `ERROR: failed to solve: failed to compute cache key: "/requirements.txt" not found: not found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The file isn't in the context root, a `.dockerignore` pattern, or building from the wrong directory.

**Diagnose →** `docker build -f services/api/Dockerfile services/api` vs `.`

**Fix →** Align the context with the `COPY` paths.

**Prevent →** Document the build context in the Makefile or Compose file.

</details>

### Q30. Build hangs at `RUN pip install ...` or `npm install` and eventually times out. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No network from build (proxy, DNS), MTU mismatch on VPN or cloud overlay, or registry outage.

**Diagnose →**
```bash
docker run --rm busybox wget -qO- https://pypi.org/simple/ | head
docker run --rm busybox ping -c1 8.8.8.8
ip link show docker0 | grep mtu
```

**Fix →** Set proxies; lower MTU in `daemon.json` (`"mtu": 1400`); set DNS servers.

**Prevent →** Use an internal package mirror.

</details>

### Q31. `failed to solve: rpc error: code = Unknown desc = failed to solve with frontend dockerfile.v0: failed to create LLB definition: dockerfile parse error on line 12: unknown instruction: RUNN` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo, or a missing line continuation backslash making the next line an instruction.

**Diagnose →** Check the stated line; `hadolint Dockerfile`.

**Fix →** Correct the instruction; ensure `\` at end of continued lines with no trailing spaces after it.

**Prevent →** Lint in CI and use an editor with Dockerfile support.

</details>

### Q32. Build works locally but fails in CI with `exec format error` in a `RUN` step. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Architecture mismatch (arm64 base layers on amd64 runner or vice versa), or QEMU binfmt not registered for cross-platform builds.

**Diagnose →**
```bash
docker image inspect base --format '{{.Architecture}}'
uname -m
docker buildx ls
```

**Fix →** Specify `--platform`, install QEMU: `docker run --privileged --rm tonistiigi/binfmt --install all`.

**Prevent →** Build multi-arch explicitly and test on both arches.

</details>

### Q33. Layer cache is never used though nothing changed (every step rebuilds). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Fresh CI runner, `--no-cache`, `ARG` with changing value (e.g. `BUILD_DATE`) declared early, `COPY . .` including changing files, or builder pruned.

**Diagnose →** `docker buildx build --progress=plain .`; diff `docker history` of two builds.

**Fix →** Move volatile `ARG`s to the end, ignore generated files, add registry cache.

**Prevent →** Treat cache hit rate as a metric.

</details>

### Q34. `ERROR: failed to solve: failed to fetch anonymous token: ... 429 Too Many Requests` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker Hub rate limit on anonymous pulls of base images.

**Diagnose →** `curl -s -I -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest` and read `ratelimit-remaining`.

**Fix →** `docker login`, use mirror/pull-through cache, switch to registry mirror images.

**Prevent →** Mirror base images internally.

</details>

### Q35. `failed to solve: failed to read expected number of bytes: unexpected EOF` or `no space left on device` during build. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Disk full on the builder, huge build context, or a flaky network.

**Diagnose →**
```bash
df -h /var/lib/docker
docker buildx du
docker system df
```

**Fix →** `docker builder prune -af --keep-storage 20GB`, `docker image prune -a`, enlarge disk.

**Prevent →** Set BuildKit GC limits and monitor disk.

</details>

### Q36. `ARG` value is empty during `RUN`: `echo $VERSION` prints nothing. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `ARG` declared before `FROM` (out of stage scope), declared in a previous stage, or not passed with `--build-arg`.

**Diagnose →** Read the Dockerfile; run with `--progress=plain`.

**Fix →** Redeclare `ARG VERSION` after `FROM` in the stage that needs it; pass `--build-arg VERSION=1.2`.

**Prevent →** Keep ARG declarations next to usage.

</details>

### Q37. `unable to prepare context: path "." not found` or build context transfer is 3 GB and takes forever. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Context includes `node_modules`, `.git`, datasets, or you ran the build in the home directory.

**Diagnose →** `du -sh * | sort -h | tail`; BuildKit prints `transferring context: 3.1GB`.

**Fix →** Build from the project directory, add `.dockerignore`.

**Prevent →** Allow-list style ignore: `*` then `!src`.

</details>

### Q38. Multi-stage build error: `COPY --from=builder /app/dist ...: not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong path in builder, build step silently output elsewhere, stage name typo, or different WORKDIR.

**Diagnose →**
```bash
docker build --target builder -t dbg .
docker run --rm dbg find / -name dist -maxdepth 4
```

**Fix →** Correct the path or `WORKDIR`; fail the build if artifacts are missing (`RUN test -f /app/dist/index.js`).

**Prevent →** Add artifact checks after build steps.

</details>

### Q39. `The command '/bin/sh -c ./gradlew build' returned a non-zero code: 126` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Script lacks execute bit (common when copied from Windows), or CRLF line endings (`/bin/sh^M: bad interpreter`).

**Diagnose →** `ls -l gradlew`; `file gradlew`; `head -1 gradlew | od -c`.

**Fix →** `chmod +x gradlew` (or `COPY --chmod=755`), `dos2unix`, set `.gitattributes` `* text=auto eol=lf` for shell scripts.

**Prevent →** `.gitattributes` for `*.sh text eol=lf`.

</details>

### Q40. `standard_init_linux.go: exec user process caused: no such file or directory` for an existing binary. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing dynamic loader or interpreter: glibc binary on Alpine/scratch, or shebang `#!/bin/bash` where bash does not exist, or CRLF in shebang.

**Diagnose →**
```bash
file app; ldd app
docker run --rm --entrypoint sh image -c 'head -1 /entrypoint.sh'
```

**Fix →** Build statically (`CGO_ENABLED=0`), use a matching base (distroless base/cc), install bash or use `#!/bin/sh`.

**Prevent →** Test the final image, not just the build stage.

</details>

### Q41. Build secret mount fails: `failed to solve: secret "npmrc" not found` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--secret id=npmrc,src=...` not passed, ID mismatch, or running with legacy builder.

**Diagnose →** Compare `id=` in the Dockerfile and CLI; check `DOCKER_BUILDKIT`.

**Fix →** `docker build --secret id=npmrc,src=$HOME/.npmrc .`; in Compose define `build.secrets`.

**Prevent →** Add a header `# syntax=docker/dockerfile:1` and use BuildKit everywhere.

</details>

### Q42. `docker build` succeeds, but the image runs old code. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The container was not recreated, a different tag is running, cache reused a stale `COPY` layer (rare), or Compose `image:` pulls an older tag.

**Diagnose →**
```bash
docker images --digests app
docker inspect -f '{{.Image}}' running_container
```

**Fix →** `docker compose up -d --build --force-recreate`; tag explicitly.

**Prevent →** Deploy by digest/immutable tags.

</details>

### Q43. `ERROR: failed to solve: dockerfile.v0: failed to resolve source metadata for docker.io/library/node:22-alpine: ... x509: certificate signed by unknown authority` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Corporate TLS inspection proxy; the CA isn't trusted by the daemon/BuildKit.

**Diagnose →** `openssl s_client -connect registry-1.docker.io:443 -showcerts`

**Fix →** Install the corporate CA into the host trust store (`update-ca-certificates`) and restart docker; for docker-container buildx driver pass `--config buildkitd.toml` with CA; inside images add CA for `RUN` downloads.

**Prevent →** Distribute CA through config management.

</details>

### Q44. `buildx` error: `ERROR: multiple platforms feature is currently not supported for docker driver` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default `docker` driver without containerd image store cannot build multi-platform images.

**Diagnose →** `docker buildx ls`

**Fix →** `docker buildx create --use --driver docker-container`, or enable the containerd image store in Docker Desktop/Engine settings.

**Prevent →** CI uses a dedicated builder.

</details>

### Q45. `--load` with multiple platforms fails: `docker exporter does not currently support exporting manifest lists` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Local classic image store can hold only one platform.

**Diagnose →** `docker buildx build --platform linux/amd64,linux/arm64 --load .` triggers the error.

**Fix →** Use `--push`, or `--output type=oci,dest=app.tar`, or build one platform with `--load`; or enable containerd image store.

**Prevent →** Test per-platform locally, push multi-arch in CI.

</details>

### Q46. `npm ERR! code ELIFECYCLE` building native modules on Alpine (`gyp ERR! find Python`). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alpine lacks python3, make, g++ needed by node-gyp.

**Diagnose →** Read the full npm log with `--progress=plain`.

**Fix →** `apk add --no-cache python3 make g++` in the build stage only, or switch to `node:22-slim`.

**Prevent →** Do native compiles in a builder stage, copy artifacts.

</details>

### Q47. `HEALTHCHECK` is missing from an image built with Podman/Buildah. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Buildah defaults to OCI format which does not support HEALTHCHECK.

**Diagnose →** `podman inspect image | jq '.[0].Config.Healthcheck'`

**Fix →** `podman build --format docker .` or define the check at runtime.

**Prevent →** Standardise on `--format docker` where health checks matter.

</details>

### Q48. Dockerfile `RUN --mount=type=cache` is ignored/errors: `the --mount option requires BuildKit` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Legacy builder in use (old Docker, `DOCKER_BUILDKIT=0`).

**Diagnose →** `docker version`; `echo $DOCKER_BUILDKIT`.

**Fix →** Upgrade to Docker 23+ (BuildKit default) or `DOCKER_BUILDKIT=1`; add the `# syntax=` line.

**Prevent →** Pin builder tooling in CI.

</details>

### Q49. `Error: buildx failed with: ERROR: failed to solve: failed to push registry/app:tag: unexpected status: 401 Unauthorized` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not logged in as the user running the build (credentials come from the client's `~/.docker/config.json`), token lacks push scope, or the repository does not exist.

**Diagnose →** `docker login reg`; `cat ~/.docker/config.json`; check token scopes.

**Fix →** Log in before building, grant `write` permission, create the repository (ECR).

**Prevent →** Use OIDC with least-privilege push roles.

</details>

### Q50. `--cache-from type=gha` shows no cache hits in GitHub Actions. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Cache scope differs per workflow/branch, cache evicted (10 GB repo limit), `mode=max` not set, or the Dockerfile changes early layers, or the cache backend isn't configured because the Actions runtime token isn't exposed to the step.

**Diagnose →** Look for `importing cache manifest from gha` in logs; check the Actions cache UI.

**Fix →** Use `scope=`, `mode=max`, use `docker/build-push-action` with `docker/setup-buildx-action`, or switch to registry cache.

**Prevent →** Keep a stable early-layer structure.

</details>

---

## Containers that exit, crash or misbehave

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. Container exits immediately with code 0 right after `docker run -d`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The main process finished (a shell with no TTY, a script that ends, a daemon that forks into the background).

**Diagnose →**
```bash
docker ps -a
docker logs c
docker inspect -f '{{.Config.Cmd}} {{.Config.Entrypoint}}' c
```

**Fix →** Run the service in the foreground (`nginx -g 'daemon off;'`), or use `-it` for shells. A container lives as long as PID 1.

**Prevent →** Use images whose CMD is a foreground process.

</details>

### Q52. `Exited (137)` with no OOM flag. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** SIGKILL: `docker kill`, a stop timeout expiry, host OOM killer (outside cgroup limit), or an orchestrator eviction.

**Diagnose →**
```bash
docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' c
dmesg -T | grep -i -E 'killed process|oom'
docker events --since 1h --filter container=c
```

**Fix →** If host OOM, add memory or limits; if timeout, handle SIGTERM and raise `--stop-timeout`.

**Prevent →** Set memory limits so the kernel kills inside the cgroup, not randomly on the host.

</details>

### Q53. `Exited (139)` segmentation fault. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Binary built for different libc (glibc vs musl) or CPU features, native module crash, emulated amd64 on arm64 running a JIT, or a genuine app bug.

**Diagnose →**
```bash
docker run --rm --entrypoint sh image -c 'uname -m; ldd /app/bin'
docker run --platform linux/amd64 ...
dmesg | grep segfault
```

**Fix →** Match base and architecture, rebuild native modules, enable core dumps (`--ulimit core=-1`) and analyze with gdb.

**Prevent →** Test images on the real target architecture.

</details>

### Q54. `Exited (126)` / `Exited (127)`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** 126: command found but not executable; 127: command not found (or missing shebang interpreter).

**Diagnose →** `docker run --rm --entrypoint sh image -c 'ls -l /app; which myapp'`

**Fix →** `chmod +x`, correct the `PATH`/absolute path, install the missing binary, fix CRLF.

**Prevent →** Use exec form with absolute paths and add a smoke test.

</details>

### Q55. `exec /entrypoint.sh: exec format error` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Script lacks a shebang, or image architecture differs from host (arm64 image on amd64).

**Diagnose →**
```bash
docker image inspect image -f '{{.Os}}/{{.Architecture}}'
docker run --rm --entrypoint head image -1 /entrypoint.sh
```

**Fix →** Add `#!/bin/sh`, pull the right `--platform`, build multi-arch.

**Prevent →** Publish multi-arch manifests.

</details>

### Q56. `docker run` says `WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)` and the app is very slow. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Running amd64 image under emulation on Apple Silicon.

**Diagnose →** `docker image inspect img -f '{{.Architecture}}'`; `docker info | grep Architecture`.

**Fix →** Pull a native arm64 variant (`--platform linux/arm64`), rebuild multi-arch, or enable Rosetta in Docker Desktop.

**Prevent →** Ship multi-arch images.

</details>

### Q57. App crashes with `Error: listen EADDRINUSE: address already in use :::3000` inside the container. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Two processes in the container on the same port (e.g. nodemon plus a second start), or a restarted process before the old one released the port.

**Diagnose →** `docker exec c ss -tlnp` (or `netstat`), `docker top c`.

**Fix →** Run a single process, fix the dev script, use `--init` to reap.

**Prevent →** One process per container.

</details>

### Q58. Container works with `docker run -it` but fails when started detached with `-d`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The app expects a TTY or stdin (interactive prompts), or reads from stdin and gets EOF.

**Diagnose →** `docker logs c`; compare with `docker run -d -t`.

**Fix →** Provide config non-interactively, use `-t`/`-i` only if truly needed, or `tail -f /dev/null` for dev placeholder (not prod).

**Prevent →** Make apps non-interactive.

</details>

### Q59. `docker exec` returns `OCI runtime exec failed: exec failed: unable to start container process: exec: "bash": executable file not found in $PATH` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Alpine/distroless/slim image without bash.

**Diagnose →** `docker exec c sh -c 'cat /etc/os-release'`

**Fix →** Use `sh`, or `docker debug c`, or attach a debug container sharing namespaces.

**Prevent →** Do not depend on bash in minimal images.

</details>

### Q60. `docker exec` fails with `cannot exec in a stopped container` or `container is not running`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Container crashed or exited.

**Diagnose →** `docker ps -a`, `docker logs c`, `docker inspect -f '{{.State.Error}}' c`.

**Fix →** Find out why it exited; to inspect its filesystem use `docker cp c:/path .` or `docker run --rm -it --entrypoint sh image`.

**Prevent →** Alert on restart count.

</details>

### Q61. Environment variable set with `-e` but the app doesn't see it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Flag placed after the image name (treated as command args), typo, entrypoint script sanitises env, app reads a config file instead.

**Diagnose →**
```bash
docker exec c env | sort
docker inspect -f '{{.Config.Env}}' c
```

**Fix →** `docker run -e FOO=bar image` (options before image). Check how the app reads variables.

**Prevent →** Print effective config on startup (without secrets).

</details>

### Q62. `docker run` argument is ignored: `docker run image --port 9000` fails with `unknown flag`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Args after the image replace `CMD` entirely (and are appended to `ENTRYPOINT`).

**Diagnose →** `docker inspect -f '{{.Config.Entrypoint}} {{.Config.Cmd}}' image`

**Fix →** Pass the full command (`image app --port 9000`) when only `CMD` is defined; with ENTRYPOINT pass only the extra args; override with `--entrypoint`.

**Prevent →** Document the entrypoint contract.

</details>

### Q63. PID 1 app doesn't reload on SIGHUP or ignores Ctrl+C in `docker run -it`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** PID 1 has no default signal handlers; a signal without a handler is ignored.

**Diagnose →** `docker exec c cat /proc/1/status | grep -E 'SigCgt|SigIgn'`

**Fix →** Run `--init`, or have the app install handlers (Node: `process.on('SIGTERM')`).

**Prevent →** Test `docker stop` time in CI.

</details>

### Q64. Container shows `Up` but the app returns 502 / connection refused from the host on published port. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App listens on 127.0.0.1 inside the container instead of 0.0.0.0, wrong container port, or the app hasn't finished booting.

**Diagnose →**
```bash
docker exec c ss -tlnp
docker port c
curl -v localhost:8080
```

**Fix →** Bind the app to `0.0.0.0` (e.g. `flask run --host 0.0.0.0`) and publish the right port.

**Prevent →** Add a health check hitting the published path.

</details>

### Q65. `standard_init_linux.go:228: exec user process caused: exec format error` on a Raspberry Pi. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image is amd64 only.

**Diagnose →** `uname -m`; `docker manifest inspect image | jq '.manifests[].platform'`.

**Fix →** Use an image with `linux/arm64` or `linux/arm/v7`, or rebuild with buildx.

**Prevent →** Publish multi-arch images.

</details>

### Q66. Container time/zone is wrong in logs. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Containers default to UTC and slim images lack tzdata.

**Diagnose →** `docker exec c date; docker exec c ls /usr/share/zoneinfo | head`.

**Fix →** Install `tzdata` and set `TZ`, or mount `/etc/localtime:ro`.

**Prevent →** Log in UTC and convert at display time.

</details>

### Q67. `Error: Cannot find module` in container but works on the host. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `node_modules` excluded by `.dockerignore` but `npm install` not run, a bind mount shadowing `/app/node_modules`, or platform-specific native modules built on the host.

**Diagnose →**
```bash
docker run --rm image ls /app/node_modules | head
docker inspect -f '{{json .Mounts}}' c
```

**Fix →** Install dependencies in the image, mask `node_modules` with a named volume when bind mounting, never copy host `node_modules`.

**Prevent →** `.dockerignore` with `node_modules` and `npm ci` in the Dockerfile.

</details>

### Q68. Python app prints nothing to `docker logs` until the container stops. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** stdout buffering when not attached to a TTY.

**Diagnose →** Compare `docker run -t` vs `-d`.

**Fix →** `ENV PYTHONUNBUFFERED=1` or `python -u`; configure logging handlers to stream.

**Prevent →** Set the variable in the base image.

</details>

### Q69. `OCI runtime create failed: ... mount: /proc: permission denied` or similar from nested containers. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Running containers inside a restricted container/LXC/CI sandbox without needed capabilities or with masked `/proc`.

**Diagnose →** `cat /proc/self/status | grep Cap`; check the outer environment's security profile.

**Fix →** Use a runtime that supports nesting (Sysbox), enable `nesting` in LXC, or use rootless with proper setup.

**Prevent →** Prefer daemonless/remote builds over nested runtimes.

</details>

### Q70. Container cannot write to a mounted directory: `Permission denied` / `EACCES`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** UID mismatch between container user and host directory owner, read-only mount (`:ro`), SELinux label missing, or read-only root filesystem.

**Diagnose →**
```bash
docker exec c id
ls -ln /host/dir
docker inspect -f '{{json .Mounts}}' c
ls -Z /host/dir   # SELinux
```

**Fix →** `chown` to the container UID, run with `--user`, add `:z`/`:Z`, drop `:ro`.

**Prevent →** Define fixed numeric UIDs.

</details>

### Q71. Application can't resolve its own hostname or `sudo: unable to resolve host`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Custom `--hostname` or `--network host` produce mismatched `/etc/hosts`; or `/etc/hosts` overwritten.

**Diagnose →** `docker exec c cat /etc/hosts; hostname`

**Fix →** Use `--add-host name:ip`, don't edit `/etc/hosts` manually (it's regenerated on start).

**Prevent →** Rely on DNS names, not hosts files.

</details>

### Q72. After `docker stop` a container takes exactly 10 seconds every time. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Process ignores SIGTERM (shell form entrypoint, PID 1 without handler).

**Diagnose →** `docker inspect -f '{{.Config.Entrypoint}} {{.Config.Cmd}}' c`; `docker exec c ps`.

**Fix →** Exec form, `exec "$@"` in scripts, `--init`, or `docker stop -t`.

**Prevent →** Measure stop time in smoke tests.

</details>

### Q73. Debug a container that crashes before you can `exec` into it. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Startup error.

**Diagnose →**
```bash
docker logs --tail 200 c
docker run --rm -it --entrypoint sh image
docker commit c debug-img && docker run --rm -it --entrypoint sh debug-img
docker run --rm -it --volumes-from c --entrypoint sh image
```

**Fix →** Reproduce with the same env and mounts, run the entrypoint manually (`sh -x /entrypoint.sh`).

**Prevent →** Make entrypoints verbose on error and log configuration validation.

</details>

### Q74. `docker cp` fails: `Error: No such container:path` or copies nothing from a volume path. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong path, the path is a `tmpfs` mount (not supported by `docker cp`), or the container doesn't exist.

**Diagnose →** `docker exec c ls -l /path`; `docker inspect -f '{{json .Mounts}}' c`.

**Fix →** Copy from a regular path, or use a helper container with `--volumes-from`; use `docker exec c tar c ... | tar x`.

**Prevent →** Know which paths are tmpfs.

</details>

### Q75. `unable to find image 'myapp:latest' locally` followed by `pull access denied for myapp, repository does not exist or may require 'docker login'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image was never built locally (or built with another tag/context), or the name lacks a registry prefix and Docker tries Docker Hub.

**Diagnose →** `docker images | grep myapp`; `docker context ls` (maybe building on a different daemon).

**Fix →** Build with `-t myapp:latest` against the same daemon, or use the full name `reg.example.com/team/myapp:latest`.

**Prevent →** Consistent naming and `DOCKER_HOST` awareness.

</details>

---

## Networking, DNS and Compose

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. `curl: (7) Failed to connect to localhost port 8080: Connection refused` after `docker run -p 8080:80 nginx`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Container not running, mapping reversed (`-p 80:8080`), app listening on another port or only on loopback, or the request is from another host while docker is remote.

**Diagnose →**
```bash
docker ps
docker port web
docker exec web ss -tlnp
ss -tlnp | grep 8080
```

**Fix →** Use `-p hostPort:containerPort` correctly, bind `0.0.0.0` in the app.

**Prevent →** Verify with `docker port` after starting.

</details>

### Q77. `Bind for 0.0.0.0:5432 failed: port is already allocated` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Another container or host process uses that port.

**Diagnose →**
```bash
sudo ss -tlnp | grep 5432
docker ps --format '{{.Names}} {{.Ports}}' | grep 5432
```

**Fix →** Stop the other process, or map another host port (`-p 5433:5432`), or drop the publish.

**Prevent →** Don't publish DB ports; use internal networks.

</details>

### Q78. Containers cannot reach the internet: `Temporary failure in name resolution` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** DNS config (host uses 127.0.0.53 stub), IP forwarding off, firewall blocking forward, or VPN conflict.

**Diagnose →**
```bash
docker run --rm busybox nslookup google.com
docker run --rm busybox ping -c1 1.1.1.1
sysctl net.ipv4.ip_forward
iptables -t nat -L POSTROUTING -n | grep MASQ
```

**Fix →** Set `"dns": ["1.1.1.1"]` or corp DNS in `daemon.json`; enable `net.ipv4.ip_forward=1`; restart docker.

**Prevent →** Standardize host DNS and forwarding sysctls.

</details>

### Q79. `ping` works to IPs but DNS names fail on the default bridge: `ping: bad address 'db'` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default bridge has no name-based discovery.

**Diagnose →** `docker network inspect bridge`.

**Fix →** Use a user-defined network or Compose.

**Prevent →** Never rely on the default bridge.

</details>

### Q80. `getaddrinfo ENOTFOUND db` in a Compose app. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service name typo, services on different networks, or the DB container exited.

**Diagnose →**
```bash
docker compose ps
docker compose exec api getent hosts db
docker network inspect <project>_default
```

**Fix →** Use the exact service name, put services on a shared network, fix the DB crash.

**Prevent →** Define networks explicitly and add health-gated `depends_on`.

</details>

### Q81. Compose: `service "api" depends on undefined service "db"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Service missing, defined under a profile that isn't active, or in an un-merged override file.

**Diagnose →** `docker compose config --services`; `docker compose --profile x config`.

**Fix →** Define the service or activate the profile; pass all `-f` files.

**Prevent →** Run `docker compose config` in CI.

</details>

### Q82. Compose: `yaml: line 14: mapping values are not allowed in this context` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Indentation/tab error, missing space after colon, or unquoted special characters (`:` in values, `@`, `*`).

**Diagnose →** `docker compose config`; `yamllint compose.yaml`.

**Fix →** Use spaces only, quote strings like `"8080:80"`, `"${VAR:-x}"`.

**Prevent →** Editor YAML linting.

</details>

### Q83. Compose: `The "DB_PASSWORD" variable is not set. Defaulting to a blank string.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `.env` missing/not in the project dir, variable only exported in a different shell, or typo.

**Diagnose →** `docker compose config | grep -i password`; `ls -la .env`.

**Fix →** Create `.env` next to compose file or use `--env-file`; require it with `${DB_PASSWORD:?missing}`.

**Prevent →** Fail fast using `:?` syntax.

</details>

### Q84. `Error response from daemon: network xxx not found` on `docker compose up`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A network was removed outside Compose while containers still reference it.

**Diagnose →** `docker network ls`; `docker compose ps -a`.

**Fix →** `docker compose down` then `up -d` (or `up --force-recreate`).

**Prevent →** Don't delete networks manually; use `docker compose down`.

</details>

### Q85. `Error response from daemon: Pool overlaps with other one on this address space` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A custom subnet collides with an existing network or host route.

**Diagnose →**
```bash
docker network ls -q | xargs docker network inspect -f '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
ip route
```

**Fix →** Choose a free subnet, remove unused networks, set `default-address-pools`.

**Prevent →** Central IP plan for Docker ranges.

</details>

### Q86. `could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Too many networks (each Compose project takes a /16 by default) have exhausted the default pools.

**Diagnose →** `docker network ls | wc -l`

**Fix →** `docker network prune`; configure smaller pools: `"default-address-pools":[{"base":"10.10.0.0/16","size":24}]`.

**Prevent →** Use /24 pools and clean up CI networks.

</details>

### Q87. Container-to-container requests fail intermittently by name after a container restarts. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Client (nginx, JVM) cached the old IP; DNS TTL and resolver caches ignore the new address.

**Diagnose →** `docker exec web getent hosts api` vs the IP in nginx upstream; `docker network inspect`.

**Fix →** In nginx use `resolver 127.0.0.11 valid=10s;` with variable in `proxy_pass`; JVM `networkaddress.cache.ttl=10`; or reload nginx.

**Prevent →** Configure resolver behaviour when using dynamic backends.

</details>

### Q88. Published port is reachable from the internet though `ufw` denies it. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Docker's DNAT rules bypass ufw's INPUT rules.

**Diagnose →** `iptables -t nat -S DOCKER`; scan with `nmap -p 8080 host`.

**Fix →** Bind `127.0.0.1:8080:80`, or add rules to `DOCKER-USER`, or use `ufw-docker` tooling, or set `"iptables": false` (with caution).

**Prevent →** Publish only what needs to be public.

</details>

### Q89. Access from container to host service fails: `connection refused host.docker.internal` on Linux. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Name not defined on Linux by default, or the host service binds only to 127.0.0.1, or a firewall blocks the bridge interface.

**Diagnose →**
```bash
docker run --rm --add-host host.docker.internal:host-gateway busybox getent hosts host.docker.internal
ss -tlnp | grep <port>
```

**Fix →** Add `extra_hosts`, bind the service to `0.0.0.0` or the bridge IP, allow `docker0` in the firewall.

**Prevent →** Prefer running dependencies as containers.

</details>

### Q90. Slow DNS: every outbound call in the container takes 5 seconds. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** AAAA lookups failing or timing out (no IPv6), conntrack race on UDP DNS, unreachable first resolver, `ndots:5` search-list expansion.

**Diagnose →**
```bash
docker exec c cat /etc/resolv.conf
docker exec c sh -c 'time getent hosts example.com'
tcpdump -ni any port 53
```

**Fix →** Remove dead resolvers, use `options single-request-reopen`, set `--dns-opt ndots:1`, use fully qualified names with trailing dot, disable IPv6 where unused.

**Prevent →** Run a local caching resolver.

</details>

### Q91. `Error response from daemon: user specified IP address is supported only when connecting to networks with user configured subnets` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--ip` used on the default bridge.

**Diagnose →** `docker network inspect <net> | jq '.[0].IPAM'`

**Fix →** Create a network with `--subnet` and attach to it.

**Prevent →** Avoid static IPs.

</details>

### Q92. Inter-host overlay network: containers on different nodes cannot ping. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Firewall blocks 4789/udp (VXLAN), 7946, 2377; MTU too high; clock skew; cloud security groups.

**Diagnose →**
```bash
docker node ls
nc -zvu node2 4789
ss -ulnp | grep 4789
docker network inspect <overlay>
```

**Fix →** Open ports between nodes, lower overlay MTU (`--opt com.docker.network.driver.mtu=1450`), sync time.

**Prevent →** Provision security groups via IaC.

</details>

### Q93. `docker compose up` creates new network every time and `docker network ls` is full of them. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Changing project name (`-p`, directory names, `COMPOSE_PROJECT_NAME`) between runs, or CI jobs leaving networks behind.

**Diagnose →** `docker network ls --filter label=com.docker.compose.project`.

**Fix →** Use `docker compose down` in cleanup; `docker network prune`; fix the project name.

**Prevent →** Set `name:` at the top of compose.yaml.

</details>

### Q94. Port mapping works on host but not from another machine on the LAN. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bound to `127.0.0.1`, cloud security group/firewall blocking, wrong IP.

**Diagnose →** `docker port c`; `ss -tlnp`; `curl` from another host; `nc -zv ip port`.

**Fix →** Publish on `0.0.0.0` (or remove the `127.0.0.1` prefix) and open the firewall.

**Prevent →** Document intended exposure.

</details>

### Q95. Websocket / long-lived connections drop through the Docker proxy after a while. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** conntrack/NAT timeouts, idle timeouts at upstream load balancer, missing keepalives.

**Diagnose →** `conntrack -L | grep <port>`; `sysctl net.netfilter.nf_conntrack_tcp_timeout_established`; check LB idle settings.

**Fix →** Enable TCP/app-level keepalives shorter than the timeout, tune idle timeouts, raise conntrack limits.

**Prevent →** Application heartbeat.

</details>

### Q96. `nginx: [emerg] host not found in upstream "api" in /etc/nginx/conf.d/default.conf:3` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** nginx started before `api` was resolvable (not up or different network); nginx resolves upstreams at startup.

**Diagnose →** `docker compose ps`; `docker compose exec nginx getent hosts api`.

**Fix →** Start order with `depends_on`, same network, or use a `resolver 127.0.0.11` plus variable-based `proxy_pass`.

**Prevent →** Health-gated startup.

</details>

### Q97. Connection from container to DB `connection reset by peer` only under load. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Conntrack table full (`nf_conntrack: table full, dropping packet`), DB max connections, ephemeral port exhaustion, idle connection killed by NAT.

**Diagnose →**
```bash
dmesg | grep conntrack
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
ss -s
```

**Fix →** Raise `nf_conntrack_max`, use connection pooling, increase port range, tune keepalive.

**Prevent →** Monitor conntrack usage.

</details>

### Q98. `docker compose up` error: `Bind for 0.0.0.0:80 failed: port is already allocated` while nothing seems to listen. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A stale `docker-proxy` or a stopped container still holding the mapping, or another Compose project.

**Diagnose →**
```bash
docker ps -a --filter publish=80
sudo ss -tlnp | grep ':80 '
ps aux | grep docker-proxy
```

**Fix →** Remove the old container (`docker rm -f`), `docker compose down --remove-orphans`, restart Docker if the proxy is orphaned.

**Prevent →** Use `--remove-orphans`.

</details>

### Q99. `network <name> declared as external, but could not be found` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The external network was never created.

**Diagnose →** `docker network ls | grep <name>`.

**Fix →** `docker network create <name>` first; or remove `external: true` so Compose creates it.

**Prevent →** Create shared networks in a bootstrap script.

</details>

### Q100. Container sees the wrong source IP (always the gateway 172.x) behind published ports. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Userland proxy or masquerade replaces client IP, especially on Docker Desktop or with IPv6 to IPv4 translation.

**Diagnose →** `docker logs web` shows 172.18.0.1; check `"userland-proxy"` setting.

**Fix →** Use `--network host` for the edge proxy, run the reverse proxy on the host, use PROXY protocol or `X-Forwarded-For`, disable `userland-proxy` where acceptable.

**Prevent →** Terminate TLS at a proxy that adds `X-Forwarded-For`.

</details>

---

## Storage, volumes and disk space

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. `no space left on device` while pulling an image or starting a container. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `/var/lib/docker` partition full (images, build cache, logs, volumes) or inodes exhausted.

**Diagnose →**
```bash
df -h /var/lib/docker; df -i /var/lib/docker
docker system df -v
```

**Fix →** `docker system prune -a` (review first), `docker builder prune`, truncate huge logs, expand the disk, or move `data-root`.

**Prevent →** Separate disk, log rotation, scheduled pruning and alerts at 80%.

</details>

### Q102. `df` shows free space but Docker says no space: inodes at 100%. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Millions of small files (node_modules layers, many tiny containers) exhaust inodes.

**Diagnose →** `df -i`; `find /var/lib/docker -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head`.

**Fix →** Prune images/containers, rebuild fs with more inodes (ext4 `-N`) or use XFS.

**Prevent →** Prefer XFS for Docker data and slim images.

</details>

### Q103. `/var/lib/docker/containers/<id>/<id>-json.log` is 50 GB. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `json-file` driver without rotation and a chatty app.

**Diagnose →** `ls -lhS /var/lib/docker/containers/*/*-json.log | head`

**Fix →** Immediately `truncate -s 0 <file>`; set `max-size`/`max-file` or `local` driver in `daemon.json`; recreate containers.

**Prevent →** Global log rotation and less verbose logging.

</details>

### Q104. `docker system prune` did not free much although `docker system df` shows large "Reclaimable". 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Images still referenced by stopped containers or tags, build cache not included, or volumes need `--volumes`.

**Diagnose →** `docker system df -v`; `docker ps -a`.

**Fix →** `docker container prune`, `docker image prune -a`, `docker builder prune -a`, `docker volume prune` (carefully).

**Prevent →** Targeted cleanup jobs per resource type.

</details>

### Q105. Volume data disappeared after `docker compose down -v` or `docker volume prune`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `-v` deletes named volumes; prune removes unused volumes.

**Diagnose →** `docker volume ls`; check shell history; look for backups.

**Fix →** Restore from backup; if none, data is gone (try filesystem snapshot or undelete tools on `/var/lib/docker/volumes`).

**Prevent →** Backups; guard destructive commands; label volumes and use `docker volume prune --filter label!=keep`.

</details>

### Q106. Data is not persisted: after container recreate the database is empty. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong mount path, anonymous volume created (image's `VOLUME`) so a new one appears each time, or Compose project name changed so the volume name differs.

**Diagnose →**
```bash
docker inspect -f '{{json .Mounts}}' db
docker volume ls
```

**Fix →** Mount a named volume at the exact data directory, reuse the same project/volume name (`name:` in compose volume).

**Prevent →** Explicit named volumes with fixed `name:`.

</details>

### Q107. `docker: Error response from daemon: invalid mount config for type "bind": bind source path does not exist` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host path missing (`--mount` does not create it) or typo; on Docker Desktop, path not shared.

**Diagnose →** `ls -ld /host/path`; Docker Desktop file-sharing settings.

**Fix →** Create the directory, fix path, add it to shared paths.

**Prevent →** Use `$PWD` absolute paths and named volumes.

</details>

### Q108. Bind mount of a single file shows stale content after editing it on host. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Editor replaced the file (new inode) so the container still sees the old inode.

**Diagnose →** `stat -c %i file` before/after save; `docker exec c cat file`.

**Fix →** Mount the parent directory, or restart the container, or edit in place (`cat > file`).

**Prevent →** Mount directories rather than single files.

</details>

### Q109. MySQL/Postgres refuses to start: `initdb: error: directory "/var/lib/postgresql/data" exists but is not empty` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Volume root contains `lost+found` (ext4 mount point) or leftover files.

**Diagnose →** `docker run --rm -v pgdata:/d alpine ls -la /d`.

**Fix →** Mount a subdirectory: `-v pgdata:/var/lib/postgresql/data` with `PGDATA=/var/lib/postgresql/data/pgdata`, or clean the directory if it is truly disposable.

**Prevent →** Set `PGDATA` to a subfolder for block-device mounts.

</details>

### Q110. Postgres: `FATAL: data directory "/var/lib/postgresql/data" has wrong ownership` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host bind mount owned by another UID; permissions not 0700.

**Diagnose →** `ls -ldn /host/pgdata`; `docker exec db id postgres`.

**Fix →** `chown -R 999:999 /host/pgdata; chmod 700`, or use a named volume.

**Prevent →** Prefer named volumes for databases.

</details>

### Q111. `docker volume rm` says `volume is in use`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** A running or stopped container still references it.

**Diagnose →** `docker ps -a --filter volume=<name>`

**Fix →** `docker rm -f` those containers, then remove the volume.

**Prevent →** Use `docker compose down` to clean up consistently.

</details>

### Q112. Files created in a bind mount are owned by root on the host. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Container process runs as root.

**Diagnose →** `ls -ln`; `docker exec c id`.

**Fix →** `docker run --user $(id -u):$(id -g)`; set `user:` in Compose; use rootless Docker (maps root to your UID).

**Prevent →** Run non-root.

</details>

### Q113. SELinux: `Permission denied` accessing a bind-mounted directory on RHEL/Fedora. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host directory has no `container_file_t` label.

**Diagnose →** `ls -Z /host/dir`; `ausearch -m avc -ts recent`.

**Fix →** Add `:z` (shared) or `:Z` (private) to the mount, or `chcon -Rt container_file_t /host/dir`. Never mount `/home` or `/` with `:Z`.

**Prevent →** Keep SELinux enforcing and label volumes.

</details>

### Q114. Slow file I/O on bind mounts with Docker Desktop (npm install, composer). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** File sharing between host and VM adds per-file overhead.

**Diagnose →** Compare timing inside container on a named volume vs bind mount.

**Fix →** Use VirtioFS, keep dependencies in named volumes, use Compose `develop.watch` sync.

**Prevent →** Store heavy-write paths in volumes.

</details>

### Q115. `docker save` / `docker load` fails with `unexpected EOF` or `invalid tar header`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Truncated transfer, disk full during save, gzip mismatch.

**Diagnose →** `tar tf image.tar | head`; `sha256sum` on both ends; `df -h`.

**Fix →** Re-save and verify checksums; `docker save img | gzip > img.tgz` and `gunzip -c | docker load`.

**Prevent →** Verify digests after transfer, or use `skopeo copy` with `oci-archive`.

</details>

### Q116. `Error response from daemon: failed to mount overlay: ... invalid argument` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Backing filesystem lacks `d_type` support (XFS with `ftype=0`), unsupported fs (e.g. some NFS, ecryptfs), or overlay kernel module missing.

**Diagnose →**
```bash
docker info | grep -E 'Storage Driver|Backing Filesystem|Supports d_type'
xfs_info /var/lib/docker | grep ftype
lsmod | grep overlay
```

**Fix →** Recreate the filesystem with `ftype=1`, or move `data-root` to ext4/XFS local disk, `modprobe overlay`.

**Prevent →** Provision `/var/lib/docker` with a supported fs.

</details>

### Q117. `docker pull` fails: `failed to register layer: ... no space left on device` though df shows space. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Inodes, a quota (`pquota`/`--storage-opt`), or the layer extraction temp directory (`/var/lib/docker/tmp`) on a different small mount.

**Diagnose →** `df -h /var/lib/docker /var/lib/docker/tmp /tmp`; `df -i`; `docker info`.

**Fix →** Free or expand; set `DOCKER_TMPDIR` on a larger disk.

**Prevent →** Monitor all relevant mounts.

</details>

### Q118. Container writable layer huge: `docker ps -s` shows SIZE 20GB (virtual 21GB). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App writes data/logs/temp files into the container FS.

**Diagnose →** `docker ps -s`; `docker diff c | head`; `docker exec c du -xh / | sort -h | tail`.

**Fix →** Move data to volumes, clean temp files, add log rotation.

**Prevent →** Read-only root FS reveals such writes early.

</details>

### Q119. Docker Desktop VM disk is full and `docker system prune` doesn't shrink the disk image. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The virtual disk (`Docker.raw`/VHDX) grows but doesn't automatically shrink.

**Diagnose →** Desktop Settings > Resources shows usage; `docker system df`.

**Fix →** Prune, then use Desktop's "Clean / Purge data", reduce the disk limit, or compact the VHDX (`Optimize-VHD`) on Windows.

**Prevent →** Set a disk image size limit and prune regularly.

</details>

### Q120. Moving `/var/lib/docker` to a new disk broke everything: `Error starting daemon: ... mkdir ... permission denied`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Copy without preserving ownership/xattrs/SELinux labels, or wrong `data-root` path.

**Diagnose →** `journalctl -u docker`; `ls -ld /newpath`; `getfattr`.

**Fix →** `systemctl stop docker`; `rsync -aHAXS /var/lib/docker/ /data/docker/`; set `"data-root": "/data/docker"`; restart.

**Prevent →** Use `rsync -aHAXS` and test on a staging host.

</details>

### Q121. Volume shared via NFS: `mount.nfs: access denied by server while mounting`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Export doesn't allow the client IP, `root_squash`, wrong NFS version/path.

**Diagnose →**
```bash
showmount -e nfs-server
sudo mount -t nfs4 nfs-server:/exports/app /mnt/test
exportfs -v   # on server
```

**Fix →** Fix `/etc/exports`, `exportfs -ra`, correct `device` and `addr` in volume options.

**Prevent →** Test mounts on every node before deploying.

</details>

### Q122. Database corrupted after the host lost power: `PANIC: could not locate a valid checkpoint record`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unclean shutdown plus storage ignoring fsync (write cache without battery), or copied files from a running DB.

**Diagnose →** Check WAL files, disk write-cache settings, filesystem errors in `dmesg`.

**Fix →** Restore from backup/PITR; as a last resort `pg_resetwal` (data loss risk).

**Prevent →** Reliable storage, replication, tested backups, and don't snapshot running DB files without proper tooling.

</details>

### Q123. Image layers listed under `/var/lib/docker/overlay2` but not in `docker images` (orphaned). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Interrupted pulls/builds or daemon crash leaving layer dirs.

**Diagnose →** `du -sh /var/lib/docker/overlay2`; compare with `docker system df`.

**Fix →** `docker system prune -a`; if still large, stop Docker, back up, and clear `/var/lib/docker` for a rebuild (images re-pulled, volumes backed up first).

**Prevent →** Avoid killing dockerd with SIGKILL; keep disks healthy.

</details>

### Q124. `docker run --tmpfs /tmp:size=64m` app dies with `No space left on device` writing temp files. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** tmpfs limit too small.

**Diagnose →** `docker exec c df -h /tmp`.

**Fix →** Increase `size=`, or write large temp files to a volume.

**Prevent →** Size tmpfs from observed use and remember it consumes memory.

</details>

### Q125. `chown: changing ownership of '/var/lib/mysql/...': Operation not permitted` at container start. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Bind mount on NFS with root_squash, a Windows/macOS file share, or rootless user namespaces where chown to a mapped UID is not allowed.

**Diagnose →** `mount | grep <path>`; `docker exec c id`.

**Fix →** Use a named volume, fix export options (`no_root_squash` or proper UID), or map `user:` to the owner.

**Prevent →** Avoid databases on network/shared filesystems.

</details>

---

## Registry, pull and push errors

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. `Error response from daemon: pull access denied for X, repository does not exist or may require 'docker login'` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Typo in the name, private repo without login, or a repo that doesn't exist for that tag/namespace.

**Diagnose →**
```bash
docker login
docker manifest inspect X:tag
```

**Fix →** Correct the name (include namespace, e.g. `library/` implied for official), log in with an account that has pull rights.

**Prevent →** Use full registry paths and test pulls in CI.

</details>

### Q127. `manifest for myapp:1.2 not found: manifest unknown: manifest unknown` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Tag does not exist (not pushed, deleted by retention policy, typo), or it exists but not for your platform.

**Diagnose →** `crane ls reg/myapp | grep 1.2`; `docker buildx imagetools inspect reg/myapp:1.2`.

**Fix →** Push the tag, rebuild, or use an existing one.

**Prevent →** Retention rules protect release tags.

</details>

### Q128. `no matching manifest for linux/arm64/v8 in the manifest list entries` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image published only for amd64.

**Diagnose →** `docker buildx imagetools inspect image | grep Platform`.

**Fix →** `docker pull --platform linux/amd64 image` (emulated), find an arm64 build, or build your own multi-arch.

**Prevent →** Publish multi-platform manifests.

</details>

### Q129. `x509: certificate signed by unknown authority` on `docker pull` from a private registry. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Self-signed or internal CA not trusted by the daemon.

**Diagnose →** `openssl s_client -connect reg.example.com:443 -showcerts </dev/null`.

**Fix →** Put the CA at `/etc/docker/certs.d/reg.example.com/ca.crt` (no restart needed) or into the system trust store, then retry.

**Prevent →** Use certificates from a trusted CA (ACME, internal PKI distributed to hosts).

</details>

### Q130. `http: server gave HTTP response to HTTPS client` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Registry serves plain HTTP (e.g. local `registry:2`) but the client tries HTTPS.

**Diagnose →** `curl -v http://host:5000/v2/`.

**Fix →** Enable TLS on the registry (preferred), or add to `insecure-registries` in `daemon.json` and restart (non-prod only).

**Prevent →** Always terminate TLS at the registry.

</details>

### Q131. `unauthorized: authentication required` on push to GitHub Container Registry. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Token lacks `write:packages`, not logged in to `ghcr.io`, or repo/package permissions not granting the workflow.

**Diagnose →** `docker login ghcr.io -u USER`; check package settings > Manage Actions access.

**Fix →** Use a PAT with `write:packages` or `GITHUB_TOKEN` with `packages: write` permission in the workflow; lowercase the image name.

**Prevent →** Grant least privilege explicitly in the workflow `permissions:`.

</details>

### Q132. ECR: `denied: Your authorization token has expired. Reauthenticate and try again.` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ECR tokens last 12 hours.

**Diagnose →** Check when you logged in.

**Fix →**
```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

**Prevent →** Use `amazon-ecr-credential-helper` or log in at the start of each job.

</details>

### Q133. ECR push: `name unknown: The repository with name 'app' does not exist in the registry` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** ECR requires the repository to be created first.

**Diagnose →** `aws ecr describe-repositories`.

**Fix →** `aws ecr create-repository --repository-name app --image-scanning-configuration scanOnPush=true`.

**Prevent →** Create repos with Terraform.

</details>

### Q134. `toomanyrequests: You have reached your pull rate limit` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Anonymous/free-tier Docker Hub limits per IP or account.

**Diagnose →** Query the `ratelimit-remaining` header from the preview endpoint.

**Fix →** Authenticate, use a paid org account/PAT, set up a pull-through cache and mirror base images.

**Prevent →** Do not pull from Hub in production paths.

</details>

### Q135. Push is very slow or fails with `received unexpected HTTP status: 502 Bad Gateway` or `413 Request Entity Too Large`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Reverse proxy in front of the registry limits body size or times out on big layers.

**Diagnose →** Proxy logs; `curl -v` upload test.

**Fix →** Nginx: `client_max_body_size 0; proxy_request_buffering off; proxy_read_timeout 900;` and required headers for registry.

**Prevent →** Use registry-specific proxy config templates.

</details>

### Q136. `docker pull` stuck on `Waiting`/`Pulling fs layer` then `unexpected EOF`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Network instability, MTU issues, proxy resets, registry throttling.

**Diagnose →**
```bash
docker pull -q img; echo $?
ping -M do -s 1472 registry-1.docker.io
journalctl -u docker | tail
```

**Fix →** Lower MTU, reduce `max-concurrent-downloads`, use a mirror, retry.

**Prevent →** Local pull-through cache.

</details>

### Q137. `Error response from daemon: Head "https://registry/v2/...": denied` with correct credentials after switching credential helpers. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Stale entry in the keychain/`config.json`, or `credHelpers` pointing to the wrong helper per registry.

**Diagnose →** `cat ~/.docker/config.json`; `docker-credential-<helper> list`.

**Fix →** `docker logout reg && docker login reg`; correct `credHelpers`.

**Prevent →** One credential strategy per environment.

</details>

### Q138. `failed to authorize: failed to fetch oauth token: ... 401` from BuildKit while `docker pull` works. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Builder uses different credentials (running as another user/sudo, remote builder), or the base image is private and no login in the build context.

**Diagnose →** `sudo -E docker buildx build`; `echo $HOME`; which `config.json` is read.

**Fix →** Log in as the same user running the build, set `DOCKER_CONFIG`, avoid `sudo` without `-E`.

**Prevent →** Standardise runner user/credentials.

</details>

### Q139. Pushed image appears on registry but nodes pull the old content with the same tag. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Node already has the tag cached and `imagePullPolicy: IfNotPresent` (or Compose didn't `pull`), or a CDN/mirror cache serves stale manifests.

**Diagnose →** Compare `docker images --digests` on the node with the registry digest.

**Fix →** Deploy by digest or unique tags; `docker compose pull`; set `imagePullPolicy: Always` only if you must keep mutable tags.

**Prevent →** Immutable tags.

</details>

### Q140. Harbor/Registry push: `blob upload unknown` or `unknown blob` mid-push. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multiple registry replicas without shared storage/sticky sessions, storage backend (S3) eventual consistency, garbage collection running during push.

**Diagnose →** Registry logs; check LB affinity and storage config.

**Fix →** Use shared backend storage, sticky upload sessions or Redis for upload state, don't run GC while writes are active (or use read-only mode).

**Prevent →** Schedule GC in maintenance windows.

</details>

### Q141. `docker manifest create` error: `... is a manifest list` / `no such manifest`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Source images not pushed to the registry yet, or a source is itself a manifest list.

**Diagnose →** `docker manifest inspect reg/app:amd64`.

**Fix →** Push per-arch images first, then create; or use `docker buildx imagetools create -t reg/app:1 reg/app:amd64 reg/app:arm64`.

**Prevent →** Prefer `buildx --platform` multi-arch builds.

</details>

### Q142. `Error response from daemon: error parsing HTTP 404 response body` when pulling from a mirror. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mirror/proxy doesn't implement the registry v2 API at that path, or returns an HTML error page.

**Diagnose →** `curl -i https://mirror/v2/`; look at response headers `Docker-Distribution-Api-Version`.

**Fix →** Correct the mirror URL/path, fix proxy rewriting, use the proper pull-through endpoint.

**Prevent →** Health-check the mirror with `/v2/`.

</details>

### Q143. `docker push` fails: `denied: requested access to the resource is denied` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Image name lacks your namespace (`myapp` instead of `user/myapp`), not logged in, or no push rights.

**Diagnose →** `docker images`; `docker info | grep Username`.

**Fix →** `docker tag myapp user/myapp:1.0 && docker push user/myapp:1.0` after login.

**Prevent →** Tag with full registry paths in CI.

</details>

### Q144. Image push works, but `docker pull` on another machine shows `image operating system "windows" cannot be used on this platform`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Windows container image on a Linux host.

**Diagnose →** `docker manifest inspect img | grep os`.

**Fix →** Use a Linux image or run on Windows with Windows containers mode.

**Prevent →** Label image OS/arch in docs.

</details>

### Q145. Registry disk usage doesn't drop after deleting tags. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Deleting tags/manifests doesn't delete blobs until garbage collection runs; deletes disabled (`REGISTRY_STORAGE_DELETE_ENABLED`).

**Diagnose →** Check registry config for `delete.enabled`.

**Fix →**
```bash
registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
```
In Harbor trigger GC from the UI/API.

**Prevent →** Scheduled GC and retention rules.

</details>

### Q146. `Error: image not known` when using `podman pull alpine`: `short-name resolution enforced but cannot prompt without a TTY`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Podman's short-name mode is `enforcing` and an unqualified name has several candidate registries in a non-interactive context.

**Diagnose →** `cat /etc/containers/registries.conf; podman info | grep -i search`.

**Fix →** Use fully qualified names (`docker.io/library/alpine`), or define `[aliases]` in `registries.conf.d`.

**Prevent →** Always qualify image names in scripts.

</details>

### Q147. `docker login` succeeds but pulls from the private registry still fail on Kubernetes with `ImagePullBackOff`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `imagePullSecrets`, wrong namespace for the secret, expired cloud token, or node can't reach the registry.

**Diagnose →**
```bash
kubectl describe pod p | tail -20
kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
crictl pull reg/app:tag    # on the node
```

**Fix →** Create the secret in the pod's namespace, link to ServiceAccount, fix network/credentials.

**Prevent →** Use cloud credential providers.

</details>

### Q148. `failed to verify certificate: x509: certificate is valid for X, not registry.local` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Hostname used differs from the certificate SAN (IP address or alias).

**Diagnose →** `openssl s_client -connect host:443 | openssl x509 -noout -text | grep -A1 'Alternative'`.

**Fix →** Use the DNS name in the SAN, or re-issue the certificate with the correct SANs.

**Prevent →** Issue certs with all hostnames and IPs.

</details>

### Q149. `Error: pull access denied` after Docker Hub changed org/repo visibility or deleted a tag. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Publisher made the repo private or removed it.

**Diagnose →** Browse the Hub page; try `docker pull` with login.

**Fix →** Switch to an alternative or your mirrored copy.

**Prevent →** Mirror third-party images you depend on.

</details>

### Q150. Pull succeeds but `docker run` says `Unable to find image ... locally` when using `DOCKER_HOST=ssh://`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Pulled on the local daemon but running on the remote one (or the reverse); context mismatch.

**Diagnose →** `docker context show; docker info | grep Name`.

**Fix →** Use the same context for both commands (`docker --context remote pull`).

**Prevent →** Show the current context in your shell prompt.

</details>

---

## Resources, health checks, logging and restarts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Container restarts repeatedly: `docker ps` shows `Restarting (1) 5 seconds ago`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App crashes on start (bad config, missing dependency, port conflict) with a restart policy re-launching it.

**Diagnose →**
```bash
docker logs --tail 100 c
docker inspect -f '{{.State.ExitCode}} {{.State.Error}} {{.RestartCount}}' c
```

**Fix →** Fix the startup error; temporarily `docker update --restart no c` to stop the loop while debugging.

**Prevent →** Use `on-failure:N` for jobs and alert on restart counts.

</details>

### Q152. `OOMKilled: true` and exit code 137. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Process exceeded `--memory`, a leak, or a limit too small for the workload (JVM, page cache for tmpfs).

**Diagnose →**
```bash
docker inspect -f '{{.State.OOMKilled}} {{.HostConfig.Memory}}' c
docker stats --no-stream c
dmesg -T | grep -i 'memory cgroup out of memory'
```

**Fix →** Raise the limit, tune the runtime (heap `-Xmx` or `MaxRAMPercentage`), fix leaks.

**Prevent →** Load-test to size limits; alert at 90% of the limit.

</details>

### Q153. Container is `unhealthy` but the app works when tested manually. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Healthcheck command missing in the image (`curl: not found`), wrong port/path, too-short timeout, runs as a different user, or checks a dependency that's down.

**Diagnose →**
```bash
docker inspect -f '{{json .State.Health}}' c | jq
docker exec c sh -c 'curl -fsS http://localhost:8080/healthz; echo $?'
```

**Fix →** Install the needed tool or use the app binary, fix URL, raise `timeout`/`start_period`.

**Prevent →** Test the health command in CI.

</details>

### Q154. `docker stats` shows 100%+ CPU and the container is slow. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Multi-core usage above 100% is normal (200% = two cores); with `--cpus` limit the app is throttled.

**Diagnose →**
```bash
docker exec c cat /sys/fs/cgroup/cpu.stat
docker exec c top -b -n1 | head -15
```

**Fix →** Increase `--cpus`, profile the hot path, scale out.

**Prevent →** Track `nr_throttled` metrics.

</details>

### Q155. `docker logs` is slow or `docker logs -f` hangs on a container with huge logs. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Large `json-file` log being read from the start.

**Diagnose →** `ls -lh /var/lib/docker/containers/<id>/<id>-json.log`.

**Fix →** `docker logs --tail 200 -f c`, `--since 10m`; rotate logs.

**Prevent →** Rotation and shipping logs to a backend.

</details>

### Q156. `Error response from daemon: configured logging driver does not support reading` when running `docker logs`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Driver such as `syslog`, `gelf`, `fluentd` or `awslogs` without dual logging support in that engine/plugin setup.

**Diagnose →** `docker inspect -f '{{.HostConfig.LogConfig.Type}}' c`.

**Fix →** Read logs in the backend, or use `local`/`json-file` driver; Docker 20.10+ dual logging usually caches for most drivers.

**Prevent →** Choose a driver with local cache (or run a collector sidecar).

</details>

### Q157. Application hangs and logs stop: log driver blocking (`fluentd: connection refused`). 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default blocking log mode: when the remote log endpoint is unavailable, writes to stdout block the app.

**Diagnose →** `docker inspect -f '{{.HostConfig.LogConfig}}' c`; `strace -p <pid>` shows `write(1, ...)` blocked; check collector.

**Fix →** `--log-opt mode=non-blocking --log-opt max-buffer-size=4m`; restore the collector.

**Prevent →** Use a local collector tailing files rather than direct network drivers.

</details>

### Q158. Container killed by the OOM killer but `docker stats` never showed high memory. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Short spikes between sampling intervals, kernel memory/tmpfs counted in the cgroup, or sub-process memory.

**Diagnose →**
```bash
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.events
cat .../memory.peak
dmesg -T | grep -B2 -A10 'Memory cgroup out of memory'
```

**Fix →** Raise the limit by the observed `memory.peak`, reduce spikes (batch size, parallelism).

**Prevent →** Alert on `memory.events` `oom_kill` counters.

</details>

### Q159. `docker update --memory` fails: `Cannot update container ... Memory limit should be smaller than already set memoryswap limit`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Existing `memory-swap` is lower than the new memory limit.

**Diagnose →** `docker inspect -f '{{.HostConfig.MemorySwap}}' c`.

**Fix →** `docker update --memory 1g --memory-swap 1g c` (set both).

**Prevent →** Set both values together.

</details>

### Q160. Limits seem ignored: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Swap accounting disabled in the kernel (common on Debian/Ubuntu defaults).

**Diagnose →** `docker info | grep -i swap`; `cat /proc/cmdline`.

**Fix →** Add `cgroup_enable=memory swapaccount=1` to GRUB (cgroup v1), or use cgroup v2 where it works; update-grub and reboot.

**Prevent →** Standardise kernel parameters in the host image.

</details>

### Q161. Restart policy `unless-stopped` container comes back after `docker stop` + daemon restart. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** It was stopped by something other than an explicit `docker stop` (e.g. Compose `down`, `kill`), or `always` was used (restarts regardless on daemon start).

**Diagnose →** `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' c`.

**Fix →** Use `unless-stopped` or `no` and stop via `docker stop`.

**Prevent →** Know the semantic difference between `always` and `unless-stopped`.

</details>

### Q162. `docker events` shows `health_status: unhealthy` flapping every minute. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Health command timing out under load (timeout 1s default 30s interval depending on setting), GC pauses, dependency checks too strict.

**Diagnose →** `docker inspect -f '{{range .State.Health.Log}}{{.Start}} {{.ExitCode}} {{.Output}}{{end}}' c`.

**Fix →** Lighter health endpoint, larger `timeout`/`retries`, separate liveness from dependency checks.

**Prevent →** Make health endpoints cheap and cached.

</details>

### Q163. Java container ignores memory limit and uses host memory (old JDK): `OutOfMemoryError` or killed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Older JDK (<8u191) doesn't read cgroup limits, so default heap is 25% of host RAM.

**Diagnose →** `docker exec c java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|UseContainerSupport'`.

**Fix →** Upgrade JDK (11+/17/21), set `-XX:MaxRAMPercentage=70`, or explicit `-Xmx`.

**Prevent →** Use current LTS base images.

</details>

### Q164. Node.js container OOM despite `--memory 512m`: V8 heap default is larger. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** V8 sizes heap from available memory/limits and Buffers live outside heap; default old-space can exceed the limit.

**Diagnose →** `docker exec c node -e "console.log(v8.getHeapStatistics().heap_size_limit/1048576)"`.

**Fix →** `NODE_OPTIONS=--max-old-space-size=384` (about 75% of limit), watch RSS.

**Prevent →** Set heap flags together with limits.

</details>

### Q165. Container can't create threads/processes: `fork: retry: Resource temporarily unavailable`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `--pids-limit` reached, or host/user `nproc` ulimit, or threads leak.

**Diagnose →**
```bash
docker inspect -f '{{.HostConfig.PidsLimit}}' c
docker exec c sh -c 'ls /proc | grep -c "^[0-9]"'
cat /sys/fs/cgroup/.../pids.current
```

**Fix →** Raise limit, fix leaking process/thread creation, reap zombies (`--init`).

**Prevent →** Alert on `pids.current`.

</details>

### Q166. `too many open files` inside a container. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Low `nofile` ulimit, FD leak, or host `fs.file-max`/inotify limits.

**Diagnose →**
```bash
docker exec c sh -c 'ulimit -n'
docker exec c ls /proc/1/fd | wc -l
sysctl fs.inotify.max_user_instances
```

**Fix →** `--ulimit nofile=65536:65536`; raise inotify sysctls on host for watchers (`ENOSPC` on inotify); fix leaks.

**Prevent →** Standard ulimits in `daemon.json` `default-ulimits`.

</details>

### Q167. Elasticsearch container exits: `max virtual memory areas vm.max_map_count [65530] is too low`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host kernel setting too low (containers share it).

**Diagnose →** `sysctl vm.max_map_count`.

**Fix →** `sudo sysctl -w vm.max_map_count=262144`; persist in `/etc/sysctl.d/99-es.conf`.

**Prevent →** Provision sysctls in host config.

</details>

### Q168. Container clock skew breaks JWT/TLS validation (`token used before issued`). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Host or Docker Desktop VM clock drift (after sleep).

**Diagnose →** `docker run --rm busybox date -u; date -u`.

**Fix →** Fix NTP/chrony on the host; restart Docker Desktop to resync VM time.

**Prevent →** Time sync monitoring.

</details>

### Q169. `docker stop` hangs then kills; app data corrupted because shutdown never completed. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Shutdown takes longer than 10 s default stop timeout.

**Diagnose →** Time `docker stop c`; look at app shutdown logs.

**Fix →** `docker stop -t 60`, `stop_grace_period: 60s`, handle SIGTERM, correct `STOPSIGNAL` (for example `SIGINT`/`SIGQUIT` for some servers).

**Prevent →** Test graceful shutdown.

</details>

### Q170. Container's DNS or hostname weirdness after `docker update`/network change: `docker update` doesn't change network. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `docker update` only changes resources and restart policy, not networks, ports or env.

**Diagnose →** `docker update --help`.

**Fix →** `docker network connect/disconnect` for networks; recreate the container for ports/env.

**Prevent →** Treat containers as immutable; change config in Compose and recreate.

</details>

### Q171. Disk I/O wait high on the host and all containers slow. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** One noisy container (logging, DB, backup), overlay copy-up on huge files, or small volume IOPS.

**Diagnose →**
```bash
iostat -x 1
sudo iotop -oPa
docker stats --no-stream
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/io.stat
```

**Fix →** Throttle (`--device-write-bps`), move hot paths to dedicated volumes, increase IOPS.

**Prevent →** Per-container IO metrics and limits.

</details>

### Q172. Memory not released after load; `docker stats` stays high. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Allocators (glibc arenas, JVM) hold memory, page cache is included, or a real leak.

**Diagnose →** `docker exec c cat /sys/fs/cgroup/memory.stat | head -20` (anon vs file); heap dumps.

**Fix →** Tune allocator (`MALLOC_ARENA_MAX=2`, jemalloc), cap heap; fix the leak if anon grows steadily.

**Prevent →** Soak tests with memory graphs.

</details>

### Q173. `docker compose up` says services are `unhealthy` and dependent services never start: `dependency failed to start: container db is unhealthy`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Health check fails (wrong credentials, wrong command, slow initialization such as a DB restore).

**Diagnose →**
```bash
docker compose ps
docker compose logs db
docker inspect --format '{{json .State.Health}}' $(docker compose ps -q db) | jq
```

**Fix →** Correct the check (`pg_isready -U $POSTGRES_USER`), add `start_period`, fix env.

**Prevent →** Test health commands manually first.

</details>

### Q174. Live logs are missing lines or delayed with `local` driver: `docker logs --since` shows fewer lines than expected after rotation. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Rotation (`max-size`, `max-file`) discarded older entries; small buffers with non-blocking mode drop messages (`max-buffer-size` exceeded).

**Diagnose →** Check `log-opts`; look for `dropped` messages in `journalctl -u docker`.

**Fix →** Raise sizes, ship logs externally before rotation, use blocking mode where loss is unacceptable.

**Prevent →** Size rotation for peak throughput, centralize logs.

</details>

### Q175. Container's CPU usage inside `top` differs from `docker stats`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `top` inside shows host-level `/proc/stat` data (no lxcfs), while `docker stats` reads cgroup accounting.

**Diagnose →** Compare `docker stats` with `cat /sys/fs/cgroup/cpu.stat` deltas.

**Fix →** Trust cgroup metrics (cAdvisor); use `nproc`/cgroup files for in-app sizing.

**Prevent →** Monitoring based on cgroup metrics, not in-container `top`.

</details>

---

## Security, rootless, Podman and containerd

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. Rootless Docker: `dockerd-rootless.sh` fails with `newuidmap: write to uid_map failed: Operation not permitted` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing `uidmap` package/setuid bit or no subuid/subgid ranges for the user.

**Diagnose →**
```bash
grep $USER /etc/subuid /etc/subgid
ls -l $(which newuidmap)
sysctl kernel.unprivileged_userns_clone user.max_user_namespaces
```

**Fix →** Install `uidmap`, add ranges (`usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER`), enable user namespaces.

**Prevent →** Include these in the base image.

</details>

### Q177. Rootless: `rootlesskit: failed to ... listen tcp 0.0.0.0:80: bind: permission denied` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Unprivileged users can't bind ports <1024.

**Diagnose →** `sysctl net.ipv4.ip_unprivileged_port_start`.

**Fix →** `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80`, or publish a high port and redirect with firewall/proxy.

**Prevent →** Use 8080/8443 internally.

</details>

### Q178. Rootless Docker stops when you log out of SSH. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The user systemd instance ends at logout.

**Diagnose →** `loginctl show-user $USER | grep Linger`.

**Fix →** `sudo loginctl enable-linger $USER`.

**Prevent →** Make it part of provisioning.

</details>

### Q179. Rootless containers: `Error: ... cannot set up namespace using "/usr/bin/newuidmap": should have setuid or have filecaps setuid` (Podman). 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** `newuidmap/newgidmap` lack setuid or file capabilities (e.g. minimal image or container-in-container).

**Diagnose →** `ls -l /usr/bin/newuidmap; getcap /usr/bin/newuidmap`.

**Fix →** `chmod u+s` or `setcap cap_setuid+ep`, reinstall `shadow-utils`/`uidmap`.

**Prevent →** Install proper packages, not manual copies.

</details>

### Q180. Podman: `Error: short-name "nginx" did not resolve to an alias and no unqualified-search registries are defined in "/etc/containers/registries.conf"` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** No `unqualified-search-registries` configured.

**Diagnose →** `grep -v '^#' /etc/containers/registries.conf`.

**Fix →** Use `docker.io/library/nginx`, or add `unqualified-search-registries = ["docker.io"]`.

**Prevent →** Fully qualified names in scripts.

</details>

### Q181. Podman rootless: `Error: cannot find newuidmap` / ping fails inside container: `ping: permission denied (are you root?)`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Ping needs raw sockets; unprivileged ICMP range excludes the user's GID.

**Diagnose →** `sysctl net.ipv4.ping_group_range`.

**Fix →** `sudo sysctl -w net.ipv4.ping_group_range="0 2147483647"`.

**Prevent →** Persist in sysctl.d.

</details>

### Q182. Rootless Podman volumes show wrong owner: files appear owned by a huge UID (e.g. 100999) on the host. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** User namespace mapping: container UID N maps to subuid base + N − 1 on host.

**Diagnose →** `podman unshare ls -ln dir`; `cat /proc/self/uid_map`.

**Fix →** Use `podman unshare chown`, `--userns=keep-id`, or `:U` volume option.

**Prevent →** Plan UID mapping for shared directories.

</details>

### Q183. `docker run --privileged` container breaks host networking after it runs `iptables -F`. 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Privileged container (or `NET_ADMIN` with host network) altered host firewall rules.

**Diagnose →** `docker inspect -f '{{.HostConfig.Privileged}} {{.HostConfig.NetworkMode}}' c`; `iptables -S`.

**Fix →** Restart Docker to recreate chains; remove privileges and host networking.

**Prevent →** Policy blocking privileged containers, `--cap-drop ALL`.

</details>

### Q184. Trivy/Scout reports a CVE in a package your app does not use. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Package present in the base image; may be unreachable or unfixed.

**Diagnose →**
```bash
trivy image --severity HIGH,CRITICAL --format json app | jq '.Results[].Vulnerabilities[]? | {PkgName,InstalledVersion,FixedVersion}'
docker run --rm --entrypoint sh app -c 'apt list --installed 2>/dev/null | head'
```

**Fix →** Remove the package, switch to slimmer base, update base, or document VEX/ignore with expiry.

**Prevent →** Minimal bases; scan on a schedule.

</details>

### Q185. `docker scan`/`docker scout` fails: `unauthorized` or `failed to resolve image`. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Not logged in to Docker Hub, `docker scan` removed (replaced by Scout), or the image is only in a registry the tool can't reach.

**Diagnose →** `docker scout version`; `docker login`.

**Fix →** Use `docker scout cves image` after login, or `trivy image`.

**Prevent →** Standardize on one scanner in CI.

</details>

### Q186. `permission denied` when container tries to write with `--read-only`: `Read-only file system`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** App writes to a path not mounted writable (`/tmp`, `/var/run`, cache dirs).

**Diagnose →** `docker run --read-only ...` then `strace -f -e trace=openat,mkdir` or app logs naming the path.

**Fix →** Add `--tmpfs /tmp --tmpfs /run` or volumes for the specific paths.

**Prevent →** Test with read-only root in CI.

</details>

### Q187. `operation not permitted` running `ip link add` or `sysctl -w` inside a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Missing capability (`NET_ADMIN`/`SYS_ADMIN`), or sysctl not namespaced.

**Diagnose →** `docker exec c capsh --print` (if available) or `grep Cap /proc/1/status`.

**Fix →** `--cap-add NET_ADMIN` for networking tools; set namespaced sysctls with `--sysctl net.ipv4.ip_forward=1`; host-wide sysctls must be set on the host.

**Prevent →** Add only needed capabilities.

</details>

### Q188. `ptrace: Operation not permitted` when using `strace`/`gdb` in a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default seccomp/capabilities block ptrace.

**Diagnose →** `docker inspect -f '{{.HostConfig.SecurityOpt}} {{.HostConfig.CapAdd}}' c`.

**Fix →** Debug-only: `--cap-add SYS_PTRACE` (and `--security-opt seccomp=unconfined` on older kernels).

**Prevent →** Use ephemeral debug containers instead of loosening prod.

</details>

### Q189. AppArmor denial: `apparmor="DENIED" operation="mount"` in dmesg when starting a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The default `docker-default` profile blocks the operation (mount, ptrace, etc.).

**Diagnose →** `dmesg | grep apparmor`; `aa-status`.

**Fix →** Write a custom profile with the needed rule (`--security-opt apparmor=custom`), or avoid the operation. Not `unconfined` unless sandboxed.

**Prevent →** Develop apps to work under default profiles.

</details>

### Q190. SELinux: `avc: denied { read } for ... scontext=system_u:system_r:container_t:s0:c1,c2 tcontext=...:user_home_t` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mounted a host directory without a container label.

**Diagnose →** `sudo ausearch -m avc -ts recent | audit2why`.

**Fix →** `:z`/`:Z` on the volume, or label `container_file_t`; avoid `--security-opt label=disable` outside tests.

**Prevent →** Keep data under labeled dirs such as `/srv`.

</details>

### Q191. containerd: `failed to pull and unpack image: failed to resolve reference ... x509` / `http: server gave HTTP response to HTTPS client` 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** containerd isn't using Docker's `certs.d`; the registry TLS/CA isn't configured for containerd.

**Diagnose →** `crictl pull reg/app:tag`; `grep -A5 registry /etc/containerd/config.toml`.

**Fix →** Configure `config_path = "/etc/containerd/certs.d"` and `hosts.toml` with CA or `skip_verify` (test only); `systemctl restart containerd`.

**Prevent →** Provision registry config for containerd with config management.

</details>

### Q192. `crictl` says `unable to determine runtime API version: ... connection refused` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** crictl endpoint not set or containerd not running.

**Diagnose →** `cat /etc/crictl.yaml`; `systemctl status containerd`; `ls /run/containerd/containerd.sock`.

**Fix →** `crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock` and start containerd.

**Prevent →** Ship `/etc/crictl.yaml` on nodes.

</details>

### Q193. `ctr images ls` shows nothing though Kubernetes runs pods. 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Wrong containerd namespace; Kubernetes uses `k8s.io`.

**Diagnose →** `ctr namespaces ls`.

**Fix →** `ctr -n k8s.io images ls` or `crictl images`.

**Prevent →** Prefer crictl on Kubernetes nodes.

</details>

### Q194. containerd: `failed to create shim task: OCI runtime create failed: ... expected cgroupsPath to be of format "slice:prefix:name"` 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Mismatch: kubelet uses systemd cgroup driver while containerd/runc uses cgroupfs (or vice versa).

**Diagnose →**
```bash
grep -n SystemdCgroup /etc/containerd/config.toml
grep cgroupDriver /var/lib/kubelet/config.yaml
```

**Fix →** Set `SystemdCgroup = true` under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` (or the v3 config equivalent) and `cgroupDriver: systemd`; restart both.

**Prevent →** Align drivers when bootstrapping nodes (kubeadm defaults).

</details>

### Q195. Podman container cannot reach another by name: `Temporary failure in name resolution` on the default network. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Default `podman` network has DNS disabled; name resolution needs a user-created network with aardvark-dns.

**Diagnose →** `podman network inspect podman | jq '.[0].dns_enabled'`.

**Fix →** `podman network create appnet` and use `--network appnet`.

**Prevent →** Always create named networks.

</details>

### Q196. Podman: `Error: creating container storage: the container name "web" is already in use` 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Previous container (possibly stopped) holds the name.

**Diagnose →** `podman ps -a --filter name=web`.

**Fix →** `podman rm -f web`; use `--replace`.

**Prevent →** `--replace` or `--rm` in scripts.

</details>

### Q197. `podman-compose`/`docker compose` against Podman socket fails: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Podman API socket isn't enabled or `DOCKER_HOST` isn't set.

**Diagnose →** `systemctl --user status podman.socket`; `ls $XDG_RUNTIME_DIR/podman/podman.sock`.

**Fix →**
```bash
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
```

**Prevent →** Set `DOCKER_HOST` in the shell profile.

</details>

### Q198. Image runs fine as root but fails as non-root (`EACCES` creating `/app/.cache` or `/.npm`). 🟢

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Home/cache directories not writable by the non-root UID; no passwd entry for arbitrary UIDs.

**Diagnose →** `docker run --user 10001 --rm image sh -c 'id; echo $HOME; touch $HOME/x'`.

**Fix →** `chown` dirs at build, set `HOME=/tmp` or a writable path, `npm_config_cache=/tmp/.npm`.

**Prevent →** Test with the production user and read-only FS.

</details>

### Q199. Docker socket proxy breaks Traefik/Portainer: `403 Forbidden` from the proxy. 🟡

<details>
<summary><b>Answer</b></summary>

**Likely causes →** The socket proxy denies API sections by default (CONTAINERS, NETWORKS, etc.).

**Diagnose →** Proxy logs; compare required API calls.

**Fix →** Enable only needed env vars (`CONTAINERS=1`, `NETWORKS=1`, `SERVICES=1`), keep `POST=0`.

**Prevent →** Document the minimal permission set per tool.

</details>

### Q200. Suspected container escape: unknown process on the host tied to a container. How do you investigate? 🔴

<details>
<summary><b>Answer</b></summary>

**Likely causes →** Privileged container, mounted socket, vulnerable runtime (runc/kernel CVE), or compromised credentials; or a miner running inside the container's namespaces.

**Diagnose →**
```bash
ps -eo pid,ppid,user,cmd --forest
docker inspect -f '{{.Id}} {{.State.Pid}}' $(docker ps -q)
ls -l /proc/<pid>/ns /proc/<pid>/exe
nsenter -t <pid> -n ss -tnp
docker diff <c>; ausearch -k docker
```

**Fix →** Isolate the host (network quarantine), preserve evidence (`docker export`, memory/disk snapshot), then rebuild host and containers from clean images and rotate all secrets that were reachable.

**Prevent →** Patch runc/kernel, drop privileges, no socket mounts, runtime detection (Falco), and egress filtering.

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
