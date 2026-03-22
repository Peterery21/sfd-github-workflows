# sfd-github-workflows

Reusable GitHub Actions workflows for SFD ERP-UEMOA.
Equivalent of `jenkins-blueprints@master` shared library.

## Available Templates

| Template | Usage |
|----------|-------|
| `java-microservice.yml` | All 18 Java Spring Boot microservices |
| `angular-pipeline.yml` | Angular frontends (sfd-angular, sfd-portail-client-angular) |
| `maven-library.yml` | Maven libraries without Docker (sfd-demo-data, etc.) |

## Usage Example (Java Microservice)

```yaml
# .github/workflows/ci.yml in your service repo
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master, main]
  pull_request:
    branches: [develop, master, main]
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

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/java-microservice.yml@main
    with:
      app-name: sfd-epargne
      dev-port: 4596
      prod-port: 4612
      java-internal-port: 4596
      health-path: /api/epargne/actuator/health
      release-branch: master
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
    secrets: inherit
```

## Release Workflow

To release a new version, go to **Actions → CI/CD Pipeline → Run workflow**:
- Check "Déclencher un release"
- Select release type: `patch` (1.2.3 → 1.2.4), `minor` (1.2.3 → 1.3.0), `major` (1.2.3 → 2.0.0)

What happens:
1. Computes new version from `pom.xml` current version + release type
2. Bumps `pom.xml` to new version (e.g. `1.2.4`)
3. Builds + tests (same binary, no double build)
4. Pushes JAR to Nexus **releases** repo via `deploy:deploy-file`
5. Builds Docker image tagged `:1.2.4` + `:latest` and pushes
6. Git commit + tag `v1.2.4` + push to branch
7. Bumps `pom.xml` to next SNAPSHOT (`1.2.5-SNAPSHOT`) + commit
8. Deploys to prod VPS + health check

> **Requires `GH_PAT` secret** (repo-level or org-level) with `contents:write` permission
> for git push. Without it, falls back to `github.token` which may fail on protected branches.

## Single Build / No Double Compile

In the normal (non-release) flow, the Nexus SNAPSHOT deploy uses `deploy:deploy-file`
to push **the exact JAR already produced by `mvn verify`** — no recompilation.
This mirrors the Jenkins `stash/unstash` pattern.

## Required Secrets (set on organization level)

| Secret | Description |
|--------|-------------|
| `REGISTRY_URL` | Docker registry URL |
| `REGISTRY_USER` | Registry username |
| `REGISTRY_PASSWORD` | Registry password |
| `SONAR_TOKEN` | SonarQube token |
| `SONAR_HOST_URL` | SonarQube URL |
| `NEXUS_URL` | Nexus URL |
| `NEXUS_USER` | Nexus username |
| `NEXUS_PASSWORD` | Nexus password |
| `VPS_HOST` | VPS IP address |
| `VPS_USER` | VPS SSH user |
| `VPS_SSH_KEY` | VPS SSH private key |
| `GH_PAT` | GitHub PAT with `contents:write` (for release commits) |

## Service Port Reference

| Service | Dev Port | Prod Port |
|---------|----------|-----------|
| sfd-commun | 4597 | 4611 |
| sfd-client | 4598 | 4613 |
| sfd-epargne | 4596 | 4612 |
| sfd-caisse | 4599 | 4614 |
| sfd-comptabilite | 4595 | 4615 |
| sfd-credit | 4594 | 4618 |
| sfd-immobilisation | 4600 | 4616 |
| sfd-reporting | 4593 | 4617 |
| sfd-stock | 4601 | 4619 |
| sfd-rh | 4602 | 4620 |
| sfd-paie | 4603 | 4621 |
| sfd-budget | 4604 | 4622 |
| sfd-transfert | 4605 | 4623 |
| sfd-suivi-evaluation | 4608 | 4624 |
| sfd-commercial | 4609 | 4625 |
| sfd-portail-client | 4610 | 4629 |
| sfd-agent-mobile | 4627 | 4628 |
