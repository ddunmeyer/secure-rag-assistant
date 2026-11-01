# RAG Application Architecture

High-level architecture diagram for the Secure RAG Assistant.

![Architecture Diagram](docs/rag-app-architecture-diagram.png)

**Editable source:** [docs/rag-app-architecture-diagram.excalidraw](docs/rag-app-architecture-diagram.excalidraw) — open at [excalidraw.com](https://excalidraw.com) (Arial/Helvetica font, captions on each node).

---

## Node guide (brief)

| Node | What it does |
|------|----------------|
| **User (Browser)** | Sends questions; reads answers, sources, and confidence. |
| **Streamlit UI (`app.py`)** | Chat interface, session state, sidebar status panel. |
| **RAG Pipeline (`rag_pipeline.py`)** | Orchestrates startup indexing and every query via `run_rag()`. |
| **`security.py`** | Validates input; blocks injection & sensitive data; sanitizes text. |
| **`conversation.py`** | Keeps chat history for follow-up questions. |
| **`workflow.py`** | Rewrites vague or follow-up queries before retrieval. |
| **`embeddings.py`** | Turns text into vectors (SentenceTransformers). |
| **`vector_store.py`** | Searches ChromaDB for similar documents. |
| **`filters.py`** | Removes weak matches; fallback when nothing is relevant. |
| **`gemini_utils.py`** | Calls Gemini with automatic retry on rate limits. |
| **`generate_answer()`** | Builds prompt with docs + history; requests LLM answer. |
| **`monitoring.py`** | Confidence score; optional hallucination grounding check. |
| **`data_loader.py`** | Loads source documents into the knowledge base. |
| **ChromaDB** | Local vector database for embedded document chunks. |
| **Google Gemini API** | Generates answers from retrieved context. |
| **`config.py` / `.env`** | Model settings, thresholds, API keys, feature flags. |
| **`langchain_demo.py`** | Optional LCEL chain + ReAct agent comparison demo. |

---

## Overview

This application uses **Retrieval-Augmented Generation (RAG)**. Instead of asking the LLM to answer from memory alone, the system first retrieves relevant documents from a knowledge base, then sends those documents as context to Gemini.

The system has two main flows:

1. **Offline indexing** — runs once at startup to load and vectorize documents
2. **Online query pipeline** — runs on every user question

---

## Components

### User interface

| Component | File | Role |
|-----------|------|------|
| **User (Browser)** | — | Sends questions and reads answers in the Streamlit web UI |
| **Streamlit UI** | `app.py` | Chat interface, session state, sidebar status, displays answers/sources/confidence |

### Configuration

| Component | File | Role |
|-----------|------|------|
| **Config** | `config.py` | API keys, model name, embedding model, thresholds, feature flags |

### Offline indexing (startup)

| Component | File | Role |
|-----------|------|------|
| **Source documents** | `data_loader.py` | Sample tech docs about Python, ML, RAG, etc. |
| **Embeddings** | `embeddings.py` | Converts text to vectors using SentenceTransformers (`all-MiniLM-L6-v2`) |
| **Vector store** | `vector_store.py` | Stores and searches vectors in **ChromaDB** |

At startup, `initialize_vector_store()` in `rag_pipeline.py` loads documents, embeds them, and persists vectors to ChromaDB.

### Online query pipeline

All query steps are orchestrated by **`rag_pipeline.py`** via `run_rag()`:

| Step | File | Role |
|------|------|------|
| 1. Security | `security.py` | Injection defense, PII/secret checks, sanitize input |
| 2. Memory | `conversation.py` | Include prior chat turns for follow-up questions |
| 3. Query rewrite | `workflow.py` | Rewrite vague/follow-up queries before retrieval (optional) |
| 4. Embed query | `embeddings.py` | Turn the search query into a vector |
| 5. Retrieve | `vector_store.py` | Find top-k similar documents in ChromaDB |
| 6. Filter | `filters.py` | Drop low-similarity results; return fallback if nothing matches |
| 7. Generate | `rag_pipeline.py` + `gemini_utils.py` | Build prompt with context; call **Gemini API** |
| 8. Monitor | `monitoring.py` | Compute confidence score; optional hallucination check |

### Optional LangChain demo

| Component | File | Role |
|-----------|------|------|
| **LangChain demo** | `langchain_demo.py` | Side-by-side LCEL chain and ReAct agent examples (Week 15.5) |

---

## Data flow

### Indexing flow

```
data_loader.py  →  embeddings.py  →  vector_store.py  →  ChromaDB
   (text docs)      (vectors)           (persist)
```

### Query flow

```
User question
  → app.py (Streamlit)
  → rag_pipeline.run_rag()
      → security.py (validate)
      → conversation.py (history context)
      → workflow.py (rewrite query, if needed)
      → embeddings.py (embed query)
      → vector_store.py (retrieve from ChromaDB)
      → filters.py (threshold + fallback)
      → Gemini API (generate answer)
      → monitoring.py (confidence / grounding)
  → app.py (display answer, sources, confidence)
  → conversation.py (save turn for follow-ups)
```

---

## External services

- **Google Gemini API** — generates answers from retrieved context
- **ChromaDB** — local vector database (in-memory / on disk)
- **SentenceTransformers** — local embedding model (no API call)

---

## Design notes

- **Separation of concerns:** Each weekly module (`security.py`, `filters.py`, etc.) handles one responsibility; `rag_pipeline.py` wires them together.
- **Graceful degradation:** Filtering, fallbacks, and API error handling prevent bad answers when retrieval fails or rate limits hit.
- **Configurable cost:** Query rewriting and hallucination checks can be toggled in `.env` to reduce Gemini API calls on the free tier.
