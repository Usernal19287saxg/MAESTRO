# Phase 3: Threat Actor Analysis — EY_Analysis

**Project:** EY_Analysis
**Date:** 2026-03-27
**Analysis Depth:** Full

---

## Threat Actor Catalog

### TA1: External Attackers (Cybercriminals)

| Field | Value |
|-------|-------|
| Actor ID | TA1 |
| Actor Category | External Attacker |
| Capability | Moderate to High |
| Motivation | Financial (data theft, ransomware, credential harvesting) |
| Target Assets | PII in RAG knowledge base, LLM API keys, agent credentials, database contents, system prompts (business logic IP) |
| Likely Attack Vectors | Prompt injection via user input or poisoned web content, credential stuffing against SSO, exploiting publicly discoverable MCP endpoints, supply chain compromise (malicious packages), API key harvesting from public repos |
| Target MAESTRO Layers | L1 (prompt injection), L2 (RAG poisoning, data exfiltration), L3 (MCP tool misuse), L4 (exposed endpoints), L7 (external API abuse) |
| Relevance | **Relevant** |
| Priority | **Critical** |

**Rationale:** GAAP's external-facing nature, Restricted data classification, and publicly discoverable MCP endpoints make it a high-value target for financially motivated attackers. The combination of autonomous execution and access to structured organizational data means a successful compromise yields both data and operational control. Prompt injection is the lowest-barrier attack vector and can cascade across the multi-agent workflow.

---

### TA2: Malicious Insiders

| Field | Value |
|-------|-------|
| Actor ID | TA2 |
| Actor Category | Malicious Insider |
| Capability | High (legitimate access, internal knowledge) |
| Motivation | Financial (data theft, sabotage), Espionage (selling IP to competitors), Disruption (disgruntled employee) |
| Target Assets | System prompts (proprietary business logic), RAG knowledge base (organizational IP), long-term agent memory (PostgreSQL), Vault credentials, MCP server implementations |
| Likely Attack Vectors | System prompt exfiltration (operators can read prompts), agent memory poisoning (DB write access to long-term memory), MCP server backdooring (modifying tool implementations), credential abuse (admin access to Vault/K8s), log tampering to cover tracks |
| Target MAESTRO Layers | L2 (memory poisoning, data exfiltration), L3 (MCP server tampering), L4 (K8s admin abuse), L5 (log tampering), L6 (secrets exfiltration) |
| Relevance | **Relevant** |
| Priority | **High** |

**Insider Access Profile:**

| Role | Assumed Count | Access Level | Risk Factor |
|------|--------------|-------------|-------------|
| Platform Admins | 3–5 | Full (Vault, K8s, all MCP servers) | High |
| ML/AI Engineers | 5–10 | Operator (model config, prompt templates, agent logic) | High |
| DevOps/Infra | 3–5 | Operator (deployment, secrets pipeline) | High |
| Security/Compliance | 2–3 | ReadOnly + audit | Medium |
| External Contractors | Assumed present | Scoped operator access | High — no long-term accountability |
| Third-party Vendor Ops | Assumed present (cloud, LLM APIs) | Vendor-side admin | Medium — outside org control |

**Key Insider Threat Vectors for Agentic Systems:**
- **System prompt exfiltration:** Operators can read business logic encoded in prompts — no exploit required, just access abuse.
- **Agent memory poisoning:** Engineers with DB write access can manipulate long-term agent memory in PostgreSQL or vector DB, causing agents to act on false information in future sessions.
- **MCP server backdooring:** Infrastructure team can modify MCP tool implementations without code review if SDLC controls are weak — a backdoored `run_python` or `call_external_api` tool is nearly undetectable without code signing.

---

### TA3: Compromised Agents

| Field | Value |
|-------|-------|
| Actor ID | TA3 |
| Actor Category | Compromised Agent |
| Capability | High (inherits full permissions of the subverted agent) |
| Motivation | N/A (instrumentalized by another threat actor via prompt injection, memory poisoning, or supply chain attack) |
| Target Assets | All assets accessible to the compromised agent role — varies by role (see Agent Role Risk Matrix below) |
| Likely Attack Vectors | Prompt injection via user input or ingested web content, memory poisoning via long-term state, tool description poisoning from rogue MCP server, supply chain compromise of agent framework dependencies, cross-agent state manipulation via shared Redis/blackboard |
| Target MAESTRO Layers | L1 (prompt injection origin), L2 (memory poisoning origin), L3 (tool misuse, workflow hijack), L7 (lateral propagation to other agents, external exfiltration) |
| Relevance | **Relevant** |
| Priority | **Critical** |

**Agent Role Risk Matrix — Impact of Compromise:**

| Agent Role | If Compromised... | Blast Radius |
|-----------|-------------------|-------------|
| **Orchestrator** | Full workflow hijack — can redirect tasks, suppress Critic validation, dispatch any subagent | **Critical** — controls all downstream agents |
| **Researcher** | Data exfiltration via web search tool (encodes sensitive data in search queries), RAG poisoning via memory writes | **High** — can poison data used by all agents |
| **Executor** | Arbitrary code execution, file system manipulation, database modification, external API calls with stored credentials | **Critical** — highest direct system impact |
| **Critic/Validator** | Guardrail bypass — approves malicious outputs, suppresses security alerts | **High** — removes safety controls for other agents |
| **Notifier** | Data exfiltration via email/Slack/webhook (encodes sensitive data in "legitimate" notifications), social engineering of external recipients | **High** — irreversible external-facing actions |

**Agentic Considerations:**
- A compromised agent exploits non-determinism to hide malicious behavior within the normal variance of LLM outputs — detection requires behavioral baselines, not rule-based checks.
- A compromised Orchestrator is the worst-case scenario: it controls the entire agent workflow and can suppress the Critic agent that would otherwise detect anomalies.
- Compromised agents inherit the identity and permissions of the legitimate agent — they are effectively insider threats operating at machine speed.

---

### TA4: Nation-State Actors

| Field | Value |
|-------|-------|
| Actor ID | TA4 |
| Actor Category | Nation-State |
| Capability | Advanced (unlimited resources, custom tooling, zero-day access, sustained campaigns) |
| Motivation | Espionage (organizational data, trade secrets, government data), Strategic disruption |
| Target Assets | RAG knowledge base (bulk organizational data), model weights (self-hosted Llama/Mistral), agent framework supply chain, LLM API traffic (intercept/modify), Vault secrets |
| Likely Attack Vectors | Supply chain compromise (malicious agent framework packages, poisoned LLM fine-tuning data), LLM API traffic interception (at network or vendor level), zero-day exploitation of K8s or MCP server vulnerabilities, long-term persistent access via compromised CI/CD pipeline |
| Target MAESTRO Layers | L1 (model supply chain), L3 (framework supply chain, MCP server compromise), L4 (infrastructure zero-days), L6 (secrets exfiltration) |
| Relevance | **Partially Relevant** (escalates to Relevant for deployments in finance, critical infrastructure, government, legal, healthcare) |
| Priority | **High** |

**Rationale:** The primary nation-state attack surface is not direct intrusion but supply chain compromise. GAAP's properties that attract state actors:
- Access to large volumes of structured organizational data via RAG
- Autonomous execution capabilities (Code Execution MCP, API Gateway MCP)
- Ability to exfiltrate data at machine speed without obvious human-visible signals
- Supply chain depth: foundation model API → agent framework → MCP servers → tool dependencies creates a long, auditable chain with many compromise points

---

### TA5: Automated Threats / Adversarial AI

| Field | Value |
|-------|-------|
| Actor ID | TA5 |
| Actor Category | Automated Threat |
| Capability | Moderate to High (AI-powered, operates at machine speed, learns from responses) |
| Motivation | Opportunistic (exploit discoverable endpoints), Financial (credential harvesting, crypto-mining via code execution), Disruption (resource exhaustion) |
| Target Assets | MCP server endpoints and tool schemas, LLM API endpoints, compute resources (via Code Execution MCP), agent credentials |
| Likely Attack Vectors | Automated MCP endpoint discovery (port scanning, `tools/list` enumeration), targeted prompt injection crafting based on discovered tool schemas, fuzzing MCP tool parameters, resource exhaustion via repeated LLM API calls or code execution, adversarial AI agents conducting multi-step attacks |
| Target MAESTRO Layers | L3 (MCP tool enumeration and misuse), L4 (endpoint discovery, resource exhaustion), L1 (automated prompt injection), L7 (A2A protocol abuse) |
| Relevance | **Relevant** |
| Priority | **High** |

**Discoverability Analysis:**
- HTTP/SSE MCP transport exposes network endpoints discoverable via port scanning
- Agent frameworks (LangChain, LangGraph, AutoGen) have well-documented default port/endpoint conventions targeted by adversarial scanners
- MCP servers registered in public registries or referenced in public repos expose full tool schemas (parameter names, types) — enables targeted payload crafting
- LLM API endpoints return model-specific error messages that fingerprint the provider and model version
- MCP tool enumeration via automated scanning was demonstrated within weeks of the MCP specification's public release — this is an active, not theoretical, threat

---

### TA6: Competitive Espionage

| Field | Value |
|-------|-------|
| Actor ID | TA6 |
| Actor Category | External Attacker (Competitor-motivated) |
| Capability | Moderate to High (may hire sophisticated operators or use insider recruitment) |
| Motivation | Espionage (organizational IP, trade secrets, proprietary workflows), Disruption (degrade competitor operations) |
| Target Assets | RAG knowledge base (organizational IP, internal documents, trade secrets), system prompts (proprietary business logic and workflows), agent workflow definitions |
| Likely Attack Vectors | Supply chain compromise (planting malicious MCP servers or framework-compatible packages), prompt injection via shared/public data sources the platform ingests, insider recruitment, credential acquisition via social engineering |
| Target MAESTRO Layers | L2 (RAG knowledge base exfiltration), L3 (system prompt theft, malicious MCP packages), L7 (poisoning shared data feeds) |
| Relevance | **Relevant** |
| Priority | **Medium** |

**Rationale:** Competitive espionage against agentic systems is asymmetric — a single successful prompt injection can cascade across an entire agent workflow before detection, causing disproportionate damage relative to traditional systems. The most likely vectors are indirect: supply chain compromise and data feed poisoning rather than direct intrusion.

---

## Threat Actor Priority Ranking

| Rank | Actor ID | Actor | Priority | Justification |
|------|----------|-------|----------|---------------|
| 1 | TA3 | Compromised Agents | **Critical** | Unique to agentic systems; inherits full agent permissions; operates at machine speed; hardest to detect due to non-determinism cover; Orchestrator compromise cascades to all agents |
| 2 | TA1 | External Attackers | **Critical** | Broadest attack surface (external-facing, discoverable endpoints); prompt injection is low-barrier and high-impact; Restricted data makes platform a high-value target |
| 3 | TA2 | Malicious Insiders | **High** | Legitimate access to system prompts, agent memory, and MCP implementations; agentic-specific vectors (memory poisoning, MCP backdooring) bypass traditional controls |
| 4 | TA5 | Automated Threats | **High** | MCP endpoint discoverability proven in practice; AI-powered scanners operate at scale; Code Execution MCP is a high-value compute target |
| 5 | TA4 | Nation-State Actors | **High** | Supply chain depth creates many compromise points; data exfiltration at machine speed is an attractive capability for state actors; relevance escalates by deployment sector |
| 6 | TA6 | Competitive Espionage | **Medium** | Indirect vectors (supply chain, data feed poisoning) are realistic; direct intrusion less likely but high-impact if successful |

---

## Prior Incident Calibration (Industry Baseline)

The following assumed prior exposure calibrates threat likelihood ratings in Phase 6:

| Incident Type | Assumed Prior Exposure | Impact on Phase 6 Likelihood |
|--------------|----------------------|------------------------------|
| Phishing / credential theft | **High** (>80% industry rate) — assume at least one operator credential has been phished | Credential-based threats rated Likely or higher |
| API key leakage | **High** — LLM API keys frequently leak via public repos | API key theft/misuse rated Likely |
| Insider data exfiltration | **Low but non-zero** — no confirmed incident, but no tooling to have detected a sophisticated one | Insider threats rated Possible |
| AI-specific incidents | **None confirmed** (novel threat class) — absence of detection ≠ absence of incident given observability gaps | AI-specific threats (prompt injection, memory poisoning) rated Possible to Likely based on attack surface analysis, not incident history |

---

## Assumptions

| ID | Assumption | Rationale | Impact if Wrong | Validation Method | Status |
|----|-----------|-----------|----------------|-------------------|--------|
| A15 | At least one operator credential has been compromised at some point (industry baseline) | >80% phishing success rate in industry data | If no credential compromise has occurred, insider/credential threat likelihood can be reduced | Review incident response records, phishing simulation results | Unvalidated |
| A16 | MCP tool schemas are discoverable by automated scanners if any MCP server uses HTTP transport | MCP `tools/list` enumeration demonstrated in practice | If all MCP servers are strictly firewalled and use stdio only, automated discovery threat is reduced | Network penetration test, port scan from external vantage | Unvalidated |
| A17 | Compromised Orchestrator agent is the worst-case agent compromise scenario | Orchestrator controls all downstream agent dispatch and can suppress Critic | If Orchestrator has limited permissions and cannot suppress Critic, blast radius is reduced | Architecture review of Orchestrator permissions (Phase 8) | Unvalidated |

---

*This threat model was generated with AI assistance using the OWASP MAESTRO Playbook. It must be reviewed by a qualified security professional before use in production risk decisions.*
