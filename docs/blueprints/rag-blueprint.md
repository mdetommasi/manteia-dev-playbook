# Blueprint — Retrieval-Augmented Generation (RAG)

- **Ingest**: pipeline chunking+embedding; qualità (dedup, PII scrub)
- **Index**: schema con metadati (source, timestamp, access level)
- **Query**: retriever ibrido + re‑ranker; caching
- **Synthesis**: template controllati, citazioni obbligatorie
- **Eval**: faithfulness, grounding rate, answer similarity, latency

Guida pratica per progettare, sviluppare e operare un sistema **RAG production‑grade** con affidabilità, scalabilità e governance. Copre ingest, indicizzazione, query & ranking, sintesi con citazioni, valutazione e osservabilità.

---

## 0) Visione architetturale (alto livello)


```
Clients
  └─> API Gateway / BFF
        └─> RAG Service (stateless)
              ├─ Retrieval Layer (Hybrid: BM25/Keyword + ANN Vector)
              │     ├─ Lexical Search (Elasticsearch / OpenSearch / Vespa)
              │     └─ Vector DB (Qdrant / Weaviate / pgvector / Milvus)
              ├─ Ranking Layer (Cross-Encoder Re-ranker)
              │     └─ bge-reranker / Cohere Rerank / Jina Reranker
              ├─ Synthesis Layer (LLM Serving + Templates + Guardrails)
              │     ├─ LLM Serving (vLLM / TGI / Ollama / Vendor API)
              │     ├─ Output Schema (JSON Schema / Pydantic / Guardrails)
              │     └─ Citations Enforcement (chunk_id + uri obbligatori)
              ├─ Policy & Safety Layer
              │     ├─ PII/Secrets Filter (Presidio / custom)
              │     └─ Content Policy (moderation / allow-list)
              ├─ Caching & Rate-Limit
              │     ├─ Context/Answer Cache (Redis)
              │     └─ Token Budgets + KV-Cache
              ├─ Observability
              │     ├─ Tracing (OpenTelemetry / Langfuse / Phoenix)
              │     └─ Metrics (Prometheus/Grafana), Logs (Loki)
              └─ Eval & RAGOps
                    ├─ Offline Eval (RAGAS/TruLens, golden set)
                    ├─ Online Experiments (A/B, Canary)
                    └─ Feedback Loop (thumbs, issue tags)

Data Plane (Ingest & Index)
  ├─ Connectors/Parsers (unstructured, Tika, trafilatura, OCR)
  ├─ Cleaning & PII scrub (Presidio) + Dedup (simhash/minhash)
  ├─ Chunking (recursive/semantic/layout-aware)
  ├─ Embedding Jobs (bge/gte/e5) + Cache
  └─ Index Build/Rotate (versioned: docs_vN) → Vector DB / Search

Platform & Ops
  ├─ Orchestrazione (Airflow/Prefect per batch; Kafka+KEDA per stream)
  ├─ CI/CD (GitHub Actions → Helm/ArgoCD; policy OPA; firma Cosign)
  ├─ Secrets (Vault/Secret Manager; OIDC), SBOM (Syft) + CVE Scan (Trivy)
  └─ Multitenancy (namespaces/ABAC; RLS su DB; quote per tenant)
```
---

## 1) Ingest

### 1.1 Sorgenti & parsing
- **Documenti**: PDF, HTML, DOCX, MD, email; 
- **Dati strutturati**: DB tabelle, CSV/Parquet; 
- **API** e **crawler**.
- **Tool**: `unstructured`, Apache Tika, `trafilatura` (HTML), `pytesseract`/OCR, `layoutparser` (struttura pagina).

### 1.2 Cleaning & qualità
- **Dedup** (simhash/minhash), normalizzazione encoding, rimozione boilerplate.
- **PII scrub**: Microsoft **Presidio** (detector+anonymizer) o equivalente.
- **Quality gates**: lunghezza minima, lingua, entropia/testi generati, blacklist domini/regex.

### 1.3 Chunking
- **Strategie**: 
    - *Recursive/semantic* (titoli + contesto), 
    - *Fixed window + overlap*, 
    - *Layout‑aware* (tabella/figura/caption).
- **Parametri tipici**: `chunk_size` 512–1.024 token, `overlap` 10–20%.
- **Metadata carry‑over**: mantieni path, sezione, pagina, heading.

### 1.4 Embedding
- **Modelli** (OSS): `bge-*`, `gte-*`, `e5-*`, `SFR-*` (text-embedding). 
- **Parametri**: normalizzazione L2, dimensione; batch async + **caching** (chiave = sha256 testo).
- **Ottimizzazioni**: quantizzazione int8/bf16 su GPU servente; **rate-limit** e **retry/backoff**.

### 1.5 Orchestrazione ingest
- **Batch**: Airflow/Prefect (DAG idempotenti, checkpoint, backfill).
- **Stream**: Kafka + consumer ingest; autoscaling (KEDA) per picchi.
- **Veridica**: Great Expectations su schemi/colonne prima di indicizzare.

---

## 2) Index

### 2.1 Schema con metadati
- Campi **obbligatori**: `doc_id`, `chunk_id`, `text`, `embedding`, `source`, `uri`, `timestamp`, `access_level`, `section`, `hash`.
- Campi **opzionali**: `author`, `tags`, `lang`, `pii_flags`, `valid_from/to`, `tenant_id`.

**Esempio (JSON)**
```json
{
  "doc_id": "DOC-123",
  "chunk_id": "DOC-123#p5_h2_c3",
  "text": "…",
  "embedding": [0.12, -0.03, …],
  "source": "handbook",
  "uri": "s3://bucket/handbook.pdf#page=5",
  "timestamp": "2025-09-23T10:00:00Z",
  "access_level": "internal",
  "section": "Policies/Security",
  "tenant_id": "acme",
  "hash": "sha256:…"
}
```

### 2.2 Motori di ricerca / Vector DB
- **Qdrant / Weaviate**: HNSW IVF, filtraggio su metadata, payload ricco.
- **pgvector** (Postgres): buon fit per stack SQL; usa **GIN/IVFFlat**.
- **Elasticsearch/Vespa**: ibrido nativo (BM25 + ANN) e DSL potente.

**Schema Qdrant (estratto, Python)**
```python
from qdrant_client import QdrantClient, models as qm
client = QdrantClient(host="qdrant", port=6333)
client.recreate_collection(
  collection_name="docs",
  vectors_config=qm.VectorParams(size=1024, distance=qm.Distance.COSINE),
  optimizers_config=qm.OptimizersConfigDiff(memmap_threshold=20000)
)
```

**pgvector (SQL)**
```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE docs (
  doc_id text, chunk_id text PRIMARY KEY,
  text text, embedding vector(1024),
  source text, uri text, timestamp timestamptz,
  access_level text, section text, tenant_id text
);
CREATE INDEX ON docs USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX ON docs (tenant_id, access_level, source);
```

### 2.3 Versioning & migrazioni
- **Index version**: `docs_v3` (nuovo embedder o parametri) → build parallelo, dopodiché *switch*.
- **Cold/Warm rebuild**: pipeline idempotente; segnala *re‑embedding backlog* e progress.

---

## 3) Query

### 3.1 Retriever ibrido
- **Sparse** (BM25/lexical) **+ Dense** (ANN su embeddings). 
- Fusione: **RRF** (reciprocal rank fusion) o score mix pesato.
- Filtri metadata: `tenant_id`, `access_level`, `time_range`, `source in (…)`.
- Diversificazione: **MMR** per ridurre ridondanza.

### 3.2 Re‑ranking
- **Cross‑encoder** (es. `bge-reranker*`, `cohere‑rerank`, `jina‑reranker`) su top‑k (es. 50→10).
- Trade‑off latenza/costo: valuta batch inferencing e soglie.

### 3.3 Caching
- **Cache query→context** (Redis) con TTL breve (minuti) e invalidazione su reindex.
- **Cache output LLM** per prompt deterministici (chiave = template+inputs+top_k+model_id).

### 3.4 Query rewriting & feedback
- **HyDE** (hypothetical doc expansion), **PRF** (pseudo‑relevance feedback), **spelling/normalize**.
- *Session memory* opzionale: contesto conversazionale controllato (limiti e TTL).

**Pseudocodice (Python)**
```python
q = normalize(user_query)
hits_sparse = bm25.search(q, filters={...}, k=100)
hits_dense = vectordb.ann(q, filters={...}, k=100)
hits = rrf_merge(hits_sparse, hits_dense, k=60)
reranked = cross_encoder.score(q, hits)[:10]
context = assemble(reranked, budget_tokens=2_000)
```

---

## 4) Synthesis

### 4.1 Prompting controllato
- **Template** con istruzioni chiare e **citazioni obbligatorie** (id chunk + uri).
- **Guardrail/Schema**: JSON Schema / Pydantic / Guardrails; gestisci *re‑ask* su output invalido.
- **Stili**: answer breve vs dettagliata; *tone* coerente con brand.

**Esempio output (JSON)**
```json
{
  "answer": "…",
  "citations": [
    {"chunk_id": "DOC-123#p5_h2_c3", "uri": "…", "span": "riga 12-18"},
    {"chunk_id": "KB-9#sec2", "uri": "…"}
  ],
  "confidence": 0.72,
  "metadata": {"model": "manteia-oss-7b@v15", "latency_ms": 840}
}
```

### 4.2 Catene di verifica
- **Self‑check**: il modello rilegge l’output e verifica copertura citazioni.
- **Attribution check**: nessuna frase “non citata” per claim fattuali.
- **Toxicity/PII**: filtro finale (policy + PII guard).

### 4.3 Serving modelli
- **vLLM / TGI / Ollama** per LLM OSS; **router** multi‑modello con quote e fallback.
- **Batching** e **KV‑cache** per ridurre latenza/costo; **streaming** dove UX lo richiede.

---

## 5) Eval

### 5.1 Offline
- **RAGAS**: faithfulness, answer relevancy, context recall/precision, answer similarity.
- **TruLens** / **Arize Phoenix**: tracing + metriche custom (tool‑use, hops, grounding rate). 
- **Golden set**: Q/A con citazioni attese; *LLM‑judge* con revisione umana su sample.

### 5.2 Online
- **A/B o canary**: confronto versione indice/modello/ranking.
- **SLO**: p95 latency, error rate, **grounding rate** ≥ soglia, tasso report “hallucination” < target.
- **Feedback** in‑app (thumbs‑up/down con motivo, “citation broken”).

**Script eval (scheletro)**
```python
from ragas import evaluate, faithfulness, answer_relevancy, context_precision, context_recall
report = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision, context_recall])
report.to_csv("artifacts/eval_ragas.csv", index=False)
```

---

## 6) Scalabilità & Performance

- **Sharding** dell’indice per tenant/namespace; **repliche** per throughput.
- **HNSW/IVF tuning**: `M`, `ef_construction`, `ef_search`, `lists` (profilare p95).
- **Reranker budget**: top‑k limitato (es. 50) + batch per GPU; early‑exit su soglia.
- **Pre‑computi**: chunk title/summary, keyword, map heading → boost ranking.
- **Cold start**: warm‑up cache (KV‑cache, top prompts), connessioni pool.
- **Async pipeline**: `asyncio`/workers; rate‑limit per upstream API.
- **CDN** per asset/documenti; **compressione** risposte (HTTP/gzip).

---

## 7) Observability & Operatività

**Metriche chiave**

- **Retrieval**: hit rate, coverage (n° sorgenti/distinte), `top_k` effettivo, `recall@k` stimata, overflow/truncation ratio.
- **Ranking**: score medio re‑ranker, % early‑exit, distribuzione qualità.
- **Synthesis**: p50/p95 latency, token in/out, cost per req, % schema‑invalid, % no‑citation.
- **Qualità**: grounding rate, faithfulness, feedback negativi, incidenti “hallucination”.
- **Infra**: CPU/GPU util, memoria, QPS, code backlog.

**Strumenti**

- **OpenTelemetry** (tracing end‑to‑end, span per retriever/reranker/synthesis).
- **Langfuse** (LLM observability), **Phoenix** (RAG analytics), **Prometheus/Grafana** per metriche, **Loki** per log. 
- **Alert**: SLO paginati (latency p95, error rate, no‑citation spike, 5xx retriever).

---

## 8) Sicurezza & Governance

- **Access control** su query e slicer: `tenant_id`, `access_level`, ABAC/RBAC; row‑level security (DB).
- **PII**: scrub in ingest, mascheramento nei log, retention minima.
- **Supply‑chain**: SBOM, firma immagini, scans CVE; segreti via vault, OIDC per deploy.
- **Prompt/data governance**: *prompt card* e *dataset card* versionate; ADR per scelte critiche.
- **Audit**: tutte le risposte con **citazioni persistite** e trace id per audit trail.

---

## 9) Cost & Efficienza

- **Caching** (embedding & completions), **batch** e **stream**; KV‑cache abilitato.
- **Router**: OSS primario, commerciale per fallback/queries complesse.
- **Top‑k adattivo** e *early‑exit*; token budget dinamico in base a confidenza.
- **Autoscaling** su QPS; spegni repliche in orari di basso traffico.

---

## 10) Multitenancy

- **Namespace per tenant** su vector DB, **chiavi di cifratura** dedicate.
- Isolamento dei metadati e quote per tenant (QPS, storage).
- Modelli/Prompt per tenant versionati e tracciati.

---

## 11) Stack di riferimento

- **Parsing**: unstructured, trafilatura, Tika, Tesseract/OCR, layoutparser  
- **PII**: Microsoft Presidio  
- **Embedding**: `bge-large`, `gte-large`, `e5-large` (o hosted equivalenti)  
- **Vector/Search**: Qdrant, Weaviate, pgvector, Elasticsearch/Vespa  
- **Retriever/Chains**: LangChain, LlamaIndex, Haystack  
- **Reranking**: `bge‑reranker`, Cohere Rerank, Jina Reranker  
- **Serving LLM**: vLLM, TGI, Ollama (OSS)  
- **Eval/Obs**: RAGAS, TruLens, Arize Phoenix, Langfuse, OpenTelemetry  
- **Orchestrazione**: Airflow/Prefect, Kafka (stream)  
- **CI/CD**: GitHub Actions + Docker Buildx, ArgoCD/Helm (K8s)

---

## 12) Esempi pratici (estratti)

**Config YAML — retriever ibrido**
```yaml
retriever:
  lexical:
    engine: elasticsearch
    k: 100
  dense:
    engine: qdrant
    top_k: 100
    ef_search: 100
  fusion:
    method: rrf
    final_k: 60
reranker:
  model: bge-reranker-large
  top_k: 10
  batch_size: 16
```

**Cache chiavi**
```text
emb:{sha256(text)[:16]} -> vector
ctx:{sha256(query+filters)[:16]} -> [chunk_id...]
ans:{sha256(template+inputs+model)[:16]} -> response JSON
```

**API risposta (schema min.)**
```json
{
  "answer": "…",
  "citations": [{"chunk_id":"…","uri":"…"}],
  "usage": {"prompt_tokens": 1200, "completion_tokens": 220},
  "trace_id": "rag-2025-09-24-abc123"
}
```

---

## 13) Check‑list DoD (RAG feature)

- [ ] Ingest idempotente, PII scrub, dedup attivo
- [ ] Index versionato (`docs_vN`) con rollback plan
- [ ] Retriever ibrido + re‑ranker con test e soglie
- [ ] Citazioni **obbligatorie** e schema JSON validato
- [ ] Eval **RAGAS** su golden set; grounding rate ≥ soglia
- [ ] Tracing OTel + dashboard (latency, cost, quality)
- [ ] Canary in produzione e rollback in 1 comando

---
Per una trattazione più dettagliata, consulta il [RAG project stack](../downloads/RAG_Project_Stack_Manteia.docx).