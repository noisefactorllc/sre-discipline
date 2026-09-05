# Configurations Reference

This reference covers traps, diagnostic commands, and checklists for application configuration files. These include JSON, YAML, environment files, and any file bind-mounted into a container. Manage them as versioned, validated artifacts.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves application config:

- [ ] **Config exists in source control** — Every file that changes runtime behavior must be in a tracked repo. Add configurations that exist only on servers to the repo before any other work.
- [ ] **Secrets templated** — secret values are `${VAR}` placeholders, never inline
- [ ] **CI render path checked** — The deployment pipeline substitutes secrets. It ships the rendered file. The repo must never contain real secret values.
- [ ] **Pre-merge schema lint runs first** — CI must check every configuration. It must block merges if any required field is missing. All deployment jobs must depend on this check.
- [ ] **Drift detection available** — A script must retrieve live configurations, replace secrets with placeholders, and compare them with the repo. This makes server-side edits visible.
- [ ] **No manual edits since last deploy** — `pull-config`-equivalent shows no drift before you start

## Traps

### Configs that exist only on the server

A configuration may exist only at `/etc/<service>/<config>` or `/home/<user>/<service>/config/` on a production host. A container mounts the file, but no repo contains it. A codebase search for an existing field returns nothing, although the running service has that field. A search for a missing required field also returns nothing. The repo cannot distinguish these cases.

Symptoms:

- Schema mismatch between image and config doesn't surface until startup
- Pre-merge validation can't catch missing fields because the configs aren't in the merge
- Server-side edits are invisible to git history. Rollback has no source of truth.
- Knowledge of "what's actually deployed" lives only in whoever last logged in

Add the configuration to source control as the FIRST step of any work that touches it. This applies even if the immediate task is different. Moving the configuration costs little. Leaving it untracked risks another incident.

### Server-side edits that survive the next deploy

Editing a server through SSH is not reproducible, even when source control contains the configuration. CI overwrites the edit at the next deployment. Git history records neither the edit nor its reason. If deployment waits weeks, the manual change persists silently. An unrelated configuration deployment can then remove the manual fix without warning.

Always edit configurations in the repo. Push the changes. Let CI deploy them. If you must edit a server during an incident or a period without CI, backport the immediate fix first. Then check that the live state matches the repo through drift detection.

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

A real secret in any repo, including a private repo, remains in git history, clones, forks, and backups. `${VAR}` placeholders solve this only if every PR includes a check for accidental real values.

Check for secret leakage separately, through a pre-commit hook or CI step. Search the diff for high-entropy strings, base64-shaped values, and known secret prefixes such as `sk_`, `pk_`, and `xoxb-`. Refuse the merge if any value looks real.

### Implicit shared schema

When multiple configs share a value (e.g., several services bind-mount different config files but all need the same database connection string), it's tempting to copy-paste. That works until the value changes. Now you have to change it in N places, miss one, and that one service silently keeps using the stale value.

Use a named secret per shared value (`SHARED_DB_CONNECTION`) referenced by every config that needs it. The render step substitutes from a single source. Updates apply to all consumers atomically.

### Configs that depend on undocumented runtime invariants

A configuration value sometimes depends on an undocumented deployment property. Examples include a matching container name, a path that must already exist, or a host port that must be free. When that property changes, the configuration breaks without an obvious cause.

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
2. **Drift is visible.** A `pull-config` equivalent shows differences between the live state and the repo. These differences can indicate a bug or a documentation gap.
3. **Rollback is `git revert`.** The deployment pipeline deploys the prior configuration. Server `.bak` files provide additional protection. They are not the source of truth.
4. **Secrets are auditable separately.** The repo defines the fields. The secret store defines their values. Rotation, leak response, and access audits each affect one concern.
5. **Cross-service consistency is enforceable.** Lint can require field X in every configuration. It catches a new service that omits the field.

The cost of getting here is one-time (the move into the repo, plus the lint). The cost of NOT being here recurs every time a deploy needs to know what the production config looks like.
