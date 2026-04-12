# Firewalls Reference

Domain-specific traps, diagnostic commands, and checklist items for firewall and network security operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves firewalls or network rules:

- [ ] **Docker iptables interaction understood** — Docker creates its own FORWARD chain rules. Host-level firewalls interact with these in non-obvious ways.
- [ ] **Container-to-host communication paths audited** — some containers reach host-bound services (databases, APIs, monitoring agents). A deny-incoming rule may block this traffic.
- [ ] **Container-to-container communication verified** — confirm the Docker bridge network will not be affected
- [ ] **Testing on ONE server first** — apply the rule to one non-critical server, verify for 24 hours, then roll
- [ ] **Rule persistence verified** — iptables rules vanish on reboot unless persisted via `iptables-persistent` or equivalent
- [ ] **systemd dependencies reviewed** — if adding or modifying systemd drop-ins, verify no `Requires=` on non-essential services
- [ ] **ufw rule files validated** — all rules inside a `*filter`/`COMMIT` block, no orphans after final `COMMIT`

## Traps

### systemd dependency chains can kill Docker at boot

If Docker's systemd unit has `Requires=<dependency>.service` and that dependency fails to start, systemd cancels Docker's start job entirely. No containers start. No error is visible unless you read the journal.

**Never use `Requires=` for services that Docker doesn't strictly need to function.** Docker manages its own iptables chains and does not need the host firewall service. Use `Wants=` + `After=` to express ordering preferences without creating fatal coupling:

```ini
# WRONG — if the firewall fails, Docker dies
[Unit]
After=ufw.service
Requires=ufw.service

# RIGHT — Docker starts after the firewall if possible, but starts regardless
[Unit]
After=ufw.service
Wants=ufw.service
```

This is especially dangerous because the dependency failure may be latent — everything works until the next reboot, when the dependent service fails to start for the first time and takes Docker down with it. The outage can last hours if no boot-time monitoring exists.

**General rule:** Before adding any systemd dependency, ask: "If this dependency fails, should the dependent service also die?" If the answer is no, use `Wants=`, not `Requires=`.

### Orphaned rules in ufw after.rules

Custom iptables rules added to `/etc/ufw/after.rules` **must** be inside a `*filter` / `COMMIT` block. Rules placed after the final `COMMIT` statement are orphaned — `iptables-restore` doesn't know which table they belong to and fails with a syntax error.

The insidious part: orphaned rules are invisible during normal operation. When ufw is reloaded (`ufw reload`), the kernel already has the filter table loaded, so the restore is incremental and the orphaned rule is silently ignored. On cold boot, `iptables-restore` runs against an empty kernel state, hits the orphaned rule, and fails — taking the entire firewall service down.

```bash
# WRONG — rule is outside any *filter/COMMIT block
*filter
...
COMMIT

# This rule has no table context — fatal on cold boot
-A DOCKER-USER -p tcp --dport 25 -j RETURN
```

```bash
# RIGHT — rule is inside the *filter/COMMIT block
*filter
...
-A DOCKER-USER -p tcp --dport 25 -j RETURN
COMMIT
```

**After any edit to ufw rule files, verify cold-load integrity:**

```bash
# Simulate cold load (tests iptables-restore parsing)
iptables-restore --test < /etc/ufw/after.rules
```

### ufw does not control Docker-published ports

Docker publishes ports by inserting rules in the FORWARD and nat chains of iptables. `ufw` operates on the INPUT chain. This means:

- `ufw deny 8080` will **NOT** block traffic to a Docker container publishing port 8080. Docker's rules process the packet before ufw sees it.
- This gives false confidence — you test, see the port is "blocked" from the host's perspective, but Docker continues accepting external traffic via its own chains.

**To block external access to Docker-published ports, you must either:**
1. Stop publishing the port externally (bind to `127.0.0.1` in compose)
2. Add rules to the `DOCKER-USER` chain (the only chain Docker respects for user filtering)

### ufw reset destroys Docker networking

**Never run `ufw reset` on a Docker host.** It destroys Docker's iptables FORWARD rules, breaking ALL container networking instantly. Every container loses the ability to communicate with other containers, the host, and the outside world.

### Deny-incoming blocks container-to-host traffic

A blanket "deny incoming" firewall policy (e.g., `ufw default deny incoming`) blocks traffic from Docker's bridge subnet to the host. If any container needs to reach a host-bound service (database on localhost, monitoring agent, etc.), this breaks it silently.

Before adding deny-incoming rules, audit what containers talk to the host:

```bash
# Check if any container connects to host-bound services
docker ps --format '{{.Names}}' | while read c; do
  echo "=== $c ==="
  docker exec $c ss -tnp 2>/dev/null | grep -v "127.0.0.1" || echo "(no connections)"
done
```

### iptables rules are ephemeral by default

Custom iptables rules (including `DOCKER-USER` chain rules) vanish on reboot. This violates reproducibility. Persist them:

```bash
# Install persistence
apt install iptables-persistent

# After adding rules, save
netfilter-persistent save

# Verify rules survive reboot
# (rules should be in /etc/iptables/rules.v4 and rules.v6)
```

All firewall rules must also exist in a committed repo (e.g., site-config) so they can be reproduced on a fresh server.

## The Correct Way to Block Docker Ports

### Option A: Don't publish the port (preferred)

If the service is only accessed via a reverse proxy (Caddy, nginx) on the Docker network, it doesn't need an externally published port at all:

```yaml
# Before (accessible from anywhere):
ports:
  - "8080:8080"

# After (accessible only from localhost):
ports:
  - "127.0.0.1:8080:8080"

# Best (no port published — proxy reaches it by container name on Docker network):
# (remove the ports section entirely)
```

This is declarative, version-controlled, and survives reboots.

### Option B: DOCKER-USER chain (if port must be published)

```bash
# Block external access, allow Docker bridge and localhost
iptables -I DOCKER-USER -p tcp --dport 8080 -j DROP
iptables -I DOCKER-USER -p tcp --dport 8080 -s 127.0.0.0/8 -j RETURN
iptables -I DOCKER-USER -p tcp --dport 8080 -s 172.16.0.0/12 -j RETURN

# Persist
netfilter-persistent save
```

## Diagnostic Commands

```bash
# Current iptables rules (all chains)
iptables -L -n -v

# Docker-specific chains
iptables -L DOCKER-USER -n -v
iptables -L DOCKER -n -v
iptables -L FORWARD -n -v

# What ufw thinks is happening (may not reflect Docker reality)
ufw status verbose

# What's actually listening externally
ss -tlnp

# Test external port access from another machine
nc -zv <server-ip> <port>

# Check if a port is reachable through Docker's chains
curl -sf http://<server-ip>:<port> --connect-timeout 5 && echo "OPEN" || echo "BLOCKED"
```

## Verification Commands

After applying firewall rules:

```bash
# Verify port is blocked from outside
curl -sf http://<server-ip>:<port> --connect-timeout 5 && echo "STILL OPEN" || echo "BLOCKED"

# Verify container-to-container communication still works
docker exec <container-a> curl -sf http://<container-b>:<port>/up

# Verify container-to-host communication still works
docker exec <container> curl -sf http://host.docker.internal:<port> 2>/dev/null || \
docker exec <container> curl -sf http://172.17.0.1:<port>

# Verify all services are healthy
docker compose ps

# Verify health endpoints
curl -sf https://<domain>/up

# Verify rules persist across reboot
cat /etc/iptables/rules.v4 | grep <port>
```
