# Containers Reference

Domain-specific traps, diagnostic commands, and checklist items for Docker container operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves containers:

- [ ] **Env file checked complete** — Compare the running container's environment (`docker inspect <name> --format '{{.Config.Env}}'`) with the `.env` file on disk. Recreation loses any variable absent from the file.
- [ ] **Volume mounts documented** — `docker inspect <name> --format '{{.Mounts}}'`. Capture every mount before any container recreation.
- [ ] **Container name matches references** — Caddyfile entries, proxy configs, and other containers may reference this container by name on the Docker network. Check the name won't change.
- [ ] **Docker network will not be modified** — if the operation touches Docker networks, see the Network Integrity section below.
- [ ] **Health checks check the full path** — `/up` endpoints must check backend reachability, not just return 200 from a proxy layer
- [ ] **Reboot resilience considered** — if migrating infrastructure, plan to check with `docker compose down && up -d` on at least one target

## Traps

### Network integrity

**Never recreate a production Docker network.** Recreating a network assigns new subnet IPs to every container on it. Every other container, proxy, and service that referenced the old IPs or relied on DNS resolution within that network breaks simultaneously. If you need additional connectivity, create a NEW network and attach containers to both.

**Never rename a production Docker network.** As with recreation, every reference to the old name breaks.

**Enable IPv6** on Docker networks that serve external traffic. Without IPv6, containers may silently fail to communicate with IPv6-only clients or upstream services. This can cause stale responses or failures that are difficult to diagnose. Check:

```bash
docker network inspect <network> | grep EnableIPv6
# Must be true

# Docker daemon config must include:
cat /etc/docker/daemon.json
# "ipv6": true, "ip6tables": true, "fixed-cidr-v6": "fd00:dead:beef::/48"
```

### Force-recreation destroys state

**Never force-recreate proxy/TLS containers** (Caddy, nginx, Traefik). On a proxy, `docker compose up -d --force-recreate` or `docker compose up -d --force-recreate <service>` can erase certificate caches, ACME account data, and session state. Use plain `docker compose up -d`. It recreates only containers whose configuration changed.

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

After a network rename, orchestrator change, or Compose restructure, running containers retain their old configuration. They appear healthy because they did not restart. A reboot, OOM kill, or crash exposes the missing resource. The container cannot start because its old network, volume, or name no longer exists.

The migration appears successful and may run for days or weeks. A later container restart then causes a permanent failure.

**After any infrastructure migration, test reboot resilience on at least one target:**

```bash
# Simulate a reboot — tears down and recreates all containers from config
docker compose down && docker compose up -d

# Verify everything came back
docker compose ps
```

**After migration, check EVERY container on EVERY affected host:**

```bash
# Check which network each container is on
docker inspect --format '{{.Name}}: {{range $net, $_ := .NetworkSettings.Networks}}{{$net}} {{end}}' $(docker ps -q)

# Anything referencing a dead network is a time bomb
```

### Storage that prune cannot see

`docker system prune` cannot inspect a volume. A self-hosted registry may store blobs in a named volume. Each CI push adds layers, but prune reclaims almost nothing. This does not mean the disk is too small. Expanding the disk delays the problem without stopping its growth.

```bash
# Prune reports near-zero reclaimable while the disk is full: look in the volumes
df -h /
du -sh /var/lib/docker/volumes/* 2>/dev/null | sort -h | tail -10
docker system df -v | head -30
```

Apply a retention policy at the application layer. Trim tags on a schedule. Keep the newest N, the last M days, and every digest currently running anywhere in the fleet. Then run the store's own garbage collection. Everything removed must be rebuildable from CI. Do not run the sweep while a build or deployment is pushing. See `scheduled-jobs.md`.

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

- **Never build images on production servers.** Build all images in CI. Pull them at runtime. Server builds are not reproducible, versioned, or suitable for rollback.
- **All containers must use `env_file` in Compose** (not inline environment variables in CI scripts or `-e` flags). The env file on the server is the single source of truth for secrets.
- **Never edit files directly on servers.** Make all configuration and code changes in the source-of-truth repo. Deploy them through CI. CI overwrites manual server edits at the next deployment. These edits cause invisible drift without git history or a rollback path. An uncommitted edit is not reproducible.
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
- **Volume-mounted config from a stale checkout:** A container may mount configuration from a server checkout. If deployment omits `git pull` before recreating the container, the container starts with old configuration.
- **Compose environment block vs env_file:** An `.env` variable has no effect if the Compose `environment:` block does not reference it. The variable remains on disk without entering the container.

**After EVERY deploy, check the new code is running:**

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

A common cause is a reverse proxy or auth layer that returns 200 from `/up` based only on its own health. Its backend can remain unreachable. The health check passes while every real request returns 502.

**Health checks must check the full request path**, not just that one layer is alive. If a service proxies to a backend, the health check must confirm the backend is reachable. If it can't do that, the health check is lying.
