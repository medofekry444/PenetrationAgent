# PenetrationAgent

full architecture

## Language/Service Split

| Layer | Tech | Why |
|---|---|---|
| Orchestrator + API + Auth + Scope Engine | ASP.NET Core | Safety-critical, needs strong typing/audit trail, not perf-bound |
| Recon/scan workers | Go | Wraps/calls Go-native tools (subfinder, httpx, naabu, dnsx, katana) directly — no subprocess overhead, real concurrency for mass probing |
| LLM/RAG/agent reasoning service | Python | FastAPI service, only place doing model calls, tool-calling loops, embeddings |
| Frontend | Next.js/React | Dashboard, findings graph, approval UI |
| Queue | Redis | Task dispatch between orchestrator and workers |
| DB | PostgreSQL + pgvector | Findings, scope config, audit log, embeddings in one place (skip Qdrant at MVP — one less service to run) |
| Object storage | MinIO | Raw tool output, screenshots, PCAPs, reports |
| Execution | Docker (rootless), each tool its own image | Isolation, reproducibility |
| Observability | OpenTelemetry → Prometheus/Grafana | You need this from day one given the audit requirements |

## LLM Strategy: Local + API Hybrid

Route by task, not by preference — the routing logic lives in the Python reasoning service:

*Local (Ollama)* — for volume, low-risk, or sensitive-data tasks:
- Parsing/summarizing raw tool output before it hits the DB
- Classifying/deduplicating findings
- Initial triage ("is this noise or worth escalating")
- Anything touching client data you don't want leaving your infra (this matters a lot for pentest engagements under NDA)
- Model choice: something like Qwen2.5-Coder or Llama 3.1 70B-class if you have the GPU; smaller (8B) models are fine for classification/summarization tasks specifically

*API (Claude/GPT)* — for the hard reasoning:
- Attack-chain hypothesis generation from the finding graph
- Final report writing (narrative, executive summary)
- Complex multi-step reasoning where a wrong local-model inference is costly
- Anything requiring the largest context window (correlating findings across a big engagement)

*Routing rule of thumb*: if a wrong answer just means slightly worse triage, use local. If a wrong answer means a false claim in a client-facing report or a missed attack chain, use API. Log which tier produced every claim — this feeds directly into your confidence system (local-model inference = Low, API-model inference = Low-Medium, only tool evidence + human = High/Confirmed).

## Component Plan

*1. Scope Engine (ASP.NET Core, build first)*
Domains/IPs/CIDRs allow+deny lists, allowed test types, rate limits, allowed hours, environment tag (prod/staging), credentials store (encrypted, scoped per-target). Every single worker call passes through this before execution — no exceptions, no bypass path even for "just recon."

*2. Tool Gateway (ASP.NET Core)*
Sits between orchestrator and workers. Checks: in scope? tool permitted? destructive? needs approval? rate ok? Assigns action to Level 1/2/3 and either dispatches immediately or queues for human approval (Level 3).

*3. Orchestrator (ASP.NET Core background services)*
Drives the state machine: target added → scope validation → passive recon → active recon → attack-surface mapping → hypothesis generation (calls Python reasoning service) → safe testing → evidence collection → validation → dedup → risk classification → report.

*4. Workers (Go)*
Thin: receive job from Redis queue, run containerized tool, normalize output to your standard JSON schema, push result back. No reasoning, no scope logic — that's already been checked before the job reached them.

*5. Reasoning Service (Python/FastAPI)*
Only place that calls LLMs. Exposes internal endpoints like /summarize-findings, /generate-hypothesis, /write-report-section. Does the local/API routing. Owns the RAG layer (pgvector) — methodology docs, past findings, payload knowledge, tool docs.

*6. Finding Graph + Report Agent*
Findings stored as nodes/edges (Postgres is fine for this at MVP — don't reach for a graph DB yet), so the reasoning service can trace chains like 443 → nginx → /api/v2 → JWT auth → /users/{id} → auth hypothesis instead of reasoning over flat alerts.

## MVP Build Order

1. Scope engine + tool gateway (nothing runs without this working first)
2. Target/scope CRUD + audit log
3. Recon worker (Go): subfinder → dnsx → httpx → naabu, normalized JSON output
4. Tech fingerprinting + endpoint collection (katana, gau)
5. Nuclei integration (Level 1/2 templates only, no exploit templates yet)
6. Local LLM summarization of raw output → findings table
7. Basic dedup + confidence scoring
8. Human review UI (approve/reject/escalate findings)
9. API-model report generation from confirmed findings
10. Only then: Level 2 fuzzing (ffuf/feroxbuster) with strict rate limits and human-visible logs

Exploit validation, cloud modules, mobile, and AD/network tooling all come after this loop is solid and trusted — not before.