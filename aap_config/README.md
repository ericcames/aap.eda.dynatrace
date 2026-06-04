# aap_config — Configuration as Code

Stands up the **DT-EDA** (Dynatrace → AAP Event-Driven Ansible, *pull* model)
objects in AAP via the upstream
[`infra.aap_configuration`](https://github.com/redhat-cop/infra.aap_configuration)
`dispatch` role. Mirrors the [`dc1.azure`](https://github.com/ericcames/dc1.azure)
`aap_config/` pattern.

## What it creates (all namespaced `DT-EDA -`)

| Layer | Objects |
|-------|---------|
| Hub   | `quay_io` remote registry → `dt_eda_de` EE repo (synced from quay.io) |
| Controller | `DT-EDA` project (this repo) · `DT-EDA` inventory · `DT-EDA - Hub Registry` credential · `DT-EDA - Dynatrace Events Ingest` credential type + credential · `DT-EDA - Notify` job template · `DT-EDA - Raise Test Problem` job template (GUI trigger, with survey) |
| EDA   | `DT-EDA - Dynatrace API` credential type + credential · `DT-EDA - Controller` credential · `DT-EDA - Hub Registry` credential · `DT-EDA - DE` decision environment · `DT-EDA` project · `DT-EDA - Problem Remediation` rulebook activation |

## Prerequisites

1. The DT-EDA decision-environment image pushed to quay.io — see
   [`docs/decision-environment.md`](../docs/decision-environment.md). `load.yml`
   drives the **PAH sync** of it.
2. Secrets exported: `source docs/dev-environment.sh` (AAP creds + `DT_API_HOST`
   / `DT_API_TOKEN`, plus `DT_API_EVENT_TOKEN` for the GUI trigger credential).
   See [`docs/dev-environment.sh.example`](../docs/dev-environment.sh.example).
3. Collections installed:
   `ansible-galaxy collection install -r collections/requirements.yml`.

## Run

```bash
source docs/dev-environment.sh && \
ansible-playbook -i aap_config/inventory/ aap_config/load.yml
```

Re-running is idempotent (every object is keyed by name). Validate only:

```bash
source docs/dev-environment.sh && \
ansible-playbook -i aap_config/inventory/ aap_config/validate.yml
```

## Auth

`load.yml` / `validate.yml` authenticate with **`AAP_CONTROLLER_USERNAME` /
`AAP_CONTROLLER_PASSWORD`** (basic auth). Nothing is stored. We do *not* mint an
OAuth token here: the certified `ansible.eda` 2.5.0 modules used by
`infra.aap_configuration`'s EDA roles do **not** accept `controller_token`, so a
token makes them fail (`Unsupported parameters: controller_token`). Username/
password works for both the Controller and EDA roles.

## Customer note (Bitbucket)

The demo syncs the rulebook from the public GitHub repo. For a customer on
Bitbucket, set `DT_EDA_SCM_URL` to their Bitbucket repo and add an EDA + a
Controller **Source Control** credential (the public-GitHub demo needs none).
