# USTBitesHub

> **Cloud-Native Food Delivery Platform** — A production-grade microservices system built on Kubernetes with full GitOps CI/CD, observability, and security layers.

[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-blue?logo=github-actions)](https://github.com/USTBitesHub)
[![GitOps](https://img.shields.io/badge/GitOps-ArgoCD-orange?logo=argo)](https://argo-cd.readthedocs.io)
[![K8s](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?logo=kubernetes)](https://kubernetes.io)
[![Helm](https://img.shields.io/badge/Packaging-Helm-0F1689?logo=helm)](https://helm.sh)

---

## Overview

USTBitesHub is a campus food delivery platform for university students. Users browse restaurants, place orders, and track delivery — all served by a distributed backend of 9 microservices running on Kubernetes.

The project is designed as a **production-readiness reference** — every architectural decision reflects a real-world tradeoff and is documented. It demonstrates end-to-end DevOps: secure multi-stage CI pipelines, GitOps deployment with ArgoCD, namespace-isolated Kubernetes workloads, and a full observability stack.

---

## Application Screenshots

### User Interface

<div align="center">
<table>
<tr>
<td align="center">
<img src="../images/application-1.png" width="300" alt="Application Screenshot 1" />
<br/><sub>Browse & Discover Restaurants</sub>
</td>
<td align="center">
<img src="../images/application-2.png" width="300" alt="Application Screenshot 2" />
<br/><sub>Place Your Order</sub>
</td>
</tr>
<tr>
<td align="center">
<img src="../images/application-3.png" width="300" alt="Application Screenshot 3" />
<br/><sub>Order Management</sub>
</td>
<td align="center">
<img src="../images/application-ai.png" width="300" alt="AI Agent Assistant" />
<br/><sub>AI-Powered Chat Assistant</sub>
</td>
</tr>
<tr>
<td align="center" colspan="2">
<img src="../images/pay-1.png" width="300" alt="Payment Gateway" />
<br/><sub>Secure Payment Processing</sub>
</td>
</tr>
</table>
</div>

---

## Architecture

```
Browser (HTTPS)
    │
    ▼
KGateway (Envoy) ─── kgateway namespace
    │
    ├──▶ ustbite-frontend ──── ustbite-frontend namespace
    │        React + Vite + TypeScript
    │
    └──▶ ustbite-backend ───── ustbite-backend namespace
             │
             ├── user-service          FastAPI + PostgreSQL  (auth, JWT)
             ├── restaurant-service    FastAPI + PostgreSQL  (menus, browse)
             ├── cart-service          FastAPI + Redis       (active cart)
             ├── order-service         FastAPI + PostgreSQL  (order saga)
             ├── payment-service       FastAPI + PostgreSQL  (payment processing)
             ├── notification-service  FastAPI + PostgreSQL  (async email/SMS)
             ├── delivery-service      FastAPI + PostgreSQL  (delivery assignment)
             └── ai-agent-service      FastAPI + LLM         (chat + tool calls)
             
             Shared infra:
             ├── RabbitMQ (topic exchange — async event bus)
             └── Redis    (cart cache)

Observability:  monitoring namespace  →  Prometheus + Grafana + Loki
GitOps:         argocd namespace      →  ArgoCD (App-of-Apps)
Auth/SSO:       keycloak namespace    →  Keycloak (OIDC for Headlamp)
```

### Event Flow (Order Placement Saga)

The order placement flow is a **Choreography-based Saga** — no central orchestrator:

```
User → KGateway → order-service
                      │
                      ├── 1. Write order (PENDING) to PostgreSQL        [sync]
                      ├── 2. HTTP POST /payments → payment-service       [sync ← only blocking call]
                      ├── 3. Update order (CONFIRMED/FAILED)             [sync]
                      ├── 4. Return response to user                     [sync]
                      │
                      └── 5. Publish "order.placed" → RabbitMQ          [async ─────────────▶]
                                                           │
                                          ┌────────────────┴──────────────────┐
                                          ▼                                   ▼
                               notification-service               delivery-service
                               (consumes order.placed)            (consumes order.confirmed)
                               sends email/SMS + ACK              assigns agent + ACK
```

> Only the payment call is synchronous — the user needs immediate confirmation.
> Everything after payment is asynchronous via RabbitMQ for resilience and decoupling.

---

## Repository Structure

This project uses a **multi-repo GitHub Organisation** at [github.com/USTBitesHub](https://github.com/USTBitesHub).

| Repository | Language | Purpose |
|---|---|---|
| `ustbite-frontend` | TypeScript | React + Vite SPA |
| `ustbite-user-service` | Python | Auth, registration, JWT issuance |
| `ustbite-restaurant-service` | Python | Restaurant & menu data |
| `ustbite-order-service` | Python | Order placement saga, RabbitMQ publisher |
| `ustbite-payment-service` | Python | Payment processing (PCI-isolated) |
| `ustbite-delivery-service` | Python | Delivery assignment, RabbitMQ consumer |
| `ustbite-notification-service` | Python | Email/SMS, RabbitMQ consumer |
| `ustbite-cart-service` | Python | Active cart management (Redis) |
| `ustbite-ai-agent-service` | Python | LLM chat agent with tool calling |
| `ustbite-workflows` | YAML | **Reusable CI/CD GitHub Actions workflows** |
| `ustbite-helm-charts` | Helm/YAML | **Helm charts + ArgoCD ApplicationSets** |

---

## Tech Stack

### Application
| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, TypeScript, TailwindCSS, shadcn/ui, Zustand, React Query |
| Backend | FastAPI, SQLAlchemy (async), Alembic, Pydantic, PyJWT, httpx |
| Async Messaging | RabbitMQ (topic exchange, durable queues, aio-pika) |
| Cache | Redis |
| Database | PostgreSQL — one instance per service (database-per-service pattern) |
| AI Agent | FastAPI + Qwen 2.5 / OpenAI API + tool-calling |

### Platform & DevOps
| Layer | Technology |
|---|---|
| Container Runtime | Docker |
| Orchestration | Kubernetes + Helm 3 |
| Ingress / API Gateway | KGateway (Envoy Gateway) |
| GitOps | ArgoCD — App-of-Apps + ApplicationSets |
| CI/CD | GitHub Actions (reusable workflows in `ustbite-workflows`) |
| Image Registries | GHCR (develop branch) · DockerHub (main branch) |
| Image Signing | Cosign (main branch only) |
| Secrets | Bitnami Sealed Secrets (AES-GCM encrypted, GitOps-compatible) |
| Policy Engine | Kyverno (admission policies, resource limits enforcement) |
| Observability | Prometheus + Grafana + Loki |
| Cluster UI | Headlamp + Keycloak OIDC |
| OIDC Provider | Keycloak |
| Security Scanning | Gitleaks · SonarQube · Snyk · Trivy |

---

## CI/CD Pipeline

Every repository uses the shared reusable workflows from `ustbite-workflows`.

### On Pull Request
```
Gitleaks (secret scan) → Sonar + Snyk (SAST + SCA) → Docker Build (dry run) → Trivy (CRITICAL scan)
```
All checks must pass before merge. Branch protection rules enforce this on both `develop` and `main`.

### On Push to `develop`
```
Build image → tag: develop-{sha8} → Push to GHCR → Helm Updater commits to values.dev.yaml
                                                              │
                                                              ▼
                                                    ArgoCD polls develop branch
                                                    → syncs Dev K8s cluster (auto)
```

### On Push to `main`
```
Build image → semver tag (service-v1.2.3) → Push to DockerHub → Cosign sign → Prod Gate (manual approval)
                                                                                          │
                                                              ┌───────────────────────────┘
                                                              ▼
                                                    Helm Updater commits to values.prod.yaml
                                                              │
                                                              ▼
                                                    ArgoCD polls main branch
                                                    → syncs Prod K8s cluster (auto, retry 3×, Slack alert on failure)
```

---

## Kubernetes Architecture

### Namespaces

| Namespace | Contents |
|---|---|
| `kgateway` | KGateway (Envoy) — single external entry point |
| `ustbite-frontend` | Frontend deployment, HPA, NetworkPolicy |
| `ustbite-backend` | All 8 backend services, RabbitMQ, Redis |
| `monitoring` | Prometheus, Grafana, Loki |
| `argocd` | ArgoCD controller |
| `keycloak` | Keycloak OIDC provider |

### Security Layers

1. **Network perimeter** — KGateway TLS termination. All services use `ClusterIP` only — no direct external exposure.
2. **Application identity** — JWT validated per-request in order-service and cart-service. User-scoped queries prevent horizontal privilege escalation.
3. **East-west isolation** — `default-deny-all` NetworkPolicy on every namespace. Explicit allow rules only. A compromised pod cannot reach unrelated namespaces.
4. **Secrets at rest** — Bitnami Sealed Secrets. Encrypted `SealedSecret` CRDs committed to Git. The private key never leaves the cluster.

### GitOps with ArgoCD (App-of-Apps)

```
ustbite-helm-charts (Git)
    │
    └── argoapps/prod/
            ├── app-of-apps.yaml          ← Root Application (sync-wave: -10)
            ├── applicationset-prod.yaml  ← Deploys 9 services, retry 3×, Slack alerts
            └── infra-application.yaml    ← Infra (sync-wave: -20): kgateway, netpol, RBAC, sealed-secrets
```

Each ArgoCD Application has `selfHeal: true` — any manual change to a resource is reverted within minutes.
`ignoreDifferences` on `/spec/replicas` allows HPA to scale pods freely without ArgoCD reverting the count.

---

## Branching Strategy

```
feature/xyz  ──PR──▶  develop  ──PR──▶  main
                          │                │
                      Dev cluster      Prod cluster
                      GHCR + sha tag   DockerHub + semver
```

- **`develop`** — integration branch. CI auto-deploys to Dev cluster on merge.
- **`main`** — production branch. Requires manual Prod Gate approval before deploy.
- **Branch protection rules** on both `develop` and `main`:
  - PR review required
  - All CI status checks must pass
  - No direct pushes allowed

---
\

### Environment Variables (order-service example)
```env
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/ustbite_orders_db
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
REDIS_URL=redis://localhost:6379
SERVICE_NAME=ustbite-order-service
SERVICE_PORT=8003
SECRET_KEY=your-jwt-secret
LOG_LEVEL=INFO
```

Each service has its own `.env.example` — copy and fill before running.

---

## Deployment (Kubernetes + ArgoCD)

### Bootstrap (first time)
```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Apply the root App-of-Apps
kubectl apply -f ustbite-helm-charts/argoapps/app-of-apps.yaml

# ArgoCD will now self-manage everything else from Git
```

### After bootstrap, all deployments happen automatically:
- Push to `develop` → ArgoCD syncs Dev cluster within 3 minutes
- Merge to `main` + approve Prod Gate → ArgoCD syncs Prod cluster

### Monitor with Headlamp
Headlamp is the web-based Kubernetes UI. Access it via the cluster LoadBalancer IP.
Log in with your Keycloak SSO credentials (OIDC) — no shared kubeconfig files required.

---

## Observability

| Tool | Purpose | Access |
|---|---|---|
| **Prometheus** | Scrapes `/metrics` from all pods | `monitoring` namespace |
| **Grafana** | Dashboards — CPU, memory, request rate, error rate per service | port-forward or Ingress |
| **Loki** | Log aggregation across all namespaces | Queried via Grafana |

### Monitoring Dashboards

<div align="center">
<table>
<tr>
<td align="center">
<img src="../images/grafana.png" width="350" alt="Grafana Dashboards" />
<br/><sub><strong>Grafana</strong> — Real-time metrics & dashboards</sub>
</td>
<td align="center">
<img src="../images/loki.png" width="350" alt="Loki Log Aggregation" />
<br/><sub><strong>Loki</strong> — Centralized log aggregation</sub>
</td>
</tr>
<tr>
<td align="center">
<img src="../images/headlamp.png" width="350" alt="Headlamp K8s UI" />
<br/><sub><strong>Headlamp</strong> — Kubernetes cluster UI</sub>
</td>
<td align="center">
<img src="../images/oidc.png" width="350" alt="Keycloak OIDC" />
<br/><sub><strong>Keycloak OIDC</strong> — Secure authentication</sub>
</td>
</tr>
</table>
</div>

### Alerting & Notifications

<div align="center">
<img src="../images/slack.png" width="400" alt="Slack Notifications" />
<br/><sub>Slack alerts for production deployments & cluster events</sub>
</div>

---

## Contributing

1. Fork the relevant service repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Open a PR against `develop`
4. All CI checks must pass (Gitleaks, Sonar+Snyk, Docker build, Trivy)
5. Get a PR review approved
6. Merge → auto-deploys to Dev cluster

---

*Built as a learning platform for cloud-native DevOps patterns. Every tradeoff is intentional and documented.*
