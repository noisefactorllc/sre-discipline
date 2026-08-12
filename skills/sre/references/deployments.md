# Deployments Reference

Domain-specific traps, diagnostic commands, and checklist items for the pipeline that ships the change: what a commit triggers, what a push carries, and how a deploy can go red or green while leaving production in a state neither result describes.

`containers.md` covers what happens to the container once the pipeline reaches it. This doc covers everything before that point.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation goes through CI/CD:

- [ ] **Trigger set computed**: you know exactly which pipelines this commit fires, and you want all of them to run
- [ ] **Functional change separated from mechanical cleanup**: a sweeping edit across many pipeline files is its own commit, pushed when its fan-out is acceptable
- [ ] **Push ancestry checked**: in any shared checkout, every commit that will land is yours or approved
- [ ] **Step ordering reviewed**: the reload or restart happens after the copy, so a failed copy leaves the process running old state
- [ ] **Every step fails loud or is explicitly advisory**: see `verification.md`
- [ ] **Deploy evidence plan is remote-only**: you will confirm from CI logs, the remote, and the running service, never from the local tree
- [ ] **Rollback is a forward action**: the previous artifact is still pullable and you know its tag or digest

## Traps

### Editing the pipeline is deploying

Deploy workflows commonly path-trigger on their own filename, so editing a workflow file IS a trigger for whatever that workflow deploys. A single tidy-up commit that strips a dead step from sixteen workflow files fires thirteen simultaneous production deploys of code that did not change, behind a cleanup with zero functional effect.

Before committing multi-pipeline edits, compute the trigger set: for each changed pipeline file, read its path filters and glob-match them against the full changed-file list. Then decide deliberately whether that fan-out is wanted.

Prefer: the functional change in one commit scoped to the files that carry it, and the mechanical sweep in a separate commit pushed when a wave of no-op deploys is acceptable. Cleanup that touches many pipelines can also simply be left to ride along with each pipeline's next natural change.

A dead endpoint called by a soft-failing step (`curl ... || echo "warning"`) is cleanup, not an incident. It does not justify a fan-out.

### Path filters do not apply to every push

Path filtering compares against a base. A push that creates a new branch may have no base to diff against, so filtered jobs run that you expected to be skipped. Do not rely on path filters as a safety mechanism for "this won't deploy anything"; rely on knowing what the pipeline does when it runs.

### Pushing a SHA pushes its ancestors

`git push origin <sha>:<branch>` pushes that commit and every ancestor it has. In a checkout shared by several people or several agent sessions, a rebase can order someone else's unpushed commit below yours, and pushing your approved SHA carries theirs to the remote as well.

```bash
# Before any push in a shared checkout: what is actually about to land?
git log origin/<branch>..<sha-to-push> --format='%h %an %s'
# Every listed commit must be yours or approved

# If a foreign commit sits below yours, replant your work on the remote tip
git rebase --onto origin/<branch> <foreign-sha>
# Or cherry-pick onto a temp branch and push that instead
```

### A red deploy that half-applied

A pipeline that fails partway leaves whatever the earlier steps did. The dangerous version: a file-sync step fails after writing the new files, and the restart step that would load them is skipped. The new configuration sits on disk, the process keeps running the old one, and the red X reads as "nothing happened" when the truth is "the change landed and was never loaded".

The common concrete cause is a sync with `--delete` running as an unprivileged user against state written by a root process inside the container (uploads, caches, generated data). rsync cannot remove the root-owned tree, exits non-zero, and everything downstream is skipped.

- Exclude live runtime state from any `--delete` sync, or invert to an explicit include list of repo-owned paths. Chasing this with one new exclusion per incident is whack-a-mole: the third occurrence is the signal to invert.
- On any red deploy, determine which step failed before concluding the change is not live, and check whether the restart ever ran.

```bash
# Did the process actually restart after the commit landed?
docker inspect <name> --format '{{.State.StartedAt}}'
git show -s --format=%cI <sha>
```

### A green deploy that did nothing

The mirror image. A remote script run through a CI action that does not set `-e` continues past its first failure, and the job reports success. A `chmod +x` on a tracked script during an earlier deploy creates a file-mode diff that blocks the next `git pull` ("Your local changes would be overwritten by merge"), so the checkout stays frozen at an old commit while every subsequent deploy goes green.

- Commit scripts with the executable bit set rather than chmod-ing them at deploy time, or `git checkout -- <file>` before pulling.
- Make the remote script fail loud (`set -euo pipefail`), and end it at EOF rather than with an explicit `exit` (see `verification.md`).
- Verify the checkout is where you think it is: `git -C <path> rev-parse HEAD` on the server, compared against the remote.

### File modes and ownership travel with the sync

`rsync -a` preserves source permissions. A directory that is 700 on a developer machine becomes unreadable to the non-root user inside the container after it lands on the server, and the service fails in ways that look like missing files. Normalize modes after copying into a served tree, or set them correctly at the source.

### Debugging "why isn't it live" from the wrong place

Covered in full in `verification.md`, repeated here because this is where it bites: the local working tree tells you nothing about a deploy. Use the remote, the CI run, the server checkout, and the live URL.

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
