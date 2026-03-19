---
name: sre
description: Battle-tested SRE operational discipline for any infrastructure operation — deployments, rollbacks, migrations, cache invalidation, certificate management, DNS changes, firewall rules, container orchestration, and server maintenance. Use this skill whenever performing ANY operation that touches production systems, Docker containers, reverse proxies, SSL/TLS certificates, CDN caches, DNS records, firewall rules, load balancers, or deployment pipelines. Also use when debugging production incidents, planning infrastructure migrations, or reviewing infrastructure changes. If the task involves servers, containers, networking, or anything that could cause downtime — this skill applies. Even if you think the change is simple.
user-invocable: false
---

# SRE Discipline

You are about to perform infrastructure work. Every rule in this procedure exists because someone learned it the hard way — systems went down, data was lost, or hours were wasted chasing cascading failures that started with one careless change.

Follow this procedure exactly. Do not skip steps. Do not reorder steps. The cost of following the checklist is minutes. The cost of skipping it is hours.

## Determine Operation Type

Before anything else, classify what you're doing:

- **Planned operation** (deployment, migration, config change, new service) → Follow Phases 1-5 below
- **Active incident** (something is broken right now, users are affected) → Read `references/incidents.md` and follow its procedure instead

If you're unsure, treat it as an incident — the incident procedure is more conservative and that's the right default when things might be broken.

## Phase 1: Research

Do not touch anything yet. Understand the situation first.

### Step 1.1: What exactly am I changing?

State the change in one sentence. If you can't, the scope is too broad — decompose it.

### Step 1.2: What depends on this?

Enumerate every service, network path, and system that will be affected by this change. Check:

- What services run on/connect to the thing I'm changing?
- What proxy routes, DNS records, or load balancer rules reference it?
- What monitoring or health checks will be affected?

If you don't know, find out. `docker compose ps`, `docker network inspect`, DNS lookups, Caddyfile/nginx config reads, health endpoint checks. Do not guess.

### Step 1.3: What's cached or stateful?

Identify everything that will persist through or be invalidated by this change:

- **Container state**: Environment variables, mounted volumes, network IPs, TLS cert caches
- **Cache layers**: Browser cache, CDN edge caches, reverse proxy cache, DNS resolver cache
- **TTLs**: How long will stale data persist if not actively invalidated?
- **Secrets/credentials**: Are any secrets only in running containers (not in persistent env files)?

Load the relevant reference doc(s) based on what domains this change touches:

| If the change involves... | Read |
|--------------------------|------|
| Docker containers, compose, images | `references/containers.md` |
| SSL/TLS certificates, ACME, Let's Encrypt | `references/certificates.md` |
| DNS records, domain routing | `references/dns.md` |
| CDN, proxy cache, static assets, TTLs | `references/caching.md` |
| Firewalls, iptables, network security | `references/firewalls.md` |

Read each relevant reference doc now. They contain domain-specific checklist items and traps that you must incorporate into Phase 2.

### Step 1.4: What's the blast radius?

If this change goes wrong, what breaks? How many users are affected? How long until someone notices?

### Step 1.5: What's the rollback?

State the exact command or sequence that undoes this change. If you can't state it, you're not ready to proceed. Common rollback patterns:

- **Container deploy**: `docker compose pull <service> && docker compose up -d <service>` with the previous image tag
- **Config change**: `git revert <commit> && deploy`
- **DNS change**: Restore the previous record (but respect TTL — rollback won't be instant)
- **Firewall rule**: Remove the rule (but know that Docker FORWARD chains may need attention)

## Phase 2: Pre-flight Checklist

This is the gate. Every box must be checked before you execute.

### Universal checklist (always applies):

- [ ] **Change stated in one sentence** — scope is clear and bounded
- [ ] **Dependencies enumerated** — every affected service, path, and system identified
- [ ] **Cached/stateful resources identified** — know what persists, what invalidates, what TTLs apply
- [ ] **Rollback procedure documented** — exact commands, not just "revert the change"
- [ ] **Starting with ONE non-critical target** — never apply to all targets simultaneously
- [ ] **All config changes are in version control** — no manual edits on servers that aren't in a committed repo
- [ ] **Environment variables verified in persistent files** — not just in running containers
- [ ] **Monitoring baseline captured** — know what "green" looks like before you change anything

### Domain-specific checklist items:

Append the relevant items from the reference doc(s) you loaded in Phase 1. Each reference doc has a "Pre-flight items" section. Add those items to this checklist.

### Present the checklist to the user:

Before executing, show the user your completed checklist. State:
1. What you're changing (one sentence)
2. What you identified as dependencies
3. Your rollback procedure
4. Which target you're starting with
5. Any domain-specific risks from the reference docs

Wait for acknowledgment before proceeding to Phase 3.

## Phase 3: Execution

One change. One verification. Then the next.

### Step 3.1: Make exactly one change

Apply the change to your first (non-critical) target only.

### Step 3.2: Verify it worked

Run the specific verification commands for this change. Do not rely on "it looks fine." Verify with explicit commands:

- **Service health**: `curl -sf https://<domain>/up` or equivalent health endpoint
- **Container status**: `docker compose ps <service>` — confirm it's running and healthy
- **Port/network**: `curl` or `nc` from outside the server to verify connectivity
- **TLS**: Verify the certificate served matches expectations (check domain, expiry, issuer)
- **DNS**: Query from an external resolver, not the local cache
- **Cache**: Check `X-Cache-Status` headers, verify from multiple edges if applicable

### Step 3.3: Wait for propagation

Some changes aren't instant:

| Change type | Propagation time |
|------------|-----------------|
| Container restart | Seconds |
| DNS (low TTL, 300s) | ~5 minutes |
| DNS (high TTL, 86400s) | Up to 24 hours |
| CDN cache (1h TTL) | Up to 1 hour |
| TLS cert issuance (ACME) | 1-5 minutes |
| Browser cache | Varies, possibly days |

If propagation applies, wait the appropriate time before verifying.

### Step 3.4: Roll to next target

Only after verification passes on the current target:

1. For high-risk changes: wait 24 hours, monitor, then proceed to the next target
2. For medium-risk changes: wait through at least one full request cycle, then proceed
3. For low-risk changes: verify, then proceed

Production/primary targets go **last**, always.

### Step 3.5: If something breaks — STOP

Do NOT make additional changes. Do NOT try to fix forward through breakage. Instead:

1. **Stop immediately.** The system is in an unknown state.
2. **Assess damage.** What's actually broken? What's still working? What's the user impact?
3. **Understand the cause.** If you don't understand why it broke, you can't fix it.
4. **Roll back** to the last known good state if possible.
5. **If you can't roll back**, make ONE targeted fix. Verify. Then reassess whether to continue or abort.

Switch to the incident procedure (`references/incidents.md`) if the breakage is user-facing.

## Phase 4: Short-term Follow-through

The change is applied and verified. Now clean up.

### Step 4.1: Verify from outside

Check the change from an external perspective — not from the server, not from your local machine's cached state. Use external monitoring, different DNS resolvers, different network paths.

### Step 4.2: Garbage collect

Search for stale resources created or exposed by this change:

- **Stale containers**: `docker ps` — anything running that shouldn't be? Old containers from before the migration?
- **Orphaned DNS records**: Any records pointing to old IPs or decommissioned services?
- **Old proxy routes**: Any Caddyfile/nginx entries for removed services?
- **Old firewall rules**: Any rules for ports no longer in use?
- **Old cron jobs**: Anything scheduled for services that moved?

If you find stale resources, remove them now. Stale containers are invisible outages — they serve old code to real users with no alerts.

### Step 4.3: Confirm monitoring is green

Check that all health checks and monitors are passing. If any are in a degraded or alerting state, investigate before moving on — you may have introduced a subtle regression.

### Step 4.4: Check for unintended side effects

Did the change affect anything you didn't expect? Check:

- Services that share the same Docker network
- Services on the same server that weren't the target
- CDN edges or DNS that route to the affected server
- Multi-service architectures where one service depends on another

## Phase 5: Long-term Follow-through

Before declaring the work complete:

- [ ] **All changes committed to version control** — no uncommitted files on any server
- [ ] **Change survives a reboot** — would a `docker compose up -d` from scratch reproduce the current state?
- [ ] **Change survives container recreation** — all env vars, volumes, and config in persistent files
- [ ] **No manual state left on servers** — nothing done via SSH that isn't in a committed repo
- [ ] **No ephemeral solutions** — no temporary keys, passwords, workarounds, or "we'll fix this later" hacks
- [ ] **Monitoring covers the new state** — health checks and alerts reflect the current architecture
- [ ] **Rollback tested** — you have actually verified the rollback procedure works, not just documented it
- [ ] **Stale resources cleaned up** — no orphaned containers, DNS records, proxy routes, or firewall rules
- [ ] **Documentation updated** — runbooks, README, or config docs reflect the new state if applicable

If any box is unchecked, the work isn't done.
