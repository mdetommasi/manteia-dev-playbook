# Visione, Missione, Valori Tecnici, Principi

## **Visione**  
Accelerare l'impatto del business Manteia con soluzioni AI **affidabili**, **manutenibili** e **scalabili**.

## **Missione**  
Mettere in produzione sistemi AI moderni (agentic RAG, ReAct, LLM OSS, MCP) con un **toolkit uniforme** e **processi ripetibili**, dalla sperimentazione al rilascio.

---

## **Valori Tecnici**
- **Qualità prima della velocità**: test, revisione, osservabilità *by default*.  
- **Semplicità deliberata**: evitare complessità non necessarie, scegliere standard aperti.  
- **Automazione**: CI/CD, linting, security scan, data checks.  
- **Trasparenza**: ADR, metriche chiare, documentazione vicina al codice.  
- **Responsible AI**: valutazioni, guardrail, privacy, reproducibility.

---

## **Principi**

### **OODA Loop**  
> *Combatti il nemico con il suo stesso ritmo.*

**Spiegazione**  
**Ciclo decisionale rapido**: Osserva → Orientati → Decidi → Agisci. Ripeti più veloce del contesto o dell’avversario.

**Come applicarlo**  
- Dedica max **20% del tempo all’analisi**, **80% all’azione**.  
- Aggiorna l’orientamento ad ogni nuovo dato rilevante.  
- Usa **micro-test** o spike per validare decisioni.

**Esempio concreto**  
> Bug critico in produzione:  
> 1. **Osserva** log e metriche → 2. **Orientati** (ipotesi: cache satura) → 3. **Decidi** (svuota cache) → 4. **Agisci** → loop finché risolto.

**Segnale di allarme** ⚠️  
> **Paralisi da analisi**: se passi più di 30 min senza agire, **forza un’azione minima**.

---

### **SHU HA RI**  
> *Assimila → Ripeti → Trascendi*

**Spiegazione**  
**Percorso marziale di apprendimento**: imita fedelmente (SHU), aggiungi consapevolezza nella ripetizione (HA), innova con padronanza (RI). Evita salti di fase.

**Come applicarlo**  
- **SHU**: segui il playbook del team senza varianti.  
- **HA**: adatta solo dopo **3 cicli di successo**.  
- **RI**: proponi nuovi pattern solo dopo averli **testati in produzione**.

**Esempio concreto**  
> Onboarding di un dev:  
> - Settimane 1–4: **SHU** (scrivi test come nel template).  
> - Mese 2: **HA** (aggiungi helper personali).  
> - Mese 6: **RI** (proponi nuovo framework di test).

**Segnale di allarme** ⚠️  
> **“RI” prematuro**: junior che riscrive tutto senza aver mai seguito le regole.

---

### **Destinazione Chiara, Percorso Incerto**  
> *Sappiamo dove, non come.*

**Spiegazione**  
**Definisci l’obiettivo finale** (OKR, KPI, milestone) ma lascia **flessibilità sul percorso**. Evita micro-pianificazione in contesti volatili.

**Come applicarlo**  
- Scrivi l’obiettivo in **1 riga** (es. “Ridurre latenza <50ms”).  
- Definisci **vincoli chiari** (budget, SLA, deadline).  
- Concedi al team **libertà tattica**.

**Esempio concreto**  
> Feature X deve essere live entro il **15/12**.  
> Il team sceglie GraphQL invece di REST → consegna in **10 gg invece di 14**.

**Segnale di allarme** ⚠️  
> **Gantt dettagliato oltre 2 settimane** → sintomo di percorso rigido.

---

### **Fallisci Presto, Costruisci Solido**  
> *Errore piccolo oggi > disastro domani.*

**Spiegazione**  
**Sperimenta in piccolo, misura, itera**. Il fallimento è un dato; la solidità nasce dal **feedback rapido**.

**Come applicarlo**  
- Ogni ipotesi = **spike <4 ore**.  
- Usa **feature flag** per nuovi flussi.  
- **Post-mortem obbligatori** su ogni fallimento (anche piccolo).

**Esempio concreto**  
> Nuova UI:  
> - Giorno 1: mock Figma + test con **5 utenti**.  
> - Giorno 2: **60% abbandono** → kill → pivot.  
> Risparmio: **3 settimane di sviluppo**.

**Segnale di allarme** ⚠️  
> “Funziona in locale” **senza rollout parziale** → ricetta per outage.
