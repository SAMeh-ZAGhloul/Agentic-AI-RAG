# RAG with SQL Router

We are developing a system that will guide you in creating a custom agent. This agent can query either your Vector DB index for RAG-based retrieval or a separate SQL query engine. 

## 🔍 **The Critical Component: Response Validation**

**While everyone is trying to build agents, no one tells you how to ensure their outputs are reliable.**

**[Cleanlab Codex](https://help.cleanlab.ai/codex/)**, developed by researchers from MIT, offers a platform to evaluate and monitor any RAG or agentic app you're building. This system integrates Cleanlab Codex for automatic response validation, ensuring your AI outputs are trustworthy and continuously improving.

### **Why Cleanlab Codex is Essential:**

- **🔍 Automatic Detection**: Detects inaccurate/unhelpful responses from your AI automatically
- **📈 Continuous Improvement**: Allows Subject Matter Experts to directly improve responses without engineering intervention  
- **🎯 Trust Scoring**: Provides reliability metrics for every response
- **🔄 Real-time Validation**: Validates queries and responses in real-time
- **📊 Analytics**: Track improvement rates and response quality over time

### **How It Works in This System:**

1. **Query Processing**: Your queries are automatically validated by Cleanlab Codex
2. **Response Validation**: AI responses are scored for reliability and accuracy
3. **SME Intervention**: Subject Matter Experts can improve responses through the Codex interface
4. **Continuous Learning**: The system learns from validated responses for future queries

We use:

- [Llama_Index](https://docs.llamaindex.ai/en/stable/) for orchestration
- [Docling](https://docling-project.github.io/docling) for simplifying document processing
- [Qdrant](https://qdrant.tech/) to self-host a VectorDB
- [BAAI/bge-reranker-base](https://huggingface.co/BAAI/bge-reranker-base) for document reranking
- **[Cleanlab Codex](https://help.cleanlab.ai/codex/)** for **response validation and reliability assurance** ⭐
- [OpenRouterAI](https://openrouter.ai/docs/quick-start) to access Alibaba's Qwen model

> **💡 Key Insight**: While most tutorials focus on building agents, **[Cleanlab Codex](https://help.cleanlab.ai/codex/)** addresses the critical gap of ensuring those agents produce reliable, trustworthy outputs.

## Architecture Overview

The system is designed with a modular and extensible architecture, incorporating advanced components for data ingestion, query processing, and reasoning.

### 2) Data Ingestion & Indexing Pipelines (Docling-first)

*   **Sources**: PDFs, Office docs, HTML, emails, S3/MinIO buckets, Git repos.
*   **Acquire**: Watcher (Kafka connectors or lightweight cron) pulls new docs to MinIO.
*   **Parse**: **Docling** as primary converter (layout, tables, figures) → normalized JSON/Markdown.
*   **OCR**: **Tesseract** for scanned PDFs; image segmentation via **layoutparser** (optional).
*   **Chunking**: structure-aware (headings, tables). Adaptive chunk size by token length (e.g., 512-1024 tokens).
*   **Embed**: BGE/E5; store **vector** in Qdrant with rich metadata; persist original text blobs in MinIO/Postgres.
*   **Graph Build**: NER & triplet extraction (spaCy/REL/REBEL) → entities + relations inserted into JanusGraph/Neo4j.
*   **Quality/Eval**: RAGAS on sampled docs; store metrics; alerts via Grafana.
*   **Reindex Triggers**: doc updates, schema changes, model upgrades (versioned embeddings), or KG enrichment jobs.

### 3) Query & Reasoning Flow (Agentic Loop)

**User → “Ask”** (UI) → **FastAPI** → **LangGraph** orchestrates the agentic loop, which may invoke **LlamaIndex Workflows** (e.g., `RouterOutputAgentWorkflow`) to dynamically decide and execute tools:

*   **Tool Selection & Execution**: The LangGraph agent, potentially via a LlamaIndex `RouterOutputAgentWorkflow`, intelligently selects between specialized tools based on the user's query.
*   **Semantic Retrieval (Vector)**: Query → dense retrieval (Qdrant/Milvus) → rerank top-k → ground LLM. Responses are enhanced with **Cleanlab Codex** for validation and trust scoring.
*   **Relational Reasoning (Graph)**: Entity linking → Cypher/Gremlin query → multi-hop path retrieval → summaries.
*   **Structured Access (SQL)**: NL→SQL using **LlamaIndex's `NLSQLTableQueryEngine`** (schema-aware) with models like SQLCoder/DIN-SQL → execute on Postgres/DuckDB → resultframes.
*   **Synthesis**: Fuse vector passages + KG facts + SQL tables → response plan.
*   **Charts/Insights**: Generate a **Vega-Lite spec** (bar/line/scatter) + textual insights (trends, anomalies, YoY/DoD deltas).
*   **Safety**: Guard the prompt and output using **Cleanlab Codex** (for response validation), NeMo/Llama Guard, plus policy checks + source attributions.
*   **Memory**: Short-term conversation in Redis; long-term artifacts in Postgres/MinIO.

### 4) MCP Integration

*   **Client**: Embed an MCP client inside the agent runtime (LangGraph node).
*   **Servers (examples)**:
    *   **mcp-filesystem**: safe, sandboxed FS access (read curated corpora from MinIO mounts).
    *   **mcp-postgres**: parameterized SQL with schema introspection for NL→SQL.
    *   **mcp-http**: controlled web fetch (with allow-listed domains) for augmentation.
    *   **mcp-shell**: limited commands for ETL jobs (doc ingestion, reindex).
*   **Policy**: Access control (RBAC via Keycloak); per-tool quotas; audit logs via OpenTelemetry.

### 5) Natural Language → SQL

*   **Models**: **SQLCoder-7B/34B**, **DIN-SQL-Llama-3**; optionally fine-tune on your schemas & query logs. The **LlamaIndex `NLSQLTableQueryEngine`** provides a robust framework for this.
*   **Schema Grounding**: Auto-describe tables/columns; maintain a **catalog** (data dictionary) exposed to the model via context.
*   **Execution Safety**: Read-only roles; SQL sandbox; dry-run with EXPLAIN; auto-LIMIT; reject DDL/DML unless explicitly allowed.
*   **Post-processing**: Convert resultsets to dataframes; feed to chart generator.

### 6) Natural Language → Charts/Insights

*   **Spec Generation**: LLM produces **Vega-Lite** JSON from a validated dataframe schema.
*   **Insight Templates**: trend, seasonality, outlier, contribution, pareto, cohort, funnel.
*   **Validation**: Schema/type checks; safe-list encodings; preview before render.
*   **Export**: PNG/SVG/PDF via Headless Chromium in a worker; store in MinIO.

### 7) Retrieval Strategies (that work in practice)

*   **Hybrid**: Dense (vectors) + sparse (BM25) + **rerank** (bge-reranker).
*   **KG-augmented**: Use graph to expand/bridge multi-hop questions; attach KG facts in context window.
*   **Citations**: Passages are always cited with doc_id + page/section.

## Set Up

Follow these steps one by one:

### Setup Milvus VectorDB

Milvus provides an installation script to install it as a docker container.

To install Milvus in Docker, you can use the following command:

```bash
curl -sfL https://raw.githubusercontent.com/milvus-io/milvus/master/scripts/standalone_embed.sh -o standalone_embed.sh

bash standalone_embed.sh start
```

### Install Dependencies

```bash
uv sync
```

## Run the Notebook

You can run the `notebook.ipynb` file to test the functionality of the code in a Jupyter Notebook environment. This notebook will help you understand routing, tool calling, and validating responses.

## Run the Application

To run the Streamlit app, use the following command:

```bash
streamlit run app.py
```

Open your browser and navigate to `http://localhost:8501` to access the app.

## 📬 Stay Updated with Our Newsletter!

**Get a FREE Data Science eBook** 📖 with 150+ essential lessons in Data Science when you subscribe to our newsletter! Stay in the loop with the latest tutorials, insights, and exclusive resources. [Subscribe now!](https://join.dailydoseofds.com)

[![Daily Dose of Data Science Newsletter](https://github.com/patchy631/ai-engineering/blob/main/resources/join_ddods.png)](https://join.dailydoseofds.com)

## Contribution

Contributions are welcome! Feel free to fork this repository and submit pull requests with your improvements.
