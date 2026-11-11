# Kubernetes Network Security Labs with Calico NetworkPolicy

**Author:** Jay Desai
**Project:** Kubernetes Network Security and Three-Tier Application Security
**Environment:** Windows 11, WSL2 Ubuntu, Docker, Kind, Kubernetes and Calico

---

## 1. Project Overview

This repository contains two practical Kubernetes networking and security labs.

### Lab 1: Kubernetes NetworkPolicy Fundamentals

A foundational exercise demonstrating:
- Kubernetes NetworkPolicy concepts
- Default-deny ingress and egress
- Pod selectors
- DNS traffic control
- Frontend-to-backend communication
- Testing authorized and unauthorized traffic

### Lab 2: Three-Tier Application Security with Calico

An advanced hands-on project demonstrating network segmentation and traffic restrictions across three application tiers:

- Frontend: NGINX
- Backend: HTTP Echo API
- Database: PostgreSQL

Calico CNI is used to enforce Kubernetes NetworkPolicies.

The main objective is to implement a zero-trust-style network model: deny traffic by default and allow only explicitly authorized communication.

## 2. Repository Structure

```text
NetworkPolicy/
│
├── README.md
├── .gitignore
│
├── kind/
│   ├── jay-netpol-kind.yaml
│   └── calico-custom-resources.yaml
│
├── netpol-lab/
│   ├── frontend-backend.yaml
│   └── policies/
│       ├── default-deny-backend.yaml
│       ├── allow-frontend.yaml
│       ├── allow-frontend-dns.yaml
│       └── allow-frontend-backend-egress.yaml
│
└── three-tier-lab/
    ├── frontend.yaml
    ├── backend.yaml
    ├── postgres.yaml
    └── policies/
        ├── default-deny-frontend-tier.yaml
        ├── allow-http-to-frontend.yaml
        ├── default-deny-frontend-egress.yaml
        ├── allow-frontend-dns-egress.yaml
        ├── allow-frontend-to-backend-egress.yaml
        ├── default-deny-backend-tier.yaml
        ├── allow-frontend-to-backend-tier.yaml
        ├── default-deny-backend-egress.yaml
        ├── allow-backend-dns-egress.yaml
        ├── allow-backend-db-egress.yaml
        ├── default-deny-db.yaml
        ├── allow-backend-to-db.yaml
        └── default-deny-db-egress.yaml
```

## 3. Technology Stack

| Technology | Version / Purpose |
|---|---|
| Kubernetes | 1.37.0 |
| Kind | 0.33.0 |
| Calico | 3.33.0 |
| Docker Engine | Container runtime |
| WSL2 Ubuntu | 24.04 |
| NGINX | 1.27 |
| PostgreSQL | 16 Alpine |
| BusyBox | 1.36 |
| HTTP Echo | 1.0.0 |

## 4. Cluster Information

| Property | Value |
|---|---|
| Kind cluster | jay-netpol |
| Kubernetes context | kind-jay-netpol |
| Node | jay-netpol-control-plane |
| CNI | Calico |
| Main application namespace | three-tier-lab |

The separate original `jay-k8s` Kubernetes cluster was preserved and not modified by this lab.

---

# Lab 1: Kubernetes NetworkPolicy Fundamentals

## 5. Architecture

The original lab contains a frontend client and an NGINX backend.

```text
Frontend Pod
     |
     | TCP 80
     v
Backend Service
     |
     v
Backend NGINX Pod
```

Namespace: `netpol-lab`

## 6. NetworkPolicies

| Policy | Purpose |
|---|---|
| default-deny-backend | Denies unauthorized backend ingress |
| allow-frontend | Allows frontend-to-backend traffic |
| allow-frontend-dns | Allows DNS queries |
| allow-frontend-backend-egress | Allows frontend egress to backend |

## 7. Deploy Lab 1

Create the namespace:

```bash
kubectl create namespace netpol-lab \
  --context kind-jay-netpol
```

Deploy frontend, backend and Service:

```bash
kubectl apply -f netpol-lab/frontend-backend.yaml \
  --context kind-jay-netpol
```

Apply NetworkPolicies:

```bash
kubectl apply -f netpol-lab/policies/ \
  --context kind-jay-netpol
```

Verify:

```bash
kubectl get pods,services,networkpolicy \
  -n netpol-lab \
  --context kind-jay-netpol
```

---

# Lab 2: Three-Tier Application Security with Calico

## 8. Three-Tier Architecture

```text
                 Trusted Ingress Client
                          |
                          | TCP 80
                          v
              +------------------------+
              |    FRONTEND TIER       |
              |    NGINX (2 replicas)  |
              +------------------------+
                          |
                          | TCP 5678
                          v
              +------------------------+
              |     BACKEND TIER       |
              | HTTP Echo API (2 pods) |
              +------------------------+
                          |
                          | TCP 5432
                          v
              +------------------------+
              |     DATABASE TIER     |
              | PostgreSQL 16 (1 pod) |
              +------------------------+
```

Namespace: `three-tier-lab`

## 9. Application Components

| Tier | Deployment | Replicas | Container Port |
|---|---|---:|---:|
| Frontend | frontend | 2 | 80 |
| Backend | backend | 2 | 5678 |
| Database | postgres | 1 | 5432 |

### Kubernetes Services

| Service | Type | Service Port | Target Port |
|---|---|---:|---:|
| frontend-service | ClusterIP | 80 | 80 |
| backend-service | ClusterIP | 80 | 5678 |
| postgres-service | ClusterIP | 5432 | 5432 |

The backend Service listens on port 80 and forwards traffic to container port 5678.

## 10. Create the Kind Cluster

The separate cluster is configured without the default CNI so Calico can be installed.

```bash
kind create cluster \
  --config kind/jay-netpol-kind.yaml \
  --image kindest/node:v1.37.0 \
  --wait 5m
```

Verify:

```bash
kubectl get nodes --context kind-jay-netpol
```

The node may initially show `NotReady` until Calico is installed.

## 11. Install Calico

Install Calico CRDs:

```bash
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml \
--context kind-jay-netpol
```

Install the Tigera Operator:

```bash
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml \
--context kind-jay-netpol
```

Wait for the operator:

```bash
kubectl get pods -n tigera-operator \
  --context kind-jay-netpol
```

Install the Calico configuration:

```bash
kubectl apply -f kind/calico-custom-resources.yaml \
  --context kind-jay-netpol
```

Verify Calico:

```bash
kubectl get pods -n calico-system \
  --context kind-jay-netpol

kubectl get tigerastatus \
  --context kind-jay-netpol

kubectl get nodes \
  --context kind-jay-netpol
```

The Calico components should become Ready and the node should become Ready.

## 12. Deploy the Three-Tier Application

Create the namespace:

```bash
kubectl create namespace three-tier-lab \
  --context kind-jay-netpol
```

### Create PostgreSQL credentials

The PostgreSQL Deployment reads credentials from a Kubernetes Secret.

Create a lab-only Secret:

```bash
kubectl create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=appuser \
  --from-literal=POSTGRES_PASSWORD='REPLACE_WITH_A_LAB_ONLY_PASSWORD' \
  --from-literal=POSTGRES_DB=appdb \
  -n three-tier-lab \
  --context kind-jay-netpol
```

Use a suitable lab-only password. Never commit actual credentials to GitHub.

### Deploy the application

```bash
kubectl apply -f three-tier-lab/postgres.yaml \
  --context kind-jay-netpol

kubectl apply -f three-tier-lab/backend.yaml \
  --context kind-jay-netpol

kubectl apply -f three-tier-lab/frontend.yaml \
  --context kind-jay-netpol
```

Verify:

```bash
kubectl get pods,services \
  -n three-tier-lab \
  --context kind-jay-netpol
```

## 13. Implement NetworkPolicies

Apply all 13 policies:

```bash
kubectl apply -f three-tier-lab/policies/ \
  --context kind-jay-netpol
```

### Frontend policies

1. `default-deny-frontend-tier` — Denies unauthorized frontend ingress.
2. `allow-http-to-frontend` — Allows HTTP from the trusted-ingress test selector.
3. `default-deny-frontend-egress` — Restricts frontend outbound traffic.
4. `allow-frontend-dns-egress` — Allows DNS over UDP/TCP 53.
5. `allow-frontend-to-backend-egress` — Allows backend access over TCP 5678.

### Backend policies

6. `default-deny-backend-tier` — Denies unauthorized backend ingress.
7. `allow-frontend-to-backend-tier` — Allows frontend-to-backend traffic.
8. `default-deny-backend-egress` — Restricts backend outbound traffic.
9. `allow-backend-dns-egress` — Allows DNS over UDP/TCP 53.
10. `allow-backend-db-egress` — Allows PostgreSQL access over TCP 5432.

### Database policies

11. `default-deny-db` — Denies unauthorized database ingress.
12. `allow-backend-to-db` — Allows backend-to-database traffic over TCP 5432.
13. `default-deny-db-egress` — Restricts database outbound traffic.

Verify:

```bash
kubectl get networkpolicy \
  -n three-tier-lab \
  --context kind-jay-netpol
```

Expected result: 13 NetworkPolicies.

## 14. Network Traffic Rules

| Source | Destination | Port | Result |
|---|---|---:|---|
| Trusted ingress test client | Frontend | 80 | Allowed |
| Frontend | CoreDNS | 53 UDP/TCP | Allowed |
| Frontend | Backend | 5678 | Allowed |
| Backend | CoreDNS | 53 UDP/TCP | Allowed |
| Backend | PostgreSQL | 5432 | Allowed |
| Untrusted client | Backend | 5678 | Blocked |
| Unauthorized clients | PostgreSQL | 5432 | Blocked |

## 15. Practical Tests Performed

### Test 1: DNS resolution

```bash
kubectl exec <frontend-pod> \
  -n three-tier-lab \
  --context kind-jay-netpol \
  -- getent hosts backend-service
```

Expected result: backend Service DNS resolves successfully.

### Test 2: Frontend-to-backend connectivity

```bash
kubectl exec <frontend-test-pod> \
  -n three-tier-lab \
  --context kind-jay-netpol \
  -- wget -T 5 -qO- http://backend-service
```

Expected response:

```text
Hello from Jay Three-Tier Backend API
```

### Test 3: Unauthorized backend access

```bash
kubectl exec <untrusted-test-pod> \
  -n three-tier-lab \
  --context kind-jay-netpol \
  -- wget -T 5 -qO- http://backend-service
```

Expected result: connection timeout.

### Test 4: Backend-to-PostgreSQL connectivity

```bash
kubectl exec <backend-test-pod> \
  -n three-tier-lab \
  --context kind-jay-netpol \
  -- pg_isready -h postgres-service \
  -p 5432 -U appuser -d appdb
```

Expected response:

```text
postgres-service:5432 - accepting connections
```

### Test 5: Verify NetworkPolicies

```bash
kubectl get networkpolicy \
  -n three-tier-lab \
  --context kind-jay-netpol
```

Expected result: all 13 policies exist.

Note: Replace the example Pod placeholders with the actual Pod names. Temporary test Pods used during these experiments were removed after testing.

---

## 16. Useful Troubleshooting Commands

```bash
# Check cluster nodes
kubectl get nodes --context kind-jay-netpol

# Check all application Pods
kubectl get pods -n three-tier-lab --context kind-jay-netpol

# Check Services
kubectl get services -n three-tier-lab --context kind-jay-netpol

# Check NetworkPolicies
kubectl get networkpolicy -n three-tier-lab --context kind-jay-netpol

# Describe a policy
kubectl describe networkpolicy default-deny-db \
  -n three-tier-lab --context kind-jay-netpol

# Inspect Calico components
kubectl get pods -n calico-system --context kind-jay-netpol

# Check Calico status
kubectl get tigerastatus --context kind-jay-netpol

# Check recent namespace events
kubectl get events -n three-tier-lab \
  --sort-by=.metadata.creationTimestamp \
  --context kind-jay-netpol
```

## 17. Important Kubernetes Concepts Learned

- Container Network Interface (CNI)
- Calico installation and configuration
- Kubernetes NetworkPolicy
- Default-deny ingress and egress
- Pod selectors and namespace selectors
- DNS traffic control
- Service-to-Pod communication
- TCP port restrictions
- Network segmentation
- Stateful connection response traffic
- Kubernetes Services and EndpointSlices
- Network troubleshooting

## 18. Production Improvements

This is an educational security lab, not a production-ready application.

Recommended future improvements:

- Configure NGINX as a reverse proxy to the backend API.
- Replace the trusted-ingress test selector with actual ingress-controller Pod and namespace selectors.
- Add PersistentVolumeClaims for PostgreSQL.
- Use a secure external secret-management solution.
- Configure TLS and application authentication.
- Add RBAC and Pod Security controls.
- Configure monitoring and logging.
- Add resource requests and limits.
- Add database backup and recovery.
- Add CI/CD deployment automation.

## 19. Important Lab Notes

- PostgreSQL currently uses ephemeral storage; data may be lost when its Pod is replaced.
- Database credentials are dummy lab-only credentials.
- The frontend serves the stock NGINX page; frontend-to-backend connectivity was tested using a temporary BusyBox client.
- The backend is an HTTP Echo demonstration service. PostgreSQL connectivity was tested separately using a temporary PostgreSQL client Pod labelled as a backend workload.
- The `trusted-ingress` label represents a test client, not a real ingress controller.
- All temporary test Pods were removed after testing.
- NetworkPolicy enforcement depends on a CNI that supports it. This is why the Calico lab uses a separate cluster.

## 20. Final Project Results

- Two Kubernetes NetworkPolicy labs documented.
- Separate Kind cluster created for Calico.
- Calico CNI installed and operational.
- Three-tier architecture deployed.
- 13 NetworkPolicies implemented in the advanced lab.
- Default-deny ingress and egress configured across all three tiers.
- Authorized communication tested successfully.
- Unauthorized backend access blocked.
- All five application Pods healthy at final verification.
- Original Kubernetes cluster preserved.

---

**Project:** Kubernetes Network Security Labs with Calico NetworkPolicy
**Author:** Jay Desai
