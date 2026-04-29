# opcli Integration Skill

## Overview

How to integrate an operator repository with
[javierdelapuente/operator-ci-poc](https://github.com/javierdelapuente/operator-ci-poc)
(`opcli`), replacing the `canonical/operator-workflows` approach.

## Files added / modified

| File | Action | Notes |
|------|--------|-------|
| `artifacts.yaml` | **Create** | Declares charms, rocks, snaps. |
| `artifacts-generated.yaml` | **Modify** | Remove legacy `file:` fields from charm resources. |
| `.jujuignore` | **Modify** | Add `*.rock` to exclude rock files from charm package. |
| `concierge.yaml` | **Create** | Declarative env provisioning (Juju, MicroK8s/LXD, snaps). |
| `spread.yaml` | **Create** | Virtual `integration-test` backend with runner labels. |
| `tests/integration/run/task.yaml` | **Create** | Spread task that invokes `opcli pytest expand`. |
| `.github/workflows/ci.yaml` | **Create** | Calls the reusable `integration-test.yml` workflow. |

## Step-by-step procedure

### 1. Install opcli

```bash
uv tool install "git+https://github.com/javierdelapuente/operator-ci-poc.git"
```

### 2. Generate `artifacts.yaml`

```bash
opcli artifacts init
```

Review the output. Add `rock:` links to OCI-image resources that are backed by
a rock in the same repo:

```yaml
resources:
  my-image:
    type: oci-image
    rock: my-rock   # must match a name under rocks:
```

### 3. Fix `artifacts-generated.yaml` (if it already exists)

Remove any `file:` fields from charm resources — they are no longer valid.
Resources should only carry `type:` and `rock:`:

```yaml
# Wrong (old format):
resources:
  my-image:
    type: oci-image
    rock: my-rock
    file: ./my_rock.rock   # <-- remove this

# Correct:
resources:
  my-image:
    type: oci-image
    rock: my-rock
```

### 4. Create `concierge.yaml`

Declare the test environment (Juju channel, providers, host snaps).

**Example (Juju 3.x with MicroK8s):**

```yaml
juju:
  channel: 3.6/stable

providers:
  microk8s:
    enable: true
    bootstrap: true
    addons:
      - hostpath-storage
      - dns
      - registry   # required for opcli provision load

host:
  snaps:
    charmcraft:
```

**Example (Juju 2.9 with MicroK8s — tested working):**

```yaml
juju:
  channel: "2.9/stable"
  extra-bootstrap-args: "--config caas-image-repo=jujusolutions"
  model-defaults:
    test-mode: "true"
    automatically-retry-hooks: "false"

providers:
  microk8s:
    enable: true
    bootstrap: true
    channel: "1.29/stable"
    addons:
      - dns
      - hostpath-storage
      - ingress
      - registry
      - metallb:10.64.140.43-10.64.140.49

host:
  snaps:
    charmcraft:
    jq:
    yq:
    rockcraft:
```

Key notes:
- Do **not** add `registry` to the MicroK8s `addons:` list — `opcli provision registry` (called by both spread's generated `prepare:` and the manual non-spread path) deploys it. Adding it to concierge.yaml is redundant.
- With **Juju 2.9**, you must pass `--config caas-image-repo=jujusolutions` via `extra-bootstrap-args`. Without it, bootstrap fails with `invalid reference format` (see Known Issues below).
- The MicroK8s `channel: 1.29/stable` keeps classic confinement, which is compatible with Juju 2.9 (the default `1.35-strict/stable` also works once `caas-image-repo` is set, but was not separately validated).

For machine charms add an `lxd:` provider block.

### 5. Generate `spread.yaml` and `tests/integration/run/task.yaml`

```bash
opcli spread init
```

Then set the `runner:` label in the generated `spread.yaml` to match your repo's
GitHub Actions runner. Use `ubuntu-latest` for GitHub-hosted runners, or
`[self-hosted, <label>]` for self-hosted:

```yaml
backends:
  integration-test:
    systems:
    - ubuntu-24.04:
        runner: ubuntu-latest   # or [self-hosted, edge] for self-hosted runners
```

### 6. Validate locally

```bash
opcli artifacts matrix    # verify build matrix
opcli spread tasks        # verify CI test matrix (runner labels appear here)
opcli spread expand       # inspect the fully expanded spread.yaml
opcli pytest expand       # verify the tox command assembled from artifacts-generated.yaml
```

### 7. Create `.github/workflows/ci.yaml`

```yaml
name: CI

on:
  pull_request:
  schedule:
    - cron: "0 15 * * SAT"

jobs:
  integration-test:
    uses: javierdelapuente/operator-ci-poc/.github/workflows/integration-test.yml@main
    permissions:
      contents: read
      packages: write
      actions: read
```

## Known issues / things to watch for

### Juju 2.9 bootstrap fails with `invalid reference format`

**Root cause (confirmed):** Canonical's official Juju 2.9 snap is built with the
Go linker flag `-X ...podcfg.JujudOCINamespace=` (empty string), deliberately
overriding the source-code default of `"jujusolutions"`. This forces an empty
OCI namespace. When `caas-image-repo` is not set, Juju constructs `/juju-db` (a
path starting with `/`) as the image reference, which is rejected by Docker's
reference parser as `invalid reference format`.

Error stack trace:
```
creating statefulset for controller: invalid reference format
  caas/kubernetes/provider.buildContainerSpecForController.func1:1244
  cloudconfig/podcfg.tagImagePath:113
  reference.Parse("/juju-db")  ← the leading "/" makes it invalid
```

**Fix**: pass `--config caas-image-repo=jujusolutions` via `extra-bootstrap-args`
in `concierge.yaml` (see step 4 above). This explicitly sets the OCI namespace so
Juju uses `jujusolutions/juju-db:4.4.30` (valid) instead of `/juju-db` (invalid).

### Rock images not loaded into registry (`opcli provision load` skipped)

The `opcli provision load` step only runs if `localhost:32000/v2/` is reachable.
The registry is deployed by `opcli provision registry` — called automatically in
spread's generated `prepare:` section, and explicitly in the manual (non-spread)
path. Do **not** add `registry` to the MicroK8s `addons:` in `concierge.yaml`;
that is redundant and was an earlier mistake.

### Rock files bundled in charm package (836MB charm, EOF on upload)

If a repo has `.rock` files in or under the project root (e.g. `indico_rock/` or
`nginx_rock/`), charmcraft will bundle them into the `.charm` zip unless excluded.
The resulting charm can be 800MB+, causing Juju's charm upload API to return EOF
before the upload completes.

**Symptom**: `POST ".../charms?...": EOF` from the Juju API during `juju deploy`.

**Fix**: add `*.rock` to `.jujuignore`:
```
*.rock
```
Rock images are separate OCI artifacts referenced via `artifacts.yaml`; they must
never be packaged inside the charm.

### `artifacts-generated.yaml` extra fields in resources

Old-style `artifacts-generated.yaml` files may have `file:` on resource entries.
opcli's Pydantic model rejects this with `Extra inputs are not permitted`.
**Fix**: remove the `file:` fields from resources (step 3 above).

### `spread tasks` `"ubuntu-latest"` default

If no `runner:` is specified in `spread.yaml`, `opcli spread tasks` emits
`"ubuntu-latest"` for every test. This is actually the correct default for
GitHub-hosted runners. Only override with `[self-hosted, <label>]` if your repo
requires self-hosted runners (e.g. for arm64 builds or privileged LXD access).

### `opcli spread tasks` not in README

The `opcli spread tasks` command (used by `integration-test.yml`) exists in the
source (`src/opcli/commands/spread.py`) but is not documented in the README as of
the initial integration (SHA `bae8e49`). This is a documentation gap in
`operator-ci-poc` — filed for awareness.

### `opcli artifacts localize` not in README

The `_CI_PREPARE` template calls `opcli artifacts localize` to convert CI-format
artifacts to local paths after downloading from GitHub artifacts. This command is
also absent from the README. Documentation gap in `operator-ci-poc`.

## Checklist for a new repo

- [ ] `artifacts.yaml` generated and rock links added
- [ ] `artifacts-generated.yaml` cleaned of legacy `file:` resource fields
- [ ] `.jujuignore` updated: add `*.rock` (and `*.charm`) to prevent bundling artifacts
- [ ] `concierge.yaml` created with correct providers and snaps
- [ ] `spread.yaml` generated and runner labels set
- [ ] `tests/integration/run/task.yaml` generated
- [ ] All `opcli` local validation commands pass (matrix, tasks, expand, pytest expand)
- [ ] `opcli artifacts build` succeeds and produces a reasonably-sized charm (< 50MB)
- [ ] `opcli provision run` + `opcli provision registry` + `opcli provision load` succeeds
- [ ] `eval "$(opcli pytest expand -- -k <test>)"` passes at least one test
- [ ] `opcli spread run` passes at least one test
- [ ] `.github/workflows/ci.yaml` created
