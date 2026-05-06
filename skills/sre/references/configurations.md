# Configurations Reference

Domain-specific traps, diagnostic commands, and checklist items for managing application configuration files (JSON, YAML, env files, anything bind-mounted into a container) as a versioned, validated artifact rather than a server-side file you edit by hand.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves application config:

- [ ] **Config exists in source control** — every file that changes runtime behavior is in a tracked repo. If a config exists only on a server, that's a latent incident; bring it into the repo before doing anything else
- [ ] **Secrets templated** — secret values are `${VAR}` placeholders, never inline
- [ ] **CI render path verified** — the deploy pipeline substitutes secrets and ships the rendered file; the repo never contains real secret values
- [ ] **Pre-merge schema lint runs first** — a CI step that walks every config and refuses to merge if a required field is missing, gating all deploy jobs behind it
- [ ] **Drift detection available** — a script that pulls live configs, redacts back to placeholders, and diffs against the repo, so server-side edits are visible
- [ ] **No manual edits since last deploy** — `pull-config`-equivalent shows no drift before you start

## Traps

### Configs that exist only on the server

The most expensive trap in this category. A config file lives at `/etc/<service>/<config>` (or `/home/<user>/<service>/config/`) on a production host. It's mounted into a container. It's never been in any repo. You search the codebase for "is this field set?" and the result is empty, but the running service has it. You search for "is the new required field set?" and the result is also empty — and that's the bug, but you can't tell from the repo.

Symptoms:

- Schema mismatch between image and config doesn't surface until startup
- Pre-merge validation can't catch missing fields because the configs aren't in the merge
- Server-side edits are invisible to git history; rollback has no source of truth
- Knowledge of "what's actually deployed" lives only in whoever last logged in

Fix: bring the config into source control as the FIRST step of any work that touches it, even if the immediate task is something else. The cost of the move is low; the cost of leaving it untracked is the next incident.

### Server-side edits that survive the next deploy

Even if a config IS in source control, an SSH-then-edit on the server is non-reproducible. CI will overwrite it on the next deploy, and there's no git history of what was changed or why. Sometimes the deploy doesn't run for weeks, so the manual change persists silently — until someone pushes a config change for a different reason and the manual fix vanishes with no warning.

Always edit configs in the repo, push, let CI ship. If you absolutely must edit on a server (incident, no-CI-window), the FIRST step after the immediate fix is to backport the change to the repo and verify with drift detection that the live state matches.

### Schema mismatch between image and config

A new image version enforces a config field at startup that the deployed configs don't have. Containers crash-loop. This is the failure mode that makes "configs in source control with pre-merge validation" non-negotiable: with the lint, the bad config never reaches the server. Without it, the bad config reaches the server, the image is pulled, every container that mounts that config crashes simultaneously.

The lint should mirror the image's startup checks. Every required field the image fails on at startup should be a lint failure at PR time.

### Per-server layout differences

Different servers may put configs at different paths (`/home/deploy/sites/<site>/config/` vs `/home/<user>/<service>/config/`, etc.). A naive deploy script that hardcodes a single base path can't deploy to all of them.

Make the base path per-server in the deploy script:

```bash
declare -A REMOTE_BASES=(
    [server-a]="/home/deploy/sites"
    [server-b]="/home/deploy/sites"
    [server-c]="/home/<user>"
)
```

So both layouts can coexist with the same deploy machinery.

### Secret leakage into the repo

A real secret value committed to even a private repo is a latent leak — it's in git history, it's in any clones, it's in any forks, it's in any backups. Templated `${VAR}` placeholders solve this only if every PR is checked for accidental real values.

Treat secret-leakage detection as a separate concern: a pre-commit hook or CI step that greps the diff for high-entropy strings, base64-shaped values, known secret prefixes (`sk_`, `pk_`, `xoxb-`, etc.) and refuses to merge if any look real.

### Implicit shared schema

When multiple configs share a value (e.g., several services bind-mount different config files but all need the same database connection string), it's tempting to copy-paste. That works until the value changes. Now you have to change it in N places, miss one, and that one service silently keeps using the stale value.

Use a named secret per shared value (`SHARED_DB_CONNECTION`) referenced by every config that needs it. The render step substitutes from a single source. Updates apply to all consumers atomically.

### Configs that depend on undocumented runtime invariants

A config field's correct value sometimes depends on a property of the deploy that isn't written down: "this field has to match the container name", "this path has to exist before the container starts", "this port has to be free on the host". When that invariant changes, the config breaks and the cause is opaque.

If a config depends on a runtime invariant, document it in the config itself (a sibling comment / README) AND in the lint:

```python
# In the schema lint:
def lint_one(path):
    cfg = json.load(open(path))
    if cfg.get("proxy_backend", {}).get("target", "").startswith("http://"):
        target_host = ...  # extract host portion
        # The host MUST match a container name on the same docker network.
        # Without that match, the proxy can't reach its backend.
        if not docker_container_name_exists(target_host):
            errors.append(f"proxy target {target_host} doesn't match any container name")
```

Lints should encode the invariants you wish someone had told you about before you debugged them at 2 AM.

## Diagnostic Commands

```bash
# Find every config the running container actually mounts
docker inspect <name> --format '{{range .Mounts}}{{.Source}}->{{.Destination}}{{"\n"}}{{end}}'

# Compare repo template against rendered live config
ssh <host> "cat /path/to/<config>" > /tmp/live.json
python3 bin/template-config redact /tmp/live.json > /tmp/live.redacted
diff repo/<server>/<site>/config/<config>.json /tmp/live.redacted

# List every config in the repo missing a required field
grep -L '"<required_field>"' <server>/*/config/<name>.json

# Find configs that have unrendered placeholders on the server (CI bug)
ssh <host> "grep -rE '\\\$\\{[A-Z_]+\\}' /path/to/configs/" 2>/dev/null

# Drift check across all servers in one pass
for server in <list>; do
  echo "=== $server ==="
  bin/pull-config $server 2>&1 | grep -E 'DRIFT|drift|^\+|^-'
done
```

## Verification Commands

After any config-touching deploy:

```bash
# Container picked up the new config (started after the deploy)
docker inspect <name> --format '{{.State.StartedAt}}'

# Service responds at expected endpoints
curl -sf https://<domain>/up

# No crash-loop / restart churn
docker ps --filter name=<name> --format '{{.Status}}'
# Should be "Up <time>" not "Restarting"

# Logs show successful config load
docker logs <name> --tail 100 2>&1 | grep -iE 'config|startup|ready'

# Drift check after deploy: repo state == live state
bin/pull-config <server>
# Expect: "No drift detected"

# Schema lint passes against the deployed tree
bin/lint-configs <server>
# Expect: exit 0
```

## What "Config-as-Code" Buys You

When configs are in source control with pre-merge validation:

1. **Schema mismatches surface at PR time, not at startup.** The lint catches missing required fields before the bad config can reach a server.
2. **Drift is visible.** A `pull-config`-equivalent shows when the live state has diverged from the repo (a real bug or a documentation gap, either way worth knowing).
3. **Rollback is `git revert`.** The deploy pipeline rolls forward to the prior config; `.bak` files on the server are belt-and-suspenders, not the source of truth.
4. **Secrets are auditable separately.** The repo shows what fields exist; the secret store shows what values they take. Rotation, leak response, and access-grant audits each touch one concern, not both.
5. **Cross-service consistency is enforceable.** A lint that knows "every config in this tree must have field X" catches the day someone adds a new service and forgets.

The cost of getting here is one-time (the move into the repo, plus the lint). The cost of NOT being here recurs every time a deploy needs to know what the production config looks like.
