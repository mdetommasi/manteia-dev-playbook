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

![CI Flow Diagram](/assets/ci_flow.svg)

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
