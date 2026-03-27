# Phase 4: Trust Boundary Analysis — EY_Analysis

**Project:** EY_Analysis
**Date:** 2026-03-27
**Analysis Depth:** Full

---

## Trust Zone Definitions

### TZ1: Public / Untrusted Zone

**Trust Level:** None
**Components:**
- External end users (browser, mobile, CLI)
- Public internet
- Public web content (ingested via Web Search MCP)
- Public/open APIs (third-party, no trust agreement)
- External agent platforms (A2A inbound)

**Security Posture:** All input from this zone is treated as adversarial. No authentication or authorization is assumed until verified at the DMZ boundary.

---

### TZ2: DMZ / Semi-Trusted Zone

**Trust Level:** Low
**Components:**
- API Gateway / Load Balancer (ingress controller)
- WAF (Web Application Firewall)
- Rate limiting layer
- OIDC/OAuth 2.0 token validation endpoint

**Security Posture:** Authenticates and rate-limits inbound traffic. Does not process business logic. Terminates TLS. First point of input validation.

---

### TZ3: Agent Execution Zone

**Trust Level:** Medium
**Components:**
- Orchestrator agent pod
- Researcher agent pod
- Executor agent pod
- Critic/Validator agent pod
- Notifier agent pod
- Agent framework runtime (LangGraph / AutoGen)
- In-context conversation buffers (ephemeral)

**Security Posture:** Agents operate under scoped service accounts. Each agent in a separate container with K8s network policies restricting lateral movement. Agents are not fully trusted — they process untrusted user input and LLM outputs, making them susceptible to prompt injection.

---

### TZ4: MCP Server Zone

**Trust Level:** Medium-High
**Components:**
- Filesystem MCP server (container)
- Code Execution MCP server (sandboxed container)
- Database MCP server (container)
- Web Search MCP server (container)
- API Gateway MCP server (container)
- Communication MCP server (container)
- Memory MCP server (container)

**Security Posture:** Each MCP server in its own container with resource limits. Agent→MCP communication secured via mTLS (HTTP transport) or process-level isolation (stdio). MCP servers have direct access to backend resources — they are the enforcement point for tool-level permissions. ConfigMap (non-secret) + Vault (credentials) for configuration.

---

### TZ5: Data Zone

**Trust Level:** High
**Components:**
- PostgreSQL (structured metadata, long-term agent memory)
- Redis (session-scoped state)
- Vector DB (Weaviate/Qdrant/Pinecone)
- Object Storage (S3-compatible)
- Internal relational databases (business data)

**Security Posture:** Network-isolated from agent pods — agents cannot reach data stores directly; access is mediated by MCP servers (Database MCP, Memory MCP, Filesystem MCP). DB credentials are dynamic (Vault, short TTL). Encryption at rest assumed for production.

---

### TZ6: Model Inference Zone

**Trust Level:** High
**Components:**
- Self-hosted GPU inference cluster (Llama 3, Mistral)
- Embedding model endpoints
- Classification model endpoints

**Security Posture:** Separate VPC/subnet from agent pod network. Cross-boundary access via internal load balancer with mTLS only. Agents never reach GPU nodes directly. Model weights stored locally on inference nodes.

---

### TZ7: Privileged / Control Plane Zone

**Trust Level:** Critical
**Components:**
- HashiCorp Vault (secrets management)
- Kubernetes control plane (API server, etcd)
- OIDC Identity Provider
- Certificate Authority (mTLS issuance)
- RBAC policy engine
- CI/CD pipeline (deployment pipeline)

**Security Posture:** Highest trust zone. Access restricted to Platform Admins (3–5 people). All access logged and audited. Vault Agent injects secrets into pods — pods do not directly access Vault API. K8s RBAC restricts control plane access by role.

---

### TZ8: Observability Zone

**Trust Level:** Medium-High
**Components:**
- OpenTelemetry collector
- Log backend (ELK stack)
- Dashboards and alerting
- HITL approval queue UI (standalone web app)
- Anomaly detection engine (rule-based)

**Security Posture:** HITL UI is a separate service outside the agent cluster — operators cannot reach agent internals from the UI. Reads from approval queue via authenticated API. Log backend contains sensitive data (prompts, responses) — access restricted by RBAC (ReadOnly, Security/Compliance roles).

---

### TZ9: External Integration Zone

**Trust Level:** Varies (Low to Medium)
**Components:**
- Vendor LLM APIs (OpenAI, Anthropic, Google) — Medium trust (contractual SLA)
- SaaS integrations (Slack, SendGrid, GitHub, Jira) — Medium trust (vendor SaaS tier)
- Partner APIs — Low-Medium trust (bilateral agreement, limited scope)
- Managed message bus (AWS SQS / Azure Service Bus) — Medium trust (cloud provider)
- Public/open APIs — Low trust (untrusted, output treated as adversarial)

**Security Posture:** Three trust tiers applied. All outbound calls authenticated (API keys from Vault). All inbound responses treated as potentially adversarial (especially public APIs and web content). Egress firewall rules restrict which MCP servers can reach which external endpoints.

---

## Trust Boundary Crossings

### TB1: Public Internet → DMZ

| Field | Value |
|-------|-------|
| Boundary ID | TB1 |
| Source Zone | TZ1 (Public / Untrusted) |
| Destination Zone | TZ2 (DMZ) |
| Crossing Components | End user browser/client → API Gateway / Load Balancer |
| Protocol | HTTPS (TLS 1.3) |
| Authentication | OIDC/OAuth 2.0 token (validated at gateway) |
| Authorization | Token scope validation; rate limiting per client |
| Input Validation | WAF rules (OWASP CRS); request schema validation |
| Encryption in Transit | Yes (TLS) |
| Logging | Yes (access logs, auth events) |
| Boundary Strength | **Strong** |

---

### TB2: DMZ → Agent Execution Zone

| Field | Value |
|-------|-------|
| Boundary ID | TB2 |
| Source Zone | TZ2 (DMZ) |
| Destination Zone | TZ3 (Agent Execution) |
| Crossing Components | API Gateway → Orchestrator agent pod |
| Protocol | HTTPS (internal TLS) |
| Authentication | Forwarded JWT with user identity claims |
| Authorization | RBAC role check (User, Operator, Admin) |
| Input Validation | Partial — gateway validates schema; agent receives sanitized but semantically unchecked user prompt |
| Encryption in Transit | Yes |
| Logging | Yes (request ID, user identity, timestamp, agent target) |
| Boundary Strength | **Moderate** — semantic content of user prompts is not validated at this boundary (prompt injection passes through) |

---

### TB3: Agent Execution Zone → MCP Server Zone

| Field | Value |
|-------|-------|
| Boundary ID | TB3 |
| Source Zone | TZ3 (Agent Execution) |
| Destination Zone | TZ4 (MCP Server) |
| Crossing Components | Agent pod → MCP server container (per-tool) |
| Protocol | HTTP/SSE with mTLS (cloud); stdio (local/sidecar) |
| Authentication | mTLS client certificate (HTTP); process-level trust (stdio) |
| Authorization | Tool-level RBAC — each MCP tool mapped to minimum required agent role |
| Input Validation | MCP server MUST validate all inputs; varies by implementation |
| Encryption in Transit | Yes (mTLS for HTTP); N/A (stdio — same host) |
| Logging | Yes (tool name, parameters, agent identity, timestamp, response status) |
| Boundary Strength | **Moderate** — mTLS and RBAC are strong, but tool-level input validation quality varies across 7 MCP servers; stdio transport lacks network-level audit |

---

### TB4: MCP Server Zone → Data Zone

| Field | Value |
|-------|-------|
| Boundary ID | TB4 |
| Source Zone | TZ4 (MCP Server) |
| Destination Zone | TZ5 (Data) |
| Crossing Components | Database MCP → PostgreSQL; Memory MCP → Redis / Vector DB; Filesystem MCP → S3 |
| Protocol | PostgreSQL wire protocol (TLS), Redis protocol (TLS), S3 API (HTTPS) |
| Authentication | Dynamic credentials from Vault (short TTL) |
| Authorization | DB-level role permissions scoped per MCP server; S3 bucket policies |
| Input Validation | Parameterized queries (Database MCP); path validation (Filesystem MCP) |
| Encryption in Transit | Yes |
| Logging | Yes (query logs, access logs) |
| Boundary Strength | **Strong** — dynamic credentials, scoped permissions, parameterized queries |

---

### TB5: Agent Execution Zone → Model Inference Zone

| Field | Value |
|-------|-------|
| Boundary ID | TB5 |
| Source Zone | TZ3 (Agent Execution) |
| Destination Zone | TZ6 (Model Inference) |
| Crossing Components | Agent pod → Internal load balancer → GPU inference endpoint |
| Protocol | HTTPS (internal) with mTLS |
| Authentication | mTLS client certificate |
| Authorization | Service account identity; inference endpoint access policy |
| Input Validation | Prompt length limits; token budget enforcement |
| Encryption in Transit | Yes (mTLS) |
| Logging | Yes (prompt, response, token count, latency, model version) |
| Boundary Strength | **Strong** — separate VPC, mTLS, agents never reach GPU nodes directly |

---

### TB6: Agent Execution Zone → External Integration Zone (Vendor LLM APIs)

| Field | Value |
|-------|-------|
| Boundary ID | TB6 |
| Source Zone | TZ3 (Agent Execution) |
| Destination Zone | TZ9 (External Integration — Vendor LLM) |
| Crossing Components | Agent pod → Vendor LLM API (OpenAI, Anthropic, Google) |
| Protocol | HTTPS (TLS 1.3) |
| Authentication | API key (from Vault, short-lived where supported) |
| Authorization | Vendor-side API key scope (model access, rate limits) |
| Input Validation | Token budget enforcement; PII scrubbing (if configured) |
| Encryption in Transit | Yes |
| Logging | Yes (prompt hash, token count, latency, cost — full prompt logged internally, not sent to vendor telemetry beyond contractual terms) |
| Boundary Strength | **Moderate** — TLS and API key auth are standard, but sensitive data (prompts containing Restricted info) crosses to a third-party environment; data handling depends on vendor DPA |

---

### TB7: MCP Server Zone → External Integration Zone (SaaS / Partner / Public APIs)

| Field | Value |
|-------|-------|
| Boundary ID | TB7 |
| Source Zone | TZ4 (MCP Server) |
| Destination Zone | TZ9 (External Integration) |
| Crossing Components | Communication MCP → Slack/SendGrid/Teams; API Gateway MCP → GitHub/Jira/CI-CD; Web Search MCP → Public web |
| Protocol | HTTPS |
| Authentication | API keys / OAuth tokens (from Vault) per service |
| Authorization | Scoped API permissions per integration; egress firewall rules per MCP server |
| Input Validation | Output sanitization before sending (varies by MCP server) |
| Encryption in Transit | Yes |
| Logging | Yes (destination, payload summary, response status) |
| Boundary Strength | **Moderate** — egress firewall and Vault-managed credentials are strong, but Communication MCP and API Gateway MCP can exfiltrate data via legitimate channels; Web Search MCP ingests untrusted content |

---

### TB8: Agent ↔ Agent (Inter-Agent Communication)

| Field | Value |
|-------|-------|
| Boundary ID | TB8 |
| Source Zone | TZ3 (Agent Execution — Agent A) |
| Destination Zone | TZ3 (Agent Execution — Agent B) |
| Crossing Components | Orchestrator ↔ Subagents (direct call); Any agent ↔ Redis/DB (shared state); Any agent ↔ Managed message bus (async) |
| Protocol | In-process (direct call); Redis/PostgreSQL protocol (shared state); SQS/Service Bus SDK (async) |
| Authentication | Direct call: framework-internal (no network auth); Shared state: DB credentials; Message bus: IAM role-based |
| Authorization | Direct call: framework dispatching logic; Shared state: no per-field authorization; Message bus: queue-level IAM |
| Input Validation | **No** — agents implicitly trust messages from other agents within the framework |
| Encryption in Transit | Direct call: N/A (in-process); Shared state: TLS to DB; Message bus: TLS (managed service) |
| Logging | Direct call: framework-level trace; Shared state: partial (DB query logs); Message bus: yes (message metadata) |
| Boundary Strength | **Weak** — inter-agent messages are implicitly trusted; no input validation, no per-message authentication; compromised agent can poison shared state or inject messages |

---

### TB9: External Agent Platform → Agent Execution Zone (A2A Inbound)

| Field | Value |
|-------|-------|
| Boundary ID | TB9 |
| Source Zone | TZ1 (Public / Untrusted) or TZ9 (External Integration) |
| Destination Zone | TZ3 (Agent Execution) |
| Crossing Components | External agent platform → A2A protocol handler / webhook endpoint → Agent pod |
| Protocol | HTTPS (A2A protocol / webhook) |
| Authentication | API key or mutual TLS (depends on integration) |
| Authorization | Scoped to specific agent roles and tool subsets |
| Input Validation | All inbound A2A messages treated as untrusted input — schema validation, content sanitization |
| Encryption in Transit | Yes (TLS) |
| Logging | Yes (source platform, message content, agent target, timestamp) |
| Boundary Strength | **Moderate** — explicit untrust assumption is good, but A2A protocol is relatively new and input validation maturity may vary |

---

### TB10: Agent Execution Zone → Observability Zone (HITL)

| Field | Value |
|-------|-------|
| Boundary ID | TB10 |
| Source Zone | TZ3 (Agent Execution) |
| Destination Zone | TZ8 (Observability) |
| Crossing Components | Agent pod → Approval queue (write); HITL UI → Approval queue (read/approve/reject) |
| Protocol | Internal API (HTTPS) |
| Authentication | Agent: service account; HITL UI: operator OIDC session |
| Authorization | Agents can only enqueue; operators can only approve/reject; neither can modify queue entries |
| Input Validation | Queue entries are structured (tool name, parameters, agent identity) — schema-validated |
| Encryption in Transit | Yes |
| Logging | Yes (full audit: who approved, when, what tool call, decision) |
| Boundary Strength | **Strong** — separation of duties enforced; HITL UI cannot reach agent internals; timeout defaults to auto-reject |

---

### TB11: Pods → Privileged Zone (Secrets)

| Field | Value |
|-------|-------|
| Boundary ID | TB11 |
| Source Zone | TZ3 (Agent) / TZ4 (MCP Server) |
| Destination Zone | TZ7 (Privileged) |
| Crossing Components | Pod → Vault Agent sidecar → HashiCorp Vault |
| Protocol | Vault Agent API (HTTPS with mTLS) |
| Authentication | Vault Agent uses K8s service account token (Vault K8s auth method) |
| Authorization | Vault policies scoped per service account — least privilege |
| Input Validation | N/A (secret retrieval, not arbitrary input) |
| Encryption in Transit | Yes (mTLS) |
| Logging | Yes (Vault audit log — every secret access logged) |
| Boundary Strength | **Strong** — pods never access Vault API directly; Vault Agent mediates; dynamic secrets with short TTL |

---

### TB12: MCP Config Update Path

| Field | Value |
|-------|-------|
| Boundary ID | TB12 |
| Source Zone | TZ7 (Privileged — CI/CD / Admin) |
| Destination Zone | TZ4 (MCP Server) |
| Crossing Components | CI/CD pipeline or Admin → K8s ConfigMap + Vault → MCP server container |
| Protocol | K8s API (HTTPS), Vault API (HTTPS) |
| Authentication | K8s RBAC (admin), Vault token (CI/CD service account) |
| Authorization | K8s RBAC restricts ConfigMap updates; Vault policy restricts credential writes |
| Input Validation | ConfigMap schema validation (if enforced); no automatic integrity check on MCP config content |
| Encryption in Transit | Yes |
| Logging | Yes (K8s audit log, Vault audit log) |
| Boundary Strength | **Moderate** — auth and logging are strong, but ConfigMap content is not integrity-checked; a compromised CI/CD pipeline or admin can redirect MCP servers to rogue endpoints without detection unless config drift monitoring is in place |

---

## Implicit Trust Assumptions (High-Risk)

| # | Implicit Trust | Risk | Affected Boundary |
|---|---------------|------|-------------------|
| IT-1 | Agents implicitly trust messages from other agents within the framework | Compromised agent can inject poisoned messages to all other agents via shared state or direct call | TB8 |
| IT-2 | MCP tool descriptions are trusted by the LLM for tool selection | Rogue or compromised MCP server can manipulate agent behavior via misleading tool descriptions | TB3 |
| IT-3 | Shared Redis/blackboard state is trusted by all agents without per-field validation | Any agent with write access can poison state consumed by all other agents | TB8 |
| IT-4 | ConfigMap content is trusted by MCP servers at startup without integrity verification | Compromised config can redirect MCP servers to rogue backends | TB12 |
| IT-5 | LLM vendor API responses are trusted as legitimate model outputs | Compromised vendor API (supply chain) or MITM could inject malicious responses | TB6 |
| IT-6 | Stderr/stdout for stdio-transport MCP servers is trusted without message authentication | Local process compromise can inject tool responses | TB3 |

---

## Agent Boundary Crossing Analysis

| Agent Role | Boundaries Crossed | Identity Propagation | Privilege Escalation Risk |
|-----------|-------------------|---------------------|--------------------------|
| **Orchestrator** | TB2 (receives user request), TB3 (dispatches to MCP), TB5/TB6 (LLM inference), TB8 (delegates to subagents) | Carries user identity from JWT; operates under its own service account for tool calls | **High** — confused deputy: user with "User" role triggers Orchestrator which operates with "Operator"-equivalent service account permissions |
| **Researcher** | TB3 (Web Search MCP, Memory MCP), TB5/TB6 (LLM inference), TB8 (returns results to Orchestrator) | Inherits session context from Orchestrator; uses own service account for MCP | **Medium** — can exfiltrate data via search queries (encoding sensitive data in web search terms) |
| **Executor** | TB3 (Code Exec MCP, Filesystem MCP, Database MCP, API Gateway MCP), TB5/TB6 (LLM inference), TB8 (receives tasks from Orchestrator) | Uses highest-privilege service account among all agents | **Critical** — broadest MCP tool access; sandbox escape = cluster compromise |
| **Critic/Validator** | TB3 (Memory MCP read-only, Database MCP read-only), TB5/TB6 (LLM inference), TB8 (reviews outputs from other agents) | Read-only service account | **Low** — but compromise removes guardrails, enabling other agents to act unchecked |
| **Notifier** | TB3 (Communication MCP), TB7 (external SaaS), TB8 (receives from Orchestrator) | Uses service account with external communication permissions | **High** — can exfiltrate data via legitimate outbound channels (email, Slack) |

---

## Boundary Strength Summary

| Boundary | Strength | Key Gap |
|----------|----------|---------|
| TB1 (Internet → DMZ) | **Strong** | — |
| TB2 (DMZ → Agents) | **Moderate** | Prompt injection passes through semantic gap |
| TB3 (Agents → MCP) | **Moderate** | Input validation quality varies; stdio lacks audit |
| TB4 (MCP → Data) | **Strong** | — |
| TB5 (Agents → Self-hosted LLM) | **Strong** | — |
| TB6 (Agents → Vendor LLM) | **Moderate** | Restricted data sent to third-party |
| TB7 (MCP → External SaaS) | **Moderate** | Exfiltration via legitimate channels |
| TB8 (Agent ↔ Agent) | **Weak** | No inter-agent input validation or message auth |
| TB9 (A2A Inbound) | **Moderate** | Protocol maturity; input validation depth |
| TB10 (Agents → HITL) | **Strong** | — |
| TB11 (Pods → Vault) | **Strong** | — |
| TB12 (Config → MCP) | **Moderate** | No config integrity verification |

---

## Assumptions

| ID | Assumption | Rationale | Impact if Wrong | Validation Method | Status |
|----|-----------|-----------|----------------|-------------------|--------|
| A18 | K8s network policies prevent agent pods from communicating directly with each other except through defined channels (shared state DB, message bus) | Container isolation architecture | If pods can communicate directly, TB8 boundary is even weaker — direct pod-to-pod injection possible | Network policy audit, penetration test | Unvalidated |
| A19 | Egress firewall rules are enforced per MCP server container, not per agent pod | Architecture description | If egress rules are per-pod or cluster-wide, any MCP server could reach any external endpoint | Firewall rule review (Phase 8) | Unvalidated |
| A20 | HITL approval queue cannot be bypassed by agents — the queue is the only path for high-risk tool execution | Stated design (TB10) | If agents can invoke high-risk tools without going through the queue, HITL is cosmetic | Code review of MCP tool invocation path (Phase 8) | Unvalidated |

---

*This threat model was generated with AI assistance using the OWASP MAESTRO Playbook. It must be reviewed by a qualified security professional before use in production risk decisions.*
