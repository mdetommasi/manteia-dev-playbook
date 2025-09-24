# MLOps / LLMOps per RAG, Agent e Modelli

- **Data Versioning**: DVC/LakeFS; dataset card, lineage e checksum.
- **Model Registry**: MLflow o equivalente, con stage (Staging/Production) e metriche.
- **Serving LLM OSS**: vLLM/TGI, Ollama per dev locale; throttling e quota per tenant.
- **Vector Store**: Weaviate, Qdrant, PgVector; schemi, TTL, politica di re-embedding.
- **Prompt Management**: versioning, A/B Prompt, canary.
- **Agent Eval**: harness di scenari, tool-ablation, replay di conversazioni reali.
- **Guardrail**: PII scrubber, output schema, content policy, rate-limit adattivo.

Questa guida definisce pratiche, componenti e automazioni per portare in produzione sistemi **RAG**, **Agent** e modelli **AI/ML** adottando un approccio **MLOps/LLMOps** rigoroso e ripetibile.

---

## 1) Scope & principi
- **Riproducibilità** end‑to‑end (codice + dati + artefatti).
- **Promozioni progressive** (Staging → Production) con **canary/blue‑green** e rollback.
- **Osservabilità by default** (metriche/log/trace + costi).
- **Sicurezza & governance**: PII, segreti, audit, policy prompt/dataset.
- **Multi‑tenant**: limiti di costo/throughput e configurazioni isolate.

---

## 2) Data Versioning (DVC/LakeFS)
Scegli tra:
- **DVC**: controllo versione a livello di file/granularità dataset, remoto (S3/MinIO/Azure/GCS), pipeline `dvc.yaml`.
- **LakeFS**: *git‑like* su oggetti (S3 compatibile), branch/tag/PR sui dati; ottimo per ambienti condivisi.

### 2.1 Setup DVC (esempio)
```bash
dvc init
dvc remote add -d storage s3://manteia-dvc # o minio://...
dvc add data/raw/sat3_eps.parquet
git add data/raw/.gitignore sat3_eps.parquet.dvc .dvc/config
git commit -m "data: track SAT3 EPS raw"
dvc push
```

### 2.2 Dataset Card (template minimo)
```markdown
# Dataset Card — SAT3_EPS_v2025_09
- **Fonte**: Telemetria EPS SAT3 (stream X)
- **Schema**: time, window, batt_volt_narrow_range, batt_charge_curr, ...
- **Qualità**: % missing, outlier policy, unità di misura
- **PII**: none | masked | pseudonymized (policy link)
- **Licenza**: internal Manteia
- **Checksum**: sha256: <...>  | **DVC rev**: <git_sha>
- **Lineage**: raw → cleaned → features_v3 (link a pipeline)
```

### 2.3 Lineage & checksum
- Mantieni **manifest** (es. `manifest.json`) con **hash**, righe, range temporali.
- In DVC usa `dvc.yaml` per dichiarare step (ETL → features → embeddings) e tracciare dipendenze/artefatti.

---

## 3) Model Registry (MLflow o equivalente)
- **Run**: parametri, metriche e artefatti (confusion matrix, fig, token usage).
- **Model**: versione con **signature/schema**, **conda/requirements** e **flavors**.
- **Stages**: `None` → `Staging` → `Production` (→ `Archived`).

### 3.1 Logging & registrazione (esempio Python)
```python
import mlflow
from mlflow.models import infer_signature

with mlflow.start_run():
    # log params/metrics
    mlflow.log_param("model", "hssae_v0")
    mlflow.log_metric("val_mae", 0.034)
    # log model
    signature = infer_signature(X_valid[:10], y_pred[:10])
    mlflow.sklearn.log_model(model, "model", signature=signature, registered_model_name="SoH_Estimator")
```

### 3.2 Promozione automatica (gate)
- Promuovi a **Staging** se: *val_loss* < soglia, *eval* RAGAS/TruLens ok, **security checks** passati.
- Promuovi a **Production** dopo **canary** ≥ N richieste con SLO rispettati.

---

## 4) LLM Serving & Inference Gateway
- **vLLM**: throughput alto, paginazione KV‑cache, multi‑tenant.
- **TGI (Text Generation Inference)**: server robusto per modelli Hugging Face.
- **Ollama**: dev locale, modelli leggeri (es. Llama‑3.*‑instruct) per PoC.

### 4.1 Pattern di deploy
- **Kubernetes** con HPA su **tokens/sec** o **QPS**; nodi GPU/CPU misti.
- **Gateway** (reverse proxy) con **routing** per canary e **quota** per tenant.
- **Caching**: prompt caching (exact/sim hash) per FAQ/template.

### 4.2 Quota & throttling (multi‑tenant)
- Budget **tokens/month** e **RPS** per `tenant_id`.
- **Leaky Bucket / Token Bucket** per rate‑limit (429 + `Retry-After`).
- **Cost guard**: stop soft al 80%, hard al 100% (alert + blocco richieste extra).

### 4.3 Telemetria
- Metriche: **latency p50/p95**, **tokens_in/out**, **errors**, **cache_hit**, **tool_latency** (per agent).
- Trace OTel con **span** per: retrieval, generation, tool call, re‑rank.

---

## 5) Vector Store (Weaviate, Qdrant, PgVector)
**Scelta rapida**:
- **Weaviate**: schema‑first, module‑rich, cloud/self‑hosted.
- **Qdrant**: HNSW + payload JSON potente, ottimo OSS.
- **PgVector**: integrazione SQL, comodo se Postgres è già standard.

### 5.1 Schema & metadata
- Collezioni per **dominio** e/o **tenant**: `kb_<tenant>_<domain>`.
- Campi base: `id`, `text`, `embedding`, `source`, `chunk_id`, `doc_id`, `lang`, `acl`.
- **TTL** su payload o *soft delete* con `deleted_at`.

### 5.2 Re‑embedding policy
- Trigger:
  1. **Upgrade modello** (es. `text-embed-v2 → v3`).
  2. **Drift contenuti** (basso recall / alto self‑BLEU tra chunk vecchi/nuovi).
  3. **SLA degradata** (precision@k sotto soglia).
- Strategia:
  - **Dual‑index** temporaneo (vecchio+nuovo) → **shadow eval** → **cutover**.
  - Job batch notturni; priorità per documenti più richiesti.

### 5.3 Parametri indice (esempi)
- Qdrant HNSW: `ef_construct=128`, `m=16`, `ef=64` (query).
- Weaviate: `vector_index_type: hnsw`, `pq.compression: true` per trade‑off memoria/latency.
- PgVector: `ivfflat lists=100` (tuna in base alla dimensione).

### 5.4 Chunking & ingest
- **Lunghezza**: 300–800 token, **overlap** 10–15%.
- Normalizza: rimuovi boilerplate, estrai tabelle, Markdown/HTML → testo pulito.
- **Doc ID** stabile per lineage e citazioni.

---

## 6) Prompt Management (versioning, A/B, canary)
- Prompt **template** (Jinja/Handlebars) con **parametri** e **guardie** (es. max token).
- **Versioning** in Git: `prompts/<task>/<name>@v<semver>.prompt`.
- **A/B**: distribuzioni pesate (50/50, 80/20). **Canary** 5–10% su nuove versioni.
- **Prompt Router**: seleziona la variante in base a `tenant|locale|persona|traffic`.
- **Telemetry**: log di variante, outcome, costi e latenza → analisi uplift.

**Esempio router (YAML)**
```yaml
task: answer_with_citations
variants:
  - id: p_v1
    weight: 0.9
    template: prompts/answer_v1.prompt
  - id: p_v2_canary
    weight: 0.1
    template: prompts/answer_v2.prompt
```

---

## 7) Agent Evaluation
- **Harness di scenari**: set di task realistici (CRUD conoscenza, multi‑hop, tool‑use, richieste ambigue).
- **Tool ablation**: esegui gli stessi scenari **senza** uno strumento (o con *fail injection*) per misurare impatto.
- **Replay**: riproduci conversazioni reali (sanitizzate) con **seed deterministici**.
- **Metriche**:
  - **Task success rate**, **groundedness/faithfulness**, **hallucination rate**
  - **Steps per task**, **tool error rate**, **latency p95**, **costo** per task
- **Online**: *shadow mode* (risposte non mostrate ma valutate), **canary** live su subset utenti.

---

## 8) Guardrail & Safety
- **PII scrubber** (regex + NER): sanifica input/log; *hash pseudonimi*.
- **Output schema**: valida JSON con **JSON Schema**; *re‑ask* automatico se invalido.
- **Content policy**: filtri categorie (toxicity/NSFW/PHI); blocklist/allowlist fonti.
- **Rate‑limit adattivo**: backoff su saturazione GPU/queue depth; riduzione token/max_new_tokens su overload.
- **Audit log**: request/response ids, model/prompt version, tool‑calls, *redaction status*.
- **Segreti**: in vault; mai in repo/log. OIDC per deploy.

---

## 9) Pipeline & CI/CD (automazioni)
- **Data & Embeddings** (notte o su evento): ETL → validate (Great Expectations) → embed → upsert VS → report.
- **Model training/eval**: train → eval (RAGAS/TruLens) → log MLflow → gate → register.
- **Serving**: build immagine (vLLM/TGI) → firma (Cosign) → deploy Staging → canary → prod.
- **Prompt update**: PR con test/golden → canary 10% → promozione.

**Estratto GitHub Actions — Data & Embeddings**
```yaml
jobs:
  embeddings:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt
      - name: Build embeddings
        run: python pipelines/build_embeddings.py --config configs/embeddings.yaml
      - name: Post metrics
        run: python scripts/post_metrics.py artifacts/embeddings_report.json
```

---

## 10) Monitoring & SLO
- **SLO** suggeriti:
  - **Availability** (99.5%+) endpoint /health e /v1/chat
  - **Latency p95**: ≤ 1200 ms (RAG retrieval); ≤ 2500 ms (LLM gen) *per tenant*
  - **Accuracy** (RAG): faithfulness ≥ soglia su set di validazione
  - **Costo**: budget/tenant; **tokens per richiesta** mediana e p95
- **Dashboard** (per servizio):
  - QPS, error rate, p50/p95, tokens_in/out, cache_hit, tool_latency, vector_store_latency
- **Tracing**:
  - *Span* per step (retrieve → re‑rank → synthesize → guardrail) con **correlation_id** end‑to‑end.

---

## 11) Snippet & reference

### 11.1 Qdrant (docker‑compose)
```yaml
version: "3.8"
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333", "6334:6334"]
    volumes: ["./qdrant:/qdrant/storage"]
```

### 11.2 vLLM (K8s deployment, estratto)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-serve
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: vllm
          image: ghcr.io/org/vllm:latest
          args: ["--model", "/models/llama-3-instruct", "--tensor-parallel-size", "2"]
          resources: { limits: { nvidia.com/gpu: 1 } }
          env:
            - name: VLLM_LOGGING_LEVEL
              value: INFO
```

### 11.3 JSON Schema (output enforcement)
```json
{
  "type": "object",
  "required": ["title", "answer", "citations"],
  "properties": {
    "title": { "type": "string" },
    "answer": { "type": "string" },
    "citations": {
      "type": "array",
      "items": { "type": "string", "format": "uri" },
      "minItems": 1
    }
  }
}
```

### 11.4 Prompt Card (template)
```markdown
# Prompt Card — answer_with_citations@v1.2
- **Goal**: risposta aderente con citazioni
- **Audience**: analisti interni
- **Inputs**: user_query, retrieved_chunks
- **Constraints**: max_tokens=800, no PII, tono professionale
- **Known failures**: riferimenti legislativi vecchi
- **Owner**: team‑rag
- **Changelog**: v1.2 → migliorata gestione tabelle
```

---

## 12) Checklist di adozione (DoD MLOps/LLMOps)
- Dati **versionati** (DVC/LakeFS) + **dataset card** + checksum.
- **Model registry** attivo (MLflow): signature, metriche, stage.
- **Serving**: gateway con routing/canary, quota per tenant, telemetria.
- **Vector store**: schema/TTL, job **re‑embedding** e policy documentata.
- **Prompt mgmt**: versioning + A/B + canary + prompt card.
- **Agent eval**: harness scenari, replay, tool ablation; report periodico.
- **Guardrail**: PII scrubber, output schema, content policy, rate‑limit adattivo.
- **CI/CD**: pipeline per data, embeddings, modelli e deploy; rollback provato.
- **Monitoring**: dashboard + alerting + cost tracking per tenant.

---

## 13) Struttura consigliata nel repo
```text
mlops/
├─ pipelines/
│  ├─ build_embeddings.py
│  └─ retrain_model.py
├─ configs/
│  ├─ embeddings.yaml
│  └─ serving-router.yaml
├─ prompts/
│  └─ answer_v1.prompt
├─ docs/
│  └─ dataset_cards/
└─ scripts/
   ├─ post_metrics.py
   └─ canary_promote.py
```
