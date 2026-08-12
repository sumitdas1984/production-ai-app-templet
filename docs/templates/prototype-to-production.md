# 🚀 Taking an AI Agent from Prototype → Production

> Building an AI agent is easy.
> Building one that survives production is a completely different problem.

A production AI agent needs more than a powerful LLM. It needs the right engineering stack **around the model** — orchestration, data, observability, testing, deployment, and runtime. This document walks through the eight layers of a practical, Python-first production AI stack, and maps each layer to the concrete folders in this repository so the theory stays grounded in code.

---

## 1. The Stack at a Glance

![Production AI Agent Stack — workflow and architecture](ai-agent-stack.png)

> **The stack around the model makes it production ready.**

---

## 2. The Eight Layers

### 🟢 Layer 1 — API & Validation: FastAPI + Pydantic

**Why it matters.** Every external interaction with your agent enters and exits through an HTTP surface. That surface is the contract you keep with the rest of the world — clients, UIs, internal services, partner integrations. If it's loose, every downstream component inherits the looseness.

**What they do.**
- **FastAPI** — async HTTP layer, OpenAPI docs for free, dependency injection for auth/db/sessions.
- **Pydantic** — single source of truth for request, response, and *internal* data shapes. Critically: also validates **structured LLM outputs** so a free-form model response can be coerced into a typed object before it touches your business logic.

**In this repo.**
- `app/main.py` — FastAPI entry point and router wiring.
- `app/models.py` — Pydantic models for requests, responses, and structured agent outputs.
- `app/config.py` — environment-driven settings (model name, provider keys, timeouts).

**Production patterns.**
- Pin a Pydantic `model_json_schema()` and pass it to the LLM (JSON mode / function calling) — never parse model text with regex.
- Validate at the **edge**, not deep inside the agent. Reject malformed input before it costs a token.
- Version your routes (`/v1/...`) the day you ship — agents are sticky contracts.

---

### 🔵 Layer 2 — Agent Orchestration: LangGraph

**Why it matters.** A prototype agent is usually a single `while tool_call: llm.chat()` loop. A production agent needs explicit control over its state, branches, retries, and human approvals. Without that, you get a system that works on Tuesday and silently degrades on Wednesday.

**What LangGraph gives you.**
- **Nodes & edges** — declarative DAG of agent steps.
- **State management** — typed, persisted, replayable state per run.
- **Tool calling** — first-class tools with schemas, not string parsing.
- **Human-in-the-loop** — interrupt + resume at any node.
- **Conditional workflows** — branches based on model output, retrieved context, or business rules.

**In this repo.**
- `app/agents/` — agent definitions and graph assembly.
- `app/components/` — reusable nodes (retrievers, routers, validators) wired into the graph.
- `app/prompts/` — versioned prompt templates, separated from control flow.
- `app/services/` — domain services the graph calls (tools, side effects).

**Production patterns.**
- Make every node **idempotent** so retries don't duplicate side effects.
- Persist checkpoints to Postgres — never lose a half-finished run.
- Keep the graph **visible** (a Mermaid export in the PR description beats a thousand words).

---

### 🟡 Layer 3 — Data & State: PostgreSQL + pgvector + Redis

**Why it matters.** Agents don't live in stateless demos. They need durable state (long-term memory, run history, audit trail), vector search (RAG, semantic recall), and fast ephemeral state (rate limits, session caches, deduplication keys).

**What they do.**
- **PostgreSQL + pgvector** — application data, audit logs, vector embeddings in one transactional store. Keeps embeddings and the metadata that owns them on the same row — no sync drift.
- **Redis** — caching, rate limiting, short-lived conversation state, fast lookups for session/tool result memoization.

**In this repo.**
- `data/raw/` — input corpora.
- `data/processed/` — cleaned, chunked, embedded data.
- `data/index_config/` — index/pgvector configuration.
- `docker-compose.yml` — local Postgres + Redis for development parity.

**Production patterns.**
- Treat embeddings like a **schema migration** — backfills are slow and you need to be able to re-run them.
- Use Redis for **rate limiting at the edge**, not as the system of record.
- Keep pgvector row counts honest: `EXPLAIN ANALYZE` your retrieval query before shipping.

---

### 🟣 Layer 4 — LLM Gateway: LiteLLM

**Why it matters.** A prototype uses one model. A production system uses many — different models for different tasks, plus fallbacks when a provider has an outage, plus the ability to swap providers without rewriting your agent.

**What LiteLLM gives you.**
- A **unified interface** across OpenAI, Anthropic, Bedrock, Vertex, self-hosted, etc.
- **Retries** with backoff.
- **Fallbacks** — if OpenAI is down, route to Anthropic; if Anthropic 429s, route to a self-hosted model.
- **Provider switching** by config, not code.

**In this repo.**
- `app/services/` — thin wrapper around LiteLLM. The agent never imports `openai` or `anthropic` directly.
- `app/config.py` — model routing rules live in config, not in code paths.

**Production patterns.**
- Separate **primary**, **fallback**, and **budget** models. A failing primary shouldn't burn your budget on the fallback.
- Log every call to Langfuse (next layer) — you cannot debug provider drift without per-call traces.
- Pin model versions (`gpt-5-2026-04-01`, not `gpt-5-latest`). Unpinned models are how you wake up to a 3am incident.

---

### 🔍 Layer 5 — Observability & LLMOps: Langfuse / Opik

**Why it matters.** Agents are non-deterministic, multi-step, and external-API-dependent. Without observability, the first time something goes wrong you find out from a customer. Observability turns "the agent is broken" into "step 3 in run `abc123` failed because retrieval returned zero rows because the pgvector index wasn't rebuilt after the schema migration."

**What to track.**
- **LLM calls** — prompt, completion, model, latency, token counts.
- **Costs** — per-run and per-feature, broken down by provider and model.
- **Traces** — the full graph execution, including tool calls and conditional branches.
- **Errors** — with full context, not just stack traces.
- **Feedback** — explicit user ratings and implicit signals (re-runs, edits, abandons).

**In this repo.**
- `observability/tracer.py` — trace emission hook for the agent.
- `observability/cost_tracker.py` — token and cost rollups.
- `observability/feedback.py` — user-feedback capture wired to traces.

**Production patterns.**
- Treat trace IDs as **first-class** in your logs and error reports.
- Sample **100% of errors** and **1–10% of successes** to control volume.
- Wire feedback into your eval pipeline (`evaluation/`) — production signal is the most honest dataset you'll ever get.

---

### ⚡ Layer 6 — Development Tooling: uv + Ruff + Pytest

**Why it matters.** A team moving fast without reproducible environments and fast feedback is a team that ships "works on my machine." For AI work specifically, the iteration loop is even tighter — you change a prompt, you need to know in seconds whether you broke something.

**What they do.**
- **uv** — dependency management and virtualenvs. Orders of magnitude faster than pip/poetry, with a lockfile that pins everything reproducibly.
- **Ruff** — lint + format, written in Rust. One tool replaces flake8, isort, black, pyupgrade.
- **Pytest** — the test runner. Use it for unit, integration, and (with care) agent evals.

**In this repo.**
- `pyproject.toml` — single source of truth for dependencies, ruff config, pytest config.
- `.python-version` — pinned Python version for uv.
- `tests/` — `test_cache.py`, `test_routing.py`, `test_retrieval.py` for unit/integration tests.
- `evaluation/offline_eval.py`, `evaluation/online_monitor.py` — eval runs, separate from unit tests.
- `scripts/` — operational scripts (`migrate.py`, `seed.py`, `healthcheck.py`).

**Production patterns.**
- Run ruff + pytest in **pre-commit** and **CI**. Block the merge on either failing.
- Keep **evals** (`evaluation/`) distinct from **tests** (`tests/`). Tests assert deterministic contracts; evals measure quality on a golden dataset. Don't conflate them.
- A 10-second test suite is worth more than a 10-minute one. Use uv's speed to your advantage.

---

### 📦 Layer 7 — Containerization & Deployment: Docker + GitHub Actions

**Why it matters.** "It works in dev" is the most common phrase in postmortems. Docker makes the artifact identical from laptop to prod. GitHub Actions makes the path from commit to deploy a reproducible pipeline that humans don't get to forget steps in.

**What they do.**
- **Docker** — packages the app + its Python version + system libs + dependencies into a single image. The image is the artifact.
- **GitHub Actions** — CI/CD: lint, test, build image, push to registry, deploy.

**In this repo.**
- `app/Dockerfile` — application image.
- `frontend/Dockerfile` — frontend image.
- `docker-compose.yml` — local multi-service stack (Postgres, Redis, app) for dev parity.
- `.github/workflows/` — CI/CD pipelines (lint, test, build, deploy).

**Production patterns.**
- **Multi-stage builds** to keep images small and secrets out of layers.
- **Pin base image digests**, not tags. `python:3.13-slim` can drift under you.
- Deploys should be **gated by tests**, not by humans clicking buttons. A failing test should block a deploy, not just a PR.

---

### ☁️ Layer 8 — Production Runtime: AWS ECS Fargate / GCP Cloud Run

**Why it matters.** You built a great container. Now where does it run? Serverless container platforms give you the operational benefits of containers (reproducibility, isolation) without the operational cost of Kubernetes for teams that don't need K8s.

**What they give you.**
- **AWS ECS Fargate** — task-based, IAM-integrated, VPC-native. Good fit when you're already deep in AWS.
- **GCP Cloud Run** — request-based autoscaling, scale-to-zero, simple. Good fit for spiky agent traffic.
- Both: **no nodes to manage**, **pay per use**, **HTTPS termination** handled.

**In this repo.**
- Deployment manifests and IaC live alongside `.github/workflows/`. Wire your platform choice into the workflow that consumes the Docker image from Layer 7.

**Production patterns.**
- Match the platform to the workload. **Bursty HTTP** → Cloud Run. **Steady long-running** agent workers → ECS Fargate (or GKE if you need K8s).
- Always set **max-scale**, not just min-scale. A bug + unbounded scale = the bill that ends the project.
- Wire platform logs and metrics into the **same observability stack** as Langfuse. Logs you can't correlate with traces are logs you can't act on.

---

## 3. Mapping the Stack to This Repository

| Layer | Component | Where it lives |
|---|---|---|
| 🟢 API & Validation | FastAPI, Pydantic | `app/main.py`, `app/models.py`, `app/config.py` |
| 🔵 Agent Orchestration | LangGraph | `app/agents/`, `app/components/`, `app/prompts/` |
| 🟡 Data & State | Postgres + pgvector, Redis | `data/`, `docker-compose.yml` |
| 🟣 LLM Gateway | LiteLLM | `app/services/`, `app/config.py` |
| 🔍 Observability | Langfuse | `observability/tracer.py`, `observability/cost_tracker.py`, `observability/feedback.py` |
| ⚡ Dev Tooling | uv, Ruff, Pytest | `pyproject.toml`, `.python-version`, `tests/`, `scripts/` |
| 📦 CI/CD | Docker, GitHub Actions | `app/Dockerfile`, `frontend/Dockerfile`, `docker-compose.yml`, `.github/workflows/` |
| ☁️ Runtime | ECS Fargate / Cloud Run | wired via `.github/workflows/` |

---

## 4. Production Readiness Checklist

Use this as a gate before declaring an agent "production-ready."

### Reliability
- [ ] Every node in the agent graph is idempotent.
- [ ] Graph state is checkpointed to durable storage (Postgres).
- [ ] LLM calls go through a gateway with retries and provider fallback.
- [ ] All external calls have explicit timeouts.

### Observability
- [ ] 100% of LLM calls are traced with prompt, completion, tokens, latency.
- [ ] Errors carry trace IDs and are surfaced in the same UI as traces.
- [ ] Cost per run is computed and alertable.
- [ ] User feedback is captured and joinable to traces.

### Data
- [ ] Embeddings live in pgvector next to the rows they describe.
- [ ] Vector retrieval queries are EXPLAIN'd and indexed.
- [ ] Redis is used for ephemeral state, not system of record.

### Quality
- [ ] Offline eval runs on every PR against `evaluation/golden_dataset.json`.
- [ ] Online monitor tracks quality on live traffic and alerts on drift.
- [ ] Ruff and Pytest block merges via CI.

### Deployment
- [ ] Image is built from a pinned base image digest.
- [ ] Secrets are injected at runtime, not baked into the image.
- [ ] Deploys are gated by passing tests, not by humans.
- [ ] Max-scale is configured; scale-to-zero is intentional.

### Security
- [ ] Pydantic validates every input at the edge.
- [ ] Tool calls are scoped and authorized (see `app/security/`).
- [ ] API auth is wired at the FastAPI layer, not retrofitted later.

---

## 5. The Important Part

The LLM is **one** component of an AI agent.

The surrounding engineering is what determines whether your agent is:

- ❌ A cool demo
- ✅ A reliable production system

> **Model + orchestration + data + observability + testing + deployment = Production AI Agent**

This isn't the only possible stack, and there are plenty of alternatives at every layer. But if you're looking for a simple, practical Python stack to start building production-grade AI agents, these eight layers are a strong foundation — and this repository is wired so each one has a clear home.

---

## 6. Further Reading

- `docs/architecture.md` — how this repo's layers connect.
- `docs/api-reference.md` — API contracts.
- `docs/deployment.md` — deploy runbook.
- `evaluation/offline_eval.py` — how to add a new eval.
- `observability/tracer.py` — how to add a new trace.