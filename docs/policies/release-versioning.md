# Release & Versioning

- **SemVer** (MAJOR.MINOR.PATCH)
- **CHANGELOG** obbligatorio
- **Release Train**: cadenza bisettimanale; hotfix fuori banda se P0/P1

Questo documento definisce **come versioniamo** e **come rilasciamo** software e asset AI in Manteia, per garantire prevedibilità, tracciabilità e rollback sicuri.

---

## 1) Regole di Versioning (SemVer)

Adottiamo **Semantic Versioning**: `MAJOR.MINOR.PATCH` (es. `1.4.2`).

- **MAJOR**: breaking changes (compatibilità **non** retro-compatibile)
- **MINOR**: nuove funzionalità **retro-compatibili**
- **PATCH**: bugfix, sicurezza, prestazioni **senza** modifiche all’API

**Pre-release**: `1.5.0-rc.1`, `2.0.0-beta.2` (solo su branch di release).  
**Build metadata** (facoltativo): `+build.shaabcdef` per tracciamento interno.

### 1.1 Cosa è “breaking” (linee guida pratiche)
- **API HTTP/GRPC**: cambio di schema, rimozione campi, semantica diversa, codici errore diversi.
- **SDK/CLI**: rimozione/rename di funzioni/flag; parametri obbligatori nuovi.
- **Formato output LLM/Agent**: modifica di **JSON schema**, rimozione di campi; *questo = MAJOR*.
- **Prompt/Tooling**: se cambia lo **schema contrattuale** (es. MCP manifest, input/output tool) ⇒ MAJOR.
- **DB/Indice vettoriale**: migrazione non backward-compatible (richiede reindex downtime) ⇒ MAJOR.

**NON-breaking (tipici MINOR/PATCH)**
- Aggiunta **campi opzionali** a strutture esistenti (documentati).
- Migliorie performance, logging, osservabilità senza cambiare contratti.
- Bugfix che non alterano l’interfaccia né l’output atteso.

### 1.2 Politica di Deprecation
- Deprecation annunciata in **CHANGELOG** e docs; flag/endpoint marcati `Deprecated` per **≥ 1 release MINOR** prima della rimozione.
- Warning a runtime (log + header `Deprecation` per API).

### 1.3 Compatibilità e supporto
- Supportiamo **ultime 2 MINOR** per ogni MAJOR attivo (es. `1.6.x` e `1.7.x`), salvo LTS esplicite.
- Patch di sicurezza/bug critici retro-portati quando sensato.

---

## 2) Versioning degli asset AI

### 2.1 Modelli
- Registrati in **Model Registry** con `model_name:model_version` e stage (`Staging`, `Production`).
- Il **SemVer del servizio** è separato dalla **versione del modello** (es. servizio `1.7.0` usa `manteia-qa:23`).

### 2.2 Prompt
- Ogni prompt ha `prompt_id@vN` e **schema di output** versionato (`schema@vN`).  
- Cambiare lo schema ⇒ **MAJOR** del servizio o routing condizionale → canary/flag.

### 2.3 Dataset & Indici
- **Dataset snapshot** con hash/versione (`dvc tag v2025.09.23`) e **data card**.
- **Vector index** con `index_name@vN` legato a: embedder, dimensione, parametri.  
  Modifica embedder/processing ⇒ **nuovo indice** (no in-place breaking).

### 2.4 Eval baselines
- Ogni release ha `eval_run_id` e metriche (es. RAGAS, task success rate).  
- Le soglie di accettazione sono parte dei **quality gates** (CI/CD).

---

## 3) CHANGELOG (obbligatorio)

Formato **Keep a Changelog** con sezioni:
- **Added**, **Changed**, **Fixed**, **Deprecated**, **Removed**, **Security**

Regole:
- Un entry per **ogni** modifica significativa.
- Riferimento a issue/PR (`#123`) e ad **ADR** se decisione architetturale.
- Generazione semi-automatica da **Conventional Commits** (es. `feat:`, `fix:`, `perf:`, `refactor:`, `chore:`).

Esempio ridotto:
```md
## [1.7.0] - 2025-09-23
### Added
- RAG reranker ibrido (bm25+dense) dietro feature flag `rerank_v2`.
### Changed
- /search include nuovo campo opzionale `source_score`.
### Fixed
- Bug rate-limit sul tool web-search in timeout bassi.
### Security
- Aggiornati openssl e uvicorn per CVE-XXXX.
```

---

## 4) Release Train (bisettimanale)

Cadenza standard: **mercoledì** settimane pari (Europa/Roma).

**Timeline tipo**
- **T-3gg (Freeze Parziale)**: niente nuove feature rischiose; solo bugfix/finishing.
- **T-2gg (RC)**: tag `vX.Y.Z-rc.1` su branch `release/X.Y`; deploy in **staging**, smoke & E2E.
- **T-0 (Go/No-Go)**: se OK ⇒ tag **annotato e firmato** `vX.Y.Z` su `main`, promozione in produzione (canary/blue-green).

**Ruoli**
- **Release Manager** (A): coordina calendario, check-list, approvazioni.
- **Tech Lead** (R): qualità tecnica, gating CI.
- **QA** (R): test E2E e non-regressioni.
- **PO** (A/C): scope e accettazione business.

**Note**
- Se CI non è verde, release posticipata o esclusione delle PR non conformi.
- Feature non pronte restano dietro **flag** o rimandate alla successiva train.

---

## 5) Hotfix (P0/P1)

- Trigger: incidenti P0/P1 (outage, regressione grave, sicurezza).  
- Branch: `hotfix/vX.Y.Z+1` da `main` **o** da tag ultimo GA.  
- Tag: `vX.Y.(Z+1)` (PATCH), rilasciato direttamente in produzione dopo smoke test.  
- Backport: merge **forward** in `main` e in branch di release attivi, aggiornando CHANGELOG.

**SLA di massima**
- P0: fix in ore, approvazione snella (1 reviewer + Release Manager).  
- P1: entro 24–48h, percorso standard ma prioritario.

---

## 6) Tagging, Branching, Artifacts

### 6.1 Branching (trunk-based)
- `main`: sempre **releasable** (protetto, CI obbligatoria).
- `feature/*`, `fix/*`, `chore/*`: PR piccole, review 1–2 approvatori.
- `release/X.Y`: stabilizzazione pre-GA; contiene solo bugfix, doc, hardening.

### 6.2 Tagging
- Tag **annotati e firmati**: `vX.Y.Z` (git tag -a -s).  
- Pre-release: `vX.Y.Z-rc.N` solo su branch di release.

### 6.3 Artifacts
- **Container**: `org/svc:vX.Y.Z` e `:git_sha` (entrambi); firma **cosign**, **SBOM** allegata.  
- **Package**: wheel/pypi privato; npm registry privato.  
- **Charts/Manifests**: Helm chart versionato (`Chart.yaml`), OCI artifact.  
- **Model/Dataset**: registri dedicati con meta (metriche, dipendenze, compatibilità).

---

## 7) Automazione della release

- **CI gates**: lint, types, unit/integration, security scans, build, SBOM, firma.  
- **CHANGELOG** e **version bump** automatici (es. derived da Conventional Commits).  
- **Release Notes** generate e pubblicate (GitHub Releases).  
- **Promozione**: staging → prod con **canary/blue-green**; rollback 1 comando.

---

## 8) Compatibilità & Policy di rimozione

- Ogni breaking richiede **MAJOR** e **piano di migrazione** documentato.  
- Deprecazioni annunciate ≥ **1 MINOR** prima della rimozione.  
- API/CLI: versionamento esplicito (`/v1/`), parallel run per 1–2 train quando necessario.

---

## 9) Check-list di Release

**Prima del tag**
- [ ] CI verde (tutti i jobs)
- [ ] Coverage ≥ soglia, zero test flakey noti
- [ ] CHANGELOG aggiornato e rivisto
- [ ] Migrazioni DB pronte con **rollback** testato
- [ ] Dashboard & alert pronti (SLO/SLI definiti)
- [ ] Note di release draftate (impatti, migrazioni, flag)

**Dopo il tag**
- [ ] Immagini firmate + SBOM pubblicata
- [ ] Deploy in **staging**, smoke OK
- [ ] Canary in produzione + monitoraggio 30–60’
- [ ] Promozione al 100% o **rollback** eseguito
- [ ] Post-release: chiusura milestone, retro tecnica, aggiornamento Roadmap

---

## 10) Esempi di bump

- Aggiungo campo **opzionale** in risposta API → `MINOR` (es. `1.6.0` → `1.7.0`)
- Modifico schema JSON output LLM → **MAJOR** (`1.7.0` → `2.0.0`)
- Fix bug di serializzazione → `PATCH` (`1.7.0` → `1.7.1`)

---

## 11) Domini multi-repo/monorepo

- **Monorepo**: tag per pacchetto (`svc-a@1.2.0`, `lib-b@0.9.1`) o workspace tool.  
- **Multi-repo**: coordination board per Release Train; matrice compatibilità tra versioni.

---

## 12) Documentazione & ADR

- Ogni breaking/decisione rilevante ha **ADR** associata.  
- Le note di release linkano ad ADR, PR e runbook di migrazione.

---