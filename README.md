# E4L DevOps Platform - Setup Guide

Complete CI/CD platform for E4L (Energy for Life) application with staging and production environments, automated testing, and quality gates.

## 📦 Asset Composition

```
scripts/                      Automation scripts for full setup
  ├── setup.sh                Master setup script (runs all below)
  ├── setup_envs.sh           Provision staging and production VMs
  ├── setup_projects.sh       Create GitLab projects and CI/CD variables
  ├── register_runner.sh      Register CI/CD runners for projects
  ├── setup_gitlab.sh         Setup GitLab runner container
  └── cleanup.sh              Tear down VMs and environments

repos/                        Application source code
  ├── backende4l/             Backend (Java/Spring Boot + Gradle)
  └── frontende4l/            Frontend (React/Webpack + Node.js)

ansible-stage/                Ansible configuration for staging VM
  ├── Vagrantfile             Vagrant VM config (192.168.56.11)
  ├── playbook.yml            Ansible provisioning playbook
  ├── hosts.ini               SSH configuration
  └── docker-compose.*.yml    Docker compose files

ansible-prod/                 Ansible configuration for production VM
  ├── Vagrantfile             Vagrant VM config (192.168.56.12)
  ├── playbook.yml            Ansible provisioning playbook
  ├── hosts.ini               SSH configuration
  └── docker-compose.*.yml    Docker compose files
```

## 🔧 Prerequisites

### Hardware Requirements
- Minimum 16 GB RAM (32 GB recommended)
- 100 GB available disk space (for VMs)
- Multi-core processor (4+ cores)

### Software Requirements
- Windows 10/11 or Linux
- VirtualBox 6.1+
- Vagrant 2.2+
- Docker Desktop / Docker Engine
- Git v2.25+
- SSH client (OpenSSH)
- curl
- GitLab CE (will be installed by setup script)
- GitLab Runner (will be installed by setup script)

### Docker Hub Account
- You need a Docker Hub account for pushing images
- Default registry: `docker.io/minfranco`
- Update CI/CD variables if using a different account

## 🌐 Port Configuration

| Service | Environment | Host Port | Container Port |
|---------|------------|-----------|----------------|
| Frontend | Both | 8082 | 80 |
| Backend | Both | 8084 | 8080 |
| MariaDB | Both | - | 3306 (internal) |
| GitLab | - | 8929 | - |

**Note:** Ports are unified across staging and production since each environment runs on a separate VM (no conflicts).

## 🗄️ Database Configuration

| Setting | Value |
|---------|-------|
| Database Name | `e4l` |
| Username | `e4l` |
| Password | `e4lpassword` |
| Root Password | `rootpassword` |
| Driver | `org.mariadb.jdbc.Driver` |

## 🏗️ Architecture Overview

### Three Environments

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           HOST MACHINE                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  DEV ENVIRONMENT (Developer Workstation)                           │ │
│  │                                                                    │ │
│  │    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐          │ │
│  │    │   IDE        │   │  Backend     │   │  Frontend    │          │ │
│  │    │   Git CLI    │   │  :8080       │   │  :3000       │          │ │
│  │    └──────────────┘   └──────────────┘   └──────────────┘          │ │
│  │                              │                                     │ │
│  └──────────────────────────────┼─────────────────────────────────────┘ │
│                                 ▼ git push (dev or main branch)         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  INTEGRATION (GitLab :8929 + Docker Hub)                           │ │
│  │                                                                    │ │
│  │  ┌────────────────────────────────────────────────────────────┐    │ │
│  │  │  PIPELINE FLOW (Branch-Based Deployment)                   │    │ │
│  │  │  • dev branch  → Deploy to STAGING                         │    │ │
│  │  │  • main branch → Deploy to PRODUCTION (after E2E tests)    │    │ │
│  │  └────────────────────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐     │
│  │ STAGING VM (e4l-stage)       │  │ PRODUCTION VM (e4l-prod)     │     │
│  │ IP: 192.168.56.11            │  │ IP: 192.168.56.12            │     │
│  │ SSH Port: 22                 │  │ SSH Port: 22                 │     │
│  │ Working Dir: /opt/e4l        │  │ Working Dir: /opt/e4l-prod   │     │
│  │                              │  │                              │     │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │     │
│  │  │ Frontend (Nginx) :8082 │  │  │  │ Frontend (Nginx) :8082 │  │     │
│  │  └───────────┬────────────┘  │  │  └───────────┬────────────┘  │     │
│  │              ▼               │  │              ▼               │     │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │     │
│  │  │ Backend (Spring) :8084 │  │  │  │ Backend (Spring) :8084 │  │     │
│  │  └───────────┬────────────┘  │  │  └───────────┬────────────┘  │     │
│  │              ▼               │  │              ▼               │     │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐  │     │
│  │  │ MariaDB :3306          │  │  │  │ MariaDB :3306          │  │     │
│  │  │ DB: e4l                │  │  │  │ DB: e4l                │  │     │
│  │  │ Container: e4l-db      │  │  │  │ Container: e4l-db      │  │     │
│  │  └────────────────────────┘  │  │  └────────────────────────┘  │     │
│  │                              │  │                              │     │
│  │  Deployed from: dev branch   │  │  Deployed from: main branch  │     │
│  └──────────────────────────────┘  └──────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Backend Pipeline (7 Stages with Quality Gates)

```
PRE-BUILD ──► BUILD ──► UNIT TEST ──► INTEGRATION TEST ──► DOCKER BUILD
    │           │            │                │                   │
Set vars   ./gradlew    JUnit tests    Spring Boot          Push image
           build        (20 tests)     integration          to Docker Hub
                                       (4 tests)

Branch determines deployment:
• dev  → STAGING (192.168.56.11)
• main → STAGING + E2E → PRODUCTION (192.168.56.12)

Only on main branch:
DEPLOY STAGING ──► E2E ACCEPTANCE TESTS ──► DEPLOY PRODUCTION
       │                    │                        │
  SSH to VM           Newman/Postman            SSH to VM
  docker-compose      4 E2E tests               docker-compose
                      (12 assertions)
                      on staging API
                      (192.168.56.11:8084)

⚠️ CRITICAL: Production deployment blocked if E2E tests fail
```

### Frontend Pipeline (7 Stages with Quality Gates)

```
BUILD ──► UNIT TEST ──► INTEGRATION TEST ──► DOCKER BUILD ──► DEPLOY STAGING
  │           │              │                    │                  │
npm ci +  Jest tests    React Testing      Build staging       SSH to VM
npm build (reducers/    Library tests      image & push        docker-compose
          actions)      (components)       to Docker Hub

Only on main branch:
E2E ACCEPTANCE TESTS ──► DOCKER BUILD (PROD) ──► DEPLOY PRODUCTION
         │                       │                        │
   Axios+Cheerio tests     Build prod image          SSH to VM
   on staging env          ONLY if E2E pass          docker-compose
   (192.168.56.11:8082)

⚠️ CRITICAL: Production deployment blocked if E2E tests fail
```

## 🚀 Setup Instructions

### Ubuntu/KVM Compatibility Fix (Ubuntu Users Only)

If you're running Ubuntu with KVM installed, you need to unload KVM kernel modules before using VirtualBox:

**For AMD processors:**
```bash
sudo rmmod kvm_amd
sudo rmmod kvm
```

**For Intel processors:**
```bash
sudo rmmod kvm_intel
sudo rmmod kvm
```

**Note:** Run the appropriate commands based on your CPU type before proceeding with the setup.

### Prerequisites Check
Ensure all software requirements are installed (VirtualBox, Vagrant, Docker, Git, SSH client, curl).

### Step-by-Step Setup

Run these commands in sequence from the project root:

```bash
# 1. Provision staging and production VMs (10-15 minutes)
./scripts/setup_envs.sh

# 2. Setup GitLab Runner (~1 minute)
./scripts/setup_gitlab.sh

# 3. Create GitLab projects and configure CI/CD variables (~1 minute)
./scripts/setup_projects.sh

# 4. Register GitLab runners (~30 seconds)
./scripts/register_runner.sh
```

**Or run the master setup script that executes all steps:**

```bash
./scripts/setup.sh
```

## 🔍 Verify Setup

After setup completes, visit these URLs:

### GitLab & Projects
- **GitLab Login:** `http://localhost:8929` (testdev / vx6Yo1Mnmn4q7D4Q)
- **Backend Repo:** `http://localhost:8929/testdev/backend`
- **Frontend Repo:** `http://localhost:8929/testdev/frontend`
- **Backend Pipeline:** `http://localhost:8929/testdev/backend/-/pipelines`
- **Frontend Pipeline:** `http://localhost:8929/testdev/frontend/-/pipelines`

### Application Endpoints
- **Staging Frontend:** `http://192.168.56.11:8082`
- **Staging Backend:** `http://192.168.56.11:8084/e4lapi/questionnaire`
- **Production Frontend:** `http://192.168.56.12:8082`
- **Production Backend:** `http://192.168.56.12:8084/e4lapi/questionnaire`

### Docker Hub
- **Images:** `https://hub.docker.com/u/minfranco`

## 🌿 Branching Strategy & Code Freeze

### Development Flow with Quality Gates

#### dev branch commits:
- Automatically build, test, and deploy to **STAGING** environment
- **Backend:** PRE-BUILD → BUILD → UNIT TEST → INTEGRATION TEST → DOCKER BUILD → DEPLOY STAGING
- **Frontend:** BUILD → UNIT TEST → INTEGRATION TEST → DOCKER BUILD → DEPLOY STAGING
- Images pushed to:
  - `minfranco/e4l-backend-stage:latest`
  - `minfranco/e4l-frontend-stage:latest`
- Deployed to: `192.168.56.11` (e4l-stage VM)
- Used for continuous integration and testing

#### main branch commits (with CODE FREEZE and E2E gating):
- Automatically build, test, **run E2E tests on staging**, then deploy to **PRODUCTION**
- **E2E Acceptance Tests:** Quality gate before production
  - Backend: Newman/Postman tests (4 tests, 12 assertions)
  - Frontend: Axios+Cheerio HTTP tests (5 tests)
  - Tests run against staging environment (`192.168.56.11`)
- **⚠️ Production deployment BLOCKED if E2E tests fail**
- Images pushed to:
  - `minfranco/e4l-backend-prod:release`
  - `minfranco/e4l-frontend-prod:release`
- Deployed to: `192.168.56.12` (e4l-prod VM)

### CODE FREEZE Workflow:
1. Develop and push to **dev** branch (deploys to staging)
2. Test and validate on staging environment
3. **CODE FREEZE:** Stop dev branch commits when ready for production
4. Create merge request: `dev → main`
5. Merge triggers E2E tests on staging
6. ✅ If E2E tests **pass** → Production image built and deployed
7. ❌ If E2E tests **fail** → Production deployment blocked, fix issues, repeat

## 🔑 SSH Access to VMs

```bash
# Staging VM
ssh -i ~/.ssh/devops_stage vagrant@192.168.56.11 -p 2222

# Production VM
ssh -i ~/.ssh/devops_prod vagrant@192.168.56.12 -p 2223
```

## 🎯 Manual Pipeline Trigger

1. Go to repository (e.g., `http://localhost:8929/testdev/backend`)
2. Navigate to: **Build > Pipelines**
3. Click **"Run Pipeline"**
4. Select branch (**dev** or **main**)
5. Click **"Run Pipeline"**

## 🧪 Testing

### Backend Tests
```bash
cd repos/backende4l

# Unit tests (20 tests)
./gradlew test

# Integration tests (4 tests)
./gradlew integrationTest

# E2E tests (4 tests, 12 assertions)
docker run --rm \
  -v "$(pwd)/tests/postman:/etc/newman" \
  postman/newman:5-alpine \
  run e4l-backend-e2e.collection.json \
  --env-var "API_URL=http://192.168.56.11:8084"
```

### Frontend Tests
```bash
cd repos/frontende4l

# Install dependencies
npm ci

# Unit tests (Jest)
npm run test:unit:ci

# Integration tests (React Testing Library)
npm run test:integration:ci

# E2E tests (Axios + Cheerio HTTP tests)
E2E_BASE_URL="http://192.168.56.11:8082" npm run test:e2e
```

## 🧹 Cleanup

### Stop and Destroy VMs
```bash
# Staging VM
cd ansible-stage && vagrant destroy -f

# Production VM
cd ansible-prod && vagrant destroy -f
```

### Stop GitLab Runner
```bash
docker-compose down
```

### Clean SSH Keys (optional)
```bash
rm ~/.ssh/devops_stage ~/.ssh/devops_stage.pub
rm ~/.ssh/devops_prod ~/.ssh/devops_prod.pub
```

### Full Cleanup Script
```bash
./scripts/cleanup.sh
```

## 🛠️ Troubleshooting

### Common Issues

**VM provisioning fails:**
- Ensure VirtualBox and Vagrant are installed correctly
- Check available disk space (need 100GB)
- Verify network connectivity for package downloads

**Pipeline fails on Docker build:**
- Verify Docker Hub credentials in GitLab CI/CD variables
- Check Docker Hub rate limits
- Ensure images names match your Docker Hub account

**E2E tests fail:**
- Verify staging VM is running: `cd ansible-stage && vagrant status`
- Check staging services: `ssh -i ~/.ssh/devops_stage vagrant@192.168.56.11 -p 2222 "docker ps"`
- Verify network connectivity to staging VM

**SSH connection refused:**
- Check VM status: `vagrant status`
- Verify SSH keys were generated: `ls -la ~/.ssh/devops_*`
- Ensure VM IP addresses match configuration

**Database connection errors:**
- Verify MariaDB container is running in VMs
- Check database credentials match configuration
- Restart database container if needed

## 📚 Additional Resources

- **Architecture Diagram:** See `architecture_diagram.txt` for detailed diagrams
- **Test Scenarios:** See `scenarios.txt` for comprehensive test scenarios
- **Backend API Docs:** Available at staging/prod backend at `/swagger-ui.html`

## � Future Improvements

### 1. Fine-Grained Container Tagging Strategy
Currently, images use simple tags like `latest` and `release`. Future improvements include:
- **Commit-based tags:** Tag images with Git commit SHA (e.g., `minfranco/e4l-backend:abc1234`)
- **Semantic versioning:** Tag production releases with version numbers (e.g., `v1.0.0`, `v1.0.1`)
- **Multi-tag strategy:** Push multiple tags simultaneously (e.g., `latest`, `v1.0.0`, `commit-abc1234`)
- **Immutable tags:** Use commit SHAs for traceability and rollback capabilities
- **Benefits:** Better version tracking, easier rollbacks, improved audit trail, simplified debugging

### 2. Monitoring and Observability Stack
Add comprehensive monitoring and logging infrastructure:
- **Prometheus:** Metrics collection from application containers and VMs
- **Grafana:** Real-time dashboards for application health, resource usage, and performance
- **Loki/ELK Stack:** Centralized logging aggregation from all services
- **Alert Manager:** Automated alerts for critical issues (service down, high error rates, resource exhaustion)
- **Health checks:** Automated endpoint monitoring with uptime tracking
- **Benefits:** Proactive issue detection, performance insights, faster troubleshooting, SLA monitoring

### 3. Automated Rollback Mechanisms
Implement safety nets for failed deployments:
- **Health check validation:** Verify service health after deployment (HTTP endpoints, database connectivity)
- **Smoke tests:** Run lightweight tests post-deployment to validate core functionality
- **Auto-rollback triggers:** Automatically revert to previous version if health checks fail
- **Docker image retention:** Keep last N successful images for quick rollback
- **Deployment strategies:** Implement blue-green or canary deployments for zero-downtime updates
- **Benefits:** Reduced downtime, faster recovery from bad deployments, improved reliability

## �📝 Summary

This platform provides:
- ✅ Automated CI/CD pipelines with GitLab
- ✅ Separate staging and production environments (VMs)
- ✅ Comprehensive testing (unit, integration, E2E)
- ✅ Quality gates blocking production on test failures
- ✅ Code freeze workflow for controlled production releases
- ✅ Docker containerization for consistency
- ✅ SSH-based deployment to VMs
- ✅ Branch-based deployment strategy

**Test Coverage:**
- Backend: 20 unit tests + 4 integration tests + 4 E2E tests
- Frontend: Jest unit tests + React Testing Library integration tests + Axios+Cheerio E2E tests
