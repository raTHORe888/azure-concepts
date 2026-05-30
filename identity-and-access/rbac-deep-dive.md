# Azure RBAC Deep Dive

## What is Azure RBAC?
Azure Role-Based Access Control (RBAC) is the authorization system that controls who can do what on which Azure resources.

Every operation on an Azure resource is governed by RBAC.

---

## The Three Questions RBAC Answers

```mermaid
flowchart LR
    WHO[Who? - Security Principal] --> WHAT[What? - Role Definition]
    WHAT --> WHERE[Where? - Scope]
    WHERE --> ACCESS[Access Decision]
```

---

## Core Concepts

| Concept | Meaning |
| --- | --- |
| Security Principal | Who: user, group, service principal, managed identity |
| Role Definition | What: set of allowed and denied actions |
| Scope | Where: management group, subscription, resource group, resource |
| Role Assignment | Binding of principal + role + scope |

---

## Scope Hierarchy

```mermaid
flowchart TD
    MG[Management Group]
    SUB[Subscription]
    RG[Resource Group]
    RES[Resource]

    MG --> SUB
    SUB --> RG
    RG --> RES
```

Permissions assigned at a higher scope are inherited by lower scopes. A role assigned at subscription level applies to all resource groups and resources within it.

---

## Role Assignment Structure

```mermaid
flowchart LR
    PRIN[Security Principal\ne.g. Managed Identity] --> RA[Role Assignment]
    ROLE[Role Definition\ne.g. Reader] --> RA
    SCOPE[Scope\ne.g. Resource Group] --> RA
    RA --> DECISION[Access Decision at Runtime]
```

---

## Common Built-In Roles

| Role | Can Read | Can Write | Can Delete | Can Manage Access |
| --- | --- | --- | --- | --- |
| Owner | ✅ | ✅ | ✅ | ✅ |
| Contributor | ✅ | ✅ | ✅ | ❌ |
| Reader | ✅ | ❌ | ❌ | ❌ |
| User Access Administrator | ✅ | ❌ | ❌ | ✅ |

Beyond these, Azure has 100+ service-specific built-in roles (e.g. Storage Blob Data Reader, Key Vault Secrets User).

---

## Custom Roles

When built-in roles are too broad or too narrow, custom roles let you define exact permissions.

A custom role specifies:
- `Actions` — allowed control-plane operations
- `NotActions` — excluded from allowed
- `DataActions` — allowed data-plane operations
- `NotDataActions` — excluded data-plane
- `AssignableScopes` — where the role can be used

---

## Allow vs Deny

- RBAC is additive by default (sum of allowed actions across all assignments)
- **Deny assignments** explicitly block actions, even if a role allows them
- Deny assignments take precedence over role assignments

```mermaid
flowchart TD
    REQ[Resource Operation Request]
    REQ --> DENY{Deny Assignment exists?}
    DENY -- Yes --> BLOCK[Access Denied]
    DENY -- No --> ALLOW{Role Assignment allows action?}
    ALLOW -- Yes --> GRANT[Access Granted]
    ALLOW -- No --> BLOCK2[Access Denied]
```

---

## Control Plane vs Data Plane

| Plane | What it controls | Example |
| --- | --- | --- |
| Control plane | Azure Resource Manager operations (manage resources) | Create/delete storage account |
| Data plane | Operations on data inside the resource | Read/write blobs in storage |

Some roles cover control plane only, some data plane only, and some both. Always check which plane a role operates on.

---

## Role Assignment Evaluation Order

```mermaid
flowchart LR
    REQ[Incoming request] --> CHECK_DENY[Check deny assignments]
    CHECK_DENY -->|Deny found| DENIED[Denied]
    CHECK_DENY -->|No deny| CHECK_ROLE[Check role assignments across all scopes]
    CHECK_ROLE -->|Permission found| GRANTED[Granted]
    CHECK_ROLE -->|Not found| DENIED2[Denied]
```

---

## Least Privilege Principle

- Only assign the role that is actually needed
- Assign at the narrowest scope possible
- Prefer data-plane roles over broad control-plane roles for app access
- Review and clean up stale role assignments regularly

---

## Practical Checklist

- Role is assigned to the correct security principal
- Scope is at minimum required level
- Role covers required data-plane and/or control-plane actions
- No over-broad built-in role used when a narrower one exists
- Stale assignments are removed on identity lifecycle changes

---

## Full RBAC Runtime Evaluation Workflow

```mermaid
flowchart TD
    REQ[Incoming API Request] --> IDENT[Identify Security Principal from token]
    IDENT --> SCOPES[Collect all scope levels for resource]
    SCOPES --> DENYCHK{Any deny assignment at any scope?}
    DENYCHK -- Yes --> DENIED[❌ Access Denied]
    DENYCHK -- No --> ROLECHK{Any role assignment grants requested action?}
    ROLECHK -- Yes --> PLANE{Check correct plane - control or data?}
    ROLECHK -- No --> DENIED2[❌ Access Denied]
    PLANE -- Matches --> GRANTED[✅ Access Granted]
    PLANE -- Mismatch --> DENIED3[❌ Access Denied]
```

---

## Scope Inheritance in Practice

```mermaid
flowchart TD
    MG[Management Group
Role: Security Reader] -->|inherited| SUB[Subscription
Role: Contributor added]
    SUB -->|inherited| RG1[Resource Group A
No extra role]
    SUB -->|inherited| RG2[Resource Group B
Role: Reader overscoped]
    RG1 -->|inherited| R1[Storage Account
Role: Storage Blob Data Reader added]
    RG2 -->|inherited| R2[Key Vault
Role: Key Vault Secrets User added]
```

---

## Role Selection Decision Tree

```mermaid
flowchart TD
    START[What does the identity need to do?]
    START --> Q1{Manage resources only - no data access?}
    Q1 -- Yes --> Q2{Write or read-only?}
    Q1 -- No --> Q3{Read and write data?}
    Q2 -- Read --> READER[Assign Reader]
    Q2 -- Write --> CONTRIB[Assign Contributor]
    Q3 -- Read only --> Q4{Service-specific data role exists?}
    Q3 -- Read and write --> Q5{Service-specific data role exists?}
    Q4 -- Yes --> DROLE[Assign data-plane reader role e.g. Blob Data Reader]
    Q4 -- No --> CUSTOM[Create custom role with exact DataActions]
    Q5 -- Yes --> DWROLE[Assign data-plane contributor role]
    Q5 -- No --> CUSTOM2[Create custom role with write DataActions]
```

---

## Control Plane vs Data Plane Visual

```mermaid
flowchart LR
    ARM[Azure Resource Manager
Control Plane] --> CP1[Create storage account]
    ARM --> CP2[Delete resource group]
    ARM --> CP3[Modify firewall rules]

    SVC[Azure Service
Data Plane] --> DP1[Read blob content]
    SVC --> DP2[Send message to queue]
    SVC --> DP3[Get secret from Key Vault]

    CP[Contributor role] -.covers.-> ARM
    DR[Storage Blob Data Reader role] -.covers.-> SVC
```

---

## Step-by-Step: Test This in Azure

### Prerequisites
- Azure CLI authenticated
- At least one resource group available

### Step 1 — List built-in role definitions
```bash
# View all built-in roles
az role definition list --query "[?roleType=='BuiltInRole'].{Name:roleName, Description:description}" -o table | head -30

# Inspect a specific role in detail
az role definition list --name "Reader" --query "[0].permissions" -o json
```
**Verify:** Reader role shows `actions: ["*/read"]` and no `notActions` that restrict it further.

### Step 2 — List role assignments at subscription scope
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az role assignment list \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
  -o table
```
**Verify:** You see your own account listed with `Owner` or `Contributor`.

### Step 3 — List role assignments at resource group scope
```bash
RG_NAME=<your-resource-group>

az role assignment list \
  --resource-group $RG_NAME \
  --query "[].{Principal:principalName, Role:roleDefinitionName, Scope:scope}" \
  -o table
```
**Verify:** Assignments scoped to the RG appear, and inherited ones from subscription are not shown here (they still apply).

### Step 4 — Check effective access for a principal
```bash
# Check what actions a user or SP can perform
SP_OBJECT_ID=<object ID of any principal>

az role assignment list \
  --assignee $SP_OBJECT_ID \
  --all \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  -o table
```
**Verify:** All roles across all scopes for that principal are listed.

### Step 5 — Create a custom role definition
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

cat > /tmp/custom-role.json << EOF
{
  "Name": "Custom Storage Reader",
  "Description": "Can list and read blobs only",
  "Actions": [
    "Microsoft.Storage/storageAccounts/read",
    "Microsoft.Storage/storageAccounts/listKeys/action"
  ],
  "NotActions": [],
  "DataActions": [
    "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
  ],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/$SUBSCRIPTION_ID"
  ]
}
EOF

az role definition create --role-definition @/tmp/custom-role.json
```
**Verify:** Custom role created with `roleType: CustomRole`.

### Step 6 — Assign and verify the custom role
```bash
SP_APP_ID=<appId of any SP>

az role assignment create \
  --assignee $SP_APP_ID \
  --role "Custom Storage Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Verify assignment exists
az role assignment list \
  --assignee $SP_APP_ID \
  --query "[].roleDefinitionName" \
  -o table
```
**Verify:** Assignment shows `Custom Storage Reader`.

### Step 7 — Verify scope inheritance (negative test)
```bash
# Grant a role only at resource group level
az role assignment create \
  --assignee $SP_APP_ID \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME"

# Try to list resources in a DIFFERENT resource group — should fail
az login --service-principal --username $SP_APP_ID --password <secret> --tenant <tenant>
az resource list --resource-group <other-rg>
```
**Verify:** Access denied for the other resource group — scope isolation confirmed.

### Step 8 — Clean up
```bash
az login  # back to your account
az role assignment delete --assignee $SP_APP_ID --role "Custom Storage Reader" --scope "/subscriptions/$SUBSCRIPTION_ID"
az role definition delete --name "Custom Storage Reader"
```

### What to Confirm End-to-End
| Check | Expected |
|---|---|
| Reader role shows `*/read` actions | Yes |
| Assignment at subscription scope visible | Yes |
| Custom role creation succeeds | Yes |
| Scope isolation: RG role doesn't grant other RG access | Yes |
| Role deletion removes access immediately | Yes |

---

## Summary
Azure RBAC is the foundation of all resource authorization. Understand principals, role definitions, and scope hierarchy first, then apply least privilege, check both planes, and use deny assignments when needed for strict access control.
