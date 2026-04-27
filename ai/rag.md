# RAG — Retrieval Augmented Generation

**One line:** Give an LLM access to your own documents at query time — it retrieves relevant chunks, then reasons over them to produce a grounded, cited answer.

---

## The Problem It Solves

LLMs are frozen at their training cutoff and have no access to private data. They hallucinate when asked about things they don't know.

```
Without RAG:
  User: "What did my notes say about KAN vs MLP?"
  LLM:  "I don't have access to your notes." (or worse — makes something up)

With RAG:
  User: "What did my notes say about KAN vs MLP?"
  RAG:  [finds your actual notes] → LLM: "According to your notes from April 2025..."
```

RAG is the dominant pattern in production AI applications. ~80% of enterprise AI projects use it. It works with any LLM — cloud or local (Ollama).

---

## Core Concept: Semantic Search

Traditional search matches keywords. Semantic search matches *meaning*.

```
Query: "neural network training tricks"

Keyword search finds: documents containing those exact words

Semantic search finds: documents about "gradient clipping", "early stopping",
                       "batch normalization" — even without the original words
```

This is possible because text is converted to **embeddings** — dense vectors where
similar meanings are close together in vector space.

---

## How RAG Works — Full Pipeline

```
── INDEXING (done once, or when docs change) ──────────────────────────────

  Your documents (PDF, markdown, text, code)
        │
        ▼
  ┌─────────────┐
  │  Chunking   │  Split into overlapping pieces, e.g. 400 tokens, 50 overlap
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │  Embedding  │  Each chunk → vector via embedding model
  │  Model      │  (Ollama: nomic-embed-text, mxbai-embed-large)
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │ Vector Store│  Persist vectors + original text + metadata
  │ (ChromaDB,  │  (doc name, page number, chunk index)
  │  Qdrant)    │
  └─────────────┘


── QUERYING (done at runtime for every question) ──────────────────────────

  User question: "how does attention work?"
        │
        ▼
  ┌─────────────┐
  │  Embed      │  question → vector (same embedding model)
  │  question   │
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │ Similarity  │  Find top-K vectors closest to question vector
  │ Search      │  (cosine similarity or dot product)
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │ Augment     │  Build prompt:
  │ Prompt      │  "Answer using this context: [chunk1] [chunk2] [chunk3]
  │             │   Question: how does attention work?"
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │   LLM       │  Generates answer grounded in retrieved chunks
  │ (Ollama,    │  Can cite sources: "According to transformer_notes.pdf..."
  │  Claude)    │
  └─────────────┘
```

**Key insight:** The LLM does not search — it only reads what you hand it. RAG handles retrieval; LLM handles reasoning.

---

## Key Components

### 1. Chunking

How you split documents matters. Bad chunks = bad retrieval.

| Strategy | How | Best for |
|----------|-----|---------|
| Fixed size | Every N tokens, with overlap | Simple, works for most cases |
| Sentence | Split at sentence boundaries | Prose, articles |
| Recursive | Try paragraph → sentence → word | General purpose (LangChain default) |
| Semantic | Group sentences by topic shift | Dense technical docs |
| AST-aware | Split by function/class/method | Source code |

**Overlap** is critical — if a key sentence falls at a chunk boundary, overlap ensures it appears in at least one chunk.

```
Chunk size:  400 tokens
Overlap:     50 tokens
Result:      each chunk shares 50 tokens with the next
```

### 2. Embedding Models

Convert text → dense vector. Same model must be used for both indexing and querying.

| Model | Dimensions | Notes |
|-------|-----------|-------|
| `nomic-embed-text` (Ollama) | 768 | Fast, local, good general purpose |
| `mxbai-embed-large` (Ollama) | 1024 | Higher quality, slower |
| `text-embedding-3-small` (OpenAI) | 1536 | Best quality, requires API key |
| `all-MiniLM-L6-v2` (HuggingFace) | 384 | Tiny, fast, decent quality |

For local-first setups: `nomic-embed-text` via Ollama is the go-to.

### 3. Vector Stores

Where vectors and their source text are stored and searched.

| Store | Type | Notes |
|-------|------|-------|
| ChromaDB | Embedded (no server) | Python-native, simplest to start |
| Qdrant | Server or embedded | Go + Python clients, production-grade |
| FAISS | In-memory library | Fast, no persistence, Meta open-source |
| pgvector | PostgreSQL extension | If you already use Postgres |
| Weaviate | Server | GraphQL API, cloud option |

For local development: **ChromaDB** (Python) or **Qdrant embedded** (Go/Python).

### 4. Retrieval

After embedding the query, find the K most similar chunks.

**Cosine similarity** — most common. Measures angle between vectors (ignores magnitude).

```
similarity = (A · B) / (|A| × |B|)
Range: -1 to 1. Closer to 1 = more similar.
```

**Top-K** — typically retrieve 3–10 chunks. Too few = missing context. Too many = LLM context overflow + noise.

**Reranking** — optional second pass: use a cross-encoder to re-score the top-K for higher precision before sending to LLM.

---

## Prompt Engineering for RAG

How you structure the augmented prompt significantly affects answer quality.

```
Basic RAG prompt:

  CONTEXT:
  [Chunk 1 — source: notes.md, section: transformers]
  Attention is a mechanism that allows the model to weigh...

  [Chunk 2 — source: paper.pdf, page: 4]
  The scaled dot-product attention is computed as...

  QUESTION:
  How does attention work?

  INSTRUCTIONS:
  Answer based only on the context above.
  If the context doesn't contain the answer, say so.
  Cite the source document for each claim.
```

Key instructions to include:
- "Answer based only on the context" — prevents hallucination
- "If the answer isn't in the context, say so" — prevents making things up
- "Cite your sources" — enables verification

---

## Evaluation: How Do You Know RAG Is Working?

| Metric | What it measures |
|--------|----------------|
| **Faithfulness** | Is the answer grounded in retrieved chunks? |
| **Answer relevance** | Does the answer address the question? |
| **Context recall** | Did retrieval find the right chunks? |
| **Context precision** | Are retrieved chunks relevant (not noisy)? |

Tools: **RAGAS** (Python library) automates these metrics using an LLM as judge.

---

## Common Failure Modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Wrong chunks retrieved | Bad chunking or embeddings | Tune chunk size, try different model |
| Answer ignores context | LLM prompt not strict enough | Add "answer ONLY from context" instruction |
| Missing answer in context | Top-K too low | Increase K, add reranking |
| Answer is too long / noisy | Top-K too high | Reduce K, filter by similarity threshold |
| Slow indexing | Large documents, slow embedder | Batch embed, use faster model |
| Stale index | Docs changed but index not updated | Add doc hash tracking, re-index on change |

---

## Advanced Patterns

**Hybrid Search** — combine vector search (semantic) + BM25 (keyword). Better for queries with specific terms (names, acronyms, IDs).

**HyDE (Hypothetical Document Embeddings)** — generate a hypothetical answer to the query, embed that, use it to search. Often retrieves better chunks than embedding the raw question.

**Parent-child chunking** — index small chunks for precise retrieval, but return the larger parent chunk to the LLM for more context.

**Agentic RAG** — LLM decides whether to retrieve, what to query, and whether retrieved results are sufficient before answering. Loop until confident.

**Multi-index** — separate vector stores per document type (code vs. prose vs. tables). Route query to the right index.

---

## Minimal Working Example (Python)

```python
# pip install chromadb ollama langchain-text-splitters

import ollama
import chromadb

client = chromadb.Client()
collection = client.create_collection("docs")

# ── Index ──────────────────────────────────────────
def index_document(text: str, doc_id: str):
    chunks = chunk_text(text)  # split into pieces
    for i, chunk in enumerate(chunks):
        embedding = ollama.embeddings(model="nomic-embed-text", prompt=chunk)
        collection.add(
            ids=[f"{doc_id}_{i}"],
            embeddings=[embedding["embedding"]],
            documents=[chunk],
            metadatas=[{"source": doc_id, "chunk": i}]
        )

# ── Query ──────────────────────────────────────────
def ask(question: str, top_k: int = 4) -> str:
    q_embedding = ollama.embeddings(model="nomic-embed-text", prompt=question)
    results = collection.query(
        query_embeddings=[q_embedding["embedding"]],
        n_results=top_k
    )

    context = "\n\n".join(results["documents"][0])
    sources = [m["source"] for m in results["metadatas"][0]]

    prompt = f"""CONTEXT:
{context}

QUESTION: {question}

Answer based only on the context. Cite sources. If not in context, say so."""

    response = ollama.chat(
        model="llama3.2",
        messages=[{"role": "user", "content": prompt}]
    )
    return response["message"]["content"], sources

def chunk_text(text: str, size: int = 400, overlap: int = 50) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), size - overlap):
        chunks.append(" ".join(words[i:i + size]))
    return chunks
```

---

## RAG in the Career Playbook Context

RAG is skill #3 on the roadmap. It underpins:
- **LLM APIs** (#2) — RAG is the most common LLM integration pattern
- **Agentic AI** (#9) — agents use RAG as their memory/knowledge layer
- **Data Engineering** (#11) — building indexing pipelines is data engineering

**What interviewers expect you to know:**
- What RAG is and why it beats fine-tuning for most use cases
- The chunking → embedding → retrieval → augment loop
- Trade-offs: chunk size, top-K, embedding model choice
- At least one vector store (ChromaDB or Qdrant)
- How to evaluate RAG quality (RAGAS or manual)

**Project to build:** Extend [secondmem](https://github.com/Rinil-Parmar/secondmem) with RAG — `secondmem ingest <file>`, `secondmem ask "<question>"`. Local-first, Ollama-native, Go implementation. Covers everything above in one demonstrable project.
