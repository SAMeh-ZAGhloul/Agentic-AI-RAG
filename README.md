# Agentic AI RAG (Open‑Source) – Architecture & System Diagram

**Goal:** A trustworthy, production‑grade Agentic RAG that unifies

1. Vector DB embeddings,
2. Graph DB embeddings,
3. Natural Language → SQL, and
4. Natural Language → Charts/Insights — (MCP and Docling.

---

## High‑Level System Diagram

```mermaid
flowchart TB
  A[UI] --> C[API]
  C --> LLMNode[LLM]
  C --> M2[Embeddings]
  C --> M3[Reranker]
  C --> T1[ETL]
  C --> T2[OCR]
  C --> T3[SQL]
  C --> T4[Chart]
  C --> T5[MCP]
  C --> V[Vector DB]
  C --> G[Graph DB]
  C --> S[Postgres]
  C --> O[Object Store]
  C --> R[Redis]
  T1 --> O
  C --> F[Ops]
  F --> X[Grafana]
  F --> Y[OpenSearch]
  D[LangGraph] --> LLMNode
  D --> M2
  D --> M3
  D --> T1
  D --> T2
  D --> T3
  D --> T4
  D --> T5
  D --> V
  D --> G
  D --> S
  D --> R
  D --> K[Bus]
```

---

## 1) Core Open‑Source Stack

* **LLMs (served via vLLM or TGI)** : Llama 3.1, Mistral 7B/8x7B, Qwen2.5
* **Embeddings** : BGE (FlagEmbedding), E5, Jina embeddings;  **Reranker** : bge‑reranker‑v2.
* **Vector DB** : **Qdrant** (simple ops) or **Milvus** (scale) or **Weaviate** (rich filters).
* **Graph DB** : **JanusGraph (Apache 2.0)** on Cassandra/Scylla/Elastic; or  **Neo4j Community (GPLv3)** ; or  **ArangoDB Community (Apache 2.0)** .
* **SQL Engine** : **PostgreSQL** (primary) + optional **DuckDB** for local/adhoc analytics.
* **ETL/Parsing** : **Docling** (PDF/Office to structured),  **Apache Tika** ,  **trafilatura** ,  **pdfminer.six** ,  **Tesseract OCR** .
* **Agent Orchestration** : **LangGraph** (deterministic agent graphs), **Haystack Agents** or **LlamaIndex Agents** (optional).
* **MCP (Model Context Protocol)** : Open‑source MCP servers for  **filesystem** ,  **Postgres** ,  **HTTP** ,  **shell** ; MCP client embedded in the agent runtime.
* **Charts/Insights** : **Vega‑Lite** (spec‑driven),  **Plotly** , or **ECharts** (web).
* **API** :  **FastAPI** ; **Auth** via Keycloak/Ory; **Rate‑limit** via Traefik/Envoy.
* **Caching** : **Redis** (prompt/context cache).
* **Storage** : **MinIO** (object store for docs, parquet, model shards).
* **Messaging** : **Kafka** or **RabbitMQ** for async pipelines.
* **Safety/Guardrails** : **NeMo Guardrails** or  **Llama Guard** , plus allow‑lists/deny‑lists.
* **Eval & QA** :  **RAGAS** ,  **DeepEval** ,  **LangSmith‑OSS alt** .
* **Observability** :  **OpenTelemetry** ,  **Prometheus** ,  **Grafana** , **OpenSearch/Kibana** for logs.
* **Deployment** : Docker/Compose

---

## 2) Data Ingestion & Indexing Pipelines (Docling‑first)

 **Sources** : PDFs, Office docs, HTML, emails, S3/MinIO buckets, Git repos.

1. **Acquire** → Watcher (Kafka connectors or lightweight cron) pulls new docs to MinIO.
2. **Parse** → **Docling** as primary converter (layout, tables, figures) → normalized JSON/Markdown.
3. **OCR** → **Tesseract** for scanned PDFs; image segmentation via **layoutparser** (optional).
4. **Chunking** → structure‑aware (headings, tables). Adaptive chunk size by token length (e.g., 512‑1024 tokens).
5. **Embed** → BGE/E5; store **vector** in Qdrant with rich metadata; persist original text blobs in MinIO/Postgres.
6. **Graph Build** → NER & triplet extraction (spaCy/REL/REBEL) → entities + relations inserted into JanusGraph/Neo4j.
7. **Quality/Eval** → RAGAS on sampled docs; store metrics; alerts via Grafana.

 **Reindex Triggers** : doc updates, schema changes, model upgrades (versioned embeddings), or KG enrichment jobs.

---

## 3) Query & Reasoning Flow (Agentic Loop)

**User → “Ask”** (UI) → **FastAPI** → **LangGraph** policy decides tools:

* **Semantic Retrieval (Vector)** : Query → dense retrieval (Qdrant) → rerank top‑k → ground LLM.
* **Relational Reasoning (Graph)** : Entity linking → Cypher/Gremlin query → multi‑hop path retrieval → summaries.
* **Structured Access (SQL)** : NL→SQL with **SQLCoder/DIN‑SQL** (schema‑aware) → execute on Postgres/DuckDB → resultframes.
* **Synthesis** : Fuse vector passages + KG facts + SQL tables → response plan.
* **Charts/Insights** : Generate a **Vega‑Lite spec** (bar/line/scatter) + textual insights (trends, anomalies, YoY/DoD deltas).
* **Safety** : Guard the prompt and output (NeMo/Llama Guard + policy checks + source attributions).
* **Memory** : Short‑term conversation in Redis; long‑term artifacts in Postgres/MinIO.

---

## 4) MCP Integration

* **Client** : Embed an MCP client inside the agent runtime (LangGraph node).
* **Servers (examples)** :
* **mcp-filesystem** : safe, sandboxed FS access (read curated corpora from MinIO mounts).
* **mcp-postgres** : parameterized SQL with schema introspection for NL→SQL.
* **mcp-http** : controlled web fetch (with allow‑listed domains) for augmentation.
* **mcp-shell** : limited commands for ETL jobs (doc ingestion, reindex).
* **Policy** : Access control (RBAC via Keycloak); per‑tool quotas; audit logs via OpenTelemetry.

---

## 5) Natural Language → SQL

* **Models** :  **SQLCoder‑7B/34B** ,  **DIN‑SQL‑Llama‑3** ; optionally fine‑tune on your schemas & query logs.
* **Schema Grounding** : Auto‑describe tables/columns; maintain a **catalog** (data dictionary) exposed to the model via context.
* **Execution Safety** : Read‑only roles; SQL sandbox; dry‑run with EXPLAIN; auto‑LIMIT; reject DDL/DML unless explicitly allowed.
* **Post‑processing** : Convert resultsets to dataframes; feed to chart generator.

---

## 6) Natural Language → Charts/Insights

* **Spec Generation** : LLM produces **Vega‑Lite** JSON from a validated dataframe schema.
* **Insight Templates** : trend, seasonality, outlier, contribution, pareto, cohort, funnel.
* **Validation** : Schema/type checks; safe‑list encodings; preview before render.
* **Export** : PNG/SVG/PDF via Headless Chromium in a worker; store in MinIO.

---

## 7) Retrieval Strategies (that work in practice)

* **Hybrid** : Dense (vectors) + sparse (BM25) + **rerank** (bge‑reranker).
* **KG‑augmented** : Use graph to expand/bridge multi‑hop questions; attach KG facts in context window.
* **Citations** : Passages are always cited with doc_id + page/section.

---

---
