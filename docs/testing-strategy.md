# Strategia di Test (dettagliata)

Questa sezione definisce processi, strumenti e pratiche per garantire qualità e sicurezza in progetti AI/ML (incl. RAG, ReAct, agenti, LLM OSS). Include piramide dei test, pacchetti consigliati, test specifici per LLM/Agent e integrazione in CI/CD.

---

## Piramide di test

### 1) Unit (~70%) — veloci e deterministici
**Obiettivo:** verificare funzioni pure, adapter dei tool, parsing/formatting, rendering di template di prompt.

**Stack consigliato**

- PyTest (fixture, parametrizzazione, marker): https://docs.pytest.org/ 
- JSON Schema / pydantic per validare strutture dati (anche output LLM):  
  	- jsonschema (Python): https://python-jsonschema.readthedocs.io/  
  	- Pydantic v2: https://docs.pydantic.dev/

**Pattern essenziale (pytest + jsonschema)**
```python
# tests/test_prompt_rendering.py
import pytest
from jsonschema import validate

PROMPT = "You are a helpful assistant. Answer: {{answer}}"
SCHEMA = {"type": "object", "properties": {"answer": {"type": "string"}}, "required": ["answer"]}

def render_prompt(answer: str):  # funzione da testare
    return {"answer": answer}

@pytest.mark.parametrize("value", ["ok", "ciao", "42"])
def test_prompt_json_schema(value):
    out = render_prompt(value)
    validate(instance=out, schema=SCHEMA)  # fallisce se la struttura non è valida
```

---

### 2) Integration (~20%) — componenti insieme (DB, vector store, API)
**Obiettivo:** verificare i confini tra componenti: retriever+DB, chiamate HTTP, job ETL.

**Stack consigliato**

- Testcontainers (avvia DB/servizi reali in container durante i test):  
  	- Python: https://testcontainers-python.readthedocs.io/
- VCR.py (registra/riproduce HTTP in “cassette” deterministiche): https://vcrpy.readthedocs.io/
- Great Expectations (validazione schema/qualità dati prima del training/embedding): https://docs.greatexpectations.io/

**Esempio (Testcontainers + Postgres)**
```python
# tests/test_repo_pg.py
import psycopg2
from testcontainers.postgres import PostgresContainer

def test_repo_query():
    with PostgresContainer("postgres:16") as pg:
        conn = psycopg2.connect(pg.get_connection_url())
        cur = conn.cursor()
        cur.execute("CREATE TABLE t(x int); INSERT INTO t VALUES (1);")
        cur.execute("SELECT COUNT(*) FROM t;")
        assert cur.fetchone()[0] == 1
        conn.close()
```

**Mock HTTP idempotente (VCR.py)**
```python
# tests/test_external_api.py
import requests
import pytest

@pytest.mark.vcr()  # richiede il plugin pytest per VCR.py, o configurazione vcr nel conftest.py
def test_external_search():
    r = requests.get("https://example.com/api?q=abc", timeout=10)
    assert r.status_code == 200
```

---

### 3) E2E (~10%) — user journey e flussi agentici
**Obiettivo:** convalidare i percorsi critici (es. “query → retrieval → sintesi → risposta con citazioni”).

**Stack consigliato**

- Playwright (browser/API test, headless su CI, ottimo per app web): https://playwright.dev/python/
- OpenTelemetry (tracing di tool-calls, latenza, token e step ReAct durante gli E2E): https://opentelemetry.io/docs/

---

## Test specifici per LLM & Agent

### A) Golden tests (snapshot/golden-master)
**Idea:** fissare input e output attesi (con tolleranze) per template o classificatori.  
**Strumenti:**

- PyTest + snapshot/approval testing (diversi plugin)  
- promptfoo (runner CLI per suite di prompt con oracle/score): https://www.promptfoo.dev/docs/

### B) Schema tests (risposte strutturate)
**Idea:** il modello deve restituire JSON conforme a schema (tipi, campi obbligatori), con eventuale “re-ask” automatico.  
**Strumenti:**

- jsonschema: https://python-jsonschema.readthedocs.io/
- Guardrails (validatori semantici, schema-first, re-ask): https://www.guardrailsai.com/

```python
from jsonschema import validate

RESPONSE_SCHEMA = {
  "type": "object",
  "properties": {"title": {"type": "string"}, "score": {"type": "number"}},
  "required": ["title", "score"]
}

def assert_llm_json(resp: dict):
    validate(instance=resp, schema=RESPONSE_SCHEMA)
```

### C) Adversarial / sicurezza
**Obiettivo:** difendersi da prompt injection, disclosure di PII/system prompt, eccessiva agency.  
**Riferimenti:**

- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- promptfoo red-team (categorie attacchi pronte): https://www.promptfoo.dev/docs/red-team/

### D) Eval quantitativa RAG/Agent
**Obiettivo:** misurare faithfulness/grounding, rilevanza, somiglianza, latenza e costi.  
**Strumenti:**

- RAGAS (metriche RAG: faithfulness, answer relevancy, context recall, ecc.): https://docs.ragas.io/
- TruLens (eval + tracing catene/agent, integrabile con diversi framework): https://www.trulens.org/

```python
# pseudo: calcolo metriche RAGAS su un dataset Q/A con contesti
# from ragas import evaluate
# evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall, context_precision, answer_similarity])
```

---

## Processo operativo (step-by-step)

1. **Test Plan & Matrix**  
   Mappa per feature: Unit ↔ Integrazione ↔ E2E + (LLM-specific: golden/schema/adversarial/eval).

2. **Determinismo**  
   Seed fisso (Python/NumPy/torch), mocking oracoli esterni, VCR.py per HTTP, “freeze time” quando serve.

3. **Fixture dati**  
   Mini-dataset *golden* versionati in repo; Great Expectations per validare colonne/valori prima dell’indicizzazione.

4. **Dipendenze effimere**  
   Testcontainers per DB/queue/vector store; usare `GenericContainer` per engine non supportati nativamente.

5. **LLM mocking & cost control**  
   - **Offline**: golden + schema + cassette HTTP.  
   - **Online smoke**: chiamate reali a quota minima su PR etichettate (es. `e2e-llm`).

6. **E2E + Telemetria**  
   Playwright per journey; OpenTelemetry per spans sugli step agentici e tool-calls (debug rapido dei fallimenti).

7. **CI/CD**  
   Job separati: `unit` (obbligatorio), `integration` (con Testcontainers), `e2e` (condizionale / notturno).  
   Pubblicare report (coverage, ragas.csv, promptfoo.json).

---

## Snippet CI (GitHub Actions — estratto)

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -U pip pytest jsonschema
      - run: pytest -q

  integration:
    runs-on: ubuntu-latest
    services:
      docker:
        image: docker:27.0-dind
    steps:
      - uses: actions/checkout@v4
      - run: pip install pytest testcontainers[postgres] psycopg2-binary
      - run: pytest -q -m "integration" || true

  llm_eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ragas trulens-eval
      - run: python scripts/run_ragas_eval.py   # genera report (json/csv) da allegare come artifact
```

---

## Definition of Done — Checklist (feature AI)

- **Unit**
  	- Test ≥ 1 per funzione/adapter critico
  	- Output LLM validato via **JSON Schema** / **pydantic**
- **Integration**
  	- Retriever ↔ Vector store testato con **Testcontainers**
  	- HTTP esterno stabilizzato con **VCR.py**
- **E2E**
  	- 1 scenario “happy path” con **Playwright**
  	- **OpenTelemetry** attivo per tracing su step/strumenti agentici
- **LLM**
  	- **Golden** + **adversarial base** (prompt injection) + 1 run **RAGAS/TruLens** su subset
- **Data**
  	- **Great Expectations** su colonne chiave prima dell’indicizzazione
- **CI/CD**
  	- Job separati (unit/integration/e2e/llm_eval) e report pubblicati come artifact

---

### Riferimenti rapidi
- PyTest: https://docs.pytest.org/  
- jsonschema: https://python-jsonschema.readthedocs.io/  
- Pydantic: https://docs.pydantic.dev/  
- Testcontainers (Python): https://testcontainers-python.readthedocs.io/  
- VCR.py: https://vcrpy.readthedocs.io/  
- Great Expectations: https://docs.greatexpectations.io/  
- Playwright (Python): https://playwright.dev/python/  
- OpenTelemetry: https://opentelemetry.io/docs/  
- promptfoo: https://www.promptfoo.dev/docs/  
- Guardrails: https://www.guardrailsai.com/  
- RAGAS: https://docs.ragas.io/  
- TruLens: https://www.trulens.org/  
- OWASP Top 10 for LLM: https://owasp.org/www-project-top-10-for-large-language-model-applications/
