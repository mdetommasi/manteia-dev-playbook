# Blueprint — Agentic RAG + ReAct

```mermaid
flowchart LR
  U[User] --> GW(API Gateway)
  GW --> AG[Coordinator Agent (ReAct)]
  AG -->|Retrieve| RET[Retriever]
  RET --> VS[(Vector Store)]
  AG -->|Tools| T1[Search Tool] & T2[Code Runner] & T3[MCP Tools]
  AG --> LLM[LLM Runtime]
  LLM --> OBS[Telemetry: traces/metrics/logs]
  AG --> SVC[Domain Services]
  SVC --> DB[(DB)]
```

**Punti chiave**
- Coordinator Agent con **ReAct** per plan/act/observe.
- Retrieval ibrido (BM25 + dense) + re-ranking.
- Tooling via **MCP** per discovery/handshake standard dei tool.
- Osservabilità end-to-end (OpenTelemetry): latenza, token, tool error rate.
- Policy di **stop-condition** e fallback (router su modelli diversi).
