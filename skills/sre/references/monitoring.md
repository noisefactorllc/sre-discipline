# Monitoring Reference

Domain-specific traps, diagnostic commands, and checklist items for creating, changing, and retiring monitors.

Monitoring changes do not affect production traffic, but they still carry risk. Alerts on brief interruptions train people to ignore them. Monitors that never alert can hide outages for days. Deleting a monitor destroys history permanently.

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
- If the provider refuses an in-place field change, create a new monitor. Monitor type is commonly immutable. Keep the old monitor and its status-page binding until the operator explicitly accepts the history loss.
- Never batch-replace monitors.

### Webhook endpoints fail silently for days

Only external systems call some endpoints, so no user notices a failure. After a library upgrade, one billing webhook returned 500. The provider silently retried it for three days. Without a warning email, the provider would have permanently disabled it a week later.

Create a monitor in the same session as each endpoint for external callbacks, including payments, VCS hooks, and provider notifications. Probe it as the provider does. Check the expected healthy rejection. An unsigned POST to a signature-checking endpoint should return a specific 4xx. A 5xx means failure. Check from every region you serve.

### Cosmetic maintenance notices do not pause anything

A status-page maintenance report communicates the work. It does not suppress checks or alerts. Apply monitor-level suppression settings (maintenance from/to/days) to every affected monitor and heartbeat.

Check by dumping the resources back from the API and counting, rather than by trusting the writes to have landed.

### Maintenance windows hide real failures

Every minute of suppression can hide a real outage. A weekly 15-minute window across 50 resources is a deliberate tradeoff. Limit each window to the time the operation needs. Include only affected hosts. Remove one-off windows when the work ends.

### Monitoring what is on disk instead of what is served

A renewal job can update a certificate file while the TLS process continues serving the old certificate from memory. A file check then passes incorrectly. Similar failures include unloaded configuration, an image that is not running, and a DNS record that did not propagate. Check the served artifact.

### Alerting on a layer's own health

A proxy or auth layer can return its own 200 while every request to its backend returns 502. The monitor then reports success incorrectly. See the false-positive health check in `containers.md`.

### Deleting monitors during cleanup

Deleting monitors during service retirement also deletes their history. The service may still need them. Disable or archive monitors before deleting them. Confirm with the operator that history loss is acceptable.

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
