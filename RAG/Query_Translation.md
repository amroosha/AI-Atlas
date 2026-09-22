# ADVANCED RAG PIPELINES
## 1. Query Translation

> **Query Translation** is the first of the three pre-retrieval stages. It rewrites, expands, or decomposes the user's raw question into one or more retrieval-ready queries before any vector lookup happens.

A user asks a question the way they speak, but the retriever wants a string that overlaps with how the corpus is written.

Query translation is the cheapest high-leverage fix available: it costs one extra generation, touches no index, and improves every stage downstream.

![Query translation](./assets/Query_translation.png)

The diagram splits query translation into 2 main approaches:

| # | Approach | Core mechanism | Reaches the retriever as |
| :--- | :--- | :--- | :--- |
| 1 | **Query Decomposition** | Reshape the question into related questions, at a different or equal level of abstraction | Several distinct queries |
| 2 | **Pseudo-Documents (HyDE)** | Fabricate an answer passage, then search with that passage instead of the question | One expanded query |

Approach 1 keeps the user's intent and changes the query's shape. Approach 2 stops using the question as a retrieval string at all.

---

### 1.1 Query Decomposition

The diagram's vertical axis is the **abstraction level** of the derived query:

![Query Decomposition](./assets/Query%20transfrom.png)

| Direction | Technique | Named method | What the retriever receives |
| :--- | :--- | :--- | :--- |
| Up, more abstraction | Step-back question | Step-back prompting | A broader question about principles and background |
| Right, same level | Rewritten question | Multi-Query, RAG-Fusion | N paraphrases of the same question |
| Down, less abstraction | Sub-question | Least-to-Most | Several smaller, more specific questions |

Abstraction is the only knob these 3 branches turn. Going up hands the retriever principles. Going down hands it narrow facts. Staying level hands it vocabulary variation, so the same semantic point is probed from several phrasings instead of one.

---

#### 1.1.1 More Abstraction: Step-Back Prompting

***Mechanism:*** Prompt the LLM to replace the specific question with a **step-back question** one level up: the general principle, concept, or category that the specific question is an instance of.

```
Original   : "Exact steps should I take to cut my monthly electricity bill down by 30% in my small apartment?"

Step-back  : "What are the general principles of home energy conservation and power usage?"

```

***Why it works:*** The step-back question contains the *criteria* rather than the *instance*. Those criteria are usually stated explicitly in source text, while the instance-level conclusion often is not. Retrieval therefore lands on the passage that actually licenses the answer. The original question is then answered from the step-back context, so reasoning is grounded in the principle and applied to the specific case.

***LangChain API:***

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

step_back_prompt = ChatPromptTemplate.from_template(
    """You are an expert at world knowledge. Your task is to step back and
paraphrase a question to a more generic step-back question, which is easier
to answer. Here are a few examples:

# USER INPUT
Could the members of The Police perform lawful arrests?

# FINAL ANSWER
what can the members of The Police do?

# USER INPUT
Jan Sindel's was born in what country?

# FINAL ANSWER
what is Jan Sindel's personal history?

# USER INPUT
{question}

# FINAL ANSWER"""
)

step_back_chain = step_back_prompt | llm | StrOutputParser()
step_back_question = step_back_chain.invoke({"question": question})
```

***Cost:*** one extra LLM call. Retrieval runs twice, once on the step-back question and once on the original.

***When to use:*** reasoning-heavy questions where the document holds premises, not verdicts: physics, law, policy, troubleshooting.

***When not to:*** lookup questions whose answer is a literal string in the corpus. Stepping back from `"What is the capital of France?"` only blurs it.

---

#### 1.1.2 Less Abstraction: Least-to-Most Question Decomposition

***Mechanism:*** Two stages. **Decompose** the hard question into a list of easier sub-questions ordered from simplest to hardest. Then **solve sequentially**, appending each previous answer to the context of the next sub-question.

```
Original      : "What is the hometown of the 2024 national men's tennis champion?"

Decompose     : 1. Who won the 2024 national men's tennis championship?
                2. Where is that player from?
                (also valid: "which country is ... from?" and "which city ...?")

Solve         : q1 -> answer A1
                q1, A1, q2 -> answer A2
                q1, A1, q2, A2, q3 -> final answer
```

***Why it works:*** each sub-question is a *composition* of the previous answers plus one new hop. The final answer is derived from chained grounded facts instead of one LLM guess over a top-k blob. This is the standard fix for multi-hop questions such as `"What is the population of the city where X was founded?"`.

***LangChain API:***

```python
decomposition_prompt = ChatPromptTemplate.from_template(
    """You are a helpful assistant that generates multiple sub-questions
related to an input question. The goal is to break down the input into a set
of sub-problems / sub-questions that can be answered in isolation.

Generate multiple related sub-questions that are semantically distinct and
each answerable on its own.

Return the sub-questions as a Python list of strings and nothing else.

This is the question you need to decompose:
{question}"""
)

decomposition_chain = (
    decomposition_prompt | llm | StrOutputParser() | (lambda x: eval(x))
)
sub_questions = decomposition_chain.invoke({"question": question})
```

***Cost:*** one decomposition call, then N retrieval and N generation calls. The most expensive branch of the three.

***When to use:*** compositional / multi-hop queries, comparison queries, and anything with an implicit chain of facts behind it.

***When not to:*** single-hop factual lookups. Decomposition adds latency and invites hallucinated intermediate hops, which then poison the final answer.

---

#### 1.1.3 Same Abstraction: Multi-Query Rewriting

***Mechanism:*** Generate N semantically distinct rephrasings of the same question at the same abstraction level, retrieve for each, then union the results.

```
Original   : "What is Task Decomposition?"
Variants   : "How can complex tasks be broken down into smaller sub-tasks?"
             "What techniques exist for task decomposition in LLM agents?"
             "Explain the concept of splitting work into manageable steps."
```

The abstraction level is deliberately held constant. The variation is **lexical and perspective only**, so the technique acts as a hedge against vocabulary mismatch rather than a change in what is being asked.

***Why it works:*** a single embedding of one phrasing is one point in vector space, and it can land in a region that misses the relevant passage. N phrasings are N points, so recall goes up. One phrasing may match *"task decomposition"*, another may match *"breaking down complex tasks"*, and the second may be the only one present in the corpus.

***LangChain API:***

> **Import-path warning (verified against this project's venv).** In `langchain` 1.4.2 the top-level package contains only `agents`, `chat_models`, `embeddings`, `mcp`, `messages`, `rate_limiters`, and `tools`. There is **no** `langchain.retrievers` module. The legacy chains and retrievers now live in the **`langchain-classic`** package, which this project has indirectly through `langchain-community` (it declares `langchain-classic<2.0.0,>=1.0.7`). So the working import is `langchain_classic`, not `langchain`.

```python
from langchain_classic.retrievers.multi_query import MultiQueryRetriever
import logging

logging.basicConfig()
logging.getLogger("langchain_classic.retrievers.multi_query").setLevel(logging.INFO)

retriever_from_llm = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
    include_original=True,   # keep the user's own query in the union
)

unique_docs = retriever_from_llm.invoke(question)
```

`from_llm` accepts exactly these parameters: `retriever`, `llm`, `prompt`, `parser_key`, `include_original`. The constructed object exposes the fields `retriever`, `llm_chain`, `parser_key`, and `include_original`, which is why the constructor route below passes `llm_chain` rather than `llm`.

`DEFAULT_QUERY_PROMPT` asks for **3** alternative questions, one per line:

```
You are an AI language model assistant. Your task is
to generate 3 different versions of the given user
question to retrieve relevant documents from a vector database.
By generating multiple perspectives on the user question,
your goal is to help the user overcome some of the limitations
of distance-based similarity search. Provide these alternative
questions separated by newlines. Original question: {question}
```

To control the count, supply a custom prompt. Internally `from_llm` builds the chain as `prompt | llm | LineListOutputParser()`, so a plain newline-separated response is parsed into a list automatically. The `parser_key` parameter still exists in the signature but its own docstring marks it DEPRECATED and it is ignored, so do not rely on it.

```python
from langchain_core.output_parsers import BaseOutputParser
from langchain_core.prompts import PromptTemplate
from typing import List

class LineListOutputParser(BaseOutputParser[List[str]]):
    def parse(self, text: str) -> List[str]:
        return [q.strip() for q in text.strip().split("\n") if q.strip()]

    @property
    def _type(self) -> str:
        return "line_list_output_parser"

QUERY_PROMPT = PromptTemplate(
    input_variables=["question"],
    template="""You are an AI language model assistant. Your task is to generate
five different versions of the given user question to retrieve relevant
documents from a vector database. By generating multiple perspectives on the
user question, your goal is to help the user overcome some of the limitations
of distance-based similarity search.

Provide these alternative questions separated by newlines.
Original question: {question}""",
)

# Either customize the prompt on from_llm ...
retriever_a = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(), llm=llm, prompt=QUERY_PROMPT
)

# ... or build the chain yourself and pass it as llm_chain.
llm_chain = QUERY_PROMPT | llm | LineListOutputParser()
retriever_b = MultiQueryRetriever(retriever=vectorstore.as_retriever(), llm_chain=llm_chain)
```

With RAG-Fusion, retrieval is followed by **Reciprocal Rank Fusion**: each document gets a score of `1 / (rank + k)` per query it appears in, scores are summed across queries, and the fused ranking drives the final context. This is what turns N result lists into one ranked list instead of an arbitrary union.

***Cost:*** one LLM call to generate N queries, then N retrieval calls. Cheaper than least-to-most, because generation still happens once.

***When to use:*** short keyword-ish queries, corpus with inconsistent terminology, and the common case where the retriever returns plausible but not-quite-right chunks. In the LangChain docs' own example, this surfaces the *"Plan-and-Solve"* and *"HuggingGPT"* passages that the raw query missed.

***When not to:*** when the question is unambiguous and the corpus vocabulary matches the user's. The extra queries then add latency and can pull in topical-but-irrelevant chunks that dilute the context window.

---

### 1.2 Pseudo-Documents: Hypothetical Document Embeddings (HyDE)

***The problem it attacks:*** dense retrieval has a structural asymmetry. Queries are short, documents are long, and they are drawn from different distributions. Even a perfect retriever comparing an embedding of `"What is Task Decomposition?"` against an embedding of a 300-word passage is comparing two unlike objects. HyDE removes the query from the comparison entirely.

***Mechanism:*** Four steps.

1. The user's question is given to an instruction-following LLM.
2. The LLM is asked to **write a passage that answers the question**. It is not asked for the answer, and correctness is not required.
3. That fabricated passage, the *hypothetical document* or *pseudo-document*, is embedded instead of the question.
4. The retrieval system searches the real corpus with that passage vector.

Two nuances make HyDE work rather than backfire:

- **Documents are indexed normally.** The hypothetical document is used only at query time. `embed_documents` is left untouched and delegates to the real embedding model, while only `embed_query` triggers generation. Verified in the source: `embed_documents` calls `self.base_embeddings.embed_documents(texts)`, and `embed_query` generates then embeds.
- **Hallucination is tolerable and often helpful.** A fabricated passage that is factually wrong but stylistically and topically right still occupies the correct region of vector space. What is being matched is the *shape of the answer*, not its truth. This is why HyDE is described as zero-shot: it needs no relevance labels and no training data.

```
Question      : "What is Task Decomposition?"

Generated     : "Task decomposition is the process of breaking a complex
pseudo-doc      task into smaller, more manageable sub-tasks that can be
                solved independently or in sequence. In LLM agent design,
                decomposition improves reliability by..."

Embedded      : embed(hypothetical passage)   <- not embed(question)
Retrieved     : chunks lexically and stylistically closer to the passage
```

***Key property:*** HyDE is **not** query expansion. It does not add synonyms to the query string. It replaces the query with a document-shaped object and changes the query vector's location in the embedding space.

***LangChain API:***

`HypotheticalDocumentEmbedder` lives in `langchain-classic` and subclasses both `Chain` and `Embeddings`, so the resulting object can be passed anywhere an embeddings instance is accepted.

```python
from langchain_classic.chains.hyde.base import HypotheticalDocumentEmbedder

# base_embeddings is your real, document-side embedding model
# (in this project: HuggingFaceEmbeddings with BAAI/bge-small-en-v1.5)
hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=llm,
    base_embeddings=base_embeddings,
    prompt_key="web_search",
)

# Now the hypothetical document embedding is available as a plain vector
hypothetical_vector = hyde_embeddings.embed_query(question)
```

`from_llm` accepts `llm`, `base_embeddings`, `prompt_key`, `custom_prompt`, and extra keyword arguments.

***The available prompt keys*** are fixed by `PROMPT_MAP`, and `prompt_key` is **required** unless you pass `custom_prompt`. Omitting both raises `ValueError` listing the valid keys:

| `prompt_key` | Intended corpus style |
| :--- | :--- |
| `web_search` | General web passages |
| `sci_fact` | Scientific claims |
| `arguana` | Argumentative text |
| `trec_covid` | Biomedical / COVID literature |
| `fiqa` | Financial question answering |
| `dbpedia_entity` | Entity descriptions |
| `trec_news` | News |
| `mr_tydi` | Multilingual |

The `web_search` template is the shortest and the usual default:

```
Please write a passage to answer the question
Question: {QUESTION}
Passage:
```

Note the input variable is `{QUESTION}`, in capitals, unlike the `{question}` used by most other LangChain prompt templates. A custom prompt must target whatever key the chain expects.

***Custom prompt example:***

```python
from langchain_core.prompts import PromptTemplate

hyde_prompt = PromptTemplate.from_template(
    """Please write a passage to answer the question.
The passage should read like an encyclopedia entry.

Question: {QUESTION}
Passage:"""
)

hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=llm,
    base_embeddings=base_embeddings,
    custom_prompt=hyde_prompt,
)
```

***Cost:*** exactly one LLM call at query time. Retrieval cost is unchanged, because only one query vector is produced. This is the cheapest of the two approach families in terms of retrieval operations, and the most expensive in terms of a single generation.

***Combining embeddings:*** when more than one hypothetical document is generated, they are combined with mean-pooling. In the installed source, `combine_embeddings` is a method that returns `np.array(embeddings).mean(axis=0)`, with a pure-Python average as fallback when NumPy is absent. It is a method rather than a constructor field, and the class sets `extra="forbid"`, so to change the combination strategy you subclass instead of passing a parameter.

***When to use:*** asymmetric search, meaning short keyword-ish queries against long documents. Also useful when the corpus is written in a register very different from how users ask questions, such as legal, medical, or API documentation.

***When not to:*** when the answer is a literal string that must be matched exactly (IDs, error codes, names), because the pseudo-document pulls the retriever toward semantically similar prose instead of the exact token. Also when generation cost or latency at query time is not affordable, since every single query now pays for one generation.

---

### 1.3 Choosing Between the Two Approaches

| Dimension | Query Decomposition | Pseudo-Documents (HyDE) |
| :--- | :--- | :--- |
| Query vectors sent to retriever | N, one per derived query | 1 |
| Direction of change | Abstraction level (up or down) or vocabulary (level) | Modality: question becomes a passage |
| Failure mode it fixes | Multi-hop reasoning, vocabulary mismatch, ambiguity | Query/document asymmetry |
| Extra LLM calls | 1 for rewriting, plus N for solving sub-questions | 1 at query time |
| Extra retrieval calls | N | None |
| Main risk | Latency; hallucinated intermediate hops poisoning the chain | Exact-match drift; generation cost on every query |
| Index changes required | None | None |
| Primary LangChain entry point | `langchain_classic.retrievers.multi_query.MultiQueryRetriever` | `langchain_classic.chains.hyde.base.HypotheticalDocumentEmbedder` |

The two are **complementary, not competing**. A common production arrangement is to decompose first and then expand each decomposed query with HyDE, so every sub-question is retrieved as a passage rather than a question. The costs compound accordingly, which is the argument for enabling decompositions only when a router or a classifier has decided the query actually needs them.

Both approaches occupy the same architectural slot: before the retriever, with no writes to the vector store, and reversible by configuration alone. That is why query translation is usually the first optimization to attempt on an underperforming RAG pipeline.

***Version note for this project.*** All import paths above were executed against this repository's virtual environment (`langchain` 1.4.2, `langchain-community` 0.4.2, `langchain-classic` >= 1.0.7 pulled in transitively). The canonical vendor documentation still shows `from langchain.retrievers.multi_query import MultiQueryRetriever` and `from langchain.chains import HypotheticalDocumentEmbedder`, which are the pre-1.0 paths and will raise `ModuleNotFoundError` here. Use `langchain_classic`.


