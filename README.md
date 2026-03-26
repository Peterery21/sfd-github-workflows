# sfd-github-workflows

Reusable GitHub Actions workflows for all Kodzotech/Evoltica projects.
Equivalent of the former `jenkins-blueprints@master` shared library — migrated March 2026.

**27 repositories** consume these workflows. All run on `ubuntu-latest` (GitHub-hosted runners).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Available Workflows](#available-workflows)
- [SSH Bridge — How Deployment Works](#ssh-bridge--how-deployment-works)
- [Docker Registry & Image Lifecycle](#docker-registry--image-lifecycle)
- [Network Topology](#network-topology)
- [Usage — Java Microservice](#usage--java-microservice)
- [Usage — Angular Application](#usage--angular-application)
- [Usage — Flutter Application](#usage--flutter-application)
- [Usage — Maven Library](#usage--maven-library)
- [Release Workflow](#release-workflow)
- [Rollback Mechanism](#rollback-mechanism)
- [Required Secrets](#required-secrets)
- [Service Port Reference](#service-port-reference)
- [Registry Cleanup](#registry-cleanup)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                GitHub Actions  (ubuntu-latest)                   │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌────────────────────┐         │
│  │  Build   │───▸│  Docker  │───▸│ Push to Private    │         │
│  │  (Maven  │    │  Build   │    │ Registry           │         │
│  │  / npm)  │    │          │    │ (registry.evoltica) │         │
│  └──────────┘    └──────────┘    └─────────┬──────────┘         │
│                                            │                    │
│  ┌─────────────────────────────────────────┼──────────────────┐ │
│  │            SSH Connection               │                  │ │
│  │    (VPS_SSH_KEY → deploy_key)           ▼                  │ │
│  │                                                            │ │
│  │  1. docker pull $IMAGE                                     │ │
│  │  2. docker stop/rm old container                           │ │
│  │  3. docker run -d --network --restart=always               │ │
│  │  4. Health check (curl → HTTP 200)                         │ │
│  │  5. Rollback if health check fails                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  E2E Tests (Angular only)                                │   │
│  │  SSH tunnel: -L PORT:localhost:PORT                       │   │
│  │  Playwright tests via localhost → tunnel → VPS container  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ SSH (port 22)
                         ▼
              ┌─────────────────────┐
              │  VPS (Hostinger)    │
              │  82.180.131.215     │
              │                     │
              │  ┌───────────────┐  │
              │  │ prod-network  │  │    ┌──────────────────────┐
              │  │  sql-server   │  │    │  Docker Registry     │
              │  │  sfd-epargne  │  │    │  :5000 (API)         │
              │  │  sfd-client   │  │    │  :9900 (UI)          │
              │  │  ...          │  │    │  registry.evoltica    │
              │  └───────────────┘  │    │  technologes.com     │
              │                     │    └──────────────────────┘
              │  ┌───────────────┐  │
              │  │ dev-network   │  │
              │  │  sql-server-  │  │
              │  │    dev        │  │
              │  │  sfd-*-dev   │  │
              │  │  ...          │  │
              │  └───────────────┘  │
              └─────────────────────┘
```

### Flow Summary

1. **Push to `develop`** → Build → Docker image tagged `{N}-SNAPSHOT` → Deploy to **dev** container
2. **Push to `master`/`main`** → Build → Docker image tagged `latest` → Deploy to **prod** container
3. **Manual release** → Version bump → Nexus releases + Docker `:version` tag → Deploy to prod

---

## Available Workflows

| Workflow | File | Used By | Purpose |
|----------|------|---------|---------|
| **Java Microservice** | `java-microservice.yml` | 17 repos | Build + Test + SonarQube + Nexus + Docker + Deploy + Health Check + Rollback |
| **Angular Pipeline** | `angular-pipeline.yml` | 4 repos | Build + Docker + Deploy + Health Check + Rollback + E2E Playwright |
| **Flutter Pipeline** | `flutter-pipeline.yml` | 2 repos | Build + Analyze + Test (no deploy) |
| **Maven Library** | `maven-library.yml` | 3 repos | Build + Test + SonarQube + Nexus SNAPSHOT (no Docker, no deploy) |
| **Nexus Upload** | `nexus-upload.yml` | Internal | One-shot: migrate legacy Jenkins JARs to Nexus |

---

## SSH Bridge — How Deployment Works

The core of the CI/CD system is the **SSH bridge**: GitHub Actions (running on GitHub's cloud) connects to the VPS via SSH to deploy containers.

### Step 1: SSH Key Setup

Every deploy job starts by writing the SSH key:

```yaml
- name: Setup SSH key
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.VPS_SSH_KEY }}" > ~/.ssh/deploy_key
    chmod 600 ~/.ssh/deploy_key
    ssh-keyscan -H ${{ secrets.VPS_HOST }} >> ~/.ssh/known_hosts 2>/dev/null || true
```

| Secret | Content | Source |
|--------|---------|--------|
| `VPS_SSH_KEY` | Full private key content (PEM format) | `~/.ssh/hostinger_vps` |
| `VPS_HOST` | `82.180.131.215` | Hostinger VPS IP |
| `VPS_USER` | `root` | VPS login user |

The `ssh-keyscan` populates `known_hosts` dynamically — no manual host key management needed.

### Step 2: Remote Command Execution

The deploy step sends a **heredoc script** via SSH:

```yaml
- name: Deploy container via SSH
  run: |
    ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=no \
      ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }} \
      CONTAINER="$CONTAINER" PORT="$PORT" IMAGE="$IMAGE" \
      bash << 'ENDSSH'
      
      # This entire script runs ON THE VPS
      docker pull "${IMAGE}"
      docker stop "${CONTAINER}" 2>/dev/null || true
      docker rm "${CONTAINER}" 2>/dev/null || true
      docker run -d --name "${CONTAINER}" --restart=always \
        --network "${NETWORK}" -p "${PORT}" "${IMAGE}"
      
    ENDSSH
```

**Key points:**
- Environment variables are passed as SSH command prefix (`CONTAINER="..." PORT="..."`)
- The heredoc `<< 'ENDSSH'` prevents local variable expansion — everything runs server-side
- Docker commands execute directly on the VPS Docker daemon

### Step 3: Health Check

**Angular apps** — health check runs **via SSH** (ports not publicly exposed):
```bash
ssh ... PORT="$PORT" bash << 'HEALTHCHECK'
  for i in $(seq 1 15); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:${PORT}")
    [ "$STATUS" = "200" ] && exit 0
    sleep 10
  done
  exit 1  # → triggers rollback
HEALTHCHECK
```

**Java microservices** — health check runs **from GitHub runner** (actuator ports are accessible):
```bash
for i in $(seq 1 15); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    "http://${{ secrets.VPS_HOST }}:${PORT}${{ inputs.health-path }}")
  [ "$STATUS" = "200" ] && exit 0
  sleep 10
done
```

### SSH Tunnel for E2E Tests (Angular only)

VPS ports may be firewalled from external IPs. Playwright tests use an **SSH tunnel** to reach the deployed app:

```bash
# Opens tunnel: GH runner localhost:4621 → VPS localhost:4621
ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=no \
  -L ${PORT}:localhost:${PORT} \
  -N -f \                        # -N = no remote command, -f = background
  ${{ secrets.VPS_USER }}@${{ secrets.VPS_HOST }}

# Playwright then tests against http://localhost:4621
npx playwright test
```

```
┌─────────────────┐          SSH tunnel           ┌──────────────┐
│  GH Actions     │  localhost:4621 ──────────▸   │  VPS         │
│  Playwright     │  ◂────────────────────────    │  Container   │
│  (chromium)     │          port 22              │  :4621       │
└─────────────────┘                               └──────────────┘
```

---

## Docker Registry & Image Lifecycle

### Registry Infrastructure

| Component | URL | Port (VPS) |
|-----------|-----|------------|
| Registry API | `registry.evolticatechnologies.com` | `:5000` |
| Registry UI | (internal only) | `:9900` |

Docker-compose: `/home/ubuntu/pierre/registry/docker-compose.yml`

### Image Tagging Strategy

| Branch | Image Tag | Container Name | Port |
|--------|-----------|----------------|------|
| `develop` (or non-release) | `{registry}/{app-name}-dev:{run_number}-SNAPSHOT` | `{app-name}-dev` | `dev-port:internal` |
| `master`/`main` | `{registry}/{app-name}:latest` | `{app-name}` | `prod-port:internal` |
| Release | `{registry}/{app-name}:{version}` + `:latest` | `{app-name}` | `prod-port:internal` |

**Example** — `sfd-epargne` push to `develop` (run #42):
```
Image: registry.evolticatechnologies.com/sfd-epargne-dev:42-SNAPSHOT
Container: sfd-epargne-dev
Port: 4596:4596
Network: dev-network
```

### Build & Push Flow

```yaml
# 1. Login to private registry
echo "$REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u $REGISTRY_USER --password-stdin

# 2. Build image (uses repo's Dockerfile)
docker build -t $IMAGE .

# 3. Push to registry
docker push $IMAGE
```

For **Java microservices**, the Docker build uses a pre-built JAR artifact:
```
Job 1 (build)  → mvn verify → uploads JAR via actions/upload-artifact
Job 2 (docker) → downloads JAR via actions/download-artifact → docker build → push
```
This ensures a **single build** — the exact JAR that passed tests is the one deployed.

### Automatic Cleanup

The registry auto-cleans old images weekly (every Sunday 3am):
- Keeps **3 most recent tags** per repo (`latest` + 2 newest `N-SNAPSHOT`)
- Runs garbage-collect to reclaim disk space
- Config: `/home/ubuntu/pierre/registry/cleanup.sh` + cron

Manual trigger:
```bash
cd /home/ubuntu/pierre/registry && docker compose --profile cleanup run --rm cleanup
```

---

## Network Topology

### Docker Networks

All containers are placed on isolated Docker networks for inter-service communication:

| Network | Used By | DB Container |
|---------|---------|------------|
| `prod-network` | All production containers | `sql-server` |
| `dev-network` | All development containers | `sql-server-dev` |

### Network Auto-Detection

The workflow auto-detects the correct network by inspecting the SQL Server container:

```bash
# Production → get network from sql-server
DOCKER_NETWORK=$(docker inspect sql-server \
  --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' | awk '{print $1}')

# Development → get network from sql-server-dev
DOCKER_NETWORK=$(docker inspect sql-server-dev \
  --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' | awk '{print $1}')

# Fallback → bridge
[ -z "$DOCKER_NETWORK" ] && DOCKER_NETWORK="bridge"
```

For projects using **MySQL** instead of SQL Server (e.g., `lecturoom-api`), the network must be specified explicitly:
```yaml
docker-network: "prod-network"
```

### Inter-Service Communication

Services on the same Docker network reach each other **by container hostname**:

```
http://sfd-commun-dev:4597/api/preference   # dev
http://sfd-commun:4597/api/preference       # prod
```

The `{DEV_SUFFIX}` placeholder in `extra-env` handles this automatically:
```yaml
extra-env: |
  PREFERENCE_SERVICE_URL=http://sfd-commun{DEV_SUFFIX}:4597/api/preference
```
- On `develop` branch → `{DEV_SUFFIX}` = `-dev`
- On `master`/`main` branch → `{DEV_SUFFIX}` = `` (empty)

---

## Usage — Java Microservice

### Minimal ci.yml

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master, main]
    paths-ignore: ['*.md', 'docs/**', '.gitignore', 'LICENSE']
  pull_request:
    branches: [develop, master, main]
    paths-ignore: ['*.md', 'docs/**', '.gitignore', 'LICENSE']
  workflow_dispatch:
    inputs:
      release:
        description: "Déclencher un release"
        type: boolean
        default: false
      release-type:
        description: "Type de release"
        type: choice
        options: [patch, minor, major]
        default: patch

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/java-microservice.yml@main
    with:
      app-name: "sfd-epargne"
      dev-port: 4596
      prod-port: 4612
      java-internal-port: 4596
      health-path: "/api/epargne/actuator/health"
      release-branch: "master"
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
    secrets: inherit
```

### With extra-env (inter-service URLs)

```yaml
# sfd-rh-service — depends on sfd-commun-service
with:
  app-name: "sfd-rh"
  dev-port: 4602
  prod-port: 4618
  java-internal-port: 4602
  health-path: "/api/rh/actuator/health"
  release-branch: "main"
  release: ${{ inputs.release || false }}
  release-type: ${{ inputs.release-type || 'patch' }}
  extra-env: |
    PREFERENCE_SERVICE_URL=http://sfd-commun{DEV_SUFFIX}:4597/api/preference
```

### With custom volumes and memory

```yaml
# sfd-stock-service — needs upload volume + more memory
with:
  app-name: "sfd-stock"
  dev-port: 4606
  prod-port: 4624
  java-internal-port: 4606
  health-path: "/api/stock/actuator/health"
  release-branch: "master"
  release: ${{ inputs.release || false }}
  release-type: ${{ inputs.release-type || 'patch' }}
  upload-folder: "/app/uploads"
  memory-limit: "2g"
  volumes: |
    {PROFILE}-stock-uploads:/app/uploads
```

### With explicit Docker network (non-SQL Server projects)

```yaml
# lecturoom-api — uses MySQL, not on SQL Server network
with:
  app-name: "lecturoom-api"
  docker-network: "prod-network"
  # ... other inputs
```

### All Java Microservice Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `app-name` | string | ✅ | — | Application name (e.g., `sfd-epargne`) |
| `dev-port` | number | ✅ | — | Host port for dev environment |
| `prod-port` | number | ✅ | — | Host port for prod environment |
| `java-internal-port` | number | ✅ | — | Spring Boot internal port |
| `health-path` | string | ✅ | — | Health check URL path (e.g., `/api/epargne/actuator/health`) |
| `release-branch` | string | | `master` | Branch that triggers prod deployment |
| `jvm-opts` | string | | `-Xms384m -Xmx768m -XX:MaxMetaspaceSize=192m -XX:+UseG1GC ...` | JVM arguments |
| `memory-limit` | string | | `1g` | Docker memory limit |
| `maven-goals` | string | | `clean verify` | Maven build goals |
| `sonar-enabled` | boolean | | `true` | Enable SonarQube analysis |
| `upload-folder` | string | | `/upload` | Application upload directory |
| `release` | boolean | | `false` | Trigger a release (version bump + tag + Nexus + Docker) |
| `release-type` | string | | `patch` | Release type: `patch`, `minor`, `major` |
| `extra-env` | string | | `""` | Extra env vars (one per line). Supports `{DEV_SUFFIX}` placeholder |
| `volumes` | string | | `""` | Docker volumes (one per line). Supports `{PROFILE}` placeholder |
| `docker-network` | string | | `""` | Explicit Docker network name. If empty, auto-detects from SQL Server |

### Pipeline Jobs

```
┌───────────────────────────────────────────────────────────────────────┐
│                     Normal Flow (release=false)                       │
│                                                                       │
│  Job 1: Build & Test (20 min timeout)                                │
│  ├─ Checkout                                                         │
│  ├─ Java 17 + Maven cache                                           │
│  ├─ Configure Maven settings (Nexus mirror)                          │
│  ├─ mvn clean verify                                                 │
│  ├─ SonarQube analysis (continue-on-error)                           │
│  ├─ Deploy SNAPSHOT to Nexus (deploy:deploy-file, same JAR)          │
│  └─ Upload JAR artifact                                              │
│                          │                                            │
│  Job 2: Docker Build & Push (15 min timeout)                         │
│  ├─ Download JAR artifact                                            │
│  ├─ Compute image tag (dev: N-SNAPSHOT, prod: latest)                │
│  ├─ docker login → docker build → docker push                       │
│  └─ Output: container-name, port-mapping, spring-profile             │
│                          │                                            │
│  Job 3: Deploy to VPS (10 min timeout)                               │
│  ├─ Setup SSH key                                                    │
│  ├─ SSH → pull, stop old, save rollback, run new container           │
│  ├─ Health check (15 attempts × 10s) via HTTP                       │
│  └─ On failure: SSH → rollback to previous image                     │
│                                                                       │
│  Service container: RabbitMQ 3 (vhost: /sfd) — for integration tests │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Usage — Angular Application

### ci.yml Example

```yaml
jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/angular-pipeline.yml@main
    with:
      app-name: "sfd-angular"
      dev-port: 4621
      prod-port: 4620
      release-branch: "master"
      e2e-enabled: true
      e2e-command: "npm run e2e:chromium"
      json-server-port: 3000
    secrets: inherit
```

### All Angular Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `app-name` | string | ✅ | — | Application name |
| `dev-port` | number | ✅ | — | Host port for dev |
| `prod-port` | number | ✅ | — | Host port for prod |
| `release-branch` | string | | `master` | Prod branch |
| `e2e-enabled` | boolean | | `false` | Enable Playwright E2E tests after deploy |
| `e2e-command` | string | | `npm run e2e:chromium` | Playwright run command |
| `e2e-base-url` | string | | `""` | Override E2E base URL (default: auto from port) |
| `json-server-port` | number | | `0` | JSON Server port (0 = disabled). Mapped as `3588:3000` (prod) or `3589:3000` (dev) |
| `internal-port` | number | | `80` | Container internal port. `80` for Nginx SPA, `4000` for SSR Node |

### Pipeline Jobs

```
Job 1: Build Angular (30 min)
├─ Node.js 20 + npm cache
├─ npm install --legacy-peer-deps
├─ ng build --configuration {development|production}
├─ docker build --build-arg APP_ENV={dev|prod}
└─ docker push

Job 2: Deploy to VPS (15 min)
├─ SSH → docker pull → stop old → run new
├─ Network: {APP_ENV}-network (created if missing)
├─ Health check via SSH (curl localhost)
└─ Rollback on failure

Job 3: E2E Playwright (30 min, optional)
├─ SSH tunnel → localhost:PORT
├─ npx playwright install chromium --with-deps
├─ Run E2E command
└─ Upload playwright-report + e2e-results artifacts
```

### SSR vs SPA

Most Angular apps serve static files via **Nginx on port 80** (SPA).
`lifeschoolbank-angular` uses **SSR with Node on port 4000**:

```yaml
with:
  internal-port: 4000  # Override default 80
```

---

## Usage — Flutter Application

### ci.yml Example

```yaml
jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/flutter-pipeline.yml@main
    with:
      app-name: "sfd_portail_client"
      shared-package-repo: "Peterery21/sfd_shared"
      shared-package-path: "sfd_shared"
    secrets: inherit
```

### All Flutter Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `app-name` | string | ✅ | — | Flutter app name |
| `flutter-version` | string | | `3.41.4` | Flutter SDK version |
| `shared-package-repo` | string | | `""` | GitHub repo for shared package (e.g., `Peterery21/sfd_shared`) |
| `shared-package-path` | string | | `sfd_shared` | Local directory name for the shared package |
| `run-tests` | boolean | | `true` | Run `flutter test` |

### Pipeline

```
Job 1: Build & Analyze (15 min)
├─ Checkout app
├─ Checkout shared package (if configured) → pubspec_overrides.yaml
├─ Flutter SDK setup (cached)
├─ flutter pub get
├─ flutter analyze --no-fatal-infos --no-fatal-warnings
└─ flutter test
```

> Flutter apps are **build-only** — no Docker, no VPS deployment. APK/IPA builds are done locally.

---

## Usage — Maven Library

### ci.yml Example

```yaml
jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/maven-library.yml@main
    with:
      app-name: "sfd-integration-api"
      sonar-enabled: true
      nexus-deploy: true
    secrets: inherit
```

### All Maven Library Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `app-name` | string | ✅ | — | Library name |
| `maven-goals` | string | | `clean verify` | Maven build goals |
| `sonar-enabled` | boolean | | `true` | Enable SonarQube |
| `nexus-deploy` | boolean | | `true` | Deploy SNAPSHOT to Nexus |

### Pipeline

```
Job 1: Build & Test (15 min)
├─ Java 17 + Maven cache
├─ Configure Maven settings (Nexus mirror)
├─ mvn clean verify
├─ SonarQube analysis (if enabled)
└─ mvn deploy (SNAPSHOT, on develop/master only)
```

> Maven libraries have **no Docker, no VPS deployment**. They produce JARs consumed by microservices.

---

## Release Workflow

Available for **Java Microservices** only. Triggered manually:

**Actions → CI/CD Pipeline → Run workflow → Check "Déclencher un release" → Select type**

| Type | Example |
|------|---------|
| `patch` | `1.2.3-SNAPSHOT` → release `1.2.4` → next `1.2.5-SNAPSHOT` |
| `minor` | `1.2.3-SNAPSHOT` → release `1.3.0` → next `1.3.1-SNAPSHOT` |
| `major` | `1.2.3-SNAPSHOT` → release `2.0.0` → next `2.0.1-SNAPSHOT` |

### Release Steps

1. Compute new version from `pom.xml` current version + release type
2. `mvn versions:set` to bump pom.xml
3. `mvn clean verify` — full build + tests
4. SonarQube analysis
5. `deploy:deploy-file` → push JAR to Nexus **releases** (same binary, no recompile)
6. `docker build` → tag with `:version` AND `:latest` → push both
7. Git commit: `release: bump version to X.Y.Z [skip ci]`
8. Git tag: `vX.Y.Z`
9. `mvn versions:set` → next SNAPSHOT
10. Git commit: `chore: prepare next development version [skip ci]`
11. `git push` commits + tag
12. Deploy release image to prod VPS + health check

> **Requires `GH_PAT` secret** with `contents:write` for git push. Standard `github.token` may fail on protected branches.

---

## Rollback Mechanism

Every deployment saves a rollback file **before** stopping the old container:

```bash
# Saved to /tmp/rollback_${CONTAINER}.env on VPS
OLD_IMAGE=registry.evoltica.../sfd-epargne-dev:41-SNAPSHOT
PROFILE=dev
PORT=4596:4596
MEMORY_LIMIT=1g
DOCKER_NETWORK=dev-network
VOLUMES_ENCODED=...
EXTRA_ENV_ENCODED=...
```

If the health check fails after 15 attempts (2.5 minutes):

```bash
# Automatic rollback — restores exact previous configuration
source /tmp/rollback_${CONTAINER}.env
docker stop "${CONTAINER}" && docker rm "${CONTAINER}"
docker run -d --name "${CONTAINER}" \
  --restart=always --memory="${MEMORY_LIMIT}" \
  --network="${DOCKER_NETWORK}" -p "${PORT}" \
  -e SPRING_PROFILES_ACTIVE="${PROFILE}" \
  "${OLD_IMAGE}"
```

The rollback preserves **all parameters**: memory, CPU, volumes, env vars, network — identical to the previous working configuration.

---

## Required Secrets

Set these on each repository (or at organization level):

| Secret | Description | Used By |
|--------|-------------|---------|
| `REGISTRY_URL` | Docker registry URL (`registry.evolticatechnologies.com`) | Java, Angular |
| `REGISTRY_USER` | Registry username | Java, Angular |
| `REGISTRY_PASSWORD` | Registry password | Java, Angular |
| `SONAR_TOKEN` | SonarQube authentication token | Java, Maven Lib |
| `SONAR_HOST_URL` | SonarQube URL (`https://sonarqube.evolticatechnologies.com/`) | Java, Maven Lib |
| `NEXUS_URL` | Nexus URL (`https://nexus.evolticatechnologies.com/`) | Java, Maven Lib |
| `NEXUS_USER` | Nexus username | Java, Maven Lib |
| `NEXUS_PASSWORD` | Nexus password | Java, Maven Lib |
| `VPS_HOST` | VPS IP address (`82.180.131.215`) | Java, Angular |
| `VPS_USER` | VPS SSH user (`root`) | Java, Angular |
| `VPS_SSH_KEY` | VPS SSH private key (full PEM content) | Java, Angular |
| `GH_PAT` | GitHub PAT with `contents:write` (for release git push + Flutter private repos) | Java (release), Flutter |

---

## Service Port Reference

### Java Microservices (SFD)

| Service | App Name | Dev Port | Prod Port | Internal Port | Health Path |
|---------|----------|----------|-----------|---------------|-------------|
| sfd-commun-service | sfd-commun | 4597 | 4611 | 4597 | `/api/preference/actuator/health` |
| sfd-client-service | sfd-client | 4598 | 4613 | 4598 | `/api/client/actuator/health` |
| sfd-epargne-service | sfd-epargne | 4596 | 4612 | 4596 | `/api/epargne/actuator/health` |
| sfd-caisse-service | sfd-caisse | 4599 | 4614 | 4599 | `/api/caisse/actuator/health` |
| sfd-comptabilite-service | sfd-comptabilite | 4595 | 4615 | 4595 | `/api/comptabilite/actuator/health` |
| sfd-credit-service | sfd-credit | 4594 | 4618 | 4594 | `/api/credit/actuator/health` |
| sfd-immobilisation-service | sfd-immobilisation | 4600 | 4616 | 4600 | `/api/immobilisation/actuator/health` |
| sfd-reporting-service | sfd-reporting | 4593 | 4617 | 4593 | `/api/reporting/actuator/health` |
| sfd-stock-service | sfd-stock | 4606 | 4624 | 4606 | `/api/stock/actuator/health` |
| sfd-rh-service | sfd-rh | 4602 | 4618 | 4602 | `/api/rh/actuator/health` |
| sfd-paie-service | sfd-paie | 4603 | 4621 | 4603 | `/api/paie/actuator/health` |
| sfd-budget-service | sfd-budget | 4604 | 4622 | 4604 | `/api/budget/actuator/health` |
| sfd-transfert-service | sfd-transfert | 4605 | 4623 | 4605 | `/api/transfert/actuator/health` |
| sfd-suivi-evaluation-service | sfd-suivi-evaluation | 4608 | 4624 | 4608 | `/api/suivi/actuator/health` |
| sfd-commercial-service | sfd-commercial | 4609 | 4625 | 4609 | `/api/commercial/actuator/health` |
| sfd-portail-client-service | sfd-portail-client | 4629 | 4630 | 4611 | `/api/portail/actuator/health` |
| sfd-agent-mobile-service | sfd-agent-mobile | 4627 | 4628 | 4610 | `/api/agent-mobile/actuator/health` |

### Non-SFD Java Microservices

| Service | Dev Port | Prod Port | Internal Port | Network |
|---------|----------|-----------|---------------|---------|
| lecturoom-api | 9410 | 9400 | 8075 | `prod-network` (explicit) |
| lifeschoolbank-api | 9300 | 9200 | 8080 | auto-detect |
| wayshare-api | 9110 | 9100 | 8085 | auto-detect |
| tetepleine-api | 9360 | 9380 | 8095 | auto-detect |

### Angular Applications

| App | Dev Port | Prod Port | Internal Port | E2E | JSON Server |
|-----|----------|-----------|---------------|-----|-------------|
| sfd-angular | 4621 | 4620 | 80 (Nginx) | ✅ | 3000 |
| sfd-portail-client-angular | 4589 | 4588 | 80 (Nginx) | ❌ | — |
| lecturoom-angular | 9430 | 9420 | 80 (Nginx) | ❌ | — |
| lecturoom-portail | 2941 | 2940 | 80 (Nginx) | ❌ | — |
| lifeschoolbank-angular | 9310 | 9210 | 4000 (SSR Node) | ❌ | — |
| tetepleine-angular | 9370 | 9390 | 80 (Nginx) | ❌ | — |

### Flutter Applications (no VPS deployment)

| App | Shared Package |
|-----|----------------|
| sfd-agent-mobile (Flutter) | — |
| sfd_portail_client | `Peterery21/sfd_shared` |

### Maven Libraries (no deployment)

| Library | SonarQube | Nexus |
|---------|-----------|-------|
| sfd-integration-api | ✅ | ✅ |
| sfd-demo-data | ✅ | ✅ |
| sfd-report-api | ✅ | ✅ |

---

## Registry Cleanup

The Docker Registry accumulates images with every CI build. An automated cleanup keeps only 3 tags per repo.

### How It Works

```
/home/ubuntu/pierre/registry/
├── docker-compose.yml      # Registry + UI + cleanup service
├── cleanup.sh              # Tag deletion script
├── cleanup-cron.sh         # Cron wrapper (cleanup + garbage-collect)
├── credentials.yml         # Registry config
├── htpasswd                # Auth credentials
└── registry-data/          # Image blobs (mounted volume)
```

**Cleanup service** (runs on demand via `profiles: ["cleanup"]`):
1. Lists all repositories via `GET /v2/_catalog`
2. For each repo, lists tags and sorts numerically
3. Keeps `latest` + 2 most recent `N-SNAPSHOT` tags (= 3 total)
4. Deletes old tags via `DELETE /v2/{repo}/manifests/{digest}`

**Garbage-collect** (runs after cleanup):
```bash
docker compose exec -T registry bin/registry garbage-collect \
  /etc/docker/registry/config.yml --delete-untagged
```

**Cron schedule**: Every Sunday at 3am UTC
```
0 3 * * 0 /home/ubuntu/pierre/registry/cleanup-cron.sh
```

**Manual run**:
```bash
cd /home/ubuntu/pierre/registry
docker compose --profile cleanup run --rm cleanup
docker compose exec -T registry bin/registry garbage-collect /etc/docker/registry/config.yml --delete-untagged
```

---

## Troubleshooting

### Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Deploy fails with "connection refused" | VPS SSH port blocked | Check VPS firewall, verify `ssh hostinger-vps` works |
| Health check fails (15/15 attempts) | App crash or slow startup | Check `docker logs {container}` on VPS |
| Docker push fails "unauthorized" | Registry credentials wrong | Verify `REGISTRY_URL`, `REGISTRY_USER`, `REGISTRY_PASSWORD` secrets |
| SonarQube fails | SonarQube server down | Non-blocking (`continue-on-error: true`) — build continues |
| Nexus deploy fails 502 | Nexus overloaded | Re-run the workflow — transient error |
| Container can't reach other services | Wrong Docker network | Add `docker-network: "prod-network"` to ci.yml inputs |
| Angular Nginx can't resolve backend | Backend not on same network | Both must be on same Docker network |
| E2E tests timeout | SSH tunnel not established | Check SSH key permissions, VPS connectivity |
| `{DEV_SUFFIX}` not replaced | Wrong `extra-env` format | Must be `KEY=http://host{DEV_SUFFIX}:port/path` (one per line) |
| Release push fails "protected branch" | Missing `GH_PAT` | Add `GH_PAT` secret with `contents:write` permission |

### Useful VPS Commands

```bash
# SSH to VPS
ssh hostinger-vps

# Check container status
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check container logs
docker logs --tail 50 sfd-epargne-dev

# Check container network
docker inspect sfd-epargne-dev --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'

# Check disk usage
du -sh /home/ubuntu/pierre/registry/registry-data/

# List registry images
curl -s http://localhost:5000/v2/_catalog | python3 -m json.tool

# List tags for a repo
curl -s http://localhost:5000/v2/sfd-angular-dev/tags/list | python3 -m json.tool

# Manual cleanup
cd /home/ubuntu/pierre/registry && docker compose --profile cleanup run --rm cleanup
```
