# Contributing

Contributions to the Dynatrace → AAP Event-Driven Ansible (pull) setup are
welcome — rulebook improvements, docs, decision-environment work, and roadmap
items. Read [`ROADMAP.md`](ROADMAP.md) and [`CLAUDE.md`](CLAUDE.md) first.

## Content & secret policy

**Never commit:**

- Dynatrace API tokens or AAP tokens
- The real Dynatrace tenant id / URL — use the `<env-id>` placeholder; the real
  value lives only in the gitignored `docs/dev-environment.sh`
- Passwords, private keys, or session cookies
- Customer or company names, or other identifying details

Per-developer values go in `docs/dev-environment.sh` (copied from
`docs/dev-environment.sh.example`). CI fails the build if the real tenant id or
that file is committed.

## Workflow

1. Branch off `main` (e.g. `phase-3/notify-only-rulebook`).
2. Make one focused change. Keep YAML clean (`yamllint .` against `.yamllint`).
3. Update `CHANGELOG.md` under `[Unreleased]`.
4. Update `ROADMAP.md` phase status or the Decisions Log if the plan changes.
5. Open a PR using the template; fill in Summary, Test plan, and Risk/rollback.

## Pull requests

- One concern per PR — group by shared root cause, not item count.
- Descriptive title, e.g. `Phase 3: notify-only Dynatrace rulebook activation`.
- Behavior changes and anything risky stay isolated.
