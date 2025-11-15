# SPEC.md — Echo Box - Technical Q&A Assistant (RAG System)
Version: 1.0
Target: Code generation (Codex)
Purpose: Provide a complete, authoritative specification for constructing the Technical Q&A Assistant system as per the ADRs and consolidated architecture.

---

## 1. Overview
Build a standalone Retrieval-Augmented Generation (RAG) system that answers natural-language questions using a curated markdown knowledge base.
The system has two phases:

### **Ingestion (offline)**
- Parse, clean, chunk, enrich, embed, and index markdown content.

### **Runtime (online)**
- Accept a question,
- retrieve relevant chunks (hybrid retrieval),
- construct a structured prompt,
- call an LLM,
- and return a grounded answer with citations.

The system must follow all architectural decisions (ADR-001 → ADR-009).
The entire system must run locally, with no external services beyond the LLM API.

## 2. Tech Stack Requirements
### **Python**
 3.11+

### **Directory structure:**

```
/app
    /ingestion
    /retrieval
    /storage
    /prompting
    /llm
    /observability
    /evaluation
    /api (optional)
```

### **Dependencies:**

- sentence-transformers or OpenAI embeddings
- chromadb or qdrant-client
- rank-bm25
- sqlite3
- markdown or mistune
- pydantic
- uvicorn (optional)
- fastapi (optional for API)

## 3. Ingestion Pipeline Requirements

Implement ingestion as a **deterministic offline pipeline**.

## 3.1 Steps (in order)

### **1. Load markdown files**
- Read all `.md` files from a configured directory.
- Store original file path + filename.

### **2. Clean & normalise text**
- Strip HTML.
- Normalise whitespace.
- Remove unsupported Markdown constructs.
- Preserve section headings.

### **3. Sentence split**
- Use a rule-based or lightweight NLP splitter (**no LLM**).
- Output a list of sentences.

### **4. Chunking (ADR-003)**
- Sentence-aware.
- Target size: **150–300 tokens**.
- Sliding overlap: **20–30%**.
- Preserve section titles in metadata.
- Generate canonical `chunk_id`.

### **5. Enrichment (ADR-005)**
- **No LLM rewriting.**
- Metadata fields:
  - `file_name`
  - `file_path`
  - `section_title`
  - `line_start`
  - `line_end`
  - `chunk_id`
  - `chunk_index`
  - `created_at`
- Record token count.

### **6. Generate embeddings (ADR-007)**
- Use OpenAI or the configured embedding provider.
- Store 1536-dim vectors (or model default).

### **7. Write Chunk Store (SQLite)**

**Table: `chunks`**

| Field          | Description |
|----------------|-------------|
| chunk_id (PK)  | canonical ID |
| text           | chunk text |
| file_name      | original file |
| file_path      | path on disk |
| section_title  | markdown section |
| line_start     | inclusive |
| line_end       | inclusive |
| chunk_index    | sequence number |
| token_count    | integer |
| created_at     | ISO timestamp |

### **8. Write Vector Index (Chroma/Qdrant)**
- Use `chunk_id` as primary key.
- Store vector + metadata.

### **9. Write Lexical Index (BM25)**
- Use cleaned chunk text.
- Maintain mapping: BM25 index → `chunk_id`.

**The pipeline must be repeatable and idempotent.**

---

# 4. Runtime Retrieval Pipeline

Runtime query handling must follow **ADR-002**.

## 4.1 Steps

### **1. Preprocess user query**
- Lowercase normalization.
- Strip punctuation.
- Tokenise for BM25.

### **2. Lexical retrieval (BM25)**
- Return top-N lexical matches with BM25 scores.

### **3. Semantic retrieval (vector search)**
- Embed query using the same embedding model.
- Retrieve top-N vector matches.

### **4. Hybrid merging**
- Normalise both score lists.
- Merge using:

```
final_score = α * semantic_score + (1 - α) * bm25_score
```
- α configurable (default = 0.5).

### **5. Fetch chunk text**
- Load selected chunks from SQLite.
- Remove duplicates.
- Return top-K final chunks.

### **6. Log retrieval transparency (ADR-008)**
Include:

- lexical hits  
- semantic hits  
- merged rankings  
- selected chunks  

---

# 5. Prompt Construction (ADR-006)

Construct a **four-block structured prompt**, in this exact order.

### **5.1 Block 1 — System Instructions**

- You are a grounded technical question-answering assistant. 
- You MUST answer ONLY using the provided context chunks. 
- If the answer is not contained within the context, respond with:
- "I don’t have enough information in the provided documents to answer that."
- Never invent facts.

### **5.2 Block 2 — User Question**

```
Question:
{{ user_query }}
```

### **5.3 Block 3 — Retrieved Context**
Format each chunk as:
```
[chunk {{chunk_id}} — {{file_name}}, lines {{line_start}}–{{line_end}}]
{{chunk_text}}
```

### **5.4 Block 4 — Answer Rules**

- Answer using only the context. 
- Cite chunks using [chunk-id]. 
- Prefer concise, technical explanations. 
- Do not use external knowledge.

# 6. LLM Provider Layer (ADR-007)

Implement a provider interface:

```
class LLMProvider:
    def embed(self, text: str) -> List[float]:
    def complete(self, prompt: str, max_tokens: int) -> str:
```

Concrete implementations:
- OpenAIProvider
- (Optional) AnthropicProvider

Provider selection is determined by a config file.

# 7. Observability (ADR-008)
Implement **local-only**, minimal observability:

## 7.1 Logging Requirements

Logs must be json with the following fields:
- timestamp
- event
- query
- bm25_results
- semantic_results
- merged_results
- selected_chunks
- llm_latency_ms
- error

## 7.2 No external systems
- No OTel
- No collectors
- No dashboards
- No metrics backend

Logs are written to the local filesystem.

## 8. Evaluation & Feedback (ADR-009)

## 8.1 Passive evaluation

Implement:
- A fixed evaluation directory: `/evaluation/eval_set.jsonl`

Each entry contains:

```json
{
  "question": "...",
  "expected_answer": "..."
}
```

## 8.2 Manual feedback

Provide optional CLI command:
qa mark-bad --id <session_id>
This simply writes metadata into a local JSON file (no scoring engine).

# 9. Non-Functional Requirements
**Grounding**: No hallucination allowed.
**Transparency**: Full retrieval trace logged.
**Maintainability**: Modular, small files, clean interfaces.
**Extensibility**: Easy to swap embedding model, LLM, or index.
**Simplicity**: Minimal dependencies, local-only architecture.

# 10. Deliverables Codex Should Generate

Codex must generate:

### **11.1 Project structure**

A full directory tree with Python packages for:

-  ingestion
-  retrieval
-  storage
-  prompting
-  llm
-  observability
-  evaluation
-  api

### **11.2 Complete implementation**

- Full ingestion pipeline
- Hybrid retrieval pipeline
- Prompt builder
- LLM provider interface + implementation
- Logging layer
- Evaluation tools
- CLI scripts

### **11.3 Configuration files**

.env
config.yaml
requirements.txt

### **11.4 Tests**

Provide PyTest tests for:
- chunking
- enrichment
- embedding stub
- retrieval merging logic
- prompt construction

# 12. What Codex Must Not Do

- Must not invent new architecture choices.
- Must not introduce LLMs into ingestion.
- Must not attempt to build dashboards, UIs, or external observability.
- Must not add cloud dependencies.
- Must not ignore grounding rules.

# end of spec
