# AKS Ingress Controllers

## The Problem: How Does External Traffic Reach Your Pods?

In Kubernetes, pods are ephemeral and get new IPs constantly. Kubernetes **Services** (ClusterIP, NodePort, LoadBalancer) provide stable networking, but they have limitations when you need to expose **multiple HTTP services** to the outside world.

### Without an Ingress Controller

```mermaid
graph LR
    subgraph "❌ One LoadBalancer per service"
        User["Internet Users"]
        User -->|"app-a.com"| LB1["Azure LB 1<br/>Public IP: 20.1.1.1<br/>💰"]
        User -->|"app-b.com"| LB2["Azure LB 2<br/>Public IP: 20.1.1.2<br/>💰"]
        User -->|"app-c.com"| LB3["Azure LB 3<br/>Public IP: 20.1.1.3<br/>💰"]

        LB1 --> SvcA["Service A"]
        LB2 --> SvcB["Service B"]
        LB3 --> SvcC["Service C"]
    end

    style LB1 fill:#6a040f,color:#fff
    style LB2 fill:#6a040f,color:#fff
    style LB3 fill:#6a040f,color:#fff
```

**Problems:**
- Each `LoadBalancer` service creates a **separate Azure Load Balancer** with its own **public IP** — expensive and wasteful
- No path-based routing (can't do `mysite.com/api` → Service A, `mysite.com/web` → Service B)
- No TLS termination at a central point
- No centralized rate limiting, auth, or header manipulation

### With an Ingress Controller

```mermaid
graph LR
    User["Internet Users"]
    User -->|"app-a.com<br/>app-b.com<br/>mysite.com/api"| LB["Single Azure LB<br/>Public IP: 20.1.1.1<br/>💰 (just one)"]

    LB --> IC["Ingress Controller<br/>(NGINX / AGIC / Traefik)"]

    IC -->|"Host: app-a.com"| SvcA["Service A"]
    IC -->|"Host: app-b.com"| SvcB["Service B"]
    IC -->|"Path: /api"| SvcC["Service C"]

    style LB fill:#2d6a4f,color:#fff
    style IC fill:#7b2d8e,color:#fff
    style SvcA fill:#1b4965,color:#fff
    style SvcB fill:#1b4965,color:#fff
    style SvcC fill:#1b4965,color:#fff
```

**One Load Balancer, one IP, multiple services** — the Ingress Controller handles routing based on hostname, path, headers, etc.

---

## What is an Ingress Controller?

An Ingress Controller is a **reverse proxy** running inside your AKS cluster that:

1. **Watches** Kubernetes `Ingress` resources for routing rules
2. **Configures** itself automatically based on those rules
3. **Routes** incoming HTTP/HTTPS traffic to the correct backend services

```mermaid
graph TB
    subgraph "Kubernetes API"
        ING["Ingress Resource<br/>(YAML rules)"]
    end

    subgraph "AKS Cluster"
        IC["Ingress Controller Pod<br/>(watches Ingress resources)"]
        SvcA["Service A<br/>ClusterIP"]
        SvcB["Service B<br/>ClusterIP"]
        PodA1["Pod A-1"]
        PodA2["Pod A-2"]
        PodB1["Pod B-1"]
    end

    ING -.->|"controller reads<br/>and auto-configures"| IC
    IC -->|"route: /app-a"| SvcA
    IC -->|"route: /app-b"| SvcB
    SvcA --> PodA1
    SvcA --> PodA2
    SvcB --> PodB1

    style ING fill:#e0a800,color:#000
    style IC fill:#7b2d8e,color:#fff
    style SvcA fill:#1b4965,color:#fff
    style SvcB fill:#1b4965,color:#fff
```

---

## Why is it Needed?

| Need | Without Ingress | With Ingress Controller |
|------|----------------|------------------------|
| **Cost** | 1 Azure LB per service ($$$) | 1 Azure LB for all services ($) |
| **Routing** | IP-based only | Host-based, path-based, header-based |
| **TLS** | Configure per service | Centralized TLS termination |
| **Rate limiting** | Custom code per service | Built-in / annotation-based |
| **Auth** | Each service handles it | Centralized (OAuth, basic auth) |
| **Observability** | Scattered metrics | Centralized access logs & metrics |
| **Canary / A-B testing** | Not possible | Traffic splitting by weight |

---

## Ingress Controller Options on AKS

```mermaid
graph TB
    IC["Which Ingress Controller?"]

    IC --> NGINX["NGINX Ingress Controller"]
    IC --> AGIC["Application Gateway<br/>Ingress Controller (AGIC)"]
    IC --> Traefik["Traefik"]
    IC --> Contour["Contour (Envoy-based)"]

    NGINX --> N1["✅ Most popular, community-driven<br/>✅ Rich annotation support<br/>✅ Runs inside the cluster<br/>⚠️ You manage scaling & HA"]
    AGIC --> A1["✅ Azure-native (L7 LB)<br/>✅ WAF support built-in<br/>✅ Managed by Azure<br/>⚠️ Azure-specific, not portable"]
    Traefik --> T1["✅ Auto cert management<br/>✅ Dashboard built-in<br/>✅ Middleware chains"]
    Contour --> C1["✅ Envoy-based, high perf<br/>✅ HTTPProxy CRD<br/>✅ Multi-team delegation"]

    style NGINX fill:#2d6a4f,color:#fff
    style AGIC fill:#1b4965,color:#fff
    style Traefik fill:#7b2d8e,color:#fff
    style Contour fill:#6a040f,color:#fff
```

| Controller | Type | Managed by | Best For |
|-----------|------|-----------|----------|
| **NGINX Ingress** | In-cluster reverse proxy | You (Helm chart) | General purpose, most flexibility |
| **AGIC** | Azure App Gateway (external) | Azure | WAF, Azure-native, enterprise |
| **Traefik** | In-cluster reverse proxy | You (Helm chart) | Auto TLS, middleware, dashboard |
| **Contour** | In-cluster (Envoy) | You (Helm chart) | High perf, multi-team delegation |

---

## How to Set Up: NGINX Ingress Controller (Most Common)

### Step 1: Install NGINX Ingress Controller via Helm

```bash
# Add the ingress-nginx repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install into its own namespace
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

```bash
# Verify — wait for the EXTERNAL-IP to be assigned
kubectl get svc -n ingress-nginx
# NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
# ingress-nginx-controller   LoadBalancer   10.0.45.100    20.1.1.1       80:31080/TCP,443:31443/TCP
```

### Step 2: Deploy Your Application Services

```yaml
# app-a.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
        - name: app-a
          image: myregistry.azurecr.io/app-a:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-a-svc
spec:
  type: ClusterIP          # No need for LoadBalancer — Ingress handles external access
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: app-a
```

### Step 3: Create Ingress Resources (Routing Rules)

#### Host-Based Routing

Route traffic based on the hostname in the request:

```yaml
# ingress-host-based.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app-a.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-a-svc
                port:
                  number: 80
    - host: app-b.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-b-svc
                port:
                  number: 80
```

```mermaid
graph LR
    User["User"]
    User -->|"app-a.mycompany.com"| IC["NGINX Ingress<br/>Controller"]
    User -->|"app-b.mycompany.com"| IC

    IC -->|"Host match:<br/>app-a.mycompany.com"| A["app-a-svc"]
    IC -->|"Host match:<br/>app-b.mycompany.com"| B["app-b-svc"]

    style IC fill:#7b2d8e,color:#fff
    style A fill:#1b4965,color:#fff
    style B fill:#2d6a4f,color:#fff
```

#### Path-Based Routing

Route traffic based on the URL path:

```yaml
# ingress-path-based.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress-paths
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - host: mysite.com
      http:
        paths:
          - path: /api/(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /web/(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: web-svc
                port:
                  number: 80
          - path: /(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: default-svc
                port:
                  number: 80
```

```mermaid
graph LR
    User["User"]
    User -->|"mysite.com/api/users"| IC["NGINX Ingress"]
    User -->|"mysite.com/web/home"| IC
    User -->|"mysite.com/anything"| IC

    IC -->|"/api/*"| API["api-svc"]
    IC -->|"/web/*"| WEB["web-svc"]
    IC -->|"/*  (fallback)"| DEF["default-svc"]

    style IC fill:#7b2d8e,color:#fff
    style API fill:#1b4965,color:#fff
    style WEB fill:#2d6a4f,color:#fff
    style DEF fill:#555,color:#fff
```

---

## TLS Termination

The Ingress Controller handles HTTPS so your backend services don't need to.

### Option A: TLS with a Kubernetes Secret

```bash
# Create a TLS secret from your certificate
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app-a.mycompany.com
      secretName: my-tls-secret
  rules:
    - host: app-a.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-a-svc
                port:
                  number: 80
```

### Option B: Automatic TLS with cert-manager (Let's Encrypt)

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

```yaml
# cluster-issuer.yaml — Let's Encrypt production issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

```yaml
# ingress with automatic TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"     # ← auto-generates cert
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app-a.mycompany.com
      secretName: app-a-tls                                 # ← cert-manager creates this
  rules:
    - host: app-a.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-a-svc
                port:
                  number: 80
```

```mermaid
sequenceDiagram
    participant User
    participant Ingress as NGINX Ingress
    participant CM as cert-manager
    participant LE as Let's Encrypt
    participant Svc as Backend Service

    Note over CM,LE: One-time certificate provisioning
    CM->>LE: 1. Request certificate for app-a.mycompany.com
    LE->>CM: 2. HTTP-01 challenge
    CM->>Ingress: 3. Solve challenge (temporary path)
    LE->>CM: 4. Certificate issued
    CM->>Ingress: 5. Store cert as K8s Secret

    Note over User,Svc: Every request
    User->>Ingress: 6. HTTPS request (app-a.mycompany.com)
    Ingress->>Ingress: 7. TLS termination (using cert)
    Ingress->>Svc: 8. Forward as HTTP (plain)
    Svc->>Ingress: 9. Response
    Ingress->>User: 10. HTTPS response
```

---

## Internal Ingress (Private / No Public IP)

For services that should only be reachable within the VNet (not from the internet):

```bash
helm install ingress-nginx-internal ingress-nginx/ingress-nginx \
  --namespace ingress-internal \
  --create-namespace \
  --set controller.ingressClassResource.name=nginx-internal \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx-internal \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-ingress
spec:
  ingressClassName: nginx-internal      # ← uses the internal ingress class
  rules:
    - host: api.internal.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: internal-api-svc
                port:
                  number: 80
```

> **Use case:** Cross-cluster communication (from our [inter-service-communication doc](inter-service-communication.md)), internal APIs, admin dashboards.

---

## AGIC (Application Gateway Ingress Controller) — Azure-Native Option

Instead of running NGINX inside the cluster, AGIC uses **Azure Application Gateway** (a managed L7 load balancer) as the ingress.

```mermaid
graph LR
    User["Internet Users"]
    User --> AG["Azure Application<br/>Gateway<br/>(managed L7 LB)"]

    AG -->|"reads Ingress<br/>resources via<br/>AGIC pod"| AGIC["AGIC Pod<br/>(in AKS cluster)"]

    AG -->|"/api"| PodA["Pod A"]
    AG -->|"/web"| PodB["Pod B"]

    subgraph "AKS Cluster"
        AGIC
        PodA
        PodB
    end

    style AG fill:#1b4965,color:#fff
    style AGIC fill:#e0a800,color:#000
    style PodA fill:#2d6a4f,color:#fff
    style PodB fill:#2d6a4f,color:#fff
```

**Key difference:** With NGINX, the reverse proxy runs **inside** the cluster. With AGIC, the reverse proxy is **outside** (Azure App Gateway) — the AGIC pod just syncs Ingress rules to the App Gateway config.

```bash
# Enable AGIC add-on on existing AKS cluster
az aks enable-addons \
  --resource-group myRG \
  --name myAKS \
  --addons ingress-appgw \
  --appgw-subnet-cidr "10.225.0.0/16" \
  --appgw-name myAppGateway
```

### NGINX vs AGIC

| | NGINX Ingress | AGIC (App Gateway) |
|---|---|---|
| **Runs** | Inside cluster (pods) | Outside cluster (Azure PaaS) |
| **Scaling** | You manage (HPA) | Azure auto-scales |
| **WAF** | Requires separate setup | Built-in (WAF v2) |
| **Cost** | Pod resources only | App Gateway SKU pricing |
| **Portability** | Works on any K8s | Azure-only |
| **Annotations** | NGINX-specific | App Gateway-specific |
| **SSL offloading** | Yes | Yes |
| **Best for** | Multi-cloud, flexibility | Azure-native, enterprise WAF |

---

## Common NGINX Ingress Annotations

```yaml
metadata:
  annotations:
    # Rewrite the URL path
    nginx.ingress.kubernetes.io/rewrite-target: /

    # Rate limiting (5 req/sec per IP)
    nginx.ingress.kubernetes.io/limit-rps: "5"

    # Request body size limit
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Timeout settings
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Force HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mysite.com"

    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret

    # Canary deployment (send 10% traffic to canary)
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

---

## Complete Traffic Flow (End-to-End)

```mermaid
graph TB
    User["User Browser<br/>app-a.mycompany.com"]

    User -->|"1. DNS resolves to<br/>20.1.1.1"| DNS["Azure DNS /<br/>External DNS"]
    DNS -->|"2. HTTPS request"| ALB["Azure Load Balancer<br/>20.1.1.1:443"]
    ALB -->|"3. Forwards to<br/>NodePort"| IC["NGINX Ingress<br/>Controller Pod"]
    IC -->|"4. TLS termination<br/>(decrypt HTTPS)"| IC
    IC -->|"5. Match host:<br/>app-a.mycompany.com<br/>Match path: /"| SVC["app-a-svc<br/>(ClusterIP)"]
    SVC -->|"6. Load balance<br/>to healthy pod"| POD1["Pod A-1"]
    SVC -->|"6. Load balance<br/>to healthy pod"| POD2["Pod A-2"]

    style User fill:#555,color:#fff
    style DNS fill:#e0a800,color:#000
    style ALB fill:#6a040f,color:#fff
    style IC fill:#7b2d8e,color:#fff
    style SVC fill:#1b4965,color:#fff
    style POD1 fill:#2d6a4f,color:#fff
    style POD2 fill:#2d6a4f,color:#fff
```

1. **User** makes a request to `app-a.mycompany.com`
2. **DNS** resolves to the Azure Load Balancer's public IP
3. **Azure LB** forwards traffic to the Ingress Controller pod (via NodePort)
4. **Ingress Controller** terminates TLS (if configured)
5. **Ingress Controller** matches the host/path rules to find the right backend service
6. **ClusterIP Service** load-balances across healthy pods

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| 404 Not Found | No matching Ingress rule for host/path | Check `kubectl get ingress` — verify host and path |
| 502 Bad Gateway | Backend service/pod is down | Check `kubectl get pods` and service endpoints |
| 503 Service Unavailable | No healthy backends | Check pod readiness probes |
| Ingress has no `ADDRESS` | LB not provisioned | Check `kubectl get svc -n ingress-nginx` for pending LB |
| TLS not working | Missing or wrong secret | Verify `kubectl get secret <tls-secret>` exists |
| Ingress rules ignored | Wrong `ingressClassName` | Ensure it matches your controller's class |

```bash
# Debug commands
kubectl get ingress                              # Check ingress rules and addresses
kubectl describe ingress <name>                  # Detailed routing info + events
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx  # Controller logs
kubectl get svc -n ingress-nginx                 # Check LB IP assignment
kubectl get endpoints <service-name>             # Verify service has healthy backends
```
