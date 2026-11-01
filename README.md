# Secure RAG Assistant

[![Tests](https://github.com/ddunmeyer/secure-rag-assistant/actions/workflows/tests.yml/badge.svg)](https://github.com/ddunmeyer/secure-rag-assistant/actions/workflows/tests.yml)

A retrieval-augmented generation (RAG) assistant built with security controls at every pipeline boundary. Questions are validated before they reach the model, sensitive data is tagged and redacted, weak retrieval is filtered out, and answers are checked for hallucination before they are shown.

Built with Python, LangChain, ChromaDB, SentenceTransformers and Google Gemini.

## Security and reliability controls

- **Input defense:** validation, sanitization and prompt-injection blocking before any LLM call, failing closed by default.
- **Sensitive data handling:** PII, PHI and credential patterns are blocked at input and redacted from logs and error output.
- **Data classification:** documents, user input, retrieved chunks and model output are tagged by sensitivity, data type and source.
- **Retrieval quality:** query rewriting, multi-hop retrieval, similarity filtering and safe fallbacks.
- **Output monitoring:** hallucination checks, with sources and confidence shown alongside each answer.
- **Automated safety tests:** 16 pytest tests run in GitHub Actions on every push, with no live LLM calls.

## What This App Does

You can ask this app questions about Python, machine learning, databases, APIs, and AI concepts. It finds the most relevant documents from its knowledge base and sends them to Gemini as context — so the answers are grounded in real information rather than guesswork.

## System Architecture

See **[ARCHITECTURE.md](ARCHITECTURE.md)** for the full diagram and component explanations.

![Architecture Diagram](docs/rag-app-architecture-diagram.png)

**Quick reference — query flow:**

```
User Query
    │
    ▼
[security.py]      ← Validate and sanitize input
    │
    ▼
[compliance.py]    ← Tag metadata and redact sensitive data
    │
    ▼
[workflow.py]      ← Rewrite query for better retrieval
    │
    ▼
[embeddings.py]    ← Convert query to a vector
    │
    ▼
[vector_store.py]  ← Find similar document vectors in ChromaDB
    │
    ▼
[filters.py]       ← Remove irrelevant results
    │
    ▼
[rag_pipeline.py]  ← Build prompt with retrieved context
    │
    ▼
  Gemini API       ← Generate answer
    │
    ▼
[monitoring.py]    ← Check for hallucinations
    │
    ▼
[app.py]           ← Display answer, sources, confidence
```

## Setup

### 1. Clone the repository
```bash
git clone https://github.com/ddunmeyer/secure-rag-assistant.git
cd secure-rag-assistant
```

### 2. Create a virtual environment
```bash
python -m venv venv
```

Activate it:
- **Mac/Linux:** `source venv/bin/activate`
- **Windows:** `venv\Scripts\activate`

### 3. Install dependencies
```bash
pip install -r requirements.txt
```

### 4. Set up your Gemini API key

Copy the example environment file:
```bash
cp .env.example .env
```

Open `.env` and replace `your-gemini-api-key-here` with your actual key.
Get a free key at: https://aistudio.google.com/apikey

### 5. Run the app
```bash
streamlit run app.py
```

The app opens in your browser at `http://localhost:8501`.

---

## File Descriptions

| File | Purpose |
|------|---------|
| `app.py` | Streamlit web interface |
| `config.py` | Configuration constants |
| `embeddings.py` | Convert text to vector embeddings |
| `vector_store.py` | Store and search vectors with ChromaDB |
| `data_loader.py` | Sample tech documents |
| `rag_pipeline.py` | Central orchestration — ties everything together |
| `langchain_engine.py` | LangChain engine (Chroma, embeddings, Gemini, LCEL / ReAct) |
| `conversation.py` | Conversation history |
| `security.py` | Input validation, injection defense, and data protection |
| `compliance.py` | Metadata tagging and log redaction for sensitive data |
| `tests/` | Automated unit and safety tests |
| `.github/workflows/tests.yml` | GitHub Actions CI — runs pytest on push/PR |
| `monitoring.py` | Hallucination detection |
| `filters.py` | Similarity filtering and fallbacks |
| `workflow.py` | Query rewriting and multi-hop retrieval |
| `langchain_demo.py` | CLI test runner for the LangChain-backed `run_rag()` |
| `LANGCHAIN.md` | LangChain architecture and configuration |
| `ARCHITECTURE.md` | High-level architecture diagram and explanation |
| `docs/` | Architecture diagram PNG and editable Excalidraw file |

### LangChain pipeline

LangChain powers the main app (`langchain_engine.py` + `rag_pipeline.py`).

```bash
pip install -r requirements.txt
streamlit run app.py
```

Optional CLI test:

```bash
python langchain_demo.py "What is Python?"
```

Set `RAG_MODE=react` in `.env` for the ReAct agent. See [LANGCHAIN.md](LANGCHAIN.md).

---

## Compliance (Metadata Tagging & Redaction)

This section documents how the app handles sensitive data responsibly. This is **compliance-aware design**, not a SOC 2 certification.

### Applicable trust principles (hypothetical use case)

Our RAG app answers questions about public tech documentation. If deployed commercially, these SOC 2-style principles would apply:

| Principle | Why it applies |
|-----------|----------------|
| **Security** | User queries reach an external LLM API and are embedded into a vector store. Access to API keys, session chat history, and ChromaDB must be protected. |
| **Confidentiality** | Users might paste emails, account numbers, or API keys into chat even though the app is for tech Q&A. Those values must not appear in logs or debug output. |
| **Privacy** | Personal identifiers (PII) and health-related text (PHI) should be blocked at input and never stored in analytics or error traces. |

### Where sensitive data could appear

| Location | Risk | Control |
|----------|------|---------|
| **User chat input** | PII, secrets, PHI pasted into the prompt | `security.py` blocks known patterns; `compliance.py` tags and redacts before logging |
| **Knowledge-base documents** | Hypothetically could contain internal/confidential docs | Sample corpus is public educational text; ingest tags each chunk as `public` / `operational` |
| **Retrieved vector chunks** | Could surface sensitive text if corpus were compromised | Each retrieved chunk re-tagged at retrieval time |
| **Model output** | LLM might echo sensitive input | Output tagged; logs use redacted text only |
| **Error messages** | Exceptions may include user text | `filters.handle_api_error()` redacts before display |
| **Session state / chat UI** | Conversation stored in Streamlit session | User sees their own input; compliance metadata shown without raw log dumps |

### Metadata tagging scheme

Every tagged item carries three dimensions:

| Field | Values | Purpose |
|-------|--------|---------|
| `sensitivity` | `public`, `internal`, `confidential`, `restricted` | Drives handling strictness |
| `data_type` | `operational`, `pii`, `phi`, `financial`, `credential` | Describes content category |
| `source` | `user_input`, `document`, `retrieved`, `model_output` | Shows pipeline boundary |

Tags are attached in `compliance.py` and stored:

- **Document ingest** — Chroma metadata on each chunk (`langchain_engine.py`)
- **User input** — when a query passes validation (`rag_pipeline.py`)
- **Retrieved chunks** — after similarity filtering (`rag_pipeline.py`)
- **Model output** — before returning the answer (`rag_pipeline.py`)

### Automated redaction

Redaction masks emails, phone numbers, SSN-like patterns, API keys, and selected financial/PHI phrases with `[REDACTED]`.

Redaction is applied at:

1. **Compliance audit logs** — `log_compliance_event()` never writes raw sensitive text
2. **Error / debug output** — `filters.handle_api_error()` and `redact_for_display()`
3. **Input boundary** — `security.py` blocks sensitive user input before it reaches Gemini (fail-closed default)

Set `ENABLE_COMPLIANCE_LOGGING=true` in `.env` to emit redacted audit events to the console.

### Assumptions and limitations

- Pattern matching is **not** full data-loss prevention; it catches common formats only.
- The knowledge base is static sample text — no real customer or employee records.
- Metadata classification is rule-based, not ML-based.
- Streamlit session memory is in-process only; production would need encrypted persistence policies.
- We document controls for learning purposes; formal SOC 2 audits require organizational process beyond code.

---

## Testing & CI/CD

### What is tested

| Test file | Covers |
|-----------|--------|
| `tests/test_basic.py` | Redaction, metadata tagging, input validation, filters, conversation formatting |
| `tests/test_safety.py` | PII/injection blocking in `run_rag()`, redacted error messages |

These tests run **without live Gemini API calls** — safety checks fail fast before the LLM is invoked.

### Why these tests matter

- **Redaction/compliance** — ensures emails and API-key patterns never appear in logs or error output
- **Input security** — confirms injection and PII blocking still work after refactors
- **Filters & conversation** — guards core RAG helper behavior used every query

### What is intentionally not tested

- Live Gemini generation (requires API key, costs quota, non-deterministic)
- Streamlit UI rendering (manual testing via `streamlit run app.py`)
- Full vector-store embedding quality (slow; depends on downloaded models)

### Run tests locally

```bash
pip install -r requirements.txt
python -m pytest
```

### Continuous integration

GitHub Actions workflow `.github/workflows/tests.yml` runs `python -m pytest` on every push and pull request.
