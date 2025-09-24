# Manteia AI Engineering Playbook — v0.1
**Data:** 2025-09-23

Benvenuti nel playbook tecnico per il nuovo team AI di **Manteia**.  
Questo repository raccoglie linee guida, blueprint e template per costruire applicazioni AI/ML **production-grade** con approcci **agili (Scrum)**, **CI/CD**, **architetture open-source** e paradigmi moderni (**ReAct agents, agentic RAG, LLM open-source, Model Context Protocol**).

---

## Obiettivi
- Allineare team cross-funzionali (Dev, QA, UX, MLOps) su **processi e standard**.
- Accelerare l’andata in produzione con **blueprint riusabili** e **policy** chiare.
- Garantire **manutenibilità, robustezza, scalabilità** e **responsible AI**.

## Pubblico
- **Developer & Tech Lead**: standard di coding, architetture, testing.
- **QA/Tester**: strategia di test LLM/agent, E2E, qualità dati.
- **UX**: linee guida di handoff, osservabilità UX, metriche.
- **MLOps/Platform**: CI/CD, serving modelli, osservabilità e costi.

---

## Struttura del repo

```text
.
├─ docs/                      # manuali, policy, pratiche e blueprint
│  ├─ vision.md
│  ├─ ways-of-working.md
│  ├─ testing-strategy.md
│  ├─ ci-cd.md
│  ├─ mlops-llmops.md
│  ├─ onboarding-checklist.md
│  ├─ blueprints/
│  │  ├─ 10-agentic-arch-blueprint.md
│  │  ├─ 11-mcp-playbook.md
│  │  └─ 12-rag-blueprint.md
│  ├─ policies/
│  │  ├─ 05-security-privacy.md
│  │  ├─ 07-data-prompt-governance.md
│  │  └─ 08-release-versioning.md
│  ├─ practices/
│  │  ├─ 06-observability.md
│  │  ├─ coding-standards-python.md
│  │  └─ coding-standards-typescript.md
│  ├─ runbooks/
│  │  └─ incident-response.md
│  └─ templates/
│     ├─ ADR-template.md
│     └─ project-structure.md
├─ .github/
│  ├─ ISSUE_TEMPLATE/
│  │  ├─ bug_report.md
│  │  └─ feature_request.md
│  ├─ workflows/
│  │  └─ ci.yml                # pipeline CI di esempio
│  └─ PULL_REQUEST_TEMPLATE.md
├─ CHANGELOG.md
└─ README.md                   # questo file
```

> Per una panoramica rapida leggi **[vision.md](vision.md)** e **[ways-of-working.md](ways-of-working.md)**.

---

## Come usare il playbook

### 1) Onboarding rapido
- Segui **[onboarding-checklist.md](onboarding-checklist.md)**.
- Configura pre-commit (ruff/black/mypy/eslint/prettier).
- Esegui i test locali e verifica gli standard.

### 2) Avvio documentazione locale (MkDocs)
```bash
pip install mkdocs mkdocs-material
mkdocs serve
# apri http://127.0.0.1:8000
```

### 3) Crea un progetto da template
- Consulta **[template/14-project-structure.md](template/14-project-structure.md)**.
- Clona lo scheletro, imposta `<package>`/`<project_name>`, abilita CI.

---

## Ways of Working (Scrum-based)
- **Eventi:** Sprint (2 settimane), Daily (15'), Planning, Review, Retro, Refinement.
- **Artefatti:** Product Backlog, Sprint Backlog, DoR/DoD.
- **Branching:** trunk-based + feature branches (`feature/...`, `fix/...`, `chore/...`).
- **PR Policy:** 1 feature = 1 PR piccola, code review obbligatoria, **CI verde**.
- **Issue Types:** feature, bug, tech-debt, research spike.
- **DoD:** lint/format, type-check OK, test (≥80% quando sensato), docs/ADR aggiornate, observability minima.
- Dettagli: **[ways-of-working.md](ways-of-working.md)**.

---

## Strategia di Test
- **Piramide:** Unit (~70%), Integration (~20%), E2E (~10%).
- **LLM/Agent:** golden tests, schema/guardrails, adversarial (prompt injection/PII), eval (RAGAS/TruLens).
- **Tooling:** pytest/coverage, testcontainers, VCR.py, Playwright, OpenTelemetry.
- Dettagli: **[testing-strategy.md](testing-strategy.md)**.

---

## CI/CD
- **CI (su PR):** lint/format (ruff, black, eslint), type-check (mypy/tsc), unit/integration test (pytest/vitest), coverage; security scans (bandit, pip-audit/npm audit, trivy); build immagine Docker.
- **CD:** staging automatica su merge in `main`; canary/blue-green; migrazioni DB con rollback; feature flags.
- **Artifact registry:** container, wheel/npm, dataset snapshot (DVC), modelli (MLflow).
- Dettagli: **[ci-cd.md](ci-cd.md)**.

---

## MLOps / LLMOps
- **Data versioning:** DVC/LakeFS, dataset card.
- **Model registry:** MLflow con stage e metriche.
- **Serving LLM OSS:** vLLM/TGI/Ollama; throttling/quota.
- **Vector store:** Qdrant/Weaviate/PgVector; re-embedding policy.
- **Guardrail:** PII scrubber, schema output, content policy.
- Dettagli: **[4-mlops-llmops.md](mlops-llmops.md)**.

---

## Security, Privacy & Governance
- **Segreti:** vault; mai in codice/CI logs; OIDC per deploy.
- **PII:** classificazione, minimizzazione, masking in log.
- **Supply-chain:** SBOM (Syft), scans CVE (Grype/TruVy), firma immagini (Cosign).
- **Prompt/Data governance:** prompt card/dataset card, retention conversazioni.
- Dettagli: **[policies/05-security-privacy.md](policies/05-security-privacy.md)**,  
  **[policies/07-data-prompt-governance.md](policies/07-data-prompt-governance.md)**.

---

## Blueprint architetturali
- **Agentic RAG + ReAct:** coordinator agent, retriever ibrido, tools via MCP, osservabilità end-to-end.  
  → **[blueprints/10-agentic-arch-blueprint.md](blueprints/10-agentic-arch-blueprint.md)**
- **MCP Playbook:** contratti, sicurezza, test di handshake.  
  → **[blueprints/11-mcp-playbook.md](blueprints/11-mcp-playbook.md)**
- **RAG e2e:** ingest/index/query/synthesis/eval.  
  → **[blueprints/12-rag-blueprint.md](blueprints/12-rag-blueprint.md)**

---

## Decisioni architetturali (ADR)
- Usa il **template ADR** per ogni decisione chiave (modello, vector store, protocolli).  
  → **[templates/ADR-template.md](templates/ADR-template.md)**  
- Stato, contesto, decisione, conseguenze, alternative.

---

## Contribuire
1. Crea una branch `feature/...` e apri una PR piccola.
2. Assicurati che **CI sia verde** e aggiorna **docs/ADR** se necessario.
3. Segui i **template PR/Issue** in `.github/`.
4. Versioning: **SemVer** + **CHANGELOG** (vedi **[policies/08-release-versioning.md](policies/08-release-versioning.md)**).

---

## Incident Response
- Runbook con triage, mitigazione, diagnosi, risoluzione e postmortem.  
  → **[runbooks/incident-response.md](runbooks/incident-response.md)**

---

## Roadmap (alto livello)
- v0.2: cookiecutter progetto agentic-RAG, dashboard SLO/SLI di esempio.
- v0.3: harness di valutazione automatica per agent, red-team suite ampliata.
- v1.0: guida di migrazione multi-tenant e best practice di costo per LLM serving.

---

## Licenza e note
- Licenza interna Manteia (o specificare licenza OSS se pubblico).
- I diagrammi e i template sono esempi: **adattare** ai requisiti di servizio e compliance locali.

---

## Contatti
- **Owner Playbook:** _Activity Owner_ / Tech Lead di riferimento  
- **Canale interno:** `#manteia-ai-eng` (annunci, domande, proposte)
- **Contribuzioni:** apri una issue/PR seguendo i template in `.github/`

> **Suggerimento:** mantieni il playbook **vicino al codice**. Ogni nuova feature dovrebbe aggiornare (se serve) **test, docs e ADR** nello stesso branch/PR.
