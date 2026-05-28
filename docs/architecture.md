# Architecture — network traffic flows

How AAP Event-Driven Ansible and Dynatrace SaaS communicate in the **pull** model.

## The one rule that shapes everything

**Every connection is initiated by AAP, outbound.** Dynatrace SaaS never opens a
connection into your network — it only ever *responds* to AAP's requests, or
*receives* AAP's write-back. Because there is no inbound, **EdgeConnect is not
required** (EdgeConnect exists only to let Dynatrace SaaS reach private
endpoints — the opposite direction).

## Topology and directions

```mermaid
flowchart LR
    subgraph onprem["On-prem OpenShift — AAP 2.6 (dev: Ansible Product Demos)"]
        eda["EDA rulebook activation<br/>source: dt_esa_api"]
        controller["Automation Controller<br/>remediation job templates"]
    end
    subgraph saas["Dynatrace SaaS — tenant &lt;env-id&gt;"]
        api["Problems API<br/>&lt;env-id&gt;.live.dynatrace.com"]
    end

    eda -->|"1 · poll: GET /api/v2/problems (outbound 443)"| api
    api -.->|"2 · HTTP response: problem JSON"| eda
    eda -->|"3 · on match: launch remediation"| controller
    eda -->|"4 · write-back: comment / close (outbound 443)"| api
```

**Legend:** solid arrows = connections **AAP initiates** (outbound HTTPS 443);
dashed arrow = the HTTP **response** that returns on AAP's own poll connection.
No arrow originates at Dynatrace.

## Ordered flow

```mermaid
sequenceDiagram
    participant EDA as AAP EDA (on-prem OpenShift)
    participant DT as Dynatrace SaaS Problems API
    participant C as Automation Controller

    loop every `delay` seconds (default 60)
        EDA->>DT: GET /api/v2/problems  (outbound 443)
        DT-->>EDA: problems JSON
    end

    Note over EDA: rulebook condition matches (event.status / event.title)
    EDA->>C: launch remediation job template
    EDA->>DT: POST comment / close problem  (Write problems scope)
    DT-->>EDA: 200 OK
```

## Why not push (and when EdgeConnect would come back)

The alternative is **push**: Dynatrace Workflows send events *into* an AAP event
stream. That requires Dynatrace SaaS to reach the **private** AAP route — an
inbound connection — which is exactly what **EdgeConnect** is for. We chose pull
specifically to avoid that. If a future use case needs Dynatrace Workflows to
orchestrate, revisit push + EdgeConnect as a separate workstream.

| | Pull (this repo) | Push (not used) |
|---|---|---|
| Who connects | AAP → Dynatrace (outbound) | Dynatrace → AAP (inbound) |
| Mechanism | `dt_esa_api` polls problems API | Event stream / `dt_webhook` |
| EdgeConnect | **Not needed** | Required for private AAP |
| Decisioning | In AAP/EDA rulebooks | In Dynatrace Workflows |

## Connectivity requirements (pull)

- Outbound **443** from the EDA activation pod to `https://<env-id>.live.dynatrace.com`.
- An **egress proxy**? Set the plugin's native `proxy:` parameter (see the rulebook).
- Dynatrace **access token** with **Read problems** + **Write problems**.
- **No** inbound firewall rules, route exposure, or EdgeConnect.

> A polished PNG of these flows can be added under [`images/`](images/) later;
> the Mermaid above renders directly on GitHub and stays diff-able.
