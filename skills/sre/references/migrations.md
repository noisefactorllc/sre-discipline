# Migrations Reference

Domain-specific traps, diagnostic commands, and checklist items for any deploy that changes the structure of shared state — database schemas, config schemas, environment variables, secrets, or anything else the application reads at startup or runtime.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation introduces a new requirement on shared state:

- [ ] **New required structure identified** — list every new field, column, env var, or secret the deploying version needs to find when it starts up
- [ ] **Shared state already has it** — verify the requirement is satisfied in production BEFORE the new code can run. `psql -c "\d <table>"`, `grep -L '<field>' configs/`, `docker inspect <name> --format '{{.Config.Env}}'`
- [ ] **Backward-compatible during overlap** — the new structure must not break the currently-deployed version. Additive changes only (new columns, new fields, new vars) until the old version is fully retired
- [ ] **Forward sequencing planned** — if dropping legacy structure is part of the work, that's a SECOND deploy after the new code is stable; not the same change
- [ ] **Pre-merge validation in place** — a CI gate that walks the source-of-truth files and refuses to merge if a required field is missing (see `configurations.md`)
- [ ] **Rollback for the migration itself** — schema migrations are state changes, not code; their rollback is a separate plan from the app rollback

## Traps

### Deploying code that requires shape it can't find

The most common version of this trap: a new app version enforces a new required field at startup (or assumes a new column exists, or reads a new env var) before the shared state has been updated. The image starts, fails fast, and crash-loops.

This breaks two ways:

1. **Image rolled before configs** — the image enforces `field_X`, configs don't have it yet. Every container that mounts the affected configs crash-loops. Fix-forward by adding the field, or rollback the image to a tag that doesn't require it.
2. **Image rolled before schema** — the image queries a column that hasn't been added yet, or a table that doesn't exist. Same crash-loop, same two fixes.

The fix is sequencing. Update the shared state first, verify, then deploy the code that depends on it.

### "It worked when I tested locally"

Local tests pass because the dev environment was set up alongside the new code. Production has been running the previous version, so its shared state matches the previous version. The first time the new code touches production state is at deploy time.

Always check production state explicitly before deploy:

```bash
# DB schema check
psql -h <prod> -c "\d <table_the_new_code_needs>"

# Config field presence
grep -L '"<new_field>"' /path/to/configs/*.json

# Env var presence in running containers vs persistent files
docker inspect <name> --format '{{.Config.Env}}' | tr ',' '\n' | sort > /tmp/running-env
sort /etc/<service>/.env > /tmp/persistent-env
diff /tmp/running-env /tmp/persistent-env
```

### One-shot drops bundled with adds

A migration that adds a new column AND drops an old column in the same change is a rolling-deploy hazard. During the rollout, half the containers run the old version (which reads the old column) and half run the new version (which reads the new column). Whichever column got dropped, that half breaks.

Split the work:

1. **Migration 1 (additive):** add the new column, backfill it, leave the old column in place.
2. **Deploy the new code** that reads/writes the new column. Old code still works because the old column still exists.
3. **Soak.** Wait long enough to be confident no rollback to the old code is needed.
4. **Migration 2 (subtractive):** drop the old column.

Same pattern applies to renaming a column, changing a column type, splitting one column into two, or any "one column becomes another" change.

### Backfill that doesn't backfill

Adding a NOT NULL column with a backfill default looks safe but isn't:

- The migration runs against the current row count. New rows inserted between "migration starts" and "migration finishes" may not get the default applied if the schema-tool's order is wrong.
- A NOT NULL column with no default rejects inserts from any in-flight transaction that doesn't know about it.
- Triggers that fire on the column may run differently than you expect when the column first appears mid-transaction.

When backfilling:

1. Add the column as nullable first (or with a default that's known-safe).
2. Backfill in batches; verify count.
3. THEN tighten the constraint (add NOT NULL, drop the default) in a second migration.

### Stale legacy fields after migration

Once a new structure is the source of truth, the old structure becomes a trap: it's still there, still readable, still has values, but those values are frozen at pre-migration time. Any code that reads it gets stale data and may make wrong decisions silently.

After moving to a new structure:

1. Audit every read site. Grep for the old field/column/var.
2. Rewrite those sites to read the new structure.
3. After all reads are migrated, drop the old structure (separate deploy).

Don't leave old reads in place "as a fallback." A fallback to stale data is worse than a crash — the crash is loud, the stale data is silent.

### Migration touches state from multiple deploys

If multiple services share a database (or config tree, or secret store), a schema change touches all of them. Don't deploy the migration without checking every service's compatibility with the new schema.

```bash
# Find every service that connects to a given DB
grep -rln '<db_connection_string>' .

# For each one, check it can run against both old and new schemas
# (additive changes are usually safe; renames and drops never are without coordination)
```

## Diagnostic Commands

```bash
# Compare schema in dev / staging / prod (pick the columns you care about)
for env in dev staging prod; do
  echo "=== $env ==="
  psql -h <$env-host> -c "\d <table>" | grep -E '<column_pattern>'
done

# Find every config that should have a new field but doesn't
grep -L '"<new_field>"' configs/**/<config_name>.json

# Check every container's actual running env against its persistent .env file
for c in $(docker ps --format '{{.Names}}'); do
  diff <(docker inspect $c --format '{{range .Config.Env}}{{println .}}{{end}}' | sort) \
       <(sort /home/<user>/$c/.env 2>/dev/null) && echo "$c: OK" || echo "$c: DRIFT"
done
```

## Verification Commands

After any migration that changed shared state:

```bash
# Schema actually changed
psql -c "\d+ <table>"

# Backfill completed (count matches expectation)
psql -c "SELECT COUNT(*) FROM <table> WHERE <new_column> IS NOT NULL"

# Indexes/constraints actually exist
psql -c "\di <table>*"
psql -c "SELECT conname, contype FROM pg_constraint WHERE conrelid = '<table>'::regclass"

# Config rendered correctly (no unrendered placeholders)
grep -E '\$\{[A-Z_]+\}' /path/to/deployed/config.json && echo "FAIL: placeholders not rendered" || echo "OK"

# Container actually picked up the new config (not still running with old)
docker inspect <name> --format '{{.State.StartedAt}}'
docker logs <name> --tail 50 2>&1 | grep -i 'config\|loaded\|startup'

# Service actually works with the new state
curl -sf https://<domain>/up && echo "OK" || echo "FAIL"
```

## Sequencing Cheat Sheet

For any change that affects both shared state and app code:

```
1. Make the shared-state change in a backward-compatible way
   (add column, add field, add env var; keep old in place)

2. Deploy and verify on a non-critical target

3. Roll out to all targets, verifying each

4. Soak. Watch monitoring. Be confident no rollback is needed.

5. (Optional) Second deploy: drop legacy structure. Same one-target-
   at-a-time discipline.
```

The single most reliable way to break production with a migration is to skip step 1 and let the new app version land before its required state exists. The single most reliable way to leave a long-tail incident behind is to skip step 5 and leave both structures live forever, with code that reads from one and writes to the other.
