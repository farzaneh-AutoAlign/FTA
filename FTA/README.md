# AI Troubleshooting Copilot (FTA)

A **RAG-backed conversational AI** that turns troubleshooting manuals (PDFs, images, text) into interactive fault trees. Users describe an error or symptom; the system retrieves relevant chunks from ingested documents, generates a structured fault tree with causes, remediations, and source citations, and supports multi-turn follow-up (narrowing, deepening, cross-referencing).

The project supports two modes:

1. **Batch pipeline** — Process a single PDF end-to-end: extract root errors (Gemini), deduplicate (OpenAI embeddings), build enhanced fault trees (GPT), save to JSON.
2. **Web app** — Ingest multiple documents into a vector + keyword index, then query via a chat UI; the agent returns fault trees grounded in retrieved context.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Setup (In-Depth)](#setup-in-depth)
- [Usage](#usage)
- [API Reference](#api-reference)
- [Implementation Plan & Roadmap](#implementation-plan--roadmap)
- [Current Limitations](#current-limitations)
- [Project Structure](#project-structure)
- [Docker](#docker)
- [Verification](#verification)

---

## Architecture Overview

| Layer | Components | Role |
|-------|------------|------|
| **Ingestion** | `ingest.py`, `parser.py`, `document_store.py` | Upload/parse PDFs, images, text; register documents; chunk with metadata |
| **Indexing** | `indexer.py` | ChromaDB (vector) + BM25 (keyword); hybrid search via RRF |
| **Extraction** | `extract.py` | Gemini: extract root error names; OpenAI: semantic deduplication |
| **Fault trees** | `fault_tree.py` | GPT: build enhanced trees (remediation, citations, confidence, gaps) |
| **Agent** | `agent.py`, `session.py` | RAG retrieval → context assembly → LLM → fault tree or text reply; session logging |
| **API & UI** | `server.py`, `static/index.html` | FastAPI endpoints; chat UI with fault tree rendering and citations |

- **Batch pipeline** uses: `ingest.upload_pdf` → `extract` → `fault_tree` → JSON output.
- **Web app** uses: document ingestion into `DocumentRegistry` + ChromaDB + BM25 → `TroubleshootingAgent.query()` → response with optional fault tree and sources.

---

## Prerequisites

- **Python 3.12+**
- **API keys and (optional) GCP:**
  - **OpenAI**: used for embeddings, fault tree generation (batch and agent), and (in batch) file upload. Set `OPENAI_API_KEY` in `.env`.
  - **Vertex AI / Gemini** (optional): used for root-error extraction in the batch pipeline and for image captioning during ingestion. Set `VERTEX_PROJECT`, `VERTEX_LOCATION`; ensure `gcloud` is configured for Application Default Credentials if using Vertex.
- **System**: For PDF parsing, PyMuPDF is used (no extra system deps on macOS; on Linux, the Dockerfile installs `build-essential`).

---

## Setup (In-Depth)

### 1. Clone and enter the project

```bash
cd /path/to/FTA
```

### 2. Create a virtual environment (recommended)

```bash
python3.12 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

Key dependencies: `openai`, `google-cloud-aiplatform`, `pydantic`, `pymupdf`, `chromadb`, `fastapi`, `uvicorn`, `python-dotenv`, `python-multipart`, `numpy`.

### 4. Configure environment variables

Create a `.env` file in the project root (the repo ignores `.env`):

```env
# Required for batch pipeline and web app (embeddings, fault trees, file upload)
OPENAI_API_KEY=sk-...

# Optional: for batch pipeline root-error extraction and image captioning
VERTEX_PROJECT=your-gcp-project-id
VERTEX_LOCATION=global
```

- If you skip Vertex: the batch pipeline will fail at the Gemini step unless you change the flow; the web app will run but image ingestion will use placeholder text instead of Gemini captions.
- For Vertex AI, ensure you have run `gcloud auth application-default login` (or equivalent) so that the client can authenticate.

### 5. Prepare directories (optional)

The app creates these at runtime if missing:

- `sessions/` — troubleshooting session JSON files
- `.chromadb/` — ChromaDB persistence
- `static/` — must contain `index.html` for the chat UI

No need to create them manually unless you want to pre-mount volumes (e.g. in Docker).

### 6. Verify installation

- **Batch pipeline** (requires one PDF and OpenAI + Gemini configured):

  ```bash
  python pipeline.py --pdf "path/to/troubleshooting.pdf"
  ```

- **Web app** (requires at least OpenAI):

  ```bash
  uvicorn server:app --host 0.0.0.0 --port 8000
  ```

  Then open `http://localhost:8000`. Ingest documents via the UI or via `POST /api/ingest` before querying.

---

## Usage

### Batch pipeline (single PDF → fault trees JSON)

1. Ensure `OPENAI_API_KEY` and (for extraction) Vertex/Gemini are set.
2. Run:

   ```bash
   python pipeline.py --pdf "06 SCA Trouble Shooting Guides Rev11072025.pptx.pdf"
   python pipeline.py --pdf manual.pdf --output results.json
   ```

3. Steps performed:
   - Upload PDF to Gemini (in-memory) and to OpenAI (file API).
   - Extract all root error names with Gemini.
   - Semantic deduplication with OpenAI embeddings.
   - For each unique error, build an enhanced fault tree with GPT (with remediation, citations, confidence, gaps).
   - Validate schema, print trees, and save to `fault_trees.json` (or `--output`).

### Web app (multi-document RAG + chat)

1. Start the server:

   ```bash
  uvicorn server:app --host 0.0.0.0 --port 8000
  ```

2. **Ingest documents**
   - **Via UI**: Use the ingest/upload control on the chat page to upload PDFs, images, or text files.
   - **Via API**: `POST /api/ingest` with multipart form data (see [API Reference](#api-reference)).

3. **Query**
   - Type a troubleshooting question or error description (e.g. “Pump not building pressure”).
   - The agent runs hybrid search (vector + BM25), assembles context, and returns either a fault tree (JSON) or a plain-text reply. The UI renders fault trees with branches, remediation steps, and source citations.

4. **Follow-up**
   - Use the same session (session ID is returned with each response) to narrow (“I already checked X”), deepen (“Tell me more about Y”), or cross-reference; the agent uses conversation history and can re-query the index.

### Ingesting a directory (programmatic)

There is no standalone `ingest_docs.py` CLI. To ingest a directory from code:

```python
from config import init_openai, init_gemini
from document_store import DocumentRegistry
from indexer import VectorIndex, BM25Index
from ingest import ingest_documents

registry = DocumentRegistry(storage_dir=".")
vector_index = VectorIndex(persist_dir=".chromadb")
bm25_index = BM25Index()
openai_client = init_openai()
gemini_model = init_gemini()  # optional, for image captioning

ingest_documents(
    paths=["./manuals/"],
    registry=registry,
    vector_index=vector_index,
    bm25_index=bm25_index,
    openai_client=openai_client,
    gemini_model=gemini_model,
)
```

The server does this internally when you call `POST /api/ingest` with uploaded files.

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Serve the chat UI (`static/index.html`). |
| `POST` | `/api/query` | Submit a troubleshooting query. Body: `{"message": "...", "session_id": "optional"}`. Returns `session_id`, `response` (text or fault tree JSON), `is_fault_tree`, `sources`, `confidence`. |
| `POST` | `/api/ingest` | Upload and ingest documents. Multipart form: `files` = list of files (PDF, image, or text). Returns `status`, `files_processed`, `total_chunks`. |
| `GET` | `/api/sessions` | List all sessions (summary: `session_id`, `created_at`, `resolution`, `turn_count`, etc.). |
| `GET` | `/api/sessions/{session_id}` | Get full session history (turns, retrieved chunks, fault trees). |
| `GET` | `/api/documents` | List ingested documents from the registry. |

**Error handling:** The server returns user-friendly messages for common issues (e.g. quota exceeded, invalid API key). See [Current Limitations](#current-limitations).

---

## Implementation Plan & Roadmap

The system is designed in phases (see `implementation_plan.md` for full detail):

| Phase | Focus | Deliverable |
|-------|--------|--------------|
| **2a** | Enhanced fault trees | Remediation, citations, confidence, gaps in pipeline output. |
| **2b** | Multi-document RAG | `parser.py`, `indexer.py`, `document_store.py`; ChromaDB + BM25; batch ingestion. |
| **2c** | Conversational agent + UI | `agent.py`, `session.py`, `server.py`, `static/index.html`; chat + fault tree rendering. |
| **2d** | Async + containerization | Async fault tree generation (optional); `Dockerfile`, `docker-compose.yml`. |

Current codebase implements 2a–2d: enhanced schema, multi-format ingestion, hybrid search, agent, session store, FastAPI + chat UI, and Docker. Async fault tree building in the batch pipeline is not yet implemented (synchronous only).

---

## Current Limitations

1. **API rate limits**
   - **OpenAI**: Request rate and token limits depend on your plan. Under heavy use (many concurrent queries or large batch runs), you may hit **429 (rate limit)**. The server maps 429 to a user-facing message suggesting checking billing/plan at https://platform.openai.com/account/billing.
   - **Vertex AI / Gemini**: Subject to Vertex quota and rate limits. Failures at startup or during extraction/captioning will log a warning or raise; the batch pipeline will fail at the Gemini step if Vertex is not configured or limited.

2. **Insufficient balance / quota**
   - **OpenAI**: If your account has **insufficient quota** or **insufficient_quota** errors, the API returns an error that the server translates to a message directing you to check your OpenAI plan and billing. This is a hard limit until you add payment method or upgrade.
   - **GCP / Vertex**: Quota and billing apply per project. If Vertex calls fail due to quota or billing, fix the GCP project billing and quotas; the app does not retry or fall back to another provider.

3. **Operational**
   - **Image captioning** requires Gemini (Vertex). If Gemini is not initialised or fails, ingested images get a placeholder chunk instead of a real caption.
   - **BM25 index** is in-memory; it is rebuilt on server startup from ChromaDB metadata so that hybrid search works after restarts.
   - **Batch pipeline** is synchronous; one fault tree per error, no parallelisation (async variant is planned in the implementation plan but not yet in code).

---

## Project Structure

```
FTA/
├── config.py              # Constants, Pydantic models, Gemini/OpenAI client init
├── ingest.py              # PDF upload (batch) + multi-format ingest_documents (RAG)
├── parser.py              # PDF (PyMuPDF), image (Gemini caption), text chunking
├── document_store.py      # Document registry (JSON sidecar)
├── indexer.py             # ChromaDB vector index, BM25, hybrid_search (RRF)
├── extract.py             # Root error extraction (Gemini), semantic dedup (OpenAI)
├── fault_tree.py          # Enhanced fault tree prompt, build, validate, print_tree
├── agent.py               # TroubleshootingAgent: RAG retrieval → LLM → fault tree/text
├── session.py             # Session and SessionStore (disk-backed)
├── server.py              # FastAPI app, /api/query, /api/ingest, /api/sessions, /api/documents
├── pipeline.py            # CLI: --pdf, --output (batch pipeline)
├── static/
│   └── index.html         # Chat UI, fault tree display, citations
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── implementation_plan.md
├── troubleshooting-copilot-workflow.md
├── troubleshooting-copilot-workflow.md.txt
├── .env                   # API keys (gitignored)
└── .gitignore
```

Generated at runtime (gitignored): `.chromadb/`, `sessions/`, `document_registry.json`, `fault_trees.json`.

---

## Docker

- **Build and run with Docker Compose:**

  ```bash
  docker-compose up --build
  ```

  The app listens on port 8000. Volumes persist ChromaDB (`.chromadb`) and sessions (`sessions`). Optionally mount a folder (e.g. `./manuals`) read-only for reference; ingestion is still done via the API or UI with uploaded files.

- **Environment:** Use an `.env` file in the same directory as `docker-compose.yml`; it is loaded via `env_file: - .env`.

- **Dockerfile:** Based on `python:3.12-slim`, installs system deps for PyMuPDF, copies the app, and runs `uvicorn server:app --host 0.0.0.0 --port 8000`.

---

## Verification

- **Phase 2a (enhanced fault trees):** Run `python pipeline.py --pdf <your.pdf>` and confirm output JSON includes `remediation`, `source`, `confidence`, and `gaps` per node/branch.
- **Phase 2b (RAG):** Ingest several PDFs/images via the UI or API; call `GET /api/documents`; run a query that matches known error text and check that returned `sources` point to the right file/page.
- **Phase 2c (agent + UI):** Open the chat UI, ingest docs, submit a query, and confirm you get a fault tree or text plus citations; test a follow-up in the same session.
- **Phase 2d (Docker):** `docker-compose up`, then repeat Phase 2c against the container; restart the container and confirm ChromaDB-backed search still works (BM25 is rebuilt on startup from ChromaDB).

---

## Review Request Template (for external code reviews)

When submitting this project for an external review (e.g. fellowship, mentorship, or code review program), you can use the following pre-filled requests.

### Request 1 — Fault tree generation & prompt

- **What do you want reviewed?**  
  “Our fault tree generation prompt + schema validation for enhanced fault trees (remediation, citations, confidence, gaps).”

- **Code snippet, file reference, or Gist link**  
  `fault_tree.py` (functions `FAULT_TREE_PROMPT`, `build_fault_tree_openai`, `validate_tree_schema`) and its usage in `pipeline.py`.

- **What’s your specific question or concern?**  
  Pick 1–2 of:
  - “Are there clearer ways to phrase the prompt to reduce schema violations and partial JSON while still keeping it grounded to the PDF content?”
  - “Is our current JSON schema for fault trees reasonable, or would you simplify/reshape it for downstream usage and maintainability?”
  - “Is `validate_tree_schema()` strict enough, or should we validate more fields and fail harder on bad outputs?”

### Request 2 — RAG ingestion, indexing, and retrieval

- **What do you want reviewed?**  
  “Our multi‑document ingestion and hybrid retrieval stack (parser, document registry, ChromaDB vector index, BM25 keyword index, and `hybrid_search`).”

- **Code snippet, file reference, or Gist link**  
  `ingest.py`, `parser.py`, `document_store.py`, `indexer.py` (especially `VectorIndex`, `BM25Index`, and `hybrid_search`).

- **What’s your specific question or concern?**  
  Pick 1–2 of:
  - “Are our chunking and metadata choices (page, section, chunk_index) appropriate for high‑quality troubleshooting retrieval, or would you adjust the strategy?”
  - “Does our hybrid vector + BM25 design (RRF) look sound and efficient, or are there obvious improvements to ranking or filtering we should consider?”
  - “Are there edge cases in ingestion (PDF/image/text) that you’d expect to break, and how would you harden this layer?”

### Request 3 — Agent, API design, and error handling

- **What do you want reviewed?**  
  “The conversational agent and FastAPI server layer that powers `/api/query`, `/api/ingest`, session storage, and error handling.”

- **Code snippet, file reference, or Gist link**  
  `agent.py`, `server.py`, `session.py`, and optionally `static/index.html` for how the UI calls the API.

- **What’s your specific question or concern?**  
  Pick 1–2 of:
  - “Is the `TroubleshootingAgent.query()` flow (context assembly, message history, parsing JSON vs text) a robust pattern, or would you structure it differently?”
  - “Is our API surface (`/api/query`, `/api/ingest`, `/api/sessions`, `/api/documents`) well‑designed for future evolution (e.g. auth, multi‑tenant use), or should we refactor it now?”
  - “Is our error handling (especially for rate limits, insufficient quota, and upstream LLM failures) sufficient for production, or what patterns would you recommend?”

### Menu of specific question ideas

You can swap any of these into the “What’s your specific question or concern?” field for a given request:

- **Prompting, LLM behavior, and schema enforcement**
  - “How can we make the prompt more robust so the model consistently returns valid JSON without post‑processing hacks?”
  - “Are we over‑constraining the model with our instructions, or is this level of rigidity appropriate for production troubleshooting output?”
  - “Would you recommend adding tool‑calling / function‑calling instead of pure JSON‑style prompts here?”

- **Data modeling and schemas**
  - “Is our fault tree JSON schema (top_event/branches/root_causes/remediation/confidence/gaps) a good long‑term structure, or would you normalize it differently?”
  - “Are we storing enough metadata on chunks (doc_id, source_file, page_number, section, chunk_index) to support advanced analytics and debugging later?”
  - “Should we promote more of the JSON schemas into typed Pydantic models to catch issues earlier?”

- **RAG ingestion, chunking, and retrieval**
  - “Is our PDF parsing and fixed‑size chunking strategy reasonable for troubleshooting tables, or should we move to table‑aware / semantic chunking?”
  - “Do the defaults for `CHUNK_SIZE` and `CHUNK_OVERLAP` look sensible for this domain, or would you tune them differently?”
  - “Are there obvious improvements to our BM25 implementation or hybrid ranking that would materially improve retrieval quality?”
  - “Are we doing enough to prevent duplicate or near‑duplicate chunks from polluting the index?”

- **Performance, scalability, and cost**
  - “Given our use of OpenAI embeddings and completion calls, what would you change to reduce cost or latency under heavy load?”
  - “Is our current design for ChromaDB usage likely to scale for thousands of documents, or should we plan a different vector store earlier?”
  - “Where would you add caching (e.g. embeddings, fault trees, or retrieval results) to balance freshness vs performance?”

- **Reliability, robustness, and error handling**
  - “Are we handling partial failures (e.g. some docs failing to ingest) in a robust way, or should we fail fast / add retries and dead‑letter semantics?”
  - “Is the current approach to schema validation and fallback parsing for LLM responses good enough, or should we fail hard and alert instead?”
  - “How can we better surface and log cases where the model hallucinates sources or remediations?”

- **API & backend design (FastAPI, sessions, server)**
  - “Is our API design (endpoints, payload shapes) idiomatic and easy to integrate with other clients, or would you reorganize the routes?”
  - “Does the current session model (`Session` and `SessionStore`) look maintainable and safe under concurrent usage?”
  - “Are there improvements you’d recommend around streaming responses for long‑running queries or fault tree generation?”

- **Frontend & UX (chat UI and tree display)**
  - “Is our current chat UI and fault tree visualization intuitive for technicians, or what key UX improvements would you suggest?”
  - “Are we exposing enough of the underlying sources and confidence scores to let users trust (or challenge) the AI’s recommendations?”
  - “How should we evolve the UI to better support follow‑up queries like narrowing, deepening, and cross‑referencing?”

- **Security, configuration, and DevOps**
  - “Are there obvious security gaps in how we handle API keys, environment variables, and uploaded files?”
  - “Does the Dockerfile + docker‑compose setup look production‑ready, or what changes would you make for a real deployment?”
  - “How would you suggest introducing observability (metrics, tracing, structured logs) into this architecture?”

- **Testing and verification**
  - “What types of tests (unit, integration, golden‑file, evaluation harnesses) would you prioritize for this kind of troubleshooting copilot?”
  - “Is there a better way to automatically verify that our fault trees stay within the schema and remain grounded in the source docs?”
  - “How would you design automated checks for retrieval quality (e.g. that the right pages appear in top‑k for key queries)?”

---

## License and References

- Specs: `troubleshooting-copilot-workflow.md` (Phase 1), `troubleshooting-copilot-workflow.md.txt` (Phase 2).
- Full implementation plan: `implementation_plan.md`.






<!-- Command to run -->

<!-- uvicorn server:app --reload -->