# Copilot instructions for wazuh-indexer-snap

This repo packages [Wazuh Indexer](https://documentation.wazuh.com/current/getting-started/components/wazuh-indexer.html)
(an OpenSearch distribution) as a snap. It is a fork of
[`canonical/opensearch-snap`](https://github.com/canonical/opensearch-snap) — the
snap is built almost entirely by staging the upstream `wazuh-indexer` deb
package and layering wrapper scripts / snapcraft hooks around it. There is no
application source code here, only packaging: `snap/snapcraft.yaml`, snap
hooks, and shell wrappers under `scripts/`.

> **See [`FORK.md`](FORK.md) for how this fork is maintained against
> upstream** (branch mapping, the `2/edge` patch-stack workflow, `git rerere`
> setup, and how to sync/rebase). Read it before touching branch history or
> adding Wazuh-specific patches.

## Build, install, and test

There is no unit-test suite or linter configured in this repo — validation is
done by building the snap and running it end-to-end.

```shell
# Build the snap (produces wazuh-indexer_<version>_amd64.snap)
snapcraft --debug

# Install it locally
sudo snap install wazuh-indexer_<version>_amd64.snap --dangerous --jailmode

# Connect required interfaces + set kernel sysctls (or do it manually, see setup-dev-env.sh)
bash setup-dev-env.sh

# Create certs, start the daemon, and init the security index (see README.md for full flags)
sudo snap run wazuh-indexer.setup --node-name cm0 --node-roles cluster_manager,data \
    --tls-priv-key-root-pass root1234 --tls-priv-key-admin-pass admin1234 \
    --tls-priv-key-node-pass node1234 --tls-init-setup yes
sudo snap start wazuh-indexer.daemon
sudo snap run wazuh-indexer.security-init --tls-priv-key-admin-pass=admin1234
```

Run all three integration checks:
```shell
bash test-dev-cluster.sh --admin-auth-password admin
```

Run a single check (each is its own snap app, defined under `apps:` in
`snap/snapcraft.yaml`, backed by `scripts/wrappers/tests/*.sh`):
```shell
sudo snap run wazuh-indexer.test-node-up --admin-auth-password admin
sudo snap run wazuh-indexer.test-cluster-health-green --admin-auth-password admin
sudo snap run wazuh-indexer.test-security-index-created --admin-auth-password admin
```

CI (`.github/workflows/ci.yaml`, reused by `on_push.yaml` and `release.yaml`)
does the same steps on a real `ubuntu-latest` runner: build → install →
connect interfaces → setup/start → curl the cluster health/prometheus
endpoints → upgrade the snap in place → re-check. There are no mocks; treat
this workflow as the closest thing to a test suite when changing packaging
behavior.

## Architecture

- `snap/snapcraft.yaml` is the source of truth: it defines the `environment:`
  block (`OPENSEARCH_*`, `OPS_ROOT`, etc.) shared by hooks and apps, the
  `parts:` that build the snap, and the `apps:` exposed as `wazuh-indexer.<app>`.
- Three parts build the snap:
  - `dependencies` — stages OS packages/libs needed at runtime (no build).
  - `wrapper-scripts` — copies `scripts/wrappers/*` and `scripts/helpers/*`
    verbatim into `opt/opensearch` inside the snap (note: this internal path
    keeps the historic `opensearch` name even though the product is now
    `wazuh-indexer` — that's intentional, not a bug to "fix").
  - `wazuh-indexer` — stages the real `wazuh-indexer` deb package, installs
    extra OpenSearch plugins (Prometheus exporter, repository-s3/gcs/azure)
    via `opensearch-plugin`, disables the shipped `opensearch.yml` defaults,
    and (in `override-prime`) renames `opensearch-plugin`/`opensearch-keystore`
    to `*.orig` so the custom `plugin-wrapper.sh`/`keystore-wrapper.sh` can
    take over — this exists to make snap refreshes/upgrades safe
    (see referenced issue canonical/opensearch-operator#537).
- `snap/hooks/install` and `snap/hooks/configure` run at install/configure time
  to lay out `$SNAP_DATA` folders, rewrite `opensearch.yml`/`jvm.options` paths,
  and set ownership to `snap_daemon`.
- Most `apps:` (`cli`, `node`, `shard`, `env`, `upgrade`, `keystore`, etc.) all
  point at the same `scripts/wrappers/bin-wrapper.sh` command, differentiated
  only by an `environment: bin_script: <name>` value — to expose a new
  OpenSearch bin command, add an app entry with the right `bin_script`, don't
  write a new wrapper script.
- `scripts/wrappers/*.sh` are the actual entry points invoked by snap apps
  (`setup.sh`, `security-init.sh`, `plugin-wrapper.sh`, `keystore-wrapper.sh`,
  `bin-wrapper.sh`, `security/tls/*`). They `source` the shared helpers in
  `scripts/helpers/` (`io.sh`, `set-conf.sh`, `snap-logger.sh`) using
  `${OPS_ROOT}/helpers/...` — helpers are never executed directly.

## Conventions

- All config mutation goes through `scripts/helpers/set-conf.sh`'s
  `set_yaml_prop` / `remove_yaml_prop` (backed by `yq`), never raw `sed`/manual
  YAML edits, so array vs. scalar quoting stays consistent.
- Privileged file operations run as the `snap_daemon` user/group via
  `set_access_restrictions` (in `io.sh`) or `setpriv` (in `bin-wrapper.sh`,
  `replace_in_file`) — follow this pattern for any new script that touches
  `$SNAP_DATA`/`$SNAP_COMMON`.
- Wrapper scripts follow a consistent CLI style: a `usage()` heredoc, a
  `parse_args` function built on `getopt` with a `LONG_OPTS_LIST` array, then
  `set_defaults`/`validate_args` before doing the actual work (see `setup.sh`,
  `test-dev-cluster.sh`).
- The release channel is derived from the snap version at publish time:
  `channel="${version%.*}/edge"` (e.g. version `4.11.0` → channel `4.11/edge`).
  Bump `version:` in `snap/snapcraft.yaml` deliberately when you want to target
  a new channel.
- This repo maintains two branches against upstream `opensearch-snap`: `main`
  (tracks `upstream/main`) and `2/edge` (tracks `upstream/2/edge`, kept as a
  small, rerere-friendly patch stack on top of a real upstream commit — see
  `FORK.md` on that branch for the sync workflow). Keep Wazuh-specific changes
  isolated per concern (snapcraft/rename, hooks, CI, scripts, docs) rather than
  mixed into large commits, so future rebases onto upstream stay easy.
