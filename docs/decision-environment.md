# DT-EDA Decision Environment — build · push · sync

The EDA rulebook activation runs in a **decision environment (DE)** image. The
stock DE does **not** contain `dynatrace.event_driven_ansible`, so we build a
custom one that adds the certified `dt_esa_api` source plugin, push it to
quay.io, and have **Private Automation Hub (PAH)** sync it so EDA pulls it from
inside the platform.

```
ansible-builder ──build──> dt-eda-de:vX.Y.Z ──push──> quay.io/zigfreed/dt-eda-de:vX.Y.Z
                                                          │ (PAH syncs — driven by load.yml)
                                                          ▼
                                          <pah-host>/dt_eda_de:vX.Y.Z ──pull──> EDA activation
```

Definition: [`decision-environment.yml`](../decision-environment.yml) ·
contents: [`decision-environment-requirements.yml`](../decision-environment-requirements.yml).

## Versioning — immutable semver (deliberate-update model)

The DE carries an **immutable semver tag** (`vMAJOR.MINOR.PATCH`), never a
floating `latest`, so a cached copy is never stale. `aap_config/group_vars/all.yml`
pins `ee_version`, and EDA pulls exactly that tag.

| Change | Bump |
|--------|------|
| CVE-only rebuild (no content change) | patch — `v1.0.1` |
| Add/upgrade a collection | minor — `v1.1.0` |
| New base image / breaking | major — `v2.0.0` |

## Prerequisites

- `pip install ansible-builder` (have 3.1.0).
- `podman login registry.redhat.io` (base image — already logged in).
- quay.io creds in `~/.config/containers/auth.json` (already present) — pushes use
  `--authfile`, no interactive login.
- `~/.ansible.cfg` a **real file** (not a symlink) with the Automation Hub token,
  `server_list = rh_certified, rh_validated, community` — pulls certified content
  first. `dynatrace.event_driven_ansible` is certified (signed) at 1.2.3.

## Build + push

```bash
cd /home/eames/git-repos/aap.eda.dynatrace

# 1. Build
ansible-builder build -f decision-environment.yml -t dt-eda-de:latest --prune-images

# 2. Tag the immutable semver + latest
podman tag dt-eda-de:latest quay.io/zigfreed/dt-eda-de:v1.0.0
podman tag dt-eda-de:latest quay.io/zigfreed/dt-eda-de:latest

# 3. Push both (creds from ~/.config/containers/auth.json)
podman push --authfile ~/.config/containers/auth.json quay.io/zigfreed/dt-eda-de:v1.0.0
podman push --authfile ~/.config/containers/auth.json quay.io/zigfreed/dt-eda-de:latest
```

## Sync into PAH + register for EDA

Handled by Config-as-Code — no manual UI steps:

```bash
source docs/dev-environment.sh && \
ansible-playbook -i aap_config/inventory/ aap_config/load.yml
```

`load.yml` creates the `quay_io` remote registry and the `dt_eda_de` Hub EE repo,
syncs the pinned tag into PAH (`aap_config/files/hub_ee_*.yml`), then registers
the `DT-EDA - DE` decision environment pointing at the PAH copy
(`aap_config/files/eda_decision_environments.yml`).

## Shipping an update

Build a new `vX.Y.Z` → push → bump `ee_version` (or export `DT_EDA_DE_VERSION`)
→ re-run `load.yml`. PAH re-syncs the new tag and EDA pulls it once.

## Hub-less smoke test

The image is public on quay.io, so EDA can pull it directly (skip PAH):

```bash
export DT_EDA_DE_IMAGE=quay.io/zigfreed/dt-eda-de:latest
```
