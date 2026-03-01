# System Architecture

This document explains the **system architecture and key technical decisions** behind the ESG Document Intelligence solution.
The focus is on **why each component was chosen**, how they work together, and how the design supports **accuracy, explainability, and production readiness**.

---

## High-Level Architecture Overview

```text
┌──────────────────────────────────────────┐
│              ESG PDF(s)                  │
│ (Public sustainability / financial docs) │
└───────────────────────┬──────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────┐
│ Ingestion Pipeline (faiss_ingest.py)     │
│ - PDF page extraction (PyPDFLoader)      │
│ - Chunking (RecursiveCharacterSplitter)  │
│ - Embedding (OpenAI embeddings)          │
│ - Persist FAISS index + metadata         │
│   (source, page, doc_id)                 │
└───────────────────────┬──────────────────┘
                        ▼
┌──────────────────────────────────────────┐
│ Searchable Stores (Persisted on disk)    │
│ - FAISS Vector Index                     │
│ - Prompt Configs (YAML prompts/**)       │
└───────────────┬───────────────┬──────────┘
                │               │
                ▼               ▼
┌──────────────────────┐   ┌──────────────────────┐
│ Streamlit UI         │   │ FastAPI Service       │
│ (app.py)             │   │ (api.py)              │
│ - Multi-thread chat  │   │ - /qa?q=...           │
│ - Streaming tokens   │   │ - Typed response      │
│ - Retrieval display  │   │ - Cache headers       │
└───────────────┬──────┘   └──────────────  ┬──────┘
                │                           │
                └──────────────┬────────────┘
                               ▼
┌──────────────────────────────────────────┐
│ Core RAG Orchestrator (main.py)          │
│                                          │
│ 1. Agentic Router (LLM)                  │
│    - mode: rag | summarize | extract_kpi │
│    - retrieval: hybrid | vector | lexical│
│                                          │
│ 2. Hybrid Retrieval                      │
│    - FAISS vector search (MMR optional)  │
│    - BM25 lexical search                 │
│    - Weighted fusion                     │
│    - Optional LLM reranker (JSON output) │
│                                          │
│ 3. Context Builder                       │
│    - [Source | Page] citation tags       │
│    - Retrieval transparency payload      │
│                                          │
│ 4. Grounded Answer Generation            │
│    - Strict YAML RAG prompt              │
│    - Deterministic "Not found" fallback  │
└───────────────────────┬──────────────────┘
                        ▼
┌──────────────────────────────────────────┐
│ Output                                   │
│ - Answer with citations                  │
│ - Sources, pages, chunk previews         │
│ - Mode + retrieval strategy              │
└───────────────┬──────────────────────────┘
                ▼
┌──────────────────────────────────────────┐
│ Evaluation & Monitoring (evals.py)       │
│ - DeepEval (faithfulness, relevance)     │
│ - Langfuse traces + metrics              │
│ - Non-blocking async execution           │
└──────────────────────────────────────────┘
```

---

## Key Technical Decisions

### 1. LLM Selection

**Model Used:** `gpt-4o-mini`

**Reasoning:**

* Provides strong reasoning and instruction-following capabilities
* Lower latency and cost compared to larger GPT-4 variants
* Supports streaming, which improves UI responsiveness
* Reliable JSON output for routing and reranking prompts

**Usage in System:**

* Agentic routing (deciding query intent and retrieval strategy)
* Optional reranking of retrieved chunks
* Grounded answer generation using strict prompts

This choice balances **answer quality, latency, and cost**, which is important for production-like systems.

---

### 2. Vector Database

**Chosen Technology:** FAISS (local, disk-persisted)

**Reasoning:**

* Fast and lightweight for semantic similarity search
* No external service dependency (easy to run locally or in Docker)
* Deterministic and reproducible results
* Ideal for case studies and prototyping while still production-relevant

**How It’s Used:**

* Stores embeddings for document chunks
* Loaded once at runtime and reused
* Supports similarity search and optional MMR for diversity

---

### 3. Chunking Strategy

**Approach:** Recursive Character-based Chunking with Overlap

**Reasoning:**

* Preserves natural text boundaries (paragraphs → sentences → characters)
* Prevents important facts from being split across chunks
* Overlap ensures continuity for cross-paragraph information
* Works well for long, narrative ESG documents

**Why This Matters:**
Poor chunking leads to retrieval failures even with good embeddings.
This strategy improves **recall without sacrificing coherence**.

---

### 4. Framework Choice

**Choice:** Custom RAG pipeline (built on LangChain primitives)

**Reasoning:**

* Full control over retrieval logic and debug payloads
* Easier to enforce strict grounding and fallback behavior
* Simpler integration of hybrid retrieval (FAISS + BM25)
* Cleaner separation of ingestion, routing, retrieval, and generation

**Why Not Full LlamaIndex / LangChain Chains:**

* Abstractions can hide important retrieval decisions
* Harder to enforce deterministic outputs and transparency
* Custom logic is easier to test, debug, and explain in interviews

---

### 5. Agentic Design

**Agent Type:** Lightweight LLM-based Router

**Agent Responsibilities:**

* Classify query intent:

  * `rag` → direct answer with citations
  * `summarize` → high-level summary
  * `extract_kpi` → numeric / metric extraction
* Select optimal retrieval strategy:

  * `vector` → semantic queries
  * `lexical` → exact numbers / KPIs
  * `hybrid` → default balanced approach

**Workflow:**

1. User query enters system
2. Router decides mode + retrieval strategy
3. Retrieval and generation follow router decision

This avoids a **one-size-fits-all RAG pipeline** and improves answer quality.

---

## Grounded Answering & Explainability

* All answers are generated **only from retrieved context**
* Citations are embedded using:
  `[Source: <file> | Page <n>]`
* If any required information is missing, the system returns exactly:
  **“Not found in provided documents.”**

Each response includes a debug payload with:

* Sources and page numbers
* Chunk previews
* Retrieval strategy used
* Routing decision
* Timestamps

This ensures **auditability and user trust**, which is critical in ESG use cases.

---

## Evaluation & Monitoring

**Evaluation Frameworks Used:**

* **DeepEval**

  * Faithfulness
  * Relevance
  * Correctness
  * Completeness
    
* **Langfuse**
  * Trace each query/answer
  * Store metrics and metadata
  * Support longitudinal monitoring

**Design Choice:**

* Evaluation runs asynchronously
* Does not block UI or API responses
* Can be enabled or disabled via config

This mirrors **real production observability patterns**.

---

## Deployment & Production Considerations

* Dockerized deployment with:

  * Non-root user
  * Environment-based secrets
* Vectorstore and prompts can be mounted or baked into image
* FastAPI provides typed contracts for downstream systems
* Streamlit UI optimized for demos and human interaction
* Test coverage includes:

  * Smoke tests
  * Retrieval contract tests
  * Optional LLM-gated evaluation tests




