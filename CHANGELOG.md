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

### Changed
- `docs/dev-environment.sh.example` — AAP auth now mirrors the dc1.azure method:
  mint a short-lived token from the admin username/password at run time (deleted
  afterward), with a UI-minted `AAP_TOKEN` kept only as an SSO/MFA escape hatch
