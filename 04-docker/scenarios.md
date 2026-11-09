# Docker — Scenario-Based Questions

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

> Real-world Docker situations and how to reason through them in an interview · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Docker](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Image design and Dockerfile decisions](#image-design-and-dockerfile-decisions) — Q1–Q25
- [Multi-stage builds, cache, tags and registries](#multi-stage-builds-cache-tags-and-registries) — Q26–Q50
- [Volumes, storage and data](#volumes-storage-and-data) — Q51–Q75
- [Networking and Compose](#networking-and-compose) — Q76–Q100
- [Security and scanning](#security-and-scanning) — Q101–Q125
- [Limits, logging, health checks and restarts](#limits-logging-health-checks-and-restarts) — Q126–Q150
- [Runtimes: containerd, OCI, rootless and Podman](#runtimes-containerd-oci-rootless-and-podman) — Q151–Q175
- [CI/CD and production rollouts](#ci-cd-and-production-rollouts) — Q176–Q200

---

## Image design and Dockerfile decisions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. A teammate's Node.js image is 1.2 GB. How do you approach shrinking it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Large image usually means a fat base, dev dependencies, build tools and a copied build context.

**What I'd do →**
1. Run `docker history --no-trunc myapp:latest` and `docker image ls` to see which layers are big.
2. Switch from `node:22` to `node:22-slim` or `node:22-alpine` (test musl compatibility first).
3. Use `npm ci --omit=dev` and add a `.dockerignore` with `node_modules`, `.git`, `*.md`.
4. Move to multi-stage so build tooling never ships.

**Why →** Every layer is additive; deleting files in a later layer does not shrink the image.

**Interview tip →** Say you measure with `docker history` or `dive` before optimizing.

</details>

### Q2. You need to run `apt-get install` in a Dockerfile. How do you write it so the layer stays small and cache-safe? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Classic layer hygiene question.

**What I'd do →**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```
1. Chain `update` and `install` in one `RUN` so a stale package index is never cached alone.
2. Use `--no-install-recommends`.
3. Remove `/var/lib/apt/lists` in the same layer.
4. Pin versions for critical packages if reproducibility matters.

**Why →** Cleanup in a separate `RUN` leaves the data in the earlier layer.

**Interview tip →** Mention BuildKit cache mounts as the modern alternative: `RUN --mount=type=cache,target=/var/cache/apt`.

</details>

### Q3. When would you pick `COPY` over `ADD`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both copy files; `ADD` has extra behaviours.

**What I'd do →** Default to `COPY`. Use `ADD` only for auto-extracting a local tar (`ADD app.tar.gz /opt/`) or for `ADD --checksum=sha256:... https://... /file` and git URLs.

**Why →** `ADD` with remote URLs and silent extraction is surprising and hard to cache or audit. For downloads I prefer `curl` plus checksum verification in a `RUN`, or `ADD --checksum`.

**Interview tip →** "COPY for intent clarity, ADD for the two cases COPY cannot do."

</details>

### Q4. A service must run a startup script and then the app. Should you use `CMD`, `ENTRYPOINT`, or both? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Entrypoint scripts are common for config templating or migrations.

**What I'd do →**
```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```
End the script with `exec "$@"` so the app replaces the shell and becomes PID 1.

**Why →** `ENTRYPOINT` is the fixed executable, `CMD` is the default, overridable argument list. Without `exec`, SIGTERM goes to the shell and the app is killed after the 10 s timeout.

**Interview tip →** Always use exec form (JSON array), never shell form, for the main process.

</details>

### Q5. Your Docker build re-installs all Python dependencies on every code change. Fix it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `COPY . .` comes before `pip install`, so any file change busts the cache.

**What I'd do →**
```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```
Add a `.dockerignore` so irrelevant changes (`.git`, `__pycache__`) do not invalidate `COPY . .` either.

**Why →** Layers are cached top-down; order from least to most frequently changing.

**Interview tip →** State the rule: "dependencies manifest first, source last."

</details>

### Q6. Product asks for a container that runs both nginx and a Python app. What do you say? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One-process-per-container is a guideline, not a law, but mixing hurts signal handling, scaling and logs.

**What I'd do →**
1. Propose two containers (nginx + app) in one Compose file or one Pod.
2. Share traffic via a user-defined network, or a Unix socket on a shared volume.
3. If truly required, use a minimal init such as `tini` or `s6-overlay` to supervise both.

**Why →** Independent restarts, independent scaling, clean `docker logs` per concern.

**Interview tip →** Acknowledge the exception (sidecars/supervised legacy apps) rather than being dogmatic.

</details>

### Q7. Your app ignores `docker stop` and takes exactly 10 seconds to die. Why and what do you change? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** The process is not receiving or handling SIGTERM, so Docker sends SIGKILL after the grace period.

**What I'd do →**
1. Check `docker inspect -f '{{.Config.Entrypoint}} {{.Config.Cmd}}' c` for shell form (`/bin/sh -c ...`).
2. Switch to exec form or add `exec` in the wrapper script.
3. If the app does not reap zombies or handle signals, run with `--init` (tini).
4. Handle SIGTERM in the app (graceful shutdown) and optionally set `STOPSIGNAL` and `--stop-timeout`.

**Why →** PID 1 in a container gets no default signal handlers from the kernel; it must install its own.

**Interview tip →** Mention `docker run --init` as the zero-code fix.

</details>

### Q8. You must pass a database password into an image at build time to download a private package. How? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `ARG`/`ENV` values persist in image history; never use them for secrets.

**What I'd do →** Use BuildKit secrets:
```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

**Why →** The secret is mounted in tmpfs for that `RUN` only and never lands in a layer or `docker history`.

**Interview tip →** Also mention `--mount=type=ssh` for private git clones.

</details>

### Q9. A developer wants `latest` base images always. Your view? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `latest` is a moving target; builds become non-reproducible.

**What I'd do →** Pin to a specific minor version (`python:3.12-slim-bookworm`) and ideally to a digest: `FROM python:3.12-slim-bookworm@sha256:...`. Let Renovate or Dependabot open PRs to bump tags and digests.

**Why →** You get reproducible builds and reviewable upgrades, and security patches still flow through automated PRs.

**Interview tip →** Tag for readability, digest for immutability; use both together.

</details>

### Q10. You need a tiny static Go binary in production. Which base image do you choose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Static binaries do not need a libc or a package manager.

**What I'd do →**
```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
ENTRYPOINT ["/app"]
```
`scratch` works too but lacks CA certificates and `/etc/passwd`; distroless static includes both.

**Why →** Minimal attack surface, a few MB image.

**Interview tip →** Mention you lose a shell; use `docker debug` or an ephemeral debug container.

</details>

### Q11. Alpine or Debian slim for a Python app with compiled wheels? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Alpine uses musl libc; many PyPI wheels are built for glibc (manylinux).

**What I'd do →** Choose `python:3.12-slim` (Debian) for packages like numpy, pandas, grpcio, cryptography. Pick Alpine only if all dependencies have musllinux wheels or you accept compiling from source.

**Why →** On Alpine, pip often compiles from source, which makes builds slow and images larger after installing a toolchain, and can produce subtle runtime differences (DNS resolver, locale, performance).

**Interview tip →** "Smaller base is not always smaller final image."

</details>

### Q12. How do you make an image reproducible so the same commit gives the same bytes? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Timestamps, unpinned packages and unordered layers cause drift.

**What I'd do →**
1. Pin base by digest and OS packages by version or use a snapshot mirror (e.g. snapshot.debian.org).
2. Use lockfiles (`npm ci`, `pip --require-hashes`, `go.sum`).
3. Build with `SOURCE_DATE_EPOCH` and `--output type=image,rewrite-timestamp=true` on BuildKit.
4. Use `-trimpath` and deterministic compilers settings.

**Why →** Reproducibility allows independent verification of provenance and attestations.

**Interview tip →** Be honest that full bit-for-bit reproducibility is hard; aim for it on the supply-chain-critical images.

</details>

### Q13. A Dockerfile has `RUN cd /app` followed by `RUN npm install` and it fails. Why? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each `RUN` starts a fresh shell; `cd` does not persist across instructions.

**What I'd do →** Use `WORKDIR /app` (which also creates the directory) and then `RUN npm install`. Or chain: `RUN cd /app && npm install`.

**Why →** Only filesystem changes persist between layers, not shell state such as cwd or exported variables.

**Interview tip →** Same reason `RUN export FOO=bar` is useless; use `ENV`.

</details>

### Q14. Your team needs different config per environment. Bake it in the image or inject it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build once, deploy many.

**What I'd do →** Keep one image for all environments and inject config at runtime via `-e`, `--env-file`, mounted config files, Compose `configs`, or Kubernetes ConfigMaps/Secrets. Only non-secret defaults go in `ENV`.

**Why →** Rebuilding per environment means what you tested is not what you deploy, and secrets could leak into layers.

**Interview tip →** Twelve-factor: config in the environment.

</details>

### Q15. You want to know exactly what commands produced an image you did not build. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Image metadata keeps a build history.

**What I'd do →**
```bash
docker history --no-trunc --format '{{.CreatedBy}}' nginx:1.27
docker inspect nginx:1.27 | jq '.[0].Config'
```
For deeper layer content use `dive nginx:1.27` or `docker save` and untar. For BuildKit images also `docker buildx imagetools inspect --format '{{json .Provenance}}' image`.

**Why →** History shows instructions; attestations give signed provenance when published.

**Interview tip →** History can be forged or squashed, so for trust use signatures and provenance.

</details>

### Q16. How do you handle timezone and locale in minimal images? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Slim images ship without `tzdata`, so `TZ=Asia/Kolkata` is silently ignored.

**What I'd do →**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends tzdata \
    && rm -rf /var/lib/apt/lists/*
ENV TZ=Asia/Kolkata
```
Better practice: keep containers on UTC and format time in the UI layer.

**Why →** Logs and cron across services stay comparable.

**Interview tip →** Mention Go's `time/tzdata` import to embed zone data in a static binary.

</details>

### Q17. A PR adds `COPY . .` and a developer's `.env` file ends up in the image. How do you prevent and remediate? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secrets in a layer are recoverable by anyone with the image.

**What I'd do →**
1. Add `.env`, `.git`, `*.pem`, `id_rsa*` to `.dockerignore` (allow-list style: ignore `*`, then `!src`).
2. Rotate every secret in that file immediately.
3. Delete the pushed tags and rebuild; deleting a later layer is not enough.
4. Add a CI scan (trivy `--scanners secret`, gitleaks).

**Why →** `docker save` exposes all layers.

**Interview tip →** Remediation order: rotate first, clean up second.

</details>

### Q18. Should you run `chmod`/`chown` in a separate `RUN` after `COPY`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A separate `RUN chown -R` duplicates all copied files into a new layer, doubling size.

**What I'd do →**
```dockerfile
COPY --chown=app:app --chmod=0755 entrypoint.sh /usr/local/bin/
COPY --chown=app:app . /app
```

**Why →** The metadata is set at copy time, with no duplicated layer.

**Interview tip →** `--chmod` requires BuildKit; it is the default builder since Docker 23.

</details>

### Q19. How do you run a container as a non-root user in a Dockerfile properly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default is root; compromise of the app gives root inside the container.

**What I'd do →**
```dockerfile
RUN groupadd -r app --gid=10001 && useradd -r -g app --uid=10001 app
WORKDIR /app
COPY --chown=app:app . .
USER 10001:10001
```
Use a numeric UID so Kubernetes `runAsNonRoot` can verify it.

**Why →** Limits blast radius and works with read-only root filesystems.

**Interview tip →** Ports below 1024 need `CAP_NET_BIND_SERVICE` or sysctl; use 8080 instead.

</details>

### Q20. What is the difference between `ARG` and `ENV`, and when does each matter? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** One is build-time, the other persists into the running container.

**What I'd do →** Use `ARG` for build parameters (`ARG APP_VERSION`) and `ENV` for runtime variables. An `ARG` declared before `FROM` is only usable in `FROM` lines unless redeclared after it. Convert when needed: `ENV VERSION=$APP_VERSION`.

**Why →** `ARG` values appear in `docker history` and do not exist at runtime; `ENV` values are visible to anyone with `docker inspect`.

**Interview tip →** Neither is a safe place for secrets.

</details>

### Q21. You must support both amd64 and arm64 users (Apple Silicon laptops, Graviton in prod). 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need a multi-arch manifest list.

**What I'd do →**
```bash
docker buildx create --use --name multi
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.example.com/app:1.4.0 --push .
```
In the Dockerfile use `ARG TARGETARCH` for arch-specific downloads and cross-compile in the build stage with `FROM --platform=$BUILDPLATFORM` to avoid slow QEMU emulation.

**Why →** One tag resolves to the right image per node.

**Interview tip →** Verify with `docker buildx imagetools inspect app:1.4.0`.

</details>

### Q22. When is `docker commit` acceptable? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** It snapshots a container's filesystem into an image.

**What I'd do →** Only for quick debugging or forensics (preserve a compromised container's state). Never for shipping: the result is not reproducible, has no recorded steps and can include secrets and caches.

**Why →** A Dockerfile in git is reviewable and rebuildable; a committed image is a black box.

**Interview tip →** `docker diff <container>` shows what changed, useful for turning manual fixes into Dockerfile lines.

</details>

### Q23. Which `HEALTHCHECK` command do you use in a distroless image with no `curl`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No shell and no curl means a Dockerfile `HEALTHCHECK CMD curl` fails.

**What I'd do →** Add a tiny health subcommand to the app binary (`/app healthcheck`) and use exec form:
```dockerfile
HEALTHCHECK --interval=15s --timeout=3s --start-period=20s --retries=3 \
  CMD ["/app", "healthcheck"]
```
Alternatively rely on orchestrator probes (Kubernetes HTTP probes) instead of Docker's.

**Why →** Keeps the image minimal while retaining health semantics.

**Interview tip →** In Kubernetes, the Dockerfile HEALTHCHECK is ignored; use probes.

</details>

### Q24. A build needs a 2 GB dataset only for tests. How do you keep it out of the image? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Anything `COPY`ed in any stage of the final image's chain stays in it.

**What I'd do →**
1. Put tests in a separate stage (`FROM build AS test`) and run with `docker build --target test`.
2. Mount the data with `RUN --mount=type=bind,source=testdata,target=/testdata ...`.
3. Exclude it in `.dockerignore` from the default context.

**Why →** Bind and cache mounts exist only during the `RUN` step and are not committed.

**Interview tip →** Use `--target` to produce different artifacts from one Dockerfile.

</details>

### Q25. How do you decide between a Dockerfile `VOLUME` instruction and not declaring it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** `VOLUME /data` forces an anonymous volume at run time and freezes later changes to that path in the build.

**What I'd do →** Avoid `VOLUME` in my own images; document paths and let operators mount named volumes or bind mounts. Use it only for images where data persistence by default is a safety feature (e.g. databases like the official postgres image).

**Why →** Any `RUN chown /data` after `VOLUME /data` is silently discarded, and anonymous volumes pile up (`docker volume ls -f dangling=true`).

**Interview tip →** Mention `docker run --rm` removes anonymous volumes with the container; `-v` flag on `docker rm -v` does too.

</details>

---

## Multi-stage builds, cache, tags and registries

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q26. Show a multi-stage Dockerfile for a Java Spring Boot app and explain what you gain. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** JDK and Maven are needed to build, only a JRE is needed to run.

**What I'd do →**
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -q dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -q package -DskipTests

FROM eclipse-temurin:21-jre
COPY --from=build /src/target/app.jar /app.jar
USER 10001
ENTRYPOINT ["java","-jar","/app.jar"]
```

**Why →** Final image carries only the JRE and jar, usually 250 MB instead of 700 MB+.

**Interview tip →** Mention the Maven cache mount for fast rebuilds.

</details>

### Q27. CI rebuilds from scratch every run and takes 12 minutes. How do you bring it down? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Ephemeral CI runners have no local layer cache.

**What I'd do →**
1. Use BuildKit remote cache:
```bash
docker buildx build \
  --cache-from type=registry,ref=reg.example.com/app:buildcache \
  --cache-to type=registry,ref=reg.example.com/app:buildcache,mode=max \
  -t reg.example.com/app:$SHA --push .
```
2. On GitHub Actions use `type=gha`.
3. Reorder the Dockerfile so deps are above source.

**Why →** `mode=max` also exports intermediate stages, so multi-stage builds get cache hits.

**Interview tip →** Say you track cache hit rate from the `CACHED` lines in build output.

</details>

### Q28. You tag every build `myapp:latest`. A rollback is requested. What happens? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `latest` was overwritten; the previous image is untagged ("dangling") or only in registry history.

**What I'd do →** Going forward tag with the git SHA and semantic version (`app:1.8.2`, `app:git-3f2c1a9`), deploy by digest, and keep `latest` only as a convenience. For the immediate rollback find the old digest in registry logs, a deploy manifest, or a node's `docker image ls --digests`.

**Why →** Immutable references make rollback a config change, not a rebuild.

**Interview tip →** Tags are mutable pointers; digests are content addresses.

</details>

### Q29. How do you promote an image from dev to prod registry without rebuilding? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same bytes must go to prod.

**What I'd do →**
```bash
docker buildx imagetools create \
  -t prod.example.com/app:1.8.2 dev.example.com/app@sha256:ab12...
# or
skopeo copy --all docker://dev.example.com/app@sha256:ab12... \
                  docker://prod.example.com/app:1.8.2
# or
crane copy dev.example.com/app@sha256:ab12... prod.example.com/app:1.8.2
```

**Why →** Digest equality proves what was tested is what runs; `--all` preserves multi-arch manifests.

**Interview tip →** `docker pull/tag/push` works but pulls only your platform.

</details>

### Q30. Disk on the build server keeps filling up with images. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old tags, dangling layers and build cache accumulate.

**What I'd do →**
```bash
docker system df
docker image prune -a --filter "until=168h"
docker builder prune --filter "until=72h" --keep-storage 20GB
```
Run it from a nightly cron or a CI cleanup job and configure BuildKit GC in `buildkitd.toml`.

**Why →** `docker system prune -a` is blunt and can delete images other jobs need.

**Interview tip →** Do not prune volumes unattended; that deletes data.

</details>

### Q31. When do you use `--no-cache` and when is it a smell? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** It forces every layer to be rebuilt.

**What I'd do →** Use it occasionally (weekly scheduled build) to pull in OS security updates when you do not pin packages, or when debugging cache poisoning. As a default in CI it is a smell: it hides poor layer ordering and slows everything.

**Why →** Better: bump a build `ARG` such as `CACHE_BUST` at a chosen layer, or use `--pull` to refresh the base.

**Interview tip →** `--pull` and `--no-cache` solve different problems.

</details>

### Q32. Explain a build where `COPY` always misses the cache even though code did not change. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cache key for `COPY` is a checksum of file contents and metadata (mode), not mtime.

**What I'd do →**
1. Look for generated files in context (`build/`, `*.log`, `.DS_Store`, version stamp files) and add them to `.dockerignore`.
2. Check that a script rewrites file permissions on checkout.
3. Run `docker buildx build --progress=plain .` and compare which step is not `CACHED`.

**Why →** One changed byte in the copied set invalidates this layer and all after it.

**Interview tip →** Copy narrowly (`COPY src/ src/`) rather than the whole context.

</details>

### Q33. Design a tagging strategy for a team that releases weekly. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need traceability, rollback and human friendliness.

**What I'd do →**
- Every CI build: `app:<git-sha>` (immutable, never re-pushed).
- Releases: `app:1.8.2`, plus floating `1.8` and `1`.
- Environments reference digests via GitOps (`image: app@sha256:...`) with the tag in a comment.
- `latest` only for dev.
- Registry tag immutability enabled (ECR, Harbor).

**Why →** Floating tags help consumers, immutable tags help operators.

**Interview tip →** Mention retention policies so old SHA tags age out.

</details>

### Q34. How do you target a single stage while debugging a multi-stage build? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need to inspect the build stage output.

**What I'd do →**
```bash
docker build --target build -t app:debug .
docker run --rm -it app:debug sh
```
Use `docker buildx build --target test --progress=plain .` to run only tests.

**Why →** BuildKit only builds stages the target depends on, which also saves time.

**Interview tip →** Name stages with `AS` for readability.

</details>

### Q35. The registry has 40,000 images and storage costs are rising. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** No retention policy, SHA tags per commit.

**What I'd do →**
1. Define policy: keep last N tags per branch, keep all release tags, expire untagged after 14 days (ECR lifecycle policy, Harbor retention, GCR cleanup policies).
2. Run garbage collection (`registry garbage-collect` for self-hosted).
3. Dedupe by promoting only release candidates to long-lived repos.

**Why →** Layers are shared, so deletes only free blobs no manifest references.

**Interview tip →** Test policies in dry-run mode first.

</details>

### Q36. You want to verify that the image a node pulled is the one CI built and signed. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tag-only trust can be hijacked by a registry compromise or accidental overwrite.

**What I'd do →**
1. Sign in CI: `cosign sign --yes reg.example.com/app@sha256:...` (keyless via OIDC).
2. Attach SBOM and provenance: `docker buildx build --sbom=true --provenance=mode=max`.
3. Enforce at admission: Kyverno or Sigstore policy-controller verifying the signature identity and issuer.
4. Locally verify: `cosign verify --certificate-identity-regexp ... --certificate-oidc-issuer https://token.actions.githubusercontent.com image@sha256:...`.

**Why →** Verifies who built it and that the content has not changed.

**Interview tip →** Sign digests, not tags.

</details>

### Q37. Pull rate limits from Docker Hub are failing CI. What are your options? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Anonymous pulls are rate limited per IP; shared NAT in CI exhausts it.

**What I'd do →**
1. Authenticate pulls with a Docker Hub service account (`docker login`).
2. Set up a pull-through cache (Harbor proxy project, ECR pull-through cache, `registry:2` with `proxy.remoteurl`).
3. Mirror critical base images into our own registry and reference those.
4. Configure `registry-mirrors` in `daemon.json`.

**Why →** Reduces external dependency and speeds up pulls.

**Interview tip →** Mirroring base images also protects you from upstream tag removal.

</details>

### Q38. What does it mean that `docker push` says "Layer already exists"? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Registries store blobs by content digest.

**What I'd do →** Nothing; it is a feature. The registry already has that layer (from a prior push or another repo with cross-repo mount), so the client skips upload. Only the manifest and changed layers transfer.

**Why →** This is why stable base layers and good ordering make pushes and pulls fast.

**Interview tip →** Same logic speeds pulls: nodes only fetch missing layers.

</details>

### Q39. How do you inspect a remote image's size, platforms and digest without pulling it? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need metadata only.

**What I'd do →**
```bash
docker buildx imagetools inspect nginx:1.27
docker manifest inspect nginx:1.27
crane manifest nginx:1.27 | jq
skopeo inspect docker://nginx:1.27
crane digest nginx:1.27
```

**Why →** These talk to the registry API and fetch manifests, not layers.

**Interview tip →** `imagetools inspect` shows the manifest list with per-platform digests.

</details>

### Q40. A multi-stage build copies from `--from=build` but the final image is still huge. Why? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Probably copying a whole directory (`COPY --from=build /src /app`) including node_modules dev deps or source, or the final stage is built `FROM build`.

**What I'd do →**
1. `docker history app` to find the heavy layer.
2. Copy only artifacts (`/src/dist`, the binary).
3. Install prod-only deps in the final stage or prune (`npm prune --omit=dev`).
4. Verify with `dive`.

**Why →** Multi-stage only helps if the last stage starts from a clean base.

**Interview tip →** Check that the last `FROM` is the slim runtime image.

</details>

### Q41. How do you share a common base layer across 30 microservices? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeating the same hardening steps in every Dockerfile wastes build time and drifts.

**What I'd do →** Build an internal golden base (`registry/base/python:3.12-2026w45`) with CA bundles, non-root user, tzdata and agent config. Rebuild weekly, scan, sign and publish. Services `FROM` it by digest; Renovate bumps the digest.

**Why →** One place to patch, shared layers on nodes, consistent security posture.

**Interview tip →** Keep the base thin; a fat shared base becomes a dependency trap.

</details>

### Q42. What is a dangling image and is it safe to delete? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** An image with no tag, shown as `<none>:<none>`, usually left when a tag moved to a new build.

**What I'd do →**
```bash
docker images -f dangling=true
docker image prune
```

**Why →** Safe unless a running container or another image depends on it, in which case Docker refuses or keeps it as a parent layer. `-a` removes all unused images, not only dangling ones.

**Interview tip →** Distinguish dangling (untagged) from unused (no container).

</details>

### Q43. Cache mounts in Dockerfiles: when are they worth it? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Package managers redownload packages on each change of the dependency layer.

**What I'd do →**
```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build go build ./...
```

**Why →** The cache directory persists between builds on the same builder but never enters the image, so you get speed without size. It is not shared across CI runners unless the builder is persistent.

**Interview tip →** On ephemeral CI combine with registry cache or a long-lived buildkit daemon.

</details>

### Q44. Your pull from a private registry works locally but fails on the Kubernetes node with containerd. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker credentials in `~/.docker/config.json` are not used by containerd or kubelet.

**What I'd do →** Create an `imagePullSecret` (`kubectl create secret docker-registry regcred ...`) and reference it in the Pod or ServiceAccount, or configure kubelet image credential providers (ECR, GCR, ACR). Check with `crictl pull` on the node.

**Why →** Each runtime has its own auth path.

**Interview tip →** Cloud credential providers avoid long-lived static passwords.

</details>

### Q45. How would you rollback a bad image deployed via Docker Compose on one host? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compose has no built-in rollback.

**What I'd do →**
1. Edit the `image:` to the previous known-good tag or digest.
2. `docker compose pull && docker compose up -d --no-deps web`.
3. Confirm with `docker compose ps` and `docker compose logs -f web`.

**Why →** Works only because old tags are retained, which is why we never overwrite tags.

**Interview tip →** For zero downtime and automatic rollback move to Swarm or Kubernetes.

</details>

### Q46. What is the difference between `docker save`/`load` and `export`/`import`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Moving images into an air-gapped network.

**What I'd do →** `docker save -o app.tar app:1.8.2` then `docker load -i app.tar` on the target: preserves layers, tags and history. `docker export` dumps a container's flattened filesystem and `docker import` makes a single-layer image, losing metadata like ENTRYPOINT and ENV.

**Why →** Use `save`/`load` for images; `export`/`import` is for filesystem snapshots.

**Interview tip →** For air-gap, `skopeo copy docker://... oci-archive:` is cleaner and can verify digests.

</details>

### Q47. You need to run integration tests against the exact image you will ship. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Testing a different build than the one released defeats the purpose.

**What I'd do →**
1. Build once and load: `docker buildx build --load -t app:$SHA .`.
2. Start dependencies with Compose, run tests against `app:$SHA` (Testcontainers or `docker compose up --wait`).
3. Push that same digest only if tests pass.

**Why →** Promote the artefact, not the source.

**Interview tip →** `docker compose up --wait` blocks until health checks pass.

</details>

### Q48. Developers complain builds are slow on Apple Silicon for amd64 targets. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** QEMU emulation of amd64 is 5–20x slower.

**What I'd do →**
1. Cross-compile natively: `FROM --platform=$BUILDPLATFORM golang AS build` and set `GOOS=$TARGETOS GOARCH=$TARGETARCH`.
2. Use remote native builders: `docker buildx create --name farm --node amd64 --platform linux/amd64 ssh://builder` plus a local arm64 node.
3. Enable Rosetta in Docker Desktop for amd64 emulation where needed.
4. Build multi-arch in CI on native runners and merge manifests.

**Why →** Native or cross-compile beats emulation.

**Interview tip →** Languages without easy cross-compilers need native nodes.

</details>

### Q49. How do you keep a `latest` tag but still know which build it points to? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Operators need traceability.

**What I'd do →** Add OCI labels at build time:
```dockerfile
LABEL org.opencontainers.image.revision=$GIT_SHA \
      org.opencontainers.image.source=https://github.com/acme/app \
      org.opencontainers.image.created=$BUILD_DATE
```
Then `docker inspect -f '{{json .Config.Labels}}' app:latest`. Or use `docker/metadata-action` to generate them.

**Why →** Labels travel with the image.

**Interview tip →** The `org.opencontainers.image.*` namespace is the standard.

</details>

### Q50. What are the risks of using a public image straight from Docker Hub in production? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain exposure.

**What I'd do →**
1. Prefer Docker Official Images, Verified Publisher or Docker Hardened Images.
2. Mirror into our registry, scan and sign before use.
3. Pin by digest.
4. Review the Dockerfile, check maintenance cadence and CVE counts.

**Why →** Tags can be repointed, images can be abandoned, and typo-squatted images carry malware.

**Interview tip →** Treat third-party images as third-party code.

</details>

---

## Volumes, storage and data

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q51. A Postgres container was removed and the data is gone. How should it have been run? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Data lived in the container's writable layer (or an anonymous volume that was removed with `--rm`/`-v`).

**What I'd do →**
```bash
docker volume create pgdata
docker run -d --name pg -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg -p 127.0.0.1:5432:5432 postgres:17
```
Back up with `pg_dump` on a schedule; do not treat the volume as a backup.

**Why →** Named volumes outlive containers and are managed by Docker.

**Interview tip →** Note that for the postgres image the mount path changed to `/var/lib/postgresql` for v18+; check the image docs.

</details>

### Q52. Bind mount or named volume for local development of a Node app? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Developers want live code reload.

**What I'd do →** Bind mount the source (`-v $PWD:/app`) and shadow `node_modules` with a named or anonymous volume (`-v /app/node_modules`) so host and container binaries do not clash. With Compose use `develop.watch` (`action: sync`) for efficient syncing.

**Why →** Bind mounts reflect host edits instantly; volumes are better for data and performance on macOS/Windows.

**Interview tip →** Bind mounts in prod couple the container to host paths; avoid them there.

</details>

### Q53. How do you back up a named volume? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Volume data lives under `/var/lib/docker/volumes`, but you should not poke at it directly.

**What I'd do →**
```bash
docker run --rm -v pgdata:/data:ro -v $PWD:/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tgz -C /data .
```
For databases prefer logical dumps or snapshot after `CHECKPOINT`/with the DB quiesced, because a file copy of a running DB can be inconsistent.

**Why →** A helper container uses the Docker API and respects drivers.

**Interview tip →** Test restores, not just backups.

</details>

### Q54. Two containers must share files. What are your options? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Typical for a web server serving files generated by an app.

**What I'd do →**
1. A shared named volume mounted in both (`-v assets:/srv/assets`, read-only in the consumer).
2. `--volumes-from producer` for a quick link.
3. For cross-host, NFS volume driver or object storage instead.

**Why →** Volumes avoid host path dependency; use `:ro` where possible.

**Interview tip →** Concurrent writers need app-level locking; Docker does not provide it.

</details>

### Q55. A container needs a writable temp area but the root filesystem is read-only. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `--read-only` is a hardening best practice but apps still write to `/tmp` and `/run`.

**What I'd do →**
```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /run -v appdata:/var/lib/app myapp
```

**Why →** Writes land only in explicitly allowed places, and tmpfs is in memory so it vanishes on stop.

**Interview tip →** tmpfs counts against the container memory limit.

</details>

### Q56. Moving a database volume from one Docker host to another. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need consistent data transfer with minimal downtime.

**What I'd do →**
1. Prefer a logical migration: `pg_dump | psql` or replication.
2. If you must copy files, stop the container first, then:
```bash
docker run --rm -v pgdata:/from alpine tar c -C /from . | \
  ssh new-host 'docker run --rm -i -v pgdata:/to alpine tar x -C /to'
```
3. Start with the same image major version.

**Why →** Copying files from a running DB risks corruption.

**Interview tip →** Same major version is mandatory for binary data directories.

</details>

### Q57. Explain how copy-on-write affects performance of a write-heavy container. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Writes to files from lower image layers trigger a copy-up to the writable layer; writes to the container layer go through overlayfs.

**What I'd do →** Put write-heavy paths (DB data, logs, queues, build caches) on volumes, which bypass the storage driver and give near-native I/O. Check driver with `docker info | grep -i storage` (expect `overlay2` or the containerd snapshotter `overlayfs`).

**Why →** Copy-up of a large file (e.g., a 1 GB file touched by 1 byte) is expensive and repeated across containers.

**Interview tip →** "Data on volumes, code in layers."

</details>

### Q58. How would you limit how much disk a container can write? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** One runaway container fills the host disk.

**What I'd do →**
1. `docker run --storage-opt size=10G` works on overlay2 only with an XFS backing filesystem mounted with `pquota`.
2. Alternatively put `/var/lib/docker` on its own filesystem or LV so it cannot starve the root disk.
3. Use `--tmpfs size=` for scratch space and log rotation for logs.
4. In Kubernetes use `ephemeral-storage` limits.

**Why →** Docker has no portable per-container disk quota on ext4.

**Interview tip →** Monitor `docker system df -v` and node disk pressure.

</details>

### Q59. A volume driver for NFS is required for a Swarm service. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Tasks can land on any node, so data must be reachable everywhere.

**What I'd do →** Use the built-in `local` driver with NFS options:
```bash
docker volume create --driver local \
  --opt type=nfs --opt o=addr=10.0.0.5,rw,nfsvers=4 \
  --opt device=:/exports/app shared-data
```
In a stack file define the volume with `driver_opts`.

**Why →** The mount happens on whichever node runs the task.

**Interview tip →** Mention that NFS adds latency and a single point of failure; databases should not live there without care.

</details>

### Q60. You need to find which container is using a volume before deleting it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `docker volume rm` fails with "volume is in use".

**What I'd do →**
```bash
docker ps -a --filter volume=pgdata --format '{{.ID}} {{.Names}} {{.Status}}'
docker volume inspect pgdata
```
Stop and remove the referencing containers, then `docker volume rm pgdata`.

**Why →** Stopped containers still hold references.

**Interview tip →** `docker volume prune` removes unused volumes (anonymous only unless `-a`); never run it blindly in prod.

</details>

### Q61. Application writes files as UID 1000 but the mounted host directory is root-owned. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Permission denied on bind mount caused by UID mismatch.

**What I'd do →**
1. `chown 1000:1000 /srv/app-data` on the host, or run with `--user $(id -u):$(id -g)`.
2. With named volumes, Docker copies ownership from the image path on first use, so `chown` in the Dockerfile before first mount.
3. On SELinux hosts add `:z` or `:Z`.

**Why →** The kernel checks numeric IDs, not names.

**Interview tip →** With rootless/userns-remap the host UID differs from the container UID.

</details>

### Q62. Should log files go to a volume? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Apps writing to `/var/log/app.log` inside the container.

**What I'd do →** Log to stdout/stderr so Docker's logging driver collects them. For legacy apps symlink `/var/log/app.log` to `/dev/stdout` as the nginx image does.

**Why →** Works with `docker logs`, centralized shippers and rotation, with no volume to clean.

**Interview tip →** Twelve-factor: treat logs as event streams.

</details>

### Q63. A legacy app needs persistent state, but the team wants to scale it to 3 replicas. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared local state blocks horizontal scaling.

**What I'd do →**
1. Identify what state it holds (sessions, uploads, cache).
2. Move sessions to Redis, uploads to S3-compatible storage, DB state to a managed database.
3. If state must stay on disk, use per-replica volumes (StatefulSet in Kubernetes) rather than a shared writable volume.

**Why →** A shared volume with several writers risks corruption and creates a single point of failure.

**Interview tip →** Externalizing state is the real fix, not a clever volume.

</details>

### Q64. How do you seed a new volume with initial data from an image? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker populates an empty named volume from the image content at the mount path on first use.

**What I'd do →** Put seed data at the path in the image, then `docker run -v seed:/usr/share/app/data app`. For databases use `/docker-entrypoint-initdb.d/*.sql` scripts which run only when the data directory is empty.

**Why →** Population happens only for an empty named volume, not for bind mounts, which mask image content.

**Interview tip →** A bind mount over a populated path hides the image's files.

</details>

### Q65. Disk shows 95% full on `/var/lib/docker`. What do you check in order? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Typically images, build cache, container logs or volumes.

**What I'd do →**
```bash
docker system df -v
du -xh --max-depth=2 /var/lib/docker | sort -h | tail
ls -lhS /var/lib/docker/containers/*/*-json.log | head
```
Then prune images/build cache, truncate offending logs (`truncate -s 0 <file>`), and set `max-size`/`max-file` log options.

**Why →** The `json-file` driver has no rotation by default.

**Interview tip →** Fix the cause (log rotation), not only the symptom.

</details>

### Q66. Choosing between `--mount` and `-v`. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Both attach storage.

**What I'd do →** Prefer `--mount type=volume,src=data,dst=/data,readonly`: explicit key-value syntax, and it errors if a bind source path does not exist, whereas `-v` silently creates a root-owned directory.

**Why →** Fewer surprises and clearer reviews.

**Interview tip →** Compose long syntax mirrors `--mount`.

</details>

### Q67. Persist a cache for CI jobs across runs on a self-hosted runner. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Repeated dependency downloads.

**What I'd do →** Mount a named volume for `~/.cache`/`~/.m2` into job containers, or rely on BuildKit cache mounts on a persistent builder. Add size limits via builder GC (`keepStorage`) and key by lockfile hash when using external caches.

**Why →** Persistent runners make local caches effective; ephemeral ones need remote caches.

**Interview tip →** Beware cache poisoning between untrusted PR builds; isolate caches per trust level.

</details>

### Q68. What happens to data when you run `docker compose down` vs `down -v`? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Common accident.

**What I'd do →** `docker compose down` removes containers and networks but keeps named volumes. `docker compose down -v` also deletes named volumes declared in the file, and anonymous ones. Use `stop` if you only want to pause.

**Why →** `-v` is destructive; bind mounts are never deleted.

**Interview tip →** Say you guard prod hosts with backups and never run `-v` there.

</details>

### Q69. Live-migrating a stateful container with minimal downtime. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker does not support live migration (CRIU checkpoint is experimental).

**What I'd do →**
1. For a DB, set up replication to the new host, wait for lag zero, promote, repoint clients.
2. For file-based state, `rsync` while running, then stop, final `rsync`, start on the new host.
3. Lower DNS TTL beforehand.

**Why →** Application-level replication is reliable; container checkpoint/restore is not production-grade.

**Interview tip →** Mention `podman container checkpoint` exists but is niche.

</details>

### Q70. A container writes huge files to `/tmp` and gets OOM-killed. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `/tmp` is probably tmpfs (counts as memory) or a mounted tmpfs.

**What I'd do →** Move large temp files to a disk volume (`-v scratch:/scratch`) and set `TMPDIR=/scratch`, or cap tmpfs with `size=`. Confirm with `docker inspect -f '{{.HostConfig.Tmpfs}}' c` and `df -h /tmp` inside.

**Why →** tmpfs pages are charged to the container's memory cgroup.

**Interview tip →** Plain overlay `/tmp` is on disk; tmpfs only if explicitly mounted.

</details>

### Q71. Sensitive data in a volume must be encrypted at rest. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker volumes are plain directories.

**What I'd do →** Encrypt below Docker: LUKS on the disk that holds `/var/lib/docker` or the volume path, or cloud-provider encrypted block storage (EBS with KMS). For app-level secrets use envelope encryption in the app or database TDE.

**Why →** Docker itself has no volume encryption feature.

**Interview tip →** Also mention key management: who unlocks LUKS on reboot.

</details>

### Q72. Configuration files need to update without rebuilding the image. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Config is separate from code.

**What I'd do →** Mount it read-only: `-v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro`, or in Compose use `configs:`. Reload without restart: `docker exec web nginx -s reload`.

**Why →** Same image, different behaviour, and a simple rollback of config.

**Interview tip →** Single-file bind mounts break on editors that replace the inode (vim); mount the directory.

</details>

### Q73. How do you check what's changed inside a running container's filesystem? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need to find unexpected writes.

**What I'd do →**
```bash
docker diff web
```
Output shows `A` (added), `C` (changed), `D` (deleted) paths versus the image.

**Why →** Reveals what should be a volume or what an attacker modified.

**Interview tip →** Good for forensics before you destroy the container.

</details>

### Q74. A shared-NFS bind mount makes the app very slow. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Metadata-heavy workloads (node_modules, PHP autoload) over NFS or macOS file sharing are slow.

**What I'd do →**
1. Keep dependencies inside the container (named volume for `node_modules`/`vendor`).
2. On Docker Desktop use VirtioFS and `develop.watch` sync.
3. Tune NFS (`nfsvers=4.1`, `noatime`, larger `rsize/wsize`) for servers.
4. Measure with `fio` or `strace -c`.

**Why →** Each `stat` becomes a network round trip.

**Interview tip →** Local volumes are almost always faster than network mounts.

</details>

### Q75. Retention for a volume holding uploaded user files. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Business data must survive host loss.

**What I'd do →** Move to object storage (S3) with versioning, or replicate volume data via snapshot schedules (EBS, LVM snapshots) and off-host backups. Document RPO/RTO and run restore drills.

**Why →** A single host volume is a single point of failure.

**Interview tip →** Ask about RPO/RTO before choosing the design.

</details>

---

## Networking and Compose

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q76. Containers on the default bridge cannot resolve each other by name. Why, and what do you do? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The default `bridge` network has no embedded DNS for container names.

**What I'd do →**
```bash
docker network create appnet
docker run -d --network appnet --name db postgres:17
docker run -d --network appnet --name api myapi   # reach db at db:5432
```
Compose does this automatically with a per-project network.

**Why →** User-defined bridges provide automatic service discovery and better isolation.

**Interview tip →** `--link` is legacy; do not use it.

</details>

### Q77. How do you expose a container only to the local host, not the internet? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `-p 5432:5432` binds to 0.0.0.0 and Docker inserts iptables rules that bypass `ufw`.

**What I'd do →** `-p 127.0.0.1:5432:5432`, or do not publish at all and let other containers connect over the Docker network. For Compose: `ports: ["127.0.0.1:5432:5432"]`.

**Why →** Published ports are DNAT'd in the `DOCKER` chain before most host firewall rules.

**Interview tip →** Use the `DOCKER-USER` chain for custom filtering.

</details>

### Q78. Design a Compose file for web + api + postgres + redis with correct startup ordering. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `depends_on` alone only waits for the container to start.

**What I'd do →**
```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
    volumes: [pgdata:/var/lib/postgresql/data]
  api:
    build: ./api
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }
  redis: { image: "redis:7" }
  web:
    image: nginx:1.27
    ports: ["80:80"]
    depends_on: [api]
volumes: { pgdata: {} }
```

**Why →** `service_healthy` gates on readiness, not just process start.

**Interview tip →** Apps should still retry connections; ordering is best effort.

</details>

### Q79. Frontend must reach the API but not the database. Segment the network. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Least-privilege networking.

**What I'd do →**
```yaml
services:
  web: { networks: [front] }
  api: { networks: [front, back] }
  db:  { networks: [back] }
networks:
  front: {}
  back: { internal: true }
```

**Why →** `db` shares no network with `web`; `internal: true` also blocks outbound internet for the back tier.

**Interview tip →** Network aliases give stable names per network.

</details>

### Q80. A container needs to call a service running on the Docker host. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** `localhost` inside the container is itself.

**What I'd do →** Docker Desktop: use `host.docker.internal`. On Linux: `--add-host=host.docker.internal:host-gateway` (or Compose `extra_hosts: ["host.docker.internal:host-gateway"]`). The host service must listen on an address reachable from the bridge, not only 127.0.0.1.

**Why →** The bridge gateway (usually 172.17.0.1) is the host from the container's view.

**Interview tip →** `--network host` is an alternative but removes isolation.

</details>

### Q81. When would you use `--network host`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Needs maximum network performance or many dynamic ports.

**What I'd do →** Use it for packet capture tools, node-level monitoring agents or high-throughput proxies on Linux. Do not combine with `-p`; those flags are ignored.

**Why →** Skips NAT and veth overhead, but port conflicts with the host are possible and the container sees all host interfaces.

**Interview tip →** It does not work the same on Docker Desktop (VM boundary).

</details>

### Q82. Two Compose projects need to talk to each other. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Each project gets its own isolated network.

**What I'd do →** Create a shared external network and attach both:
```yaml
networks:
  shared:
    external: true
    name: platform-net
```
`docker network create platform-net` once; services join with an explicit alias.

**Why →** Explicit shared network avoids publishing ports just for inter-project calls.

**Interview tip →** Name collisions on service names can occur; use aliases.

</details>

### Q83. How do you route traffic to multiple Compose services by hostname? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** One host, several web apps on port 80/443.

**What I'd do →** Run a reverse proxy container (Traefik or Nginx Proxy Manager/Caddy) on a shared network. With Traefik use labels:
```yaml
labels:
  - "traefik.http.routers.api.rule=Host(`api.example.com`)"
  - "traefik.http.services.api.loadbalancer.server.port=8080"
```
Mount the Docker socket read-only into Traefik via a socket proxy.

**Why →** Auto-discovery plus automated TLS (Let's Encrypt).

**Interview tip →** Exposing the raw Docker socket to the proxy is a risk; use `docker-socket-proxy`.

</details>

### Q84. You need a fixed IP for a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some legacy firewalls or configs require static IPs.

**What I'd do →**
```bash
docker network create --subnet 172.30.0.0/24 fixed
docker run --network fixed --ip 172.30.0.10 myapp
```
In Compose, set `ipam` config and `ipv4_address`.

**Why →** Static IPs only work on user-defined networks with a subnet.

**Interview tip →** Prefer DNS names; static IPs signal tight coupling.

</details>

### Q85. Docker's default subnet collides with the company VPN. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `172.17.0.0/16` or auto-assigned `172.18+` overlap with corporate ranges and break routing.

**What I'd do →** Change in `/etc/docker/daemon.json`:
```json
{
  "bip": "10.200.0.1/24",
  "default-address-pools": [{"base": "10.201.0.0/16", "size": 24}]
}
```
Restart docker and recreate networks.

**Why →** Pool applies to new networks; existing ones keep the old subnet.

**Interview tip →** Plan address space with the network team up front.

</details>

### Q86. How do you inspect which containers are attached to a network and their IPs? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need a quick topology view.

**What I'd do →**
```bash
docker network ls
docker network inspect appnet \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

**Why →** Shows membership, subnet and gateway.

**Interview tip →** `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c` for one container.

</details>

### Q87. Load balance three replicas on a single Docker host without Kubernetes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compose `deploy.replicas` works with `docker compose up --scale`.

**What I'd do →**
```bash
docker compose up -d --scale api=3
```
Do not publish a fixed host port on the scaled service; put nginx/Traefik in front, which resolves `api` via Docker DNS (round-robin across A records).

**Why →** Docker's embedded DNS returns all replica IPs.

**Interview tip →** Nginx caches DNS at start; use `resolver 127.0.0.11` with a variable in `proxy_pass`.

</details>

### Q88. Outbound HTTPS from containers works only through a corporate proxy. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Build and runtime need proxy settings; daemon pulls need them separately.

**What I'd do →**
1. Client side: `~/.docker/config.json` `"proxies": {"default": {"httpProxy": "...", "noProxy": "..."}}` injects env vars into containers and builds.
2. Daemon pulls: systemd drop-in `Environment="HTTPS_PROXY=..."` for `docker.service`.
3. Install the corporate CA into base images.

**Why →** The daemon, build containers and runtime containers are three separate places.

**Interview tip →** Do not bake proxy settings into images.

</details>

### Q89. Explain how you would debug a container with no shell and no network tools. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Distroless image, need to run `tcpdump` or `curl`.

**What I'd do →**
```bash
docker run --rm -it --network container:myapp --pid container:myapp \
  nicolaka/netshoot
# or
docker debug myapp
# kubernetes
kubectl debug -it pod/myapp --image=nicolaka/netshoot --target=myapp
```

**Why →** Joining the target's network (and PID) namespace gives full tooling without changing the image.

**Interview tip →** Never install tools into prod images "just for debugging".

</details>

### Q90. IPv6 is needed for containers. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker IPv6 is off by default.

**What I'd do →** In `daemon.json`: `{"ipv6": true, "fixed-cidr-v6": "2001:db8:1::/64"}` (with `ip6tables: true` on recent versions) and for user networks `docker network create --ipv6 --subnet 2001:db8:2::/64 v6net`. Verify routing and firewall for the prefix.

**Why →** Docker must hand out routable or NAT'd v6 addresses.

**Interview tip →** Recent Docker releases (27+) improved IPv6 and `ip6tables` support.

</details>

### Q91. Which Compose features do you use for local dev vs production? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Same services, different needs.

**What I'd do →** Base `compose.yaml` holds services; `compose.override.yaml` (auto-loaded) adds bind mounts, debug ports and `develop.watch`; `compose.prod.yaml` sets `restart: unless-stopped`, resource limits and no bind mounts: `docker compose -f compose.yaml -f compose.prod.yaml up -d`. Use profiles for optional tools.

**Why →** Layered files avoid duplication.

**Interview tip →** `docker compose config` shows the merged result.

</details>

### Q92. The app works with `docker run` but inside Compose cannot read an env var. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Compose interpolates `${VAR}` from the shell and `.env`; `environment:` and `env_file:` differ.

**What I'd do →** Check with `docker compose config` to see resolved values; use `env_file: .env.app` for container env and `.env` for substitution in the compose file itself. Quote values with spaces.

**Why →** `.env` is for Compose-file variables unless you also list it under `env_file`.

**Interview tip →** Precedence: shell > `environment` > `env_file` > image `ENV`.

</details>

### Q93. How do you handle secrets in Compose? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Env vars leak in `docker inspect` and logs.

**What I'd do →**
```yaml
services:
  api:
    secrets: [db_password]
    environment: { DB_PASSWORD_FILE: /run/secrets/db_password }
secrets:
  db_password: { file: ./secrets/db_password.txt }
```
App reads the file at start. Keep the secrets directory out of git.

**Why →** Mounted as a read-only tmpfs file at `/run/secrets`.

**Interview tip →** In production use a secret manager (Vault, AWS Secrets Manager).

</details>

### Q94. Expose container port range dynamically and discover the mapping. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several instances on one host, avoiding port clashes.

**What I'd do →**
```bash
docker run -d -p 8080 myapp          # random host port
docker port <container> 8080
```
Or `-P` to publish all `EXPOSE`d ports to random ports.

**Why →** `EXPOSE` is documentation only; `-p`/`-P` actually publishes.

**Interview tip →** A common trick question.

</details>

### Q95. You need containers on different hosts to communicate privately. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bridge networks are host-local.

**What I'd do →** Docker Swarm overlay network (`docker network create -d overlay --attachable --opt encrypted app`), which uses VXLAN on UDP 4789 plus ports 2377/7946. Otherwise use Kubernetes CNI, or WireGuard mesh between hosts.

**Why →** Overlay encapsulates traffic across hosts; `--opt encrypted` adds IPsec at a CPU cost.

**Interview tip →** Open firewall ports 2377/tcp, 7946/tcp+udp, 4789/udp.

</details>

### Q96. Make a database reachable only by the app, and the app only through a reverse proxy, on a single host. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Minimise published ports.

**What I'd do →** Publish only the proxy (`80/443`). DB and app have no `ports:` entries and sit on private networks. Verify:
```bash
docker compose ps --format '{{.Name}} {{.Ports}}'
ss -tlnp | grep docker-proxy
```

**Why →** Unpublished services are unreachable from outside the Docker networks.

**Interview tip →** Scan the host with `nmap` to confirm.

</details>

### Q97. Explain what Docker does with iptables when you publish a port. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Interviewers probe whether you know why `ufw` doesn't block it.

**What I'd do →** Inspect:
```bash
iptables -t nat -L DOCKER -n
iptables -L DOCKER-USER -n
```
Docker adds a DNAT rule in the `nat` table for `-p`, a MASQUERADE for egress, and forwarding rules in `filter`. Place your restrictions in `DOCKER-USER`, which is evaluated first.

**Why →** Traffic is DNAT'd in PREROUTING and goes through FORWARD, never reaching the INPUT chain where ufw typically sits. Newer engines can use nftables (experimental in 29).

**Interview tip →** Bind published ports to specific IPs as the simplest control.

</details>

### Q98. Run a one-off migration job with Compose without keeping the container. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need to run a command once in the service context.

**What I'd do →**
```bash
docker compose run --rm api python manage.py migrate
```
Use `profiles: ["tools"]` so it does not start with `up`.

**Why →** `run` creates a new container using the service config, `--rm` removes it afterwards, while `exec` attaches to a running one.

**Interview tip →** Use `exec` for running containers, `run` for fresh ones.

</details>

### Q99. Compose healthcheck for a service depends on the app being ready, not just the port open. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A TCP-open port can still return 503.

**What I'd do →**
```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]
  interval: 10s
  timeout: 3s
  retries: 5
  start_period: 30s
```
Make `/healthz` check dependencies (cache or DB) with care, since cascading failures can occur.

**Why →** `-f` makes curl fail on HTTP errors; `start_period` avoids false negatives during boot.

**Interview tip →** Separate liveness from readiness semantics.

</details>

### Q100. Which DNS settings do you adjust when containers cannot resolve internal company hosts? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Containers on user-defined networks use Docker's embedded DNS at 127.0.0.11, forwarding to the host's resolvers; with systemd-resolved the host stub 127.0.0.53 is unusable.

**What I'd do →**
```json
{"dns": ["10.0.0.2"], "dns-search": ["corp.example.com"]}
```
in `daemon.json`, or per run `--dns 10.0.0.2 --dns-search corp.example.com`. Test with `docker run --rm busybox nslookup intranet.corp.example.com`.

**Why →** Docker substitutes the real upstream resolvers from `/run/systemd/resolve/resolv.conf`.

**Interview tip →** Check VPN split-DNS interactions.

</details>

---

## Security and scanning

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q101. A scan reports 300 CVEs in your base image. How do you triage? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Raw counts are noise; exploitability and fixability matter.

**What I'd do →**
1. `trivy image --severity HIGH,CRITICAL --ignore-unfixed app:1.8.2`.
2. Separate OS packages from app dependencies and see which are actually in the runtime path.
3. Rebuild on a newer base or move to slim/distroless to drop unused packages.
4. Use VEX statements or `.trivyignore` with expiry and justification for accepted risks.

**Why →** Most CVEs live in packages the app never loads.

**Interview tip →** Fix by removing packages first, patching second.

</details>

### Q102. Add vulnerability scanning to CI so critical findings block the merge. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shift-left gating.

**What I'd do →**
```bash
trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed app:$SHA
# or
grype app:$SHA --fail-on high
# or
docker scout cves --exit-code --only-severity critical,high app:$SHA
```
Also scan the Dockerfile with `hadolint` and IaC with `trivy config`.

**Why →** Exit code fails the pipeline; scheduled rescans catch newly disclosed CVEs.

**Interview tip →** A passing scan today does not mean safe tomorrow; rescan deployed images.

</details>

### Q103. Why is mounting `/var/run/docker.sock` into a container dangerous? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** CI agents and dashboards often ask for it.

**What I'd do →** Treat socket access as root on the host: anyone can start a privileged container mounting `/`. Alternatives: rootless Docker or Podman for CI, Kaniko/BuildKit/buildah for in-cluster builds, a socket proxy that allows only specific API calls, or Sysbox/DinD in a sandbox.

**Why →** The API has no per-call authorization by default.

**Interview tip →** Show the one-liner: `docker run -v /:/host --privileged ...` is possible with the socket.

</details>

### Q104. A developer asks for `--privileged`. What do you do? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Usually only a few capabilities or devices are needed.

**What I'd do →** Ask what for. Replace with the narrow need: `--cap-add NET_ADMIN`, `--device /dev/fuse`, `--cap-add SYS_PTRACE` for debugging. For Docker-in-Docker use rootless DinD or Sysbox.

**Why →** `--privileged` gives all capabilities, disables seccomp/AppArmor and exposes host devices.

**Interview tip →** Start from `--cap-drop ALL` and add back.

</details>

### Q105. Harden a container run command for a stateless web API. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Apply defense in depth.

**What I'd do →**
```bash
docker run -d --name api \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --pids-limit 200 --memory 512m --cpus 1 \
  --user 10001:10001 \
  -p 127.0.0.1:8080:8080 myapi:1.8.2
```

**Why →** Each flag removes a class of post-exploitation action.

**Interview tip →** Mention keeping the default seccomp profile on and not disabling it.

</details>

### Q106. A pentester got root inside a container. How bad is it? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Depends on isolation settings.

**What I'd do →** Check: is it privileged? docker.sock mounted? host PID/net namespace? sensitive host paths bind-mounted? dangerous capabilities (`SYS_ADMIN`, `SYS_MODULE`)? kernel version (container escapes like runc CVEs)? If none, root in the container is still uid 0 mapped to host root unless userns is on, but confined by namespaces, cgroups, seccomp and capabilities.

**Why →** Container root is not host root only because of those layers.

**Interview tip →** Use `capsh --print` and `cat /proc/self/status | grep Cap` inside to assess.

</details>

### Q107. Choose a base image strategy for security across the company. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hundreds of images, inconsistent bases.

**What I'd do →** Standardize on minimal maintained bases (distroless, Chainguard, Docker Hardened Images, or Alpine/Debian slim), mirror them, rebuild nightly, sign, SBOM and attach attestations. Block unapproved registries in admission control.

**Why →** Fewer packages mean fewer CVEs and faster patching, and a central pipeline gives one patch point.

**Interview tip →** Explain the tradeoff: debuggability versus attack surface.

</details>

### Q108. How do you generate and use an SBOM for images? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** When a new CVE like Log4Shell drops, you need to know which images contain it.

**What I'd do →**
```bash
syft app:1.8.2 -o spdx-json > sbom.spdx.json
docker buildx build --sbom=true --push -t reg/app:1.8.2 .
grype sbom:sbom.spdx.json
```
Store SBOMs next to images (attached as OCI attestations) and index them.

**Why →** Query inventory instead of rescanning everything.

**Interview tip →** `cosign attest --type spdxjson` signs the SBOM.

</details>

### Q109. How do you prevent a container from gaining privileges via setuid binaries? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** setuid binaries like `su` or `ping` can be abused.

**What I'd do →** `--security-opt no-new-privileges:true` and strip setuid bits in the image: `RUN find / -perm /6000 -type f -exec chmod a-s {} + || true`. Drop capabilities.

**Why →** `no-new-privileges` sets the kernel flag so execve cannot raise privileges.

**Interview tip →** In Kubernetes: `allowPrivilegeEscalation: false`.

</details>

### Q110. Customize seccomp for an app that needs one blocked syscall. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker's default profile blocks about 50 syscalls (e.g., `ptrace` on older kernels, `mount`, `keyctl`).

**What I'd do →**
1. Save the default profile, add only the required syscall to the allow list.
2. `docker run --security-opt seccomp=./profile.json app`.
3. Find needed calls with `strace -c -f` or `docker run --security-opt seccomp=unconfined` only in a test.

**Why →** A tailored profile is safer than `unconfined`.

**Interview tip →** Never ship `seccomp=unconfined`.

</details>

### Q111. Secrets showing up in `docker inspect` output of production containers. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Passed as `-e DB_PASSWORD=...`; anyone with Docker access can read them, and they land in shell history and CI logs.

**What I'd do →** Use secrets files (`/run/secrets`), `--env-file` with strict permissions at minimum, or fetch at start from Vault/AWS SM. Restrict Docker API access and rotate leaked values.

**Why →** `docker inspect` reveals `.Config.Env` in plain text.

**Interview tip →** Orchestrator-native secrets beat env vars.

</details>

### Q112. Enforce that only signed images from your registry can run on hosts. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Docker Engine alone has Docker Content Trust (Notary v1) which is legacy.

**What I'd do →** In Kubernetes use an admission policy (Kyverno `verifyImages`, Sigstore policy-controller, Ratify/Gatekeeper). On standalone hosts configure containerd/Podman `policy.json` with `sigstoreSigned` requirements and restrict registries.

**Why →** Enforcement must happen at pull/admission time, not in docs.

**Interview tip →** Mention that DCT is deprecated in favor of Sigstore/cosign.

</details>

### Q113. Review a Dockerfile: `FROM ubuntu`, `RUN curl ... | bash`, no `USER`. What are your comments? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Several smells.

**What I'd do →**
1. Pin base tag/digest, prefer a smaller base.
2. Do not pipe remote scripts to bash; download, verify checksum, then run.
3. Add a non-root `USER`.
4. Combine layers, clean caches.
5. Add `HEALTHCHECK` and `.dockerignore`; run `hadolint`.

**Why →** Unverified downloads are a supply-chain risk.

**Interview tip →** Lead with the security items, then efficiency.

</details>

### Q114. Run untrusted code (CI jobs from forks) safely with Docker. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard containers share the host kernel.

**What I'd do →** Use stronger isolation: gVisor (`--runtime=runsc`), Kata Containers or Firecracker microVMs, ephemeral runners per job, no socket mounts, no secrets for fork PRs, egress filtering and resource limits.

**Why →** Kernel exploits cross standard container boundaries; sandboxes reduce that surface.

**Interview tip →** Isolation level should match trust level.

</details>

### Q115. Keep Docker daemon API from being exposed insecurely. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** `-H tcp://0.0.0.0:2375` gives unauthenticated root to the network (cryptominers love it).

**What I'd do →** Use the Unix socket locally, or SSH (`DOCKER_HOST=ssh://user@host`), or TLS with client certs on 2376 (`--tlsverify`). Firewall the port and audit with `ss -tlnp | grep dockerd`.

**Why →** Remote API equals remote root.

**Interview tip →** `docker context create` makes SSH access ergonomic.

</details>

### Q116. How do you apply AppArmor or SELinux with Docker? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Mandatory access control adds a second barrier.

**What I'd do →** Ubuntu: default `docker-default` AppArmor profile is applied automatically; custom: `--security-opt apparmor=myprofile`. RHEL/Fedora: enable `"selinux-enabled": true` in `daemon.json`, label mounts with `:z`/`:Z`, and keep `container_t` confinement.

**Why →** Even if processes escape other controls, MAC denies access outside policy.

**Interview tip →** Never run `setenforce 0` to fix a mount error; fix labels.

</details>

### Q117. Detect drift: a running container behaves differently than its image. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Possible compromise or unmanaged changes.

**What I'd do →** `docker diff`, compare running processes (`docker top`), check network connections (`ss -tnp` inside netns via `nsenter`), then use runtime tools (Falco, Tetragon) to alert on unexpected exec/file writes. Make containers read-only to turn drift into failures.

**Why →** Immutability makes anomalies loud.

**Interview tip →** Snapshot with `docker export` for forensics before killing it.

</details>

### Q118. Scan an image that is already deployed, without rebuilding it. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** New CVEs appear after deployment.

**What I'd do →** Scan by digest from the registry (`trivy image reg/app@sha256:...`), schedule daily scans in the registry (ECR enhanced scanning, Harbor, Docker Scout) and alert owners. Rebuild and redeploy when fixes exist.

**Why →** Vulnerabilities are discovered over time; images are static.

**Interview tip →** Track MTTR for patching images as a KPI.

</details>

### Q119. What do you do about Dockerfile `ADD https://...` with no checksum? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unverified remote content.

**What I'd do →**
```dockerfile
ADD --checksum=sha256:9f86d0... https://example.com/tool.tar.gz /tmp/
```
or `curl` and `sha256sum -c`. Prefer package managers with signature verification.

**Why →** Prevents tampering and silent upstream changes.

**Interview tip →** Same principle for `go install` or `pip install` from unpinned sources.

</details>

### Q120. User namespaces: when would you enable `userns-remap`? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** You want container root to map to an unprivileged host UID without going fully rootless.

**What I'd do →** Set `{"userns-remap": "default"}` in `daemon.json`; Docker creates `dockremap` and uses `/etc/subuid`. Be aware: images and volumes get re-owned under a separate storage dir, and features like `--privileged`, host networking and some volume drivers are restricted.

**Why →** Root escape lands as a non-root UID on the host.

**Interview tip →** Rootless mode is the stronger alternative for new setups.

</details>

### Q121. How do you handle the Docker group on shared servers? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Membership of `docker` group equals root.

**What I'd do →** Keep it to a minimal set of admins, use rootless Docker or Podman for developers, or `sudo` with logging. Audit with `getent group docker` and enable auditd rules on `/usr/bin/dockerd` and the socket.

**Why →** The group grants full control of the daemon.

**Interview tip →** Say "docker group is passwordless root."

</details>

### Q122. Proving which source commit and builder produced an image (SLSA). 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Supply-chain audit requirement.

**What I'd do →** Build with BuildKit provenance (`--provenance=mode=max`), or the SLSA GitHub generator, and sign using cosign keyless. Verify with `cosign verify-attestation --type slsaprovenance ...` or `gh attestation verify oci://reg/app@sha256:... -R org/repo`.

**Why →** Provenance links artifact to builder identity and source.

**Interview tip →** Aim for SLSA Level 3 with isolated, ephemeral builders.

</details>

### Q123. Reduce capabilities for an nginx container that binds port 80 as non-root. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** nginx master needs a few caps for root start; unprivileged variant avoids them.

**What I'd do →** Use `nginxinc/nginx-unprivileged` listening on 8080, then `--cap-drop ALL`. If binding low ports is mandatory, `--cap-add NET_BIND_SERVICE` or `--sysctl net.ipv4.ip_unprivileged_port_start=80`.

**Why →** Minimal capabilities with no root user reduce risk.

**Interview tip →** Publish `-p 80:8080` on the host side instead.

</details>

### Q124. Stop leaking build-time secrets through `docker history` in an existing pipeline. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old Dockerfiles use `ARG TOKEN` and `RUN git clone https://$TOKEN@...`.

**What I'd do →** Replace with `RUN --mount=type=secret,id=gh_token ...` or `--mount=type=ssh`; rotate leaked tokens; purge old tags; add a Trivy secret scan to CI.

**Why →** ARG values are stored in image metadata and visible in history.

**Interview tip →** Rotate first.

</details>

### Q125. Compliance wants evidence that containers do not run as root. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need automated audit.

**What I'd do →**
```bash
docker ps -q | xargs docker inspect -f '{{.Name}} user={{.Config.User}} priv={{.HostConfig.Privileged}}'
```
Build a CI check with `hadolint` rule DL3002, `trivy config`, and in Kubernetes enforce Pod Security Standards "restricted" or Kyverno policy `require-run-as-nonroot`. Run `docker-bench-security` for the host.

**Why →** Policy-as-code produces repeatable evidence.

**Interview tip →** CIS Docker Benchmark is the common reference.

</details>

---

## Limits, logging, health checks and restarts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q126. A Java container is OOM-killed at 512 MB. Why isn't the heap the whole story? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** JVM memory includes heap, metaspace, thread stacks, code cache and native buffers.

**What I'd do →**
1. Use container-aware flags: `-XX:MaxRAMPercentage=70`.
2. Check `docker inspect -f '{{.State.OOMKilled}}' c` and `dmesg | grep -i oom`.
3. Add native memory tracking `-XX:NativeMemoryTracking=summary` and size the limit accordingly.

**Why →** The cgroup counts total RSS, not just heap.

**Interview tip →** Leave 25–30% headroom above heap.

</details>

### Q127. How do you cap CPU and memory for a container and what happens on exceed? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Resource governance.

**What I'd do →**
```bash
docker run -d --memory 512m --memory-swap 512m --cpus 1.5 myapp
```
CPU over the limit is throttled (CFS quota); memory over the limit triggers the kernel OOM killer for that cgroup.

**Why →** Setting `--memory-swap` equal to `--memory` disables swap use.

**Interview tip →** CPU = throttling, memory = kill.

</details>

### Q128. Latency spikes though CPU usage looks low (50%). 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** CFS throttling can occur in bursts even at modest averages.

**What I'd do →** Check `cat /sys/fs/cgroup/cpu.stat` inside the container (cgroup v2) for `nr_throttled` and `throttled_usec`. Raise or remove the CPU limit, use `--cpu-shares`/requests only, or tune the app's thread pools to match `--cpus`.

**Why →** A multi-threaded app can burn its 100 ms quota in 20 ms and then stall.

**Interview tip →** Many teams set requests but no CPU limits for this reason.

</details>

### Q129. `docker logs` is empty but the app is running. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** The app logs to a file, not stdout/stderr, or the logging driver is not `json-file`/`local`.

**What I'd do →** `docker inspect -f '{{.HostConfig.LogConfig.Type}}' c`; `docker exec c ls -l /var/log`. Redirect to stdout (`ln -sf /dev/stdout /var/log/app.log`) or configure app logging.

**Why →** `docker logs` reads only for drivers that support it (e.g. not `syslog` before newer dual-logging).

**Interview tip →** Dual logging in 20.10+ keeps `docker logs` working with remote drivers.

</details>

### Q130. Configure log rotation globally. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default `json-file` logs grow unbounded.

**What I'd do →**
```json
{
  "log-driver": "local",
  "log-opts": {"max-size": "20m", "max-file": "5"}
}
```
in `/etc/docker/daemon.json`; `systemctl restart docker`. Existing containers need recreation.

**Why →** The `local` driver compresses and rotates by default.

**Interview tip →** Changes apply only to new containers.

</details>

### Q131. Ship container logs to a central system. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Hosts are ephemeral, logs must be centralised.

**What I'd do →** Options: run a collector (Fluent Bit, Vector, Promtail/Alloy) tailing `/var/lib/docker/containers/*/*-json.log` with Docker metadata; or use a driver (`fluentd`, `gelf`, `awslogs`, `loki` plugin). Collector approach is more resilient because blocking drivers can stall the app.

**Why →** The default `blocking` log mode can slow the app if the remote is down; set `mode=non-blocking` with `max-buffer-size`.

**Interview tip →** Structured JSON logs make queries easy.

</details>

### Q132. Define a good restart policy for a production service. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Containers should survive crashes and reboots.

**What I'd do →** `--restart unless-stopped` (or `always`); `on-failure:5` for batch jobs. In Compose: `restart: unless-stopped`.

**Why →** `unless-stopped` honours an operator's explicit stop across daemon restarts, while `always` restarts regardless. Backoff doubles up to 1 minute for repeated failures.

**Interview tip →** Restart policy is not orchestration; it will not reschedule on another host.

</details>

### Q133. Container is in a restart loop. How do you triage quickly? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Process exits immediately and restart policy keeps trying.

**What I'd do →**
```bash
docker ps -a
docker logs --tail 100 c
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}' c
```
Run it without restart policy and with an overridden entrypoint: `docker run --rm -it --entrypoint sh image`.

**Why →** Exit code 137 = SIGKILL (OOM?), 1 = app error, 126/127 = command not executable/not found.

**Interview tip →** Read exit codes fluently.

</details>

### Q134. Health check marks a container unhealthy but nothing restarts it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Plain Docker only labels health; it doesn't act on it.

**What I'd do →** Use Swarm (restarts unhealthy tasks), Kubernetes probes, or a watcher such as `autoheal` container, or systemd `Restart=` plus `docker kill` hooks. Compose `depends_on: condition: service_healthy` uses it for ordering only.

**Why →** The engine records state `healthy/unhealthy` for orchestrators to use.

**Interview tip →** This is a classic trick question.

</details>

### Q135. Tune a health check for a slow-starting app (2 minutes boot). 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Early failures would mark it unhealthy.

**What I'd do →**
```dockerfile
HEALTHCHECK --start-period=150s --interval=10s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/healthz || exit 1
```

**Why →** Failures during `start-period` don't count toward retries, and the first success ends it.

**Interview tip →** Make sure `wget`/`curl` exists in the image.

</details>

### Q136. A batch container should use at most 2 CPUs on a 32-core node, and pin to specific cores. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Noisy neighbours or NUMA concerns.

**What I'd do →** `--cpus 2` (quota) or `--cpuset-cpus 0-1` (pinning). Use cpuset for latency-sensitive workloads and quota for fair sharing. Check inside with `nproc` (reflects cpuset not quota).

**Why →** Quota limits time; cpuset limits which cores.

**Interview tip →** Apps reading `nproc` size thread pools by cpuset, not quota.

</details>

### Q137. Prevent a fork bomb in a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unbounded PIDs can exhaust the host process table.

**What I'd do →** `docker run --pids-limit 256 ...` and a daemon default via `"default-pids-limit"` where supported. Add `--ulimit nofile=65536:65536` for file descriptors.

**Why →** The pids cgroup controller stops fork calls beyond the limit.

**Interview tip →** Test with `:(){ :|:& };:` in a sandbox.

</details>

### Q138. Compose limits: where do you set CPU/memory in Compose v2? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old `mem_limit` vs `deploy.resources`.

**What I'd do →**
```yaml
services:
  api:
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
        reservations: { memory: 256M }
```
Modern `docker compose` applies `deploy.resources` without Swarm. Verify with `docker stats` and `docker inspect`.

**Why →** One syntax across Compose and Swarm.

**Interview tip →** Reservation is a soft guarantee; limit is a hard cap.

</details>

### Q139. Memory usage seems to grow in `docker stats` but the app is fine. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Page cache counts toward the cgroup's usage.

**What I'd do →** Compare `memory.current` with `memory.stat` (`anon` vs `file`) in `/sys/fs/cgroup`. Newer `docker stats` subtracts inactive file cache. Alert on working set and OOM events, not raw usage.

**Why →** Reclaimable cache is freed under pressure; it isn't a leak.

**Interview tip →** Working set = usage minus inactive_file.

</details>

### Q140. Graceful shutdown with a 60-second drain for in-flight requests. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default stop timeout is 10 s.

**What I'd do →** Handle SIGTERM in the app (stop accepting, finish requests), then `docker stop -t 60` or Compose `stop_grace_period: 60s`. Set `STOPSIGNAL SIGQUIT` for nginx-style graceful stop.

**Why →** Docker sends the stop signal, waits, then SIGKILL.

**Interview tip →** Verify the process is PID 1 or forwards signals.

</details>

### Q141. Monitor containers in production, what do you collect? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need metrics, logs and events.

**What I'd do →** cAdvisor or the engine's Prometheus endpoint (`"metrics-addr": "127.0.0.1:9323"`) for container metrics, node exporter for hosts, logs via Fluent Bit/Loki, alerts on restarts, OOM kills, unhealthy status and throttling. `docker events --filter type=container` for live events.

**Why →** Restarts and OOMs are the earliest failure signals.

**Interview tip →** Alert on rate of restarts, not individual ones.

</details>

### Q142. Different log formats from many containers are hard to query. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unstructured text.

**What I'd do →** Standardize on JSON logs (timestamp, level, service, trace_id) from the app and let the collector add container labels. Use Docker `labels`/`tag` log option to add metadata: `--log-opt tag="{{.Name}}"`.

**Why →** Consistent fields enable dashboards and alerts.

**Interview tip →** Correlation IDs across services are the payoff.

</details>

### Q143. A container's clock drifts or time jumps. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Containers use the host kernel clock; there is no per-container clock (except time namespaces, not used by Docker by default).

**What I'd do →** Fix NTP/chrony on the host. On Docker Desktop, restart the VM after sleep. Never run an NTP daemon inside a normal container (`SYS_TIME` needed).

**Why →** One kernel clock per host or VM.

**Interview tip →** Mention Docker Desktop VM drift after laptop sleep.

</details>

### Q144. Set ulimits for a database container that needs many file descriptors. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Default `nofile` may be too low (or very high in newer systemd setups causing slowdowns).

**What I'd do →**
```bash
docker run --ulimit nofile=65536:65536 --ulimit memlock=-1:-1 db
```
In Compose, `ulimits:` key. Verify with `docker exec db sh -c 'ulimit -n'`.

**Why →** Limits are per-process rlimits set at container start.

**Interview tip →** Elasticsearch needs `vm.max_map_count` set on the host, not in the container.

</details>

### Q145. How do you restrict disk IO of a noisy container? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Backup job saturates disk.

**What I'd do →** `--device-read-bps /dev/nvme0n1:50mb --device-write-bps /dev/nvme0n1:50mb` or `--device-write-iops`. Works on cgroup v2 hosts with the io controller, for block devices (not volumes on network filesystems).

**Why →** Throttles throughput per device.

**Interview tip →** Use the underlying device name from `lsblk`.

</details>

### Q146. Which cgroup version are you on and why does it matter? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Some images or tools assume v1 paths.

**What I'd do →**
```bash
stat -fc %T /sys/fs/cgroup   # cgroup2fs => v2
docker info | grep -i cgroup
```
Update old monitoring agents and JVMs (JDK 15+/8u372+ understand v2).

**Why →** v2 is the unified hierarchy, default on modern distros, and gives better memory/IO accounting.

**Interview tip →** Old Java versions ignoring v2 limits was a notorious pitfall.

</details>

### Q147. Containers should start automatically after a host reboot, but not a one-off debug container. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Restart policies plus the Docker service enabled.

**What I'd do →** `systemctl enable docker`, run services with `--restart unless-stopped`, debug containers with `--rm` and no policy. Check with `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' c`.

**Why →** Restart policy applies when the daemon starts.

**Interview tip →** `live-restore` keeps containers up during daemon restarts.

</details>

### Q148. Explain `live-restore` and when to use it. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Daemon upgrades would otherwise stop all containers.

**What I'd do →** Set `{"live-restore": true}` in `daemon.json`, `systemctl reload docker` (SIGHUP), then upgrades or restarts of dockerd leave containers running (containerd-shim keeps them). Not supported in Swarm mode and certain config changes still need container recreation.

**Why →** Reduces downtime during patching.

**Interview tip →** Cannot be combined with Swarm.

</details>

### Q149. Alert when a container hits OOM. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Need signal before customers complain.

**What I'd do →** Subscribe: `docker events --filter event=oom`, scrape cAdvisor metric `container_oom_events_total`, and alert from Prometheus. Also watch `container_memory_working_set_bytes / limit > 0.9`.

**Why →** OOM kill leaves exit code 137 and `OOMKilled=true` only until the container is removed.

**Interview tip →** Alert on approach, not only on death.

</details>

### Q150. Detect and handle zombie processes in a container. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** PID 1 doesn't reap orphaned children, so `ps` shows `<defunct>`.

**What I'd do →** `docker run --init` (tini) or Compose `init: true`; or bake `tini` in the image as ENTRYPOINT. Verify: `docker exec c ps -eo pid,ppid,stat,cmd | grep Z`.

**Why →** An init reaps children and forwards signals.

**Interview tip →** Tini is bundled with Docker since 1.13.

</details>

---

## Runtimes: containerd, OCI, rootless and Podman

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q151. Explain the path from `docker run` to a running process. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** A layered architecture question.

**What I'd do →** Walk the chain: `docker` CLI → REST API on `dockerd` → `containerd` (images, snapshots, lifecycle) → `containerd-shim-runc-v2` (one per container/pod) → `runc` (creates namespaces, cgroups, execs the process, then exits). The shim keeps stdio and reports exit status.
```bash
ps -ef --forest | grep -E 'dockerd|containerd|runc'
```

**Why →** The shim lets the daemon restart without killing containers.

**Interview tip →** `runc` is the reference OCI runtime implementation.

</details>

### Q152. Kubernetes removed dockershim. What does that mean for your Docker-built images? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Common confusion.

**What I'd do →** Nothing changes for images: they are OCI images and run on containerd or CRI-O. Only the Docker Engine as a node runtime was removed in 1.24 (use `cri-dockerd` if you truly need it). Replace `docker ps` on nodes with `crictl ps` or `nerdctl`.

**Why →** Image and runtime specs are standardized.

**Interview tip →** "Docker images yes, Docker daemon on nodes no."

</details>

### Q153. Run rootless Docker for developers on a shared Linux box. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Remove the root-equivalent daemon.

**What I'd do →**
```bash
sudo apt-get install -y uidmap docker-ce-rootless-extras
dockerd-rootless-setuptool.sh install
systemctl --user enable --now docker
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
loginctl enable-linger $USER
```
Check `/etc/subuid` and `/etc/subgid` ranges exist.

**Why →** The daemon and containers run in a user namespace; a breakout lands as an unprivileged user.

**Interview tip →** Limits: ports below 1024, some network modes and `--privileged`-style features.

</details>

### Q154. Podman vs Docker: when would you pick Podman? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choose by constraints.

**What I'd do →** Podman: daemonless, rootless by default, fork/exec model, systemd integration (Quadlet), pods, runs well on RHEL/Fedora. Docker: ecosystem, Compose maturity, Desktop experience, Swarm. CLI is mostly compatible (`alias docker=podman`); `podman compose` or `docker-compose` with the Podman socket handles Compose files.

**Why →** No central daemon means no single root-owned process to attack or crash.

**Interview tip →** Both consume OCI images and Dockerfiles (Containerfile).

</details>

### Q155. Run a Podman container as a systemd service. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Replace Docker's restart policy with systemd.

**What I'd do →** Use Quadlet. Create `~/.config/containers/systemd/web.container`:
```ini
[Container]
Image=docker.io/library/nginx:1.27
PublishPort=8080:80

[Service]
Restart=always

[Install]
WantedBy=default.target
```
```bash
systemctl --user daemon-reload && systemctl --user start web.service
```

**Why →** `podman generate systemd` is deprecated in favor of Quadlet.

**Interview tip →** Run `loginctl enable-linger` for boot-time user services.

</details>

### Q156. What is the OCI and which specs does it define? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Foundational standards question.

**What I'd do →** The Open Container Initiative defines three specs: Runtime Spec (config.json and lifecycle for bundles, implemented by runc, crun, youki, gVisor), Image Spec (layers, config, manifest) and Distribution Spec (registry API).

**Why →** Interoperability: build with BuildKit, store in any registry, run with any runtime.

**Interview tip →** Mention Docker v2 schema 2 manifests are close cousins of OCI manifests.

</details>

### Q157. Compare `runc`, `crun`, `gVisor` and `Kata`. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Runtime selection by isolation need.

**What I'd do →** `runc` (Go, default), `crun` (C, faster start, lower memory, default in Podman on Fedora), `runsc`/gVisor (user-space kernel intercepting syscalls, stronger isolation, syscall overhead), Kata (each container/pod in a lightweight VM, strongest isolation, higher startup and memory cost). Select via `--runtime` or Kubernetes RuntimeClass.

**Why →** Trade isolation against performance and compatibility.

**Interview tip →** Match the runtime to the tenant trust level.

</details>

### Q158. Use `nerdctl` or `ctr` or `crictl`? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** On a containerd node, which CLI?

**What I'd do →** `crictl` for Kubernetes node debugging (CRI view: pods, containers, `crictl logs`). `nerdctl` for a Docker-like experience on containerd (Compose, build with BuildKit). `ctr` is a low-level debug tool and not meant for daily use. Namespaces: Kubernetes uses `k8s.io`, so `ctr -n k8s.io images ls`.

**Why →** Each has a different abstraction level.

**Interview tip →** Know the `-n k8s.io` namespace.

</details>

### Q159. Move from Docker Engine to containerd on a standalone host. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fewer moving parts, no dockerd.

**What I'd do →**
1. Install containerd, CNI plugins, BuildKit and `nerdctl`.
2. Replace `docker run` flags (mostly compatible) and `docker compose` with `nerdctl compose`.
3. Configure `/etc/containerd/config.toml` (`SystemdCgroup = true`, registry mirrors).
4. Migrate images with `docker save | nerdctl load`.
5. Run side by side for a canary workload first.

**Why →** Smaller attack surface; same runtime Kubernetes uses.

**Interview tip →** Not every Docker feature exists (Swarm, some plugins).

</details>

### Q160. Explain OCI image layout: manifest, config, layers. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Know the data model.

**What I'd do →** Manifest lists config digest and ordered layer digests with media types; the config JSON holds env, entrypoint, rootfs diff_ids and history; each layer is a compressed tar of filesystem changes (with whiteout files for deletes). An index (manifest list) points to per-platform manifests.
```bash
skopeo copy docker://alpine:3.20 oci:/tmp/alpine:3.20
cat /tmp/alpine/index.json | jq
```

**Why →** Content addressing gives deduplication and integrity.

**Interview tip →** Layer digest (compressed) differs from diff_id (uncompressed).

</details>

### Q161. A rootless Podman container cannot bind port 80. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Unprivileged users cannot bind ports below 1024 by default.

**What I'd do →** `sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80` (persist in `/etc/sysctl.d/`), or publish 8080 and front it with a reverse proxy or socket activation.

**Why →** Kernel restriction on privileged ports.

**Interview tip →** The sysctl affects all users; document it.

</details>

### Q162. Generate Kubernetes YAML from a running Podman pod. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Bridge from local dev to cluster.

**What I'd do →**
```bash
podman pod create --name shop -p 8080:80
podman run -d --pod shop --name web nginx:1.27
podman generate kube shop > shop.yaml
podman kube play shop.yaml
```

**Why →** Podman understands pods natively.

**Interview tip →** Treat generated YAML as a starting point to edit.

</details>

### Q163. What are the storage drivers and which would you use? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Know the options.

**What I'd do →** `overlay2` is the default and recommended (ext4 or xfs with `d_type`); newer Docker uses the containerd image store with the `overlayfs` snapshotter; `btrfs`/`zfs` when the host already uses them for snapshots; `devicemapper` and `aufs` are removed/deprecated. Check `docker info -f '{{.Driver}}'`.

**Why →** overlay2 has the best performance and kernel support.

**Interview tip →** Moving `/var/lib/docker` requires `data-root` in `daemon.json`.

</details>

### Q164. Use buildah or Kaniko to build images without a Docker daemon in CI. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Cannot expose the Docker socket in shared runners.

**What I'd do →** Kaniko runs in a pod without privileges:
```bash
/kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile \
  --destination reg.example.com/app:$CI_COMMIT_SHA --cache=true
```
Or `buildah bud` rootless, or BuildKit rootless (`rootlesskit buildkitd`). Note that Kaniko's upstream is archived; forks and BuildKit are common alternatives now.

**Why →** No privileged daemon needed.

**Interview tip →** Say you verify maintenance status of tools before adopting.

</details>

### Q165. Run Docker-in-Docker safely for CI. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Jobs need to run `docker build` and `docker run`.

**What I'd do →** Prefer socket-less builds (BuildKit, Kaniko). If DinD is needed, use `docker:dind-rootless` or Sysbox runtime so inner Docker does not require `--privileged`, give each job its own ephemeral daemon, and use TLS (`DOCKER_TLS_CERTDIR=/certs`).

**Why →** Privileged DinD is host-root equivalent.

**Interview tip →** Socket mounting shares the host daemon; DinD isolates it but is privileged.

</details>

### Q166. How are containers isolated at kernel level? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fundamental concept.

**What I'd do →** Namespaces (pid, net, mnt, uts, ipc, user, cgroup) restrict what a process sees; cgroups limit what it can use; capabilities, seccomp and LSMs (AppArmor/SELinux) restrict what it can do.
```bash
lsns -p $(docker inspect -f '{{.State.Pid}}' web)
```

**Why →** Containers are regular processes with restricted views, not VMs.

**Interview tip →** Shared kernel is the key difference from VMs.

</details>

### Q167. Check how a container's cgroup limits look on the host. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Verify limits are really applied.

**What I'd do →**
```bash
CID=$(docker inspect -f '{{.Id}}' web)
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/cpu.max
```
(systemd cgroup driver, cgroup v2 paths).

**Why →** Proves the engine translated flags into kernel limits.

**Interview tip →** Use `systemd-cgls` or `systemd-cgtop` for a live view.

</details>

### Q168. Pull images through a local cache for a remote office. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Slow WAN, repeated pulls.

**What I'd do →** Run a registry in pull-through mode:
```yaml
proxy:
  remoteurl: https://registry-1.docker.io
```
Configure clients: `{"registry-mirrors": ["https://mirror.office.example.com"]}`. Or Harbor proxy-cache projects.

**Why →** Layers download once per site.

**Interview tip →** `registry-mirrors` applies to Docker Hub only.

</details>

### Q169. Verify what capabilities a running container has and why it matters. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Audit privileges.

**What I'd do →**
```bash
docker inspect -f '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}' c
docker exec c grep Cap /proc/1/status
capsh --decode=00000000a80425fb
```

**Why →** Default set includes `NET_RAW`, `CHOWN`, `SETUID` etc.; drop what the app doesn't need.

**Interview tip →** `NET_RAW` allows ARP spoofing in shared bridges.

</details>

### Q170. Migrate Compose files to Kubernetes. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Moving up from a single host.

**What I'd do →** Use `kompose convert` as a first draft, then hand-edit: replace `depends_on` with readiness probes, volumes with PVCs, `ports` with Services/Ingress, secrets with Secret objects, resources with requests/limits. Package with Helm or Kustomize.

**Why →** Tools translate structure, not operational semantics.

**Interview tip →** Check that images can run as non-root; Pod Security rules are stricter.

</details>

### Q171. Why might `docker build` and `podman build` give different results? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Builders are not identical.

**What I'd do →** Differences: Docker uses BuildKit (cache/secret mounts, `# syntax` frontends), Podman uses Buildah (supports most BuildKit mount types but not every feature); default image format (Podman builds OCI, may drop HEALTHCHECK unless `--format docker`), rootless UID mapping affecting file ownership, and short-name resolution via `registries.conf`.

**Why →** `HEALTHCHECK` isn't part of the OCI image config.

**Interview tip →** Use fully qualified image names with Podman.

</details>

### Q172. Configure registry mirrors and insecure registries for containerd. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Private registry with self-signed TLS.

**What I'd do →** Preferred: trust the CA. With containerd use `config_path` and `/etc/containerd/certs.d/reg.example.com/hosts.toml`:
```toml
server = "https://reg.example.com"
[host."https://reg.example.com"]
  ca = "/etc/containerd/certs.d/reg.example.com/ca.crt"
```
For Docker: place the CA in `/etc/docker/certs.d/reg.example.com/ca.crt`, no restart needed.

**Why →** Avoids disabling TLS verification.

**Interview tip →** `insecure-registries` is a last resort.

</details>

### Q173. Explain how image layers are stored and shared on a node. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Understand deduplication.

**What I'd do →** Each layer is unpacked once into a snapshot under `/var/lib/docker/overlay2/<id>` (or containerd snapshotter). Containers stack read-only lower dirs with one writable upper dir via overlayfs. Images sharing a base reuse the same lower layers.
```bash
docker inspect -f '{{json .GraphDriver.Data}}' web | jq
```

**Why →** Saves disk and speeds up pulls/starts.

**Interview tip →** `docker system df` double-counts shared layers in "reclaimable".

</details>

### Q174. Use Podman to run a pod-like stack without root and expose it. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Developer laptop or small server.

**What I'd do →**
```bash
podman network create appnet
podman run -d --name db --network appnet -v pgdata:/var/lib/postgresql/data postgres:17
podman run -d --name api --network appnet -p 8080:8080 localhost/api:1
```
Use netavark/aardvark-dns for name resolution and `podman volume` for storage.

**Why →** Works rootless with slirp4netns or pasta networking.

**Interview tip →** Rootless networking is slower; mention `pasta` as the newer default.

</details>

### Q175. Checkpoint and restore a container. When is it useful? 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast warm start or forensic capture.

**What I'd do →** Podman: `podman container checkpoint web -e /tmp/web.tar.gz` then `podman container restore -i /tmp/web.tar.gz`. Docker: experimental `docker checkpoint create`. Both rely on CRIU, which has limits with network connections, GPUs and some kernel features.

**Why →** Useful for slow-starting apps and forensic snapshots, not a replacement for HA.

**Interview tip →** Kubernetes forensic checkpointing is an alpha/beta feature in kubelet.

</details>

---

## CI/CD and production rollouts

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q176. Design a GitHub Actions workflow that builds, scans and pushes an image. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Standard CI for containers.

**What I'd do →**
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: docker/setup-buildx-action@v3
  - uses: docker/login-action@v3
    with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
  - uses: docker/build-push-action@v6
    with:
      push: false
      load: true
      tags: ghcr.io/acme/app:${{ github.sha }}
      cache-from: type=gha
      cache-to: type=gha,mode=max
  - run: trivy image --exit-code 1 --severity CRITICAL ghcr.io/acme/app:${{ github.sha }}
  - run: docker push ghcr.io/acme/app:${{ github.sha }}
```

**Why →** Scan before push so bad images never reach the registry.

**Interview tip →** Pin actions by SHA in sensitive repos.

</details>

### Q177. Zero-downtime deploy on a single Docker host. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** No orchestrator available.

**What I'd do →** Blue/green with a reverse proxy: start the new version as `api-green`, wait for healthy, switch the proxy upstream (Traefik label or nginx include + `nginx -s reload`), drain and stop `api-blue`. Tools such as `docker rollout` plugin or Compose + Traefik automate this.

**Why →** Proxy switch is atomic; old containers finish in-flight requests.

**Interview tip →** Also mention DB migrations must be backward compatible.

</details>

### Q178. Deploy via GitOps with image digests. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Want auditable deployments.

**What I'd do →** CI builds and pushes `app:$SHA`, then updates the deployment repo (Kustomize `images:` or Helm values) to the digest via PR. Argo CD or Flux syncs. Image Updater/Flux image automation can also do it.

**Why →** Git history is the deployment history; rollback is a revert.

**Interview tip →** Separate app repo from environment repo.

</details>

### Q179. Canary a new image on a Swarm service. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Gradual rollout.

**What I'd do →**
```bash
docker service update --image reg/app:1.9.0 \
  --update-parallelism 1 --update-delay 30s \
  --update-failure-action rollback --update-monitor 60s app
docker service rollback app
```

**Why →** Rolls task by task and rolls back automatically on failed health.

**Interview tip →** For weighted traffic splitting you need a service mesh or Traefik weighted services.

</details>

### Q180. Docker build is flaky in CI due to network errors on package installs. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** External mirrors fail intermittently.

**What I'd do →** Use internal package proxies (Artifactory/Nexus/Athens), add retries (`pip --retries 5`, `apt -o Acquire::Retries=3`), vendor dependencies for critical paths, and use cache mounts.

**Why →** Reduces external dependency on the critical path.

**Interview tip →** Flaky builds are an availability problem for the whole team.

</details>

### Q181. Run database migrations in the deploy flow of containerized apps. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Old and new code run concurrently during rollout.

**What I'd do →** Run migrations as a one-off job (`docker compose run --rm migrate`, a Kubernetes Job or init step) before switching traffic, using expand/contract: add columns first, deploy, then drop old ones later. Take a backup before destructive changes.

**Why →** Backward-compatible schemas enable rollback.

**Interview tip →** Never run migrations in every replica's entrypoint.

</details>

### Q182. How do you keep dev, CI and prod consistent? 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** "Works on my machine."

**What I'd do →** Same Dockerfile everywhere, same image promoted across environments, config via env, dev containers/Compose mirroring prod dependencies versions (postgres:17, not "whatever is on the laptop").

**Why →** Differences in environment are the root of most bugs.

**Interview tip →** Build once, promote by digest.

</details>

### Q183. Estimate and reduce deployment time for a 1.5 GB image across 50 nodes. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Registry egress and pull time dominate.

**What I'd do →** Shrink the image, order layers so base layers are shared, use a regional mirror or pull-through cache, pre-pull (`DaemonSet` or `docker pull` in bake), enable lazy pulling (stargz/SOCI, Nydus) and parallel layer downloads (`max-concurrent-downloads`). Use zstd compression for faster extraction.

**Why →** Pull + unpack time often exceeds app start.

**Interview tip →** Measure first: pull time vs extract vs start.

</details>

### Q184. Handle environment-specific build args without separate images. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Front-end apps embedding API URLs at build time.

**What I'd do →** Use runtime config: serve `/config.js` generated at container start from env vars via entrypoint (`envsubst`), so one image runs everywhere.

**Why →** Avoids per-environment builds.

**Interview tip →** Build-time injection is only acceptable for non-env-specific values.

</details>

### Q185. Test Dockerfile changes safely before merging. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** A broken Dockerfile blocks everyone.

**What I'd do →** PR pipeline: `hadolint Dockerfile`, `docker buildx build --check`, build for all target platforms without pushing, run container structure tests (`container-structure-test`) or a smoke test hitting `/healthz`.

**Why →** Catches issues early.

**Interview tip →** `docker build --check` validates against Dockerfile build checks.

</details>

### Q186. Rollback strategy when a new image passes health checks but errors under load. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Health checks are shallow.

**What I'd do →** Monitor error rate and latency SLOs during rollout, auto-rollback via progressive delivery (Argo Rollouts, Flagger) or Swarm `--update-monitor`, keep the previous digest in the deployment record, and use feature flags to separate deploy from release.

**Why →** Behavioural regressions need metrics-based gates.

**Interview tip →** Describe rollout gates on SLIs.

</details>

### Q187. How do you manage Docker daemon configuration across a fleet? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Consistency of `daemon.json`.

**What I'd do →** Manage with Ansible/Terraform/cloud-init, version in git, validate with `dockerd --validate --config-file /etc/docker/daemon.json`, and roll changes in batches. Many settings need a restart; use `live-restore` to reduce impact.

**Why →** Drift in log options or address pools causes outages.

**Interview tip →** Test config before restart; a bad JSON stops dockerd.

</details>

### Q188. Docker Hub is down during a production deploy. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** External dependency on the critical path.

**What I'd do →** Deploy from our own registry (images already mirrored), check nodes have cached layers, and for the future mirror all base images and use pull-through caches.

**Why →** Production deployment should not depend on third parties.

**Interview tip →** Reference the 2021-era rate limit and outages as motivation.

</details>

### Q189. Cut CI costs for container builds. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Heavy builds on every push.

**What I'd do →** Path filters to build only changed services, remote caches, smaller images, multi-stage targets, avoid multi-arch on every PR (build arm64 only on main/tags), and use larger fewer runners with persistent BuildKit.

**Why →** Time and compute are proportional to rebuilt layers.

**Interview tip →** Track build minutes per service.

</details>

### Q190. A new junior pushed an image with a secret. Incident steps. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Secret exposure incident.

**What I'd do →**
1. Revoke and rotate the secret immediately.
2. Identify exposure: pull logs, who pulled the tag.
3. Delete image tags and untagged manifests, run registry GC.
4. Add `.dockerignore` and secret scanning.
5. Write a blameless postmortem.

**Why →** Rotation is the only real remediation.

**Interview tip →** Time-to-rotate matters more than time-to-delete.

</details>

### Q191. Manage multiple environments with one Compose repo. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** dev/staging/prod variants.

**What I'd do →** `compose.yaml` base + `compose.<env>.yaml` overrides, `--env-file .env.<env>`, and `COMPOSE_PROJECT_NAME` per environment so networks and volumes are separated. Validate with `docker compose config`.

**Why →** Project names isolate resources on the same host.

**Interview tip →** Use profiles for optional services (e.g., `debug`).

</details>

### Q192. Make image builds faster for a monorepo with 20 services. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Shared context makes everything rebuild.

**What I'd do →** Use `docker buildx bake` with a `docker-bake.hcl` defining targets and shared base, per-service `.dockerignore` (via `Dockerfile.dockerignore`), `COPY` only the service path plus shared libs, remote cache and `--set *.cache-to`. Detect changed services in CI with git diff.

**Why →** Bake builds targets in parallel with shared cache.

**Interview tip →** Context size matters: use named contexts to limit it.

</details>

### Q193. Patch a critical OpenSSL CVE across 80 images. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fleet-wide urgent patch.

**What I'd do →** Query the SBOM index to find affected images, rebuild the golden base first, then trigger downstream rebuilds (Renovate/Dependabot or a pipeline fan-out), scan, sign and roll out by priority (internet-facing first). Track completion in a dashboard.

**Why →** Central bases and inventory make this a day, not a month.

**Interview tip →** Mention measuring blast radius with SBOMs.

</details>

### Q194. Where do you run containers for a small team: Compose on a VM, ECS, or Kubernetes? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Choose by scale and team skill.

**What I'd do →** A few services and small team: Compose on a VM (or ECS Fargate/Cloud Run if on cloud) with backups and monitoring. Kubernetes when you need autoscaling, multi-team multi-tenant, rich rollout strategies, or many services. Avoid operating Kubernetes without capacity to maintain it.

**Why →** Operational cost should match need.

**Interview tip →** Defend the choice with requirements, not trends.

</details>

### Q195. Run scheduled jobs in containers. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Periodic tasks like cleanup.

**What I'd do →** Use the host's cron/systemd timer calling `docker run --rm job:1.0`, or a scheduler container, or Kubernetes CronJob. Avoid running `crond` inside app containers.

**Why →** Jobs get their own limits, logs and exit codes.

**Interview tip →** Make jobs idempotent.

</details>

### Q196. Reduce attack surface of CI build environment itself. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Builders have registry credentials.

**What I'd do →** Ephemeral runners, short-lived OIDC credentials to the registry (no static keys), least-privilege push scopes per repo, no socket sharing, network egress allow-lists, and signed provenance for builds.

**Why →** A compromised builder can push poisoned images.

**Interview tip →** OIDC federation removes stored registry passwords.

</details>

### Q197. How do you version and publish a base image used by other teams? 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Internal platform responsibility.

**What I'd do →** Publish immutable dated tags (`2026.11.09`) and moving tags (`3.12`), changelog, SBOM and signatures, a deprecation policy, and automated PRs to consumers. Monitor adoption.

**Why →** Consumers choose stability or freshness knowingly.

**Interview tip →** Treat internal images as products.

</details>

### Q198. Containerize a legacy app that writes config on startup and expects root. 🔴

<details>
<summary><b>Answer</b></summary>

**Situation read →** Not cloud-native.

**What I'd do →** Wrap with an entrypoint that templates config from env into a writable dir, run as non-root with `chown` on needed dirs at build time, mount state on a volume, send logs to stdout, and add a health script. If root is unavoidable, drop capabilities, use userns/rootless and read-only mounts.

**Why →** Incremental hardening beats rewriting.

**Interview tip →** Document the exceptions and set a remediation plan.

</details>

### Q199. Choose between `docker compose watch`, bind mounts and rebuilds for developer inner loop. 🟡

<details>
<summary><b>Answer</b></summary>

**Situation read →** Fast feedback matters.

**What I'd do →** `develop.watch` with `sync` for interpreted code, `sync+restart` for config changes, `rebuild` for dependency manifest changes:
```yaml
develop:
  watch:
    - { action: sync, path: ./src, target: /app/src }
    - { action: rebuild, path: package.json }
```

**Why →** Avoids bind-mount perf and ownership problems while keeping speed.

**Interview tip →** Run with `docker compose up --watch`.

</details>

### Q200. Walk me through taking a container app from laptop to production at a high level. 🟢

<details>
<summary><b>Answer</b></summary>

**Situation read →** Open-ended end-to-end question.

**What I'd do →**
1. Write a Dockerfile (multi-stage, non-root, pinned base, healthcheck).
2. Compose for local dependencies.
3. CI: lint, build, test, scan, SBOM, sign, push by SHA.
4. Promote the same digest through environments via GitOps.
5. Run with limits, probes, centralized logs and metrics.
6. Plan rollbacks, backups and patch cadence.

**Why →** Every step reduces a class of production failures.

**Interview tip →** Structure the answer as build, ship, run.

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
