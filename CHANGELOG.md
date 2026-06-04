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
  `infra.aap_configuration.dispatch`, with a self-managing short-lived token
  (`tasks/aap_token_*.yml`) and `validate_tasks.yml` post-apply assertions
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

### Changed
- `collections/requirements.yml` — pin `dynatrace.event_driven_ansible 1.2.3`
  (certified) and add the `infra.aap_configuration` 4.4.0 CaC collection stack
- `rulebooks/dynatrace_problems.yml` — notify-only `run_job_template` action with
  CaC-injected match string + a re-trigger `throttle` (no hard-coded secrets/match)
- `docs/dev-environment.sh.example` — add `DT_API_EVENT_TOKEN` (events.ingest),
  `DT_MATCH_TITLE`, and optional CaC overrides
- ROADMAP Phases 1–3 advanced to 🔄/✅; Decisions Log rows for CaC, DE delivery,
  certified content, notify-only, synthetic problems, and demo SCM
- `docs/dev-environment.sh.example` — AAP auth now mirrors the dc1.azure method:
  mint a short-lived token from the admin username/password at run time (deleted
  afterward), with a UI-minted `AAP_TOKEN` kept only as an SSO/MFA escape hatch
