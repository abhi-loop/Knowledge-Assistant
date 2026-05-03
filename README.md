# 🧠 Knowledge Assistant — AI-Powered Document Intelligence Platform

> Upload your documents. Ask questions. Get structured, exam-ready answers powered by LLMs.

Knowledge Assistant is a full-stack **Retrieval-Augmented Generation (RAG)** application. Users upload PDF or text documents, which are chunked, embedded, and stored in a FAISS vector index. When a question is asked, the most relevant chunks are retrieved and sent to a Hugging Face LLM to generate a clean, structured answer. The entire system is production-grade — with async task processing, Redis caching, JWT auth, and a full observability stack (ELK + Prometheus).

---

## ✨ Features

- 📄 **Document Upload** — Upload PDFs or plain-text files through a clean web UI
- ⚡ **Async Processing** — Celery + RabbitMQ handle chunking & embedding in the background
- 🔍 **Semantic Search** — FAISS vector search retrieves the most relevant chunks per query
- 🤖 **LLM-Powered Answers** — `Qwen/Qwen2.5-7B-Instruct` via Hugging Face Inference API generates structured responses
- 🧱 **Per-Document Isolation** — Each document has its own FAISS index; queries are always scoped to a single document
- ⚡ **Redis Caching** — Repeated questions are served instantly from cache (24h TTL)
- 🔐 **JWT Authentication** — Secure register/login flow with `bcrypt` password hashing
- 📊 **Observability** — Prometheus metrics exposed at `/metrics`; Logstash ships logs to Elasticsearch; Kibana for dashboards
- 🗑️ **Document Management** — Delete documents and their vector data cleanly from the UI

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser (Frontend)                      │
│              HTML + CSS + JS   (Vanilla, no framework)          │
└───────────────────────────┬─────────────────────────────────────┘
                            │ REST API
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FastAPI Backend  :8000                      │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ /auth/*    │  │  /rag/upload │  │  /rag/query            │  │
│  │ JWT Login  │  │  /rag/status │  │  /rag/documents/*      │  │
│  └────────────┘  └──────┬───────┘  └──────────┬─────────────┘  │
│                         │                     │                 │
│            Enqueue Task │              1. Redis Cache?          │
│                         ▼              2. FAISS Search          │
│              ┌─────────────────────┐   3. HF LLM Call          │
│              │  Celery Worker      │   4. Cache result          │
│              │  (RabbitMQ broker)  │                            │
│              │  - pdfplumber parse │                            │
│              │  - Chunker (1k/200) │                            │
│              │  - HF Embeddings    │                            │
│              │  - FAISS index      │                            │
│              └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
        │              │               │              │
        ▼              ▼               ▼              ▼
   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐
   │  Redis  │   │ RabbitMQ │   │  SQLite  │   │  FAISS Index │
   │  :6379  │   │  :5672   │   │  app.db  │   │  (per doc)   │
   └─────────┘   └──────────┘   └──────────┘   └──────────────┘

Observability Stack:
   Logstash :5000 (UDP) → Elasticsearch :9200 → Kibana :5601
   Prometheus :9090  ←  /metrics (FastAPI + Celery)
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | HTML5, CSS3, Vanilla JS, Marked.js |
| **Backend** | FastAPI, Python 3.11 |
| **Task Queue** | Celery 5, RabbitMQ |
| **Vector Store** | FAISS (`faiss-cpu`) |
| **Embeddings** | `sentence-transformers/all-MiniLM-L6-v2` (HF Inference API) |
| **LLM** | `Qwen/Qwen2.5-7B-Instruct` (HF Inference API) |
| **Caching** | Redis |
| **Database** | SQLite via SQLAlchemy |
| **Auth** | JWT (`python-jose`) + Bcrypt (`passlib`) |
| **PDF Parsing** | pdfplumber, PyMuPDF |
| **Logging** | Logstash (UDP) → Elasticsearch → Kibana |
| **Metrics** | Prometheus + `prometheus-fastapi-instrumentator` |
| **Containerization** | Docker, Docker Compose |

---

## 🚀 Getting Started

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- A [Hugging Face account](https://huggingface.co) with an API token that has **Inference API** access

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/knowledge-assistant.git
cd knowledge-assistant
```

### 2. Configure environment variables

Create `backend/.env`:

```env
HF_TOKEN=hf_your_token_here
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=amqp://guest:guest@rabbitmq:5672//
CELERY_RESULT_BACKEND=redis://redis:6379/0
JWT_SECRET=your_random_secret_here
```

> ⚠️ **Never commit your `.env` file.** It is already excluded in `.gitignore`.

Generate a secure `JWT_SECRET`:
```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

### 3. Build and run

```bash
docker-compose up --build
```

All services will start automatically. On first run, Docker will pull images and build the backend — this may take a few minutes.

### 4. Open the frontend

Serve the `frontend/` folder using a static file server. The easiest way is the **VS Code Live Server** extension — right-click `index.html` → *Open with Live Server*.

Alternatively, use Python:
```bash
cd frontend
python -m http.server 5500
```

Then visit: **http://localhost:5500**

---

## 📡 Service URLs

| Service | URL | Notes |
|---|---|---|
| **Backend API** | http://localhost:8000 | FastAPI |
| **API Docs** | http://localhost:8000/docs | Swagger UI |
| **Prometheus Metrics** | http://localhost:9090 | Scrapes `/metrics` |
| **Kibana** | http://localhost:5601 | Log dashboards |
| **RabbitMQ Dashboard** | http://localhost:15672 | `guest` / `guest` |
| **Elasticsearch** | http://localhost:9200 | Raw search API |

---

## 📁 Project Structure

```
├── backend/
│   ├── main.py                  # FastAPI app, CORS, Prometheus, Logstash handler
│   ├── auth_router.py           # /auth/register, /auth/login endpoints
│   ├── auth_models.py           # SQLAlchemy User model
│   ├── auth_dependencies.py     # JWT token verification dependency
│   ├── jwt_config.py            # Token creation/decoding
│   ├── password_utils.py        # bcrypt hashing
│   ├── database.py              # SQLAlchemy engine + session
│   ├── models.py                # Base models
│   ├── logging_config.py        # Console logging setup
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env                     # ← create this (not committed)
│   └── rag/
│       ├── celery_app.py        # Celery instance (RabbitMQ broker)
│       ├── celery_metrics.py    # Prometheus counters for Celery tasks
│       ├── chunker.py           # Sliding window text chunker
│       ├── db.py                # Document & Chunk SQLAlchemy models
│       ├── embeddings.py        # HF Inference API embedding calls
│       ├── llm.py               # HF Inference API LLM calls (Qwen)
│       ├── rag_router.py        # /rag/* API endpoints
│       ├── redis_client.py      # Redis connection
│       ├── tasks.py             # Celery task: parse → chunk → embed → index
│       ├── utils.py             # Cache key generation
│       └── vectorstore.py       # FAISS-based per-document vector store
├── frontend/
│   ├── index.html               # Single-page app (login, signup, chat, doc manager)
│   ├── styles.css               # Full dark-mode UI styles
│   └── script.js                # Auth, upload, polling, chat, document list logic
├── logstash/
│   └── pipeline/
│       └── pipeline.conf        # UDP → Elasticsearch pipeline
├── prometheus.yml               # Prometheus scrape config
├── docker-compose.yml           # Full stack orchestration
└── .gitignore
```

---

## 🔄 How It Works — End to End

```
1. User registers / logs in  →  JWT token stored in localStorage

2. User uploads a document  →  POST /rag/upload
   ├── FastAPI saves a Document record (SQLite)
   ├── Celery task queued via RabbitMQ
   └── Returns { task_id, doc_id }

3. Frontend polls  →  GET /rag/status/{task_id}
   └── Celery worker:
       ├── Extracts text (pdfplumber for PDF, UTF-8 for text)
       ├── Chunks text (1000 chars, 200 overlap)
       ├── Embeds each chunk  →  HF all-MiniLM-L6-v2 (384-dim)
       ├── Stores vectors  →  FAISS (per-doc index on disk)
       └── Saves chunks to SQLite

4. User selects a document + asks a question  →  POST /rag/query
   ├── Check Redis cache  →  cache hit? return instantly
   ├── Embed question  →  HF all-MiniLM-L6-v2
   ├── FAISS search (top 10 chunks for that doc_id)
   ├── Build context from chunk texts
   ├── Call Qwen2.5-7B-Instruct  →  structured answer
   ├── Cache answer in Redis (24h TTL)
   └── Return { answer, hits, source }
```

---

## 🧹 Stopping the Stack

```bash
docker-compose down          # stop containers
docker-compose down -v       # stop + delete volumes (Elasticsearch data etc.)
```

---

## 📝 License

This project is open source and available under the [MIT License](LICENSE).

---

*Built with FastAPI, Celery, FAISS, and Hugging Face Inference API.*
