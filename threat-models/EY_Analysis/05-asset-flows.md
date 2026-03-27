# Phase 5: Asset Flow Analysis — EY_Analysis

**Project:** EY_Analysis
**Date:** 2026-03-27
**Analysis Depth:** Full

---

## Critical Asset Inventory

### AF1: User PII

| Field | Value |
|-------|-------|
| Asset ID | AF1 |
| Asset Name | User PII (Personally Identifiable Information) |
| Classification | **Restricted** |
| Created At | User input (TZ1 → TZ2 → TZ3); external data sources via API Gateway MCP (TZ9) |
| Stored At | PostgreSQL (long-term memory, TZ5); Redis (session state, TZ5); Vector DB (if embedded in RAG chunks, TZ5); Log backend (if logged in prompts/responses, TZ8) |
| Transmitted Via | HTTPS (user → DMZ → agent); mTLS (agent → MCP → DB); HTTPS (agent → vendor LLM API); HTTPS (Notifier → Slack/email via Communication MCP) |
| Processed By | Orchestrator (routing), Researcher (retrieval), Executor (DB queries), LLM (inference — both vendor API and self-hosted) |
| Destroyed | Session data: Redis TTL expiry. Long-term memory: manual deletion or retention policy. Logs: retention policy (e.g., 90 days). Vendor LLM API: per vendor DPA (varies). **Gap: no automatic PII expiry in PostgreSQL long-term memory.** |
| Trust Zones Traversed | TZ1 → TZ2 → TZ3 → TZ4 → TZ5 (storage); TZ3 → TZ9 (vendor LLM); TZ4 → TZ9 (external SaaS via Communication MCP) |
| Exposure Points | **EP1:** Vendor LLM API (PII in prompts sent to third-party). **EP2:** Log backend (PII in logged prompts/responses). **EP3:** Communication MCP (PII in outbound emails/Slack). **EP4:** Web Search MCP (PII could be encoded in search queries). **EP5:** Vector DB (PII embedded in RAG chunks — not easily deletable). |
| Protection Controls | TLS in transit (all hops); OIDC auth at entry; DB credentials via Vault; RBAC on DB access. **Gaps:** No automatic PII detection/masking before LLM calls; no PII scrubbing in logs; no right-to-erasure mechanism for PII embedded in vector DB. |

---

### AF2: Credentials and API Keys

| Field | Value |
|-------|-------|
| Asset ID | AF2 |
| Asset Name | Credentials, API Keys, Service Account Tokens |
| Classification | **Restricted** |
| Created At | HashiCorp Vault (TZ7); OIDC IdP (TZ7); Vendor API key provisioning (TZ9) |
| Stored At | Vault (primary, TZ7); Vault Agent sidecar (ephemeral injection into pod, TZ3/TZ4); K8s Secrets (if any — should be Vault-managed); MCP ConfigMap (endpoint URLs — should NOT contain credentials, but risk of config drift) |
| Transmitted Via | Vault Agent → pod (in-memory injection via mTLS); Agent → vendor LLM API (API key in HTTPS header); Agent → MCP server (mTLS client cert); MCP server → external API (API key in HTTPS header) |
| Processed By | Vault Agent (injection), Agent pods (API calls), MCP servers (external API calls) |
| Destroyed | Dynamic secrets: automatic TTL expiry in Vault. Static API keys: manual rotation. mTLS certs: CA-managed expiry. **Gap: vendor API keys may have long rotation cycles.** |
| Trust Zones Traversed | TZ7 → TZ3/TZ4 (injection); TZ3 → TZ9 (vendor API calls); TZ4 → TZ9 (external API calls); TZ4 → TZ5 (DB credentials) |
| Exposure Points | **EP6:** Pod environment (if secrets leak to env vars or container filesystem). **EP7:** Log backend (if API keys are logged in request headers or error messages). **EP8:** MCP ConfigMap (if credentials accidentally stored in non-secret ConfigMap). **EP9:** Vendor LLM API (API key transmitted in header — interceptable if TLS is compromised). **EP10:** Code Execution MCP (user-controlled code could read environment or filesystem for injected secrets). |
| Protection Controls | Vault dynamic secrets (short TTL); mTLS for Vault Agent; no secrets in env vars or config files (policy); Vault audit logging. **Gaps:** Code Execution MCP sandbox must prevent secret access from user code; log scrubbing for credentials not confirmed. |

---

### AF3: System Prompts (Business Logic)

| Field | Value |
|-------|-------|
| Asset ID | AF3 |
| Asset Name | System Prompts, Workflow Definitions, Guardrail Instructions |
| Classification | **Confidential** |
| Created At | Development team (source code / configuration, TZ7 via CI/CD); stored in K8s ConfigMap or agent framework config |
| Stored At | K8s ConfigMap (TZ4); Agent framework configuration (TZ3); Source code repository (TZ7); Possibly cached in Redis session state (TZ5) |
| Transmitted Via | CI/CD pipeline → K8s ConfigMap (TZ7 → TZ4); ConfigMap → agent pod at startup (TZ4 → TZ3); Agent pod → LLM API as system message (TZ3 → TZ6/TZ9) |
| Processed By | Agent framework (loads at startup), LLM (processes as system message), Critic/Validator (enforces guardrails) |
| Destroyed | Replaced on redeployment. Previous versions in source control history. **Never truly destroyed if source repo retains history.** |
| Trust Zones Traversed | TZ7 → TZ4 → TZ3 → TZ6 (self-hosted LLM) / TZ9 (vendor LLM) |
| Exposure Points | **EP11:** Vendor LLM API (system prompts sent as part of every request — vendor can read business logic). **EP12:** Log backend (system prompts logged with every LLM call). **EP13:** Operator access (ML/AI engineers and platform admins can read prompts). **EP14:** Prompt extraction attacks (adversary uses prompt injection to cause LLM to output its system prompt). **EP15:** ConfigMap (readable by any pod with K8s API access if RBAC is insufficient). |
| Protection Controls | Source code access controls; K8s RBAC on ConfigMap; TLS in transit. **Gaps:** No runtime protection against prompt extraction; system prompts are sent in plaintext to vendor LLM APIs; logged in full in observability stack. |

---

### AF4: RAG Knowledge Base

| Field | Value |
|-------|-------|
| Asset ID | AF4 |
| Asset Name | RAG Knowledge Base (Documents, Embeddings, Chunks) |
| Classification | **Confidential to Restricted** (varies by content — may contain trade secrets, internal documents, PII) |
| Created At | Document ingestion pipeline (TZ5 — S3 → chunking → embedding model → Vector DB); Internal databases (TZ5); External data feeds (TZ9) |
| Stored At | Object Storage / S3 (source documents, TZ5); Vector DB (embeddings + chunks, TZ5); PostgreSQL (metadata, TZ5) |
| Transmitted Via | S3 API → embedding pipeline (TZ5 internal); Embedding pipeline → self-hosted model (TZ5 → TZ6); Vector DB API (TZ4 → TZ5 via Database/Memory MCP); External data feed → ingestion pipeline (TZ9 → TZ5) |
| Processed By | Embedding model (TZ6), RAG pipeline (TZ5), Researcher agent (via MCP, TZ3), LLM (retrieved chunks included in context, TZ6/TZ9) |
| Destroyed | Document deletion from S3 removes source. **Gap: Embeddings in vector DB are not automatically deleted when source documents are removed.** Chunk metadata in PostgreSQL may persist. |
| Trust Zones Traversed | TZ9 (external feeds) → TZ5 (storage) → TZ6 (embedding) → TZ5 (vector DB) → TZ4 (MCP retrieval) → TZ3 (agent context) → TZ6/TZ9 (LLM inference) |
| Exposure Points | **EP16:** External data feed ingestion (poisoned data enters RAG pipeline — no content validation at TZ9 → TZ5 boundary). **EP17:** Vector DB (contains organizational IP; difficult to apply fine-grained access control on vector search results). **EP18:** LLM context (retrieved chunks sent to vendor LLM API — vendor can read organizational documents). **EP19:** Agent responses (LLM may include verbatim RAG content in responses visible to users with lower clearance). |
| Protection Controls | S3 bucket policies; Vector DB authentication; network isolation of data zone. **Gaps:** No content classification on RAG chunks; no access control per document sensitivity; no poisoning detection on ingested external data; no automatic sync between source document deletion and embedding deletion. |

---

### AF5: Agent Memory and Session State

| Field | Value |
|-------|-------|
| Asset ID | AF5 |
| Asset Name | Agent Memory (Short-term, Session, Long-term) |
| Classification | **Confidential** (contains intermediate reasoning, plans, user context, potentially PII) |
| Created At | Agent pods during task execution (TZ3) |
| Stored At | In-context buffer (ephemeral, TZ3 — lost on session end); Redis (session-scoped, TTL-bound, TZ5); PostgreSQL / Vector DB (long-term memory, persistent, TZ5) |
| Transmitted Via | Agent → Redis (session state writes, TZ3 → TZ5 via Memory MCP); Agent → PostgreSQL (long-term memory writes, TZ3 → TZ5 via Memory MCP); Agent ↔ Agent (shared state via Redis blackboard, TB8) |
| Processed By | All agents (read/write via Memory MCP); LLM (memory included in context for continuity) |
| Destroyed | In-context: session end. Session-scoped: Redis TTL expiry (configurable). Long-term: **manual deletion only — no automatic expiry policy by default.** |
| Trust Zones Traversed | TZ3 (creation) → TZ4 (Memory MCP) → TZ5 (storage) → TZ4 (retrieval) → TZ3 (consumption by any agent) → TZ6/TZ9 (included in LLM context) |
| Exposure Points | **EP20:** Shared state / blackboard (any agent can read any other agent's state — no per-agent isolation in Redis). **EP21:** Long-term memory poisoning (attacker writes false memories that persist across sessions). **EP22:** Cross-session data leakage (long-term memory from one user's session accessible in another's if no session isolation). **EP23:** LLM context (memory included in prompts — sent to vendor LLM). |
| Protection Controls | Redis TTL (session); DB credentials via Vault; TLS in transit. **Gaps:** No per-agent or per-user isolation in shared state; no integrity validation on memory reads; no automatic expiry for long-term memory; no encryption at rest for Redis by default. |

---

### AF6: Audit Logs

| Field | Value |
|-------|-------|
| Asset ID | AF6 |
| Asset Name | Audit Logs (LLM calls, tool invocations, agent actions, HITL decisions) |
| Classification | **Confidential** (contains prompts, responses, PII, business logic) |
| Created At | All components emit logs (TZ3, TZ4, TZ5, TZ6, TZ7, TZ8) |
| Stored At | OpenTelemetry collector (TZ8); Log backend / ELK (TZ8); Vault audit log (TZ7); K8s audit log (TZ7) |
| Transmitted Via | OpenTelemetry protocol (TZ3/TZ4 → TZ8); Vault audit → log backend (TZ7 → TZ8) |
| Processed By | Log backend (indexing, search); Alerting engine (anomaly detection); HITL UI (approval queue display); Security/Compliance team (audit review) |
| Destroyed | Retention policy (e.g., 90 days for operational logs, 1 year+ for compliance). **Gap: retention policy not enforced automatically in generic baseline.** |
| Trust Zones Traversed | All zones → TZ8 (collection); TZ8 → Security/Compliance team (read access) |
| Exposure Points | **EP24:** Log backend contains full prompts and responses — effectively a copy of all sensitive data that passed through the system. **EP25:** Log access by operators (broader access than needed if RBAC on logs is coarse). **EP26:** Log exfiltration (compromise of ELK = compromise of all historical data). **EP27:** PII in logs creates GDPR right-to-erasure compliance challenge. |
| Protection Controls | TLS in transit; RBAC on log access (ReadOnly, Security roles); separate observability zone. **Gaps:** No PII masking in logs; no field-level encryption; logs contain credentials if error messages include headers; no automatic retention enforcement. |

---

### AF7: Model Weights (Self-Hosted)

| Field | Value |
|-------|-------|
| Asset ID | AF7 |
| Asset Name | Self-Hosted Model Weights (Llama 3, Mistral, Embedding Models) |
| Classification | **Confidential** (organizational investment; fine-tuned variants would be Restricted) |
| Created At | Downloaded from model registries (Hugging Face, etc.) during deployment; fine-tuned variants created during training runs |
| Stored At | GPU inference nodes (local filesystem, TZ6); Model registry / artifact store (TZ7 via CI/CD) |
| Transmitted Via | Model registry → GPU nodes (deployment pipeline, TZ7 → TZ6); Not transmitted at runtime (inference is local) |
| Processed By | GPU inference endpoints (TZ6) |
| Destroyed | Replaced on model update. Previous versions in artifact store. |
| Trust Zones Traversed | TZ7 (artifact store) → TZ6 (inference cluster) |
| Exposure Points | **EP28:** Model supply chain (downloading from public registries — poisoned model weights). **EP29:** GPU node compromise (attacker with node access can exfiltrate weights). **EP30:** Model API responses could leak training data via extraction attacks. |
| Protection Controls | Separate VPC for inference cluster; mTLS access; restricted node access. **Gaps:** No model integrity verification (hash checking) at deployment; no runtime model extraction detection. |

---

### AF8: MCP Server Configuration

| Field | Value |
|-------|-------|
| Asset ID | AF8 |
| Asset Name | MCP Server Configuration (Endpoint URLs, Tool Definitions, Runtime Parameters) |
| Classification | **Confidential** (controls agent behavior; credentials component is Restricted) |
| Created At | Development team (source code / CI/CD pipeline, TZ7) |
| Stored At | K8s ConfigMap (non-secret config, TZ4); Vault (endpoint URLs with embedded credentials, TZ7) |
| Transmitted Via | CI/CD → K8s API (ConfigMap update, TZ7 → TZ4); Vault → MCP server pod (credential injection, TZ7 → TZ4) |
| Processed By | MCP server containers at startup (TZ4); K8s controller (ConfigMap distribution) |
| Destroyed | Replaced on redeployment. Previous versions in CI/CD history. |
| Trust Zones Traversed | TZ7 → TZ4 (deployment); TZ4 → MCP servers (runtime) |
| Exposure Points | **EP31:** ConfigMap readable by any pod with K8s API access (if RBAC insufficient). **EP32:** Config drift — unauthorized modification of ConfigMap redirects MCP servers to rogue backends (TB12). **EP33:** CI/CD pipeline compromise enables attacker to inject malicious MCP config at deployment. |
| Protection Controls | K8s RBAC; Vault for credentials; CI/CD pipeline auth. **Gaps:** No integrity verification on ConfigMap content; no automated config drift detection; runtime reconfiguration without image rebuild increases change surface. |

---

## Asset-to-Trust-Zone Exposure Matrix

| Asset | TZ1 Public | TZ2 DMZ | TZ3 Agent | TZ4 MCP | TZ5 Data | TZ6 Model | TZ7 Privileged | TZ8 Observability | TZ9 External |
|-------|-----------|---------|-----------|---------|----------|-----------|----------------|-------------------|-------------|
| AF1 PII | Origin | Transit | Process | Transit | **Store** | Process | — | **Store** (logs) | **Exposed** (LLM API, SaaS) |
| AF2 Credentials | — | — | Ephemeral | Ephemeral | — | — | **Store** (Vault) | **Risk** (log leak) | Transit (API headers) |
| AF3 System Prompts | — | — | Process | Transit | Cache | Process | **Store** (source) | **Store** (logs) | **Exposed** (LLM API) |
| AF4 RAG Knowledge | — | — | Process | Transit | **Store** | Process (embed) | — | — | **Exposed** (LLM API); Ingest (feeds) |
| AF5 Agent Memory | — | — | Process | Transit | **Store** | — | — | — | **Exposed** (LLM context) |
| AF6 Audit Logs | — | — | Emit | Emit | — | — | Emit (Vault) | **Store** | — |
| AF7 Model Weights | — | — | — | — | — | **Store** | Store (artifacts) | — | Origin (download) |
| AF8 MCP Config | — | — | — | **Store**/Process | — | — | **Store** (source) | — | — |

**Legend:** Origin = created here; Transit = passes through; Process = used/transformed; Store = persisted; Exposed = leaves organizational boundary; Emit = generates data; Risk = unintended exposure possible; Cache = temporary storage.

---

## Data Lineage Gaps

| # | Gap | Affected Assets | Risk |
|---|-----|----------------|------|
| DLG-1 | No automatic PII detection or masking before data is sent to vendor LLM APIs | AF1, AF4, AF5 | Restricted PII transmitted to third-party environments without classification-aware controls |
| DLG-2 | No sync between source document deletion and vector DB embedding deletion | AF4 | "Ghost" data persists in RAG after source is removed — GDPR right-to-erasure violation |
| DLG-3 | No per-agent or per-user isolation in shared Redis state | AF5 | Cross-user data leakage; cross-agent state poisoning |
| DLG-4 | Audit logs contain full prompts/responses without PII masking | AF1, AF3, AF6 | Log backend becomes a concentrated exfiltration target; GDPR compliance challenge |
| DLG-5 | No model integrity verification during deployment | AF7 | Poisoned model weights could be deployed without detection |
| DLG-6 | No automated config drift detection for MCP ConfigMaps | AF8 | Rogue MCP endpoint redirection goes undetected |
| DLG-7 | No automatic expiry for long-term agent memory in PostgreSQL | AF5 | Stale and potentially sensitive memory accumulates indefinitely |
| DLG-8 | Credential exposure detection gap in Code Execution MCP sandbox | AF2 | User-submitted code could read injected secrets if sandbox isolation is incomplete |

---

## Assumptions

| ID | Assumption | Rationale | Impact if Wrong | Validation Method | Status |
|----|-----------|-----------|----------------|-------------------|--------|
| A21 | Redis does not encrypt data at rest in the generic baseline | Common default for Redis deployments | Session state (AF5) containing PII/context is stored unencrypted — accessible if Redis node is compromised | Redis configuration audit | Unvalidated |
| A22 | Vector DB does not support per-document access control (ACL per embedding) | Limitation of most vector DB implementations | Researcher agent retrieves all matching documents regardless of the requesting user's clearance level | Vector DB feature review | Unvalidated |
| A23 | Log retention policy exists but is not automatically enforced | Common operational gap | Logs accumulate indefinitely, increasing blast radius of log backend compromise | Log backend configuration audit | Unvalidated |
| A24 | PII is not automatically detected or masked in any data flow | No PII detection tooling assumed in generic baseline | PII flows uncontrolled through all trust zones and persists in logs, memory, and LLM vendor systems | Architecture review for PII tooling | Unvalidated |

---

*This threat model was generated with AI assistance using the OWASP MAESTRO Playbook. It must be reviewed by a qualified security professional before use in production risk decisions.*
