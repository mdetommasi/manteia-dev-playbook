# CI/CD Playbook – GitLab Edition  
**Mission Critical AI/ML Systems**  
*Microservizi, RAG, Agentic AI, Pipeline Dati*

---

## **Visione d'Insieme**

Un flusso **sicuro, automatico, osservabile e resiliente** per sistemi AI mission-critical.  
Progettato per **GitLab CI/CD**, con **zero dipendenze da GitHub Actions**.

> **Target DORA**: *Elite Performer*  
> Daily Deploy | Lead Time <1 giorno | CFR <10% | MTTR <1h

---

## **Principi Chiave**

| Principio | Descrizione |
|---------|-----------|
| **Ogni merge è verde** | Lint, test, security, build immagine → tutti passano |
| **Shift-left security** | SBOM, SCA, container scan, secret detection in CI |
| **Promozioni progressive** | `dev → staging → canary → prod` con rollback automatico |
| **Immutable artifacts** | Immagini firmate, versionate, riproducibili |
| **Osservabilità by default** | Metriche, log, trace in ogni ambiente |

---

## **Flusso Pipeline: Overview**

```svg
<?xml version="1.0" encoding="UTF-8"?>
<svg width="900" height="600" viewBox="0 0 900 600" xmlns="http://www.w3.org/2000/svg" style="font-family:Arial,Helvetica,sans-serif;">
  <!-- Background -->
  <rect width="100%" height="100%" fill="#f8f9fa"/>

  <!-- Title -->
  <text x="450" y="40" text-anchor="middle" font-size="24" font-weight="bold" fill="#2c3e50">Pipeline CI/CD – AI Systems</text>

  <!-- Local Dev -->
  <g transform="translate(50,80)">
    <rect x="0" y="0" width="150" height="80" rx="12" fill="#e3f2fd" stroke="#2196f3" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#1565c0">DEV LOCAL</text>
    <text x="75" y="50" text-anchor="middle" font-size="11" fill="#1976d2">pre-commit</text>
    <text x="75" y="65" text-anchor="middle" font-size="11" fill="#1976d2">lint + unit</text>
  </g>

  <!-- Arrow down -->
  <path d="M 125 160 L 125 180 M 115 170 L 125 180 L 135 170" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Feature Branch -->
  <g transform="translate(50,200)">
    <rect x="0" y="0" width="150" height="80" rx="12" fill="#fff3e0" stroke="#ff9800" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">FEATURE</text>
    <text x="75" y="50" text-anchor="middle" font-size="11" fill="#f57c00">build + unit</text>
    <text x="75" y="65" text-anchor="middle" font-size="11" fill="#f57c00">SonarQube</text>
  </g>

  <!-- Arrow right -->
  <path d="M 200 240 L 250 240 M 240 230 L 250 240 L 240 250" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Merge Request -->
  <g transform="translate(270,200)">
    <rect x="0" y="0" width="150" height="80" rx="12" fill="#fff3e0" stroke="#ff9800" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#e65100">MERGE REQUEST</text>
    <text x="75" y="50" text-anchor="middle" font-size="11" fill="#f57c00">E2E + contract</text>
    <text x="75" y="65" text-anchor="middle" font-size="11" fill="#f57c00">security scan</text>
  </g>

  <!-- Arrow down -->
  <path d="M 345 280 L 345 300 M 335 290 L 345 300 L 355 290" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Main Pipeline -->
  <g transform="translate(270,320)">
    <rect x="0" y="0" width="150" height="100" rx="12" fill="#e8f5e9" stroke="#4caf50" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#2e7d32">MAIN</text>
    <text x="75" y="48" text-anchor="middle" font-size="11" fill="#388e3c">full test</text>
    <text x="75" y="63" text-anchor="middle" font-size="11" fill="#388e3c">build multi-arch</text>
    <text x="75" y="78" text-anchor="middle" font-size="11" fill="#388e3c">cosign sign</text>
    <text x="75" y="93" text-anchor="middle" font-size="11" fill="#388e3c">push registry</text>
  </g>

  <!-- Arrow right -->
  <path d="M 420 370 L 470 370 M 460 360 L 470 370 L 460 380" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Staging -->
  <g transform="translate(490,320)">
    <rect x="0" y="0" width="150" height="100" rx="12" fill="#e0f7fa" stroke="#00acc1" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#006064">STAGING</text>
    <text x="75" y="48" text-anchor="middle" font-size="11" fill="#00838f">auto-deploy</text>
    <text x="75" y="63" text-anchor="middle" font-size="11" fill="#00838f">smoke + sanity</text>
    <text x="75" y="78" text-anchor="middle" font-size="11" fill="#00838f">performance</text>
    <text x="75" y="93" text-anchor="middle" font-size="11" fill="#00838f">synthetic ON</text>
  </g>

  <!-- Arrow down -->
  <path d="M 565 420 L 565 440 M 555 430 L 565 440 L 575 430" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Prod Approval -->
  <g transform="translate(490,460)">
    <rect x="0" y="0" width="150" height="60" rx="12" fill="#fce4ec" stroke="#e91e63" stroke-width="2"/>
    <text x="75" y="30" text-anchor="middle" font-size="14" font-weight="bold" fill="#880e4f">APPROVAL</text>
    <text x="75" y="48" text-anchor="middle" font-size="11" fill="#c2185b">manual gate</text>
  </g>

  <!-- Arrow down -->
  <path d="M 565 520 L 565 540 M 555 530 L 565 540 L 575 530" fill="none" stroke="#555" stroke-width="2"/>

  <!-- Production Block -->
  <g transform="translate(700,80)">
    <rect x="0" y="0" width="160" height="460" rx="15" fill="#ffebee" stroke="#d32f2f" stroke-width="3"/>
    <text x="80" y="35" text-anchor="middle" font-size="16" font-weight="bold" fill="#b71c1c">PRODUZIONE</text>
    
    <!-- Canary -->
    <rect x="10" y="55" width="140" height="120" rx="10" fill="#fff8e1" stroke="#ffb300" stroke-width="1.5"/>
    <text x="80" y="80" text-anchor="middle" font-size="13" font-weight="bold" fill="#ff6f00">CANARY</text>
    <text x="80" y="100" text-anchor="middle" font-size="10" fill="#e65100">5% → 10min</text>
    <text x="80" y="115" text-anchor="middle" font-size="10" fill="#e65100">25% → 20min</text>
    <text x="80" y="130" text-anchor="middle" font-size="10" fill="#e65100">50% → 30min</text>
    <text x="80" y="145" text-anchor="middle" font-size="10" fill="#e65100">100%</text>

    <!-- Post-Deploy -->
    <rect x="10" y="185" width="140" height="80" rx="10" fill="#e8f5e9" stroke="#43a047" stroke-width="1.5"/>
    <text x="80" y="210" text-anchor="middle" font-size="13" font-weight="bold" fill="#2e7d32">POST-DEPLOY</text>
    <text x="80" y="230" text-anchor="middle" font-size="10" fill="#388e3c">smoke prod</text>
    <text x="80" y="245" text-anchor="middle" font-size="10" fill="#388e3c">health check</text>
    <text x="80" y="260" text-anchor="middle" font-size="10" fill="#388e3c">auto-rollback</text>

    <!-- Monitoring -->
    <rect x="10" y="275" width="140" height="160" rx="10" fill="#e3f2fd" stroke="#1976d2" stroke-width="1.5"/>
    <text x="80" y="300" text-anchor="middle" font-size="13" font-weight="bold" fill="#0d47a1">MONITORING</text>
    <text x="80" y="320" text-anchor="middle" font-size="10" fill="#1565c0">RUM</text>
    <text x="80" y="335" text-anchor="middle" font-size="10" fill="#1565c0">synthetic 5min</text>
    <text x="80" y="350" text-anchor="middle" font-size="10" fill="#1565c0">tracing</text>
    <text x="80" y="365" text-anchor="middle" font-size="10" fill="#1565c0">logs</text>
    <text x="80" y="380" text-anchor="middle" font-size="10" fill="#1565c0">alerts</text>
    <text x="80" y="415" text-anchor="middle" font-size="9" fill="#1976d2">Golden Signals 24/7</text>
  </g>

  <!-- Legend -->
  <g transform="translate(50,540)">
    <text x="0" y="0" font-size="12" font-weight="bold" fill="#2c3e50">Legenda</text>
    <rect x="0" y="10" width="15" height="15" rx="4" fill="#e3f2fd" stroke="#2196f3"/>
    <text x="25" y="22" font-size="11" fill="#333">Dev Locale</text>
    
    <rect x="120" y="10" width="15" height="15" rx="4" fill="#fff3e0" stroke="#ff9800"/>
    <text x="145" y="22" font-size="11" fill="#333">CI (Feature/MR)</text>
    
    <rect x="240" y="10" width="15" height="15" rx="4" fill="#e8f5e9" stroke="#4caf50"/>
    <text x="265" y="22" font-size="11" fill="#333">Main Build</text>
    
    <rect x="360" y="10" width="15" height="15" rx="4" fill="#e0f7fa" stroke="#00acc1"/>
    <text x="385" y="22" font-size="11" fill="#333">Staging Auto</text>
    
    <rect x="480" y="10" width="15" height="15" rx="4" fill="#fce4ec" stroke="#e91e63"/>
    <text x="505" y="22" font-size="11" fill="#333">Prod Gate</text>
    
    <rect x="600" y="10" width="15" height="15" rx="4" fill="#ffebee" stroke="#d32f2f"/>
    <text x="625" y="22" font-size="11" fill="#333">Produzione</text>
  </g>

  <!-- Footer -->
  <text x="450" y="580" text-anchor="middle" font-size="11" fill="#7f8c8d">
    DORA Elite: Daily Deploy • &lt;1 giorno Lead Time • &lt;10% CFR • &lt;1h MTTR
  </text>
</svg>
```

---

## **Pipeline: 9 Stage GitLab**

```yaml
stages:
  - pre-commit      # Locale: lint + unit base
  - feature         # Branch: build + quality gate
  - merge-request   # MR: integration/E2E + security scan
  - main            # Main: build multi-arch + firma
  - staging         # Auto-deploy + test completi
  - approval        # Manual gate per prod
  - canary          # Deploy progressivo
  - post-deploy     # Smoke prod + health check
  - monitoring      # Observability 24/7
```

---

## **Stage Dettagliato**

### **1. Pre-commit (Locale)**
```yaml
pre-commit:
  stage: pre-commit
  script:
    - ruff check . && black --check .
    - mypy src/ --strict
    - pytest -m "not integration and not e2e" --maxfail=1
  only: [branches, merge_requests]
```

**Tools**:
- Python: `ruff`, `black`, `mypy`
- Node: `eslint`, `prettier`, `tsc --noEmit`
- Hook: `.pre-commit-config.yaml`

---

### **2. Feature (Branch)**
```yaml
feature_build:
  stage: feature
  script:
    - pip install -r requirements.txt
    - ruff check . && black --check .
    - mypy src/
    - pytest -m unit --junitxml=report.xml --cov=src --cov-report=xml
    - sonar-scanner
  artifacts:
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

**Quality gate**: SonarQube con soglia 70% coverage

---

### **3. Merge Request**

#### **Test avanzati (provvisorio: da definire tool E2E)**
```yaml
mr_integration:
  stage: merge-request
  script:
    # Integration con Testcontainers
    - pytest -m integration --junitxml=integration-report.xml
    # Contract testing
    - pytest -m contract
    # E2E con Playwright
    - playwright install --with-deps
    - pytest -m e2e --headed=false
  artifacts:
    reports:
      junit: integration-report.xml
```

#### **Security scan completo (provvisorio)**
```yaml
mr_security:
  stage: merge-request
  image: aquasec/trivy:latest
  script:
    # Build immagine
    - docker build -t $CI_REGISTRY_IMAGE:mr-$CI_MERGE_REQUEST_IID .
    
    # Trivy: filesystem + container
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
    
    # SBOM generation
    - syft $IMAGE -o spdx-json > sbom.json
    
    # CVE scan con Grype
    - grype sbom.json --fail-on high
    
    # Secret detection
    - gitleaks detect --no-git --verbose
    
    # Code security
    - bandit -r src/ -f json -o bandit-report.json
    - pip-audit --format json --output pip-audit.json
  artifacts:
    paths: [sbom.json, bandit-report.json, pip-audit.json]
    expire_in: 1 week
  rules:
    - if: '$CI_MERGE_REQUEST_IID'
```

**Security tools**:
- Container: `trivy` (FS + image)
- SBOM: `syft` (SPDX format)
- CVE: `grype`
- Secrets: `gitleaks`
- Code: `bandit` (Python) | `npm audit` (Node)
- Deps: `pip-audit` | `npm audit`
- IaC: `checkov` / `tfsec`

**Branch protection**:
- 1–2 approvatori
- Required checks: lint, tests, security
- CODEOWNERS file
- Conventional commits

---

### **4. Main (Build & Sign)**
```yaml
main_build:
  stage: main
  script:
    - docker buildx create --use
    - |
      docker buildx build \
        --platform linux/amd64,linux/arm64 \
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache \
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max \
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        --tag $CI_REGISTRY_IMAGE:latest \
        --push .
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

main_sign:
  stage: main
  script:
    - cosign sign --yes $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - cosign attach sbom --sbom sbom.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  needs: [main_build, mr_security]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

**Hardening immagine**:
- Base: `distroless` / `slim`
- User: non-root
- Security: `--cap-drop ALL`
- No shell se non necessaria

---

### **5. Staging (Auto-deploy)**
```yaml
deploy_staging:
  stage: staging
  script:
    - ./scripts-ci/deploy/deploy-to-staging.sh
    - ./scripts-ci/test/smoke-test.sh --env staging --timeout 60
    - ./scripts-ci/test/sanity-test.sh
    - ./scripts-ci/test/performance-test.sh --duration 5m
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop_staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

### **6. Approval (Manual Gate)**
```yaml
approve_production:
  stage: approval
  script: echo "⏸️ Awaiting production approval"
  when: manual
  allow_failure: false
  environment:
    name: production
    action: prepare
```

---

### **7. Canary (Progressive Rollout)**
```yaml
canary_deploy:
  stage: canary
  script:
    # 5% traffic
    - ./scripts-ci/deploy/canary-deploy.sh --percent 5
    - sleep 600
    - ./scripts-ci/monitoring/check-canary-health.sh || exit 1
    
    # 25% traffic
    - ./scripts-ci/deploy/canary-deploy.sh --percent 25
    - sleep 1200
    - ./scripts-ci/monitoring/check-canary-health.sh || exit 1
    
    # 50% traffic
    - ./scripts-ci/deploy/canary-deploy.sh --percent 50
    - sleep 1800
    - ./scripts-ci/monitoring/check-canary-health.sh || exit 1
    
    # 100% traffic
    - ./scripts-ci/deploy/canary-deploy.sh --percent 100
  on_failure:
    - ./scripts-ci/deploy/rollback-deployment.sh --immediate
  environment:
    name: production
    url: https://production.example.com
  needs: [approve_production]
```

**Alternative strategie**:
- **Blue-Green**: switch atomico tra 2 ambienti
- **Feature Flags**: Unleash/Flagsmith per toggle graduali

---

### **8. Post-deploy (provvisorio)**
```yaml
post_deploy:
  stage: post-deploy
  script:
    - ./scripts-ci/test/smoke-test.sh --env prod --timeout 120
    - kubectl get pods -n prod -o wide
    - ./scripts-ci/monitoring/check-golden-signals.sh
  retry:
    max: 2
    when: script_failure
  on_failure:
    - ./scripts-ci/deploy/rollback-deployment.sh --immediate
    - ./scripts-ci/alerts/notify-oncall.sh
  environment:
    name: production
```

**Golden Signals**:
- **Latency**: p50, p95, p99
- **Errors**: rate 4xx/5xx
- **Traffic**: RPS
- **Saturation**: CPU/Memory

---

### **9. Monitoring (Continuous) (in via di definizione)**
```yaml
monitoring:
  stage: monitoring
  script:
    - echo "🔍 Monitoring attivo 24/7"
    - echo "RUM + Synthetic tests ogni 5min"
    - echo "OpenTelemetry → Prometheus → Grafana"
  when: manual
  allow_failure: true
```

**Stack**:
- **RUM**: User experience reale
- **Synthetic**: endpoint critici ogni 5min
- **Tracing**: OpenTelemetry (LLM token, latenza tools)
- **Logs**: structured JSON → aggregazione
- **Alerts**: on-call + runbook

---

## **Artifact Management**

### **Container Registry**
```bash
# Naming convention
$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA    # Immutable
$CI_REGISTRY_IMAGE:v1.2.3            # SemVer
$CI_REGISTRY_IMAGE:latest            # Solo staging
```

**Registry**: GHCR / ECR / GCR / ACR  
**Firma**: `cosign` + SBOM  
**Retention**: 30gg SHA, ∞ semver

### **Packages**
- **Python**: PyPI privato (Artifactory/Nexus)
- **Node**: npm registry privato

### **ML Artifacts**
- **Dataset**: DVC → S3/MinIO
- **Modelli**: MLflow Registry (`Staging` → `Production`)

---

## **Gestione Segreti (provvisorio)**

```yaml
# .gitlab-ci.yml
variables:
  VAULT_ADDR: https://vault.internal
  
deploy_production:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  script:
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role=gitlab-ci jwt=$GITLAB_OIDC_TOKEN)
    - export DB_PASSWORD=$(vault kv get -field=password secret/prod/db)
```

**Best practice**:
- **OIDC** GitLab → Cloud (zero secret statici)
- **Runtime**: Vault / GitLab Masked Variables
- **Mount**: K8s Secrets / env
- **Scan**: `gitleaks` in ogni stage

---

## **Migrazioni Database**

```yaml
migrate_staging:
  stage: staging
  before_script:
    - ./scripts-ci/db/backup-db.sh staging
  script:
    - alembic upgrade head
  on_failure:
    - alembic downgrade -1
    - ./scripts-ci/db/restore-db.sh staging
```

**Tools**: Alembic / Prisma / Flyway / Liquibase

**Regole**:
- Backward-compatible (expand → migrate → contract)
- Idempotenza (safe re-run)
- Backup automatico
- Rollback scriptato

---

## **GitOps (K8s)**

```yaml
# Alternative a deploy diretto
gitops_promote:
  stage: staging
  script:
    - git clone https://gitlab.com/org/k8s-manifests.git
    - cd k8s-manifests
    - yq -i '.image.tag = "$CI_COMMIT_SHA"' apps/myapp/values.yaml
    - git commit -am " Deploy $CI_COMMIT_SHA to staging"
    - git push
  # Argo CD sincronizza automaticamente
```

**Tools**: Argo CD / Flux  
**Benefit**: audit trail, rollback dichiarativo, drift detection

---

## **Versioning & Release**

```yaml
release:
  stage: main
  script:
    - npx semantic-release
  only:
    - main
  artifacts:
    paths: [CHANGELOG.md]
```

**Convention**:
- **SemVer**: `v1.2.3` (API/librerie)
- **Conventional commits**: `feat:`, `fix:`, `BREAKING:`
- **Changelog**: auto-generato
- **Release train**: settimanale + hotfix

---

## **Rollback & Incident**

```bash
# 1-comando rollback
helm rollback myapp 0                    # ultima revisione
kubectl rollout undo deployment/myapp    # K8s nativo
argocd app rollback myapp                # GitOps
```

**Incident response**:
1. **Feature flag kill switch** (spegni funzionalità)
2. **Rollback immediato** (<5min)
3. **Postmortem** entro 48h (timeline + azioni)

---

## **Directory `scripts-ci/` (provvisoria**

```bash
scripts-ci/
├── deploy/
│   ├── canary-deploy.sh          # Rollout progressivo
│   ├── deploy-to-staging.sh      # Deploy staging
│   └── rollback-deployment.sh    # Rollback automatico
├── test/
│   ├── smoke-test.sh             # Test base post-deploy
│   ├── sanity-test.sh            # Validazione funzionale
│   └── performance-test.sh       # Load testing
├── monitoring/
│   ├── check-canary-health.sh    # Validazione canary
│   └── check-golden-signals.sh   # Metriche SLO
├── db/
│   ├── backup-db.sh              # Backup pre-migrazione
│   └── restore-db.sh             # Restore su errore
├── alerts/
│   └── notify-oncall.sh          # Notifica team on-call
└── README.md
```

---

## **Tools Reference**

| Categoria | Tool | Link |
|-----------|------|------|
| **Lint** | ruff, black, eslint, prettier | [ruff](https://docs.astral.sh/ruff/) |
| **Type** | mypy, tsc | [mypy](https://mypy.readthedocs.io/) |
| **Test** | pytest, vitest, Testcontainers | [pytest](https://docs.pytest.org/) |
| **E2E** | Playwright | [playwright](https://playwright.dev/) |
| **Security** | trivy, bandit, gitleaks | [trivy](https://aquasecurity.github.io/trivy/) |
| **SBOM** | syft, grype | [syft](https://anchore.com/opensource/syft/) |
| **Sign** | cosign | [cosign](https://docs.sigstore.dev/cosign/) |
| **Policy** | OPA/Conftest | [opa](https://www.openpolicyagent.org/) |
| **ML** | DVC, MLflow | [dvc](https://dvc.org/) · [mlflow](https://mlflow.org/) |
| **GitOps** | Argo CD, Flux | [argo-cd](https://argo-cd.readthedocs.io/) |
| **Flags** | Unleash, Flagsmith | [unleash](https://www.getunleash.io/) |
| **Observability** | OpenTelemetry, Prometheus | [otel](https://opentelemetry.io/) |

---

## **DORA Metrics**

| Metrica | Baseline | Target Elite | Status |
|---------|----------|--------------|--------|
| **Deployment Frequency** | 1–2/settimana | **Giornaliero** | 🔄 |
| **Lead Time** | 5–10 giorni | **<1 giorno** | 🔄 |
| **Change Failure Rate** | 10–20% | **<10%** | 🔄 |
| **MTTR** | 4–24 ore | **<1 ora** | 🔄 |

---

## **Quick Start**

```bash
#TO DO setup repo
```

---

*Versione 1.0 – Mission Critical AI/ML Systems*
