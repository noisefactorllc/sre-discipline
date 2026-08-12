# Verification Reference

Domain-specific traps, diagnostic commands, and checklist items for proving that a change did what you think it did.

Every other reference doc in this skill tells you to verify. This one exists because the verification itself lies more often than people expect. A probe that returns a clean number can be measuring the wrong thing, reporting someone else's exit code, or answering a question you did not ask. An unreliable instrument is worse than no instrument: it converts "I don't know" into a confident false claim, and the operator acts on it.

## Pre-flight Items

Add these to your Phase 2 checklist for any operation you intend to verify (which is all of them):

- [ ] **Exit status comes from the tool itself**: not from a pipeline, not from a remote login shell, not from a wrapper that swallows it
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

An explicit `exit 0` at the end of an SSH heredoc script can return 1 to the client. With `set -e` still active, bash runs `~/.bash_logout` on login-shell exit, and a default logout script that fails without a TTY (Ubuntu's `clear_console -q`) replaces the status you meant to return.

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

`curl -sf https://example.com/health || echo "Warning: health check failed"` always succeeds. So does any remote script run through a CI action that does not set `-e`, where the first command fails and the rest run anyway, and the job goes green. A pipeline full of these reports success for a deploy that did nothing.

Decide deliberately for each step whether it is a gate (must fail the run) or advisory (must not). Then make the code say so. An advisory step that everyone believes is a gate is the worst of both.

### Under-specified probes manufacture outages

`ping -c 1 -W 1` against a device that routinely ignores its first packet reports it down. Reporting that as a finding sends the operator chasing a failure that does not exist. The same applies to a single un-retried HTTP probe against anything that was just rebooted, is power-saving, or is behind a wireless hop.

- Minimum `ping -c 3` with a realistic timeout.
- Wrap HTTP and API probes in a retry loop.
- Watch for the convergence pattern: early misses that become consistent hits mean the target was settling, not failing.
- When a probe result contradicts what the operator observes directly, their observation wins. Fix the instrument, do not defend the number.

### Filtered queries are not absence proofs

A filtered API query returning an empty list means "this filter matched nothing right now", not "the thing does not exist". `gh run list --commit <sha>` has been observed returning `[]` ten minutes after a push while the unfiltered list showed all three runs for that exact head SHA, already complete. Never build a wait loop or an absence claim on a server-side filter.

```bash
# WRONG: empty result treated as "no runs exist"
gh run list --commit "$SHA" --json status

# RIGHT: list unfiltered, filter client-side on a field you can see
gh run list --limit 50 --json headSha,name,status,conclusion \
  | jq --arg sha "$SHA" '[.[] | select(.headSha | startswith($sha))]'
```

### Local working-tree state is not deploy evidence

When the question is "why isn't my change live", the local checkout is the wrong place to look. In any environment where more than one machine or session pushes, the working tree in front of you reflects neither what was pushed nor what is deployed. An uncommitted file here explains nothing about a change someone pushed from elsewhere, and chasing it burns the investigation.

Diagnose from authoritative layers only: the remote (`gh api repos/<owner>/<repo>/commits/<sha>`), the CI run logs, the live URL with cache-busting, the running container. Treat the pushed commit on the remote and the deployed artifact as ground truth.

### Negative assertions go stale silently

A check that asserts an absence ("this file never ships", "this variable is not set") encodes a fact about someone else's component, and nothing forces it to track that component's evolution. When the other side changes, the check either stays green while describing a mechanism that no longer exists, or goes red as a tripwire that needs manual cleanup rather than signalling a real defect.

Assert the positive contract instead: the file IS shipped at this path, the variable IS set to this canonical value. That fails loudly the moment the mechanism is removed. If an absence genuinely is the contract, pair it with a positive control in the same check so a vacuous pass is impossible.

### Retries and raised timeouts destroy evidence

Labelling a failure "flaky" ends the investigation, and every fix that raises a number (gate 30s to 120s, cap 300s to 600s, `retries: 2`) buries the signal deeper. Retries are especially corrosive: a masked failure leaves no artifact at all.

- Get the controlled comparison first: did this same step pass on previous runs, and does re-running the single failed job on the same commit pass? Same code passing is evidence about the environment, not the product.
- Reproduce on real hardware before theorizing. A step that takes 22s locally and over 12 minutes on a shared runner: that gap IS the finding.
- Fix diagnosability, never the number. Make the failure dump state (readyState, network log, screenshot, config actually loaded) so the next occurrence starts from evidence instead of a bare stack trace.

### Verifying from the wrong side of the change

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

Before writing any result the operator will read, ask: could this number be an artifact of how I measured it? If yes, measure again properly before reporting.
