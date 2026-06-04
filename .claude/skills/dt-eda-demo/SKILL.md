---
name: dt-eda-demo
description: >-
  Demonstrate the DT-EDA pull loop live: raise a synthetic Dynatrace problem and
  watch EDA poll it, match the rulebook, and fire the DT-EDA - Notify job in AAP.
  TRIGGER when the user asks to run / show / test the demo, raise a test problem,
  or verify the loop fires end-to-end. SKIP if DT-EDA isn't installed yet (run
  dt-eda-install first).
---

# Run the DT-EDA live demo

Drive this interactively. It raises a synthetic problem via
[`playbooks/raise_test_problem.yml`](../../../playbooks/raise_test_problem.yml)
and confirms the `DT-EDA - Notify` job fires. Requires DT-EDA already installed
(skill `dt-eda-install`) with the activation **running**.

## Guardrails

- **Never print secret values.** Tokens come from the gitignored
  `docs/dev-environment.sh`; reference by name only.
- Ingesting a `CUSTOM_ALERT` opens a real (short-lived, auto-expiring) problem in
  the Dynatrace tenant — fine for a demo tenant. Confirm the tenant is the demo
  one before raising.
- This is **notify-only** — the action just logs the event; nothing is remediated.

## Steps

1. **Prereqs.** Confirm DT-EDA is installed and the activation is `running`
   (query `/api/eda/v1/activations/?name=DT-EDA%20-%20Problem%20Remediation`). If
   not running, point the user at skill `dt-eda-install`. Confirm the env is
   sourced (`! source docs/dev-environment.sh`) and `DT_API_EVENT_TOKEN` is set —
   a SEPARATE Dynatrace token with the **Ingest events** (events.ingest) scope
   (distinct from the rulebook's Read/Write-problems token). If missing, ask the
   user to create it in the Dynatrace UI and export it.

2. **(Optional) clean slate.** Offer to close any leftover open `DT-EDA Synthetic`
   problems first so the board is clean — find them via
   `/api/v2/problems?problemSelector=status("OPEN")` filtered on title, then
   `POST /api/v2/problems/<id>/close`.

3. **Raise the problem.**
   ```bash
   source docs/dev-environment.sh && ansible-playbook playbooks/raise_test_problem.yml
   ```
   Expect `eventIngestResults ... status: OK`. The event title embeds
   `DT_MATCH_TITLE` plus a timestamp, so each run is a unique problem that fires
   fresh (the rulebook `throttle` dedupes only repeats of the *same* title).
   If no problem opens, set `DT_TEST_ENTITY_SELECTOR` (e.g. `type(HOST)`) and
   retry — some tenants need the event attached to an entity.

4. **Watch it fire.** Poll for a NEW `DT-EDA - Notify` job (id greater than the
   previous latest), within roughly one poll cycle (`DT_POLL_DELAY`, default 60s):
   ```bash
   curl -sk -u "$AAP_CONTROLLER_USERNAME:$AAP_CONTROLLER_PASSWORD" \
     "$AAP_HOSTNAME/api/controller/v2/unified_jobs/?name=DT-EDA%20-%20Notify&order_by=-created&page_size=1" \
     | python3 -c "import sys,json;r=json.load(sys.stdin)['results'];print(r[0]['id'],r[0]['status']) if r else print('none')"
   ```
   When it reaches `successful`, fetch its stdout and show the
   `DT-EDA notify — Dynatrace problem matched: title=... status='OPEN'` line to
   confirm the event data flowed through.

5. **Report.** State the path that fired: Dynatrace problem → `dt_esa_api` poll →
   rulebook match → `run_job_template` → `DT-EDA - Notify` (successful). Point the
   user to the activation in AAP (Decisions) and the JT (Templates) to show it in
   the UI. Remind them it's notify-only.
