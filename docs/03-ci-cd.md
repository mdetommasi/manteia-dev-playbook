# CI/CD Playbook

Questa sezione definisce principi, processi e strumenti per **Continuous Integration** e **Continuous Delivery/Deployment** in progetti AI/ML (microservizi, RAG/agent, pipeline dati). È pensata per GitHub Actions, ma i concetti sono trasferibili ad altri sistemi (GitLab CI, Jenkins, Azure DevOps).

---

## Principi chiave
- **Ogni PR deve essere “verde”**: lint, type-check, test, security scans e build immagine passano.
- **Shift-left sulla sicurezza**: SBOM, SCA, scans container e segreti nel ciclo CI.
- **Promozioni progressive**: `dev → staging → prod` con canary/blue-green e rollback.
- **Immutable artifacts**: versioni firmate e tracciabili, build riproducibili.
- **Osservabilità by default**: metriche/log/traces attivi in ogni ambiente.

---

## CI (per PR)

### 1) Lint & Format
- **Python**: `ruff`, `black`  
- **TypeScript/Node**: `eslint`, `prettier`  
- **Dockerfile**: `hadolint`  
- **Shell**: `shellcheck`  
- Hook locali con **pre-commit** per coerenza.

### 2) Type-check
- **Python**: `mypy` (strict dove sensato)  
- **TypeScript**: `tsc --noEmit`

### 3) Test & Coverage
- **Python**: `pytest` + `coverage.py` (soglia consigliata 70–80% line, esclusi e2e)  
- **Node**: `vitest` o `jest`  
- **Integration**: **Testcontainers** per DB/queue/vector store; **VCR.py** o `msw` per HTTP.  
- **E2E**: **Playwright** su percorsi critici (opzionale su PR, obbligatorio su `main` notturno).

### 4) Security Scans
- **Code/Deps (Python)**: `bandit`, `pip-audit` (o `safety`)  
- **Code/Deps (Node)**: `npm audit` (o `yarn audit`)  
- **Segreti**: `gitleaks`/`trufflehog`  
- **Container**: `trivy` (FS e imagem), **SBOM** con `syft` + **CVE scan** con `grype`  
- **IaC**: `checkov` / `tfsec` (se usi Terraform)

### 5) Build immagine Docker
- **buildx** con target multi-arch se necessario
- **cache** di layer su GHCR/registry
- **hardening**: base image distroless/slim, utente non-root, `--cap-drop`, no shell dove non serve
- **firma**: `cosign sign` (più **attestation** per SLSA/in-toto, se previsto)

### 6) Policy Gate (opzionale)
- **OPA/Conftest** per policy su Dockerfile/K8s manifest (es.: niente `latest`, user non-root, risorse minime)
- **Admission controller** (es. Gatekeeper) in ambienti K8s

### 7) Caching & Speed
- `actions/cache` per pip/npm
- Test paralleli e **matrix** (es. Py 3.10/3.11)
- Re-use artefatti (wheel, build) tra job

### 8) Branch Protection & Qualità PR
- **Branch**: trunk-based + feature branches
- **Rule**: review 1–2 approvatori, required checks (lint, tests, security), **CODEOWNERS**
- **Conventional commits** (facilita changelog/semver)

---

## CD (Continuous Delivery/Deployment)

### Ambienti & Promozioni
- `dev`: deploy ad ogni merge su branch dedicato (o su PR con preview)
- `staging`: **automatico** al merge in `main` (smoke test post-deploy)
- `prod`: **manual gate** (approval) o **promozione GitOps**

### Strategie di rilascio
- **Canary**: % traffico graduale su nuova versione + metriche SLO (error rate/latency) → *autopromote/rollback*
- **Blue-Green**: due ambienti identici; switch del router quando “green” è ready
- **Feature Flags**: rilasci oscurati dietro toggle (Unleash/Flagsmith)

### Migrazioni DB con rollback
- **Strumenti**: Alembic (Python), Prisma/Flyway/Liquibase  
- **Regole**: backward-compatible (expand→migrate→contract), migrazioni idempotenti, hook automatici in CD, **backup** prima di migrazioni rischiose, **rollback** scriptato.

### GitOps (consigliato per K8s)
- **Argo CD** o **Flux**: il cluster converge allo stato dichiarativo (Helm/Kustomize)
- **Promozione** = PR al repo “infra”/“manifests” che aggiorna il tag immagine

### Osservabilità post-deploy
- **SLO/SLI**: p95 latency, error rate, saturazione
- **Alert**: canali on-call; **runbook** di rollback documentato
- **Tracing**: OpenTelemetry per hop agentici/tool (LLM token usage, latenza tools)

---

## Artifact Registry

### Container images
- **GHCR** / ECR / GCR / ACR (nomina: `org/svc:{git_sha|semver}`)
- **Firma** con `cosign` + **SBOM** allegato (syft)

### Package
- **Python**: wheel su PyPI privato (Artifactory/Nexus/GH Packages)
- **Node**: npm package su registry privato

### Dati & Modelli
- **Dataset snapshot**: **DVC** verso S3/MinIO con hash/versioni
- **Modelli**: **MLflow Model Registry** con stage (`Staging`, `Production`), metriche e lineage

---

## Esempi GitHub Actions (estratti)

### CI — Python + Node + Security + Docker build
```yaml
name: ci
on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  python:
    runs-on: ubuntu-latest
    strategy:
      matrix: { python-version: ["3.10", "3.11"] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "${{ matrix.python-version }}", cache: 'pip' }
      - run: pip install -U pip && pip install ruff black mypy pytest coverage bandit pip-audit
      - name: Lint/Format/Types
        run: ruff check . && black --check . && mypy src || true
      - name: Tests
        run: pytest -q --maxfail=1
      - name: Coverage report
        run: coverage run -m pytest && coverage xml && coverage report -m
      - name: Security (bandit/pip-audit)
        run: bandit -q -r src || true && pip-audit || true

  node:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci || npm i
      - run: npm run lint --if-present || npx eslint . || true
      - run: npm run test --if-present || npx vitest run || true
      - run: npm audit --audit-level=moderate || true

  container:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # per OIDC (cosign)
    steps:
      - uses: actions/checkout@v4
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build & Push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: SBOM + Scan
        run: |
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b . v1.0.0
          curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b . v0.77.0
          ./syft ghcr.io/${{ github.repository }}:${{ github.sha }} -o spdx-json > sbom.json
          ./grype ghcr.io/${{ github.repository }}:${{ github.sha }} --fail-on high || true
      - name: Sign image (cosign)
        run: |
          COSIGN_EXPERIMENTAL=1 cosign sign ghcr.io/${{ github.repository }}:${{ github.sha }}             --yes --oidc-provider github-actions

```

### CD — Staging auto + Canary/Blue-Green (K8s)
```yaml
name: cd
on:
  push:
    branches: [ main ]

jobs:
  deploy_staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Render Helm chart
        run: helm upgrade --install svc charts/svc -n staging              --set image.tag=${{ github.sha }} --wait
      - name: Smoke tests
        run: ./scripts/smoke_tests.sh

  promote_prod:
    needs: [deploy_staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://your-prod-url.example
    steps:
      - uses: actions/checkout@v4
      - name: Canary 10%
        run: kubectl -n prod apply -f k8s/canary-10.yaml && ./scripts/canary_check.sh
      - name: Promote 100% (Blue-Green switch)
        run: kubectl -n prod apply -f k8s/rollout-green.yaml
      - name: Post-deploy checks
        run: ./scripts/post_deploy_checks.sh
```

> **Nota**: in alternativa, usa **Argo CD** (GitOps): la pipeline aggiorna un **repo di manifest**; Argo sincronizza il cluster.

---

## Gestione segreti e identità
- **OIDC** GitHub→Cloud (niente secret statici) per push su registry e deploy su K8s
- **Vault/Secret Manager** per runtime; mount come env/secret K8s
- **Regole**: no segreti in repo/log; scans con `gitleaks` in CI

---

## Versioning & Release
- **SemVer** per API/librerie; immagini container taggate `{semver|git_sha}`
- **CHANGELOG** auto (conventional commits + release-please/semantic-release)
- **Release train**: cadenza (settimanale/bisettimanale) + hotfix fuori banda

---

## Rollback & Incident Response
- **Rollback** in 1 comando (helm rollback/argo rollback/kubectl rollout undo)
- **Feature-flag kill switch** per spegnere funzionalità difettose
- **Postmortem** con timeline e azioni correttive (owner + due date)

---

## Tooling (link rapidi)
- Ruff: https://docs.astral.sh/ruff/ — Black: https://black.readthedocs.io/
- ESLint: https://eslint.org/ — Prettier: https://prettier.io/
- Mypy: https://mypy.readthedocs.io/ — Pytest: https://docs.pytest.org/
- Vitest: https://vitest.dev/ — Jest: https://jestjs.io/
- Testcontainers: https://testcontainers-python.readthedocs.io/
- VCR.py: https://vcrpy.readthedocs.io/ — MSW: https://mswjs.io/
- Bandit: https://bandit.readthedocs.io/ — pip-audit: https://pip-audit.readthedocs.io/
- Trivy: https://aquasecurity.github.io/trivy/ — Syft: https://anchore.com/opensource/syft/ — Grype: https://anchore.com/opensource/grype/
- Cosign: https://docs.sigstore.dev/cosign/ — Conftest/OPA: https://www.openpolicyagent.org/docs/latest/
- Hadolint: https://github.com/hadolint/hadolint — ShellCheck: https://www.shellcheck.net/
- DVC: https://dvc.org/ — MLflow: https://mlflow.org/
- Argo CD: https://argo-cd.readthedocs.io/ — Flux: https://fluxcd.io/
- Unleash: https://www.getunleash.io/ — Flagsmith: https://www.flagsmith.com/
- OpenTelemetry: https://opentelemetry.io/docs/

