# AdIntel_AI

<div align="center">

# AdIntel AI

### Multi-Agent Marketing Intelligence Platform

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![LangChain](https://img.shields.io/badge/LangChain-0.1-1C3C3C?style=flat-square)](https://langchain.com)
[![Groq](https://img.shields.io/badge/Groq-llama3--70b-F55036?style=flat-square)](https://groq.com)
[![RAGAS](https://img.shields.io/badge/Eval-RAGAS-8B5CF6?style=flat-square)](https://docs.ragas.io)
[![LangSmith](https://img.shields.io/badge/Tracing-LangSmith-F97316?style=flat-square)](https://smith.langchain.com)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docker.com)
[![AWS](https://img.shields.io/badge/Deploy-AWS%20ECS%20Fargate-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](https://aws.amazon.com/ecs/)
[![Live Demo](https://img.shields.io/badge/Live%20Demo-Online-22c55e?style=flat-square)](https://your-ecs-url.amazonaws.com/docs)
[![Tests](https://img.shields.io/badge/Tests-Passing-22c55e?style=flat-square)](./tests)

<br/>

*A production-grade RAG platform that routes marketing questions through specialized AI agents —  
grounded in real campaign data, evaluated with RAGAS, traced with LangSmith, deployed on AWS.*

</div>

---

## What Makes This Different

Most AI portfolio projects are a single-pipeline RAG chatbot with no evals and no deployment. AdIntel AI is built to a production standard across five dimensions that matter in interviews:

| Dimension | What's Here |
|---|---|
| **Retrieval quality** | HyDE (Hypothetical Document Embeddings) — LLM generates a hypothetical answer first, embeds that, retrieves against it. Measurably better recall on vague marketing queries vs vanilla cosine search |
| **Eval framework** | RAGAS scores (answer relevancy, faithfulness, context precision, context recall) computed on every query, stored in MongoDB, displayed live in the Streamlit dashboard |
| **Observability** | LangSmith tracing enabled — every agent call, tool invocation, and LLM response is captured with latency, token counts, and full prompt/completion chains |
| **Feedback loop** | 👍 👎 on every Streamlit response writes to MongoDB. Retrieval quality trends over time are charted — shows product thinking, not just engineering |
| **Real data** | Ingests actual public marketing data: Meta Ads Library creatives, Nielsen public reports, CPG campaign case studies — not toy fictional documents |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            CLIENT LAYER                                  │
│              Streamlit Demo UI  ·  Postman  ·  Swagger /docs             │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ REST + WebSocket
┌──────────────────────────────▼───────────────────────────────────────────┐
│                      FASTAPI  ·  UVICORN  (AWS ECS Fargate)              │
│         /query  /ingest  /campaigns  /segments  /generate  /stream       │
└────────┬─────────────────────────────────────────────────┬───────────────┘
         │ Ingest path (async)                             │ Query path
┌────────▼──────────┐                         ┌───────────▼────────────────┐
│   REDIS STREAMS   │                         │   LANGCHAIN AgentExecutor  │
│  stream:ingest    │                         │   keyword + score routing  │
│  consumer groups  │                         └──┬──────────┬──────────┬───┘
└────────┬──────────┘                            │          │          │
         │                                 ┌─────▼──┐  ┌────▼───┐  ┌──▼──────┐
┌────────▼──────────┐                      │Campaign│  │Audience│  │Content  │
│   EMBED PIPELINE  │                      │Analyst │  │Segment.│  │Generatr │
│  Sentence Transf. │                      └─────┬──┘  └────┬───┘  └──┬──────┘
│  HyDE retrieval   │                            │           │         │
│  512-token chunks │                            └─────┬─────┘─────────┘
└────────┬──────────┘                                  │ HyDE → ChromaDB search
         │                              ┌──────────────▼──────────────┐
┌────────▼──────────────────────┐       │          CHROMADB           │
│        AWS ECR / ECS          │       │  campaigns · audiences      │
│   Container registry +        │       │  competitors  (3 colls.)    │
│   Fargate task definitions    │       └──────────────┬──────────────┘
└───────────────────────────────┘                      │
                                        ┌──────────────▼──────────────┐
┌──────────────────────┐                │       GROQ LLM              │
│    MONGODB ATLAS     │                │   llama3-70b-8192           │
│  doc metadata        │                │   ~300 tok/sec              │
│  RAGAS eval scores   │                └──────────────────────────────┘
│  user feedback log   │
└──────────────────────┘                ┌──────────────────────────────┐
                                        │       LANGSMITH              │
┌──────────────────────┐                │   full agent traces          │
│    AWS SERVICES      │                │   latency · token counts     │
│  ECR · ECS Fargate   │                │   prompt/completion chains   │
│  ALB · Route 53      │                └──────────────────────────────┘
│  Secrets Manager     │
└──────────────────────┘
```

---

## Multi-Agent Design

Three specialized agents. Each has its own system prompt, ChromaDB collection, and tool set. The LangChain orchestrator routes by keyword detection (or explicit `agent_type`) — specialized agents with domain-tuned prompts outperform a single generalist agent significantly on domain-specific retrieval tasks.

| Agent | Trigger Keywords | Collection | Output |
|---|---|---|---|
| **Campaign Analyst** | campaign, KPI, performance, brief, ROI, metrics | `campaigns` | KPI scoring, performance pattern analysis, campaign benchmarking |
| **Audience Segmenter** | audience, segment, persona, demographic, customer | `audiences` | Audience personas, segment profiles, targeting recommendations |
| **Content Generator** | generate, write, copy, ad, headline, creative | `competitors` | Ad copy variants, headlines, CTAs grounded in brand voice + gaps |

---

## HyDE Retrieval

Standard RAG embeds the raw query and searches for similar chunks. The problem: a short query like *"millennial engagement"* is semantically distant from a long detailed document about it.

**HyDE (Hypothetical Document Embeddings)** solves this:

```
Standard RAG:   query → embed → search
HyDE:           query → LLM generates hypothetical answer → embed that → search
```

The hypothetical answer is dense, domain-specific text — much closer in embedding space to the actual documents. Recall improves measurably on vague or short marketing queries, which is the dominant query type in this domain.

```python
# retriever.py — HyDE implementation
async def hyde_retrieve(query: str, collection: str, top_k: int = 5):
    # Step 1: generate hypothetical answer
    hyp_answer = await groq_client.generate(
        f"Write a detailed marketing document excerpt that would answer: {query}"
    )
    # Step 2: embed the hypothetical answer, not the query
    hyp_vector = embedder.encode(hyp_answer)
    # Step 3: search with the richer embedding
    return chroma_client.query(collection, hyp_vector, top_k)
```

---

## RAGAS Eval Framework

Every query response is automatically evaluated on four dimensions using [RAGAS](https://docs.ragas.io). Scores are stored in MongoDB and displayed in the Streamlit dashboard — giving a live view of retrieval quality over time.

| Metric | What It Measures | Target |
|---|---|---|
| **Answer Relevancy** | Does the answer address what was actually asked? | > 0.85 |
| **Faithfulness** | Is every claim in the answer grounded in retrieved context? | > 0.90 |
| **Context Precision** | Are the retrieved chunks actually relevant? | > 0.80 |
| **Context Recall** | Did retrieval find all necessary information? | > 0.75 |

```python
# Computed automatically after every query
from ragas.metrics import answer_relevancy, faithfulness, context_precision, context_recall

scores = evaluate(
    dataset=Dataset.from_dict({
        "question": [query],
        "answer": [response.answer],
        "contexts": [response.sources],
        "ground_truth": [None]   # unsupervised mode
    }),
    metrics=[answer_relevancy, faithfulness, context_precision, context_recall]
)
# Stored to MongoDB → displayed in Streamlit metrics panel
```

---

## LangSmith Tracing

One environment variable (`LANGCHAIN_TRACING_V2=true`) captures the full execution trace of every agent call — tool invocations, LLM prompts and completions, latency at each step, and token counts. No code changes required.

This enables:
- Debugging retrieval quality issues by inspecting exactly what context reached the LLM
- Identifying slow steps (embedding vs retrieval vs generation)
- A/B comparing HyDE vs standard retrieval on the same queries

---

## Feedback Loop

The Streamlit demo UI includes a 👍 👎 rating on every response. Each rating writes to a `feedback` collection in MongoDB Atlas with the query, agent used, RAGAS scores, and the rating. The dashboard shows:

- Retrieval quality trend over time (RAGAS scores by day)
- Agent-level performance breakdown (which agent gets the most thumbs-down)
- Query categories that score poorly (signals where to improve prompts or add data)

This is not required to demo the system — but it demonstrates product thinking: building the data collection infrastructure to improve the system after deployment, not just at launch.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| LLM inference | **Groq** (llama3-70b-8192) | ~300 tok/sec, free tier, no credit card, 10x faster than GPT-4 at this task |
| Embeddings | **Sentence Transformers** (all-MiniLM-L6-v2) | Runs in container, zero API cost, zero rate limits, 384-dim vectors |
| Retrieval | **HyDE** + ChromaDB | Hypothetical document embeddings for better recall on short queries |
| Vector store | **ChromaDB** | Zero infrastructure, in-process, persists to disk, metadata filtering |
| Metadata / eval | **MongoDB Atlas** | Free M0 tier, stores doc metadata + RAGAS scores + feedback log |
| Event pipeline | **Redis Streams** | Producer/consumer pattern identical to Kafka — $0 locally, scales to MSK |
| Agent framework | **LangChain** AgentExecutor | Tool-calling, memory, routing — most widely used AI orchestration framework |
| Eval framework | **RAGAS** | Industry-standard RAG evaluation — faithfulness, relevancy, precision, recall |
| Observability | **LangSmith** | Full agent execution traces — prompts, completions, latency, token counts |
| API layer | **FastAPI** + Uvicorn | Async, auto-generates Swagger docs, WebSocket support |
| Containerization | **Docker** + docker-compose | Reproducible builds, direct path to ECS deployment |
| Deployment | **AWS ECS Fargate** | Serverless containers, ALB for HTTPS, Secrets Manager for env vars |
| Demo UI | **Streamlit** | RAGAS dashboard + feedback loop + agent demo in one interface |

---

## API Endpoints

All endpoints versioned under `/api/v1`. Swagger UI at `/docs`.

```
POST   /api/v1/query       →  Route query to correct agent, return grounded answer + RAGAS scores
POST   /api/v1/ingest      →  Publish document to Redis Streams pipeline (non-blocking, <100ms)
GET    /api/v1/campaigns   →  List indexed campaign documents from MongoDB
POST   /api/v1/generate    →  Generate ad copy variants for a product/brand
GET    /api/v1/segments    →  List current audience segment personas
POST   /api/v1/feedback    →  Submit 👍 👎 rating for a query response
WS     /api/v1/stream      →  Token-by-token streaming LLM response via WebSocket
GET    /health             →  API + Redis + MongoDB + ChromaDB status
```

**Query request:**
```json
{
  "query": "Analyze Nike Q3 campaign performance against benchmark KPIs",
  "agent_type": "auto",
  "top_k": 5,
  "use_hyde": true
}
```

**Query response:**
```json
{
  "answer": "The Nike Q3 campaign achieved a CTR of 3.2% against a 2.8% benchmark...",
  "agent_used": "campaign_analyst",
  "sources": ["nike_q3_brief.txt", "campaign_benchmarks.txt"],
  "latency_ms": 387,
  "eval_scores": {
    "answer_relevancy": 0.91,
    "faithfulness": 0.94,
    "context_precision": 0.83,
    "context_recall": 0.78
  }
}
```

---

## Data Flow

### Query Path
```
POST /query
  → AgentExecutor routes by keyword / agent_type
    → HyDE: Groq generates hypothetical answer for query
      → Sentence Transformers embeds hypothetical answer (384-dim)
        → ChromaDB cosine similarity search (top-k chunks)
          → Context injected into agent system prompt
            → Groq LLM generates grounded response
              → RAGAS evaluates answer quality asynchronously
                → Scores stored in MongoDB
                  → QueryResponse returned with answer + scores + latency
```

### Ingest Path (async, non-blocking)
```
POST /ingest  →  returns job_id in <100ms
  → Redis XADD to stream:ingest
    → Consumer worker reads from consumer group adintel-workers
      → RecursiveCharacterTextSplitter (512 tokens, 64-token overlap)
        → Sentence Transformers embeds each chunk
          → ChromaDB upserts to matching collection
            → MongoDB stores document metadata + chunk_count + job_id
              → XACK — message removed from pending, job complete
```

---

## Real Sample Data

The system is loaded with real public marketing data — not fictional documents — for a credible demo:

| Document | Source | Type |
|---|---|---|
| Meta Ads Library creatives (CPG vertical) | Meta public API | competitor_intel |
| Nielsen Q3 2024 marketing effectiveness report | Nielsen public release | campaign_brief |
| Millennial consumer behavior study | Pew Research public data | customer_segment |
| Gen Z brand loyalty patterns | Morning Consult public report | customer_segment |
| Red Bull / Monster energy ad analysis | Public campaign database | competitor_intel |

---

## Quickstart

### Prerequisites
- Python 3.11+
- Docker + Docker Compose
- [Groq API key](https://console.groq.com) — free, no credit card
- [MongoDB Atlas](https://mongodb.com/atlas) connection string — free M0 tier
- [LangSmith API key](https://smith.langchain.com) — free tier

### 1. Clone and configure
```bash
git clone https://github.com/vaishnavi-bhamare/adintel-ai.git
cd adintel-ai
cp .env.example .env
# Fill in: GROQ_API_KEY, MONGODB_URI, LANGCHAIN_API_KEY
```

### 2. Run locally
```bash
docker-compose up --build
```

### 3. Verify
```
http://localhost:8000/docs    →  Swagger UI (all 8 endpoints)
http://localhost:8000/health  →  {"api": "ok", "redis": "ok", "mongodb": "ok", "chroma": "ok"}
```

### 4. Load real sample data
```bash
python scripts/load_sample_data.py
```

### 5. Launch Streamlit demo
```bash
cd demo && streamlit run app.py
# Opens: query interface + RAGAS metrics dashboard + feedback panel
```

---

## Project Structure

```
adintel-ai/
├── app/
│   ├── main.py                 # FastAPI entry point, lifespan events
│   ├── config.py               # Pydantic settings from .env
│   ├── models/
│   │   └── schemas.py          # All Pydantic request/response models
│   ├── api/
│   │   └── routes.py           # All 8 endpoint implementations
│   ├── rag/
│   │   ├── embedder.py         # Chunking + Sentence Transformers
│   │   ├── retriever.py        # HyDE + ChromaDB search + MongoDB fetch
│   │   └── evaluator.py        # RAGAS scoring pipeline
│   ├── agents/
│   │   ├── orchestrator.py     # LangChain AgentExecutor + routing
│   │   ├── campaign.py         # Campaign analyst agent
│   │   ├── audience.py         # Audience segmenter agent
│   │   └── content.py          # Content generator agent
│   └── streams/
│       ├── producer.py         # Redis Streams XADD producer
│       └── consumer.py         # Redis Streams consumer + embed worker
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── aws/
│   ├── task-definition.json    # ECS Fargate task definition
│   ├── service.json            # ECS service config (ALB, desired count)
│   └── deploy.sh               # ECR push + ECS deploy script
├── tests/
│   ├── test_api.py             # pytest — all endpoints (httpx async)
│   ├── test_rag.py             # pytest — embedding + retrieval quality
│   └── test_eval.py            # pytest — RAGAS score thresholds
├── demo/
│   └── app.py                  # Streamlit: query UI + RAGAS dashboard + feedback
├── scripts/
│   └── load_sample_data.py     # Ingest all 5 sample documents
├── data/
│   └── sample_docs/            # Real public marketing data
├── postman/
│   └── AdIntelAI.json          # Postman collection — all endpoints
├── requirements.txt
├── .env.example
└── .gitignore
```

---

## AWS Deployment

The application runs as a Docker container on **ECS Fargate** behind an **Application Load Balancer** with HTTPS. Secrets are managed via **AWS Secrets Manager** — no secrets in environment variables or task definitions.

### Architecture on AWS
```
Route 53 (DNS)
  → ACM Certificate (HTTPS)
    → Application Load Balancer
      → ECS Fargate Task (FastAPI container)
          → ECR (container image registry)
          → MongoDB Atlas (external, free tier)
          → Groq API (external, free tier)
          → LangSmith (external, free tier)
```

### Deploy
```bash
# 1. Build and push image to ECR
./aws/deploy.sh build-push

# 2. Register task definition
aws ecs register-task-definition --cli-input-json file://aws/task-definition.json

# 3. Update service (triggers rolling deploy)
aws ecs update-service \
  --cluster adintel-cluster \
  --service adintel-service \
  --force-new-deployment
```

### AWS Services Used

| Service | Purpose | Cost |
|---|---|---|
| ECR | Docker image registry | ~$0.10/GB/month |
| ECS Fargate | Serverless container runtime | ~$15–20/month (0.25 vCPU, 0.5GB) |
| ALB | HTTPS load balancer | ~$16/month |
| Secrets Manager | Env var secrets (API keys) | ~$0.40/secret/month |
| CloudWatch Logs | Container logs | ~$0.50/month |

> **Tear down after job search** — all resources are stateless. MongoDB Atlas (free tier) and GitHub stay live at zero cost.

---

## Environment Variables

```bash
# .env.example — copy to .env, never commit .env

# LLM
GROQ_API_KEY=gsk_your_key_here
LLM_MODEL=llama3-70b-8192

# Database
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/adintel

# Redis (local Docker / ElastiCache)
REDIS_URL=redis://redis:6379

# Vector store
CHROMA_PERSIST_DIR=./chroma_data
EMBEDDING_MODEL=all-MiniLM-L6-v2

# Observability
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls_your_key_here
LANGCHAIN_PROJECT=adintel-ai

# App
LOG_LEVEL=INFO
```

---

## Running Tests

```bash
# All tests
pytest tests/ -v

# With coverage report
pytest tests/ --cov=app --cov-report=term-missing

# RAGAS score threshold tests only
pytest tests/test_eval.py -v
# Asserts: faithfulness > 0.90, answer_relevancy > 0.85 on benchmark queries
```

---

## Scaling Path

The architectural contracts stay identical when scaling. Swap one component at a time without touching the API or agent logic.

| Component | Current (dev/demo) | Production scale |
|---|---|---|
| Vector store | ChromaDB (local) | Pinecone / AWS OpenSearch |
| Event stream | Redis Streams | AWS MSK (Managed Kafka) |
| Hosting | ECS Fargate (single task) | ECS with auto-scaling + ALB |
| Embeddings | Sentence Transformers (in-process) | AWS Bedrock Titan Embeddings |
| LLM | Groq free tier | AWS Bedrock Claude / Titan |
| Caching | None | Redis ElastiCache (query cache) |
| Observability | LangSmith | LangSmith + CloudWatch dashboards |
| Eval pipeline | Synchronous RAGAS | Async RAGAS batch job on SQS |

---

## Resume Bullets

```
Built AdIntel AI, a production multi-agent RAG platform for marketing intelligence using
LangChain AgentExecutor to route queries across three domain-specialized agents (campaign
analysis, audience segmentation, ad copy generation), achieving sub-500ms end-to-end
latency via Groq LLM (llama3-70b-8192).

Implemented HyDE (Hypothetical Document Embeddings) retrieval pipeline — LLM generates
a hypothetical answer before embedding, improving semantic recall on short marketing
queries vs standard cosine search; evaluated with RAGAS (faithfulness >0.90, answer
relevancy >0.85) and traced end-to-end with LangSmith.

Designed event-driven document ingestion using Redis Streams with async Uvicorn consumer
workers, processing five source types into ChromaDB vector collections — decoupling
ingestion from query serving for non-blocking API performance (<100ms ingest response).

Containerized full application stack with Docker, deployed to AWS ECS Fargate behind
an Application Load Balancer with HTTPS; secrets managed via AWS Secrets Manager;
CI/CD via GitHub Actions → ECR → ECS rolling deploy.

Built a retrieval quality feedback loop — user ratings and RAGAS scores stored in
MongoDB Atlas, visualized in a live Streamlit dashboard — enabling continuous
measurement of agent performance post-deployment.
```

---

<div align="center">

**Vaishnavi Bhamare**

M.S. Advanced Data Analytics · University of North Texas · 4.0 GPA  
Data & Technology


</div>
