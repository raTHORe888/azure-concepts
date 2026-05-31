# AKS Reliability and Disaster Recovery

## Why this matters
Production workloads need resilience against zone, node, and region failures.

## Reliability layers
- Multi-zone node pools
- Pod disruption budgets and anti-affinity
- Backup/restore for cluster state and data
- Multi-region failover strategy

```mermaid
flowchart LR
    USERS[Users] --> GSLB[Global Traffic Manager]
    GSLB --> R1[AKS Region A]
    GSLB --> R2[AKS Region B]
    R1 --> DB1[(Primary data)]
    R2 --> DB2[(Replica/DR data)]
```

## DR workflow
```mermaid
flowchart TD
    START[Define RTO/RPO] --> ARCH[Design zone/region topology]
    ARCH --> BACKUP[Set backup/restore strategy]
    BACKUP --> DRILL[Run failover drills]
    DRILL --> IMPROVE[Fix gaps and update runbooks]
```

## Portal checks
1. Node pools distributed across zones
2. Backup configuration status
3. Traffic failover profile health
4. Cross-region data replication health

## Azure CLI checks
```bash
# Node pool zone info
az aks nodepool list -g <rg> --cluster-name <aks> --query "[].{name:name,zones:availabilityZones}" -o table

# Pod spread and disruptions
kubectl get pdb -A
kubectl get pods -A -o wide
```

## What good looks like
- DR drill meets target $RTO$ and $RPO$
- Region failure leads to controlled failover
