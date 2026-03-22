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
    secrets: inherit
```

## Required Secrets (set on each repo)

| Secret | Description |
|--------|-------------|
| `REGISTRY_URL` | Docker registry URL |
| `REGISTRY_USER` | Registry username |
| `REGISTRY_PASSWORD` | Registry password |
| `SONAR_TOKEN` | SonarQube token |
| `SONAR_HOST_URL` | SonarQube URL |
| `VPS_HOST` | VPS IP address |
| `VPS_USER` | VPS SSH user |
| `VPS_SSH_KEY` | VPS SSH private key |
