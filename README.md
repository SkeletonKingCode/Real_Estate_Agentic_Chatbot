# Ali Real Estate — Evaluation Suite for Autonomous Agentic Chatbot

A fully local, CPU‑optimised conversational AI system built for a Pakistani property agency.  
No cloud APIs. No RAG. No tools. All intelligence from prompt design and context management.

This repository also includes a comprehensive automated evaluation suite that measures correctness, performance, and scalability of the chatbot.

---

## Contributors

- **Wajeeha Mahmood** — `23i-0105`
- **Muhammad Alyun Shah** — `23i-0022`
- **Awais Ali** — `23i-0080`

---

## Table of Contents

- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Setup & Run](#setup--run)
- [API Reference](#api-reference)
- [Conversation Flow](#conversation-flow)
- [Context & Memory Design](#context--memory-design)
- [Performance Benchmarks](#performance-benchmarks)
- [Evaluation Suite](#evaluation-suite)
- [Known Limitations](#known-limitations)
- [Video Demo](#video-demo)

---

## Architecture

```
Browser (index.html)
       │  WebSocket /ws/chat  &  REST /session
       ▼
FastAPI (backend/api/main.py)
       │  stream_response() async generator
       ▼
Conversation Manager (backend/Conversation/conversation.py)
  ├── Session store (in-memory dict, UUID keyed, 30-min TTL)
  ├── Stage machine  greeting → category_selection → subtype_selection → closing
  ├── State extraction  (selected_category, selected_subtype, selected_price)
  ├── Dynamic system prompt  CORE_IDENTITY + CONVERSATION STATE + stage hint
  └── Context window  sliding last-10-turns window
       │  ollama.AsyncClient.chat(..., stream=True)
       ▼
Ollama (local daemon, port 11434)
       │
       ▼
ali-realestate  (qwen3.5:2b, GGUF quantized, CPU inference)
```

---

## Project Structure

```
NLP_Assignment-3/
├── backend/
│   ├── api/
│   │   └── main.py                  # FastAPI + WebSocket server
│   ├── Conversation/
│   │   └── conversation.py          # Session mgmt, prompt orchestration, Ollama streaming
│   ├── Ollama/
│   │   ├── Modelfile                # Custom ali-realestate model definition
│   │   └── ModelCreation.sh         # ollama create + run commands
│   └── Voice/
│       ├── asr.py                   # ASR — faster-whisper speech-to-text
│       └── tts.py                   # TTS — piper text-to-speech
├── frontend/
│   └── index.html                   # ChatGPT-style web UI (single file, no build step)
├── tests/                           # Evaluation suite (Assignment 4)
│   ├── conftest.py                  # Shared fixtures (mocks Ollama/ASR/TTS)
│   ├── test_conversation.py         # Unit tests for conversation manager
│   ├── test_api.py                  # Integration tests for REST endpoints
│   ├── test_calculator.py           # Calculator tool tests
│   ├── test_weather.py              # Weather tool tests
│   ├── test_calendar.py             # Calendar tool tests
│   ├── test_crm.py                  # CRM tool tests
│   ├── test_orchestrator.py         # Tool orchestrator tests
│   ├── test_rag.py                  # RAG metrics tests
│   ├── test_conversational_correctness.py # E2E dialogue validation
│   ├── test_performance.py          # Latency benchmarks
│   ├── test_throughput.py           # Concurrency sweep
│   ├── test_llm_judge.py            # LLM-as-judge evaluation
│   └── test_data/                   # JSON fixtures for evaluations
│       ├── test_conversations.json
│       ├── rag_ground_truth.json
│       └── tool_invocation_test_set.json
├── voices/                          # Piper TTS voice models
│   ├── en_US-lessac-medium.onnx
│   └── en_US-lessac-medium.onnx.json
├── eval_results/                    # Generated evaluation reports
├── Dockerfile
├── docker-compose.yml
├── vercel.json                      # Vercel deployment config
├── requirements.txt
├── run.sh                           # Local one-command start script
├── run_evals.py                     # Master runner for evaluation suite
├── pyproject.toml                   # Pytest configuration
├── Ali_Chatbot.postman_collection.json
└── README.md
```

---

## Setup & Run

### Prerequisites

- Docker + Docker Compose  **or**  Python 3.10+ and Ollama installed locally
- Minimum 4 GB RAM (model is ~1.5 GB quantized)
- For evaluation suite: Python 3.10+ and the chatbot backend running (if using live tests)

### Option A — Docker Compose (recommended)

```bash
git clone <repo-url> && cd ali-realestate
docker compose up --build
```

On first start, Ollama will download `qwen3.5:2b` (~1.5 GB) and build the custom model.  
Open **http://localhost:8000** — wait for the API health check to return `"status": "ok"`.

### Option B — Local (no Docker)

```bash
# 1. Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 2. Create the custom model
ollama create ali-realestate -f backend/Ollama/Modelfile

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Start the API
python3 backend/api/main.py

# 5. Open the frontend
open http://0.0.0.0:8000   # or serve via any static file server
```

### Option C — Vercel (frontend only)

The chat UI is deployed as a static site on Vercel. The backend still runs locally.

```bash
# Install Vercel CLI (if not already installed)
npm i -g vercel

# Deploy from the project root
vercel --prod
```

> **Note:** The Vercel deployment serves only the frontend. Point the backend at your local machine or any server running the FastAPI + Ollama stack.

---

## Running the Evaluation Suite

The evaluation suite can be run independently of the live server for component tests, or against a running instance for end‑to‑end and performance tests.

### Installation (for evaluation)

```bash
# Install test dependencies
pip install pytest pytest-asyncio websockets httpx
```

### Running Tests

```bash
# Run all unit/component tests (NO server needed)
python run_evals.py --unit

# Run full suite (server + Ollama REQUIRED)
python run_evals.py --all

# Run only performance benchmarks
python run_evals.py --perf

# Run LLM-as-judge evaluation
python run_evals.py --judge

# Generate report from existing results
python run_evals.py --report-only

# Or use pytest directly
pytest tests/ -v                          # All tests
pytest tests/test_calculator.py -v        # Single component
pytest tests/ -k "not TestPerformance" -v # Skip perf tests
```

### Configuration (Environment Variables)

| Variable | Default | Description |
|----------|---------|-------------|
| `CHATBOT_BASE_URL` | `http://localhost:8000` | Chatbot REST API base URL |
| `CHATBOT_WS_URL` | `ws://localhost:8000/ws/chat` | WebSocket endpoint URL |
| `JUDGE_MODEL` | `ali-realestate` | Ollama model for LLM-as-judge |
| `PERF_NUM_TRIALS` | `30` | Number of trials per latency scenario |
| `MAX_TTFT` | `2.0` | Acceptable TTFT threshold (seconds) |
| `MAX_E2E` | `10.0` | Acceptable E2E threshold (seconds) |

---

## API Reference

### REST

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET`  | `/health` | Liveness probe, returns active WS connection count |
| `POST` | `/session` | Create a session → `{"session_id": "<uuid>"}` |
| `GET`  | `/session/{id}` | Get stage + selection state for a session |
| `DELETE` | `/session/{id}` | Delete a session immediately |

### WebSocket — `ws://host/ws/chat`

**Client → Server** (JSON)
```json
{ "session_id": "<uuid>", "message": "I want to buy a house" }
```

**Server → Client** (JSON frames)
```json
{ "type": "token",           "data": "Sure! Here" }          // streamed tokens
{ "type": "done",            "data": "" }                    // end of turn
{ "type": "state",           "data": { ...session_info } }  // updated session state
{ "type": "session_created", "data": "<new-uuid>" }         // auto-created session
{ "type": "error",           "data": "..." }                 // error message
```

---

## Conversation Flow

```
Greeting
  ↓  user mentions "house" / "shop" / "apartment"
Category Selection
  ↓  user mentions size ("10 marla", "1 bedroom", etc.)
Subtype Selection
  ↓  user says "schedule" / "visit" / "book"
Closing
```

### Authorised Inventory

| Category | Subtype | Price |
|----------|---------|-------|
| Shops | 5 Marla | PKR 1.2 Crore |
| Shops | 8 Marla | PKR 2.1 Crore |
| Shops | 1 Kanal | PKR 3.8 Crore |
| Houses/Villas | 5 Marla | PKR 1.8 Crore |
| Houses/Villas | 7 Marla | PKR 2.6 Crore |
| Houses/Villas | 10 Marla | PKR 4.2 Crore |
| Houses/Villas | 1 Kanal Villa | PKR 8.5 Crore |
| Apartments | 1 Bedroom | PKR 55 Lac |
| Apartments | 2 Bedroom | PKR 95 Lac |
| Apartments | 3 Bedroom | PKR 1.5 Crore |

---

## Context & Memory Design

### Problem
Small 2B models cannot reliably re-infer user choices from raw history alone — especially after an off‑topic detour or after the context window trims old turns.

### Solution: Explicit State Injection

Every turn, the system prompt includes a `CONVERSATION STATE` block:

```
CONVERSATION STATE  (tracked by the system — treat as ground truth)
--------------------------------------------------------------------
Stage             : subtype_selection
Category chosen   : Houses/Villas
Subtype chosen    : 10 Marla House
Price confirmed   : PKR 4.2 Crore
--------------------------------------------------------------------
IMPORTANT: Do NOT ask the customer again about choices already made above.
```

This is computed deterministically in Python from keyword matching — the model never has to infer it. The context window slides over the last 10 turn‑pairs; no greeting‑pinning is used (it caused the model to re‑ask already‑answered questions).

---

## Performance Benchmarks

> Measured on: Intel Core i7-12th Gen, 16 GB RAM, no GPU.

| Metric | Value |
|--------|-------|
| Model | qwen3.5:2b (Q4_K_M GGUF) |
| Time to first token (TTFT) | ~1.8 s |
| Token throughput | ~12 tok/s |
| Peak RAM usage | ~2.1 GB |
| Concurrent sessions tested | 5 (sequential WS connections) |
| Session TTL | 30 minutes |
| Context window | Last 10 turn-pairs |

> Note: Benchmarks are approximate. Results vary by hardware and model quantization level.

---

## Evaluation Suite

The evaluation suite systematically measures the chatbot’s correctness, performance, and scalability across all major components.

### What is Evaluated

| Dimension | Components | Metrics |
|-----------|-----------|---------|
| **Conversational Correctness** | Full dialogues | Task completion, policy adherence, coherence |
| **RAG Component** | Retrieval pipeline | Precision@k, Recall@k, context relevance, faithfulness |
| **CRM Tool** | CRUD operations | Data correctness, persistence, semantic matching |
| **Calculator Tool** | Math evaluation | Functional correctness, error handling, security |
| **Weather Tool** | API integration | Valid/invalid inputs, timeout handling |
| **Calendar Tool** | Event management | CRUD, date filtering, ordering |
| **Tool Orchestrator** | JSON parsing | Tool call extraction, execution, caching |
| **Latency** | WebSocket streaming | TTFT, inter‑token latency, E2E |
| **Throughput** | Concurrency | Max users, breakpoint, turns/sec |

### How Metrics Are Computed

#### RAG Metrics

| Metric | Formula | Description |
|--------|---------|-------------|
| **Precision@k** | `relevant_in_top_k / k` | Fraction of retrieved chunks that are relevant |
| **Recall@k** | `relevant_in_top_k / total_relevant` | Fraction of relevant docs successfully retrieved |
| **Context Relevance** | `useful_chunks / total_chunks` | Proportion of chunks containing expected keywords |
| **Faithfulness** | `matched_keywords / expected_keywords` | Keyword overlap between context and ground truth |

Relevance is determined by checking if retrieved chunks contain content from annotated source documents or expected keywords from `rag_ground_truth.json`.

#### Latency Metrics

| Metric | Description |
|--------|-------------|
| **TTFT** | Time from sending message to receiving first token (via WebSocket) |
| **ITL** | Average time between consecutive tokens |
| **E2E** | Time from sending message to receiving last token |

Each metric reports **mean, median, P90, P99** across 30+ trials per scenario.

#### Throughput Metrics

| Metric | Description |
|--------|-------------|
| **Max Sustainable Concurrency** | Highest concurrent users where median TTFT < 2s AND median E2E < 10s |
| **Breakpoint** | Concurrency level where latency exceeds thresholds |
| **Turns/sec** | Total turns completed per second at sustainable concurrency |

#### LLM-as-Judge Metrics

The judge LLM evaluates using structured rubrics and scores on a 0–1 scale:

| Metric | What It Measures |
|--------|-----------------|
| **Task Completion** | Did the bot fulfil the user's request? |
| **Policy Adherence** | Did the bot refuse off‑topic requests appropriately? |
| **Coherence** | Does the bot remember context and avoid contradictions? |
| **Faithfulness** | Is the answer grounded in retrieved documents? |

### Interpreting Results

#### What is "Good" Performance?

| Metric | Good | Acceptable | Poor |
|--------|------|------------|------|
| Precision@3 | > 0.7 | 0.4–0.7 | < 0.4 |
| Recall@3 | > 0.6 | 0.3–0.6 | < 0.3 |
| Context Relevance | > 0.7 | 0.4–0.7 | < 0.4 |
| Faithfulness | > 0.8 | 0.5–0.8 | < 0.5 |
| Task Completion (judge) | > 0.8 | 0.5–0.8 | < 0.5 |
| TTFT (simple) | < 1.0s | 1.0–2.0s | > 2.0s |
| E2E (simple) | < 5.0s | 5.0–10.0s | > 10.0s |
| Sustainable concurrency | > 5 users | 2–5 users | < 2 users |

---

## Known Limitations

- **Single process, in‑memory sessions** — sessions are lost on restart. For production, replace `_sessions` dict with Redis.
- **Keyword‑based stage machine** — complex phrasings ("I'd fancy something about 1000 sq ft") may not trigger transitions correctly. A small intent classifier would improve robustness.
- **English only** — the Modelfile and stage logic assume English input. Urdu/Roman Urdu support requires prompt additions.
- **CPU latency** — first token takes ~2 s on a laptop CPU. A GPU or Apple Silicon chip reduces this to <0.5 s.
- **Single worker** — `--workers 1` in uvicorn ensures the in‑memory session store is consistent. Scaling to multiple workers requires an external session store.
- **Model limitations** — The 2B model may hallucinate JSON syntax in long prompts, affecting tool call accuracy.
- **External APIs** — Weather tool tests depend on `wttr.in` availability (mocked in unit tests).
- **LLM Judge bias** — The judge uses the same model family; ideally a larger/different model should be used.
- **RAG index required** — RAG evaluation tests require the index to be pre‑built via `python backend/RAG/indexer.py`.
- **Single LLM thread** — Throughput is bottlenecked by Ollama’s sequential inference.

---

## Video Demo

**[YouTube Link]** — [Watch on YouTube](https://youtu.be)

---
