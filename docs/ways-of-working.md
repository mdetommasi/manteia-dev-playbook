# Ways of Working (Scrum-based) — Dettagli operativi

Linee guida pratiche per organizzare lavoro, processi e qualità del team Manteia AI.

---

## 1) Eventi Scrum (cadenza suggerita)

- **Sprint**: 2 settimane (lun→ven). Obiettivi chiari, outcome misurabili.
- **Daily (15')**: blocchi, rischi, micro‑commitment al giorno; niente status meeting esteso.
- **Backlog Refinement (60–90')**: 1–2 volte per sprint; grooming di epiche/story; DoR.
- **Sprint Planning (90’–120’)**: definire *Sprint Goal*, capacity, commitment; stime/agreement.
- **Sprint Review (45–60')**: demo end‑to‑end, metriche, feedback stakeholder.
- **Retrospective (45–60')**: miglioramenti concreti con owner e due date (Start/Stop/Continue, 5 Whys).

**Time‑boxing & regole**
- Hard stop, agenda pre‑condivisa, note in repo (cartella `docs/cerimonie`).
- Ogni meeting chiude con decisioni/owner/next step (no “TODO” vaghi).

---

## 2) Ruoli & responsabilità (RACI sintetico)

- **Product Owner (PO)**: vision, priorità backlog, accettazione. *A/R* su scope.
- **Scrum Master**: rimuove impedimenti, cura processo. *A* su cerimonie.
- **Tech Lead**: qualità tecnica, architetture, code review avanzate. *A/R* su debt e standard.
- **Developer**: implementazione, test, doc, osservabilità. *R* su feature.
- **QA/Tester**: strategia di test, E2E, qualità dati. *R* su coperture.
- **UX**: ricerca/design, prototipi, naming. *R* su coerenza UI/UX.
- **MLOps/Platform**: CI/CD, serving, monitoring, costi modelli. *R* su pipeline.

---

## 3) Artefatti e workflow

- **Product Backlog**: epiche/story con *benefit* e *Acceptance Criteria (AC)* chiari.
- **Sprint Backlog**: selezione basata su capacity reale (ferie, on‑call, ecc.).
- **Board Kanban**: colonne minime `Todo → In Progress → In Review → In Test → Done`, WIP limit su “In Progress”.
- **DoR (Definition of Ready)** — una story è pronta se:
	- Obiettivo e valore utente chiari
	- AC elencati e testabili
  	- Dipendenze note, dati disponibili o simulabili
  	- Stima (story points) assegnata
  	- Impatti sicurezza/PII/infra dichiarati
- **DoD (Definition of Done)** — una story è *Done* se:
  	- Lint/format **OK** (pre‑commit), **type‑check OK**
  - **Test**: unit + integrazione (e2e se percorso critico); **coverage ≥ 80%** quando sensato
  - **Docs** aggiornate (README/CHANGELOG); **ADR** per decisioni chiave
  - **Observability**: metriche/log/traces minime e allarmi base
  - **Security**: scan dipendenze/secret, check di compliance (se applicabile)
  - **CI verde** su PR e *review* approvata

---

## 4) Branching & PR policy

- **Trunk‑based** con *feature branches*:
  - `feature/<slug>`, `fix/<slug>`, `chore/<slug>`
- **PR piccole** (target 200–400 LOC), descrizione con *why/what/how*, screenshot/log quando utile.
- **Review**: 1–2 approvatori; usa **CODEOWNERS** per aree.
- **Quality gates** obbligatori: lint, types, unit, integrazione chiave, security scans, build container.
- **Merge**: `squash` con *conventional commit* (`feat:`, `fix:`, `chore:`).

**Template PR (estratto)**
```md
## Scopo
Breve descrizione del valore/risultato.

## Changes
- …

## Test
- Unit: …
- Integration: …
- E2E (se critico): …

## Checklist
- [ ] CI verde (lint/types/tests)
- [ ] Docs/ADR aggiornati
- [ ] Security scans eseguite
```

---

## 5) Issue types, stima e priorità

- **Feature**: valore utente; AC chiari.
- **Bug**: passi di riproduzione, atteso vs ottenuto, log/screenshot.
- **Tech‑debt**: rischio/beneficio, effort, piano di riduzione.
- **Research spike** (time‑boxed, es. 4–8h): *deliverable* = nota/PoC + decisione.
- **Stima**: Story Points (Fibonacci 1,2,3,5,8,13). Spike **senza** punti, solo tempo.
- **Priorità**: `P0` (bloccante) → `P3` (nice‑to‑have).

**Label suggerite**
- `type:feature|bug|debt|spike`, `area:api|ui|rag|mlops|data`, `priority:P0|P1|P2|P3`, `risk:low|med|high`

---

## 6) Accettazione & AC (Acceptance Criteria)

- Gherkin/BDD semplice (Given/When/Then) **o** checklist puntuale.
- Per AI/RAG: definire *metriche minime* (es. **faithfulness**, **latency p95**, **token cost**).

**Esempio AC**
```md
Given un documento sorgente
When interrogo l’assistente con una domanda coperta dal documento
Then la risposta contiene una citazione corretta
And il tempo di risposta p95 < 1200ms
And nessun PII viene esposto nei log
```

---

## 7) Working Agreements (team)

- **Focus time**: ogni giorno 2 blocchi da 90' senza meeting.
- **Fusi orari**: Europa/Roma come riferimento; finestre sincrone 10:00–12:00, 15:00–17:00.
- **Comunicazione**: canali Slack dedicati per squad (`#squad-…`), *thread‑first*, decisioni riepilogate in ADR.
- **Pair/Mob programming**: almeno 1 sessione/sett per onboarding o parti complesse.
- **Bus factor**: nessuna area critica senza backup; rotazione “owner modulo”.

---

## 8) Qualità tecnica & sicurezza (by default)

- **Static checks**: ruff/black/mypy, eslint/prettier/tsc.
- **Security**: bandit, pip‑audit/npm audit, secret scan (gitleaks), SCA su container (trivy), SBOM (syft).
- **Data/PII**: classificazione, mascheramento log, policy di retention.
- **Observability**: OpenTelemetry per traces; metriche minime (latency, error rate, token usage).

---

## 9) AI‑specific (RAG/Agent/LLM)

- **Prompt card** e **Dataset card** versionate nel repo.
- **Eval harness** nello sprint: golden, schema tests, adversarial base; **RAGAS/TruLens** per regressioni.
- **Guardrail**: JSON schema/Guardrails, filtri PII, contenuti; test negoziazione tool (MCP) e fallback.
- **Canary** per modelli/parametri (temperature/top‑p/router) con rollback rapido.

---

## 10) Metriche di processo (Sprint Health)

- **Velocity** (trend 3 sprint), **commitment reliability** (% done/committed).
- **Lead time** PR, **review time**, **defect rate** (bug P0/P1 per sprint).
- **Coverage** e **test flakiness**.
- **Costo infra/modelli** (budget sprint, p95 costo/call).

---

## 11) Esempi pratici

**Template story (estratto)**
```md
### User Story
Come <ruolo> voglio <bisogno> così da <valore>.

### Acceptance Criteria
- …
- …

### Test Plan
- Unit: …
- Integration: …
- E2E: …

### Observability
- Metriche: …
- Log/Traces: …

### Security & Data
- PII: sì/no; misure: …
- Permessi/secret: …
```

**Esempio DoR/DoD (checklist pronta)**
```md
## DoR
- [ ] AC chiari e testabili
- [ ] Dipendenze note
- [ ] Dati disponibili/mock
- [ ] Stima assegnata
- [ ] Impatti sicurezza/infra noti

## DoD
- [ ] Lint/Types/Test OK (≥80% quando sensato)
- [ ] Docs/ADR aggiornati
- [ ] Observability base attiva
- [ ] Security scans eseguite
- [ ] Review approvata, CI verde
```

---

## 12) Collegamento con CI/CD

- **PR gate**: i check DoD sono riflessi nei job CI (lint/types/tests/security/build).
- **Staging auto** al merge in `main`, **Review** dimostrabile su ambiente.
- **Retro**: ogni azione di miglioramento di processo ha issue/owner e scadenza.

---

## 13) Tooling consigliato (promemoria rapido)

- **Gestione lavoro**: GitHub Projects/Issues con template e label.
- **PR template/Issue template/ADR**: in `.github/` e `docs/templates/`.
- **Pre‑commit**: ruff/black/mypy/eslint/prettier.
- **Observability**: OpenTelemetry, dashboard SLO/SLI.
- **Security**: gitleaks, bandit, pip‑audit/npm audit, trivy, syft/grype.
- **Eval AI**: RAGAS, TruLens, promptfoo, Guardrails.
