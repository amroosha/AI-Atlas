# Retrieval Augmented Generation

> **RAG** lets an LLM answer questions using knowledge it wasn't trained on
> (private docs, recent data, etc.) by fetching relevant context at query
> time instead of relying purely on the model's frozen parameters.


## The Core Idea

RAG is built around a simple loop:

```
Question → Indexing → Retrieval → Generation → Answer
```

## Visual Overview

This is the SOTA model of RAG according to freecodecamp Langchain engineer.
*I will build this step by step in a notebook enviroment while explaining each section*

![RAG from scratch](./assets/RAG_from_scratch.png)

## Pipeline Stages

The diagram above breaks RAG into 6 stages, in order. The first three shape
*what* we ask for, the last three decide *how* we get and use it.

1. ### Query Translation
   Rewrite or expand the user's raw question into a better search query.

2. ### Routing
   Decide which data source, index, or tool the query should be sent to.

3. ### Query Construction
   Turn the query into the exact structured form the target store expects
   (filters, SQL, metadata constraints, etc.).

4. ### Indexing
   Chunk, embed, and store documents ahead of time so they're searchable.

5. ### Retrieval
   Fetch the most relevant chunks for the query from the index.

6. ### Generation
   Feed the query + retrieved chunks to the LLM to produce a grounded answer.

---

**Lets start with the basics**

# Naive RAG

![NAIVE RAG](./assets/Naive_RAG.png)

A simple summary of the three core stages in a naive Retrieval-Augmented Generation (RAG) pipeline:
- **Indexing**
- **Retrieval**
- **Generation**

## Indexing

Indexing is split between two representations:

- **Statistical**: usually a Bag of Words, where each word has a frequency of occurrence. The representation is very sparse, and the most common search method is **BM25**.
- **Machine learned**: represents vector embeddings, which are dense vectors. Most search algorithms here rely on **ANN (Approximate Nearest Neighbor)** methods like **HNSW**.

Each document is split into multiple chunks, since embedding models have limited context windows.

## Retrieval

Once documents are indexed, retrieval finds the most relevant chunks for a given query:

- The **query** is transformed into the same representation used during indexing (sparse vector for BM25, dense embedding for vector search).
- A **similarity search** is performed (e.g., cosine similarity for embeddings, term-overlap scoring for BM25) to rank chunks by relevance.
- The **top-k** most relevant chunks are selected and passed forward as context.

Some pipelines combine both statistical and machine-learned retrieval (**hybrid search**) to balance precision and recall.

## Generation

The retrieved chunks are used to augment the input to a language model:

- The **retrieved context** is combined with the original query, typically inserted into a prompt template.
- This augmented prompt is passed to a **generative model (LLM)**, which produces the final answer grounded in the retrieved information.
- Because the model relies on retrieved context rather than only its parametric knowledge, this reduces hallucination and allows access to up-to-date or domain-specific information.

## Reasoning Models vs General LLMs in RAG

Grounding (reducing hallucination via context) and reasoning ability are **separate concerns**:

- **General LLMs** are usually sufficient for **naive RAG** (single retrieval pass → stuff context → generate), since the task is mainly extracting/synthesizing facts already present in the context.
- **Reasoning models** become more valuable in **advanced RAG** workflows, where the bottleneck shifts from "does the model have the facts" to "can the model correctly combine/use them":

| Scenario | Why reasoning helps |
|---|---|
| Multi-hop retrieval (combining facts across chunks) | Needs to chain information logically |
| Conflicting sources | Needs to weigh evidence and decide what to trust |
| Complex synthesis (comparison, summarization across docs) | Benefits from structured, step-by-step processing |
| Agentic/iterative RAG (query rewriting, re-retrieval, self-critique) | Requires planning and decision-making |

**Takeaway:** Match model capability to the complexity of reasoning required by the pipeline, don't default to reasoning models unless the workflow demands it.

## Context Quantity vs. Reliability

A common misconception: **more retrieved context = higher accuracy**. In practice, this often breaks down.

**Why more context can hurt:**

- **"Lost in the Middle" effect** — LLMs attend more strongly to the start/end of context; relevant info in the middle can be under-weighted or ignored.
- **Distraction from irrelevant chunks** — large top-k retrieval introduces noise, increasing the risk of picking up tangential or incorrect information.
- **Conflicting information** — more chunks increases the chance of contradictions (e.g., outdated vs. updated docs), which weaker models struggle to resolve.
- **Model-specific context handling** — context window size ≠ effective context utilization; some models degrade well before hitting their stated limit.

**What actually improves reliability:**

| Factor | Effect |
|---|---|
| Retrieval precision (relevant chunks only) | More reliable than raw recall (many chunks) |
| Reranking before feeding to LLM | Pushes relevant chunks into positions the model attends to |
| Chunk ordering | Place key chunks near the start/end of the prompt |
| Context compression/summarization | Reduces noise while preserving key facts |
| Right-sized top-k | Smaller, higher-quality k often beats large k |

**Takeaway:** Quality and placement of context matters more than quantity. Good RAG pipelines prioritize **retrieval precision + reranking + smart context assembly** over maximizing the number of chunks stuffed into the prompt.

This document outlines the modern dependency specifications, parameter configurations, and structural workflow for a production-ready **Naive RAG Pipeline** utilizing **Groq**, **Hugging Face BGE-1.5 Small**, **ChromaDB**, and **LangSmith**.

# Libraries and parameters explanation

## 1. Environment & Package Matrix

| Category | Component | Package Name / Dependency | Import Path | Primary Role |
| :--- | :--- | :--- | :--- | :--- |
| **Orchestration** | Core Abstractions | `langchain-core` | `langchain_core.prompts`, `langchain_core.runnables` | Standardized interfaces (LCEL), prompt templates, output parsers |
| **Vector Database** | Vector Store | `langchain-chroma` | `from langchain_chroma import Chroma` | High-performance vector indexing, similarity search, and persistence |
| **Embeddings** | Local Vectorizer | `langchain-huggingface` | `from langchain_huggingface import HuggingFaceEmbeddings` | Dense vector generation using local transformer architectures |
| **Generation LLM**| LPUs Inference | `langchain-groq` | `from langchain_groq import ChatGroq` | Ultra-fast cloud text generation using open-weights models |
| **Observability** | Tracing & Evaluation | `langsmith` | `from langsmith import Client` | Automated execution logging, latency breakdown, and debugging |
| **Underlying ML** | Sentence Engine | `sentence-transformers` | *(Internal dependency of HuggingFaceEmbeddings)* | PyTorch-based execution backend for embedding models |

---

## 2. Parameter Blueprint

| Library / Module | Class / Method | Parameter Name | Sample / Recommended Value | Description & Technical Impact |
| :--- | :--- | :--- | :--- | :--- |
| **Environment** | `os.environ` | `LANGCHAIN_TRACING_V2` | `"true"` | Enables automatic background span tracing to *LangSmith* |
| **Environment** | `os.environ` | `LANGCHAIN_API_KEY` | `"lsv2_pt_..."` | Authentication token for logging traces into *LangSmith* |
| **Environment** | `os.environ` | `GROQ_API_KEY` | `"gsk_..."` | Authentication key required for remote *Groq API* inference |
| **Embeddings** | `HuggingFaceEmbeddings` | `model_name` | `"BAAI/bge-small-en-v1.5"` | Hugging Face model repository string; outputs **384-dimensional** vectors |
| **Embeddings** | `HuggingFaceEmbeddings` | `model_kwargs` | `{'device': 'cpu'}` | Computation target device (*cpu* or *cuda*) |
| **Embeddings** | `HuggingFaceEmbeddings` | `encode_kwargs` | `{'normalize_embeddings': True}` | Normalizes vectors to **unit length** (enables exact *cosine similarity* calculations via dot product) |
| **Vector Database**| `Chroma.from_documents` | `persist_directory` | `"./chroma_db"` | Disk directory path; passing this prevents in-memory loss by forcing **SQLite + Parquet disk storage** |
| **Vector Database**| `vectorstore.as_retriever`| `search_kwargs` | `{"k": 3}` | Top-K constraint specifying how many nearest neighbor documents to retrieve |
| **Generation LLM**| `ChatGroq` | `temperature` | `0.0` | Controls output randomness; **0.0** enforces deterministic, factual generation |


## Technical Notes & Architectural Insights

* ***Package Isolation Strategy***: Never import partner integrations from `langchain_community` in modern codebases. Partner packages like `langchain-groq` and `langchain-chroma` provide *decoupled dependency chains*, reducing overall project installation size and preventing dependency version conflicts.
* ***Embedding Model Efficiency***: *BAAI/bge-small-en-v1.5* generates **384-dimensional dense vectors**, striking an ideal balance between low RAM footprint, ultra-fast local CPU inference, and state-of-the-art semantic representation quality.
* ***Chroma Persistence Architecture***: If `persist_directory` is omitted during instantiation, Chroma defaults to **DuckDB / RAM-only storage**, causing instant document loss upon Python process exit. Adding a directory path ensures permanent storage via **SQLite tables** and vector index files.
* ***LCEL Pipe Syntax Mechanism***: The `|` operator in *LangChain Expression Language (LCEL)* binds components into a unified execution graph using standard **Runnable protocols** (`invoke`, `stream`, `batch`).
* ***LangSmith Non-blocking Tracing***: Tracing enabled via `LANGCHAIN_TRACING_V2="true"` operates **asynchronously** in native background threads, ensuring that observability features do not add API latency to user-facing RAG runs.

* **Production-Grade Alternatives for WebBaseLoader**
   - Firecrawl (langchain-community / firecrawl-py): Converts full websites or pages directly into clean Markdown.
   - Spider (langchain-community): Ultra-fast crawler optimized for LLM indexing.
---

#