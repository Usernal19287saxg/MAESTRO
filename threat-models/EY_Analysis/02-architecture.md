# Phase 2: Architecture Analysis — EY_Analysis

**Project:** EY_Analysis
**Date:** 2026-03-27
**Analysis Depth:** Full
**System Type:** Multi-Agent
**Uses MCP:** Yes

---

## MAESTRO Layer Mapping

| MAESTRO Layer | System Components & Features | Notes |
|---------------|------------------------------|-------|
| **L1. Foundation Models** | Primary reasoning models (GPT-4o / Claude 3.x / Gemini 1.5 Pro class) via vendor API; open-weight models (Llama 3, Mistral) self-hosted on GPU cluster; secondary models for embedding generation and classification | Dual access: API + self-hosted. No fine-tuning at baseline; fine-tuning marked as high-risk configuration overlay. Self-hosted models introduce model supply chain risk |
| **L2. Data Operations** | RAG pipeline (chunking + embedding); Vector DB (Weaviate/Qdrant self-hosted or Pinecone SaaS); Object storage (S3-compatible); PostgreSQL (structured metadata + long-term memory); Redis (session-scoped state); Internal relational DBs; Code repositories | Three-tier memory: in-context (ephemeral), session-scoped (Redis, TTL-bound), long-term (PostgreSQL/vector DB, persistent). External data sources: public web, SaaS APIs, partner feeds |
| **L3. Agent Frameworks** | Graph-based workflow engine (LangGraph-class); Role-based multi-agent framework (AutoGen/CrewAI-class); 5 agent roles (Orchestrator, Researcher, Executor, Critic/Validator, Notifier); 7 MCP servers (Filesystem, Code Execution, Database, Web Search, API Gateway, Communication, Memory); MCP transport: HTTP/SSE (primary) + stdio (local) | Three inter-agent communication patterns: direct call, shared state (blackboard), message bus (async). MCP is the universal tool interface layer |
| **L4. Deployment Infrastructure** | Kubernetes (production); Docker Compose (dev/test); Separate containers per agent role; Separate containers per MCP server; Network policies for pod isolation; GPU inference cluster (self-hosted models); Cloud-native (AWS/Azure/GCP agnostic); On-prem variant for regulated deployments | Resource limits per container. Network policies restrict lateral movement. Hybrid deployment for self-hosted LLM workloads |
| **L5. Evaluation & Observability** | Centralized log aggregation (ELK / OpenTelemetry); Per-LLM-call logging (prompt, response, tokens, latency, tool invocations, agent role); Dashboards for per-agent activity; Alerting (error rate, latency, anomalous tool call volume); HITL approval queue UI; Rule-based anomaly detection | HITL: high-risk tool calls routed to human review; timeout → auto-reject (fail-safe). ML-based behavioral anomaly detection is roadmap only (not implemented at baseline) |
| **L6. Security & Compliance** | OIDC/OAuth 2.0 SSO (users); JWT + API key pairs (API clients); mTLS (agent↔MCP HTTP transport); Process-level trust (stdio transport); RBAC: 4 roles (Admin, Operator, User, ReadOnly); Tool-level permissions per MCP tool; HashiCorp Vault (or cloud-native equivalent); Dynamic secrets with short TTL; Vault Agent for secret injection | No secrets in env vars or config files in production. Agent service accounts scoped per role. GDPR, EU AI Act, NIST AI RMF, ISO/IEC 42001 as baseline compliance |
| **L7. Agent Ecosystem** | External SaaS: email (SMTP/SendGrid), Slack, Teams; Dev tooling: GitHub/GitLab, Jira, CI/CD APIs; Cloud services: S3, cloud functions; Web search APIs; A2A cross-platform agent invocation (Google A2A spec + webhook/API); Three trust tiers: Vendor SaaS, Partner APIs, Public/open APIs | Cross-platform agent calls treated as untrusted input. Payment/ERP as deployment-specific overlay. All public API output treated as potentially adversarial |

---

## Component Inventory

### Foundation Models (L1)

| Component | Type | Access Method | Trust Level |
|-----------|------|--------------|-------------|
| GPT-4o / Claude 3.x / Gemini 1.5 Pro | Primary reasoning LLM | Vendor API | Vendor-trusted (contractual) |
| Llama 3 / Mistral | Open-weight LLM | Self-hosted GPU cluster | Self-managed (full control, supply chain risk) |
| Embedding model | Specialized model | API or self-hosted | Same as parent model |
| Classification model | Specialized model | API or self-hosted | Same as parent model |

### Agent Roles (L3)

| Role | Function | Privilege Level | MCP Tools Accessible | Risk Profile |
|------|----------|----------------|---------------------|-------------|
| **Orchestrator** | Plans, delegates, coordinates multi-step workflows | High — controls agent dispatch | All (read); limited write via delegation | Compromise = full workflow hijack |
| **Researcher** | Retrieval, web search, document analysis | Medium — read-heavy | Web Search MCP, Memory MCP (read), Database MCP (read-only) | Prompt injection via web content |
| **Executor** | Code execution, file operations, API calls | Critical — write access to systems | Code Execution MCP, Filesystem MCP, Database MCP (read/write), API Gateway MCP | Highest blast radius; sandbox is primary control |
| **Critic/Validator** | Output review, guardrail enforcement | Medium — read + policy enforcement | Memory MCP (read), Database MCP (read-only) | Bypass = guardrails disabled |
| **Notifier** | External communications (email, Slack, webhooks) | High — external-facing actions | Communication MCP | Exfiltration vector; irreversible actions |

### MCP Server Inventory (L3 + L4)

| MCP Server | Tools Exposed | Trust Level | Transport | Primary Threats |
|------------|--------------|-------------|-----------|-----------------|
| **Filesystem MCP** | `read_file`, `write_file`, `list_directory`, `delete_file` | High-risk | stdio / HTTP | Path traversal, unauthorized file access, data exfiltration |
| **Code Execution MCP** | `run_python`, `run_bash`, `run_sql` | Critical-risk | HTTP/SSE | Remote code execution, sandbox escape, resource exhaustion |
| **Database MCP** | `query`, `insert`, `update`, `schema_inspect` | High-risk | HTTP/SSE | SQL injection via prompt, data exfiltration, unauthorized modification |
| **Web Search MCP** | `search`, `fetch_url` | Medium-risk | HTTP/SSE | SSRF, prompt injection via web content, data leakage in queries |
| **API Gateway MCP** | `call_external_api` | High-risk | HTTP/SSE | Credential exposure, SSRF, unauthorized API access |
| **Communication MCP** | `send_email`, `post_slack`, `create_ticket` | High-risk | HTTP/SSE | Data exfiltration, social engineering, spam, irreversible actions |
| **Memory MCP** | `store_memory`, `retrieve_memory`, `delete_memory` | Medium-risk | stdio / HTTP | Memory poisoning, state manipulation, cross-session data leakage |

### Data Stores (L2)

| Store | Type | Data Held | Persistence | Access Pattern |
|-------|------|-----------|-------------|----------------|
| Vector DB (Weaviate/Qdrant/Pinecone) | Vector store | Embeddings, RAG chunks | Persistent | Researcher agent reads; RAG pipeline writes |
| PostgreSQL | Relational DB | Structured metadata, long-term agent memory | Persistent | Multiple agents via Database MCP |
| Redis | In-memory store | Session state, conversation buffers | TTL-bound | All agents (session-scoped) |
| Object Storage (S3-compatible) | Blob store | Documents, files, artifacts | Persistent | Filesystem MCP, RAG pipeline |
| Internal Relational DBs | Relational DB | Business data (deployment-specific) | Persistent | Executor, Researcher via Database MCP |

---

## Inter-Agent Communication Patterns

| Pattern | Mechanism | Agents Involved | Security Properties |
|---------|-----------|----------------|---------------------|
| **Direct Call** | Synchronous invocation within framework | Orchestrator → all subagents | In-process; no network traversal. Trust depends on framework integrity |
| **Shared State (Blackboard)** | Redis/DB read-write | All agents | Concurrent access; requires state isolation. Poisoned state affects all readers |
| **Message Bus (Async)** | Event-driven queue | Executor ↔ Notifier, cross-workflow triggers | Decoupled; requires message authentication. Replay and injection risks |

---

## Cross-Layer Components

Several components span multiple MAESTRO layers:

| Component | Layers | Notes |
|-----------|--------|-------|
| MCP servers | L3 (tool definitions) + L4 (runtime containers) + L6 (access control) | Tool logic at L3, container isolation at L4, RBAC enforcement at L6 |
| Agent service accounts | L3 (agent identity) + L4 (K8s service accounts) + L6 (Vault credentials) | Identity originates at L6, propagates through L4, used at L3 |
| HITL approval queue | L3 (tool call interception) + L5 (review UI) + L6 (policy enforcement) | Policy defined at L6, enforced at L3, reviewed at L5 |
| RAG pipeline | L1 (embedding model) + L2 (vector DB, document store) + L3 (retrieval tool) | Data ingested at L2, embeddings via L1, served to agents at L3 |
| A2A cross-platform calls | L3 (agent framework) + L7 (external agents) + L6 (trust policy) | Untrusted boundary; requires input validation and output sanitization |
| Centralized logging | L4 (infrastructure) + L5 (observability) + L6 (audit compliance) | All layers emit logs; aggregation at L5; retention policy at L6 |

---

## Critical Data Flows

| # | Flow | Path | Sensitivity | Security Controls |
|---|------|------|-------------|-------------------|
| DF-1 | User prompt → Orchestrator → LLM | User → API Gateway → Orchestrator pod → LLM API | Confidential (may contain PII, business context) | TLS in transit, OIDC auth, input sanitization |
| DF-2 | LLM response → Tool invocation → MCP server | LLM API → Agent pod → MCP server container | Varies (up to Restricted) | mTLS (HTTP), RBAC tool-level check, HITL gate for high-risk tools |
| DF-3 | RAG retrieval | Agent → Vector DB → Embedding model → Agent | Confidential (business documents) | Network policy, DB auth, embedding model access control |
| DF-4 | Agent → External service (email, Slack, API) | Notifier/Executor → Communication/API Gateway MCP → External | Restricted (may contain PII, credentials) | HITL approval, credential via Vault, egress firewall rules |
| DF-5 | Agent ↔ Agent (shared state) | Agent pod → Redis/PostgreSQL → Agent pod | Confidential (intermediate reasoning, plans) | Network policy, DB auth, no encryption at rest by default (gap) |
| DF-6 | A2A cross-platform | Agent → A2A protocol/webhook → External agent platform | Untrusted (bidirectional) | Input validation, output sanitization, no implicit trust |
| DF-7 | Audit logs | All components → OpenTelemetry collector → Log backend | Confidential (contains prompts, responses, actions) | TLS in transit, RBAC on log access, retention policy |
| DF-8 | Secrets retrieval | Agent pod → Vault Agent → HashiCorp Vault | Restricted (credentials, API keys) | mTLS, short-lived dynamic secrets, Vault audit log |

---

## Architecture Assumptions

| ID | Assumption | Rationale | Impact if Wrong | Validation Method | Status |
|----|-----------|-----------|----------------|-------------------|--------|
| A7 | Kubernetes network policies effectively isolate agent pods and MCP server pods | Container-per-role architecture relies on network segmentation | Lateral movement between agents; compromised Executor reaches all MCP servers | Penetration test, network policy audit (Phase 8) | Unvalidated |
| A8 | MCP server containers have no outbound internet access except where explicitly required (Web Search, Communication, API Gateway) | Egress firewall is configured per container | Data exfiltration via any MCP server with internet access | Network policy review (Phase 8) | Unvalidated |
| A9 | Code Execution MCP sandbox prevents container escape and host access | Sandbox is the primary control for the highest-risk MCP server | Full cluster compromise via sandbox escape | Sandbox security audit, penetration test | Unvalidated |
| A10 | HITL approval queue timeout defaults to auto-reject (fail-safe) | Stated design; prevents unapproved actions on timeout | If fail-open, timeout bypasses human review entirely | Configuration audit (Phase 8) | Unvalidated |
| A11 | All LLM API calls use TLS and do not log full prompts to vendor-side telemetry beyond contractual terms | Vendor API data handling per contractual agreement | Restricted data exposed to LLM vendor beyond agreed scope | Vendor contract review, DPA audit | Unvalidated |
| A12 | Redis session store enforces TTL and does not persist session data beyond configured timeout | Ephemeral session design | Stale session data persists; cross-session data leakage | Configuration audit (Phase 8) | Unvalidated |
| A13 | A2A cross-platform agent calls are treated as untrusted by default | Stated architecture principle | If trusted by default, external agents can inject commands or poisoned data | Code review of A2A handler (Phase 8) | Unvalidated |
| A14 | mTLS is enforced on all HTTP-transport MCP connections in production | Security architecture specification | Without mTLS, agent↔MCP traffic can be intercepted or spoofed within the cluster | Certificate and config audit (Phase 8) | Unvalidated |

---

## Architecture Diagram (Text)

```
                                    ┌─────────────────────────────────┐
                                    │         EXTERNAL ZONE           │
                                    │                                 │
                                    │  LLM Vendor APIs (L1)          │
                                    │  External SaaS (Slack, GitHub)  │
                                    │  Partner APIs                   │
                                    │  Public Web                     │
                                    │  External Agent Platforms (A2A) │
                                    └──────────────┬──────────────────┘
                                                   │ TLS / mTLS
                                    ┌──────────────▼──────────────────┐
                                    │       DMZ / API GATEWAY         │
                                    │  OIDC/OAuth 2.0 auth (L6)      │
                                    │  Rate limiting, WAF             │
                                    └──────────────┬──────────────────┘
                                                   │
                    ┌──────────────────────────────▼──────────────────────────────┐
                    │                    PLATFORM ZONE (K8s Cluster)               │
                    │                                                              │
                    │  ┌─────────────────────────────────────────────────────┐     │
                    │  │              AGENT PODS (L3)                        │     │
                    │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │     │
                    │  │  │Orchestratr│ │Researcher│ │ Executor │           │     │
                    │  │  └────┬─────┘ └────┬─────┘ └────┬─────┘           │     │
                    │  │  ┌────┴─────┐ ┌────┴─────┐                        │     │
                    │  │  │  Critic  │ │ Notifier │                        │     │
                    │  │  └──────────┘ └──────────┘                        │     │
                    │  └─────────────────────┬───────────────────────────────┘     │
                    │                        │ mTLS / stdio                         │
                    │  ┌─────────────────────▼───────────────────────────────┐     │
                    │  │           MCP SERVER PODS (L3 + L4)                 │     │
                    │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │     │
                    │  │  │ File │ │ Code │ │  DB  │ │ Web  │ │  API │    │     │
                    │  │  │System│ │ Exec │ │      │ │Search│ │Gateway│    │     │
                    │  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘    │     │
                    │  │  ┌──────┐ ┌──────┐                                │     │
                    │  │  │Comms │ │Memory│                                │     │
                    │  │  └──────┘ └──────┘                                │     │
                    │  └─────────────────────────────────────────────────────┘     │
                    │                        │                                     │
                    │  ┌─────────────────────▼───────────────────────────────┐     │
                    │  │              DATA LAYER (L2)                        │     │
                    │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │     │
                    │  │  │Vector DB │ │PostgreSQL│ │  Redis   │           │     │
                    │  │  └──────────┘ └──────────┘ └──────────┘           │     │
                    │  │  ┌──────────┐ ┌──────────┐                        │     │
                    │  │  │   S3     │ │Internal  │                        │     │
                    │  │  │ Storage  │ │   DBs    │                        │     │
                    │  │  └──────────┘ └──────────┘                        │     │
                    │  └─────────────────────────────────────────────────────┘     │
                    │                                                              │
                    │  ┌─────────────────────────────────────────────────────┐     │
                    │  │  OBSERVABILITY (L5)          SECURITY (L6)          │     │
                    │  │  OpenTelemetry Collector     HashiCorp Vault        │     │
                    │  │  Log Backend (ELK)           OIDC Provider          │     │
                    │  │  HITL Approval Queue UI      RBAC Policy Engine     │     │
                    │  │  Alerting/Dashboards         Certificate Authority  │     │
                    │  └─────────────────────────────────────────────────────┘     │
                    │                                                              │
                    │  ┌─────────────────────────────────────────────────────┐     │
                    │  │  GPU INFERENCE CLUSTER (L1 + L4)                    │     │
                    │  │  Self-hosted Llama 3 / Mistral                      │     │
                    │  │  Embedding model                                    │     │
                    │  └─────────────────────────────────────────────────────┘     │
                    └──────────────────────────────────────────────────────────────┘
```

---

*This threat model was generated with AI assistance using the OWASP MAESTRO Playbook. It must be reviewed by a qualified security professional before use in production risk decisions.*
