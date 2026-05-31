# Why Network and Identity Must Be Designed Together

## Core idea

A secure request succeeds only when **both** are true:
1. Network path exists
2. Caller is authorized

Mathematically:

$$
Access = Network\_Allowed \land Identity\_Allowed
$$

If either side fails, request fails.

---

## 1) Two-gate model

```mermaid
flowchart LR
    REQ[Service Request] --> G1{Network gate\nNSG/UDR/DNS/Private Link}
    G1 -->|Pass| G2{Identity gate\nEntra/RBAC/MI/Token}
    G1 -->|Fail| NETFAIL[Timeout/No route/Unreachable]
    G2 -->|Pass| OK[Authorized and successful]
    G2 -->|Fail| IDFAIL[401/403 Unauthorized]
```

### Interpretation
- Network gate answers: **Can packet reach target?**
- Identity gate answers: **Can caller perform action?**

---

## 2) Real production failure patterns

| Symptom | Network status | Identity status | Typical root cause |
|---|---|---|---|
| Timeout | Fail | Unknown | NSG/UDR/DNS/private endpoint issue |
| 401/403 | Pass | Fail | missing RBAC role / wrong token audience |
| Intermittent failures | Partial | Partial | DNS split-horizon, route asymmetry, token expiry skew |
| Works from one subnet only | Mixed | Pass | subnet policy mismatch |

---

## 3) End-to-end request example

```mermaid
sequenceDiagram
    participant APP as App (Managed Identity)
    participant DNS as Private DNS
    participant NET as VNet Path (NSG/UDR)
    participant AAD as Entra ID
    participant KV as Key Vault (Private Endpoint)

    APP->>DNS: Resolve vault.privatelink.vaultcore.azure.net
    DNS-->>APP: Private IP
    APP->>AAD: Get access token
    AAD-->>APP: Bearer token
    APP->>NET: Connect to private endpoint
    NET->>KV: Forward request
    KV->>KV: Validate token + RBAC
    KV-->>APP: Success/403
```

---

## 4) Design workflow (network + identity together)

```mermaid
flowchart TD
    START[New service integration] --> NET1[Design private network path]
    NET1 --> ID1[Design service identity and roles]
    ID1 --> NET2[Apply NSG/UDR/DNS/Private Endpoint]
    NET2 --> ID2[Configure managed identity/RBAC]
    ID2 --> TEST[Test network reachability]
    TEST --> TEST2[Test token and authorization]
    TEST2 --> GO[Go-live + monitor both dimensions]
```

### Practical checklist
- Connectivity test from runtime subnet
- DNS resolution test of private FQDN
- Token acquisition test (`managed identity`/service principal)
- Authorization test against exact target action

---

## 5) Troubleshooting decision flow

```mermaid
flowchart TD
    ISSUE[Request failed] --> Q1{Can resolve target FQDN?}
    Q1 -->|No| D1[Fix DNS zone link/record]
    Q1 -->|Yes| Q2{Can reach target IP:port?}
    Q2 -->|No| D2[Fix NSG/UDR/Firewall route]
    Q2 -->|Yes| Q3{Token valid and audience correct?}
    Q3 -->|No| D3[Fix identity config/token scope]
    Q3 -->|Yes| Q4{RBAC role present?}
    Q4 -->|No| D4[Grant minimum required role]
    Q4 -->|Yes| APP[Check application logic/service health]
```

---

## Summary

- Network-only design is incomplete.
- Identity-only design is incomplete.
- Production-ready design always validates both gates in one workflow.
