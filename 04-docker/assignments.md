# Docker & Containers — Assignments

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

> 25 practical assignments: 10 🟢 Beginner, 10 🟡 Intermediate, 5 🔴 Advanced. Do them on a real machine; reading is not enough.

[⬅ Back to Docker & Containers](./README.md) · [🏠 Home](../README.md)

## Index

| # | Level | Assignment |
|---|-------|------------|
| 1 | 🟢 | Run your first containers |
| 2 | 🟢 | Images, layers and inspection |
| 3 | 🟢 | Your first Dockerfile |
| 4 | 🟢 | Port publishing and environment variables |
| 5 | 🟢 | Volumes vs bind mounts |
| 6 | 🟢 | `.dockerignore` and the build context |
| 7 | 🟢 | `CMD` vs `ENTRYPOINT` |
| 8 | 🟢 | User-defined networks and DNS |
| 9 | 🟢 | Your first Compose file |
| 10 | 🟢 | Tag and push to a registry |
| 11 | 🟡 | Multi-stage build and image size |
| 12 | 🟡 | Layer cache optimization |
| 13 | 🟡 | Non-root and read-only containers |
| 14 | 🟡 | Healthchecks and health-gated startup |
| 15 | 🟡 | Compose env files, profiles and overrides |
| 16 | 🟡 | Resource limits and the OOM killer |
| 17 | 🟡 | Scan and fix with Trivy |
| 18 | 🟡 | Back up and restore a database volume |
| 19 | 🟡 | Debug a crash-looping container |
| 20 | 🟡 | BuildKit cache and secret mounts |
| 21 | 🔴 | Multi-architecture images with buildx |
| 22 | 🔴 | Distroless and scratch images |
| 23 | 🔴 | Build a container by hand with namespaces and cgroups |
| 24 | 🔴 | CI pipeline: build, scan, sign, SBOM, push |
| 25 | 🔴 | Private registry with TLS, auth and promotion by digest |

---

## 🟢 Beginner

### Assignment 1. Run your first containers 🟢

**Objective:** Get comfortable with the container lifecycle commands.

**Tasks**
1. Run `nginx:alpine` detached, named `web1`, with host port 8081 mapped to container port 80.
2. Fetch the page with `curl`, then read the container's logs.
3. Open a shell inside it with `docker exec` and list `/usr/share/nginx/html`.
4. Stop it, list all containers including stopped ones, restart it, then remove it.
5. Run `alpine echo hello` with `--rm` and confirm no container is left behind.

**Expected outcome / acceptance criteria**
- `curl localhost:8081` returns the nginx welcome page.
- `docker ps -a` shows no leftover containers after step 5.
- You can explain the difference between `stop`, `kill` and `rm`.

<details>
<summary>Hints</summary>

`docker run -d --name web1 -p 8081:80 nginx:alpine`, `docker logs web1`, `docker exec -it web1 sh`, `docker ps -a`. `--rm` deletes the container when the process exits.

</details>

**Bonus:** Run `docker stats --no-stream` and `docker top web1`; identify the PID of the nginx master process on the host using `docker inspect -f '{{.State.Pid}}' web1`.

### Assignment 2. Images, layers and inspection 🟢

**Objective:** Understand that an image is a stack of layers plus metadata.

**Tasks**
1. Pull `alpine:3.20`, `python:3.12-slim`, and `python:3.12`; compare sizes.
2. Run `docker history` on each and identify the largest layer.
3. Use `docker inspect` to find the default `Cmd`, `Env`, and `Architecture` of `python:3.12-slim`.
4. Pull the same image by digest (get it from `docker images --digests`).
5. Delete the images and note which layers were shared.

**Expected outcome / acceptance criteria**
- A short note with the three sizes and the reason for the difference (base OS packages and toolchains).
- You can pull by digest and explain why a digest is immutable but a tag is not.

<details>
<summary>Hints</summary>

`docker images --digests`, `docker history --no-trunc`, `docker inspect --format '{{json .Config}}' python:3.12-slim | jq`. Pull by digest: `docker pull python@sha256:<digest>`.

</details>

**Bonus:** Install `dive` and explore `python:3.12` layer by layer; list the three biggest files.

### Assignment 3. Your first Dockerfile 🟢

**Objective:** Package a static site into an image.

**Tasks**
1. Create `index.html` with any content.
2. Write a Dockerfile `FROM nginx:alpine` that copies it to `/usr/share/nginx/html/`.
3. Build it as `mysite:1.0` and run it on port 8082.
4. Change the HTML, rebuild as `mysite:1.1`, and watch which layers are rebuilt versus cached.
5. Run both versions side by side on ports 8082 and 8083.

**Expected outcome / acceptance criteria**
- Two images, two running containers, different content.
- Build output shows `CACHED` for the base layer on the second build.

<details>
<summary>Hints</summary>

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```
`docker build -t mysite:1.0 .`

</details>

**Bonus:** Add `LABEL org.opencontainers.image.authors` and view it with `docker inspect`.

### Assignment 4. Port publishing and environment variables 🟢

**Objective:** Configure containers at runtime without rebuilding.

**Tasks**
1. Run `postgres:16-alpine` with `POSTGRES_PASSWORD` set via `-e`, without publishing a port. Try connecting from the host and observe the failure.
2. Re-run publishing `127.0.0.1:5433:5432` and connect with `psql` (or `docker exec ... psql`).
3. Run `docker run --rm -e GREETING=hi alpine sh -c 'echo $GREETING'`.
4. Move the variables into an env file and use `--env-file`.
5. Run the container with `-P` and find which random host port was assigned.

**Expected outcome / acceptance criteria**
- Connection fails without `-p`, succeeds with it, and is bound only to localhost.
- You can explain `0.0.0.0:5433` vs `127.0.0.1:5433` and why the second is safer.

<details>
<summary>Hints</summary>

`docker port <name>` shows mappings. `--env-file ./db.env` takes `KEY=value` lines without quotes or `export`.

</details>

**Bonus:** Inspect `iptables -t nat -L DOCKER -n` and find the DNAT rule created for your published port.

### Assignment 5. Volumes vs bind mounts 🟢

**Objective:** Prove which storage type survives container deletion.

**Tasks**
1. Run a container that writes a file to `/data` with no volume. Remove it, run a new one, and confirm the file is gone.
2. Repeat with a named volume `mydata` and confirm the file survives.
3. Repeat with a bind mount of `$(pwd)/hostdata` and compare file ownership on the host.
4. Inspect the volume and locate its mountpoint.
5. Mount a bind mount as read-only (`:ro`) and show that writes fail.

**Expected outcome / acceptance criteria**
- A table of the three cases with "persists? yes/no" and who owns the files.
- `touch: ... Read-only file system` observed for the `:ro` mount.

<details>
<summary>Hints</summary>

`docker run --rm -v mydata:/data alpine sh -c 'echo hi > /data/f'`; `docker volume inspect mydata`; `-v "$PWD/hostdata:/data:ro"` or `--mount type=bind,src=...,dst=/data,readonly`.

</details>

**Bonus:** Use a `tmpfs` mount and verify that data disappears when the container stops.

### Assignment 6. `.dockerignore` and the build context 🟢

**Objective:** See how the build context affects speed and security.

**Tasks**
1. Create a project directory containing a 200 MB dummy file (`dd if=/dev/zero of=big.bin bs=1M count=200`) and a fake `.env` with a secret.
2. Build with a Dockerfile that only copies `app.txt` and note the "transferring context" size.
3. Add a `.dockerignore` excluding `big.bin`, `.env`, `.git`, and rebuild.
4. Write a Dockerfile with `COPY . .` first without `.dockerignore`, then verify the secret is inside the image.

**Expected outcome / acceptance criteria**
- Context size drops from about 200 MB to a few KB.
- You demonstrate `docker run --rm img cat /app/.env` leaking the secret, then show the fix.

<details>
<summary>Hints</summary>

Build with `--progress=plain` to see context transfer. `.dockerignore` patterns are relative to the context root.

</details>

**Bonus:** Use `docker build --no-cache --progress=plain` and explain why even a deleted secret in an earlier layer is still extractable.

### Assignment 7. `CMD` vs `ENTRYPOINT` 🟢

**Objective:** Predict exactly what runs for each combination.

**Tasks**
1. Build three images that print their arguments: only `CMD ["echo","default"]`; only `ENTRYPOINT ["echo"]`; both `ENTRYPOINT ["echo"]` and `CMD ["default"]`.
2. Run each with no arguments and with `extra args`.
3. Create a fourth image using shell form (`CMD echo $HOME`) and show it runs through `/bin/sh -c`.
4. Override the entrypoint at runtime with `--entrypoint sh`.

**Expected outcome / acceptance criteria**
- A 3x2 table of outputs that you predicted before running.
- You can explain why exec form is required for signal handling.

<details>
<summary>Hints</summary>

`docker run img extra args` replaces `CMD` and is appended to `ENTRYPOINT`. In shell form PID 1 is `sh`, check with `docker exec c ps`.

</details>

**Bonus:** Show that `docker stop` takes 10 seconds with a shell-form `sleep` command and returns instantly with exec form plus a trap or `--init`.

### Assignment 8. User-defined networks and DNS 🟢

**Objective:** Show service discovery by name.

**Tasks**
1. Start two containers on the default bridge and try `ping` by name. Observe failure.
2. Create `docker network create appnet`, start the same two containers on it, and ping by name.
3. Run `nginx` on `appnet` and `curl http://<name>` from a second container.
4. Inspect the network to see subnet and attached containers.
5. Attach a running container to a second network with `docker network connect`.

**Expected outcome / acceptance criteria**
- Default bridge: `ping: bad address`; user-defined network: replies.
- You can state where the embedded DNS lives (`127.0.0.11`) and why it needs a user-defined network.

<details>
<summary>Hints</summary>

`docker run -dit --name a --network appnet alpine sh`, `docker exec a ping -c1 b`, `docker network inspect appnet`, `cat /etc/resolv.conf` in the container.

</details>

**Bonus:** Create a network with `--internal` and prove containers on it cannot reach the internet.

### Assignment 9. Your first Compose file 🟢

**Objective:** Describe a two-service app declaratively.

**Tasks**
1. Write `compose.yaml` with `web` (nginx, port 8080) and `cache` (redis:7-alpine).
2. Run `docker compose up -d` and verify with `docker compose ps`.
3. From `web`, resolve `cache` by name.
4. View logs with `docker compose logs -f cache`.
5. Tear down with `docker compose down` and confirm networks and containers are gone.

**Expected outcome / acceptance criteria**
- One command brings both up and one command removes both.
- `docker compose exec web ping -c1 cache` works.

<details>
<summary>Hints</summary>

Compose creates a default network named `<project>_default`. Run `docker compose config` to see the resolved file.

</details>

**Bonus:** Add a named volume to redis and prove that data survives `down` but not `down -v`.

### Assignment 10. Tag and push to a registry 🟢

**Objective:** Publish an image with a sensible tag scheme.

**Tasks**
1. Create a free account on Docker Hub or GHCR and log in with an access token (not your password).
2. Tag `mysite:1.0` as `<registry>/<user>/mysite:1.0` and `:1`.
3. Push both tags and verify them on the registry website.
4. Remove local images, pull from the registry, and run it.
5. Explain why you should not rely on `:latest`.

**Expected outcome / acceptance criteria**
- Both tags visible on the registry and both resolve to the same digest.
- A pull on a clean machine (or after `docker rmi`) runs the app.

<details>
<summary>Hints</summary>

`echo $TOKEN | docker login -u USER --password-stdin`, `docker tag src dst`, `docker push dst`. GHCR needs lowercase names.

</details>

**Bonus:** Make the repository private and show the `pull access denied` error when logged out.

---

## 🟡 Intermediate

### Assignment 11. Multi-stage build and image size 🟡

**Objective:** Cut an image by at least 80% without changing behaviour.

**Tasks**
1. Write a small Go (or Node/Python) HTTP service with a single-stage Dockerfile on `golang:1.23` (or the full language image) and record the size.
2. Convert it to a multi-stage build that copies only the binary (or production dependencies) into a minimal runtime stage.
3. Record the new size and run both to prove identical responses.
4. Add `--target` builds: a `test` stage that runs unit tests and a `runtime` stage.

**Expected outcome / acceptance criteria**
- Final image at least 80% smaller (for Go: under 20 MB on alpine).
- `docker build --target test .` runs tests; the default build does not include test files.

<details>
<summary>Hints</summary>

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM alpine:3.20
COPY --from=build /out/app /app
USER 10001
ENTRYPOINT ["/app"]
```

</details>

**Bonus:** Add a third stage `FROM scratch` and compare size and debuggability.

### Assignment 12. Layer cache optimization 🟡

**Objective:** Make rebuilds after a code change take seconds.

**Tasks**
1. Take a Node or Python app whose Dockerfile does `COPY . .` before installing dependencies. Time a rebuild after editing one source file.
2. Reorder: copy the dependency manifest first, install, then copy the source. Time it again.
3. Use `RUN --mount=type=cache` for the package manager cache.
4. Find a `RUN apt-get update` followed by a separate `RUN apt-get install` and merge them correctly, cleaning lists in the same layer.

**Expected outcome / acceptance criteria**
- Before/after timings recorded; the post-edit rebuild skips the dependency install.
- You can explain why `apt-get update` alone in a layer causes stale package indexes.

<details>
<summary>Hints</summary>

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```
Use `docker build --progress=plain` to see `CACHED` markers.

</details>

**Bonus:** Add `--cache-from`/`--cache-to type=registry` and show a cold CI-like build reusing remote cache.

### Assignment 13. Non-root and read-only containers 🟡

**Objective:** Apply the container hardening baseline to a real image.

**Tasks**
1. Show that a default `python:3.12-slim` container runs as root (`id`).
2. Create a dedicated user (`adduser`/`useradd` with a fixed UID such as 10001) and set `USER` in the Dockerfile; fix file ownership with `COPY --chown`.
3. Run with `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges`.
4. Make the app listen on a port above 1024 and confirm it works without `NET_BIND_SERVICE`.
5. Demonstrate that writing outside `/tmp` fails.

**Expected outcome / acceptance criteria**
- `docker exec c id` returns a non-zero UID.
- The app serves traffic under all hardening flags; `docker inspect` shows `ReadonlyRootfs: true` and `CapDrop: [ALL]`.

<details>
<summary>Hints</summary>

On Alpine: `addgroup -S app -g 10001 && adduser -S app -u 10001 -G app`. Debian: `useradd --uid 10001 --no-create-home --shell /usr/sbin/nologin app`. If the app needs a writable cache directory, point it to `/tmp` via an env var.

</details>

**Bonus:** Run the same container on Kubernetes-style settings: `--user 12345:12345` with an arbitrary UID and make it still work.

### Assignment 14. Healthchecks and health-gated startup 🟡

**Objective:** Make startup order depend on actual readiness.

**Tasks**
1. Write a Compose stack with `postgres:16-alpine` and an app that connects to it.
2. Add a `pg_isready` healthcheck on the database and an HTTP healthcheck on the app.
3. Use `depends_on: condition: service_healthy`.
4. Compare with plain `depends_on` by adding `sleep` to the DB entrypoint and observing the app crash.
5. Break the app's health endpoint and watch the status change to `unhealthy` in `docker ps`.

**Expected outcome / acceptance criteria**
- With health-gating, the app starts only after the database reports `healthy`.
- `docker inspect --format '{{json .State.Health}}'` shows the failing log entries.

<details>
<summary>Hints</summary>

`test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]`; use `start_period` so slow boots do not count as failures. Healthchecks do not restart containers by themselves in plain Docker.

</details>

**Bonus:** Add `restart: unless-stopped` plus a small "autoheal" sidecar, or explain why Kubernetes liveness probes replace this.

### Assignment 15. Compose env files, profiles and overrides 🟡

**Objective:** Run one codebase in dev and prod modes cleanly.

**Tasks**
1. Create `compose.yaml` (base), `compose.override.yaml` (dev: bind-mount source, debug ports) and `compose.prod.yaml` (no bind mounts, restart policies, resource limits).
2. Use `.env` for interpolation and `env_file` for container variables; show the difference.
3. Add an `adminer` service under `profiles: ["debug"]`.
4. Run `docker compose up -d`, `docker compose --profile debug up -d`, and `docker compose -f compose.yaml -f compose.prod.yaml up -d`.
5. Use `docker compose config` to prove the merged result.

**Expected outcome / acceptance criteria**
- `adminer` starts only with the profile; prod config contains no bind mounts.
- A required variable uses `${VAR:?message}` and fails fast when missing.

<details>
<summary>Hints</summary>

`docker compose` automatically merges `compose.override.yaml`. With `-f`, you must list every file yourself. Lists such as `ports` are appended on merge; use `!reset` (Compose 2.24+) to clear them.

</details>

**Bonus:** Use Compose `secrets:` with a file source and read it from `/run/secrets/<name>`.

### Assignment 16. Resource limits and the OOM killer 🟡

**Objective:** See memory and CPU limits enforced.

**Tasks**
1. Run a container with `--memory 64m --memory-swap 64m` that allocates memory in a loop (for example `python -c "a=[]\nwhile True: a.append(bytearray(10**6))"`).
2. Confirm exit code 137 and `OOMKilled: true`.
3. Run a CPU burner with `--cpus 0.5` and verify via `docker stats` that it stays near 50%.
4. Set `--pids-limit 50` and run a fork bomb-like loop safely inside the container.
5. Read the matching cgroup files on the host.

**Expected outcome / acceptance criteria**
- `docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}'` prints `true 137`.
- You can explain why limits matter for noisy neighbours and why JVM/Node need to be told about the limit.

<details>
<summary>Hints</summary>

cgroup v2 files: `/sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max` and `cpu.max`. `dmesg | grep -i 'killed process'` shows the kernel message.

</details>

**Bonus:** Run a Node app with `--max-old-space-size` set to 75% of the container limit and compare behaviour.

### Assignment 17. Scan and fix with Trivy 🟡

**Objective:** Reduce real vulnerabilities, not just collect reports.

**Tasks**
1. Scan `node:18` (or another old image) with `trivy image --severity HIGH,CRITICAL`.
2. Scan your own multi-stage image from the lab and compare counts.
3. Fix at least five findings by changing base image, bumping dependencies, or removing packages.
4. Run `trivy config` on your Dockerfile and Compose file and fix at least two misconfigurations.
5. Add `--exit-code 1 --ignore-unfixed` to produce a CI-style gate and a `.trivyignore` entry with a reason and date.

**Expected outcome / acceptance criteria**
- A before/after table of finding counts per severity.
- Gate command exits 0 on the fixed image and 1 on the old image.

<details>
<summary>Hints</summary>

`trivy image --format table --severity HIGH,CRITICAL <img>`, `trivy image --scanners vuln,secret,misconfig <img>`. Prefer `-slim`/alpine/distroless bases, and rebuild with `--pull`.

</details>

**Bonus:** Generate a CycloneDX SBOM and query it with `jq` for a specific package.

### Assignment 18. Back up and restore a database volume 🟡

**Objective:** Prove you can recover a stateful container.

**Tasks**
1. Start Postgres with a named volume and insert sample data.
2. Make a logical backup with `pg_dump` and a physical copy of the volume using a throwaway `tar` container.
3. Delete the container and volume, create a new volume, and restore from each backup.
4. Verify row counts match.
5. Document the recovery steps in a short runbook.

**Expected outcome / acceptance criteria**
- Data restored byte for byte (row counts and a checksum query).
- Runbook lists commands, expected duration, and when to prefer logical vs physical backups.

<details>
<summary>Hints</summary>

```bash
docker exec db pg_dump -U shack -Fc shackdb > backup.dump
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine tar czf /backup/pgdata.tgz -C /data .
```
Physical copy of a running database may be inconsistent; stop the container or use `pg_basebackup`. Restore logical with `pg_restore -U shack -d shackdb --clean backup.dump`.

</details>

**Bonus:** Schedule the logical backup with a Compose service on a cron-like loop and ship it to S3-compatible storage (MinIO).

### Assignment 19. Debug a crash-looping container 🟡

**Objective:** Diagnose from symptoms using only Docker tooling.

**Tasks**
1. Start the broken stack below and find out why `api` keeps restarting, without reading the source first.
2. Find the exit code, last logs, and the exact failing command.
3. Fix it and write a 3-line root-cause summary.
4. Repeat for two more failures you introduce yourself: wrong `CMD` path, and a missing environment variable.

```yaml
services:
  api:
    image: node:22-alpine
    command: ["node", "/app/server.js"]
    restart: always
    environment:
      DB_HOST: database
```

**Expected outcome / acceptance criteria**
- You identify `Cannot find module '/app/server.js'` (exit 1) with `docker logs` and `docker inspect`.
- Three distinct failure types diagnosed, each with the commands you used.

<details>
<summary>Hints</summary>

`docker ps -a`, `docker logs --tail 50 <c>`, `docker inspect -f '{{.State.ExitCode}} {{.State.Error}} {{.RestartCount}}' <c>`, `docker events --since 5m`. To poke around a crashing container: `docker run --rm -it --entrypoint sh <image>`.

</details>

**Bonus:** Explain exit codes 0, 1, 125, 126, 127, 137, 139, 143.

### Assignment 20. BuildKit cache and secret mounts 🟡

**Objective:** Use credentials during a build without leaving them in the image.

**Tasks**
1. Write a Dockerfile that installs a private-style dependency using a token with `ARG TOKEN` and show that `docker history` exposes it.
2. Rewrite it with `RUN --mount=type=secret,id=npm_token` and pass `--secret id=npm_token,src=./token.txt`.
3. Prove the secret is absent from `docker history` and from the filesystem of the final image.
4. Add `--mount=type=cache` for the package cache and compare rebuild times.

**Expected outcome / acceptance criteria**
- `docker history --no-trunc` leaks the token in version 1 and not in version 2.
- Searching the image filesystem (`docker save | tar -x` then `grep -r`) finds nothing.

<details>
<summary>Hints</summary>

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```
Secret mounts are available only during that `RUN` and are not stored in layers.

</details>

**Bonus:** Use `--mount=type=ssh` with `docker build --ssh default` to clone a private Git repository.

---

## 🔴 Advanced

### Assignment 21. Multi-architecture images with buildx 🔴

**Objective:** Publish one tag that works on amd64 and arm64.

**Tasks**
1. Create a buildx builder with the `docker-container` driver.
2. Build and push `linux/amd64,linux/arm64` for your app to a registry.
3. Inspect the manifest list and verify both platform entries.
4. Run the arm64 variant on an amd64 host through QEMU (`tonistiigi/binfmt`) and confirm `uname -m`.
5. Compare build time of emulated builds with cross-compilation using `--platform=$BUILDPLATFORM` and `TARGETARCH` in a Go Dockerfile.

**Expected outcome / acceptance criteria**
- `docker buildx imagetools inspect <image>` lists both architectures under one tag.
- Cross-compiled build is measurably faster than emulation.

<details>
<summary>Hints</summary>

```bash
docker buildx create --name multi --driver docker-container --use
docker run --privileged --rm tonistiigi/binfmt --install all
docker buildx build --platform linux/amd64,linux/arm64 -t REG/app:1.0 --push .
docker run --rm --platform linux/arm64 REG/app:1.0 uname -m
```
In the Dockerfile: `FROM --platform=$BUILDPLATFORM golang:1.23 AS build` then `GOOS=linux GOARCH=$TARGETARCH go build`.

</details>

**Bonus:** Add registry-backed cache (`--cache-to type=registry,mode=max`) and measure the second build.

### Assignment 22. Distroless and scratch images 🔴

**Objective:** Ship the smallest runnable image and still be able to debug it.

**Tasks**
1. Build a static Go binary (`CGO_ENABLED=0`) and run it from `FROM scratch`. Fix the missing CA certificates and timezone data problems you hit.
2. Build the same app on `gcr.io/distroless/static-debian12:nonroot` and a Node app on `gcr.io/distroless/nodejs22-debian12:nonroot`.
3. Compare sizes and Trivy findings across alpine, distroless and scratch.
4. Debug the distroless container using `docker debug` or an ephemeral `nicolaka/netshoot` sidecar sharing its namespaces.
5. Add a healthcheck without a shell or `curl` (hint: the app binary gets a `--healthcheck` flag).

**Expected outcome / acceptance criteria**
- Scratch/distroless image under 15 MB for Go and runs as non-root.
- You can reach into the running container with a debug tool even though it has no shell.

<details>
<summary>Hints</summary>

Copy certs: `COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`. For a numeric user in scratch: `USER 10001:10001`. Debug sidecar: `docker run --rm -it --pid container:app --net container:app nicolaka/netshoot`.

</details>

**Bonus:** Write a short decision guide: when would you not use scratch or distroless?

### Assignment 23. Build a container by hand with namespaces and cgroups 🔴

**Objective:** Demystify containers by creating one without Docker.

**Tasks**
1. Export an Alpine root filesystem: `docker export $(docker create alpine) | tar -x -C rootfs`.
2. Use `unshare` with PID, mount, UTS, IPC and network namespaces and `chroot` into `rootfs`; mount `/proc`.
3. Show that PID 1 inside differs from the host PID, and change the hostname without affecting the host.
4. Create a cgroup v2 group, set `memory.max` to 50 MB and `pids.max` to 20, put your shell into it, and trigger the limits.
5. Connect the network namespace to the host with a veth pair and a bridge or NAT rule so it can reach the internet.

**Expected outcome / acceptance criteria**
- `ps` inside shows only your processes; `hostname` differs from the host.
- Exceeding `memory.max` kills the process; `pids.max` makes `fork` fail.
- A short write-up mapping each step to the Docker feature it implements.

<details>
<summary>Hints</summary>

```bash
sudo unshare --pid --fork --mount --uts --ipc --net --mount-proc=rootfs/proc chroot rootfs /bin/sh
sudo mkdir /sys/fs/cgroup/demo
echo 50M | sudo tee /sys/fs/cgroup/demo/memory.max
echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs
```
Veth: `ip link add veth0 type veth peer name veth1`, move one end with `ip link set veth1 netns <pid>`, then address, route and `iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -j MASQUERADE`. Use `pivot_root` instead of `chroot` for the bonus.

</details>

**Bonus:** Add a user namespace (`--user --map-root-user`) and explain how rootless containers use it.

### Assignment 24. CI pipeline: build, scan, sign, SBOM, push 🔴

**Objective:** Produce a trustworthy, traceable image from every merge.

**Tasks**
1. Create a GitHub Actions workflow that builds with `docker/build-push-action`, using `docker/metadata-action` to generate tags (semver on git tags, `sha-<short>` on every commit).
2. Run Trivy as a gate (`exit-code: 1`, `severity: HIGH,CRITICAL`, `ignore-unfixed: true`) before the push.
3. Push to GHCR using the built-in `GITHUB_TOKEN` with least-privilege `permissions`.
4. Generate an SBOM and provenance attestation (`sbom: true`, `provenance: mode=max`) and sign the image keylessly with `cosign`.
5. Verify the signature locally with `cosign verify` and fail a deployment script when verification fails.

**Expected outcome / acceptance criteria**
- A push of tag `v1.2.3` creates `1.2.3`, `1.2`, and `sha-xxxxxxx` tags; a vulnerable base image makes the pipeline fail before push.
- `cosign verify` succeeds with the expected certificate identity and issuer.

<details>
<summary>Hints</summary>

Permissions block: `contents: read`, `packages: write`, `id-token: write` (for keyless signing). Sign by digest, not tag: `cosign sign --yes ghcr.io/org/app@${{ steps.build.outputs.digest }}`. Verify with `--certificate-identity-regexp` and `--certificate-oidc-issuer https://token.actions.githubusercontent.com`. Cache with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.

</details>

**Bonus:** Add a scheduled nightly workflow that re-scans the latest released image and opens an issue on new HIGH findings.

### Assignment 25. Private registry with TLS, auth and promotion by digest 🔴

**Objective:** Run and operate your own registry and promote images between environments safely.

**Tasks**
1. Run `registry:2` with TLS (self-signed CA or a real certificate) and htpasswd basic auth, storing data on a named volume.
2. Push an image tagged `dev-<sha>`; log in from a second host or container that trusts your CA.
3. Implement promotion: after tests pass, retag the same digest as `staging`, then `prod-1.0.0`, with `docker buildx imagetools create` or `crane copy`, without rebuilding.
4. Prove that `staging` and `prod-1.0.0` have identical digests and that deployments reference the digest.
5. Enable deletion in the registry config, delete an old tag, and run garbage collection to reclaim space.
6. Configure a pull-through cache (`proxy.remoteurl`) for Docker Hub and set `registry-mirrors` in `daemon.json`.

**Expected outcome / acceptance criteria**
- Unauthenticated pull fails with `no basic auth credentials`; authenticated pull works over HTTPS.
- Digest equality demonstrated; disk usage drops after garbage collection.

<details>
<summary>Hints</summary>

`htpasswd -Bbn user pass > auth/htpasswd`. Env vars: `REGISTRY_AUTH=htpasswd`, `REGISTRY_AUTH_HTPASSWD_REALM=Registry`, `REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd`, `REGISTRY_HTTP_TLS_CERTIFICATE`, `REGISTRY_HTTP_TLS_KEY`, `REGISTRY_STORAGE_DELETE_ENABLED=true`. GC: `docker exec registry registry garbage-collect /etc/docker/registry/config.yml`. Trust the CA by placing it in `/etc/docker/certs.d/<host:port>/ca.crt`.

</details>

**Bonus:** Replace `registry:2` with Harbor or an ECR/GAR repository and add tag immutability plus a lifecycle policy.

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
