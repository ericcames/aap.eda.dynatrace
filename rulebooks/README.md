# Rulebooks

Event-Driven Ansible rulebooks for the Dynatrace **pull** integration.

| Rulebook | Purpose |
|----------|---------|
| [`dynatrace_problems.yml`](dynatrace_problems.yml) | Poll the Dynatrace problems API (`dt_esa_api`) and launch remediation on a match. **Scaffold** — see [ROADMAP](../ROADMAP.md) Phase 3. |

## How secrets reach the source plugin

The rulebook references `{{ DT_API_HOST }}` and `{{ DT_API_TOKEN }}` rather than literals:

- **Local dev (`ansible-rulebook`):** `source docs/dev-environment.sh` then pass them through, e.g. `ansible-rulebook --rulebook rulebooks/dynatrace_problems.yml -i inventory --env-vars DT_API_HOST,DT_API_TOKEN`.
- **In AAP:** provide them via an EDA **credential** attached to the rulebook activation — never inline in this file.

## Important notes

- `dt_api_host` is the **API** host (`https://<env-id>.live.dynatrace.com`), not the `apps` UI host.
- Token needs **Read problems** + **Write problems** scopes.
- Events are **flat** — match `event.title` / `event.status`. Inspect a real event in Automation Decisions before hardening conditions.
- Begin **notify-only**; add real remediation and the re-trigger guard in Phase 4 (see the architecture doc).
