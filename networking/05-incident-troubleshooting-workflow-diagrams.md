# Azure Networking Incident Troubleshooting Workflow Diagrams

## What is it?
This topic is an incident response playbook for diagnosing Azure network faults using ordered decision steps.

## What is it used for?
It is used by operators to reduce MTTR for DNS, routing, NSG, private endpoint, and identity-adjacent issues.

## Why is it important?
A standard troubleshooting sequence avoids random trial-and-error and speeds safe recovery.

## Workflow
```mermaid
flowchart LR
    A[Alert received] --> S[Scope impact]
    S --> D[Run DNS/path checks]
    D --> C[Validate policy controls]
    C --> F[Apply fix and verify]
```

## Goal

Use a structured, repeatable workflow to debug Azure networking incidents quickly.

---

## 1) Master incident workflow

```mermaid
flowchart TD
    ALERT[Incident alert: timeout/latency/error] --> SCOPE[Define impact scope]
    SCOPE --> DNS{DNS resolution ok?}
    DNS -->|No| FIXDNS[Fix DNS records/zone links/resolver]
    DNS -->|Yes| PATH{Route path correct?}
    PATH -->|No| FIXROUTE[Fix UDR/next hop/firewall path]
    PATH -->|Yes| NSG{NSG allows required flow?}
    NSG -->|No| FIXNSG[Fix NSG priority/rules]
    NSG -->|Yes| PE{Private endpoint healthy?}
    PE -->|No| FIXPE[Fix private endpoint + DNS mapping]
    PE -->|Yes| ID{Identity/RBAC issue?}
    ID -->|Yes| FIXID[Fix token audience/role assignment]
    ID -->|No| APP[Inspect app/backend health]
```

---

## 2) Workflow by symptom

## A) Timeout / connection failure

```mermaid
flowchart LR
    T[Timeout] --> D1[Check DNS resolution]
    D1 --> D2[Check route effective path]
    D2 --> D3[Check NSG allow rules]
    D3 --> D4[Check destination health probes]
```

Typical causes:
- unresolved private DNS name
- blocked port by NSG/firewall
- wrong next hop in route table

---

## B) 403 / unauthorized after network appears healthy

```mermaid
flowchart LR
    E[403 Unauthorized] --> N[Network path is fine]
    N --> TOK[Validate token issuer/audience]
    TOK --> RBAC[Validate RBAC role assignment]
    RBAC --> OK[Retest request]
```

Typical causes:
- missing role assignment on target resource
- wrong managed identity used at runtime
- wrong token audience/scope

---

## C) Intermittent failures

```mermaid
flowchart TD
    I[Intermittent failures] --> Q1{One subnet only?}
    Q1 -->|Yes| S[Subnet-specific NSG/UDR drift]
    Q1 -->|No| Q2{One region only?}
    Q2 -->|Yes| R[Regional DNS or edge issue]
    Q2 -->|No| Q3{Time-based pattern?}
    Q3 -->|Yes| T[Token expiry, autoscale, connection reuse]
    Q3 -->|No| B[Backend saturation/probe instability]
```

---

## 3) Incident command checklist

### Minute 0-5
- Confirm impacted services and regions
- Capture exact error type (timeout/reset/403/5xx)
- Validate DNS response from affected runtime

### Minute 5-15
- Validate route/NSG/firewall path
- Validate private endpoint and DNS mapping
- Validate backend health probe states

### Minute 15-30
- Validate identity and role assignments
- Compare healthy vs unhealthy subnet/region config
- Apply minimal safe rollback/fix

---

## 4) Evidence-first troubleshooting flow

```mermaid
flowchart TD
    START[Issue reported] --> E1[Collect evidence: logs, traces, flow logs]
    E1 --> E2[Reproduce from affected source]
    E2 --> E3[Pinpoint failing hop]
    E3 --> E4[Apply smallest corrective change]
    E4 --> E5[Verify recovery + monitor]
    E5 --> POST[Write post-incident action items]
```

---

## 5) Post-incident improvement workflow

```mermaid
flowchart LR
    RCA[Root cause identified] --> G1[Add guardrail policy]
    RCA --> G2[Add alert for early detection]
    RCA --> G3[Update runbook]
    RCA --> G4[Schedule game day test]
```

### Typical preventive actions
- DNS monitoring for private zones
- NSG/UDR drift detection
- private endpoint health alerting
- identity role change audit alerts

---

## Summary

Use this order consistently:
1. DNS
2. Route path
3. NSG/firewall
4. Private endpoint
5. Identity/RBAC
6. App/backend health

This keeps incident response fast and avoids random trial-and-error.
