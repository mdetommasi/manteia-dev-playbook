# Runbook — Incident Response

1. **Triage**: severità, blast radius, canale di guerra, owner on-call
2. **Mitigazione**: rollback/canary freeze, rate-limit
3. **Diagnosi**: log/traces, ultime release, dipendenze
4. **Risoluzione**: fix + test
5. **Postmortem**: timeline, 5 whys, azioni correttive, owner e due-date

**Scopo:** gestire incidenti in produzione (RAG/Agent/LLM e servizi correlati) minimizzando impatto su utenti e business, ripristinando lo stato stabile e prevenendo recidive.

---

## 0) Principi chiave
- **Safety first:** proteggi dati/PII e riduci il blast radius prima della diagnosi profonda.
- **Una sola regia:** nomina subito l’**Incident Commander (IC)**. Tutte le decisioni passano dall’IC.
- **Comunicazioni chiare e time-boxed:** aggiornamenti regolari (es. ogni 15–30’) su canale dedicato.
- **Evidence-based:** log/trace/metriche prima delle ipotesi; timeline precisa.
- **Fix forward quando possibile, rollback quando necessario** (rapido e verificabile).
- **Postmortem blameless** con azioni correttive tracciate e con owner/date.

---

## 1) Ruoli & responsabilità
- **Incident Commander (IC):** coordina triage/mitigazione/diagnosi, decide priorità e comunica stato.
- **On-call Owner (tecnico):** esegue comandi, rollback, patch temporanee; delega a SME quando serve.
- **Comms Lead:** aggiorna stakeholder (interni/esterni), mantiene status page e incident ticket.
- **Scribe:** registra timeline, decisioni, metriche (può essere il Comms Lead in team piccoli).
- **SME (LLM/RAG/DB/Infra/Sec):** supporto specialistico per area.

> Nota: in turni notturni/weeekend i ruoli possono essere cumulati; l’IC resta **uno**.

---

## 2) Classificazione & SLA
| Severità | Criterio                                                | Esempi                                                    | SLA iniziale |
|---------:|---------------------------------------------------------|-----------------------------------------------------------|-------------:|
| **P0**   | Interruzione completa / rischio legale/PII critico      | API 5xx > 50%, leak PII, agent che esegue azioni errate   | Triage ≤ 5’  |
| **P1**   | Degradazione ampia, impatto clienti paganti              | p95 latency x2, risposte non “faithful” su casi critici   | Triage ≤ 15’ |
| **P2**   | Impatto limitato o servizio non core                     | una feature KO, costo anomalo ma sotto soglia             | Triage ≤ 60’ |
| **P3**   | Graffio minore / bug a basso impatto                     | cosmetic, log rumorosi                                    | Pianifica    |

**SLO minimi di riferimento:** Availability ≥ 99.5%, p95 LLM gen ≤ 2500ms, p95 retrieval ≤ 1200ms, faithfulness ≥ soglia.

---

## 3) Triage (entro i primi minuti)
**Checklist rapida**
1. **Nomina IC** e apri **war room**: `#war-incident-YYYYMMDD-HHMM` (Slack) + link Meet/Zoom.
2. **Blast radius:** chi è impattato? (tenant, regione, prodotto). % richieste in errore, PII a rischio?
3. **Severità preliminare** (P0–P3) e **ticket** (INC-YYYY-####).
4. **Owner on-call** assegnato.
5. **Freeze** su deploy/canary se necessario.
6. **Comms v0** (template sotto) con sintesi e prossimo aggiornamento (es. +15’).

**Template messaggio iniziale**
```
[INC-2025-0923-001][P1] Sintesi: aumento 5xx su /v1/chat (p95 +80%) dalle 10:14 CET.
Impatto: ~35% richieste EU tenants A/B. Mitigazione in corso. Prossimo update: 10:30.
IC: <nome>; On-call: <nome>; War room: #war-incident-20250923-1015
```

---

## 4) Mitigazione (stabilizza prima di investigare)
**Opzioni comuni (scegli in base al sintomo):**
- **Rollback** ultima release *o* **freeze canary** (resta al 0–10%).  
  - Helm: `helm rollback <rel> <rev>`  
  - Argo CD: `argocd app rollback <app> <rev>`  
  - Kubectl: `kubectl rollout undo deploy/<svc>`
- **Feature flag kill switch** sulla funzionalità/agent incriminato.
- **Rate-limit / shed load** (tenant-based o globale).  
  - Gateway: abbassa RPS, imposta `429 + Retry-After`.
- **Riduzione costi/risorse** (incidenti di saturazione): diminuisci `max_new_tokens`, **disabilita tool non essenziali**, alza soglie di cache.
- **Isolamento tenant** (multi-tenant): taglia traffico verso tenant-fonte di attacco/abuso, preservando gli altri.
- **Guardrail hardening** per incidenti *content-safety/PII*: abilita filtri severi, forza output JSON schema, disattiva tool scrittura esterna.

> Documenta ogni azione in timeline (chi, quando, output).

---

## 5) Diagnosi (quando l’impatto è sotto controllo)
**Checklist dati**
- **Metriche**: QPS, 5xx rate, p50/p95, tokens_in/out, cache_hit, tool_latency, vector_store_latency.
- **Traces** (OTel): step agent (retrieve → synth → tool-call), errori/codici, tempi per span.
- **Log**: errori recenti, spike, RI pattern; confronta *prima/dopo* release.
- **Ultime release**: commit/PR, prompt/versioni, cambi di modello, parametri (temp/top-p), tool/MCP, schema VS.
- **Dipendenze**: DB/VS/LLM serving (vLLM/TGI), rete, quota GPU/CPU, limiti container.

**Comandi utili (K8s)**
```bash
kubectl get pods -n <ns>
kubectl logs deploy/<svc> -n <ns> --since=30m
kubectl describe deploy/<svc> -n <ns>
kubectl rollout history deploy/<svc> -n <ns>
```

**LLM/RAG specifici**
- Verifica **rotture prompt** (diff tra versioni), **drift embeddings** o schema vector store cambiato.
- Controlla **guardrails** (schema JSON, PII filters) e **tool** (time-out/malfunzionamenti).

---

## 6) Risoluzione (fix)
1. **Repro minimo** (test o script) → valida l’ipotesi root cause.
2. **Fix** con **test** (unit/integration) + **golden/adversarial** se agente/LLM.
3. **Canary** del fix (5–10%) + **monitoraggio** SLO per X minuti.
4. **Promozione** a 100% se stabile.
5. **Rimuovi mitigazioni temporanee** (rate-limit, flag, freeze), *una per volta* con controllo.

**Checklist di pre‑merge (estratto)**
- [ ] Test verdi (incl. LLM eval minima su scenari affetti)
- [ ] Changelog/ADR aggiornati se necessario
- [ ] Rollback plan pronto (versione precedente ancora deployabile)

---

## 7) Postmortem (entro 48–72h) — blameless
**Struttura consigliata**
- **Titolo/ID**: INC-YYYY-#### — Severità
- **Sintesi** (1–2 paragrafi) e impatto (utenti, % richieste, durata, costi)
- **Timeline** (UTC/CET): eventi, segnali, decisioni, rollback/fix
- **Root Cause**: analisi tecnica + **5 Whys**
- **Cosa ha funzionato** / **cosa no**
- **Azioni correttive** (prevenzione/detection/operability) con **owner** e **due-date**
- **Follow-up**: task di hardening (guardrail, rate-limit, test, osservabilità)
- **Allegati**: grafici, query, PR/commit, log redatti

**Template 5 Whys (mini)**
```
1. Perché è successo?
2. Perché la causa 1 è stata possibile?
3. Perché non è stata rilevata prima?
4. Perché i controlli/guardie non hanno impedito l’impatto?
5. Cosa impedisce la recidiva in futuro?
```

---

## 8) Comunicazioni (modelli)
**Update periodico (interno)**
```
[INC-2025-0923-001][P1][T+30’] Stato: mitigazione attiva (rate-limit 20% EU).
Prossimi passi: rollback v1.12.3 se p95 > 2.5s entro 10’. Prossimo update: 11:00.
```
**Chiusura incidente**
```
[INC-2025-0923-001] Risolto alle 11:42 CET. Root cause: regressione nel retriever.
Impatto: 32% richieste EU tra 10:14–11:42. Azioni: fix rilasciato (v1.12.4), postmortem entro 72h.
```

**Status page (breve)**
- Sintesi, orario inizio/fine, impatto, stato attuale, ETA prossimo update, link postmortem.

---

## 9) Playbook rapidi (estratti)
**Rollback**
```bash
# Helm
helm history <release> -n <ns>
helm rollback <release> <revision> -n <ns>

# Argo CD
argocd app history <app>
argocd app rollback <app> <revision>

# Kubernetes (deploy standard)
kubectl rollout history deploy/<svc> -n <ns>
kubectl rollout undo deploy/<svc> -n <ns>
```

**Rate-limit (gateway generico)**
```yaml
# Esempio concettuale (Envoy/NGINX): 100 rps per tenant
rate_limit:
  key: "tenant_id"
  requests_per_unit: 100
  unit: "second"
  action_on_exceed: "429"
```

**Feature flag kill switch (pseudo)**
```yaml
features:
  agent_tools_write_external: false
  rag_citations_strict: true
```

---

## 10) Ready-to-Use Checklist (stampabile)
- [ ] IC nominato, war room aperta, ticket creato
- [ ] Severità e blast radius definiti
- [ ] Mitigazione iniziale attivata (rollback/freeze/flag/rate-limit)
- [ ] Log/trace/metriche raccolti
- [ ] Root cause validata con repro
- [ ] Fix sviluppato + test + canary e monitoraggio
- [ ] Comunicazioni regolari e chiusura con stato “risolto”
- [ ] Postmortem blameless pubblicato con azioni/owner/date
- [ ] Follow-up tracciati (Jira/GitHub Issues) e verificati a scadenza

---

## 11) Appendice LLM/Agent-specific
- **PII/Content leak**: attiva PII scrubber, alza severità filtri, disabilita tool esterni, verifica prompt/template.
- **Hallucinations spike**: controlla retriever/vector store, drift embeddings, config decoding; forza schema output.
- **Cost spike**: riduci max tokens/temperature, abilita cache, quota per tenant; verifica routing modello.
- **Timeout tool**: imposta circuit-breaker e retry esponenziale; degrada la capability (risposta parziale + avviso).

---


