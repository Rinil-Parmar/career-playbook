# RAG Extension for secondmem

**Goal:** Upgrade secondmem's keyword search (FTS5) to semantic vector search — find notes by meaning, not just exact words.

---

## Current vs RAG Flow

```
── CURRENT ──────────────────────────────────────────────────
secondmem ask "question"
  → FTS5 keyword search on nodes (SQLite)
  → Returns nodes matching keywords
  → LLM answers from node summaries

── WITH RAG ─────────────────────────────────────────────────
secondmem ask "question"
  → Embed question → vector (Ollama: nomic-embed-text)
  → Cosine similarity search across stored chunk vectors
  → Returns top-K semantically relevant chunks
  → LLM answers with cited source files
```

**Key gain:** "how do I speed up training?" finds notes about "learning rate scheduling" and "gradient clipping" — even without those exact words.

---

## Architecture

```
INGEST (per file):
  file content
    → chunk (400 tokens, 50 overlap)
    → embed each chunk via Ollama /api/embeddings
    → store in chunks table (SQLite BLOB)
    → existing node creation unchanged

ASK (per query):
  question
    → embed via Ollama
    → cosine similarity vs all chunk vectors (in-memory Go)
    → top-K chunks → build context string
    → LLM with RAG prompt → cited answer
```

---

## Database Schema Addition

New table alongside existing `nodes`:

```sql
-- graph/migrations/002_chunks.sql
CREATE TABLE IF NOT EXISTS chunks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    node_id     INTEGER NOT NULL REFERENCES nodes(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content     TEXT    NOT NULL,
    embedding   BLOB    NOT NULL   -- []float32 as raw little-endian bytes
);

CREATE INDEX IF NOT EXISTS idx_chunks_node_id ON chunks(node_id);
```

---

## New / Modified Files

```
graph/
  chunks.go          NEW — SaveChunk, SearchByVector, cosine similarity
  migrate.go         MODIFY — run migration 002

providers/
  ollama.go          MODIFY — add Embed(text string) ([]float32, error)

agent/
  ingest.go          MODIFY — chunk + embed after node creation
  ask.go             MODIFY — replace FTS with vector search + RAG prompt
```

No new Go dependencies. Everything uses existing `go.mod`:
- `modernc.org/sqlite` — vector storage as BLOB
- `net/http` — already used for Ollama calls
- Cosine similarity: pure Go math, ~5 lines

---

## Implementation Detail

### `providers/ollama.go` — add Embed method

```go
type ollamaEmbedRequest struct {
    Model  string `json:"model"`
    Prompt string `json:"prompt"`
}

type ollamaEmbedResponse struct {
    Embedding []float32 `json:"embedding"`
}

func (p *OllamaProvider) Embed(text string) ([]float32, error) {
    reqBody := ollamaEmbedRequest{Model: "nomic-embed-text", Prompt: text}
    jsonData, err := json.Marshal(reqBody)
    if err != nil {
        return nil, err
    }

    resp, err := p.client.Post(p.baseURL+"/api/embeddings", "application/json", bytes.NewReader(jsonData))
    if err != nil {
        return nil, fmt.Errorf("embed request failed: %w", err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("ollama embed status %d: %s", resp.StatusCode, body)
    }

    var result ollamaEmbedResponse
    if err := json.Unmarshal(body, &result); err != nil {
        return nil, err
    }
    return result.Embedding, nil
}
```

### `graph/chunks.go` — storage + cosine similarity

```go
package graph

import (
    "encoding/binary"
    "math"
)

type Chunk struct {
    ID         int64
    NodeID     int64
    ChunkIndex int
    Content    string
    FilePath   string  // joined from nodes table
}

func (g *Graph) SaveChunk(nodeID int64, index int, content string, embedding []float32) error {
    blob := make([]byte, len(embedding)*4)
    for i, v := range embedding {
        binary.LittleEndian.PutUint32(blob[i*4:], math.Float32bits(v))
    }
    _, err := g.db.Exec(
        `INSERT INTO chunks (node_id, chunk_index, content, embedding) VALUES (?, ?, ?, ?)
         ON CONFLICT DO NOTHING`,
        nodeID, index, content, blob,
    )
    return err
}

func (g *Graph) SearchByVector(query []float32, topK int) ([]Chunk, error) {
    rows, err := g.db.Query(`
        SELECT c.id, c.node_id, c.chunk_index, c.content, c.embedding, n.file_path
        FROM chunks c JOIN nodes n ON n.id = c.node_id`)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    type scored struct {
        chunk Chunk
        score float64
    }
    var results []scored

    for rows.Next() {
        var c Chunk
        var blob []byte
        if err := rows.Scan(&c.ID, &c.NodeID, &c.ChunkIndex, &c.Content, &blob, &c.FilePath); err != nil {
            return nil, err
        }
        vec := blobToFloat32(blob)
        score := cosine(query, vec)
        results = append(results, scored{c, score})
    }

    // sort descending by score
    sort.Slice(results, func(i, j int) bool {
        return results[i].score > results[j].score
    })

    chunks := make([]Chunk, 0, topK)
    for i := 0; i < topK && i < len(results); i++ {
        chunks = append(chunks, results[i].chunk)
    }
    return chunks, nil
}

func cosine(a, b []float32) float64 {
    var dot, normA, normB float64
    for i := range a {
        dot += float64(a[i]) * float64(b[i])
        normA += float64(a[i]) * float64(a[i])
        normB += float64(b[i]) * float64(b[i])
    }
    if normA == 0 || normB == 0 {
        return 0
    }
    return dot / (math.Sqrt(normA) * math.Sqrt(normB))
}

func blobToFloat32(blob []byte) []float32 {
    vec := make([]float32, len(blob)/4)
    for i := range vec {
        bits := binary.LittleEndian.Uint32(blob[i*4:])
        vec[i] = math.Float32frombits(bits)
    }
    return vec
}
```

### `agent/ingest.go` — add chunking + embedding

After existing node upsert, add:

```go
// chunk the raw content
chunks := chunkText(rawContent, 400, 50)

// embed and store each chunk
for i, chunk := range chunks {
    vec, err := provider.Embed(chunk)
    if err != nil {
        fmt.Printf("  warning: embed chunk %d failed: %v\n", i, err)
        continue
    }
    if err := g.SaveChunk(nodeID, i, chunk, vec); err != nil {
        fmt.Printf("  warning: save chunk %d failed: %v\n", i, err)
    }
}

// chunkText splits text into overlapping windows by word count
func chunkText(text string, size, overlap int) []string {
    words := strings.Fields(text)
    var chunks []string
    for i := 0; i < len(words); i += size - overlap {
        end := i + size
        if end > len(words) {
            end = len(words)
        }
        chunks = append(chunks, strings.Join(words[i:end], " "))
        if end == len(words) {
            break
        }
    }
    return chunks
}
```

### `agent/ask.go` — switch to vector retrieval

```go
// embed the question
qVec, err := provider.Embed(question)
if err != nil {
    // fallback to FTS if embed fails
    return agent.AskFTS(cfg, provider, g, question, cite)
}

// semantic retrieval
chunks, err := g.SearchByVector(qVec, 5)
if err != nil || len(chunks) == 0 {
    return agent.AskFTS(cfg, provider, g, question, cite)
}

// build context
var contextParts []string
for _, c := range chunks {
    contextParts = append(contextParts, fmt.Sprintf("[%s]\n%s", c.FilePath, c.Content))
}
context := strings.Join(contextParts, "\n\n---\n\n")

systemPrompt := `You are a knowledge assistant. Answer using ONLY the provided context.
If the answer is not in the context, say so. Cite the source file for each claim.`

userPrompt := fmt.Sprintf("CONTEXT:\n%s\n\nQUESTION: %s", context, question)

return provider.Complete(systemPrompt, userPrompt)
```

---

## Build Phases

| Phase | What | Files touched |
|-------|------|--------------|
| 1 | Add Embed() to OllamaProvider | `providers/ollama.go` |
| 2 | Add chunks table + graph methods | `graph/chunks.go`, `graph/migrate.go`, `graph/migrations/002_chunks.sql` |
| 3 | Chunk + embed on ingest | `agent/ingest.go` |
| 4 | Semantic search in ask | `agent/ask.go` |

Test after each phase. Phase 1+2 can be verified without touching ingest or ask.

---

## Testing

```bash
# Phase 1 — verify embed works
ollama pull nomic-embed-text
# run a quick Go test that calls provider.Embed("hello world") and prints vector length

# Phase 2 — verify storage
secondmem ingest some_file.md
# check SQLite: SELECT count(*) FROM chunks;

# Phase 3+4 — end to end
secondmem ask "what did I write about X?"
# should cite source file and return semantically relevant content
```

---

## Embedding Model

`nomic-embed-text` via Ollama — 768-dimension vectors, fast, local, no API key.

```bash
ollama pull nomic-embed-text
```

Must be pulled before running ingest. Add to README as prerequisite.
