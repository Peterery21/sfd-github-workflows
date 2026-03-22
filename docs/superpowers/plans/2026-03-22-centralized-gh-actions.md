# Centralized GitHub Actions Blueprint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Répliquer exactement le pattern Jenkins `@Library('jenkins-blueprints') _ + javaMicroservicePipeline(params)` en GitHub Actions — les repos de service ont un `ci.yml` IDENTIQUE et sans config métier (zéro params service), toute la config est centralisée dans `sfd-github-workflows`.

**Architecture:** Un fichier `services-registry.yml` dans `sfd-github-workflows` contient la config complète de tous les 22 services (ports, health-path, template, release-branch). Un `dispatcher.yml` lit ce registre basé sur `github.repository` (disponible dans les reusable workflows), sélectionne le template (`java-microservice`, `angular-pipeline`, `maven-library`) et l'appelle dynamiquement. Chaque repo de service a un `ci.yml` de 20 lignes **identique** (seuls `release`/`release-type` sont en `with:` car ce sont des inputs runtime de l'utilisateur, pas de la config service).

**Tech Stack:** GitHub Actions reusable workflows (`workflow_call`), nested reusable workflows (support depuis nov 2022, max 4 niveaux), Python 3 (disponible sur ubuntu-latest) pour lire le YAML du registre, `$GITHUB_OUTPUT` pour passer les outputs entre jobs.

---

## Principe clé : équivalence Jenkins ↔ GitHub Actions

| Jenkins | GitHub Actions |
|---------|---------------|
| `jenkins-blueprints` repo | `sfd-github-workflows` repo |
| `@Library('jenkins-blueprints@master') _` | `uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main` |
| `javaMicroservicePipeline(appName: 'sfd-epargne', ...)` | `dispatcher.yml` lit `services-registry.yml` basé sur `github.repository` |
| `Jenkinsfile` identique entre services | `ci.yml` identique entre services |
| `parameters { booleanParam(name: 'Release') }` | `workflow_dispatch.inputs.release` |
| Config service dans la shared library | Config service dans `services-registry.yml` |

---

## Structure des fichiers à créer/modifier

### Dans `sfd-github-workflows` (repo central)

```
sfd-github-workflows/
├── services-registry.yml          ← NOUVEAU : config de tous les 22 services
├── .github/workflows/
│   ├── dispatcher.yml             ← NOUVEAU : lit registry → appelle le bon template
│   ├── java-microservice.yml      ← MODIFIER : inputs deviennent tous optionnels
│   ├── angular-pipeline.yml       ← MODIFIER : inputs deviennent tous optionnels
│   ├── maven-library.yml          ← MODIFIER : inputs deviennent tous optionnels
│   └── nexus-upload.yml           ← SUPPRIMER (migration one-time, hors CI régulier)
```

### Dans chaque repo de service (22 repos)

```
<service-repo>/
└── .github/workflows/
    └── ci.yml    ← REMPLACER : version zéro-config (identique pour tous)
```

**3 variantes de `ci.yml` (selon le type de service) :**
- `ci-java.yml` template → 17 repos Java (microservices)
- `ci-angular.yml` template → 2 repos Angular
- `ci-lib.yml` template → 3 libs Maven (pas de release inputs)

---

## Task 1 : Créer `services-registry.yml`

**Fichiers :**
- Créer : `services-registry.yml` (racine du repo `sfd-github-workflows`)

Ce fichier est la **source de vérité unique** pour tous les paramètres de déploiement. La clé est `<org>/<repo>` pour matcher `github.repository` exactement.

- [ ] **Créer `services-registry.yml`**

```yaml
# services-registry.yml
# Clé = github.repository (org/repo) — source de vérité unique pour tous les services SFD
# Modifier ICI quand un port ou une config change. Jamais dans les repos de service.

Peterery21/sfd-commun-service:
  template: java
  app-name: sfd-commun
  dev-port: 4597
  prod-port: 4611
  internal-port: 4597
  health-path: /api/preference/actuator/health
  release-branch: master

Peterery21/sfd-epargne-service:
  template: java
  app-name: sfd-epargne
  dev-port: 4596
  prod-port: 4612
  internal-port: 4596
  health-path: /api/epargne/actuator/health
  release-branch: master

Peterery21/sfd-client-service:
  template: java
  app-name: sfd-client
  dev-port: 4598
  prod-port: 4613
  internal-port: 4598
  health-path: /api/client/actuator/health
  release-branch: master

Peterery21/sfd-caisse-service:
  template: java
  app-name: sfd-caisse
  dev-port: 4599
  prod-port: 4614
  internal-port: 4599
  health-path: /api/caisse/actuator/health
  release-branch: master

Peterery21/sfd-comptabilite-service:
  template: java
  app-name: sfd-comptabilite
  dev-port: 4595
  prod-port: 4615
  internal-port: 4595
  health-path: /api/comptabilite/actuator/health
  release-branch: master

Peterery21/sfd-credit-service:
  template: java
  app-name: sfd-credit
  dev-port: 4594
  prod-port: 4618
  internal-port: 4594
  health-path: /api/credit/actuator/health
  release-branch: master

Peterery21/sfd-stock-service:
  template: java
  app-name: sfd-stock
  dev-port: 4601
  prod-port: 4619
  internal-port: 4601
  health-path: /api/stock/actuator/health
  release-branch: master

Peterery21/sfd-commercial-service:
  template: java
  app-name: sfd-commercial
  dev-port: 4609
  prod-port: 4625
  internal-port: 4609
  health-path: /api/commercial/actuator/health
  release-branch: master

Peterery21/sfd-reporting-service:
  template: java
  app-name: sfd-reporting
  dev-port: 4593
  prod-port: 4617
  internal-port: 4593
  health-path: /api/reporting/actuator/health
  release-branch: master

Peterery21/sfd-suivi-evaluation-service:
  template: java
  app-name: sfd-suivi-evaluation
  dev-port: 4608
  prod-port: 4624
  internal-port: 4608
  health-path: /api/suivi-evaluation/actuator/health
  release-branch: master

Peterery21/sfd-portail-client-service:
  template: java
  app-name: sfd-portail-client
  dev-port: 4610
  prod-port: 4629
  internal-port: 4610
  health-path: /api/portail-client/actuator/health
  release-branch: master

Peterery21/sfd-immobilisation-service:
  template: java
  app-name: sfd-immobilisation
  dev-port: 4600
  prod-port: 4616
  internal-port: 4600
  health-path: /api/immobilisation/actuator/health
  release-branch: main

Peterery21/sfd-rh-service:
  template: java
  app-name: sfd-rh
  dev-port: 4602
  prod-port: 4620
  internal-port: 4602
  health-path: /api/rh/actuator/health
  release-branch: main

Peterery21/sfd-paie-service:
  template: java
  app-name: sfd-paie
  dev-port: 4603
  prod-port: 4621
  internal-port: 4603
  health-path: /api/paie/actuator/health
  release-branch: main

Peterery21/sfd-budget-service:
  template: java
  app-name: sfd-budget
  dev-port: 4604
  prod-port: 4622
  internal-port: 4604
  health-path: /api/budget/actuator/health
  release-branch: main

Peterery21/sfd-transfert-service:
  template: java
  app-name: sfd-transfert
  dev-port: 4605
  prod-port: 4623
  internal-port: 4605
  health-path: /api/transfert/actuator/health
  release-branch: main

Peterery21/sfd-agent-mobile-service:
  template: java
  app-name: sfd-agent-mobile
  dev-port: 4627
  prod-port: 4628
  internal-port: 4627
  health-path: /api/agent-mobile/actuator/health
  release-branch: main

Peterery21/sfd-angular:
  template: angular
  app-name: sfd-angular
  dev-port: 4621
  prod-port: 4620
  release-branch: master
  e2e-enabled: true
  e2e-command: npm run e2e:chromium
  json-server-port: 3000

Peterery21/sfd-portail-client-angular:
  template: angular
  app-name: sfd-portail-client-angular
  dev-port: 4589
  prod-port: 4588
  release-branch: master
  e2e-enabled: false
  json-server-port: 0

Peterery21/sfd-demo-data:
  template: maven-library
  app-name: sfd-demo-data
  sonar-enabled: true
  nexus-deploy: true

Peterery21/sfd-integration-api:
  template: maven-library
  app-name: sfd-integration-api
  sonar-enabled: true
  nexus-deploy: true

Peterery21/sfd-report-api:
  template: maven-library
  app-name: sfd-report-api
  sonar-enabled: true
  nexus-deploy: true
```

- [ ] **Commit**

```bash
cd /tmp/sfd-github-workflows
git add services-registry.yml
git commit -m "feat: add services-registry.yml — central config for all 22 services"
```

---

## Task 2 : Créer `dispatcher.yml`

**Fichiers :**
- Créer : `.github/workflows/dispatcher.yml`

Le dispatcher est le cœur du système. Il :
1. Checkout le repo `sfd-github-workflows` pour lire `services-registry.yml`
2. Lit la config du service appelant via `github.repository`
3. Exporte tous les paramètres dans `$GITHUB_OUTPUT`
4. Lance conditionnellement le bon template (`java`, `angular`, ou `maven-library`)

**Note technique :** Dans un reusable workflow appelé via `workflow_call`, `github.repository` est bien le repo APPELANT (ex: `Peterery21/sfd-epargne-service`), pas le repo central. C'est ce qui rend cette architecture possible.

**Note sur les nested reusable workflows :** `dispatcher.yml` (appelé par les repos de service) appelle à son tour `java-microservice.yml` — c'est un niveau de nesting valide (max 4 niveaux supportés par GitHub Actions depuis nov 2022).

- [ ] **Créer `.github/workflows/dispatcher.yml`**

```yaml
name: Dispatcher

on:
  workflow_call:
    inputs:
      release:
        description: "Déclencher un release"
        type: boolean
        default: false
      release-type:
        description: "Type de release (patch/minor/major)"
        type: string
        default: "patch"
    secrets:
      REGISTRY_URL:
        required: true
      REGISTRY_USER:
        required: true
      REGISTRY_PASSWORD:
        required: true
      SONAR_TOKEN:
        required: false
      SONAR_HOST_URL:
        required: false
      NEXUS_URL:
        required: false
      NEXUS_USER:
        required: false
      NEXUS_PASSWORD:
        required: false
      VPS_HOST:
        required: true
      VPS_USER:
        required: true
      VPS_SSH_KEY:
        required: true
      GH_PAT:
        required: false

jobs:
  # ─────────────────────────────────────────
  # JOB 1 : Lire le registre et résoudre la config du service appelant
  # ─────────────────────────────────────────
  resolve:
    name: Resolve service config
    runs-on: ubuntu-latest
    outputs:
      template:       ${{ steps.lookup.outputs.template }}
      app-name:       ${{ steps.lookup.outputs.app-name }}
      dev-port:       ${{ steps.lookup.outputs.dev-port }}
      prod-port:      ${{ steps.lookup.outputs.prod-port }}
      internal-port:  ${{ steps.lookup.outputs.internal-port }}
      health-path:    ${{ steps.lookup.outputs.health-path }}
      release-branch: ${{ steps.lookup.outputs.release-branch }}
      e2e-enabled:    ${{ steps.lookup.outputs.e2e-enabled }}
      e2e-command:    ${{ steps.lookup.outputs.e2e-command }}
      json-server-port: ${{ steps.lookup.outputs.json-server-port }}
      sonar-enabled:  ${{ steps.lookup.outputs.sonar-enabled }}
      nexus-deploy:   ${{ steps.lookup.outputs.nexus-deploy }}

    steps:
      - name: Checkout sfd-github-workflows (registre)
        uses: actions/checkout@v4
        with:
          repository: Peterery21/sfd-github-workflows
          ref: main
          token: ${{ secrets.GH_PAT || github.token }}

      - name: Lookup config pour ${{ github.repository }}
        id: lookup
        run: |
          python3 - << 'PYEOF'
          import yaml, os, sys

          with open('services-registry.yml') as f:
              registry = yaml.safe_load(f)

          repo = os.environ['GITHUB_REPOSITORY']
          config = registry.get(repo)

          if not config:
              print(f"::error::Service '{repo}' non trouvé dans services-registry.yml")
              print(f"  Clés disponibles: {list(registry.keys())}")
              sys.exit(1)

          # Valeurs par défaut pour champs optionnels
          defaults = {
              'e2e-enabled': 'false',
              'e2e-command': 'npm run e2e:chromium',
              'json-server-port': '0',
              'sonar-enabled': 'true',
              'nexus-deploy': 'false',
              'release-branch': 'master',
              'internal-port': str(config.get('dev-port', 0)),
          }

          with open(os.environ['GITHUB_OUTPUT'], 'a') as out:
              for key, default in defaults.items():
                  value = config.get(key, default)
                  out.write(f"{key}={value}\n")
              # Champs obligatoires
              for key in ['template', 'app-name', 'dev-port', 'prod-port', 'health-path']:
                  if key not in config and key not in defaults:
                      print(f"::error::Champ '{key}' manquant pour {repo}")
                      sys.exit(1)
                  out.write(f"{key}={config.get(key, defaults.get(key, ''))}\n")

          print(f"✅ Config résolue pour {repo}: template={config.get('template')}, app={config.get('app-name')}")
          PYEOF

  # ─────────────────────────────────────────
  # JOB 2a : Pipeline Java microservice
  # ─────────────────────────────────────────
  java:
    name: Java Microservice Pipeline
    needs: resolve
    if: needs.resolve.outputs.template == 'java'
    uses: Peterery21/sfd-github-workflows/.github/workflows/java-microservice.yml@main
    with:
      app-name:          ${{ needs.resolve.outputs.app-name }}
      dev-port:          ${{ fromJSON(needs.resolve.outputs.dev-port) }}
      prod-port:         ${{ fromJSON(needs.resolve.outputs.prod-port) }}
      java-internal-port: ${{ fromJSON(needs.resolve.outputs.internal-port) }}
      health-path:       ${{ needs.resolve.outputs.health-path }}
      release-branch:    ${{ needs.resolve.outputs.release-branch }}
      release:           ${{ inputs.release }}
      release-type:      ${{ inputs.release-type }}
    secrets: inherit

  # ─────────────────────────────────────────
  # JOB 2b : Pipeline Angular
  # ─────────────────────────────────────────
  angular:
    name: Angular Pipeline
    needs: resolve
    if: needs.resolve.outputs.template == 'angular'
    uses: Peterery21/sfd-github-workflows/.github/workflows/angular-pipeline.yml@main
    with:
      app-name:          ${{ needs.resolve.outputs.app-name }}
      dev-port:          ${{ fromJSON(needs.resolve.outputs.dev-port) }}
      prod-port:         ${{ fromJSON(needs.resolve.outputs.prod-port) }}
      release-branch:    ${{ needs.resolve.outputs.release-branch }}
      e2e-enabled:       ${{ fromJSON(needs.resolve.outputs.e2e-enabled) }}
      e2e-command:       ${{ needs.resolve.outputs.e2e-command }}
      json-server-port:  ${{ fromJSON(needs.resolve.outputs.json-server-port) }}
    secrets: inherit

  # ─────────────────────────────────────────
  # JOB 2c : Pipeline Maven Library
  # ─────────────────────────────────────────
  maven-library:
    name: Maven Library Pipeline
    needs: resolve
    if: needs.resolve.outputs.template == 'maven-library'
    uses: Peterery21/sfd-github-workflows/.github/workflows/maven-library.yml@main
    with:
      app-name:      ${{ needs.resolve.outputs.app-name }}
      sonar-enabled: ${{ fromJSON(needs.resolve.outputs.sonar-enabled) }}
      nexus-deploy:  ${{ fromJSON(needs.resolve.outputs.nexus-deploy) }}
    secrets: inherit
```

- [ ] **Commit**

```bash
git add .github/workflows/dispatcher.yml
git commit -m "feat: add dispatcher.yml — reads services-registry.yml, routes to correct template"
```

---

## Task 3 : Rendre les inputs optionnels dans les templates

**Problème :** Actuellement, `java-microservice.yml` a `required: true` sur ses inputs. Comme le dispatcher les fournit toujours, c'est OK — mais si on veut pouvoir tester un template directement (sans passer par le dispatcher), les inputs doivent avoir des defaults.

**Fichiers :**
- Modifier : `.github/workflows/java-microservice.yml` — `required: true` → `required: false` + defaults
- Modifier : `.github/workflows/angular-pipeline.yml` — idem
- Modifier : `.github/workflows/maven-library.yml` — idem

- [ ] **Dans `java-microservice.yml` : passer tous les inputs service en `required: false`**

Remplacer :
```yaml
      app-name:
        type: string
        required: true
      dev-port:
        type: number
        required: true
      prod-port:
        type: number
        required: true
      java-internal-port:
        type: number
        required: true
      health-path:
        type: string
        required: true
```

Par :
```yaml
      app-name:
        type: string
        required: false
        default: ""
      dev-port:
        type: number
        required: false
        default: 0
      prod-port:
        type: number
        required: false
        default: 0
      java-internal-port:
        type: number
        required: false
        default: 0
      health-path:
        type: string
        required: false
        default: ""
```

- [ ] **Idem dans `angular-pipeline.yml`** pour `app-name`, `dev-port`, `prod-port`

- [ ] **Idem dans `maven-library.yml`** pour `app-name`

- [ ] **Supprimer `nexus-upload.yml`** (workflow de migration one-time, ne fait pas partie du CI régulier)

```bash
git rm .github/workflows/nexus-upload.yml
```

- [ ] **Commit**

```bash
git add .github/workflows/
git commit -m "refactor: make template inputs optional (routed via dispatcher), remove nexus-upload"
```

---

## Task 4 : Template `ci.yml` zéro-config par type de service

**Principe :** Un `ci.yml` dans chaque repo de service est **l'équivalent exact du Jenkinsfile**. Il ne contient aucune config service. Il appelle le dispatcher et passe uniquement les inputs runtime (`release`, `release-type`).

**3 templates :**

### Template A — Java microservices (17 repos)

```yaml
# .github/workflows/ci.yml
# ⚠️ NE PAS MODIFIER — la config service est dans sfd-github-workflows/services-registry.yml
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master, main]
  pull_request:
    branches: [develop, master, main]
  workflow_dispatch:
    inputs:
      release:
        description: "Déclencher un release (bump version + tag + Docker :version)"
        type: boolean
        default: false
      release-type:
        description: "Type de release"
        type: choice
        options: [patch, minor, major]
        default: patch

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    with:
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
    secrets: inherit
```

### Template B — Angular frontends (2 repos)

```yaml
# .github/workflows/ci.yml
# ⚠️ NE PAS MODIFIER — la config service est dans sfd-github-workflows/services-registry.yml
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
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    with:
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
    secrets: inherit
```

*(Identique au Template A — même fichier pour Java et Angular)*

### Template C — Maven libraries (3 repos, pas de release)

```yaml
# .github/workflows/ci.yml
# ⚠️ NE PAS MODIFIER — la config service est dans sfd-github-workflows/services-registry.yml
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master]
  pull_request:
    branches: [develop, master]

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    secrets: inherit
```

- [ ] **Créer les templates dans `sfd-github-workflows/templates/`**

```bash
mkdir -p /tmp/sfd-github-workflows/.github/workflows/templates
```

Créer :
- `templates/ci-service.yml` (Java + Angular — avec release inputs)
- `templates/ci-library.yml` (Maven libs — sans release inputs)

- [ ] **Commit**

```bash
git add .github/workflows/templates/
git commit -m "feat: add zero-config ci.yml templates (equivalent to Jenkinsfile)"
```

---

## Task 5 : Mettre à jour tous les repos de service

Remplacer le `ci.yml` de chaque repo par la version zéro-config.

**Script de mise à jour :**

- [ ] **Mettre à jour les 19 repos Java + Angular**

```bash
CI_JAVA=$(cat << 'EOF'
# .github/workflows/ci.yml
# ⚠️ NE PAS MODIFIER — la config service est dans sfd-github-workflows/services-registry.yml
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master, main]
  pull_request:
    branches: [develop, master, main]
  workflow_dispatch:
    inputs:
      release:
        description: "Déclencher un release (bump version + tag + Docker :version)"
        type: boolean
        default: false
      release-type:
        description: "Type de release"
        type: choice
        options: [patch, minor, major]
        default: patch

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    with:
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
    secrets: inherit
EOF
)

for SVC in sfd-commun-service sfd-epargne-service sfd-client-service sfd-caisse-service \
  sfd-comptabilite-service sfd-credit-service sfd-stock-service sfd-commercial-service \
  sfd-reporting-service sfd-suivi-evaluation-service sfd-portail-client-service \
  sfd-immobilisation-service sfd-rh-service sfd-paie-service sfd-budget-service \
  sfd-transfert-service sfd-agent-mobile-service sfd-angular sfd-portail-client-angular; do
  cd /Users/pierreadopre/Projects/erp-sfd/$SVC
  git fetch origin develop -q --no-rebase 2>/dev/null || true
  git reset --hard origin/develop -q
  echo "$CI_JAVA" > .github/workflows/ci.yml
  git add .github/workflows/ci.yml
  git commit -m "ci: simplify to zero-config (config centralisée dans services-registry.yml)" -q
  git push origin develop -q
  echo "✅ $SVC"
done
```

- [ ] **Mettre à jour les 3 libs Maven**

```bash
CI_LIB=$(cat << 'EOF'
# .github/workflows/ci.yml
# ⚠️ NE PAS MODIFIER — la config service est dans sfd-github-workflows/services-registry.yml
name: CI/CD Pipeline
on:
  push:
    branches: [develop, master]
  pull_request:
    branches: [develop, master]

jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    secrets: inherit
EOF
)

for SVC in sfd-demo-data sfd-integration-api sfd-report-api; do
  cd /Users/pierreadopre/Projects/erp-sfd/$SVC
  git fetch origin develop -q --no-rebase 2>/dev/null || true
  git reset --hard origin/develop -q
  echo "$CI_LIB" > .github/workflows/ci.yml
  git add .github/workflows/ci.yml
  git commit -m "ci: simplify to zero-config (config centralisée dans services-registry.yml)" -q
  git push origin develop -q
  echo "✅ $SVC"
done
```

---

## Task 6 : Push final et vérification

- [ ] **Push `sfd-github-workflows`**

```bash
cd /tmp/sfd-github-workflows
git push origin main
```

- [ ] **Vérifier sur GitHub que le `dispatcher.yml` est présent**

URL : `https://github.com/Peterery21/sfd-github-workflows/blob/main/.github/workflows/dispatcher.yml`

- [ ] **Déclencher un test sur `sfd-commun-service`**

```bash
cd /Users/pierreadopre/Projects/erp-sfd/sfd-commun-service
# Pousser un commit trivial sur develop pour déclencher le workflow
git commit --allow-empty -m "ci: trigger test pipeline [skip ci]"
git push origin develop
```

Puis vérifier sur GitHub Actions : `https://github.com/Peterery21/sfd-commun-service/actions`

Le pipeline doit passer par : `resolve` → lit `sfd-commun-service` dans le registre → `java` job → build/test/sonar/docker/deploy.

- [ ] **Vérifier le log du job `resolve`**

Dans le log GitHub Actions, chercher :
```
✅ Config résolue pour Peterery21/sfd-commun-service: template=java, app=sfd-commun
```

---

## Résultat attendu

### Avant (actuel)

```yaml
# ci.yml dans sfd-epargne-service (14 lignes de config service)
jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/java-microservice.yml@main
    with:
      app-name: sfd-epargne          # ← config service
      dev-port: 4596                 # ← config service
      prod-port: 4612                # ← config service
      java-internal-port: 4596       # ← config service
      health-path: /api/epargne/...  # ← config service
      release-branch: master         # ← config service
      release: ${{ inputs.release || false }}
      release-type: ${{ inputs.release-type || 'patch' }}
```

### Après (cible)

```yaml
# ci.yml dans sfd-epargne-service (IDENTIQUE à tous les autres repos Java)
jobs:
  pipeline:
    uses: Peterery21/sfd-github-workflows/.github/workflows/dispatcher.yml@main
    with:
      release: ${{ inputs.release || false }}      # runtime input utilisateur
      release-type: ${{ inputs.release-type || 'patch' }}  # runtime input utilisateur
    secrets: inherit
    # Toute la config (ports, health-path, etc.) est dans services-registry.yml
```

### Modifier la config d'un service

**Avant :** éditer `sfd-epargne-service/.github/workflows/ci.yml`
**Après :** éditer `sfd-github-workflows/services-registry.yml` ← **un seul endroit**

### Ajouter un nouveau service

**Avant :** créer un `ci.yml` avec tous les params dans le nouveau repo
**Après :** ajouter une entrée dans `services-registry.yml` + copier le template `ci-service.yml` dans le nouveau repo

---

## Secrets org-level à configurer (prérequis)

Ces secrets doivent être définis dans **Settings > Secrets > Actions** de l'organisation `Peterery21` :

| Secret | Valeur |
|--------|--------|
| `REGISTRY_URL` | `registry.evolticatechnologies.com` |
| `REGISTRY_USER` | `admin` |
| `REGISTRY_PASSWORD` | mot de passe registry |
| `SONAR_TOKEN` | `sqa_53a1bc3c43e58b44889646b167ba08de37110d4f` |
| `SONAR_HOST_URL` | `https://sonarqube.evolticatechnologies.com` |
| `NEXUS_URL` | `https://nexus.evolticatechnologies.com` |
| `NEXUS_USER` | `admin` |
| `NEXUS_PASSWORD` | mot de passe Nexus |
| `VPS_HOST` | `82.180.131.215` |
| `VPS_USER` | user SSH VPS |
| `VPS_SSH_KEY` | clé privée SSH VPS |
| `GH_PAT` | token GitHub `contents:write` (pour les releases) |
