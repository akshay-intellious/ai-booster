# AI SDLC Booster — BA Agent

An AI-powered Business Analyst Agent with Jira/GitHub integration, powered by Gemini 2.0 Flash and Vertex AI RAG.

## Project Structure

```
ai-sdlc-ui/
├── app/
│   ├── frontend/           # Next.js 16 + React 19 UI
│   │   ├── app/            # Next.js App Router pages
│   │   │   ├── ba-agent/   # BA Agent chat interface
│   │   │   ├── dashboard/  # Project dashboard
│   │   │   └── project/    # Project detail pages
│   │   ├── components/     # React components (ui, dashboard, landing)
│   │   ├── lib/            # Frontend utilities
│   │   ├── public/         # Static assets
│   │   ├── package.json    # Node.js dependencies
│   │   ├── tsconfig.json   # TypeScript config
│   │   └── Dockerfile      # Frontend container
│   └── backend/            # FastAPI Python backend
│       ├── main.py         # Entry point (port 8000)
│       ├── routers/        # API route handlers
│       ├── services/       # Business logic
│       │   ├── ba_agent_service.py      # Core chat engine
│       │   ├── intent_classifier.py     # LLM intent classification
│       │   ├── user_learning_service.py # Feedback & learning
│       │   ├── conversation_service.py  # Conversation storage
│       │   ├── jira_service.py          # Jira API integration
│       │   └── local_github_search.py   # Code search
│       ├── models/         # Pydantic schemas
│       ├── utils/          # Logging utilities
│       ├── config/         # Configuration files
│       ├── requirements.txt
│       └── Dockerfile      # Backend container
├── docs/                   # Documentation & diagrams
│   ├── diagrams/           # Architecture diagrams (.drawio, .md)
│   └── *.md                # Guides, testing docs
├── docker-compose.yml      # Full stack orchestration
├── dev.sh                  # Local development script
├── .gitignore
└── README.md
```

## Quick Start

### Local Development

**Backend:**
```bash
cd app/backend
export GOOGLE_APPLICATION_CREDENTIALS=gcp-service-account.json
pip install -r requirements.txt
python -m uvicorn main:app --host 0.0.0.0 --port 8000
```

**Frontend:**
```bash
cd app/frontend
npm install
npm run dev -- -p 3000
```

**Or use dev.sh:**
```bash
./dev.sh
```

### Docker

```bash
docker-compose up --build
```

## Tech Stack

| Component | Technology |
|---|---|
| Frontend | Next.js 16, React 19, Tailwind CSS, Radix UI |
| Backend | FastAPI, Python 3.12 |
| AI Model | Gemini 2.0 Flash (Vertex AI) |
| Embeddings | text-embedding-004 (768 dims) |
| Vector DB | Vertex AI RAG Corpus |
| Jira | Atlassian REST API |
| Code Search | Local GitHub Index (12 repos, 45K+ chunks) |

## Architecture

See [docs/diagrams/BA_Agent_Architecture_Flow.md](docs/diagrams/BA_Agent_Architecture_Flow.md) for the complete architecture diagram.
# ai-booster-poc

AI-Booster is a proof-of-concept that demonstrates a secure, Cloud Run–based backend and frontend, fronted by Cloud Armor–protected HTTPS load balancers and CI/CD workflows in GitHub Actions.

- Backend and frontend run on Cloud Run (v2) with ingress restricted to **internal-and-cloud-load-balancing**.
- External access is only via HTTPS load balancers protected by Cloud Armor and Cloudflare Zero Trust.
- Frontend → backend traffic is routed through a **Serverless VPC Access connector** and **Cloud NAT** so it always originates from a single, allowlisted egress IP.


## Key Documentation

- **Contribution & PR workflow:** [CONTRIBUTION_PROCESS.md](CONTRIBUTION_PROCESS.md)
- **Developer setup & local checks:** [DEVELOPER_SETUP.md](DEVELOPER_SETUP.md)
- **Terragrunt infrastructure:** [infra/terragrunt/README.md](infra/terragrunt/README.md)
- **Multi-environment guide:** [docs/terragrunt-multi-environment-infrastructure.md](docs/terragrunt-multi-environment-infrastructure.md)
- **Architecture strategy:** [docs/architecture-multi-environment-strategy.md](docs/architecture-multi-environment-strategy.md)
- **Legacy Terraform:** [infra/terraform/README.md](infra/terraform/README.md) (deprecated, use Terragrunt)


## High-Level Architecture

- **Cloud Run services**
    - `ai-booster-frontend-main` – React/Vite frontend behind a HTTPS load balancer.
    - `ai-booster-backend-main` – Python backend behind a HTTPS load balancer.

- **DNS & Load Balancers (main)**
    - Frontend: `ai-booster.corp.bigcommerce.net` → frontend HTTPS LB → `ai-booster-frontend-main`.
    - Backend:  `ai-booster-api.corp.bigcommerce.net` → backend HTTPS LB → `ai-booster-backend-main`.

- **Networking & NAT**
    - Custom VPC `ai-booster-vpc` with small dedicated subnet for Serverless VPC Access.
    - Connector `ai-booster-svpc` and Cloud NAT with static IP `ai-booster-frontend-nat-ip`.
    - Frontend Cloud Run uses the connector with `ALL_TRAFFIC` egress so all outbound traffic (including backend calls) uses the NAT IP.

- **Cloud Armor**
    - Frontend LB: policy `cloudflare-only-frontend` – allows only Cloudflare/WARP corporate ranges.
    - Backend LB: policy `cloudflare-only-backend` – allows Cloudflare/WARP ranges **plus** the NAT IP `/32`.


## CI/CD Overview

The repository uses GitHub Actions with **clear separation** between app deployments and infrastructure:

### Workflow Comparison: Developer vs DevOps

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    DEVELOPER WORKFLOW (App Code)                                        │
│                    v3_ci_cd_orchestrator.yml                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Feature Branch Push ───► DEV Environment ───► Open PR ───► TEST Environment          │
│   (app/backend/**)         (fast feedback)      (push)       (quality gates)           │
│   (app/frontend/**)                                                                     │
│                                                     │                                   │
│                                                     ▼                                   │
│                                              Merge to Main ───► PRE-PROD Environment   │
│                                                                                         │
│   Trigger: push to feature/* branches                                                   │
│   Trigger: push to branches with open PR                                                │
│   Trigger: push to main                                                                 │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    DEVOPS WORKFLOW (Infrastructure)                                      │
│                    v3_terragrunt_ci.yml                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   Feature Branch ───► Open PR (required) ───► TEST Environment ───► Merge to Main      │
│   (infra/terragrunt/**)   (pull_request)      FOUNDATION + APP      │                   │
│                                                                      ▼                  │
│   ⚠️ NO workflow runs                                           PRE-PROD → PROD        │
│      until PR is opened                                         (with approvals)       │
│                                                                                         │
│   Trigger: pull_request (opened, synchronize, reopened) ONLY                           │
│   Trigger: push to main                                                                 │
│                                                                                         │
│   Why PR-only? Infra changes need review before any apply.                             │
│   Open a Draft PR for validation without review requirement.                           │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Workflow Separation Table

| Scope | CI/CD Workflow | Teardown Workflow | Trigger | Trigger Paths |
|-------|---------------|-------------------|---------|---------------|
| **App Code** | `v3_ci_cd_orchestrator.yml` | `v3_pr_teardown.yml` | `push` (feature + main) | `app/backend/**`, `app/frontend/**` |
| **Infrastructure** | `v3_terragrunt_ci.yml` | `v3_terragrunt_teardown.yml` | `pull_request` + `push` (main only) | `infra/terragrunt/**` |

> **Note:** Docs-only or config-only changes do NOT trigger any workflows.

### App Workflows (Developer)

- **Trigger:** Push to any branch (feature branches, PRs, main)
- **Quality Gates:** Backend linting, formatting, tests; Frontend ESLint, Jest, Vite build
- **Container Build:** Pushes to Google Artifact Registry (GAR)
- **Cloud Run Deploy:** Deploys backend/frontend with HTTP LB and Cloud Armor
- **Teardown:** Cleans up PR-specific Cloud Run resources on PR close

### Infrastructure Workflows (DevOps)

- **Trigger:** Pull request events ONLY (no push trigger for feature branches)
- **Terragrunt CI:** Apply TEST layers on PR, Apply PRE-PROD/PROD on main merge
- **Multi-environment:** TEST → PRE-PROD (1 approval) → PROD (2 approvals)
- **Teardown:** Destroys PR-isolated Terragrunt resources on PR close

For details on contribution workflow, PR checks, and preview environments, see [CONTRIBUTION_PROCESS.md](CONTRIBUTION_PROCESS.md).
