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
- The `registry` addon is **required** so `opcli provision load` can push rock images to `localhost:32000`.
- With **Juju 2.9**, you must pass `--config caas-image-repo=jujusolutions` via `extra-bootstrap-args`. Without it, bootstrap fails with `invalid reference format` (see Known Issues below).
- The MicroK8s `channel: 1.29/stable` keeps classic confinement, which is compatible with Juju 2.9 (the default `1.35-strict/stable` also works once `caas-image-repo` is set, but was not separately validated).

For machine charms add an `lxd:` provider block.

### 5. Generate `spread.yaml` and `tests/integration/run/task.yaml`

```bash
opcli spread init
```

Then add the `runner:` label for your self-hosted GitHub Actions runner to the
generated `spread.yaml`:

```yaml
backends:
  integration-test:
    systems:
    - ubuntu-24.04:
        runner: [self-hosted, edge]   # adjust to your runner labels
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
Without the MicroK8s `registry` addon, this check fails and images are never pushed,
so `opcli pytest expand` falls back to local `.rock` file paths that Juju cannot use.

**Fix**: add `registry` to the MicroK8s `addons:` list in `concierge.yaml`.

### `artifacts-generated.yaml` extra fields in resources

Old-style `artifacts-generated.yaml` files may have `file:` on resource entries.
opcli's Pydantic model rejects this with `Extra inputs are not permitted`.
**Fix**: remove the `file:` fields from resources (step 3 above).

### `spread tasks` `"ubuntu-latest"` default

If no `runner:` is specified in `spread.yaml`, `opcli spread tasks` emits
`"ubuntu-latest"` for every test. Integration tests that need MicroK8s or LXD
**must** run on self-hosted runners — set the `runner:` field.

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
- [ ] `concierge.yaml` created with correct providers and snaps
- [ ] `spread.yaml` generated and runner labels set
- [ ] `tests/integration/run/task.yaml` generated
- [ ] All `opcli` local validation commands pass (matrix, tasks, expand, pytest expand)
- [ ] `.github/workflows/ci.yaml` created
