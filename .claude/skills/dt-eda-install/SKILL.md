---
name: dt-eda-install
description: >-
  Install (or re-install) the DT-EDA Dynatrace-pull integration into a working
  AAP via aap_config/load.yml — creates the Controller + EDA objects, syncs the
  decision environment into PAH, and starts the rulebook activation. TRIGGER when
  the user asks to install / bootstrap / set up DT-EDA on AAP, deploy the
  Config-as-Code, or "run load.yml". SKIP for normal code edits, or if the user
  only wants to inspect aap_config/ without applying it.
---

# Install DT-EDA into AAP

Drive the install interactively. Authoritative reference:
[`aap_config/README.md`](../../../aap_config/README.md) — follow its steps; this
skill adds the interactive checking and reporting. The decision environment must
be built + pushed first (skill `dt-eda-build-de`).

## Guardrails

- **Never print secret values** or put them in tool-call arguments. Check env
  vars by **name only** (`printenv NAME >/dev/null && echo set`). When one is
  missing, ask the *user* to export it — suggest `! export NAME=...` in the
  prompt so it lands in this session's shell without you echoing it. The repo's
  canonical secret file is the gitignored `docs/dev-environment.sh`.
- This **mutates a live AAP** (creates credentials/projects/inventory/JT, EDA
  objects, and a running activation). Confirm `AAP_HOSTNAME` before running.
- Idempotent + additive — objects are namespaced `DT-EDA -`, so re-runs are safe.
- **AUTH = username/password, NOT a token.** Do not set `AAP_TOKEN`: the certified
  `ansible.eda` 2.5.0 modules reject `controller_token` and the EDA roles fail.
- **All exports and the playbook must be in ONE shell call** — shell state does
  not persist across separate Bash invocations.

## Steps

1. **Prerequisites.** Verify `ansible-galaxy` is available and the repo has
   `aap_config/`. Confirm the DT-EDA decision environment image is pushed to
   quay.io (run skill `dt-eda-build-de` if not). Confirm the user has an AAP
   instance with EDA enabled and a Private Automation Hub (PAH).

2. **Collections.** Confirm `~/.ansible.cfg` is a real file (not a symlink) with
   `server_list = rh_certified, rh_validated, community`, then:
   `ansible-galaxy collection install -r collections/requirements.yml`.

3. **Secrets / environment.** If `docs/dev-environment.sh` exists, have the user
   `! source docs/dev-environment.sh`. Otherwise check each var by name (from
   `docs/dev-environment.sh.example`). Required: `AAP_HOSTNAME`,
   `AAP_CONTROLLER_USERNAME`, `AAP_CONTROLLER_PASSWORD`, `DT_API_HOST`,
   `DT_API_TOKEN` (Read + Write problems). Optional with good defaults:
   `AAP_VALIDATE_CERTS` (false), `DT_EDA_ORG` (`IT Service Automation`),
   `AH_HOSTNAME` (gateway host), `DT_EDA_DE_VERSION` (`v1.0.0`), `DT_EDA_DE_IMAGE`
   (PAH copy; set to `quay.io/zigfreed/dt-eda-de:latest` for a Hub-less smoke
   test), `DT_EDA_SCM_BRANCH` (`main`), `DT_POLL_DELAY` (60). `DT_API_EVENT_TOKEN`
   is only needed later for the demo (skill `dt-eda-demo`), not for install.

4. **Apply.** Confirm `AAP_HOSTNAME`, then run exports + playbook in one call:
   ```bash
   source docs/dev-environment.sh && \
   ansible-playbook -i aap_config/inventory/ aap_config/load.yml
   ```
   `load.yml` drives the PAH sync of the DE image, creates all DT-EDA objects,
   attaches the registry credential to the decision environment via the EDA API
   (the `ansible.eda.decision_environment` module ignores its `credential` arg),
   restarts the activation if needed, then imports `validate_tasks.yml` which
   asserts every object exists. Expect `failed=0` and a list of `OK:` / `OK
   (EDA):` lines.

5. **Verify the activation is running.** The activation must pull the DE from PAH
   and start:
   ```bash
   curl -sk -u "$AAP_CONTROLLER_USERNAME:$AAP_CONTROLLER_PASSWORD" \
     "$AAP_HOSTNAME/api/eda/v1/activations/?name=DT-EDA%20-%20Problem%20Remediation" \
     | python3 -c "import sys,json;[print(a['status'],'|',a.get('status_message')) for a in json.load(sys.stdin)['results']]"
   ```
   If status is `failed` with an image-pull error, the DE registry credential
   isn't attached or the image/tag is wrong — re-run `load.yml` (its post-dispatch
   block re-attaches the credential and restarts). If status is `running`, you're
   done.

6. **Report.** Summarize the created objects: Controller `DT-EDA` project +
   inventory, `DT-EDA - Hub Registry` credential, `DT-EDA - Notify` JT; EDA
   `DT-EDA - Dynatrace API` credential type + credential, `DT-EDA - Controller`
   and `DT-EDA - Hub Registry` credentials, `DT-EDA - DE` decision environment,
   `DT-EDA` project, and `DT-EDA - Problem Remediation` activation (running). Tell
   the user to run skill `dt-eda-demo` to see the loop fire. The flow is
   notify-only — no remediation until that's deliberately added.
