# Monitoring Reference

Domain-specific traps, diagnostic commands, and checklist items for creating, changing, and retiring monitors.

Monitoring changes look low-risk because they touch no production traffic. They are not. A monitor that pages on a blip trains people to ignore alerts, a monitor that never fires hides an outage for days, and a deleted monitor destroys history that cannot be rebuilt.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation creates, modifies, or removes monitoring:

- [ ] **Baseline captured**: you know what green looks like right now, before the change
- [ ] **Every check has hysteresis**: a non-zero confirmation period, so one failed sample cannot page a human
- [ ] **New externally-called endpoints get a monitor in the same session**: webhooks and callbacks especially
- [ ] **Changes are applied in place**: PATCH the existing monitor, never delete-and-recreate
- [ ] **Immutable fields identified**: know which fields your provider refuses to change before planning the edit
- [ ] **Planned disruption suppressed at the monitor level**: and the suppression window is as tight as the work requires
- [ ] **The check exercises the real path**: from outside, through every layer, against what is actually served

## Traps

### Zero hysteresis pages on blips

A monitor with no confirmation period turns a 12-second container restart during a routine deploy into an SMS and a phone call. The service auto-recovered before anyone could read the alert, and the alert could not be prevented after the fact.

Set a non-zero confirmation period on every check. Treat `confirmation_period: 0` as a defect whenever you see one during an audit, not as a stylistic choice.

### History belongs to the resource, not the check config

Uptime history is attached to the monitor ID. Deleting a monitor to replace it with a better-configured one destroys every day of accumulated history for that service, permanently, with no restore path. Batch-replacing a set of monitors can wipe months across a whole status page in one command.

- Migrating a check URL, threshold, or region set: PATCH the existing monitor.
- If the provider refuses to change a field in place (monitor type is commonly immutable), create the new monitor but keep the old one and its status-page binding until the operator has explicitly accepted the history loss.
- Never batch-replace monitors.

### Webhook endpoints fail silently for days

An endpoint that only external systems call has no user to notice when it breaks. A billing webhook that started returning 500 after a library upgrade was retried silently by the provider for three days, and would have been disabled permanently a week later, had a warning email not surfaced it by luck.

Every endpoint that receives external callbacks (payments, VCS hooks, provider notifications) gets a monitor at creation time, in the same session. Probe it the way the provider does and assert the healthy rejection: an unsigned POST to a signature-verifying endpoint should return a specific 4xx, and a 5xx means broken. Check from every region you serve.

### Cosmetic maintenance notices do not pause anything

Publishing a maintenance report on a status page is a communication artifact. It does not suppress checks, and monitors will still fire during the window. Suppression is a monitor-level setting (maintenance from/to/days) and has to be applied to every monitor AND every heartbeat that the work touches, not just the ones you remember.

Verify by dumping the resources back from the API and counting, rather than by trusting the writes to have landed.

### Maintenance windows hide real failures

Every minute of suppression is a minute in which a genuine outage does not alert. A weekly 15-minute window across 50 resources is a real, deliberate trade. Keep windows as narrow as the operation needs, scope them to the affected hosts, and remove one-off windows when the work is done.

### Monitoring what is on disk instead of what is served

A renewal job writes a fresh certificate to disk and the file-based check goes green, while the process that serves TLS keeps the old certificate in memory until reloaded. The same shape appears everywhere: config present on disk but not loaded, image pulled but not running, DNS record created but not propagated. Check the served artifact.

### Alerting on a layer's own health

A check that hits a proxy or auth layer and receives that layer's own 200 will stay green while every real request behind it returns 502. This is the false-positive health check from `containers.md`, viewed from the monitoring side: the monitor is only as honest as the endpoint it calls.

### Deleting monitors during cleanup

Retiring a service tempts you to delete its monitors as part of the sweep. That deletes the history too, and it is common to discover afterwards that the service was not fully retired. Disable or archive before deleting, and confirm with the operator that history loss is acceptable.

## Diagnostic Commands

Provider APIs differ. The shape of the check is what matters.

```bash
# Dump every monitor and inspect the fields that matter, rather than trusting the UI
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors" \
  | jq '.data[] | {id, url: .attributes.url, type: .attributes.monitor_type,
                   confirm: .attributes.confirmation_period}'

# Find zero-hysteresis monitors (defects)
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors" \
  | jq '[.data[] | select(.attributes.confirmation_period == 0)
         | {id, url: .attributes.url}]'

# Confirm a maintenance window actually landed on every resource
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors" \
  | jq '[.data[] | {id, from: .attributes.maintenance_from,
                    to: .attributes.maintenance_to, days: .attributes.maintenance_days}]'

# Count monitors vs count with the window set: the two numbers must match
```

```bash
# What does the check actually see? Run its exact request by hand
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://<domain>/<webhook-path>"
# Expect the signature-rejection code, not 5xx and not 200
```

## Verification Commands

```bash
# The monitor exists, is enabled, and has hysteresis
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors/<id>" \
  | jq '{url: .data.attributes.url, paused: .data.attributes.paused,
         confirm: .data.attributes.confirmation_period}'

# The monitor's history survived the change (spot-check the oldest available datapoint)
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors/<id>/sla?from=<old-date>&to=<today>"

# Post-change: all monitors green, and none silently paused
curl -sf -H "Authorization: Bearer $TOKEN" "$API/monitors" \
  | jq '[.data[] | select(.attributes.status != "up" or .attributes.paused == true)
         | {id, url: .attributes.url, status: .attributes.status, paused: .attributes.paused}]'
# Expect: []
```

A monitor you created but never saw fail is unproven. Where it is safe to do so, break the thing on purpose once and confirm the alert arrives.
