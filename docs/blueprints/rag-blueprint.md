# Blueprint — Retrieval-Augmented Generation (RAG)

- **Ingest**: pipeline chunking+embedding; qualità (dedup, PII scrub)
- **Index**: schema con metadati (source, timestamp, access level)
- **Query**: retriever ibrido + re‑ranker; caching
- **Synthesis**: template controllati, citazioni obbligatorie
- **Eval**: faithfulness, grounding rate, answer similarity, latency
