---
name: sre
description: SRE operational discipline for infrastructure operations. These include deployments, rollbacks, migrations, cache invalidation, certificate management, DNS changes, firewall rules, container orchestration, and server maintenance. Use this skill for ANY operation that touches production systems, Docker containers, reverse proxies, SSL/TLS certificates, or CDN caches. Also use it for operations that touch DNS records, firewall rules, load balancers, or deployment pipelines. Use it when debugging production incidents, planning infrastructure migrations, or reviewing infrastructure changes. This skill applies to tasks involving servers, containers, networking, or anything that could cause downtime. It applies even if the change seems simple.
user-invocable: false
---

# SRE Discipline

You are about to perform infrastructure work. Each rule addresses failures from a careless change: outages, lost data, or hours spent investigating additional failures.

Follow this procedure exactly. Do not skip steps. Do not reorder steps. The cost of following the checklist is minutes. The cost of skipping it is hours.

## Determine Operation Type

Before anything else, classify what you're doing:

- **Planned operation** (deployment, migration, config change, new service) → Follow Phases 1-5 below
- **Active incident** (something is broken now and users are affected) → Read `references/incidents.md`. Follow its procedure instead.

If you are unsure, treat the operation as an incident. The incident procedure is more conservative when something might be broken.

## Phase 1: Research

Do not change anything yet. Understand the situation first.

### Step 1.1: What exactly am I changing?

State the change in one sentence. If you cannot, divide the scope into smaller changes.

### Step 1.2: What depends on this?

Enumerate every service, network path, and system that will be affected by this change. Check:

- What services run on/connect to the thing I'm changing?
- What proxy routes, DNS records, or load balancer rules reference it?
- What monitoring or health checks will be affected?

If you do not know, investigate. Use `docker compose ps`, `docker network inspect`, DNS lookups, proxy configuration files, and health endpoint checks. Do not guess.

### Step 1.3: What's cached or stateful?

Identify everything that will persist through or be invalidated by this change:

- **Container state**: Environment variables, mounted volumes, network IPs, TLS cert caches
- **Cache layers**: Browser cache, CDN edge caches, reverse proxy cache, DNS resolver cache
- **TTLs**: How long will stale data persist if not actively invalidated?
- **Secrets/credentials**: Are any secrets only in running containers (not in persistent env files)?
- **Shared-state structure**: Does the change require new database columns, configuration fields, environment variables, or secrets? These requirements must exist in production BEFORE the new code can run. See `references/migrations.md`.
- **Configuration source of truth**: Do the affected configurations exist in a tracked repo, or only on the server? Untracked configurations prevent audits and validation at PR time. See `references/configurations.md`.

Load the relevant reference doc(s) based on what domains this change touches:

| If the change involves... | Read |
|--------------------------|------|
| Docker containers, compose, images | `references/containers.md` |
| SSL/TLS certificates, ACME, Let's Encrypt | `references/certificates.md` |
| DNS records, domain routing | `references/dns.md` |
| CDN, proxy cache, static assets, TTLs | `references/caching.md` |
| Firewalls, iptables, network security | `references/firewalls.md` |
| Database schemas, config fields, env vars (new requirements on shared state) | `references/migrations.md` |
| Application config files (JSON/YAML/env), secret templating, drift detection | `references/configurations.md` |
| CI/CD pipelines, what a commit triggers, what a push carries, deploys that half-apply | `references/deployments.md` |
| Cron jobs, systemd timers, renewal loops, retention sweeps, unattended maintenance | `references/scheduled-jobs.md` |
| Uptime checks, alert thresholds, maintenance windows, retiring a monitor | `references/monitoring.md` |
| Proving the change worked (always: read this before you write any verification command) | `references/verification.md` |

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
- [ ] **All configs live in version control** — Every configuration that affects runtime behavior must be a tracked file. Routing edits through the repo is insufficient. Add configurations that exist only on a server to the repo before this work starts.
- [ ] **Pre-merge validation gates the deploy** — CI lint must check every configuration. It must block merges if any required field is missing. All deployment jobs must depend on this check.
- [ ] **New shared-state requirements landed first** — The new version may require new database columns, configuration fields, or environment variables. Apply these changes to production BEFORE the new code can run. Check them before that code runs.
- [ ] **Environment variables checked in persistent files** — not just in running containers
- [ ] **Monitoring baseline captured** — know what "green" looks like before you change anything
- [ ] **Verification plan can actually fail** — Identify the command that proves the change worked. Its exit status must come from the tool itself. A pipe or login shell must not replace that status. Use evidence from the deployed system. See `references/verification.md`.
- [ ] **Trigger set known** — If CI ships the change, identify every pipeline the commit triggers. Every triggered pipeline must be intentional. See `references/deployments.md`.

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

### Step 3.2: Check it worked

Run the specific verification commands for this change. Do not rely on "it looks fine." Check with explicit commands:

- **Service health**: `curl -sf https://<domain>/up` or equivalent health endpoint
- **Container status**: `docker compose ps <service>` — confirm it's running and healthy
- **Port/network**: `curl` or `nc` from outside the server to check connectivity
- **TLS**: Check the certificate served matches expectations (check domain, expiry, issuer)
- **DNS**: Query from an external resolver, not the local cache
- **Cache**: Check `X-Cache-Status` headers, check from multiple edges if applicable

Measurement tools can report misleading results. A pipeline's exit code belongs to its last command. A remote login shell can replace an SSH heredoc's status. A single-packet ping can falsely report an outage. An empty filtered query does not prove absence. Before relying on verification output, read `references/verification.md`.

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

If propagation applies, wait the appropriate time before checking.

### Step 3.4: Proceed to the next target

Only after verification passes on the current target:

1. For high-risk changes: monitor for 24 hours before proceeding to the next target.
2. For medium-risk changes: wait through at least one full request cycle before proceeding.
3. For low-risk changes: check the current target before proceeding.

Production/primary targets go **last**, always.

### Step 3.5: If something breaks — STOP

Do NOT make additional changes. Do NOT try to fix forward through breakage. Instead:

1. **Stop immediately.** The system is in an unknown state.
2. **Assess damage.** What's actually broken? What's still working? What's the user impact?
3. **Understand the cause.** If you don't understand why it broke, you can't fix it.
4. **Roll back** to the last known good state if possible.
5. **If you cannot roll back**, make ONE targeted fix. Check the result. Then reassess whether to continue or abort.

Switch to the incident procedure (`references/incidents.md`) if the breakage is user-facing.

## Phase 4: Short-term Follow-through

The change is applied and checked. Now remove stale resources.

### Step 4.1: Check from outside

Check the change from outside the server and your local machine's cached state. Use external monitoring, different DNS resolvers, and different network paths.

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

If the change added a service, an externally called endpoint, or a failure mode, add monitoring before completing the work. If you suppressed alerts for maintenance, remove that suppression. See `references/monitoring.md`.

### Step 4.4: Check for unintended side effects

Did the change affect anything you didn't expect? Check:

- Services that share the same Docker network
- Services on the same server that weren't the target
- CDN edges or DNS that route to the affected server
- Multi-service architectures where one service depends on another

## Phase 5: Long-term Follow-through

Before declaring the work complete:

- [ ] **All changes committed to version control** — no uncommitted files on any server
- [ ] **Change survives a reboot** — would a `docker compose up -d` from scratch reproduce the current state? If you migrated infrastructure, test it: `docker compose down && docker compose up -d` on at least one target.
- [ ] **Change survives container recreation** — all env vars, volumes, and config in persistent files
- [ ] **No stale resources left running** — running containers mask stale configuration until the next restart. After any migration, check every container on every affected host is on the correct network/config.
- [ ] **No manual state left on servers** — nothing done via SSH that isn't in a committed repo
- [ ] **No ephemeral solutions** — no temporary keys, passwords, workarounds, or "we'll fix this later" hacks
- [ ] **Monitoring covers the new state** — health checks and alerts reflect the current architecture
- [ ] **Rollback tested** — you have actually checked the rollback procedure works, not just documented it
- [ ] **Stale resources cleaned up** — no orphaned containers, DNS records, proxy routes, or firewall rules
- [ ] **Documentation updated** — runbooks, README, or config docs reflect the new state if applicable

If any box is unchecked, the work isn't done.
