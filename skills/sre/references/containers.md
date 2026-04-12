# Containers Reference

Domain-specific traps, diagnostic commands, and checklist items for Docker container operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves containers:

- [ ] **Env file verified complete** — compare running container's env (`docker inspect <name> --format '{{.Config.Env}}'`) against the `.env` file on disk. Any variable in the running container but NOT in the file will be lost on recreation.
- [ ] **Volume mounts documented** — `docker inspect <name> --format '{{.Mounts}}'`. Capture every mount before any container recreation.
- [ ] **Container name matches references** — Caddyfile entries, proxy configs, and other containers may reference this container by name on the Docker network. Verify the name won't change.
- [ ] **Docker network will not be modified** — if the operation touches Docker networks, see the Network Integrity section below.
- [ ] **Health checks verify the full path** — `/up` endpoints must check backend reachability, not just return 200 from a proxy layer
- [ ] **Reboot resilience considered** — if migrating infrastructure, plan to verify with `docker compose down && up -d` on at least one target

## Traps

### Network integrity

**Never recreate a production Docker network.** Recreating a network assigns new subnet IPs to every container on it. Every other container, proxy, and service that referenced the old IPs or relied on DNS resolution within that network breaks simultaneously. If you need additional connectivity, create a NEW network and attach containers to both.

**Never rename a production Docker network.** Same consequences as recreation — every reference to the old name breaks.

**Ensure IPv6 is enabled** on Docker networks serving external traffic. Without IPv6, containers may silently fail to communicate with IPv6-only clients or upstream services, causing stale responses or silent failures that are extremely hard to diagnose. Check:

```bash
docker network inspect <network> | grep EnableIPv6
# Must be true

# Docker daemon config must include:
cat /etc/docker/daemon.json
# "ipv6": true, "ip6tables": true, "fixed-cidr-v6": "fd00:dead:beef::/48"
```

### Force-recreation destroys state

**Never force-recreate proxy/TLS containers** (Caddy, nginx, Traefik). `docker compose up -d --force-recreate` or `docker compose up -d --force-recreate <service>` on a proxy can wipe certificate caches, ACME account data, and session state. Use plain `docker compose up -d` which only recreates containers whose configuration has actually changed.

### Stale containers after migration

When migrating between orchestration systems (standalone `docker run` to Compose, one Compose file to another), old containers keep running and serving traffic on their old ports. Users may hit weeks-old code with no alerts. After any migration:

```bash
# List ALL running containers
docker ps --format '{{.Names}}'

# List compose-managed containers
docker compose ps --format '{{.Name}}'

# Anything in docker ps but NOT in docker compose ps is a rogue container
```

Remove rogue containers immediately. They are invisible outages.

### Running containers mask stale configuration

After any migration (network rename, orchestrator change, compose restructure), containers that were already running keep their old configuration. They appear healthy because they never needed to restart. The first reboot, OOM kill, or crash reveals the truth: the container can't start because the resource it depends on (old network, old volume, old name) no longer exists.

This is a time bomb. The migration "succeeds" and may run for days or weeks until something triggers a container restart, at which point it fails permanently.

**After any infrastructure migration, test reboot resilience on at least one target:**

```bash
# Simulate a reboot — tears down and recreates all containers from config
docker compose down && docker compose up -d

# Verify everything came back
docker compose ps
```

**After migration, verify EVERY container on EVERY affected host:**

```bash
# Check which network each container is on
docker inspect --format '{{.Name}}: {{range $net, $_ := .NetworkSettings.Networks}}{{$net}} {{end}}' $(docker ps -q)

# Anything referencing a dead network is a time bomb
```

### Environment variable loss

The most common cause of post-recreation breakage: a container was originally started with `-e VAR=value` flags that exist only in shell history, not in any file. On recreation, those variables are gone and the service breaks in subtle ways.

Before recreating ANY container:

```bash
# Dump current environment
docker inspect <name> --format '{{range .Config.Env}}{{println .}}{{end}}'

# Compare against the env file
cat /path/to/.env

# Every variable in the inspect output must exist in the file
```

## Diagnostic Commands

```bash
# Container status and health
docker compose ps
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

# Container logs (last 50 lines)
docker logs <name> --tail 50

# Container environment
docker inspect <name> --format '{{range .Config.Env}}{{println .}}{{end}}'

# Container mounts
docker inspect <name> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println ""}}{{end}}'

# Container network
docker inspect <name> --format '{{range $net, $cfg := .NetworkSettings.Networks}}{{$net}}: {{$cfg.IPAddress}}{{println ""}}{{end}}'

# Network containers
docker network inspect <network> --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println ""}}{{end}}'

# What's listening on a port
ss -tlnp | grep <port>

# What Docker containers publish a port
docker ps --format '{{.Names}} {{.Ports}}' | grep <port>
```

## Verification Commands

```bash
# Service health via health endpoint
curl -sf https://<domain>/up

# Container is running and healthy
docker compose ps <service>

# Container responds on its internal port (from another container)
docker exec <other-container> curl -sf http://<target-container>:<port>/up

# No rogue containers after migration
diff <(docker ps --format '{{.Names}}' | sort) <(docker compose ps --format '{{.Name}}' | sort)
```

## Deployment Discipline

- **Never build images on production servers.** All images are built in CI and pulled at runtime. Building on the server means the image isn't reproducible, isn't versioned, and can't be rolled back.
- **All containers must use `env_file` in Compose** (not inline environment variables in CI scripts or `-e` flags). The env file on the server is the single source of truth for secrets.
- **Never edit files directly on servers.** All configuration and code changes must be made in the source-of-truth repo and deployed via CI. SSH-editing a config file on a server is non-reproducible — CI will overwrite it on the next deploy, creating invisible state drift with no git history and no rollback path. If it's not committed, it doesn't exist.
- **Every deployment must be rollback-capable.** Know the exact command to roll back before deploying:

```bash
# Rollback to previous image
docker compose pull <service>  # with previous tag in compose file
docker compose up -d <service>
```

### The deploy didn't deploy

A deploy pipeline can report success while the running container still serves old code. This is the most common post-deploy failure mode. Causes:

- **Docker build cache:** The image layer cache serves a stale build despite source changes. The image tag updates, but the binary inside is unchanged.
- **`docker compose start` vs `up -d`:** `start` reuses the existing container without pulling or recreating. `up -d` recreates containers whose configuration changed. If your deploy script uses `start`, it doesn't deploy.
- **Volume-mounted config from a stale checkout:** If a container mounts config from a git checkout on the server, and the deploy didn't `git pull` that checkout before recreating the container, the container starts with old config.
- **Compose environment block vs env_file:** Adding a variable to an `.env` file has no effect if the compose file uses an `environment:` block that doesn't reference it. The variable exists on disk but never enters the container.

**After EVERY deploy, verify the new code is running:**

```bash
# Check container creation time — was it actually recreated?
docker inspect <name> --format '{{.Created}}'

# Check a known change is present in the running container
docker exec <name> grep '<key_change>' <file>

# Check the image hash matches what CI built
docker inspect <name> --format '{{.Image}}'
```

If the deploy pipeline ran but the container wasn't recreated, **the deploy did not deploy.**

### False-positive health checks

A health check endpoint that returns 200 when the service is broken is worse than no health check — it actively masks outages.

Common cause: a reverse proxy or auth layer exposes `/up` and returns 200 based on its own health, but the backend it proxies to is unreachable. The health check passes while every real request returns 502.

**Health checks must verify the full request path**, not just that one layer is alive. If a service proxies to a backend, the health check must confirm the backend is reachable. If it can't do that, the health check is lying.
