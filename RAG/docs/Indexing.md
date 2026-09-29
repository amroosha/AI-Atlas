# ADVANCED RAG PIPELINES
## 4. Indexing

> Offline preparation of documents for retrieval: split them into units, decide what each unit represents, and optionally build abstractions above them.

**Scope:** document → index (chunking, representations, tree structures, multi-vector scoring). Not query-time retrieval, ranking, or fusion.

---

### 4.1 Chunking

**Why it exists**

- Embedding models and prompts accept bounded input; documents do not fit.
- One vector per document averages its topics into a blur.
- Chunk boundaries decide what a single vector can answer.

**Core mechanism**

1. Pick the unit: characters, tokens, sentences, or document structure.
2. Split until every chunk fits the budget (embedding window, prompt share).
3. Optionally overlap boundary text, so a fact cut in half stays retrievable.
4. Attach metadata (header path, source, position) for later filters and routing.

**Variants**

| Variant | Mechanism | Cost | Best for |
|---|---|---|---|
| Recursive / fixed | Separator hierarchy (`"\n\n"` → `"\n"` → `" "` → `""`) with `chunk_size` + `chunk_overlap` | Free, local, deterministic | Default for unstructured prose |
| Token-based | Split on the model tokenizer (`tiktoken`) | Free, local | Matching an embedding or prompt token budget |
| Structure-aware | Split on headers; header path becomes metadata | Free, local | Docs sites, wikis, code, notebooks |
| Semantic | Embed each sentence ±1 neighbour, cut where cosine distance exceeds the 95th percentile | 1 embedding call per sentence | Topic shifts with no structural markers |
| Agentic | LLM returns the indices where the topic shifts; text is sliced verbatim | 1 LLM call per passage window | High-value long-form or narrative corpora |

**Minimal example**

```python
# rule based: size and overlap are the only knobs
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)

# semantic: embed sentences one neighbour deep, cut above the 95th percentile of gaps
vectors = embed([neighbourhood(sentence) for sentence in sentences])
threshold = np.percentile(1 - (vectors[:-1] * vectors[1:]).sum(axis=1), 95)

# agentic: the LLM only proposes boundaries, the source text is sliced as-is
starts = llm.with_structured_output(Boundaries).invoke(numbered_sentences).start_indices
```

LangChain's `SemanticChunker` is the reference implementation of the percentile variant, defaults `breakpoint_threshold_type="percentile"` and amount `95`; it lives in the sunset `langchain-experimental` package and is ~20 lines to re-implement.

**Choosing**

```
Unstructured text, no constraints   → recursive split with 10–20% overlap
Token budget is the constraint      → tiktoken-based recursive split
Structure exists (headers, HTML)    → structure-aware split, keep the header path
Topic shifts mid-paragraph          → semantic split (embedding cost, variable sizes)
Boundary quality justifies LLM cost → agentic split
```

**Failure modes**

- Answer split across two chunks → overlap too small → raise overlap, or return neighbouring chunks (parent-document retrieval).
- Retrieved chunk carries irrelevant text → chunk too large for the query → shrink chunk size, expect more chunks retrieved.
- One chunk hides two topics → fixed sizing ignored structure → structure-aware or semantic splitting.
- Many degenerate one-sentence chunks → semantic threshold too low → raise the percentile, or merge below a minimum size.
- Semantic chunk overflows the embedding window → distance-based cuts have no size ceiling → add a max-size guard.
- Tool call rejected (`tool_use_failed` on Groq `gpt-oss`) → too many sentences in one call → prompt iteratively over small windows.

**Uncertainty**

- No universal chunk size or overlap; both depend on corpus and model `(unverified)`.
- Chroma's token-level evaluation (10 chunks retrieved, `text-embedding-3-large`): ClusterSemantic 96.2 recall, TokenText-400 95.1, Recursive-400 94.5 `(Smith & Troynikov, 2024)`.
- Overlap is not a free win: Recursive-400 scored 89.9 without overlap against 88.3 with 200-token overlap at 5 chunks, and 61.4 against 73.3 at minimum retrieval.

---

### 4.2 Multi-representation Indexing

> Index a searchable representation of a document, but return the full document it came from.

![Multi-representation Indexing](assets/multi-representation_indexing.png)

**Why it exists**

- Whole-document embeddings dilute every topic into one vector.
- Small chunks lose the surrounding context the answer needs.
- A chunk's wording rarely matches the user's question, while the answer inside it does.

**Core mechanism**

1. Derive one or more representations per document (summary, generated questions).
2. Embed the representations under a stable `doc_id`.
3. Store the full originals in a document store under the same `doc_id`.
4. At query time, search the representations, then fetch the originals by id.

**Variants**

| Variant | Indexed unit | Returned unit | Index-time cost | Best for |
|---|---|---|---|---|
| Summary index | LLM summary per document | Full document | 1 LLM call per document | Long documents, small corpora |
| Hypothetical questions | Questions the document answers | Full document | k LLM calls per document | Query-phrasing mismatch |
| Parent-document | Child chunk embeddings | Parent chunk or section | None | Documents where boundary context matters |

**Minimal example**

```python
summaries = [summarize(doc.page_content) for doc in docs]
doc_ids = [str(uuid4()) for _ in docs]

vectorstore.add_documents(                       # search the summaries
    [Document(page_content=s, metadata={id_key: i}) for s, i in zip(summaries, doc_ids)],
    ids=doc_ids,
)
store.mset(list(zip(doc_ids, docs)))             # keep the full documents

retriever = MultiVectorRetriever(vectorstore=vectorstore, docstore=store, id_key=id_key)
```

`MultiVectorRetriever` searches the vector store and returns `docstore.mget(ids)`, so the derived representations and the originals behave as one index.

**Choosing**

- Documents exceed the embedding window → summary index.
- Users phrase questions unlike the corpus → hypothetical questions.
- Chunks are the right unit but cuts lose context → parent-document retriever.

**Failure modes**

- Document never retrieved → the summary omits the answering detail → index raw chunks alongside the summaries.
- Token bill explodes → whole documents returned per hit → rerank, cap how many are returned, or return the parent chunk.
- Orphans after edits → vector store and docstore drift apart → write and delete both stores under one key.
- Retriever returns nothing after a restart → the docstore was process-local (`InMemoryStore`) → persist it (Redis, SQLite).

**Uncertainty**

- No public benchmark isolates the summary variant against plain chunking; gains are task-dependent `(unverified)`.

---

### 4.3 RAPTOR

> **R**ecursive **A**bstractive **P**rocessing for **T**ree-**O**rganized **R**etrieval: embed, cluster and summarize chunks progressively upward, then retrieve from the resulting tree `(Sarthi et al., 2024)`.

![RAPTOR tree](assets/Raptor.png)

**Why it exists**

- Chunk-only retrieval answers "what does this passage say", not "what is this corpus about".
- Multi-step questions need evidence from many chunks; a top-k of fragments shares no level of abstraction.
- Long or multi-document corpora exceed what any single chunk represents.

**Core mechanism**

1. Split the corpus into small chunks (~100 tokens in the paper) and embed them.
2. Reduce dimensions (UMAP to 10) and soft-cluster with a Gaussian mixture, so a chunk may join several clusters.
3. Summarize every cluster with an LLM; the summary becomes a node one level up, linked to its members.
4. Re-embed the summaries and repeat until a single root remains.
5. Index every node in one pool (collapsed tree), or traverse level by level (tree traversal).

**Variants**

| Variant | Mechanism | Cost | Best for |
|---|---|---|---|
| Collapsed tree | All levels in one index, one similarity search | 1 search over a larger index | Default; lets the query pick its own abstraction level |
| Tree traversal | Search the root, descend into the best children | Several searches plus traversal logic | Corpora where coarse-to-fine pruning beats one large index |

**Minimal example**

```python
for level in range(1, max_levels + 1):
    probabilities = GaussianMixture(n_components=k, random_state=0).fit(vectors).predict_proba(vectors)
    summaries = [
        summarize("\n\n".join(texts[i] for i in members(probabilities, cluster))[:MAX_CLUSTER_CHARS])
        for cluster in range(k)
    ]
    nodes = [Document(page_content=summary, metadata={"level": level}) for summary in summaries]

vectorstore.add_documents(every_node_from_every_level)   # collapsed tree
```

Cluster text is truncated to a bounded budget before summarization; the LLM never receives a whole document.

**Choosing**

- Queries need synthesis across many chunks → RAPTOR.
- The answer sits in one short span → plain chunk retrieval is cheaper and sharper.
- Summarization budget is tight → keep the tree shallow; most of the gain comes from the first level or two `(unverified)`.

**Failure modes**

- A cited claim is wrong → a summary hallucinated it, and the tree now indexes it → constrain the prompt to extraction and inspect summaries before indexing (paper's Appendix E).
- Higher levels lose detail → cluster text exceeded the summarization window and was truncated → cap cluster size, split oversized clusters.
- Build cost explodes → roughly `nodes / cluster_size` LLM calls per level → limit depth and clusters, cache summaries.
- Retrieval returns only broad summaries → high-level nodes dominate the pool → filter by level, or weight leaf chunks up.

**Uncertainty**

- The context window is not the constraint: RAPTOR never summarizes a whole document, only clusters of small chunks, so each call sees a few thousand tokens. The real cost is one summarization call per cluster per level.
- RAPTOR targets holistic, multi-step questions over long or multi-document corpora, not "the answer is one sentence in one document" `(Sarthi et al., 2024)`.
- UMAP plus a Gaussian mixture was chosen after an ablation (paper's Appendix B); other corpora may cluster less cleanly `(unverified)`.
- The official `raptor` repository is research code with no maintained release; production use is a hand-rolled tree.

---

### 4.4 Late-Interaction Indexing (ColBERT)

> ColBERT scores a query by matching every query token against every document token (MaxSim) instead of comparing two pooled vectors `(Khattab & Zaharia, 2020)`.

**Why it exists**

- One vector per chunk discards the token-level evidence that separates near-identical passages.
- Cross-encoders score pairs accurately but cannot be pre-computed.

**Core mechanism**

1. Encode query and document separately, keeping one vector per token.
2. Score with MaxSim: each query token keeps its best document token, the results are summed.
3. Pre-compute document vectors offline, so online work is token-level dot products.

| Engine | Mechanism | Space vs single vector | Latency |
|---|---|---|---|
| ColBERT | Per-token vectors, MaxSim | Order of magnitude larger | ~2 orders of magnitude faster per query than BERT rankers |
| ColBERTv2 | Residual compression of token vectors | 6–10× smaller than ColBERT | Comparable quality, much smaller index |
| PLAID | Centroid interaction and pruning over ColBERTv2 | Same as ColBERTv2 | Tens of ms on GPU, tens to hundreds on CPU at 140M passages |

Index size scales with chunk length (`tokens × dims × bytes`), so cost grows with both corpus and chunk size.

**Choosing**

- Rerank a few hundred dense/BM25 candidates → strongest precision gain per unit of effort.
- First-stage retrieval → only with a multi-vector-capable store (Vespa, Qdrant, Weaviate, LanceDB) and a latency budget.
- Small corpus or no GPU → a cross-encoder reranker is the cheaper return.
- Not implemented in the notebook: it needs a PLAID-style index and multi-vector storage this stack does not provide.

**Uncertainty**

- Extra dependencies (`colbert-ai`/PLAID, `Ragatouille`) and fast-moving trainer support; pin versions.

---

## References

- [RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval](https://arxiv.org/abs/2401.18059) — tree construction, clustering, retrieval modes, hallucination analysis.
- [LumberChunker: Long-Form Narrative Document Segmentation](https://arxiv.org/abs/2406.17526) — LLM-proposed chunk boundaries, +7.37% DCG@20 over the best baseline.
- [Evaluating Chunking Strategies for Retrieval](https://research.trychroma.com/evaluating-chunking) — token-level chunking evaluation and the recall figures above.
- [langchain_experimental/text_splitter.py](https://github.com/langchain-ai/langchain-experimental/blob/a6c66481ee56b38166b3daeea7eb767eac92146b/libs/experimental/langchain_experimental/text_splitter.py) — `SemanticChunker` algorithm and defaults.
- [langchain-experimental on PyPI](https://pypi.org/project/langchain-experimental/) — sunset notice for the package holding `SemanticChunker`.
- [ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT](https://arxiv.org/abs/2004.12832) — late interaction and MaxSim.
- [ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction](https://arxiv.org/abs/2112.01488) — residual compression and space footprint.
- [PLAID: An Efficient Engine for Late Interaction Retrieval](https://arxiv.org/abs/2205.09707) — late-interaction latency figures.
- [LangChain RAG From Scratch](https://github.com/langchain-ai/rag-from-scratch) — multi-representation indexing pattern implemented in the notebook.

