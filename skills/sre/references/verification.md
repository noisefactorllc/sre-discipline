# Verification Reference

Domain-specific traps, diagnostic commands, and checklist items for proving that a change did what you think it did.

Every other reference tells you to check. This reference explains how verification itself can mislead. A probe can measure the wrong target, report another command's exit code, or answer the wrong question. An unreliable instrument converts uncertainty into a false claim. The operator then acts on that claim.

## Pre-flight Items

Add these to your Phase 2 checklist for any operation you intend to check (which is all of them):

- [ ] **Exit status comes from the tool itself**: A pipeline, remote login shell, or wrapper must not replace that status.
- [ ] **Probes are specified for the worst case**: multiple attempts, real timeouts, retries on anything cold, sleeping, rebooting, or behind a mesh
- [ ] **Evidence comes from the deployed layer**: the remote, the live URL, the running container. Never the local working tree
- [ ] **Absence claims come from unfiltered queries**: a filtered query returning empty is not proof that the thing does not exist
- [ ] **Assertions state a positive contract**: "X is present and does Y", not "X never happens"
- [ ] **Every gate can actually fail**: no `|| true`, no `|| echo "warning"`, no step whose failure is invisible

## Traps

### Pipes eat exit codes

`cmd | tail -20` reports tail's exit status, which is essentially always 0. Truncation compounds it: the tail window can cut the failure summary while leaving a reassuring "426 passed" line in view. A test run with five failures reads as green, exit 0.

```bash
# WRONG: the status is tail's, and the failure block may be off-screen
npx playwright test | tail -30

# RIGHT: capture everything, record the runner's own status, then read
npx playwright test > run.log 2>&1; echo "EXIT=$?" >> run.log
tail -40 run.log
```

Where the runner writes its own result artifact (Playwright's `test-results/.last-run.json`, JUnit XML, a JSON report), trust that over anything you scraped from a terminal.

### Remote shells rewrite exit codes

An explicit `exit 0` at the end of an SSH heredoc can return 1 to the client. With `set -e` active, bash runs `~/.bash_logout` when the login shell exits. A logout command such as Ubuntu's `clear_console -q` can fail without a TTY and replace the intended status.

```bash
# WRONG: explicit exit inside a login-shell heredoc under set -e
ssh user@host <<'REMOTE'
set -euo pipefail
do_the_thing
exit 0
REMOTE

# RIGHT: let the script end at EOF, branch with if/else instead of exit
ssh user@host <<'REMOTE'
set -euo pipefail
if needs_bootstrap; then
  bootstrap
else
  do_the_thing
fi
REMOTE
```

The general rule: the status you read must be the status of the thing you ran. Anything between the two (a pipe, a login shell, a CI action wrapper) is a place where it gets replaced.

### Steps that cannot fail are not gates

`curl -sf https://example.com/health || echo "Warning: health check failed"` always succeeds. A remote CI script without `-e` can also continue after a failed command and report success. Such pipelines can report a successful deployment when no deployment occurred.

Decide deliberately for each step whether it is a gate (must fail the run) or advisory (must not). Then make the code say so. An advisory step that everyone believes is a gate is the worst of both.

### Under-specified probes manufacture outages

`ping -c 1 -W 1` against a device that routinely ignores its first packet reports it down. Reporting that as a finding sends the operator chasing a failure that does not exist. The same applies to a single un-retried HTTP probe against anything that was just rebooted, is power-saving, or is behind a wireless hop.

- Minimum `ping -c 3` with a realistic timeout.
- Wrap HTTP and API probes in a retry loop.
- Watch for the convergence pattern: early misses that become consistent hits mean the target was settling, not failing.
- When a probe result contradicts what the operator observes directly, their observation wins. Fix the instrument, do not defend the number.

### Filtered queries are not absence proofs

A filtered API query that returns an empty list proves only that the filter matched nothing then. In one observed case, `gh run list --commit <sha>` returned `[]` ten minutes after a push. The unfiltered list showed three completed runs for that exact head SHA. Never base a wait loop or absence claim on a server-side filter.

```bash
# WRONG: empty result treated as "no runs exist"
gh run list --commit "$SHA" --json status

# RIGHT: list unfiltered, filter client-side on a field you can see
gh run list --limit 50 --json headSha,name,status,conclusion \
  | jq --arg sha "$SHA" '[.[] | select(.headSha | startswith($sha))]'
```

### Local working-tree state is not deploy evidence

The local checkout does not explain why a change is not live. When several machines or sessions push, a local working tree represents neither the pushed commit nor the deployed artifact. An uncommitted local file does not explain a change pushed elsewhere.

Diagnose only from authoritative layers: the remote (`gh api repos/<owner>/<repo>/commits/<sha>`), CI logs, live URL with cache-busting, and running container. Treat the remote commit and deployed artifact as ground truth.

### Negative assertions go stale silently

An absence check, such as "this file never ships" or "this variable is not set", depends on another component. Nothing forces the check to track that component's changes. It can keep passing while describing a mechanism that no longer exists. It can also fail without identifying a real defect, requiring manual cleanup.

Assert the positive contract instead: the file IS shipped at this path, the variable IS set to this canonical value. That fails loudly the moment the mechanism is removed. If an absence genuinely is the contract, pair it with a positive control in the same check so a vacuous pass is impossible.

### Retries and raised timeouts destroy evidence

Calling a failure "flaky" ends the investigation. Increasing a limit from 30s to 120s or 300s to 600s can hide the evidence. Adding `retries: 2` can do the same. A failure that retries conceal leaves no artifact.

- Obtain a controlled comparison first. Did the same step pass on previous runs? Does the single failed job pass on the same commit when rerun? Identical code passing provides evidence about the environment, not the product.
- Reproduce on real hardware before theorizing. A step that takes 22s locally and over 12 minutes on a shared runner: that gap IS the finding.
- Improve diagnostics instead of increasing limits. Make failures capture state such as readyState, network logs, screenshots, and the loaded configuration. The next investigation can then start with evidence instead of only a stack trace.

### Checking from the wrong side of the change

Checking the thing you changed from a position that bypasses the layer you changed proves nothing. This shows up in every domain:

| You changed | Wrong check | Right check |
|-------------|-------------|-------------|
| CDN cache | curl the origin directly | curl through the edge, read cache-status headers, check every edge |
| DNS record | dig against the local resolver | dig against several external resolvers |
| TLS cert | inspect the file on disk | `openssl s_client` against the served endpoint |
| Container config | read the file in the repo | inspect the running container's env and start time |
| Health of a proxied service | hit the proxy's own status | hit a path that exercises the backend |

## Diagnostic Commands

```bash
# Capture full output plus the runner's real status
<command> > run.log 2>&1; echo "EXIT=$?" >> run.log

# Preserve the true status through a pipe (bash/zsh)
set -o pipefail
<command> | tee run.log

# Read the status of a specific stage rather than the pipeline
<command> | tee run.log; echo "stage statuses: ${PIPESTATUS[@]}"

# Retry loop for a cold or flaky-by-nature target
for i in 1 2 3 4 5; do
  curl -sf --max-time 10 "https://<domain>/up" && break
  echo "attempt $i failed"; sleep 5
done

# Multi-packet reachability, not a single probe
ping -c 3 -W 2 <host>

# Was this container actually recreated by the deploy?
docker inspect <name> --format '{{.State.StartedAt}}'
```

## Verification Commands

```bash
# The claim "the deploy is live" needs all three, from the deployed side
gh run list --limit 20 --json headSha,name,conclusion | jq --arg sha "$SHA" \
  '[.[] | select(.headSha | startswith($sha))]'          # CI actually ran and passed
docker inspect <name> --format '{{.State.StartedAt}}'    # process restarted after the commit
curl -sf "https://<domain>/deployment-meta.json"         # served artifact reports the new SHA

# The claim "the service is healthy" needs a path that reaches the backend
curl -sf "https://<domain>/up" && echo OK || echo FAIL

# The claim "the host is unreachable" needs more than one packet
ping -c 3 -W 2 <host> || echo "confirmed unreachable after 3 packets"
```

Before reporting a result, consider whether the measurement method could explain it. If so, measure again correctly before reporting.
