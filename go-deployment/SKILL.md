---
name: go-deployment
description: Containerize and ship a Go (golang) service — multi-stage Dockerfiles, static builds (CGO_ENABLED=0), distroless/scratch non-root images, supply-chain hardening (SBOM, cosign signing, pinned digests), and Kubernetes runtime (liveness/readiness probes, graceful shutdown, GOMAXPROCS/GOMEMLIMIT for container limits). Use when writing a Dockerfile, building/optimizing images, or deploying a Go service.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Go Deployment & Containerization

Get a Go binary into a small, secure, container that behaves correctly under
an orchestrator. Go's single static binary makes this easy — a correct image
is **5–20 MB**, not 300 MB+. Pairs with [[go-tooling-ci]] (build in CI),
[[go-security]]/[[go-security-architecture]] (hardening), and
[[go-observability]] (health/probes).

## The static binary is the whole trick
Go compiles to one self-contained binary, so the runtime image needs almost
nothing. Build it static and stripped:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -ldflags="-w -s" -trimpath -o /app ./cmd/server
```
- **`CGO_ENABLED=0`** → no libc dependency → runs on `scratch`/distroless.
- **`-ldflags="-w -s"`** strips debug info/symbol table (smaller binary).
- **`-trimpath`** removes local filesystem paths (reproducible, no info leak).
- Set `GOARCH` to the target (or build multi-arch with `buildx`).

## Multi-stage Dockerfile (the standard)
Compile in a full toolchain stage, copy only the binary into a tiny runtime.

```dockerfile
# ---- build ----
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                      # cache deps layer (changes rarely)
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -trimpath -o /app ./cmd/server

# ---- runtime ----
FROM gcr.io/distroless/static:nonroot
COPY --from=build /app /app
USER nonroot:nonroot                     # UID 65532, no shell, no pkg manager
EXPOSE 8080
ENTRYPOINT ["/app"]
```
Key moves: copy `go.mod`/`go.sum` and `go mod download` **before** the source
so the dependency layer caches across builds; final stage carries only the
binary.

## Choosing the runtime base
| Base | Contains | Use when |
|------|----------|----------|
| **`distroless/static:nonroot`** | CA certs, tzdata, `/etc/passwd`, nonroot user — **no shell/pkg mgr** | **Default.** You make HTTPS calls / need certs+tz. |
| **`scratch`** | Nothing | Absolute minimum; you must `COPY` CA certs + add a nonroot user yourself. |
| **`alpine`** | musl + shell + apk | Only if you genuinely need a shell or musl libs; bigger attack surface. |

Prefer **distroless `:nonroot`** — smallest sane default with certs and a
non-root user baked in. Avoid shipping a shell/package manager to production.

## Container hardening checklist
- **Run as non-root** (`USER nonroot` / `securityContext.runAsNonRoot: true`).
- **Read-only root filesystem** (`readOnlyRootFilesystem: true`); mount a
  writable `emptyDir` only if the app must write temp files.
- **Drop all Linux capabilities** (`capabilities.drop: [ALL]`); a Go web
  service needs none.
- **No shell/pkg manager** in the image (distroless/scratch gives this free).
- **Pin the base by digest** (`golang:1.26@sha256:...`) for reproducibility.
- `seccompProfile: RuntimeDefault`; no `privileged`, no host mounts.

## Tell the Go runtime about container limits
Inside a container with CPU/memory limits, the Go runtime by default sees the
**host's** resources, which causes too many GC threads and OOM kills. Fix:
- **`GOMAXPROCS`** → match the CPU limit. Use Uber's
  [`automaxprocs`](https://github.com/uber-go/automaxprocs) (blank import) to
  set it from the cgroup automatically, or set the env var to the CPU limit.
- **`GOMEMLIMIT`** → set a soft memory limit a bit below the container's hard
  limit (e.g. `GOMEMLIMIT=900MiB` for a 1Gi pod) so the GC works harder before
  the kernel OOM-kills you.

```go
import _ "go.uber.org/automaxprocs"   // sets GOMAXPROCS from cgroup CPU quota
```

## Kubernetes runtime wiring
- **Probes** (see [[go-observability]] for the endpoints):
  - **liveness** → cheap `/healthz`; restart if it fails.
  - **readiness** → `/readyz` that checks deps (DB, etc.); gates traffic.
  - **startup** → for slow boots, so liveness doesn't kill a still-initializing
    pod.
- **Graceful shutdown**: the app must catch `SIGTERM` (`signal.NotifyContext`)
  and drain in-flight work ([[go-concurrency]], [[go-http-api]]). Set
  `terminationGracePeriodSeconds` to cover your drain time; on rolling deploys
  flip readiness to false first, then drain.
- **Config via env** (12-factor); **secrets** from a secret manager / mounted,
  never baked into the image ([[go-security-architecture]]).
- Set **resource `requests`/`limits`** and align `GOMAXPROCS`/`GOMEMLIMIT` to
  them.

## Supply-chain hardening
SBOMs and signed images are now table stakes (and a federal requirement for
some buyers). Build a chain of trust from source to running container:
- **Generate an SBOM** per image (`syft`, or **`ko`** which emits SPDX SBOMs by
  default; `--sbom=cyclonedx` for CycloneDX).
- **Sign images with [cosign](https://docs.sigstore.dev/)** — prefer **keyless**
  signing (OIDC via GitHub Actions → Fulcio short-lived cert → Rekor
  transparency log); no long-lived keys to manage.
- **Pin** base images and deps by digest; commit `go.sum`; run `govulncheck` in
  CI ([[go-tooling-ci]], [[go-security]]).
- **Enforce** signatures at admission (e.g. policy-controller / Kyverno) so
  unsigned images can't run.

## `ko` — Dockerfile-free Go images
For pure-Go services, [`ko`](https://ko.build/) builds a minimal image straight
from a Go import path — no Dockerfile, distroless base, multi-arch, **SBOM by
default**, and integrates with cosign. Great default for Go-only services;
reach for a Dockerfile when you need non-Go assets in the image.

```bash
KO_DOCKER_REPO=ghcr.io/you/app ko build ./cmd/server
```

## Anti-patterns
- Single-stage build shipping the whole `golang` toolchain (~1 GB image).
- `FROM alpine` (or full distros) with a shell/pkg manager in prod.
- Running as root / writable root FS / `--privileged`.
- Ignoring container limits → Go spawns host-CPU threads, GC thrash, OOM kills.
- Unsigned images, no SBOM, `:latest` base tags (unpinned, unreproducible).
- Baking secrets or config into the image.
- No graceful shutdown → dropped requests on every deploy.

## References
- [Multi-stage Go Docker builds (5–20 MB images)](https://oneuptime.com/blog/post/2026-01-07-go-docker-multi-stage/view)
- [Distroless static:nonroot](https://github.com/GoogleContainerTools/distroless)
- [ko — container images for Go](https://ko.build/)
- [Sigstore cosign (keyless signing)](https://docs.sigstore.dev/)
- [Make Go container-resource-limit aware (GOMAXPROCS/GOMEMLIMIT)](https://kupczynski.info/posts/go-container-aware/)
- [Kubernetes liveness/readiness/startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
