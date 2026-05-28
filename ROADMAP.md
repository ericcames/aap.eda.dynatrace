# aap.eda.dynatrace — Roadmap

## Vision

Let **Ansible Automation Platform** react to **Dynatrace** problems automatically,
using Event-Driven Ansible in the **pull** model:

> *"Dynatrace sees the problem. AAP fixes it — on your terms, from inside your
> own network."*

EDA's `dynatrace.event_driven_ansible.dt_esa_api` source polls the Dynatrace
problems API over outbound HTTPS, evaluates rulebook conditions, and launches
remediation (a Controller job template or a playbook), optionally closing the
loop back to the Dynatrace problem. Because AAP initiates every connection,
**no EdgeConnect is required**.

Seeded from the `automation-topics` conversation-prep session
`2026-05-28_customer-sse_dynatrace-aap-eda-pull-integration` (that session stays
the planning artifact; this repo is the working build).

---

## Guiding Principles

- **Pull, not push** — AAP polls Dynatrace (outbound). This is the whole reason
  EdgeConnect is out of scope. Push (Dynatrace Workflows → AAP event stream) is a
  separate future workstream that would reintroduce EdgeConnect.
- **Verify against the upstream collection** — plugin args and event shape come
  from `github.com/Dynatrace/Dynatrace-EventDrivenAnsible`, not assumptions.
- **Notify before you remediate** — every environment starts notify-only, then
  human-gated, then full auto. Blast radius is controlled in two places: the
  Dynatrace management zone/tag *and* the rulebook condition.
- **Secrets never land in the repo** — the real tenant id and all tokens live in
  the gitignored `docs/dev-environment.sh`; committed files use `<env-id>` and
  `REPLACE_ME_*`. CI fails the build on a leak.
- **Ship `ansible.cfg.example`, never a live `ansible.cfg`** — Ansible loads one
  cfg (no merge); a live project-local cfg shadows the home cfg and breaks Hub
  installs of the certified collection.
- **Stands on the Ansible Product Demo AAP** — dev target is the product-demos
  AAP 2.6; install additively and namespace objects so it co-exists.
- **Build in 2.6, promote to 2.5** — prove the loop in the 2.6 dev/test env,
  then promote the proven artifacts to a 2.5 production env.

---

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│ On-prem OpenShift — AAP 2.6 (dev: Ansible Product Demos)        │
│   EDA rulebook activation                                       │
│     └─ source: dynatrace.event_driven_ansible.dt_esa_api        │
│   Automation Controller                                         │
│     └─ remediation job templates                                │
└───────────────┬───────────────────────────────────────────────┘
                │  outbound HTTPS 443 (poll + write-back)
                ▼
┌───────────────────────────────────────────────────────────────┐
│ Dynatrace SaaS — tenant <env-id>                                │
│   Problems API   <env-id>.live.dynatrace.com                    │
└───────────────────────────────────────────────────────────────┘
```

All connections are AAP-initiated and outbound. Full request/response and
sequence diagrams: [`docs/architecture.md`](docs/architecture.md).

---

## Phases

### Phase 0 — Discovery & prerequisites  ⬜
Close the unknowns before building.

- ⬜ Confirm the 2.5 production topology (OpenShift vs VMs; private vs reachable)
- ⬜ Confirm outbound 443 egress (and any proxy) from 2.6 (and 2.5) to `<env-id>.live.dynatrace.com`
- ⬜ Confirm EDA is licensed/enabled in both 2.5 and 2.6; decision-environment registry reachable
- ⬜ Identify the token owner; confirm **Read problems + Write problems** scopes grantable
- ⬜ Pick one low-blast-radius first use case (problem → remediation)
- ⬜ Define success criteria + a kill switch

**Exit criteria:** egress, token feasibility, EDA availability, and the first use case all confirmed.

### Phase 1 — Dynatrace setup  ⬜
- ⬜ Create the API token (**Read problems + Write problems**); record the tenant API host
- ⬜ Define a scoping strategy (a dedicated management zone or tag for the pilot)
- ⬜ Establish a repeatable way to raise a synthetic test problem
- ⬜ Smoke-test the token: `GET /api/v2/problems` returns 200

**Exit criteria:** token authenticates against the problems API; a test problem can be raised on demand.

### Phase 2 — AAP 2.6 foundation (decision environment)  ⬜
- ⬜ Build/extend a **decision environment** including `dynatrace.event_driven_ansible`; push to the DE registry
- ⬜ Create the **credential** holding the Dynatrace token (injects `DT_API_TOKEN`)
- ⬜ Create the **project** (this repo) for the rulebook
- ⬜ If action = `run_job_template`: ensure the Controller JT + EDA→Controller link exist

**Exit criteria:** DE pullable in EDA, credential stored, project syncs, Controller target reachable.

### Phase 3 — Rulebook & activation (notify-only)  ⬜
- ⬜ Author the rulebook from [`rulebooks/dynatrace_problems.yml`](rulebooks/dynatrace_problems.yml)
- ⬜ Point the first action at a benign notify/log job (observe the real event safely)
- ⬜ Create and start the rulebook activation
- ⬜ Tune the condition against the real event shape (`event.title` / `event.status`)

**Exit criteria:** activation polls cleanly; a test problem produces an event with the expected fields.

### Phase 4 — End-to-end validation  ⬜
- ⬜ Happy path: problem → event → match → action fires → optional write-back
- ⬜ Negative path: non-matching problems don't fire
- ⬜ **Idempotency / re-trigger:** an OPEN problem reappears every poll — guard with `throttle: once_within` keyed on the problem id so remediation doesn't relaunch each cycle
- ⬜ Failure modes: token expiry, proxy down, Dynatrace 429, activation restart
- ⬜ Record detection-to-action latency vs `delay`

**Exit criteria:** happy + negative + idempotency behave; latency acceptable; failure modes understood.

### Phase 5 — Hardening  ⬜
- ⬜ Secrets in a proper store + rotation plan; least privilege; scoped management zone
- ⬜ Safety ramp: notify-only → human-gated (AAP workflow approval node) → full auto
- ⬜ Observability: monitor activation health; forward EDA logs; alert if the activation is down
- ⬜ Tune `delay` to respect API limits; documented kill switch

**Exit criteria:** secrets, idempotency guardrails, monitoring, and kill switch in place; sign-off on the auto-remediation policy.

### Phase 6 — Promote to 2.5 production  ⬜
- ⬜ Version-parity check: 2.5 EDA controller / `ansible-rulebook` / DE base supports the rulebook as written
- ⬜ Replicate artifacts: DE (same collection version), credential, project, activation
- ⬜ Prod-scoped token; confirm prod egress + proxy
- ⬜ Canary cutover: notify-only in prod (or a canary management zone), then enable remediation

**Exit criteria:** prod activation validated against a controlled problem; remediation enabled per the agreed ramp.

### Phase 7 — Operationalize & scale  ⬜
- ⬜ Onboard more problem→automation pairings via a standard rulebook pattern
- ⬜ Ops runbook (add a pairing, disable, rotate the token)
- ⬜ Outcome metrics: MTTR delta, % auto-remediated, false-fire rate

**Exit criteria:** at least one additional pairing onboarded via the documented pattern.

---

## Naming Conventions

To co-exist safely in a shared AAP instance, namespace objects with a `DT-EDA -` prefix.

| Object type        | Pattern                          | Example                              |
|--------------------|----------------------------------|--------------------------------------|
| AAP credential     | `DT-EDA - <purpose>`             | `DT-EDA - Dynatrace API Token`       |
| EDA project        | `DT-EDA`                         | `DT-EDA`                             |
| EDA rulebook activation | `DT-EDA - <use case>`       | `DT-EDA - Problem Remediation`       |
| Controller JT      | `DT-EDA - <verb> <object>`       | `DT-EDA - Remediate Disk Pressure`   |
| Decision environment | `DT-EDA - DE`                  | `DT-EDA - DE`                        |

---

## Decisions Log

| Date       | Decision                                                        | Rationale |
|------------|-----------------------------------------------------------------|-----------|
| 2026-05-28 | **Pull** model (`dt_esa_api` polls Dynatrace), not push          | Outbound-only from on-prem; removes the EdgeConnect dependency; AAP owns decisioning |
| 2026-05-28 | **No EdgeConnect**                                               | EdgeConnect is inbound-only (SaaS → private); irrelevant to an outbound poll. Confirmed in Dynatrace EdgeConnect docs |
| 2026-05-28 | Dev AAP = **Ansible Product Demos** AAP 2.6                     | Available on-prem OpenShift env; install additively + namespaced |
| 2026-05-28 | Build in **2.6**, promote to **2.5 production**                 | Prove the loop in test before touching prod; artifacts are portable |
| 2026-05-28 | Plugin args / event shape **verified against upstream collection** | `dt_api_host`/`dt_api_token`/`delay`/`proxy`; flat `event.title`/`event.status`; token needs Read + Write problems |
| 2026-05-28 | Secrets via gitignored **`docs/dev-environment.sh`**; commit only `.sh.example` | Sourceable in one shell call; keeps tokens + real tenant id out of this public repo |
| 2026-05-28 | Mirror **`dc1.azure`** repo conventions                          | Canonical mature-repo template (ROADMAP shape, ansible.cfg.example, .yamllint, CLAUDE.md, CODEOWNERS) |

---

## Risks / Open Questions

- **2.5 production topology** — OpenShift or VMs? Private? Affects promotion logistics, not the pull design.
- **Egress** — does each environment allow outbound 443 to Dynatrace, and is there a proxy to honor (use the plugin `proxy:` param)?
- **Re-trigger storms** — an OPEN problem reappears every poll; without a `throttle` guard, remediation could relaunch every `delay` seconds. Validated in Phase 4.
- **Token scope + rotation** — least privilege (Read + Write problems) and a rotation plan; never in the rulebook.
- **EDA version parity** — confirm 2.5 supports the rulebook/DE as built in 2.6 rather than assuming it.
- **Push, later?** — if Dynatrace Workflows must orchestrate, that's push + EdgeConnect, a separate effort.

---

## Reference Repositories

| Repo | Role |
|------|------|
| [Dynatrace/Dynatrace-EventDrivenAnsible](https://github.com/Dynatrace/Dynatrace-EventDrivenAnsible) | Source of truth for `dt_esa_api` args + event shape |
| [dc1.azure](https://github.com/ericcames/dc1.azure) | Convention template (ROADMAP shape, secrets pattern, lint, CLAUDE.md) |
| [aap.as.code](https://github.com/ericcames/aap.as.code) | AAP bootstrap / CaC patterns |

---

## Status Legend

- ✅ Complete
- 🔄 In progress
- ⬜ Not started
