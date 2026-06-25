# HelloDeploy

[![CI/CD Pipeline](https://github.com/akanksv/hello-deploy/actions/workflows/pipeline.yml/badge.svg)](https://github.com/akanksv/hello-deploy/actions/workflows/pipeline.yml)

A containerized **Hello World** web application with automated testing, immutable image publishing, staged deployment, production promotion, health verification, rollback support, monitoring endpoints, and Terraform-based infrastructure management.

## Quick Links

### Project Resources

- [GitHub Repository](https://github.com/akanksv/hello-deploy)
- [CI/CD Workflow](https://github.com/akanksv/hello-deploy/actions/workflows/pipeline.yml)
- [Production Application](http://13.50.21.171)
- [Production Health](http://13.50.21.171/health)
- [Production Readiness](http://13.50.21.171/ready)
- [Production Version](http://13.50.21.171/version)
- [Production Metrics](http://13.50.21.171/metrics)

### Documentation Navigation

- [Verification Procedure](#verification-procedure)
- [Project Overview](#project-overview)
- [Engineering Goals](#engineering-goals)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Application Endpoints](#application-endpoints)
- [Docker Setup](#docker-setup)
- [Local Development](#local-development)
- [Local Quality Checks](#local-quality-checks)
- [CI/CD Pipeline](#cicd-pipeline)
- [GitHub Configuration](#github-configuration)
- [Deployment](#deployment)
- [Rollback Strategy](#rollback-strategy)
- [Monitoring and Logging](#monitoring-and-logging)
- [Infrastructure as Code](#infrastructure-as-code)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Operational Commands](#operational-commands)
- [Technical Implementation Summary](#technical-implementation-summary)
- [Conclusion](#conclusion)

> The production links are available only while the AWS EC2 instance is running.

<a id="verification-procedure"></a>

## Verification Procedure

The project can be assessed without access to private AWS or GitHub credentials.

1. The [CI/CD workflow history](https://github.com/akanksv/hello-deploy/actions/workflows/pipeline.yml) provides records of successful runs on `main`.
2. A successful run includes completed linting, testing, container validation, image publication, staging deployment, and production deployment stages.
3. The [production health endpoint](http://13.50.21.171/health) is expected to report `healthy`.
4. The [production version endpoint](http://13.50.21.171/version) is expected to report the `production` environment and a Git commit SHA.
5. Independent local execution is documented in the [Local Development](#local-development) section.

The internal staging service, AWS account, SSH credentials, GitHub environment secrets, and Terraform state are intentionally not public.

---

<a id="project-overview"></a>

## 1. Project Overview

HelloDeploy is a deployment engineering project built around a deliberately small FastAPI service. The application remains simple by design so that the repository can focus on reproducible builds, release traceability, staged promotion, automated verification, operational visibility, and recovery behavior.

The project implements:

- A FastAPI web application that returns **Hello World**
- Docker-based application packaging
- Docker Compose orchestration
- Nginx reverse proxying
- Automated linting, testing, Compose validation, and container smoke testing
- Immutable Docker image publishing to GitHub Container Registry
- Separate staging and production deployments
- SSH-based automated deployment to an AWS EC2 instance
- Health, readiness, version, and Prometheus metrics endpoints
- Automatic rollback to the previous healthy release
- Infrastructure management with Terraform
- A self-hosted GitHub Actions runner on the deployment server

The repository is available at:

```text
https://github.com/akanksv/hello-deploy
```

---

<a id="engineering-goals"></a>

## 2. Engineering Goals

The implementation is designed around the following engineering goals:

1. Package the application in a reproducible container image.
2. Validate application code and deployment configuration automatically.
3. Build and test the real container before publication.
4. Publish every release with an immutable Git commit SHA tag.
5. Deploy the same image first to staging and then to production.
6. Verify that the deployed version matches the triggering commit.
7. Restore the previous release automatically if a new deployment is unhealthy.
8. Store infrastructure configuration as code.
9. Keep credentials outside the repository.
10. Provide observable health and metrics endpoints.

---

<a id="technology-stack"></a>

## 3. Technology Stack

| Area | Technology |
|---|---|
| Application | Python 3.12, FastAPI, Uvicorn |
| Testing | Pytest, HTTPX |
| Linting | Ruff |
| Containerization | Docker |
| Orchestration | Docker Compose |
| Reverse proxy | Nginx |
| CI/CD | GitHub Actions |
| Container registry | GitHub Container Registry |
| Deployment host | AWS EC2, Ubuntu/Linux |
| Infrastructure as Code | Terraform |
| Monitoring | Prometheus client metrics |
| Deployment transport | SSH and SCP |
| Version identification | Git commit SHA |

---

<a id="architecture"></a>

## 4. Architecture

The following diagram shows the implemented HelloDeploy architecture, including the GitHub Actions CI/CD pipeline, immutable image publication to GHCR, staged deployment on AWS EC2, production approval, release verification, rollback controls, operational endpoints, security measures, and Terraform infrastructure representation.

![HelloDeploy overall architecture](docs/hellodeploy-architecture.png)

> The same commit-SHA-tagged Docker image is built once, tested, published to GHCR, deployed to staging, verified, and then promoted unchanged to production. If a candidate deployment does not become healthy, the deployment script restores the previous release configuration and verifies the recovered service.

### Deployment environments

| Environment | Compose project | Host port | Public access | Verification URL |
|---|---|---:|---|---|
| Staging | `hello-staging` | `8080` | Not publicly exposed | `http://127.0.0.1:8080` |
| Production | `hello-production` | `80` | Public | `http://127.0.0.1:80` |

Deployment jobs execute on a Linux self-hosted runner installed on the EC2 host. The release procedure still uses SSH and SCP, which keeps deployment responsibilities explicit and preserves the same deployment contract if the runner and target host are separated in a future architecture.

---

<a id="repository-structure"></a>

## 5. Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── pipeline.yml
├── app/
│   ├── __init__.py
│   └── main.py
├── deploy/
│   ├── deploy.sh
│   └── nginx.conf
├── infrastructure/
│   ├── .terraform.lock.hcl
│   ├── ec2.tf
│   ├── imports.tf
│   ├── providers.tf
│   └── versions.tf
├── docs/
│   └── hellodeploy-architecture.png
├── tests/
│   └── test_app.py
├── .dockerignore
├── .env.example
├── .gitattributes
├── .gitignore
├── compose.local.yml
├── compose.yml
├── Dockerfile
├── pyproject.toml
├── requirements-dev.txt
└── requirements.txt
```

| File or directory | Purpose |
|---|---|
| `app/main.py` | FastAPI application and operational endpoints |
| `tests/test_app.py` | Automated endpoint tests |
| `Dockerfile` | Multi-stage production image |
| `compose.yml` | Production-style application and Nginx services |
| `compose.local.yml` | Local image build override |
| `deploy/deploy.sh` | Deployment, health verification, and rollback logic |
| `deploy/nginx.conf` | Reverse proxy configuration |
| `.github/workflows/pipeline.yml` | Complete CI/CD pipeline |
| `infrastructure/` | Terraform configuration for the AWS EC2 instance |
| `docs/hellodeploy-architecture.png` | Final system architecture diagram used in the project documentation |
| `.env.example` | Example runtime configuration |
| `.dockerignore` | Files excluded from Docker build context |
| `.gitignore` | Local, secret, generated, and Terraform state exclusions |

---

<a id="application-endpoints"></a>

## 6. Application Endpoints

| Endpoint | Purpose | Expected result |
|---|---|---|
| `/` | Human-readable application page | HTML page containing `Hello World!` |
| `/health` | Liveness and deployment health | `{"status":"healthy"}` |
| `/ready` | Readiness verification | `{"status":"ready"}` |
| `/version` | Deployment proof | Environment, commit SHA, build time, and current time |
| `/metrics` | Prometheus-compatible metrics | Request counters and deployment information |

Example production checks:

```bash
curl --fail http://13.50.21.171/health
curl --fail http://13.50.21.171/ready
curl --fail http://13.50.21.171/version
curl --fail http://13.50.21.171/metrics
```

The `version` field is injected into the image at build time and is used by the pipeline to prove that the expected commit was deployed.

---

<a id="docker-setup"></a>

## 7. Docker Setup

### 7.1 Dockerfile

The Dockerfile uses a multi-stage build:

1. The builder stage creates a virtual environment and installs Python dependencies.
2. The runtime stage copies only the virtual environment and application source.
3. The application runs as the non-root user `appuser`.
4. The Git commit SHA and build time are stored as image metadata and environment variables.
5. A container-level health check calls the application health endpoint.

Important production characteristics:

- Python 3.12 slim base image
- Non-root runtime user
- Minimal runtime content
- Built-in health check
- OCI image metadata
- No development dependencies in the production image

### 7.2 Docker Compose

`compose.yml` defines two services:

#### `app`

- Runs the immutable application image
- Exposes port `8000` only to the internal Compose network
- Uses an application health check
- Uses a read-only filesystem
- Uses a temporary filesystem for `/tmp`
- Enables `no-new-privileges`
- Limits Docker JSON log file size and retention

#### `proxy`

- Uses `nginx:1.27-alpine`
- Waits for the application service to become healthy
- Maps the configured host port to container port `80`
- Proxies requests to `app:8000`
- Uses a read-only Nginx configuration mount
- Enables `no-new-privileges`
- Applies log rotation

Both services communicate through a dedicated bridge network named `application`.

---

<a id="local-development"></a>

## 8. Local Development

### Prerequisites

- Git
- Docker Engine or Docker Desktop
- Docker Compose v2
- Python 3.12 for direct test execution

### Clone and configure

```bash
git clone https://github.com/akanksv/hello-deploy.git
cd hello-deploy
cp .env.example .env
```

PowerShell equivalent:

```powershell
Copy-Item .env.example .env
```

### Start locally

```bash
docker compose \
  -f compose.yml \
  -f compose.local.yml \
  up --build --detach
```

The application is then available at:

```text
http://127.0.0.1:8080
```

### Verify locally

```bash
curl --fail http://127.0.0.1:8080/health
curl --fail http://127.0.0.1:8080/ready
curl --fail http://127.0.0.1:8080/version
```

### View status and logs

```bash
docker compose -f compose.yml -f compose.local.yml ps
docker compose -f compose.yml -f compose.local.yml logs --follow
```

### Stop locally

```bash
docker compose -f compose.yml -f compose.local.yml down
```

---

<a id="local-quality-checks"></a>

## 9. Local Quality Checks

```bash
python -m venv .venv
```

Linux or macOS Bash:

```bash
source .venv/bin/activate
```

Windows Git Bash:

```bash
source .venv/Scripts/activate
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install dependencies and run checks:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt -r requirements-dev.txt
ruff check app tests
pytest --verbose
cp .env.example .env
docker compose -f compose.yml -f compose.local.yml config --quiet
```

---

<a id="cicd-pipeline"></a>

## 10. CI/CD Pipeline

The workflow is defined in:

```text
.github/workflows/pipeline.yml
```

### Triggers

The workflow runs on:

- Every pull request
- Pushes to `main`, except changes limited to Markdown files or the `docs/` directory
- Manual execution through `workflow_dispatch`

Pull requests perform validation and container testing but do not publish or deploy release images. A commit that changes both documentation and application or deployment files still starts the complete pipeline.

### Pipeline stages

```mermaid
flowchart LR
    A[Pull Request or Push] --> B[Lint, Test, and Validate]
    B --> C[Build and Smoke-Test Container]
    C --> D{Main branch?}
    D -->|No| E[Finish]
    D -->|Yes| F[Publish Immutable Image]
    F --> G[Deploy Staging]
    G --> H[Verify Staging Commit]
    H --> I[Production Environment]
    I --> J[Deploy Production]
    J --> K[Verify Production Commit]
```

#### Stage 1: Lint, test, and validate

Runs on a GitHub-hosted Ubuntu runner and performs dependency installation, Ruff linting, Pytest execution, and Docker Compose validation.

#### Stage 2: Build and smoke-test container

Builds a real container, starts it, checks `/health`, verifies `/version`, compares the reported version with `GITHUB_SHA`, collects logs after failure, and removes the test container.

#### Stage 3: Publish immutable image

Runs only for `main` and publishes:

```text
ghcr.io/akanksv/hello-deploy:${GITHUB_SHA}
```

The commit SHA is the immutable release tag.

#### Stage 4: Deploy staging

Runs on the self-hosted Linux runner. It prepares SSH credentials, validates the host key, tests SSH access, transfers deployment files, deploys `hello-staging`, verifies health, and confirms the expected commit through `http://127.0.0.1:8080`.

#### Stage 5: Deploy production

Runs after staging succeeds. It deploys the same immutable image to `hello-production` and verifies the expected commit through `http://127.0.0.1:80`.

The GitHub `production` environment can be configured with required reviewers so deployment pauses for approval.

### Concurrency

Workflow concurrency prevents uncontrolled parallel execution for the same Git reference. Production also uses a dedicated concurrency group to prevent overlapping production deployments.

---

<a id="github-configuration"></a>

## 11. GitHub Configuration

### Environments

- `staging`
- `production`

### Required environment secrets

| Secret | Purpose |
|---|---|
| `SSH_PRIVATE_KEY` | Private key used by the deployment job |
| `SSH_KNOWN_HOSTS` | Trusted SSH host key entry |
| `SSH_USER` | Remote deployment user |
| `SSH_HOST` | Deployment host address |

### Required environment variables

#### Staging

| Variable | Value or purpose |
|---|---|
| `APPLICATION_URL` | `http://127.0.0.1:8080` |
| `FORCE_UNHEALTHY` | `false` during normal operation; `true` enables controlled rollback testing |

#### Production

| Variable | Value or purpose |
|---|---|
| `APPLICATION_URL` | `http://127.0.0.1:80` |

The loopback addresses are intentional. The self-hosted runner is installed on the EC2 host, so verification is independent of the developer's router, location, or public IP.

---

<a id="deployment"></a>

## 12. Deployment

### Normal deployment process

1. A change is developed on a feature branch.
2. A pull request is opened for review.
3. Linting, tests, Compose validation, and container smoke testing are executed automatically.
4. Following review, the change is merged into `main`.
5. A release image is built and published with the merge commit SHA as its tag.
6. The image is deployed to staging.
7. Staging health and commit identity are verified.
8. Production approval is requested when environment protection is enabled.
9. The same immutable image is deployed to production.
10. Production health and commit identity are verified.

### Deployment directories

```text
/opt/hello-deploy/staging
/opt/hello-deploy/production
```

Each environment has an independent Compose project, `.env`, `.env.previous`, and deployment directory.

### Manual verification on the EC2 host

```bash
curl --fail http://127.0.0.1:8080/health
curl --fail http://127.0.0.1:8080/version
curl --fail http://127.0.0.1:80/health
curl --fail http://127.0.0.1:80/version
```

### Public production verification

```bash
curl --fail http://13.50.21.171/health
curl --fail http://13.50.21.171/version
```

Port `8080` is intentionally not publicly exposed.

---

<a id="rollback-strategy"></a>

## 13. Rollback Strategy

Rollback logic is implemented in `deploy/deploy.sh`.

Before deployment, the current `.env` file is copied to `.env.previous`. The script then writes the candidate configuration, pulls the candidate image, recreates the services, and waits for the application container to become healthy.

If the candidate remains unhealthy, the script restores `.env.previous`, recreates the previous release, verifies its health, and returns a failure status so the pipeline records the failed candidate deployment.

### Rollback test mechanism

Controlled rollback testing is supported in the staging environment through:

```text
FORCE_UNHEALTHY=true
```

With this value enabled, the candidate health endpoint returns HTTP `503`, causing the deployment script to attempt restoration of the previous healthy release. Normal staging operation uses:

```text
FORCE_UNHEALTHY=false
```

Production always uses `FORCE_UNHEALTHY=false`.

---

<a id="monitoring-and-logging"></a>

## 14. Monitoring and Logging

| Capability | Endpoint or mechanism |
|---|---|
| Liveness | `/health` |
| Readiness | `/ready` |
| Deployment identity | `/version` |
| Prometheus metrics | `/metrics` |
| Container logs | Docker `json-file` driver |
| Log rotation | `10m` maximum size, `3` files |

Example log commands:

```bash
docker compose \
  --project-name hello-staging \
  --file /opt/hello-deploy/staging/compose.yml \
  logs --tail=100
```

```bash
docker compose \
  --project-name hello-production \
  --file /opt/hello-deploy/production/compose.yml \
  logs --tail=100
```

---

<a id="infrastructure-as-code"></a>

## 15. Infrastructure as Code

Terraform configuration is stored in `infrastructure/`.

This directory defines the AWS provider and the EC2 deployment host declaratively. The configuration captures the host characteristics required by the application platform and applies lifecycle protection to reduce the risk of accidental destruction.

| Item | Configuration |
|---|---|
| Provider | AWS |
| Region | `eu-north-1` |
| Availability Zone | `eu-north-1b` |
| Instance type | `t3.micro` |
| Production public address | `13.50.21.171` |
| Root volume | 8 GiB `gp3` |
| Instance metadata | IMDSv2 required |

The EC2 resource includes:

```hcl
lifecycle {
  prevent_destroy = true
}
```

### Terraform commands

```bash
cd infrastructure
terraform init
terraform fmt -check
terraform validate
terraform plan
```

Infrastructure changes are applied only after the Terraform plan has been reviewed:

```bash
terraform apply
```

### Terraform state

Terraform state is excluded from Git through `.gitignore`. The current implementation uses local state, so reliable infrastructure operations depend on preserving the correct state file securely.

For collaborative or long-lived operation, the state should be migrated to a secured remote backend with locking, versioning, restricted access, and recovery controls.

---

<a id="security-considerations"></a>

## 16. Security Considerations

### Implemented controls

- Default GitHub Actions permission is `contents: read`
- Package write permission is limited to the image publication job
- Deployment credentials are stored as GitHub environment secrets
- Strict SSH host key checking is enabled
- Temporary SSH key files are deleted after deployment
- Images use immutable commit SHA tags
- The application runs as a non-root user
- Containers use `no-new-privileges`
- The application container uses a read-only filesystem
- Docker logs are rotated
- Staging port `8080` is not publicly exposed
- Terraform state is excluded from source control
- EC2 metadata requires IMDSv2
- Terraform uses `prevent_destroy`
- Production follows successful staging verification

### Known limitations and trade-offs

1. **HTTP only:** HTTPS is not yet configured.
2. **SSH exposure:** Port `22` is publicly reachable. AWS Systems Manager, a VPN, or a restricted network path would be stronger.
3. **Single host:** Staging, production, and the self-hosted runner share one EC2 instance.
4. **Local Terraform state:** Collaboration requires secure state sharing or a remote backend.
5. **Unencrypted root volume:** The root EBS volume is not encrypted.
6. **Public metrics:** `/metrics` is reachable through production and should be protected in a stricter environment.
7. **No high availability:** There is no load balancer, auto scaling, or multi-zone redundancy.

---

<a id="troubleshooting"></a>

## 17. Troubleshooting

### Workflow waits for a self-hosted runner

This condition indicates that the EC2 instance or runner is offline. The EC2 instance must be running, and the Linux x64 runner status must appear as online under **Settings → Actions → Runners**.

### Staging reports the production environment

The required environment-specific values are:

```text
staging APPLICATION_URL = http://127.0.0.1:8080
production APPLICATION_URL = http://127.0.0.1:80
```

### External access to port 8080 fails

This is the expected behavior because staging is internal. Staging verification is performed on the EC2 host:

```bash
curl --fail http://127.0.0.1:8080/health
```

### Version verification fails

Relevant diagnostic information includes the `/version` response, the image tag in `.env`, and the running container image:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Each environment must reference the correct `APPLICATION_URL`.

### SSH connection fails

Relevant checks include `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `SSH_KNOWN_HOSTS`, EC2 instance status, security-group rules, and SSH service availability.

### Candidate release is unhealthy

Diagnostic sources include Docker Compose logs, image availability, deployment variables, the `/health` response, and the status of the previous release after rollback.

### Terraform reports unexpected changes

The plan should not be applied until the active AWS account, region, state file, and configuration have been verified and the proposed changes have been compared with the intended infrastructure. Terraform state must be backed up before any repair or reconciliation operation.

---

<a id="operational-commands"></a>

## 18. Operational Commands

### Production checks from an external machine

```bash
curl --fail http://13.50.21.171/health
curl --fail http://13.50.21.171/ready
curl --fail http://13.50.21.171/version
```

### Staging checks from the EC2 host

```bash
curl --fail http://127.0.0.1:8080/health
curl --fail http://127.0.0.1:8080/version
```

### Container status

```bash
docker ps
```

### Staging status

```bash
cd /opt/hello-deploy/staging
docker compose --project-name hello-staging ps
```

### Production status

```bash
cd /opt/hello-deploy/production
docker compose --project-name hello-production ps
```

When the EC2 instance is stopped, the application, staging environment, and self-hosted runner are unavailable. The associated Elastic IP remains unchanged after restart.

---

<a id="technical-implementation-summary"></a>

## 19. Technical Implementation (Summary)

| Technical area | Implementation |
|---|---|
| Delivery architecture | GitHub Actions, GitHub Container Registry, AWS EC2, and distinct staging and production release paths |
| Container image construction | Multi-stage Python image with a non-root runtime user and embedded release metadata |
| Multi-service runtime | Docker Compose defines the application, Nginx proxy, isolated network, health checks, security controls, and log rotation |
| Repository automation | GitHub provides source control, pull request validation, workflow execution, environment controls, and release history |
| Automated build | Reproducible test and release images are built directly from repository contents |
| Automated verification | Ruff, Pytest, Compose validation, and real-container smoke testing run before release publication |
| Execution environments | GitHub-hosted runners perform validation, while a self-hosted Linux runner performs controlled deployment |
| Remote release transport | SSH validates connectivity, SCP transfers release files, and the deployment script applies the release |
| Service reconciliation | Docker Compose recreates services and removes obsolete containers during deployment |
| External availability | The production service is reachable through the EC2 Elastic IP on port 80 |
| Credential protection | SSH credentials and host verification data are stored in GitHub environment secrets |
| Configuration management | Runtime settings are supplied through `.env` files, GitHub variables, and deployment-time environment values |
| Release reproducibility | Slim multi-stage images are tagged with the exact Git commit SHA |
| Runtime health control | Health checks are enforced in the application, image, Compose stack, deployment script, and CI/CD workflow |
| Recovery behavior | Failed candidate releases trigger restoration and verification of the previous healthy configuration |
| Observability | Prometheus-compatible metrics, version metadata, health endpoints, and rotated container logs are available |
| Environment isolation | Staging and production use independent Compose projects, ports, directories, and GitHub environments |
| Declarative infrastructure | Terraform records the EC2 deployment host, provider constraints, lifecycle protection, and repeatable planning commands |
| Operational diagnosis | The documentation includes failure symptoms, likely causes, verification commands, and recovery actions |

---

<a id="conclusion"></a>

## 20. Conclusion

HelloDeploy presents a complete and auditable release path for a small web service. It integrates container construction, automated validation, immutable release identification, staged promotion, SSH-based deployment, runtime verification, rollback handling, operational metrics, security hardening, and declarative infrastructure management.

The current single-instance topology keeps the system economical and understandable while preserving the essential controls of a disciplined delivery process. A production-scale evolution would add HTTPS, centrally managed Terraform state, encrypted storage, private administration through AWS Systems Manager or a controlled network path, separation of runner and workload hosts, and high-availability infrastructure.
