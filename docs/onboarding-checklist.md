# Onboarding Checklist (New Joiner)

- Accessi: repo, CI, registri artifact, vault, dashboard
- Dev env: SDK, pre-commit, container runtime
- Leggere: vision, ways-of-working, 02-testing, ci-cd
- Fare: primo ticket “Good first issue”, aprire 1 PR
- Shadowing: 1 sprint con pair programming

# Onboarding Checklist — New Joiner (Manteia AI)

Benvenuta/o! Questa checklist ti guida nei **primi 10–15 giorni**, così da renderti produttiva/o in fretta e in sicurezza.

---

## 0) Overview (cosa fare in ordine)

1. **Accessi** (SSO/MFA) → repo, CI, registri artifact, vault, dashboard.
2. **Dev env** → SDK, pre-commit, runtime container.
3. **Leggere** → vision, ways-of-working, testing, CI/CD.
4. **Fare** → 1 ticket *Good First Issue*, aprire 1 PR piccola.
5. **Shadowing** → 1 sprint in pair programming con rotazione.

---

## 1) Accessi (Day 1)

- [ ] **SSO + MFA** attivi (IdP aziendale)
- [ ] **GitHub org** (repo privati, Projects, wiki)
- [ ] **CI** (GitHub Actions / runner), permessi minimi
- [ ] **Artifact Registry**:
  - [ ] Container (es. GHCR o ECR)
  - [ ] Pacchetti (Python wheel, npm)
  - [ ] **MLflow** Model Registry (lettura/scrittura)
  - [ ] **DVC** remote (dataset snapshot)
- [ ] **Vault / Secret Manager** (policy *least privilege*)
- [ ] **Dashboards** di osservabilità:
  - [ ] Grafana (metriche, SLO/SLI)
  - [ ] Loki/Kibana (log)
  - [ ] Langfuse/Phoenix (LLM traces)
- [ ] **Ticketing/Boards** (GitHub Projects/Jira) con i template

> Se qualcosa manca, apri un ticket *Access Request* (template in `.github/ISSUE_TEMPLATE`).

---

## 2) Dev Environment (Day 1)

### 2.1 Requisiti base

- [ ] **Git** (commit signing consigliato)
- [ ] **Python 3.11** (pyenv o sistema), **Node 20** (nvm), **Make**
- [ ] **Docker** (o Podman) funzionante (`docker ps`)
- [ ] **Pre-commit** installato globalmente o per repo
- [ ] **SSH keys** o **HTTPS + token** configurati

### 2.2 Setup rapido (Unix/macOS; adatta per Windows)

```bash
# Python
pyenv install -s 3.11.9 && pyenv local 3.11.9
python -m venv .venv && source .venv/bin/activate
pip install -U pip

# Repo
git clone <git@github.com:manteia/<repo>.git> && cd <repo>
pip install -r requirements-dev.txt  # o `uv pip sync` se usate uv
pre-commit install

# Node
nvm install 20 && nvm use 20
npm ci  # se c'è frontend/tooling

# Docker
docker ps

# DVC & MLflow (se presente)
dvc pull  # scarica dataset snapshot
export MLFLOW_TRACKING_URI=<url>
```

> **Nota:** non committare mai segreti; usa **Vault/Secret Manager** e variabili d’ambiente locali.

---

## 3) Conoscere il Playbook (Day 1–2)

- [ ] **Vision** → `docs/00-vision.md`
- [ ] **Ways of Working** → `docs/01-ways-of-working.md`
- [ ] **Testing Strategy** → `docs/02-testing-strategy.md`
- [ ] **CI/CD** → `docs/03-ci-cd.md`
- [ ] **Security & Governance** → `docs/policies/05-security-privacy.md`  
- [ ] **Blueprint** (RAG/Agent/MCP) → `docs/blueprints/`

Facoltativo: avvia la doc localmente

```bash
pip install mkdocs mkdocs-material
mkdocs serve  # http://127.0.0.1:8000
```

---

## 4) Fare (First Issue & First PR) — Day 2–3

- [ ] Prendi un ticket **“Good First Issue”** (label `good first issue`) dal board.
- [ ] Segui i **template** PR/Issue (`.github/`):
  - PR **piccola** (200–400 LOC), descrizione con *why/what/how*
  - Allegare test e screenshot/log quando utile
- [ ] Rispetta **quality gates**:
  - [ ] Lint/format (ruff, black / eslint, prettier)
  - [ ] Type-check (mypy / tsc)
  - [ ] Test `pytest`/`vitest` e **coverage** (≥ 80% quando sensato)
  - [ ] Security scans (bandit, pip-audit/npm audit; trivy su container)
  - [ ] Build container (se applicabile)

### Comandi tipici

```bash
pytest -q && coverage run -m pytest && coverage report -m
ruff check . && black --check . && mypy src
npm run lint && npm run test --if-present
```

---

## 5) Shadowing (1 sprint in pair) — Day 3–10

**Obiettivo:** imparare *doing by doing*, ridurre bus factor.

- [ ] **Assegnazione buddy** (Dev/QA/UX a seconda del ruolo)
- [ ] **Rotazione pair**: almeno 2 moduli/aree nel periodo (es. API + RAG ingest)
- [ ] **Sessioni**: 2–3 al giorno, 60–90’ (driver/navigator)
- [ ] **Checklist tecnica** in ogni sessione:
  - [ ] Convenzioni di codice e naming
  - [ ] Test-first (piramide: unit → integrazione)
  - [ ] Tracing/metriche dove manca (*observability by default*)
  - [ ] PR piccola a fine giornata quando possibile
- [ ] **Retro di metà sprint** (15’) con feedback bidirezionale
- [ ] **Demo finale** al Review: cosa hai toccato, cosa hai imparato

---

## 6) Sicurezza & Compliance (Day 1–7)

- [ ] **MFA ovunque**, password manager
- [ ] **Secret scanning**: attiva pre-commit `gitleaks` e verifica CI
- [ ] **PII**: leggi le regole di data handling; logging privacy-aware
- [ ] **SBOM & Supply-chain**: leggi policy firma immagini (Cosign), scans (Trivy)
- [ ] **Accessi minimi**: chiedi *solo* ciò che serve, revoca accessi inutilizzati

---

## 7) Comunicazione & Rituali

- [ ] Join canali Slack: `#manteia-ai-eng`, `#squad-<nome>`
- [ ] Meeting fissi (Europa/Roma):
  - **Daily** 15’ (10:00) — blocchi e micro-commitment
  - **Planning/Review/Retro/Refinement** a calendario
- [ ] *Thread-first*: condividi decisioni in **ADR** e link nelle PR

---

## 8) Definition of Ready/Done (promemoria)

**DoR (prima di iniziare)**  

- [ ] AC chiari e testabili  
- [ ] Dipendenze note/dati disponibili o mock  
- [ ] Stima assegnata, rischi mappati

**DoD (per chiudere)**  

- [ ] Lint/Types/Test OK (≥80% quando sensato)  
- [ ] Docs/ADR aggiornati  
- [ ] Observability base (metriche/log/traces)  
- [ ] Security scans e CI **verde**  

---

## 9) Obiettivi di fine primo sprint

- [ ] 1–2 PR **mergeate** su aree diverse (es. bug + piccola feature)
- [ ] Copertura test aumentata dove mancava
- [ ] 1 **ADR** (anche minore) redatta con buddy
- [ ] Accessi completi e ambienti locali pronti
- [ ] Partecipazione attiva a Review/Retro con feedback

---

## 10) Appendix — setup utile

### Git (firma consigliata)

```bash
git config --global user.name "<Nome Cognome>"
git config --global user.email "<email@azienda.com>"
git config --global commit.gpgsign true   # oppure ssh signing
```

### Docker login (GHCR)

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin
```

### Pre-commit (install globale)

```bash
pipx install pre-commit  # o pip install --user pre-commit
```

### MkDocs (doc locale)

```bash
pip install mkdocs mkdocs-material && mkdocs serve
```

---
