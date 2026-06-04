# Screenshots

Committed (not gitignored) so they render on GitHub. The docs reference these by
filename; capture and drop them in here. **Never include secrets, real tenant
ids/URLs, or customer data** in a screenshot — crop or redact first.

## Shot list

Referenced by [`INSTALL.md`](../INSTALL.md) and [`DEMO.md`](../DEMO.md):

| File | What to capture |
|------|-----------------|
| `dt-token-scopes.png` | Dynatrace Generate-token screen with **Read/Write problems** (and a 2nd with **Ingest events**) selected |
| `pah-ee-synced.png` | PAH → Execution Environments showing `dt_eda_de` with its tags |
| `aap-de-create.png` | AAP → Automation Decisions → Decision Environments → `DT-EDA - DE` (image + credential) |
| `aap-eda-credtype.png` | EDA Credential Type `DT-EDA - Dynatrace API` (inputs + injectors) |
| `aap-eda-creds.png` | EDA Credentials list showing the `DT-EDA -` credentials |
| `aap-notify-jt.png` | Controller Job Template `DT-EDA - Notify` (project + playbook) |
| `aap-activation-create.png` | Rulebook Activation create form (`DT-EDA - Problem Remediation`) |
| `aap-activation-running.png` | Activation list with status **Running** |
| `dt-problem-open.png` | Dynatrace Problems with the open `DT-EDA Synthetic` problem |
| `dt-metric-event.png` | Dynatrace metric-event/threshold config (optional, method B) |
| `aap-rule-audit.png` | Automation Decisions → Rule Audit showing the matched event |
| `aap-notify-job-output.png` | `DT-EDA - Notify` job output with the matched-problem line |
