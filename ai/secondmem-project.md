# secondmem — Full Project Breakdown

**Repo:** [github.com/Rinil-Parmar/secondmem](https://github.com/Rinil-Parmar/secondmem)

**One line:** AI-powered local knowledge management CLI — ingest anything, query by meaning, everything stays on your machine as plain markdown.

---

## What It Is

A personal knowledge management CLI written in Go. Feed it anything — a note, article, PDF, code snippet — and it automatically organizes it into structured markdown files. Ask questions in natural language and it retrieves the most relevant knowledge and answers using an LLM.

No cloud storage. No external databases. Everything lives in `~/.secondmem/`.

---

## Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Language | Go | Fast, single binary, no runtime |
| CLI framework | Cobra + Viper | Commands, flags, config |
| Storage | SQLite (`modernc.org/sqlite`) | Pure Go, no CGO, embedded |
| Full-text search | SQLite FTS5 | Built-in, no extra deps |
| Vector storage | SQLite BLOB | float32 arrays, cosine similarity in Go |
| LLM (chat) | GitHub Copilot / OpenAI / Ollama | Provider-swappable |
| LLM (embed) | Same provider — `text-embedding-3-small` or `nomic-embed-text` | No separate service needed |
| Document parsing | `ledongthuc/pdf` | PDF → text extraction |

---

## Architecture: What Lives Where

```
~/.secondmem/
├── config.toml          — provider, model, paths
├── skill.md             — agent constitution (system prompt)
├── secondmem.db         — SQLite: nodes, edges, FTS5, chunks (vectors)
└── knowledge/
    ├── hierarchy.md     — root table of contents
    ├── ai-ml/
    │   ├── hierarchy.md
    │   └── transformers-self-attention.md
    └── engineering/
        └── ...
```

```
source code/
├── cmd/          — CLI commands (ingest, ask, tree, stats, rebalance...)
├── agent/        — AI logic (classify, ingest, ask, dedup, crossref...)
├── graph/        — SQLite graph: nodes, edges, FTS5, chunks, migrations
├── providers/    — LLM backends: Ollama, OpenAI, Copilot + Embedder interface
├── parsers/      — PDF and text file parsing
└── config/       — config loading with defaults
```

---

## Pipeline 1: Ingestion

When you run `secondmem ingest "some text"` or `secondmem ingest file.md`:

```
Input (text / file / PDF / directory)
         │
         ▼
┌─────────────────────┐
│  Step 1: Exact Dedup│  SHA256 hash checked against all stored hashes
└─────────────────────┘  → skip if identical content already ingested
         │
         ▼
┌─────────────────────┐
│  Step 2: Classify   │  LLM call → directory, filename, summary,
│  (LLM)              │  keywords, related_topics
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Step 3: Semantic   │  FTS finds top-3 keyword matches → LLM compares
│  Dedup (LLM)        │  → skip if >70% duplicate (unless --force)
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Step 4: Write file │  Appends markdown entry: timestamp, summary,
│                     │  source snippet, keywords, SHA256 hash
└─────────────────────┘  → ~/.secondmem/knowledge/{dir}/{file}.md
         │
         ▼
┌─────────────────────┐
│  Step 5: Hierarchy  │  Updates directory/hierarchy.md + root hierarchy.md
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Step 6: Graph      │  SQLite upsert: nodes (filepath, title, summary,
│  Update             │  keywords) + edges to related topic nodes
└─────────────────────┘  FTS5 index updated automatically via triggers
         │
         ▼
┌─────────────────────┐
│  Step 7: RAG Index  │  chunkText(content, size=400, overlap=50)
│                     │  → Embed each chunk via LLM provider API
│                     │  → Store (node_id, chunk_index, content, BLOB)
└─────────────────────┘  in chunks table
         │
         ▼
┌─────────────────────┐
│  Step 8: Cross-Refs │  LLM identifies related files → inserts
│                     │  bidirectional wikilinks between .md files
└─────────────────────┘
```

---

## Pipeline 2: Query (Ask)

When you run `secondmem ask "how does attention work?"`:

```
Question
    │
    ▼
┌──────────────────────────┐
│  Path A: Vector Search   │  Embed(question) → cosine similarity vs all
│  (RAG — primary)         │  stored chunk vectors → top-6 chunks
└──────────────────────────┘
    │                │
    │ chunks found   │ no chunks (empty KB or embed failed)
    │                ▼
    │   ┌──────────────────────────┐
    │   │  Path B: FTS Fallback    │  LLM rewrites question → FTS keywords
    │   │                          │  → FTS5 prefix search → edge traversal
    │   │                          │  → top-7 full .md files as context
    │   └──────────────────────────┘
    │                │
    └────────────────┘
                     │
                     ▼
            ┌─────────────────┐
            │  LLM Answer     │  "Answer ONLY from context. Cite sources."
            └─────────────────┘
                     │
                     ▼
            Answer (+ source file paths with --cite)
```

---

## How RAG Works Here

### Chunking

```go
// 400 words per chunk, 50-word overlap
// overlap ensures no key sentence is lost at a boundary
func chunkText(text string, size=400, overlap=50) []string
```

A 1,000-word note becomes ~3 overlapping chunks. Each embedded independently.

### Embedding Models

| Provider | Embed model | Dimensions |
|----------|------------|------------|
| Copilot / OpenAI | `text-embedding-3-small` | 1536 |
| Ollama | `nomic-embed-text` | 768 |

Same model used for indexing and querying — mandatory, vectors must be in the same space.

### Vector Storage in SQLite

```sql
CREATE TABLE chunks (
    id          INTEGER PRIMARY KEY,
    node_id     INTEGER REFERENCES nodes(id) ON DELETE CASCADE,
    chunk_index INTEGER,
    content     TEXT,
    embedding   BLOB,       -- []float32 as little-endian bytes
    UNIQUE(node_id, chunk_index)
)
```

float32 ↔ bytes encoding:
```go
// encode
binary.LittleEndian.PutUint32(blob[i*4:], math.Float32bits(vec[i]))

// decode
math.Float32frombits(binary.LittleEndian.Uint32(blob[i*4:]))
```

### Cosine Similarity (pure Go)

```go
func cosine(a, b []float32) float64 {
    var dot, normA, normB float64
    for i := range a {
        dot   += float64(a[i]) * float64(b[i])
        normA += float64(a[i]) * float64(a[i])
        normB += float64(b[i]) * float64(b[i])
    }
    return dot / (math.Sqrt(normA) * math.Sqrt(normB))
}
```

Loads all vectors into memory, sorts by score, returns top-K. Fast for a personal KB (thousands of chunks, not millions).

---

## Provider Abstraction

Two interfaces in `providers/provider.go`:

```go
type LLMProvider interface {
    Complete(systemPrompt, userPrompt string) (string, error)
}

type Embedder interface {
    Embed(text string) ([]float32, error)
}
```

All three providers implement both. In the cmd layer, a single type assertion handles it:

```go
provider, _ := newProvider(cfg)
embedder, _ := provider.(providers.Embedder)  // zero config needed
```

---

## The Graph (LORE-GRAPH)

SQLite with four layers:

| Table | Purpose |
|-------|---------|
| `nodes` | One row per .md file — filepath, title, summary, keywords |
| `edges` | Relationships between nodes with weights |
| `nodes_fts` | FTS5 virtual table — auto-synced via SQLite triggers |
| `chunks` | RAG vector index — chunk content + embedding BLOB |

FTS5 is the fallback when no chunks exist (content ingested before RAG was added). Edge traversal expands results — if `transformers.md` matches, its related node `attention-mechanism.md` is also pulled in.

---

## Migration System

Auto-runs on `graph.Open()` using Go's `embed.FS`:

```
graph/migrations/
  001_initial.sql   — nodes, edges, FTS5, triggers
  002_chunks.sql    — chunks table (RAG addition)
```

Tracks applied versions in `schema_migrations` table. No external migration tool.

---

## Key Design Decisions

**Why SQLite for vectors?** Personal KB = thousands of chunks, not millions. In-memory cosine similarity takes milliseconds. Avoids running a separate vector DB process (Qdrant, Weaviate) which defeats the local-first goal.

**Why two retrieval paths?** FTS still works on notes ingested before RAG was added. Vector search wins when chunks exist; FTS is the safety net.

**Why chunk original content, not the stored .md file?** The stored file has markdown formatting, timestamps, hash lines. Original content is cleaner signal for embedding.

**Why 400 words / 50 overlap?** 400 words fits within `text-embedding-3-small`'s 8191 token limit. 50-word overlap ensures boundary sentences aren't lost.

**Why type-assert for Embedder?** All providers support embeddings. Zero config — users don't need to know embeddings exist.

---

## Further Reading

- [RAG deep dive](./rag.md) — chunking, embeddings, vector stores, cosine similarity, evaluation
- [secondmem RAG extension plan](./secondmem-rag.md) — architecture decisions, full Go implementation
