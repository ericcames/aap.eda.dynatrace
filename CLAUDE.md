# Claude Code working guidelines — aap.eda.dynatrace

Repo-specific guidance for AI contributors. Read [`ROADMAP.md`](ROADMAP.md) first.

## What this repo is

A working setup for **Dynatrace → AAP Event-Driven Ansible** in the **pull**
model: EDA's `dt_esa_api` polls the Dynatrace problems API and launches
remediation. It is *not* conversation prep — the planning lives in the
`automation-topics` session it was seeded from.

## Invariants (do not break without a Decisions Log entry)

- **Pull, outbound-only.** AAP initiates every connection to Dynatrace. **No
  EdgeConnect.** If you find yourself adding inbound/event-stream/EdgeConnect
  config, that's the push model — a separate workstream, not this repo.
- **No secrets, ever, in the repo.** Real Dynatrace tenant id and all tokens
  live only in the gitignored `docs/dev-environment.sh`. Committed files use
  `<env-id>` / `REPLACE_ME_*`. CI fails on a tenant-id leak.
- **Secrets file is `docs/dev-environment.sh`** (sourceable, gitignored) — never
  a `.md`. Commit only `docs/dev-environment.sh.example`.
- **Never ship a live `ansible.cfg`.** Only `ansible.cfg.example`. A live
  project-local cfg shadows `~/.ansible.cfg` and breaks Hub collection installs.
- **Verify against the upstream collection.** Plugin args and event shape come
  from `github.com/Dynatrace/Dynatrace-EventDrivenAnsible`, not memory. Facts
  so far: args `dt_api_host` / `dt_api_token` / `delay` / `proxy`; events are
  flat (`event.title`, `event.status`); token scopes **Read + Write problems**;
  API host is `<env-id>.live.dynatrace.com` (not `apps`).
- **Notify before remediate.** Default new automation to notify-only.

## Conventions

- Mirror [`dc1.azure`](https://github.com/ericcames/dc1.azure) structure/tone.
- Namespace AAP/EDA objects with a `DT-EDA -` prefix (see ROADMAP).
- YAML must pass `yamllint` against `.yamllint` (CI gate).
- One concern per PR; update `CHANGELOG.md` under `[Unreleased]`; update the
  ROADMAP phase status / Decisions Log when the plan changes.
- Dev target is the **Ansible Product Demos** AAP 2.6 — install additively.
