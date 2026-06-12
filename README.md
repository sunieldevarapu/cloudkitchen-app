# CloudKitchen

[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](.github/workflows)
[![Go](https://img.shields.io/badge/Go-1.22-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)](https://react.dev)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes&logoColor=white)](https://aws.amazon.com/eks/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

> A cloud-native, event-driven **food-delivery platform** built as 8 Go
> microservices plus a React frontend, deployed to **AWS EKS** via **GitOps**
> with a full observability and security baseline.

CloudKitchen is a portfolio-grade reference platform demonstrating microservice
design, async messaging, infrastructure-as-code, GitOps delivery, and
production-style monitoring/logging/security.

## What it is

- **8 backend microservices** (`auth`, `user`, `restaurant`, `menu`, `order`,
  `payment`, `delivery`, `notification`) written in Go, each listening on
  `:8080` and exposing `/metrics`, `/healthz`, `/readyz` with structured JSON logs.
- **React frontend** SPA served by nginx.
- **Sync** comms over REST, **async** comms over NATS JetStream events.
- Backed by **PostgreSQL**, **Redis**, and **NATS (JetStream)**.
- Shipped to **EKS** (`us-east-1`) through **GitHub Actions -> ECR -> ArgoCD**.

## Architecture

```mermaid
flowchart TB
    user([Browser / Mobile])

    subgraph edge[Edge]
      traefik[Traefik Ingress + TLS]
    end

    fe[React Frontend]

    subgraph svc[cloudkitchen namespace - Go services :8080]
      auth[auth]
      usr[user]
      rest[restaurant]
      menu[menu]
      order[order]
      pay[payment]
      del[delivery]
      notif[notification]
    end

    subgraph data[Backing services]
      pg[(PostgreSQL)]
      redis[(Redis)]
      mq{{"NATS (JetStream)"}}
    end

    user --> traefik --> fe
    traefik --> auth & usr & rest & menu & order & pay & del & notif

    auth & usr & rest & menu & order & pay & del & notif --> pg
    auth & order --> redis
    auth & usr & rest & menu & order & pay & del & notif <--> mq
```

- **Frontend**: React SPA served by nginx; talks to the services through Traefik.
- **8 Go microservices** listen on `:8080`, each exposing `/metrics`, `/healthz`, `/readyz`, and structured JSON logs to stdout.
- **Sync** communication is REST over HTTP. **Async** communication is event-driven over **NATS JetStream**.
- **PostgreSQL** is the system of record; **Redis** handles sessions/caching; **NATS (JetStream)** is the event bus.

See [`docs/architecture/PHASE-1.md`](docs/architecture/PHASE-1.md) for the full design — including the CI/CD pipeline diagram, GitOps flow, event catalog, and observability/security baseline.

## Tech stack

| Layer            | Technology |
|------------------|------------|
| Backend services | Go 1.22 (HTTP REST, Prometheus client, structured JSON logging) |
| Frontend         | React 18 + Vite, served by nginx |
| Data store       | PostgreSQL 16 |
| Cache / sessions | Redis 7 |
| Messaging        | NATS 2.10 + JetStream (event bus) |
| Containers       | Docker (per-service Dockerfiles) |
| Orchestration    | Kubernetes (AWS EKS) |
| Ingress / TLS    | Traefik + cert-manager (Let's Encrypt) |
| GitOps           | ArgoCD |
| Packaging        | Helm |
| IaC              | Terraform (VPC, EKS, ECR, IAM/IRSA) — `us-east-1` |
| CI/CD            | GitHub Actions (matrix build, Trivy gate, ECR push, values bump) |
| Metrics          | Prometheus (kube-prometheus-stack) + Grafana |
| Logging          | Loki + Promtail |
| Security scan    | Trivy (CI gate + optional trivy-operator) |

## Repository layout (flat monorepo)

```
cloudkitchen-app/
├── auth/            # Go service — authentication & JWT
├── user/            # Go service — user profiles
├── restaurant/      # Go service — restaurant management
├── menu/            # Go service — menu items
├── order/           # Go service — order lifecycle
├── payment/         # Go service — payments
├── delivery/        # Go service — delivery tracking
├── notification/    # Go service — notifications
├── frontend/        # React SPA
├── helm/            # Helm chart(s)
├── terraform/       # AWS infra (VPC, EKS, ECR, IAM/IRSA)
├── argocd/          # ArgoCD Applications (App-of-Apps)
├── monitoring/      # Prometheus + Grafana values & dashboards
├── logging/         # Loki + Promtail values
├── security/        # cert-manager, network policies, PSS, trivy, secrets
├── docker/          # docker-compose local stack
├── scripts/         # build / seed / port-forward / kubeconfig helpers
├── docs/            # architecture & docs index
├── .github/         # GitHub Actions workflows
└── README.md
```

## Quickstart (local, docker-compose)

```sh
# from the repo root
docker compose -f docker/docker-compose.yml up --build
```

Then:

| Component    | URL                     |
|--------------|-------------------------|
| Frontend     | http://localhost:3000   |
| auth         | http://localhost:8081   |
| user         | http://localhost:8082   |
| restaurant   | http://localhost:8083   |
| menu         | http://localhost:8084   |
| order        | http://localhost:8085   |
| payment      | http://localhost:8086   |
| delivery     | http://localhost:8087   |
| notification | http://localhost:8088   |
| NATS monitor | http://localhost:8222   |

Seed demo data (users per role, a restaurant, menu items, an order):

```sh
./scripts/seed.sh
```

Full local instructions: [`docker/README.md`](docker/README.md).

## Deployment overview

```mermaid
flowchart LR
    tf[Terraform] --> eks[(EKS us-east-1)]
    push[git push] --> gha[GitHub Actions]
    gha --> trivy[Trivy] --> ecr[(ECR)]
    ecr --> bump[bump helm/cloudkitchen/values.yaml] --> commit[commit]
    commit --> argo[ArgoCD auto-sync] --> eks
```

1. **Provision** infrastructure with **Terraform** (VPC, EKS, ECR, IAM/IRSA) in
   `us-east-1`.
2. **Bootstrap** the cluster: namespaces, Traefik, cert-manager, ArgoCD,
   kube-prometheus-stack, Loki/Promtail.
3. **CI** (GitHub Actions): matrix build per service -> **Trivy** scan ->
   push to **ECR** -> bump image tags in `helm/cloudkitchen/values.yaml` ->
   commit. **No `helm upgrade` in CI.**
4. **GitOps**: **ArgoCD** detects the committed change and **auto-syncs** the
   Helm release to EKS.

See the area guides:
[monitoring](monitoring/README.md) ·
[logging](logging/README.md) ·
[security](security/README.md) ·
[docs index](docs/README.md).

## License

MIT — see `LICENSE` (placeholder).
