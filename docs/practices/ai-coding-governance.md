# Generative AI per la scrittura del codice (“vibe coding”) — Linee guida e governance (Manteia)

Questa sezione definisce **pratiche, limiti e controlli** per l’uso dell’AI generativa nella produzione di codice in Manteia. L’obiettivo è **accelerare** lo sviluppo mantenendo **sicurezza, qualità, compliance** e **tracciabilità**.

---

## 1) Obiettivi
- **Produttività**: scaffolding rapido, refactor guidato, generazione test e doc.
- **Qualità**: output verificabile, standard coerenti, difetti in calo.
- **Sicurezza**: nessuna esposizione di segreti/PII, dipendenze sicure.
- **Compliance**: rispetto licenze OSS e proprietà intellettuale.
- **Auditabilità**: chi ha generato cosa, con quali modelli, quando e perché.

---

## 2) Ambito & definizioni
- **Vibe coding**: uso dell’AI per suggerire/creare codice, test, script, doc, query.
- **Assistita vs autonoma**: l’AI **assiste**; l’approvazione finale è **umana**.
- **Modelli**: LLM OSS self-hosted (vLLM/TGI) o provider esterni tramite **gateway aziendale**.

---

## 3) Casi d’uso: consentiti / limitati / vietati
**Consentiti (default)**
- Boilerplate/scaffolding (API, DTO, config), refactor non-breaking, test unitari/integrazione, migrazioni dati generate e **revisionate**, docstring/README, script DevOps controllati.
- Query SQL/DSL **non produttive** (provvisorie) su dump **sanificati**.

**Limitati (richiede attenzione/approvazione)**
- Generazione di **policy di sicurezza**, criptografia, autenticazione → revisione **Security Champion**.
- Codice che manipola **dati sensibili** o percorsi **critici** di business → 2° reviewer obbligatorio, test E2E.
- Tool/agent con **accesso a sistemi esterni** → MCP con policy allow-list e rate-limit.

**Vietati**
- Incollare nel prompt **segreti, chiavi, PII** o porzioni estese di codice proprietario non necessari al task.
- Accettare output AI senza **test** e **review**.
- Copiare codice generato che replica **identico** snippet OSS senza rispettare la **licenza**.

---

## 4) Flusso operativo standard (PR AI-assisted)
1. **Prompt sicuro**: contesto minimo, no segreti/PII; usa snippet ridotti & mock.
2. **Generazione**: modello approvato (router aziendale); specifica *cosa* e *perché*.
3. **Autoverifica**: l’autore esegue lint, type-check, test; aggiunge note in PR su cosa è AI-generated.
4. **Review umana**: almeno 1–2 reviewer; attenzione a sicurezza, performance, correttezza semantica.
5. **Quality gates CI**: lint/format, type-check, test/coverage, **security scans**, build container.
6. **Merge**: solo con CI verde; **tag** di tracciamento (vedi §10).

---

## 5) Prompting & contesto (do/don’t)
**Do**
- Specificare **contratti**: firma funzione, tipi, interfacce, AC (Given/When/Then).
- Imporre **stile** (PEP8/ruff, eslint/prettier), performance target, error handling.
- Richiedere **test** e **docstring** nel risultato.

**Don’t**
- Incollare config `.env`, token, dataset con PII, dump DB reali.
- Fornire **repository completi** come contesto; preferire file selezionati o API schema.

---

## 6) Architettura di adozione (enterprise)
- **LLM Gateway** aziendale con: routing modelli (OSS/commerciali), **policy DLP**, **redaction** segreti, logging, quote, e **MCP** per tool standard.
- **Self-hosting** (vLLM/TGI/Ollama) per codice sensibile; vendor esterni come **fallback**.
- **Sandbox** di esecuzione (container non privilegiati) per “code runner” e valutazioni rapide.
- **RAG di codice** (facoltativo): indicizzare **solo** repo interni sanitizzati; separare per tenant/permessi.

---

## 7) Sicurezza & DLP
- **Redaction automatica** di pattern segreti nei prompt/output (API keys, credenziali, PII).
- **Secret scanning** (gitleaks/trufflehog) su PR; **policy OPA** su file sensibili.
- **Dependency scanning** (pip-audit/npm audit, trivy per container) e **SBOM** (syft) obbligatori.
- **Network egress** controllato: modelli esterni solo tramite gateway; **OIDC** per identity.
- **Log privacy-aware**: niente prompt completi in log; **hash** o partial redaction per audit.

---

## 8) IP & licenze
- Usare **modelli/strumenti approvati** e repository di **snippet interni**.
- Scansione **similarità/OSS license** (es. scancode-toolkit/FOSSA/Snyk) su PR rilevanti.
- Se un output coincide con codice OSS, rispettare **licenza** (header, NOTICE) o rigenerare.
- Evitare contaminazione **copyleft** in librerie che richiedono linkage statico non desiderato.
- Documentare la **fonte** quando l’AI si ispira a esempi pubblici noti.

---

## 9) Metriche & SLO (misurare l’impatto)
- **Lead time PR** e **tempo review** (atteso in calo).
- **Defect rate** post-merge (atteso invariato o in diminuzione).
- **Coverage** e tasso test flakey.
- **Costo/token** per riga accettata (budget per team).
- **Adozione**: % PR con tag AI-assisted; **churn** da rigenerazioni.

---

## 10) Tracciabilità & audit
- Annotare PR con label `ai-assisted` e commit trailer:  
  `Co-authored-by: manteia-ai-bot <ai@manteia>`
- **Trace ID** del gateway nel body PR (modello, versione, costi, latenza).
- Conservare **prompt sintetico** (non sensibile) e parametri (temperature/top-p/router).

---

## 11) Quality bar (accettazione)
- **Lint/format** e **type-check** passano.
- **Test**: unit per ogni funzione generata; integrazione/E2E se tocca percorsi critici.
- **Security scans** senza issue High/Critical non giustificate.
- **Performance**: nessuna regressione misurabile nei benchmark del modulo.
- **Docs**: docstring/README aggiornati; **ADR** se decisione tecnica rilevante.

---

## 12) Ruoli & responsabilità (RACI sintetico)
- **Developer (R)**: prompt sicuro, autoverifica, PR.
- **Reviewer (R)**: controlli qualità/sicurezza, feedback.
- **Tech Lead (A)**: approvazione finale, eccezioni ai limiti.
- **Security Champion (C/A per security)**: verifica su codice sensibile/crypto/auth.
- **MLOps/Platform (C)**: gestione gateway, costi, modelli approvati.
- **Legal/OSS (C)**: policy licenze e revisione casi dubbi.

---

## 13) Modelli e tool approvati (esempi)
- **Modelli OSS**: `Manteia-OSS-8B` (self-host vLLM), `bge/gte` per embedding, `bge-reranker` per ranking.
- **Provider**: endpoint esterni (OpenAI/Anthropic/Azure) solo via **gateway** con DLP attivo.
- **IDE**: estensioni AI solo se configurate per usare il **gateway**; disabilitato l’accesso diretto.
- **Eval**: promptfoo per suite di prompt; **TruLens/RAGAS** per qualità nei moduli AI.

---

## 14) Anti-pattern (da evitare)
- “**Paste & pray**”: incollare output senza comprenderlo.
- “**Prompt infinito**”: incollare repo intero nel contesto.
- “**No tests, no review**”: saltare le barriere di qualità.
- “**Segreti nel prompt**”: qualunque forma.
- “**Dipendenze casuali**”: introdurre librerie senza scansione/ADR.

---

## 15) Maturity model (adozione per fasi)
1. **Pilot**: pochi moduli, tracciamento e controlli manuali, cost ceiling.
2. **Standard**: gateway centralizzato, policy DLP, tagging PR, metriche nel dashboard.
3. **Avanzato**: RAG di codice interno, MCP tools per refactor/sicurezza, A/B di prompt, budget dinamico.

---

## 16) DoD per PR AI-assisted
- [ ] Prompt sicuro (niente segreti/PII)
- [ ] Lint/Types/Test **OK** (≥80% quando sensato), coverage invariata o in crescita
- [ ] Security scans **OK** (no High/Critical aperti)
- [ ] Docs/ADR aggiornati se necessario
- [ ] Label `ai-assisted` + trailer commit + trace ID gateway
- [ ] Review approvata, **CI verde**

---

**Owner:** Tech Lead + Security Champion (Activity Owner)  
**Ultimo aggiornamento:** 2025-09-27
