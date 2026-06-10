---
name: dockerfile-hardening-guide
version: 1.4.2
description: Harden Dockerfiles for security and size — non-root users, pinned bases, multi-stage builds, minimal layers, and supply-chain hygiene.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - docker
  - security
  - containers
  - devops
  - hardening
---
# dockerfile-hardening-guide

Turn a working-but-risky Dockerfile into a small, reproducible, least-privilege
image. Use this when reviewing or authoring container builds.

## Checklist

### Base image

- **Pin by digest**, not a moving tag: `FROM node:20.14.0-bookworm-slim@sha256:...`.
  Tags like `latest` make builds non-reproducible.
- Prefer `-slim` or `distroless` over full distros. Fewer packages means fewer
  CVEs and a smaller image.
- Use multi-stage builds so compilers and dev dependencies never reach the
  runtime image.

### Run as non-root

Containers default to UID 0. Create and switch to an unprivileged user:

```dockerfile
RUN groupadd --gid 10001 app \
 && useradd  --uid 10001 --gid app --no-create-home app
USER 10001:10001
```

Use a numeric UID in `USER` so Kubernetes `runAsNonRoot` checks pass even if the
username is not resolvable.

### Minimize layers and content

- Combine related `RUN` steps and clean caches in the same layer:

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

- Add a `.dockerignore` (`.git`, `node_modules`, `*.env`, `tests/`) so secrets
  and bulk never enter the build context.
- Copy only what you need. `COPY . .` drags in everything.

### Secrets and build args

- Never `ARG`/`ENV` a secret — it persists in image history and `docker history`
  reveals it. Use BuildKit secret mounts instead:

```dockerfile
RUN --mount=type=secret,id=npmrc npm ci
```

- Do not bake tokens into layers. Inject runtime config via the orchestrator.

### Multi-stage example

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.22-bookworm@sha256:... AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

## Runtime hardening

- Add a `HEALTHCHECK` so orchestrators detect a wedged process.
- Set `WORKDIR` explicitly; never run from `/`.
- Drop Linux capabilities and set a read-only root filesystem at the orchestrator
  level (`securityContext.readOnlyRootFilesystem: true`).

## Scoring table

| Control                 | Risk if missing            | Priority |
|-------------------------|----------------------------|----------|
| Non-root `USER`         | container escape blast radius | high  |
| Pinned base digest      | non-reproducible / poisoned base | high |
| No secrets in layers    | credential leak            | high     |
| `.dockerignore`         | secret/source leak, bloat  | medium   |
| Multi-stage build       | bloated attack surface     | medium   |
| `HEALTHCHECK`           | silent failures            | low      |

## Agent guidance

- When reviewing, output a prioritized list (high → low), not a flat dump.
- Verify the base image still builds after switching to `-slim`/distroless.
- Confirm the app actually runs as the chosen UID before approving.
