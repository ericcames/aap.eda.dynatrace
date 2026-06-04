---
name: dt-eda-build-de
description: >-
  Build the DT-EDA decision environment with ansible-builder and push it to
  quay.io (immutable semver tag) so EDA/PAH can consume it. TRIGGER when the user
  asks to build / rebuild / push / bump the decision environment (DE), the EE/DE
  image, or "the dt-eda-de image". This is the prerequisite for dt-eda-install.
  SKIP for normal code edits, or if the image is already pushed and unchanged.
---

# Build & push the DT-EDA decision environment

Drive this interactively. Authoritative reference:
[`docs/decision-environment.md`](../../../docs/decision-environment.md) — follow
its steps; this skill adds the checking and reporting around them. The DE
definition is [`decision-environment.yml`](../../../decision-environment.yml);
its contents come from
[`decision-environment-requirements.yml`](../../../decision-environment-requirements.yml).

## Guardrails

- **Never print secret values** or put them in tool-call arguments. quay.io and
  registry.redhat.io creds come from `~/.config/containers/auth.json` and
  `~/.ansible.cfg` — reference them, don't echo them.
- Pushing an image is an **outward-facing, hard-to-reverse** action. Confirm the
  target repo + tag with the user before `podman push`.
- The DE uses an **immutable semver tag** (`vMAJOR.MINOR.PATCH`), never a moving
  `latest` for EDA to pin. Bump per the rule in `docs/decision-environment.md`:
  CVE-only → patch · +collection → minor · new base → major.

## Steps

1. **Prerequisites.** Verify `ansible-builder` and `podman` are installed and the
   user is logged in to `registry.redhat.io` (base image). Confirm
   `~/.ansible.cfg` is a **real file, not a symlink** (ansible-builder's COPY/ADD
   does not follow symlinks) and that its `server_list` includes `rh_certified`
   so the certified `dynatrace.event_driven_ansible` is pulled. Confirm quay.io
   creds exist in `~/.config/containers/auth.json` (check by registry name only).

2. **Pick the version.** Ask the user for the semver tag (default to the
   `ee_version` in `aap_config/group_vars/all.yml`, e.g. `v1.0.0`). For a content
   change, bump it per the rule above.

3. **Build.** `ansible-builder build -f decision-environment.yml -t dt-eda-de:latest --prune-images`.
   The definition already handles two `de-minimal-rhel9` quirks — do NOT remove
   them: it reasserts the `python3` alternative to 3.12 after `microdnf upgrade`
   (the upgrade flips it to 3.9, which has no ansible), and excludes
   `systemd-python` from reinstall (the base copy is fine; it can't rebuild in
   the de-minimal builder stage). If a build fails, check those first.

4. **Verify the image** before pushing:
   ```bash
   podman run --rm dt-eda-de:latest bash -lc '/usr/bin/python3 --version; ansible-rulebook --version; \
     ls /usr/share/ansible/collections/ansible_collections/dynatrace/event_driven_ansible/extensions/eda/plugins/event_source/'
   ```
   Expect python 3.12, `ansible-rulebook`, and `dt_esa_api.py` present.

5. **Tag + push.** Confirm the repo/tag, then:
   ```bash
   podman tag dt-eda-de:latest quay.io/zigfreed/dt-eda-de:<vX.Y.Z>
   podman tag dt-eda-de:latest quay.io/zigfreed/dt-eda-de:latest
   podman push --authfile ~/.config/containers/auth.json quay.io/zigfreed/dt-eda-de:<vX.Y.Z>
   podman push --authfile ~/.config/containers/auth.json quay.io/zigfreed/dt-eda-de:latest
   ```
   If the push returns `unauthorized` / `access denied`, the quay repo likely
   doesn't exist yet or the robot account lacks Write. Ask the user to create the
   **public** `zigfreed/dt-eda-de` repo and grant the robot Write — you cannot do
   this from the CLI.

6. **Report.** Confirm the pushed tags (the Quay tag API:
   `https://quay.io/api/v1/repository/zigfreed/dt-eda-de/tag/?onlyActiveTags=true`).
   If `ee_version` changed, remind the user to bump it in
   `aap_config/group_vars/all.yml` (or export `DT_EDA_DE_VERSION`) and re-run
   `dt-eda-install` so PAH re-syncs the new tag.
