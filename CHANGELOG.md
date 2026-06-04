# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added
- Initial repository scaffold for the Dynatrace → AAP Event-Driven Ansible **pull** integration
- `ROADMAP.md` — Vision, Guiding Principles, Architecture, Phases 0–7 (seeded from the `automation-topics` planning session), Naming Conventions, Decisions Log, Risks
- `docs/architecture.md` — network-traffic flow diagrams (Mermaid): topology, ordered sequence, and the pull-vs-push / no-EdgeConnect rationale
- `rulebooks/dynatrace_problems.yml` — scaffold `dt_esa_api` rulebook (args verified against the upstream `Dynatrace/Dynatrace-EventDrivenAnsible` collection)
- `collections/requirements.yml` — `dynatrace.event_driven_ansible`
- Secrets pattern: `docs/dev-environment.sh.example` (committed) → `docs/dev-environment.sh` (gitignored)
- `ansible.cfg.example` — Hub `galaxy_server` template (never a live `ansible.cfg`)
- Community health files: `LICENSE` (MIT), `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `CLAUDE.md`, `CODEOWNERS`, `.github/` (SECURITY, issue templates, PR template)
- CI (`.github/workflows/ci.yml`): required-files check, `yamllint`, and guards that the secrets file and real tenant id are never committed
- `.yamllint` lint config
- **Configuration as Code** under `aap_config/` (mirrors `dc1.azure`): `load.yml` /
  `validate.yml` apply + verify every `DT-EDA -` object via
  `infra.aap_configuration.dispatch` (username/password auth) with
  `validate_tasks.yml` post-apply assertions
- `aap_config/files/*` CaC data: Hub EE registry/repository sync, Controller
  project/inventory/credential/`DT-EDA - Notify` JT, and the EDA credential type,
  credentials, decision environment, project, and `DT-EDA - Problem Remediation`
  rulebook activation
- `decision-environment.yml` + `decision-environment-requirements.yml` — custom
  DT-EDA decision environment (base `de-minimal-rhel9:1.1.7-21` + certified
  `dynatrace.event_driven_ansible` 1.2.3); built with `ansible-builder`, pushed to
  `quay.io/zigfreed/dt-eda-de` (immutable semver), synced into PAH
- `playbooks/notify_problem.yml` — notify-only action that debugs the matched event
- `playbooks/raise_test_problem.yml` — raise a synthetic problem on demand via the
  Dynatrace Events API v2 (`events.ingest` token)
- `docs/decision-environment.md` — DE build/push/PAH-sync runbook
- `docs/INSTALL.md` — full setup: **manual (AAP UI, by hand)** and **automated
  (CaC)** paths, with inline DE build/tag/push (customer pushes straight to PAH;
  demo uses quay→PAH), operator + Bitbucket adaptation, and screenshot slots
- `docs/DEMO.md` — meeting runbook: both trigger methods (helper playbook + real
  metric-event threshold), the AAP observe-walkthrough, talk track, and
  copy-paste prompts for the Claude running the demo
- `docs/images/README.md` — screenshot shot list (slots referenced by the docs)
- 13 setup/demo screenshots under `docs/images/` (token scopes, PAH EE, the AAP
  objects, rule audit, notify job, problem, metric event), embedded into
  `INSTALL.md`/`DEMO.md`; tenant ids/hostnames/token ids/IPs redacted
- Getting-started Claude Code skills under `.claude/skills/`: `dt-eda-build-de`
  (build + push the decision environment), `dt-eda-install` (apply `load.yml`),
  and `dt-eda-demo` (raise a synthetic problem and watch the loop fire)

### Fixed
- `aap_config/` — live bring-up against APD AAP 2.6 surfaced certified
  `ansible.eda` 2.5.0 incompatibilities with `infra.aap_configuration` 4.4.0's
  EDA roles; resolved so `load.yml` runs clean end-to-end and the activation fires:
  - auth switched from a minted OAuth token to **username/password** (the EDA
    modules reject `controller_token`); removed `aap_config/tasks/aap_token_*.yml`
  - dropped `scm_branch` from the EDA project (module has no such field; EDA syncs
    the repo default branch)
  - `load.yml` now attaches the registry credential to the decision environment via
    the EDA API and restarts the activation (the `decision_environment` module
    silently ignores its `credential` arg, so the DE couldn't pull from PAH)
- `decision-environment.yml` — make the DE actually build on `de-minimal-rhel9`:
  exclude `systemd-python` from reinstall (the base copy is fine; it can't rebuild
  in the de-minimal builder stage), and reassert the `python3` alternative to 3.12
  after `microdnf upgrade` (the upgrade flips it to 3.9, which has no ansible —
  breaking the build check and runtime `ansible-rulebook`)

### Changed
- DT-EDA objects now live in the **`IT Service Automation`** organization
  (`my_organization`, override via `DT_EDA_ORG`)
- `DT-EDA` controller project: `scm_update_on_launch: false` — the rulebook/
  playbooks are stable, so no git re-pull on every Notify launch
- Decision environment `pull_policy: missing` (was `always`) — correct for the
  immutable semver tag; set via the EDA API in `load.yml` (`de_pull_policy`,
  override `DT_EDA_DE_PULL_POLICY`)
- `collections/requirements.yml` — pin `dynatrace.event_driven_ansible 1.2.3`
  (certified) and add the `infra.aap_configuration` 4.4.0 CaC collection stack
- `rulebooks/dynatrace_problems.yml` — notify-only `run_job_template` action with
  CaC-injected match string + a re-trigger `throttle` (no hard-coded secrets/match);
  tuned against the verified flat `dt_esa_api` event shape — throttle keyed on
  `event.problemId`, and richer fields (problemId, displayId, severity, impact,
  affected entities, management zones) passed to the Notify job
- Match is now a **regex** (`event.title is search`); default
  `DT_MATCH_TITLE = "DT-EDA Synthetic|High Disk Usage"` so the synthetic helper AND
  a real **High Disk Usage** metric-event problem both fire the loop (demo method
  B). Synthetic problem title decoupled into `DT_SYNTHETIC_TITLE`. Added optional
  `DT_API_SETTINGS_TOKEN` (settings scope) for managing the metric event via API.
  Live-trigger on real hosts tracked in dc1.azure#1 (OneAgent at provisioning)
- `docs/dev-environment.sh.example` — add `DT_API_EVENT_TOKEN` (events.ingest),
  `DT_MATCH_TITLE`, and optional CaC overrides
- ROADMAP Phases 1–3 advanced to 🔄/✅; Decisions Log rows for CaC, DE delivery,
  certified content, notify-only, synthetic problems, and demo SCM
- `docs/dev-environment.sh.example` — AAP auth now mirrors the dc1.azure method:
  mint a short-lived token from the admin username/password at run time (deleted
  afterward), with a UI-minted `AAP_TOKEN` kept only as an SSO/MFA escape hatch
