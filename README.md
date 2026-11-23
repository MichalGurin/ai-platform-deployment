# AI Platform (unified deployment)

Single, deployable AI platform composed from three parts that live in their own
public GitHub repositories (consumed here as git submodules):

| Part | Repository | Responsibility |
| --- | --- | --- |
| AI Quality & Safety Gateway | [MichalGurin/llm-gateway](https://github.com/MichalGurin/llm-gateway) | Safety (PII redaction, prompt-injection detection), token bucket + daily budget, provider fallback (local/vLLM → OpenAI → Anthropic), SSE streaming proxy, OpenAI-compatible `POST /v1/chat/completions` and `POST /v1/embeddings`, gRPC contract, Prometheus metrics |
| RAG chain | [MichalGurin/rag-chain](https://github.com/MichalGurin/rag-chain) | Document upload → chunk → embed → hybrid retrieval (BM25 + pgvector) → rerank → grounded, cited answers; LLM cost guardrails; chat API (`/chat`) and document ingestion |
| SandboxMCP | [MichalGurin/mcp-server-sandbox](https://github.com/MichalGurin/mcp-server-sandbox) | MCP server with guarded tools/resources/prompts: read-only SQL, log tailing, isolated Docker execution |

## Unified LLM traffic path

All LLM traffic of every component flows through the gateway, so there is exactly
one place for safety, rate limiting, budgeting, fallback, and observability:

```
clients / coding agents ──▶ llm-gateway ──▶ local backend (Ollama / vLLM / llama.cpp)
                                        │
                                        └──▶ OpenAI / Anthropic (via fallback)
```

- The **RAG API** sends its chat completions (primary, fallback, query expansion,
  grounding, general-knowledge) *and* its embeddings to the gateway via
  `LLM_BASE_URL=http://gateway:8000/v1` and `LLM_API_KEY=<GATEWAY_API_KEY>`.
- The **gateway** proxies to the configured OpenAI-compatible backend and normalizes
  responses to the OpenAI chat-completions and embeddings formats.
- The **coding agent** (OpenCode, Claude Code, Cursor, …) can use the gateway as its
  OpenAI-compatible model provider (`baseURL: http://<gateway>:8000/v1`) and the
  SandboxMCP server for guarded DB/log/docker tools.

## Architecture

```
                        ┌────────────────────────────────────────────────────────┐
                        │                                                            │
              ┌─────────▼─────────┐         ┌───────────────────▼─────────────────┐
              │ Frontend (Svelte)  │         │ Coding agent (OpenCode, …)          │
              │ → RAG API /chat   │         │  MCP tools → SandboxMCP            │
              └─────────┬─────────┘         │  LLM → llm-gateway                 │
                        │                  └───────────▲─────────────────────┘
              ┌─────────▼─────────┐         ┌───────┴─────────────────────┐
              │ RAG API           │         │ SandboxMCP                   │
              │ (retrieval,       │         │ (read-only DB, logs,         │
              │  grounding,       │         │  isolated Docker execution)  │
              │  citations)       │         │                              │
              └─────────┬─────────┘         └───────────────▲──────────────┘
                        │ LLM                            │ DB = same PostgreSQL
                        ▼                                │
              ┌────────────────┐                       │
              │ llm-gateway    │───────────────────────┘
              │ (chat +       │        ┌────────────────────────────┐
              │  embeddings)  │───────▶│ PostgreSQL + pgvector      │
              └───────┬────────┘        │ (RAG corpus + app DB)     │
                      │                 └────────────────────────────┘
                      ▼
              ┌────────────────┐
              │ Local backend  │  (Ollama / vLLM / llama.cpp)
              └────────────────┘
```

## Quickstart

```bash
git clone <this-repository>
cd ai-platform
git submodule update --init --recursive

cp .env.example .env
# set GATEWAY_API_KEY (min. 32 chars) and, if needed, provider credentials

docker compose up --build
```

Services started:

- `gateway` – AI Quality & Safety Gateway (`http://localhost:8000`, gRPC `50051`)
- `agent-api` – RAG API (`http://localhost:8000`, docs at `/docs`)
- `frontend` – Svelte UI (`http://localhost:5173`)
- `ingestion-worker` / `job-dispatcher` – Celery ingestion
- `db` – PostgreSQL + pgvector
- `ollama` (+ `ollama-model-init`) – local chat/embedding models
- `redis` – rate-limit/budget/cache broker
- `mcp` – SandboxMCP (HTTP on `127.0.0.1:8001`, DB = the RAG PostgreSQL)

## Notes

- The gateway now exposes both `POST /v1/chat/completions` and `POST /v1/embeddings`,
  so RAG chat and RAG retrieval/embedding use one unified, safety-gated LLM path.
- The RAG API is pointed at the gateway via `LLM_BASE_URL=http://gateway:8000/v1` and
  authenticates with the gateway bearer key via `LLM_API_KEY`.
- SandboxMCP is pointed at the shared RAG PostgreSQL via `SANDBOX_DB_URL` and can run
  the RAG test suite inside an isolated Docker container via `run_isolated_test`.
