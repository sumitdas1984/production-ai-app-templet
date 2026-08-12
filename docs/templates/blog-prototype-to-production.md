# Taking an AI Agent from Prototype to Production: A Practical, Python-First Stack

*Most agents die in production. Here's the engineering around the model that keeps them alive.*

---

Building an AI agent is easy.

> Building one that survives production is a completely different problem.

A production AI agent needs more than a powerful LLM. It needs the right engineering stack **around the model** — orchestration, data, observability, testing, deployment, and runtime. Here's the eight-layer, Python-first stack that takes one from prototype to a system you can actually rely on.

---

## The Stack at a Glance

![Production AI Agent Stack — workflow and architecture](ai-agent-stack.png)

> **The stack around the model makes it production ready.**

---

## Layer 1 — 🟢 API & Validation: FastAPI + Pydantic

Every external interaction with your agent — from your web UI, your mobile app, your internal services, your partner integrations — enters and exits through one HTTP surface. That surface is the contract you keep with the rest of the world. If it's loose, every downstream component inherits the looseness.

**FastAPI** handles the HTTP layer. Async-native, dependency injection for auth/db/sessions, automatic OpenAPI docs.

**Pydantic** does more than validate request bodies. It's the single source of truth for *every* shape in your system — including the structured outputs you coerce out of the LLM. If you're parsing model output with regex in 2026, you're doing it wrong. Pass a `model_json_schema()` to the model, get JSON back, validate it into a typed object, and never again debug a stringified JSON that lost a brace.

**Production patterns:**
- Validate at the **edge**, not deep inside the agent. Reject malformed input before it costs a token.
- Version your routes (`/v1/...`) the day you ship — agents are sticky contracts.
- Pin your Pydantic schemas. Free-form validation is a future outage.

---

## Layer 2 — 🔵 Agent Orchestration: LangGraph

A prototype agent is usually a single loop: `while tool_call: llm.chat()`. A production agent needs explicit control over its state, branches, retries, and human approvals. Without that, you get a system that quietly degrades in ways nobody notices until a customer does.

**LangGraph** turns your agent into a declarative DAG with first-class support for:

- **Nodes & edges** — your workflow, as code.
- **State management** — typed, persisted, replayable.
- **Tool calling** — schemas, not string parsing.
- **Human-in-the-loop** — interrupt and resume at any node.
- **Conditional workflows** — branches based on what the model returned, what you retrieved, or what your business rules say.

**Production patterns:**
- Make every node **idempotent**. Retries will happen. Side effects must not double-fire.
- Persist checkpoints to Postgres — never lose a half-finished run.
- Export the graph as a Mermaid diagram in the PR description. A picture beats a thousand words and a hundred "wait, what does this node do?" Slack messages.

---

## Layer 3 — 🟡 Data & State: PostgreSQL + pgvector + Redis

Agents don't live in stateless demos. They need durable state (run history, audit trail, long-term memory), vector search (RAG, semantic recall), and fast ephemeral state (rate limits, session caches, deduplication keys).

**PostgreSQL + pgvector** keeps application data, audit logs, and vector embeddings in one transactional store. The embedding lives on the same row as the metadata that owns it — no sync drift between your vector index and your source-of-truth table.

**Redis** is for everything that's fast and short-lived:

- Caching
- Rate limiting
- Short-term conversation state
- Memoizing expensive tool results within a run

**Production patterns:**
- Treat embedding migrations like **schema migrations**. Backfills are slow; you need to be able to re-run them safely.
- Use Redis for **edge rate limiting**, not as your system of record.
- `EXPLAIN ANALYZE` your vector retrieval query before shipping. "Fast enough" is not a benchmark.

---

## Layer 4 — 🟣 LLM Gateway: LiteLLM

A prototype uses one model. A production system uses many — different models for different tasks, plus fallbacks when a provider has an outage, plus the ability to swap providers without rewriting your agent.

**LiteLLM** is a unified interface across OpenAI, Anthropic, Bedrock, Vertex, and self-hosted models. It gives you:

- **Retries** with backoff.
- **Fallbacks** — if OpenAI is down, route to Anthropic. If Anthropic 429s, route to a self-hosted model.
- **Provider switching by config**, not by code change.

If one provider goes down, your entire agent doesn't have to go down with it.

**Production patterns:**
- Separate **primary**, **fallback**, and **budget** models. A failing primary shouldn't burn your budget on the fallback.
- Log every call to your observability stack (next layer). You cannot debug provider drift without per-call traces.
- Pin model versions (`gpt-5-2026-04-01`, not `gpt-5-latest`). Unpinned models are how you wake up to a 3 a.m. incident.

---

## Layer 5 — 🔍 Observability & LLMOps: Langfuse (or Opik)

Agents are non-deterministic, multi-step, and external-API-dependent. Without observability, the first time something goes wrong, you find out from a customer.

Observability turns "the agent is broken" into *"step 3 in run `abc123` failed because retrieval returned zero rows because the pgvector index wasn't rebuilt after the schema migration."*

Track everything:

- **LLM calls** — prompt, completion, model, latency, tokens.
- **Costs** — per run, per feature, per provider.
- **Traces** — the full graph execution, including tool calls and conditional branches.
- **Errors** — with full context, not just stack traces.
- **Feedback** — explicit user ratings and implicit signals (re-runs, edits, abandons).

**Production patterns:**
- Treat trace IDs as **first-class** in your logs and error reports.
- Sample **100% of errors**, **1–10% of successes** to control volume.
- Wire feedback into your eval pipeline. Production signal is the most honest dataset you'll ever get.

---

## Layer 6 — ⚡ Development Tooling: uv + Ruff + Pytest

A team moving fast without reproducible environments and fast feedback is a team that ships "works on my machine." For AI work specifically, the iteration loop is even tighter — you change a prompt, you need to know in seconds whether you broke something.

**uv** is dependency management and virtualenvs, orders of magnitude faster than pip or Poetry, with a lockfile that pins everything reproducibly.

**Ruff** is lint + format in Rust. One tool replaces flake8, isort, black, and pyupgrade.

**Pytest** is the test runner — unit, integration, and (with care) agent evals.

**Production patterns:**
- Run ruff + pytest in **pre-commit** and **CI**. Block the merge on either failing.
- Keep **evals** distinct from **tests**. Tests assert deterministic contracts; evals measure quality on a golden dataset. Don't conflate them.
- A 10-second test suite is worth more than a 10-minute one. Use uv's speed to your advantage.

---

## Layer 7 — 📦 Containerization & Deployment: Docker + GitHub Actions

"It works in dev" is the most common phrase in postmortems. Docker makes the artifact identical from laptop to prod. GitHub Actions makes the path from commit to deploy a reproducible pipeline that humans don't get to forget steps in.

**Docker** packages the app + its Python version + system libs + dependencies into a single image. The image *is* the artifact.

**GitHub Actions** runs CI/CD: lint → test → build image → push to registry → deploy. No buttons, no "I forgot to run the migrations," no "oh, that branch wasn't deployed."

**Production patterns:**
- **Multi-stage builds** to keep images small and secrets out of layers.
- **Pin base image digests**, not tags. `python:3.13-slim` can drift under you.
- Deploys should be **gated by tests**, not by humans clicking buttons. A failing test should block a deploy, not just a PR.

---

## Layer 8 — ☁️ Production Runtime: AWS ECS Fargate / GCP Cloud Run

You built a great container. Now where does it run?

Serverless container platforms give you the operational benefits of containers (reproducibility, isolation) without the operational cost of Kubernetes for teams that don't need K8s.

**AWS ECS Fargate** — task-based, IAM-integrated, VPC-native. Good fit when you're already deep in AWS.

**GCP Cloud Run** — request-based autoscaling, scale-to-zero, dead simple. Good fit for spiky agent traffic.

Both: no nodes to manage, pay per use, HTTPS termination handled.

**Production patterns:**
- Match the platform to the workload. **Bursty HTTP** → Cloud Run. **Steady long-running agent workers** → ECS Fargate.
- Always set **max-scale**, not just min-scale. A bug plus unbounded scale equals the bill that ends the project.
- Wire platform logs and metrics into the **same observability stack** as your LLM traces. Logs you can't correlate with traces are logs you can't act on.

---

## The Production Readiness Checklist

Use this as a gate before declaring an agent "production-ready."

### Reliability
- [ ] Every node in the agent graph is idempotent.
- [ ] Graph state is checkpointed to durable storage.
- [ ] LLM calls go through a gateway with retries and provider fallback.
- [ ] All external calls have explicit timeouts.

### Observability
- [ ] 100% of LLM calls are traced with prompt, completion, tokens, latency.
- [ ] Errors carry trace IDs and are surfaced in the same UI as traces.
- [ ] Cost per run is computed and alertable.
- [ ] User feedback is captured and joinable to traces.

### Data
- [ ] Embeddings live next to the rows they describe.
- [ ] Vector retrieval queries are `EXPLAIN`'d and indexed.
- [ ] Ephemeral state lives in Redis; nothing important does.

### Quality
- [ ] Offline eval runs on every PR against a golden dataset.
- [ ] Online monitor tracks quality on live traffic and alerts on drift.
- [ ] Lint and tests block merges via CI.

### Deployment
- [ ] Image is built from a pinned base image digest.
- [ ] Secrets are injected at runtime, not baked into the image.
- [ ] Deploys are gated by passing tests, not by humans.
- [ ] Max-scale is configured; scale-to-zero is intentional.

### Security
- [ ] Every input is validated at the edge.
- [ ] Tool calls are scoped and authorized.
- [ ] API auth is wired at the HTTP layer, not retrofitted later.

---

## The Part Most People Skip

The LLM is **one** component of an AI agent.

The model gets the attention. The engineering around it does the work.

> **Model + orchestration + data + observability + testing + deployment = Production AI Agent**

This isn't the only stack that works, and there are real alternatives at every layer — different orchestrators, different gateways, different runtimes. But if you're looking for a simple, practical Python foundation to build production-grade agents on, these eight layers will take you from a notebook demo to a system you can actually sleep through the night with.

Pick the layers. Wire them in order. Ship.

*— Originally published on Medium.*