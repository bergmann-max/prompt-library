# Containerization & Docker Best Practices
## Core Principles

1. **Immutability** - Never modify running containers. New change = new image. Use semantic versioning tags. Enables instant rollbacks.
2. **Portability** - Externalize config via env vars. Avoid environment-specific values baked in. Use multi-platform base images when needed.
3. **Isolation** - Design containers by function, not by raw process count. A container may host multiple tightly-coupled processes (e.g. an app plus its sidecar helper) as long as they share one lifecycle. For anything beyond a single-binary app, use a proper init system as PID 1 (`tini` for simple reaping/signaling, `s6-overlay` or `pebble` for multi-process supervision, logging, and graceful shutdown) instead of a hand-rolled `entrypoint.sh` job-control script. Use container networking, resource limits, named volumes.
4. **Efficiency** - Smaller images = faster builds/pulls, less attack surface, lower cost. Ongoing optimization, not one-time.

## Dockerfile Best Practices
### Multi-Stage Builds (Golden Rule)

Separate build-time from runtime dependencies. Name stages descriptively. Copy only necessary artifacts between stages.

```dockerfile
# Stage 1: Dependencies
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Stage 2: Build
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Test
FROM build AS test
RUN npm run test && npm run lint

# Stage 4: Production
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Base Image Selection

- Prefer `alpine`, `slim`, `distroless` variants
- Use official language-specific images
- Pin versions - never `latest` in production. Note that even a version tag like `1.0.x` is still a floating reference, not a true pin - for reproducible builds pin by digest (`image@sha256:...`)
- Prefer base images with clear stability signals (e.g. `edge`/`stable` suffixes, LTS tracks) over bleeding-edge/nightly builds
- Update regularly for security patches, but treat every update as a risk trade-off (known CVE fix vs. newly introduced, unreviewed code) - prefer maintainers that backport CVE fixes instead of always bumping to the newest upstream release

### Layer Optimization

- Order: least-changing -> most-changing instructions
- Combine `RUN` commands, clean up in same layer
- Place `COPY .` last

```dockerfile
# BAD: Multiple layers, no cleanup
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get clean

# GOOD: Single layer with cleanup
RUN apt-get update && \
    apt-get install -y python3 python3-pip && \
    pip3 install flask && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
```

### .dockerignore

Always maintain. Exclude: `.git*`, `node_modules`, build artifacts, `.env.*`, `*.log`, IDE files, docs, tests.

```dockerignore
.git*
node_modules
dist
build
.env.*
*.log
coverage
.vscode
.idea
.DS_Store
*.md
test/
tests/
```

### COPY Strategy

Copy dependency manifests first (caching), then source. Be specific - avoid `COPY . .` when possible.

```dockerfile
COPY package*.json ./
RUN npm ci
COPY src/ ./src/
COPY config/ ./config/
```

### User & Port

Run as non-root. Create dedicated user. Use `EXPOSE` for documentation.

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 8080
CMD ["node", "dist/main.js"]
```

**UID choice matters, not just non-root:**

- Avoid UID < 1000 - reserved for root and system users on most distros
- Avoid the obvious low end of the dynamic range too (e.g. UID 1000/1001), since these commonly collide with real host users (e.g. the default `ubuntu` user is UID 1000 on many cloud images) - a container breakout would then land on an existing host identity
- Choose a high, non-reserved UID with low collision probability with the host (e.g. UID/GID in the `500000+` range, well above the typical dynamic host-user range of `1000-60000`)
- Be consistent with UID choices across containers that share volumes, to avoid `permission denied` errors on mounts

### CMD & ENTRYPOINT

- `ENTRYPOINT` = executable, `CMD` = default args
- Always use exec form (`["cmd", "arg"]`) for proper signal handling
- Shell form breaks PID 1 signal propagation

### Environment Variables

Externalize config. Provide defaults with `ENV`. Never hardcode secrets.

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ARG BUILD_VERSION
ENV APP_VERSION=$BUILD_VERSION
```

## Security
### Non-Root (Required)

Always `USER non-root`. Root containers can escape to host via runtime vulnerabilities.

### Minimal Base Images

Fewer packages = fewer CVEs. Prefer: `distroless` > `alpine` > `slim` > full distro.

**But small/distroless is not automatically secure.** All images - including distroless ones - must stay auditable: most vulnerability scanners rely on package-manager metadata (e.g. `dpkg` status) inside the image to detect installed software. Top-down distroless builds that manually strip this metadata can produce **CVE false negatives** (scanner reports zero vulnerabilities because it can't see the packages at all), which is worse than a false positive. When building distroless images, use tooling that preserves auditable package metadata (e.g. Chisel for Ubuntu-based slices) rather than ad-hoc file copying.

### Scanning & Linting

Integrate into CI:
- `hadolint` - Dockerfile linting
- `trivy` / `snyk` - image vulnerability scanning
- Fail builds on critical vulnerabilities

### Image Signing

Use Cosign or Docker Content Trust. Verify before deploying.

```bash
cosign sign --key cosign.key myregistry.com/myapp:v1.0.0
cosign verify --key cosign.pub myregistry.com/myapp:v1.0.0
```

### Capabilities & Read-Only FS

Drop all unnecessary capabilities. Use `--read-only` where possible. Apply seccomp profiles.

### No Secrets in Layers

Never `COPY secrets.txt`. Use runtime secrets (K8s Secrets, Vault, Docker Secrets). Build args visible in history too.

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl --fail http://localhost:8080/health || exit 1
```

## Runtime & Orchestration
### Resource Limits

Always set CPU/memory limits. Monitor and tune.

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### Logging

Log to STDOUT/STDERR. Use structured JSON logging. Aggregate with Fluentd/Loki/Logstash.

### Persistent Storage

Use named volumes or PVS. Never store data in container writable layer.

```yaml
services:
  database:
    image: postgres:13
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

### Networking

Create custom networks for isolation. Use `internal: true` for backend networks. Implement network policies in K8s.

```yaml
networks:
  frontend:
  backend:
    internal: true
```

### Orchestration

K8s for complex/large-scale. Use rolling updates, resource requests/limits, self-healing, service discovery.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:v1.2.3
          resources:
            requests: { memory: "64Mi", cpu: "250m" }
            limits: { memory: "128Mi", cpu: "500m" }
```

## Review Checklist

- [ ] Multi-stage build used?
- [ ] Minimal, pinned base image?
- [ ] Layers optimized (combined RUN, cleanup)?
- [ ] `.dockerignore` present?
- [ ] Specific `COPY` instructions?
- [ ] Non-root `USER`?
- [ ] `HEALTHCHECK` defined?
- [ ] No secrets in layers?
- [ ] Config via env vars?
- [ ] SAST tools in CI (hadolint, trivy)?

## Troubleshooting

| Problem | Fix |
|---|---|
| Large image | `docker history`, multi-stage, smaller base, cleanup layers |
| Slow builds | Order instructions for cache, use `.dockerignore` |
| Container crashes | Check CMD/ENTRYPOINT, `docker logs`, verify deps in final image |
| Permission errors | Check `USER` permissions, volume mount ownership |
| Network issues | Verify EXPOSE vs published ports, check network config |
