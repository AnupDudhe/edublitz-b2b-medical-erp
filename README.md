# EduBlitz Medical B2B ERP System

A production-grade **Medical Domain B2B ERP** for hospitals, distributors, and administrators. The stack is **three Spring Boot microservices**, a **React (Vite)** SPA, and **MongoDB** (Atlas or self-hosted).

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CloudFront CDN                           │
│                    (React Frontend via S3)                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              AWS ALB Ingress Controller (EKS)                   │
└──────┬─────────────────────┬──────────────────────┬────────────┘
       │                     │                      │
┌──────▼──────┐    ┌─────────▼────────┐   ┌────────▼────────┐
│ user-service│    │ product-service  │   │  order-service  │
│  Port: 8081 │    │   Port: 8082     │   │   Port: 8083    │
│             │    │                  │   │                 │
│ Auth / JWT  │    │ Catalog / Stock  │   │ Order lifecycle │
│ Roles/Orgs  │    │ Batches / Reserve│   │ (+ product API) │
└──────┬──────┘    └─────────┬────────┘   └────────┬────────┘
       │                     │                      │
┌──────▼─────────────────────▼──────────────────────▼─────────────┐
│                     MongoDB Atlas (or local)                    │
│   users_db          products_db            orders_db            │
└──────────────────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer        | Technology                                      |
|--------------|-------------------------------------------------|
| Frontend     | React 18 + Vite + TailwindCSS + TanStack Query  |
| Backend      | Spring Boot 3.x (3 microservices)               |
| Database     | MongoDB (Atlas recommended)                     |
| Auth         | JWT (HMAC-SHA256 / HS256), shared secret        |
| Cloud        | AWS (EKS, S3, CloudFront, Route53) — optional   |
| IaC          | Terraform (modular)                             |
| CI/CD        | Jenkins (see `jenkins/`)                        |
| Containers   | Docker + Kubernetes manifests in `k8s/`         |
| API Docs     | Swagger / OpenAPI 3.0 per service               |

## Domain Highlights

- **Catalog**: Active products only appear in hospital/distributor listings; soft-deleted products free their **SKU** for reuse.
- **Inventory**: Stock is tracked per **product + warehouse + batch** (`POST /products/inventory`). **Available** (sellable) = stored quantity minus reserved.
- **Orders**: Hospitals place orders; **distributors** (or admins) **approve** only when enough sellable stock exists — approval calls product-service to **reserve** stock (multi-batch allocation). Distributors only act on orders assigned to their **organization ID**.
- **Admin UI**: Organization **MongoDB IDs** are listed under **Organizations** for integration and user registration.

## Services

| Service         | Port | Responsibilities |
|-----------------|------|------------------|
| user-service    | 8081 | Auth, JWT, users, organizations, audit hooks |
| product-service | 8082 | Products, inventory batches, reserve/release APIs |
| order-service   | 8083 | Orders; calls product-service over HTTP with forwarded JWT |

## Roles

| Role        | Access |
|-------------|--------|
| ADMIN       | Organizations, all products/inventory (scoped APIs), all orders |
| DISTRIBUTOR | Own catalog & stock batches, incoming orders for own org |
| HOSPITAL    | Browse catalog, create/track own org’s orders |

## Project Structure

```
├── frontend/           # React + Vite (HashRouter for static hosting)
├── user-service/
├── product-service/
├── order-service/
├── docker/
├── k8s/
├── terraform/
├── jenkins/
└── docs/               # Deployment & architecture guides
```

## Documentation

| Document | Description |
|----------|-------------|
| [docs/README.md](docs/README.md) | Index of all guides |
| [docs/MANUAL_DEPLOYMENT.md](docs/MANUAL_DEPLOYMENT.md) | Run locally without Docker |
| [docs/DOCKER_DEPLOYMENT.md](docs/DOCKER_DEPLOYMENT.md) | Docker Compose + Atlas |
| [docs/KUBERNETES_DEPLOYMENT.md](docs/KUBERNETES_DEPLOYMENT.md) | EKS + AWS Load Balancer Controller |
| [k8s/README.md](k8s/README.md) | `kubectl apply` order |
| [docs/TERRAFORM_DEPLOYMENT.md](docs/TERRAFORM_DEPLOYMENT.md) | AWS infrastructure |
| [terraform/README.md](terraform/README.md) | Terraform modules |
| [docs/JENKINS_DEPLOYMENT.md](docs/JENKINS_DEPLOYMENT.md) | CI/CD pipelines |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Service boundaries & data flows |

## Quick Start

1. **Local:** [docs/MANUAL_DEPLOYMENT.md](docs/MANUAL_DEPLOYMENT.md) — HashRouter URLs like `http://localhost:5173/#/login`.
2. **Docker Compose:** [docs/DOCKER_DEPLOYMENT.md](docs/DOCKER_DEPLOYMENT.md) — APIs only; Atlas via `docker/.env`; frontend elsewhere (e.g. S3 + CloudFront).
3. **Kubernetes:** [docs/KUBERNETES_DEPLOYMENT.md](docs/KUBERNETES_DEPLOYMENT.md) + [k8s/README.md](k8s/README.md).
4. **Terraform:** [docs/TERRAFORM_DEPLOYMENT.md](docs/TERRAFORM_DEPLOYMENT.md).

## Prerequisites

| Tool | Notes |
|------|--------|
| **JDK 17** | Use for **running** services. Set `JAVA_HOME` to JDK 17 before **`mvn`** if your default JDK is newer (avoids Lombok/compiler issues). |
| Maven 3.9+ | `mvn clean package` per service |
| Node.js 18+ | Frontend |
| MongoDB | Local or Atlas; set **`MONGODB_URI`** (see each `.env.example`) |
| Docker | Optional (Compose) |

## Configuration

- **`application.yml`** defaults use **local MongoDB** (`mongodb://127.0.0.1:27017/...`). Set **`MONGODB_URI`** for Atlas.
- Copy **`.env.example` → `.env`** per service (gitignored). Load env before `java -jar`, e.g. `set -a && source .env && set +a` — see [MANUAL_DEPLOYMENT.md](docs/MANUAL_DEPLOYMENT.md).
- **Same `JWT_SECRET`** on user-, product-, and order-service.

## Security Notes

- Bearer JWT on APIs except public auth routes.
- **order-service → product-service** over HTTP with JWT (no shared DB).
- Use K8s Secrets / AWS Secrets Manager in production.

## Development

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.5/install.sh | bash
source ~/.bashrc
nvm install node
node -v
npm -v
apt-get update && apt-get install -y libatomic1
cd frontend && npm install && npm run dev
npm run lint && npm run build    # frontend/.eslintrc.cjs

export JAVA_HOME=$(/usr/libexec/java_home -v 17 2>/dev/null)   # macOS
cd user-service && mvn clean package -DskipTests

 aws s3 cp  dist/  s3://mederp --recursive
```

`*/target/` and `frontend/dist/` are gitignored. Clean with `mvn clean` and delete `frontend/dist` if needed.

## License

Proprietary — **Edublitz — Powered by Greamio Technologies Pvt Ltd.**  
See [LICENSE](LICENSE). All rights reserved.

admin@medicalerp.com
Admin@123456
