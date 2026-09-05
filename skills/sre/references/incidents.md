# Incident Response Reference

This document overrides the normal 5-phase procedure when something is actively broken and users are affected.

The normal procedure covers planned changes. During an incident, restore service first. Understand the cause second. Make the permanent fix third. Continue to make one change before each verification. Additional changes to a broken system cause most catastrophic incident escalations.

## Incident Procedure

### Step 1: STOP

Stop the current work. If a planned change is in progress, stop the rollout. Do not make the next change. Do not try to fix forward through the breakage.

### Step 2: Assess

Identify what is actually broken now. Do not infer the failure from your recent change.

```bash
# What services are down?
curl -sf https://<domain>/up && echo "UP" || echo "DOWN"

# What containers are running/stopped?
docker compose ps

# What do the logs say?
docker logs <service> --tail 50

# What does external monitoring say?
# (Check Betterstack, Datadog, Pingdom, or whatever monitoring is in place)
```

Answer these questions before doing anything else:
1. **What's broken?** (Which services, which endpoints, which users affected)
2. **What still works?** (Don't assume everything is down — check)
3. **What's the user impact right now?** (Complete outage? Partial? Degraded performance?)
4. **When did it start?** (Correlate with recent changes, deploys, or external events)

### Step 3: Understand the cause

Your first guess is often wrong — especially if you just made a change and assume it's related. Check your theory before acting on it.

Common wrong-first-guesses:
- "My recent change broke it" — maybe, but check the actual error. It could be unrelated.
- "The service crashed" — check if it's actually running. Maybe the proxy route is wrong.
- "DNS is broken" — check if the service responds when you bypass DNS.
- "The cert expired" — check what cert is actually being served.

```bash
# Check the actual error (not what you expect)
curl -vI https://<domain> 2>&1 | tail -20

# Check from outside (not from the server)
# Use external monitoring or curl from a different network
```

### Step 4: Restore service

Priority is getting users back online, not fixing the root cause elegantly.

**If you can roll back:**
```bash
# Rollback to previous known-good state
docker compose pull <service>  # with previous tag
docker compose up -d <service>
```

**If you can't roll back**, make ONE targeted fix:
1. State what you're fixing and why
2. Make the fix
3. Check it worked
4. If it didn't work, revert it — do NOT add another fix on top

### Step 5: Check

Check from the user's perspective, not from the server:

```bash
# Service responds correctly
curl -sf https://<domain>/up

# From outside your network if possible
# Check external monitoring is green
```

### Step 6: Stabilize and decide

Once service is restored:
1. Is the fix permanent, or is this a temporary restoration?
2. If temporary, what's the permanent fix? (But don't implement it now — schedule it as planned work)
3. Are there other systems affected that you haven't checked?
4. Update monitoring/status page if applicable

## Anti-Patterns to Block

These are the behaviors that escalate incidents. Recognize them and stop.

### Fix cascading

You make a fix. It doesn't work. You make another fix on top. That doesn't work either. Now you have the original problem plus two half-applied fixes, and the system state is incomprehensible.

**When a fix doesn't work, revert it before trying something else.** Return to a known state.

### Commit thrashing

Under pressure, it's tempting to push rapid-fire commits to CI hoping one will work. Each broken commit wastes CI time, pollutes git history, and may trigger deploy pipelines that make things worse.

**Get the fix right locally. Check it locally. Push ONE clean commit.**

### Only fixing one target

You fix the service on one server, declare success, and move on to cleanup. Meanwhile the same failure exists on every other server with the same architecture. Users on those servers are still down.

**If one server is affected, immediately check every server with the same architecture.** Restore service across ALL affected infrastructure before file cleanup, post-mortems, or other work. All users must regain service first.

### Scope creep

"While I'm fixing this, let me also clean up that other thing I noticed."

**No. Fix the incident. Check the fix. File tickets for everything else.** Mixing incident response with maintenance work is how you turn a 30-minute incident into a 3-hour incident.

### Wrong initial theory

You just pushed a DNS change, and the site is now down. The change might be responsible. Check the actual error before rolling back. A container crash, expired certificate, CDN issue, or upstream failure could have coincided with your change.

**Check the actual error, not what you expect the error to be.**

### Making changes to a broken system

When a system is in an unknown state (something broke and you're not sure why), every additional change increases the unknown. Adding a firewall rule while debugging a proxy issue. Restarting a database while investigating a networking problem.

**Fix one thing. Check. Then decide whether to continue.**

## Post-Incident

After the incident is resolved and the immediate pressure is off:

1. **Document what happened** — what broke, what caused it, what fixed it
2. **Identify the root cause** — not just the proximate cause ("the config was wrong") but the systemic cause ("we don't validate configs before deploying")
3. **Do not publish an unproven mechanism as the cause.** A plausible timeline does not prove a root cause. Check the mechanism against unit configuration, log timestamps, and the actual invocation source before reporting it to the team. A wrong root cause ends the investigation while the defect remains. The team must later retract it.
4. **Identify prevention measures** — what would prevent this class of incident in the future?
5. **Review infrastructure changes made during the incident.** Urgent changes often introduce new failure modes. A systemd dependency for boot ordering may create a fatal cascade. A firewall rule may interrupt traffic. Treat every incident change as provisional until you review it after the immediate pressure ends.
6. **Return to the normal 5-phase procedure** for any follow-up work
