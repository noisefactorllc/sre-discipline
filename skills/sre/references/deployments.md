# Deployments Reference

This reference covers traps, diagnostic commands, and checklists for deployment pipelines. It explains commit triggers, push ancestry, and deployments whose reported result differs from production state.

`containers.md` covers what happens to the container once the pipeline reaches it. This doc covers everything before that point.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation goes through CI/CD:

- [ ] **Trigger set computed**: you know exactly which pipelines this commit fires, and you want all of them to run
- [ ] **Functional change separated from mechanical cleanup**: Keep broad pipeline cleanup in a separate commit. Push it when its downstream effects are acceptable.
- [ ] **Push ancestry checked**: in any shared checkout, every commit that will land is yours or approved
- [ ] **Step ordering reviewed**: the reload or restart happens after the copy, so a failed copy leaves the process running old state
- [ ] **Every step fails loud or is explicitly advisory**: see `verification.md`
- [ ] **Deploy evidence plan is remote-only**: you will confirm from CI logs, the remote, and the running service, never from the local tree
- [ ] **Rollback is a forward action**: the previous artifact is still pullable and you know its tag or digest

## Traps

### Editing the pipeline is deploying

Deployment workflows commonly trigger on changes to their own filenames. Editing a workflow can therefore deploy everything that workflow controls. For example, removing a dead step from sixteen workflows can trigger thirteen simultaneous production deployments. The cleanup changes no functional code.

Before committing changes to multiple pipelines, determine every trigger. Read each changed workflow's path filters. Match those filters against the full changed-file list. Then decide whether all resulting runs are wanted.

Prefer one commit for the functional change, limited to the necessary files. Keep mechanical cleanup in a separate commit. Push that commit when deployments without functional changes are acceptable. Alternatively, include each cleanup with the next planned change to its pipeline.

A dead endpoint called by a soft-failing step (`curl ... || echo "warning"`) is cleanup, not an incident. It does not justify a fan-out.

### Path filters do not apply to every push

Path filtering compares against a base. A new branch push may have no base, so jobs that you expected to skip can run. Do not rely on path filters alone to prevent deployments. Know what each pipeline does when it runs.

### Pushing a SHA pushes its ancestors

`git push origin <sha>:<branch>` pushes that commit and every ancestor. In a shared checkout, a rebase can place another person's unpushed commit below yours. Pushing your approved SHA then also pushes their commit.

```bash
# Before any push in a shared checkout: what is actually about to land?
git log origin/<branch>..<sha-to-push> --format='%h %an %s'
# Every listed commit must be yours or approved

# If a foreign commit sits below yours, replant your work on the remote tip
git rebase --onto origin/<branch> <foreign-sha>
# Or cherry-pick onto a temp branch and push that instead
```

### A red deploy that half-applied

A pipeline failure leaves the effects of earlier steps in place. A file-sync step can write new files and then fail. The pipeline skips the restart that would load those files. The new configuration exists on disk, but the process still runs the old configuration. A failure result does not mean that nothing changed.

A common cause is an unprivileged `--delete` sync against container state that a root process wrote, such as uploads or caches. rsync cannot remove the root-owned tree and exits non-zero. The pipeline then skips later steps.

- Exclude live runtime state from any `--delete` sync, or use an explicit list of repo-owned paths. Repeated exclusions address each incident separately. By the third occurrence, use the explicit include list.
- On a failed deployment, identify the failed step before concluding that the change is not live. Check whether the restart ran.

```bash
# Did the process actually restart after the commit landed?
docker inspect <name> --format '{{.State.StartedAt}}'
git show -s --format=%cI <sha>
```

### A green deploy that did nothing

A remote CI script without `-e` can continue after its first failure and report success. An earlier deployment may run `chmod +x` on a tracked script. The resulting file-mode difference blocks the next `git pull` with "Your local changes would be overwritten by merge". The checkout remains at an old commit while later deployments report success.

- Commit scripts with the executable bit set, or run `git checkout -- <file>` before pulling.
- Make the remote script fail visibly with `set -euo pipefail`. End it at EOF instead of an explicit `exit`. See `verification.md`.
- Check the checkout is where you think it is: `git -C <path> rev-parse HEAD` on the server, compared against the remote.

### File modes and ownership travel with the sync

`rsync -a` preserves source permissions. A directory with mode 700 on a developer machine becomes unreadable to a non-root container user after deployment. The resulting errors can resemble missing files. Normalize modes after copying into a served tree, or set the correct modes at the source.

### Debugging "why isn't it live" from the wrong place

The local working tree does not prove what a deployment did. Use the remote, CI run, server checkout, and live URL. See `verification.md`.

## Diagnostic Commands

```bash
# What will this commit trigger? Read the filters, match them yourself
git diff --name-only origin/<branch>..HEAD
grep -A8 'on:' .github/workflows/<file>.yml | sed -n '/paths:/,/^$/p'

# What runs actually fired for a SHA (unfiltered; see verification.md)
gh run list --limit 50 --json headSha,name,status,conclusion \
  | jq --arg sha "$SHA" '[.[] | select(.headSha | startswith($sha))]'

# Why did a run fail, and at which step
gh run view <run-id> --log-failed

# What is actually checked out on the server
ssh <host> 'git -C <path> log -1 --format="%h %cI %s" && git -C <path> status --short'

# Which step of a compose-based deploy last touched the container
docker inspect <name> --format '{{.Created}} {{.State.StartedAt}} {{.Image}}'
```

## Verification Commands

```bash
# 1. CI ran and passed for this exact commit
gh run list --limit 20 --json headSha,name,conclusion \
  | jq --arg sha "$SHA" '[.[] | select(.headSha | startswith($sha))]'

# 2. The artifact on the server corresponds to that commit
ssh <host> 'git -C <path> rev-parse HEAD'
docker inspect <name> --format '{{.Image}}'

# 3. The process restarted after the commit
docker inspect <name> --format '{{.State.StartedAt}}'

# 4. The served response reflects the new code
curl -sf "https://<domain>/deployment-meta.json"
curl -sf "https://<domain>/up"
```

All four, every time. Any one of them alone has a failure mode where it reports success for a deploy that did not happen.
