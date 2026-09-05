# Scheduled Jobs Reference

This reference covers traps, diagnostic commands, and checklists for jobs that operate on production without a human present. These include cron jobs, systemd timers, renewal loops, and retention sweeps.

Unattended automation requires stricter controls than interactive changes. Nobody watches when it fails, and it can affect everything within its permissions. A job that reboots servers, deletes data, or renews certificates must work correctly when nobody observes the run.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation creates or modifies a scheduled job:

- [ ] **Schedule carries an explicit timezone**: pin it to UTC rather than inheriting the host's
- [ ] **Host timezone checked and self-consistent**: check the effective timezone, not just one config file
- [ ] **The job re-checks its own preconditions at run time**: fail closed if the conditions it assumes are not true right now
- [ ] **Concurrency understood**: the job either takes a lock, gates on in-flight work, or is genuinely safe to run alongside anything
- [ ] **One owner per state directory**: exactly one process manages a given lockfile, cert store, or data dir
- [ ] **Deploying the job does not run the job**: stop the timer before reloading unit files
- [ ] **Deletions are bounded and reconstructible**: a retention sweep can name what it will remove and how to get it back
- [ ] **Failure is visible**: the job reports its outcome somewhere a human or a monitor sees

## Traps

### Unpinned schedules fire in host-local time

A systemd `OnCalendar=05:00` schedule uses the host's timezone. On a US-timezone host, a job intended for 05:00 UTC instead ran at 11:00 UTC. It rebooted a server hours outside the approved window.

Worse, a host's timezone configuration can be internally inconsistent: `/etc/localtime` pointing at one zone while `/etc/timezone` says another. The effective zone (the one systemd uses) is whatever `timedatectl` reports, which may be neither of the ones you read.

- Pin every schedule: `OnCalendar=Mon *-*-* 05:00:00 UTC`, or `CRON_TZ=UTC` for cron.
- Pin all hosts, not just the one you caught. Hosts that happen to be UTC today are only correct by accident.
- Fix the underlying inconsistency (`timedatectl set-timezone UTC`) so the pin becomes redundant rather than load-bearing.

### Acting on a condition without checking the clock

A job that only checks a reboot-required flag will reboot whenever it runs. Calculating a maintenance window from the current time does not check whether the approved window is active. A manual start, deployment trigger, or catch-up run can therefore cause an unscheduled production reboot.

Every destructive scheduled job should fail closed on its own preconditions:

```bash
# In the job itself, not only in the timer
now=$(date -u +%H%M)
start=$(date -u -d "$REBOOT_TIME" +%H%M)
end=$(date -u -d "$REBOOT_TIME + $WINDOW_MINUTES minutes" +%H%M)
if [ "$now" -lt "$start" ] || [ "$now" -gt "$end" ]; then
  echo "refusing: $now UTC is outside window $start-$end"; exit 0
fi
```

The timer says when to try. The job decides whether it is allowed.

### Deploying the schedule can trigger it

Reloading units, restarting a timer, or reinstalling a cron entry can immediately run the job. This depends on the unit configuration and deployment procedure. For destructive jobs, even a documentation-level change can therefore trigger a production action.

Stop the timer. Reload the unit files. Install the schedule. Then start the timer. `Persistent=no` prevents replay of missed events, so a catch-up run is often the wrong explanation. See the note on unproven mechanisms below.

### Two processes, one state directory

Two renewal processes sharing `/etc/letsencrypt`, such as a container loop and host cron job, compete for the same lockfile. One silently fails with "Another instance is already running". Its certificates stop renewing while the other process still has healthy logs.

Exactly one process owns each state directory. When consolidating, check that the remaining process retains the required capabilities. A base image without a DNS plugin cannot renew DNS-01 certificates. It discovers this only at renewal time, possibly weeks later.

### Renew is not reload

A renewal job that writes a new certificate to disk has not finished the job. Servers cache certificates in memory at load time, so a certificate renewed on disk is not served until the process reloads. A host that has been up for days will serve the expired one until something restarts it.

Pair every renewal with a reload, and check against the served certificate rather than the file:

```bash
# Self-reloading nginx container: reload every 6h alongside the renewal loop
command: /bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g "daemon off;"'

# Or reload only when the serial changes, from the renewal hook
```

### Non-interactive renewals stall on purpose

`certbot renew` in non-interactive mode adds a random delay of up to several minutes to spread load on the CA. A startup renewal that appears hung usually is not. Poll the certificate on disk. It changes after the delay and challenge.

### Garbage collection racing with writers

A retention sweep that deletes untagged artifacts while a build is pushing can delete blobs the in-flight push is still assembling. Gate the sweep on no build or deploy being in flight, or use whatever quiescence mechanism the store provides.

### Storage the usual tools cannot see

`docker system prune` cannot inspect a volume. A self-hosted registry may store blobs in a named volume. Each CI push consumes more disk space while prune reclaims almost nothing. This does not prove the disk is too small.

Apply a retention policy at the application layer. For an image registry, trim tags on a schedule. Keep `latest`, the newest N, the last M days, and every digest currently running anywhere in the fleet. Then run the registry's own garbage collection. Expanding the disk delays the problem without stopping its growth.

### Suppressing alerts for the job's own disruption

If the job causes a brief outage by design (a reboot, a restart), suppress the alerting at the monitor level for exactly that window. A status-page notice does not suppress anything. See `monitoring.md`.

### Publishing an unproven mechanism as the cause

When an unattended job does something unexpected, the trigger is often not what it looks like. Confirm the mechanism (unit configuration, log timestamps, invocation source) before naming it as the cause in anything the operator or the team reads. A wrong root cause that gets circulated is worse than an open question, because it ends the investigation and the real defect stays.

In the reboot case above, a catch-up run did not replay a missed timer event. The actual defect was the missing window check.

## Diagnostic Commands

```bash
# The effective timezone, which may match neither config file
timedatectl
readlink -f /etc/localtime; cat /etc/timezone

# When will this timer actually fire, in both zones
systemctl list-timers --all | grep <unit>
systemd-analyze calendar 'Mon *-*-* 05:00:00 UTC'

# What the unit is configured to do
systemctl cat <unit>.timer <unit>.service

# Did it run, and with what result
journalctl -u <unit> --since '7 days ago' --no-pager

# Cron equivalents
crontab -l -u <user>; cat /etc/cron.d/*
grep -r CRON /var/log/syslog | tail -20

# Is more than one process managing the same state dir
ps aux | grep -c certbot
ls -la /etc/letsencrypt/.certbot.lock 2>/dev/null

# What is actually consuming the disk (volumes are invisible to prune)
df -h /
du -sh /var/lib/docker/volumes/* 2>/dev/null | sort -h | tail -10
docker system df -v | head -30
```

## Verification Commands

```bash
# The schedule resolves to the intended UTC time
systemctl list-timers <unit> --no-pager
# NEXT column must be the intended UTC instant, not host-local

# The guard actually refuses out of window (safe to test: it should no-op)
sudo systemctl start <unit>.service
journalctl -u <unit> -n 20 --no-pager
# Expect the refusal message, not the action

# The renewal produced a served change, not just a disk change
echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null \
  | openssl x509 -noout -dates

# The retention job left the things that must survive
# (every currently-running artifact is still resolvable)

# The job survives a reboot
systemctl is-enabled <unit>.timer
```

A scheduled job is not checked until you have seen it run on its own schedule and produce the intended result. Until then you have tested the code, not the automation.
