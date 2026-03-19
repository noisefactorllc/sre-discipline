# Firewalls Reference

Domain-specific traps, diagnostic commands, and checklist items for firewall and network security operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves firewalls or network rules:

- [ ] **Docker iptables interaction understood** — Docker creates its own FORWARD chain rules. Host-level firewalls interact with these in non-obvious ways.
- [ ] **Container-to-host communication paths audited** — some containers reach host-bound services (databases, APIs, monitoring agents). A deny-incoming rule may block this traffic.
- [ ] **Container-to-container communication verified** — confirm the Docker bridge network will not be affected
- [ ] **Testing on ONE server first** — apply the rule to one non-critical server, verify for 24 hours, then roll
- [ ] **Rule persistence verified** — iptables rules vanish on reboot unless persisted via `iptables-persistent` or equivalent

## Traps

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
