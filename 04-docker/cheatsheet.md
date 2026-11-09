# Docker & Containers — Cheatsheet

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

> Dense command and config reference for daily use and interview prep. Docker Engine 26+, Compose v2, BuildKit.

[⬅ Back to Docker & Containers](./README.md) · [🏠 Home](../README.md)

## Contents

[Run and manage](#run-and-manage-containers) · [Inspect and debug](#inspect-and-debug) · [Images](#images) · [Build](#build-and-buildx) · [Dockerfile](#dockerfile-reference) · [Networking](#networking) · [Storage](#volumes-and-mounts) · [Compose](#docker-compose) · [Registry](#registry-and-tagging) · [Security](#security-hardening) · [Trivy](#trivy) · [Cleanup](#cleanup) · [Exit codes](#exit-codes) · [Top 15](#top-15-interview-one-liners)

---

## Run and manage containers

| Task | Command |
|------|---------|
| Run detached with name and port | `docker run -d --name web -p 8080:80 nginx:alpine` |
| Bind only to localhost | `docker run -d -p 127.0.0.1:5432:5432 postgres:16-alpine` |
| Interactive shell, auto-remove | `docker run --rm -it alpine sh` |
| Environment variables | `docker run -e KEY=val --env-file app.env img` |
| Restart policy | `--restart unless-stopped` (`no`, `on-failure:5`, `always`) |
| Override entrypoint | `docker run --rm -it --entrypoint sh img` |
| Run as specific user | `docker run --user 10001:10001 img` |
| Proper PID 1 (reaps zombies) | `docker run --init img` |
| Set hostname / workdir | `--hostname h1 -w /app` |
| Stop gracefully (custom timeout) | `docker stop -t 30 web` |
| Kill with signal | `docker kill -s HUP web` |
| Start / restart / pause | `docker start web`, `docker restart web`, `docker pause web` |
| Rename | `docker rename old new` |
| List running / all | `docker ps`, `docker ps -a` |
| Custom ps format | `docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'` |
| Wait for exit code | `docker wait web` |
| Copy files | `docker cp web:/etc/nginx/nginx.conf .` and `docker cp ./f web:/tmp/` |
| Update limits live | `docker update --memory 512m --cpus 1 web` |

## Inspect and debug

| Task | Command |
|------|---------|
| Logs, follow, last 100, with time | `docker logs -f --tail 100 -t web` |
| Logs since | `docker logs --since 10m web` |
| Shell into running container | `docker exec -it web sh` |
| Run one command | `docker exec web cat /etc/resolv.conf` |
| Full JSON | `docker inspect web` |
| Exit code, OOM, restarts | `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}} {{.RestartCount}}' web` |
| IP address | `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' web` |
| Health history | `docker inspect -f '{{json .State.Health}}' web` |
| Host PID | `docker inspect -f '{{.State.Pid}}' web` |
| Processes in container | `docker top web` |
| Live resource usage | `docker stats --no-stream` |
| Filesystem changes | `docker diff web` |
| Daemon events | `docker events --since 10m --filter type=container` |
| Port mappings | `docker port web` |
| Debug shell-less container | `docker debug web` or `docker run --rm -it --pid container:web --net container:web nicolaka/netshoot` |
| Disk usage by type | `docker system df -v` |
| Daemon info and storage driver | `docker info` |

## Images

| Task | Command |
|------|---------|
| Pull specific tag or digest | `docker pull nginx:1.27-alpine`, `docker pull nginx@sha256:<digest>` |
| List with digests | `docker images --digests` |
| Layers and sizes | `docker history --no-trunc img` |
| Config as JSON | `docker inspect --format '{{json .Config}}' img \| jq` |
| Tag | `docker tag img:dev reg.io/team/img:1.0.0` |
| Save / load tarball | `docker save -o img.tar img:1.0`, `docker load -i img.tar` |
| Export / import filesystem | `docker export web > fs.tar`, `docker import fs.tar img:flat` |
| Remove | `docker rmi img`, `docker image prune -a --filter until=168h` |
| Commit (debugging only) | `docker commit web img:debug` |
| Search layers interactively | `dive img` |

## Build and buildx

| Task | Command |
|------|---------|
| Basic build | `docker build -t app:dev .` |
| Different Dockerfile / context | `docker build -f docker/Dockerfile.prod -t app ./svc` |
| Build a stage | `docker build --target test .` |
| Build args | `docker build --build-arg VERSION=1.2.3 .` |
| No cache, fresh base | `docker build --no-cache --pull .` |
| Plain progress logs | `docker build --progress=plain .` |
| Secret mount | `docker build --secret id=npm,src=./token.txt .` |
| SSH mount | `docker build --ssh default .` |
| Create builder | `docker buildx create --name multi --driver docker-container --use` |
| Multi-arch push | `docker buildx build --platform linux/amd64,linux/arm64 -t reg/app:1.0 --push .` |
| Registry cache | `--cache-from type=registry,ref=reg/app:cache --cache-to type=registry,ref=reg/app:cache,mode=max` |
| GitHub Actions cache | `--cache-from type=gha --cache-to type=gha,mode=max` |
| Load multi-platform locally | `docker buildx build --platform linux/arm64 --load .` (one platform at a time) |
| SBOM + provenance | `docker buildx build --sbom=true --provenance=mode=max --push .` |
| Inspect manifest list | `docker buildx imagetools inspect reg/app:1.0` |
| Copy/retag without rebuild | `docker buildx imagetools create -t reg/app:prod reg/app:sha-abc1234` |
| Prune build cache | `docker builder prune --keep-storage 5GB` |

## Dockerfile reference

| Instruction | Notes |
|-------------|-------|
| `FROM image:tag AS name` | Pin versions; `FROM --platform=$BUILDPLATFORM` for cross-compiles |
| `ARG NAME=default` | Build-time only; before first `FROM` to parametrize the base |
| `ENV KEY=value` | Persists into runtime |
| `WORKDIR /app` | Creates the directory; prefer over `RUN cd` |
| `COPY --chown=u:g src dst` | Prefer over `ADD`; `COPY --from=stage` for multi-stage |
| `RUN cmd` | One layer per instruction; chain with `&&` and clean up in the same `RUN` |
| `RUN --mount=type=cache,target=/root/.cache` | Persistent package cache across builds |
| `RUN --mount=type=secret,id=x` | Secret available at `/run/secrets/x` for that step only |
| `USER 10001:10001` | Use a numeric UID so Kubernetes `runAsNonRoot` can verify it |
| `EXPOSE 8080` | Documentation only; does not publish |
| `ENTRYPOINT ["app"]` / `CMD ["--flag"]` | Exec form always; CMD gives default args |
| `HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 CMD cmd` | Exit 0 healthy, 1 unhealthy |
| `STOPSIGNAL SIGTERM` | Signal for `docker stop` |
| `LABEL org.opencontainers.image.source=URL` | OCI standard metadata |
| `VOLUME /data` | Declares anonymous volume; use sparingly |

Production skeleton:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node . .
USER node
EXPOSE 3000
HEALTHCHECK CMD node -e "fetch('http://127.0.0.1:3000/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "server.js"]
```

## Networking

| Task | Command |
|------|---------|
| Create user-defined bridge | `docker network create appnet` |
| Internal-only network | `docker network create --internal backend` |
| Custom subnet | `docker network create --subnet 172.30.0.0/24 appnet` |
| Run on network | `docker run -d --network appnet --name api img` |
| Add network alias | `--network-alias api-v2` |
| Connect / disconnect running | `docker network connect appnet web`, `docker network disconnect appnet web` |
| Inspect members | `docker network inspect appnet` |
| Share another container's network | `docker run --network container:web nicolaka/netshoot` |
| Host networking | `docker run --network host img` (Linux only, no isolation) |
| No networking | `docker run --network none img` |
| Add host entry | `--add-host db.local:10.0.0.5` |
| Reach host from container | `--add-host host.docker.internal:host-gateway` |
| Custom DNS | `--dns 1.1.1.1` |

| Driver | Use |
|--------|-----|
| `bridge` | Single host, default; user-defined adds DNS |
| `host` | No NAT, max performance, no isolation |
| `none` | Fully isolated |
| `overlay` | Multi-host (Swarm) |
| `macvlan` / `ipvlan` | Container gets its own L2 address on the LAN |

Embedded DNS is at `127.0.0.11` on user-defined networks. Published ports use iptables DNAT: `sudo iptables -t nat -L DOCKER -n`.

## Volumes and mounts

| Task | Command |
|------|---------|
| Create / list / inspect | `docker volume create pgdata`, `docker volume ls`, `docker volume inspect pgdata` |
| Named volume | `-v pgdata:/var/lib/postgresql/data` |
| Bind mount (read-only) | `-v "$PWD/conf:/etc/app:ro"` |
| Explicit syntax | `--mount type=bind,src=$PWD/conf,dst=/etc/app,readonly` |
| tmpfs | `--tmpfs /tmp:rw,size=64m` |
| Read-only root + tmp | `--read-only --tmpfs /tmp` |
| Back up a volume | `docker run --rm -v pgdata:/data -v "$PWD":/b alpine tar czf /b/pgdata.tgz -C /data .` |
| Restore a volume | `docker run --rm -v pgdata:/data -v "$PWD":/b alpine tar xzf /b/pgdata.tgz -C /data` |
| Postgres logical dump | `docker exec db pg_dump -U user -Fc dbname > db.dump` |
| Remove unused volumes | `docker volume prune` |

Bind mount `-v` creates a missing host directory as root; `--mount` errors instead. Prefer `--mount` in scripts.

## Docker Compose

| Task | Command |
|------|---------|
| Up detached with build | `docker compose up -d --build` |
| Up without building | `docker compose up -d --no-build` |
| Recreate only changed | `docker compose up -d` |
| Force recreate | `docker compose up -d --force-recreate` |
| Down (keep volumes) | `docker compose down` |
| Down + volumes + orphans | `docker compose down -v --remove-orphans` |
| Status | `docker compose ps -a` |
| Logs of one service | `docker compose logs -f --tail 100 api` |
| Exec / one-off run | `docker compose exec api sh`, `docker compose run --rm api npm test` |
| Scale | `docker compose up -d --scale worker=3` |
| Pull / build / push | `docker compose pull`, `docker compose build --pull`, `docker compose push` |
| Validate and show merged config | `docker compose config` (use `--quiet` in CI) |
| Several files | `docker compose -f compose.yaml -f compose.prod.yaml up -d` |
| Profile | `docker compose --profile debug up -d` |
| Watch mode (dev) | `docker compose watch` |
| Wait for healthy | `docker compose up -d --wait` |

```yaml
name: myapp
services:
  api:
    build: ./api
    image: ${REGISTRY}/api:${TAG:-dev}
    env_file: [./env/api.env]
    environment:
      DB_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    depends_on:
      db: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3000/healthz"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    deploy:
      resources:
        limits: { cpus: "0.5", memory: 256M }
    networks: [frontend, backend]
  db:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
    networks: [backend]
networks:
  frontend:
  backend:
    internal: true
volumes:
  pgdata:
```

Variable rules: `.env` feeds `${VAR}` interpolation in `compose.yaml`; `env_file:` and `environment:` feed the container. Escape a literal `$` as `$$`. Precedence (high to low): `docker compose run -e`, `environment:`, `env_file:`, image `ENV`.

## Registry and tagging

| Task | Command |
|------|---------|
| Login (token via stdin) | `echo "$TOKEN" \| docker login ghcr.io -u USER --password-stdin` |
| Logout | `docker logout ghcr.io` |
| Push all tags | `docker push --all-tags ghcr.io/user/app` |
| Get digest | `docker inspect --format '{{index .RepoDigests 0}}' ghcr.io/user/app:1.0.0` |
| Pull by digest | `docker pull ghcr.io/user/app@sha256:<digest>` |
| List remote tags | `crane ls ghcr.io/user/app` |
| Copy between registries | `crane copy src dst` |
| Local registry | `docker run -d -p 5000:5000 --name registry registry:2` |

Reference format: `[registry[:port]/]namespace/name[:tag][@sha256:digest]`.

Tag scheme we use in production: `1.4.2` (immutable release), `1.4` (moving minor), `sha-abc1234` (traceability, unique per commit), `main` or `edge` (moving, non-prod only). No `latest` in deploy manifests. Deploy by digest when you need a guarantee.

## Security hardening

| Control | Flag or setting |
|---------|-----------------|
| Non-root | `USER 10001:10001` or `--user 10001:10001` |
| Read-only root FS | `--read-only --tmpfs /tmp` |
| Drop all capabilities | `--cap-drop ALL` (add back, e.g. `--cap-add NET_BIND_SERVICE`) |
| Block privilege escalation | `--security-opt no-new-privileges:true` |
| Default seccomp profile | on by default; custom: `--security-opt seccomp=profile.json` |
| Resource limits | `--memory 256m --cpus 0.5 --pids-limit 200` |
| Never | `--privileged`, mounting `/var/run/docker.sock`, `--network host` unless required |
| Secrets | BuildKit `--mount=type=secret`; at runtime Compose `secrets:` or a secret manager, not `ENV` |
| Pin bases | `FROM node:22.11-alpine3.20` (or `@sha256:`) |
| User namespace remap | `"userns-remap": "default"` in `/etc/docker/daemon.json` |
| Rootless Docker | `dockerd-rootless-setuptool.sh install` |
| Image trust | `cosign sign`, `cosign verify`, deploy by digest |

## Trivy

| Task | Command |
|------|---------|
| Scan image, gate for CI | `trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app:1.0` |
| Scan with secrets and misconfig | `trivy image --scanners vuln,secret,misconfig app:1.0` |
| Scan Dockerfile / Compose / IaC | `trivy config .` |
| Scan source dependencies | `trivy fs --scanners vuln,secret .` |
| SBOM (CycloneDX / SPDX) | `trivy image --format cyclonedx -o sbom.json app:1.0` and `--format spdx-json` |
| Scan an SBOM later | `trivy sbom sbom.json` |
| JSON report | `trivy image -f json -o report.json app:1.0` |
| Ignore file | `.trivyignore` with one CVE per line, reason in a comment |
| Refresh DB | `trivy image --download-db-only` |
| Run as container | `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy image app:1.0` |

## Cleanup

| Task | Command |
|------|---------|
| Remove stopped containers | `docker container prune -f` |
| Remove dangling images | `docker image prune -f` |
| Remove all unused images | `docker image prune -a -f` |
| Remove unused networks | `docker network prune -f` |
| Remove unused volumes | `docker volume prune -f` |
| Everything unused (no volumes) | `docker system prune -f` |
| Everything incl. volumes | `docker system prune -a --volumes -f` (destructive) |
| Remove all containers | `docker rm -f $(docker ps -aq)` |
| Disk usage | `docker system df` |
| Log rotation | `--log-opt max-size=10m --log-opt max-file=3` |

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Clean exit |
| 1 | Application error (read the logs) |
| 125 | `docker run` itself failed (bad flag, daemon error) |
| 126 | Command found but not executable (permissions) |
| 127 | Command not found (wrong `CMD` or missing binary) |
| 137 | SIGKILL: OOM kill, `docker kill`, or stop timeout exceeded |
| 139 | SIGSEGV: segmentation fault |
| 143 | SIGTERM: graceful stop |

## Frequent error messages

| Message | Meaning and quick fix |
|---------|-----------------------|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Daemon down or no permission: `systemctl status docker`, add user to `docker` group |
| `permission denied while trying to connect to the Docker daemon socket` | Not in `docker` group; `sudo usermod -aG docker $USER` and re-login |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | Host port taken: `ss -ltnp \| grep :80` |
| `exec format error` | Wrong CPU architecture or missing shebang: build with `--platform` |
| `no space left on device` | `docker system df`, then prune; check `/var/lib/docker` |
| `pull access denied` / `manifest unknown` | Wrong name or tag, or not logged in |
| `OCI runtime create failed ... executable file not found in $PATH` | Bad `CMD`/`ENTRYPOINT` or distroless without shell |
| `network ... not found` / `Pool overlaps with other one` | Stale network or subnet clash: `docker network prune`, choose another subnet |

---

## Top 15 interview one-liners

1. **Container vs VM:** a container is an isolated process sharing the host kernel (namespaces plus cgroups); a VM runs its own kernel on virtualized hardware.
2. **Image vs container:** an image is an immutable stack of layers plus config; a container is an image plus a writable layer plus a running process.
3. **CMD vs ENTRYPOINT:** `ENTRYPOINT` is the executable, `CMD` is its default arguments; `docker run img args` replaces `CMD` and appends to `ENTRYPOINT`.
4. **COPY vs ADD:** use `COPY`; `ADD` also fetches URLs and auto-extracts archives, which is surprising and less auditable.
5. **Shrinking an image:** multi-stage build, small base (alpine, slim, distroless), `.dockerignore`, no package caches, combine and clean up inside one `RUN`.
6. **Why order matters:** cache is invalidated at the first changed layer, so copy dependency manifests and install before copying source.
7. **Data persistence:** use named volumes for databases, bind mounts for dev, tmpfs for scratch; container filesystem is ephemeral.
8. **Container-to-container communication:** put them on a user-defined network and use service names, because Docker's embedded DNS only exists there.
9. **EXPOSE vs `-p`:** `EXPOSE` is documentation; `-p host:container` actually publishes the port through iptables DNAT.
10. **Run as non-root:** set a numeric `USER`, drop capabilities, `no-new-privileges`, read-only root filesystem, ports above 1024.
11. **Secrets:** never in `ENV`, `ARG` or `COPY`; they stay in `docker history`. Use BuildKit secret mounts at build time and secret managers or Compose secrets at runtime.
12. **Exit code 137:** SIGKILL, usually the OOM killer; confirm with `docker inspect -f '{{.State.OOMKilled}}'` and raise the limit or fix the leak.
13. **`depends_on` pitfall:** it orders start, not readiness; add a `healthcheck` and `condition: service_healthy`.
14. **Tagging in production:** immutable semver plus git SHA tag, deploy by version or digest, never `latest`; scan with Trivy and fail on fixable HIGH/CRITICAL.
15. **Why `docker stop` is slow:** the app is ignoring `SIGTERM` (often because a shell is PID 1); use exec-form `CMD`, handle the signal, or add `--init`.

---

[⬅ Back to Docker & Containers](./README.md) · [🏠 Home](../README.md)

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
