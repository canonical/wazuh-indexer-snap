# Fork maintenance

This repository (`canonical/wazuh-indexer-snap`) is a fork of
[`canonical/opensearch-snap`](https://github.com/canonical/opensearch-snap)
("upstream"). It packages the same OpenSearch snap, rebranded and adjusted for
use as the Wazuh indexer.

## Remotes

Add the upstream repository as a remote once, locally:

```shell
git remote add upstream git@github.com:canonical/opensearch-snap.git
git fetch upstream
```

## Branch mapping

| This repo (`origin`) | Upstream branch        | Notes                                   |
|-----------------------|-------------------------|------------------------------------------|
| `2/edge`              | `upstream/2/edge`       | Track 2 (OpenSearch 2.x) — clean history |
| `main`                | `upstream/main`         | Historical/legacy, messy history         |

`2/edge` is maintained as: **an upstream commit, plus a small stack of
Wazuh-specific patches on top**. It is *not* a squash of upstream history —
upstream's own commits are kept as-is so `git log`, `git blame` and future
rebases behave normally against `opensearch-snap`.

Current state of `2/edge`:

- Base commit (last commit consumed from upstream):
  `upstream/2/edge` @ `7bfaef0` — "Add non-core plugins (#107)".
- On top of it, 5 patches carry all Wazuh customizations, grouped by concern:
  1. `snap/snapcraft.yaml` rename + Wazuh apt repository
  2. `snap/hooks/install` path fixes tied to the rename
  3. CI / release / Jira sync workflow adjustments
  4. Local dev/test script adjustments
  5. Docs (README/CONTRIBUTOR/SECURITY)

`main` predates this cleanup: it also forks from the `2/edge` upstream track,
but its history (73 commits, including several "fix stuff"/"debug" commits and
a few "merge upstream" commits that are not real git merges) has not been
rewritten. Rebuilding `main` the same way `2/edge` was rebuilt is a reasonable
future follow-up, but is out of scope here.

## Why this matters for `git rerere`

Keeping our changes as a short, stable stack of patches (rather than one
giant diff or an entangled history) means that when upstream publishes new
commits and we rebase our patches on top, the same textual conflicts tend to
reappear in the same few files. With `git rerere` enabled, git remembers how
we resolved a conflict the first time and replays that resolution
automatically on the next rebase — as long as the patch stack stays small and
each patch stays focused on one concern, rerere resolutions stay valid across
syncs.

Enable it once, locally:

```shell
git config --global rerere.enabled true
git config --global rerere.autoupdate true
```

## Syncing `2/edge` with upstream

1. Fetch upstream and see what changed since our last synced commit:

   ```shell
   git fetch upstream
   git log --oneline 7bfaef0..upstream/2/edge   # replace 7bfaef0 with the current base
   ```

2. Rebase our patch stack onto the new upstream tip:

   ```shell
   git checkout 2/edge
   git rebase --onto upstream/2/edge 7bfaef0 2/edge
   ```

3. Resolve any conflicts (rerere will auto-resolve repeats). Fix up each patch
   in place — keep the same grouping (snapcraft/hooks, CI, scripts, docs) so
   future rebases stay easy to reason about. Split out a new patch if upstream
   introduces something that needs a dedicated Wazuh-specific change.

4. Update this file's "Current state of `2/edge`" section with the new base
   commit.

5. Run `just lint`, `just static` and `just test` (or the repo's existing
   validation commands) before pushing.

6. Force-push the rebased branch: `git push --force-with-lease origin 2/edge`.
   Coordinate with other contributors beforehand — this rewrites branch
   history.

## Adding a new Wazuh-specific change

Prefer amending/extending the existing patch that matches its concern
(snapcraft/rename, hooks, CI, scripts, docs) instead of adding a new one-off
commit, so the patch stack doesn't grow unbounded. Only add a new commit when
the change doesn't fit any existing category.
