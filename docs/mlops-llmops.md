# MLOps / LLMOps per RAG, Agent e Modelli

- **Data Versioning**: DVC/LakeFS; dataset card, lineage e checksum.
- **Model Registry**: MLflow o equivalente, con stage (Staging/Production) e metriche.
- **Serving LLM OSS**: vLLM/TGI, Ollama per dev locale; throttling e quota per tenant.
- **Vector Store**: Weaviate, Qdrant, PgVector; schemi, TTL, politica di re-embedding.
- **Prompt Management**: versioning, A/B Prompt, canary.
- **Agent Eval**: harness di scenari, tool-ablation, replay di conversazioni reali.
- **Guardrail**: PII scrubber, output schema, content policy, rate-limit adattivo.
