# Phase 7: Mitigation Planning — EY_Analysis

**Project:** EY_Analysis
**Date:** 2026-03-27
**Analysis Depth:** Full

---

## Mitigation Strategy

Per `guides/risk-scoring.md`:
- **Critical threats:** At least 1 Preventive AND 1 Detective mitigation
- **High threats:** At least 1 Preventive OR 1 Detective, plus 1 Corrective mitigation
- Catalog IDs from `playbook/09-mitigation-catalog.md` used where applicable

---

## Critical Threat Mitigations (19 threats)

### EY-T1: Direct Prompt Injection → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Input guardrails — prompt injection detection before all LLM calls | L1-P1 | Preventive | Medium | Medium | Not Implemented |
| System prompt hardening — structured prompts with delimiters and instruction hierarchy | L1-P4 | Preventive | Low | Medium | Partially Implemented |
| Output anomaly detection — flag unexpected topics, format violations | L1-D1 | Detective | Medium | Medium | Not Implemented |
| Prompt injection logging and alerting | L1-D4 | Detective | Low | Medium | Partially Implemented |
| Visible input monitoring notification | L1-R1 | Deterrent | Low | Low | Not Implemented |

**Gap flag:** No single preventive control can fully stop prompt injection. Defense-in-depth required across L1 (input guardrails), L3 (tool allowlisting), and L5 (HITL gate).

---

### EY-T2: Indirect Prompt Injection (RAG/Web) → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| RAG source document integrity checks | L2-P1 | Preventive | Medium | Medium | Not Implemented |
| Input sanitization for RAG queries | L2-P4 | Preventive | Medium | Medium | Not Implemented |
| External API input validation (Web Search MCP output) | L7-P3 | Preventive | Medium | Medium | Not Implemented |
| Data pipeline integrity monitoring | L2-D1 | Detective | Medium | Medium | Not Implemented |
| RAG retrieval auditing | L2-D3 | Detective | Low | Medium | Not Implemented |

---

### EY-T3: Cascading Hallucinations → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Output validation against schemas and business rules | L1-P2 | Preventive | Medium | High | Not Implemented |
| Hallucination detection / fact-checking | L1-D2 | Detective | High | Medium | Not Implemented |
| Output quarantine and re-processing | L1-C2 | Corrective | Medium | Medium | Not Implemented |
| Temperature/sampling controls for critical decisions | L1-P5 | Preventive | Low | Medium | Not Implemented |

---

### EY-T5: System Prompt Extraction → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| System prompt hardening | L1-P4 | Preventive | Low | Medium | Partially Implemented |
| Output anomaly detection (detect prompt echo) | L1-D1 | Detective | Medium | Medium | Not Implemented |
| Prompt injection logging | L1-D4 | Detective | Low | Medium | Partially Implemented |

---

### EY-T6: RAG Knowledge Base Poisoning → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| RAG source document integrity (checksums, signatures) | L2-P1 | Preventive | Medium | High | Not Implemented |
| Document source authentication | L2-R2 | Deterrent | Low | Medium | Not Implemented |
| Data pipeline integrity monitoring | L2-D1 | Detective | Medium | Medium | Not Implemented |
| Embedding drift detection | L2-D2 | Detective | Medium | Medium | Not Implemented |
| Knowledge base rollback | L2-C1 | Corrective | Medium | High | Not Implemented |

---

### EY-T7: Semantic Drift → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Automated embedding re-indexing on source doc change | L2-P3 | Preventive | Medium | High | Not Implemented |
| Embedding drift detection | L2-D2 | Detective | Medium | Medium | Not Implemented |
| Emergency RAG bypass (serve curated data) | L2-C3 | Corrective | High | High | Not Implemented |

---

### EY-T9: RAG Data Exfiltration → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Vector DB access controls (per-role, per-agent isolation) | L2-P2 | Preventive | High | High | Not Implemented |
| RAG retrieval auditing | L2-D3 | Detective | Low | Medium | Not Implemented |
| PII detection and masking before LLM calls | Custom (no catalog ID) | Preventive | High | High | Not Implemented |

**Gap flag:** No per-document ACL in vector DB (A22). This is a high-cost architectural gap.

---

### EY-T10: MCP Tool Misuse → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Tool allowlisting and parameter scoping per agent role | L3-P1 | Preventive | Medium | High | Partially Implemented |
| HITL gate for high-risk tool invocations | L5-P2 | Preventive | Medium | High | Partially Implemented |
| Tool invocation monitoring and alerting | L3-D1 | Detective | Low | Medium | Partially Implemented |
| Circuit breaker on anomalous tool usage | L3-C1 | Corrective | Medium | High | Not Implemented |
| Spending and action limits per agent per session | L3-R3 | Deterrent | Low | Medium | Not Implemented |

---

### EY-T11: Tool Description Poisoning → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| MCP server registry and verification | L7-P2 | Preventive | Medium | High | Not Implemented |
| Plugin/server verification (signatures, checksums) | L3-P4 | Preventive | Medium | High | Not Implemented |
| Config drift detection for MCP ConfigMaps | Custom (no catalog ID) | Detective | Medium | High | Not Implemented |
| Agent behavioral profiling (detect changed tool selection patterns) | L3-D3 | Detective | High | Medium | Not Implemented |

---

### EY-T12: Orchestrator Workflow Bypass → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Workflow validation — enforce sequential execution, mandatory steps | L3-P2 | Preventive | Medium | High | Not Implemented |
| Workflow state monitoring | L3-D2 | Detective | Medium | Medium | Not Implemented |
| HITL gate enforced by platform, not by agents (A3) | L5-P2 | Preventive | Medium | High | Partially Implemented |
| Workflow rollback | L3-C2 | Corrective | Medium | Medium | Not Implemented |

---

### EY-T16: MCP Network Exposure → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Network segmentation — restrict MCP server access to authorized agent pods only | L4-P2 | Preventive | Medium | High | Partially Implemented |
| MCP server permission isolation (bind to localhost, least-privilege) | L6-P5 | Preventive | Low | High | Not Implemented |
| Infrastructure anomaly detection (unexpected connections) | L4-D1 | Detective | Medium | Medium | Partially Implemented |

---

### EY-T17: Credential Exposure → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Secrets management via Vault (dynamic secrets, short TTL) | L4-P3 | Preventive | Medium | High | Implemented |
| Least-privilege IAM per component | L4-P1 | Preventive | Medium | High | Partially Implemented |
| Credential leak scanning in repos, logs, configs | L4-D3 | Detective | Low | Medium | Not Implemented |
| Credential rotation on compromise | L4-C1 | Corrective | Medium | High | Partially Implemented |

---

### EY-T18: Code Exec MCP Sandbox Escape → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Agent sandboxing (restricted FS, network, process) | L3-P3 | Preventive | High | High | Partially Implemented |
| Code signing and image verification | L4-P4 | Preventive | Medium | Medium | Not Implemented |
| Infrastructure anomaly detection (escape indicators) | L4-D1 | Detective | Medium | Medium | Partially Implemented |
| Infrastructure isolation on compromise | L4-C2 | Corrective | Medium | High | Not Implemented |

---

### EY-T20: HITL Overwhelm → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| HITL capacity planning — queue management, auto-throttle | L5-P2 | Preventive | Medium | High | Not Implemented |
| HITL effectiveness monitoring (rubber-stamp detection) | L5-D2 | Detective | Medium | High | Not Implemented |
| HITL overload escalation (additional reviewers, throughput reduction) | L5-C2 | Corrective | Medium | Medium | Not Implemented |
| Human trust calibration monitoring | L7-D3 | Detective | Medium | Medium | Not Implemented |

---

### EY-T23: Confused Deputy → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| End-to-end authorization propagation (user identity through agent chain) | L6-P2 | Preventive | High | High | Not Implemented |
| RBAC with separation of duties | L6-P1 | Preventive | Medium | High | Partially Implemented |
| Privilege chain monitoring | CL-D3 | Detective | Medium | High | Not Implemented |
| Privilege usage monitoring | L6-D2 | Detective | Medium | Medium | Not Implemented |

**Gap flag:** This is the highest-impact structural gap. Without L6-P2 (end-to-end auth propagation), every user effectively operates with agent service account privileges.

---

### EY-T25: Data Residency Violation → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Data residency controls in code and infrastructure | L6-P4 | Preventive | High | High | Not Implemented |
| PII detection and masking before external LLM API calls | Custom | Preventive | High | High | Not Implemented |
| Policy violation alerting | L6-D1 | Detective | Medium | Medium | Not Implemented |
| Compliance drift detection | L6-D3 | Detective | Medium | Medium | Not Implemented |

---

### EY-T28: Supply Chain Compromise → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Code signing and image verification for all artifacts | L4-P4 | Preventive | Medium | High | Not Implemented |
| Plugin/dependency verification (signatures, checksums, SBOM) | L3-P4 | Preventive | Medium | High | Not Implemented |
| Supply chain transparency (SBOM publication) | L7-R2 | Deterrent | Medium | Medium | Not Implemented |
| External dependency monitoring | L7-D2 | Detective | Medium | Medium | Not Implemented |

---

### EY-T29: Hallucination→RAG→Tool Chain → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Output validation before tool invocation | L1-P2 | Preventive | Medium | High | Not Implemented |
| Data provenance tracking (distinguish LLM-generated from source docs) | L2-R1 | Preventive | High | High | Not Implemented |
| Cross-layer correlation (detect multi-stage attack pattern) | CL-D1 | Detective | High | High | Not Implemented |
| Cascading rollback across all impacted layers | CL-C2 | Corrective | High | High | Not Implemented |

---

### EY-T30: Cascading Trust Failure → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Trust boundary enforcement at every crossing | CL-P2 | Preventive | High | High | Partially Implemented |
| Privilege minimization across layers | CL-P3 | Preventive | Medium | High | Partially Implemented |
| Cascading failure detection | CL-D2 | Detective | High | High | Not Implemented |
| System-wide circuit breaker | CL-C1 | Corrective | Medium | High | Not Implemented |

---

### EY-T31: Exfiltration via Legitimate Channels → Risk: Critical

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| HITL gate for all outbound communications | L5-P2 | Preventive | Medium | High | Partially Implemented |
| Egress content inspection / DLP for agent outputs | Custom | Preventive | High | High | Not Implemented |
| Data flow integrity monitoring | CL-D4 | Detective | High | Medium | Not Implemented |
| Inter-agent traffic analysis (exfiltration indicators) | L7-D4 | Detective | High | Medium | Not Implemented |

**Gap flag:** Standard DLP cannot detect steganographic exfiltration (BV-8) or data encoded in search queries. Custom controls needed.

---

## High Threat Mitigations (12 threats)

### EY-T4: Model Inconsistency → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Temperature/sampling controls for critical paths | L1-P5 | Preventive | Low | Medium | Not Implemented |
| Behavioral drift detection | L5-D1 | Detective | Medium | Medium | Not Implemented |
| Model rollback on degradation | L1-C1 | Corrective | Medium | Medium | Not Implemented |

---

### EY-T8: Shared Memory Poisoning → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Per-agent memory isolation (separate Redis namespaces/DBs) | Custom | Preventive | Medium | High | Not Implemented |
| Shared memory access logging | L2-D4 | Detective | Low | Medium | Not Implemented |
| Contaminated data quarantine | L2-C2 | Corrective | Medium | Medium | Not Implemented |

---

### EY-T13: Runaway Agent → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Runaway agent detection (iteration/time limits) | L3-D4 | Detective | Low | High | Not Implemented |
| Circuit breaker | L3-C1 | Corrective | Medium | High | Not Implemented |
| Spending and action limits | L3-R3 | Deterrent | Low | Medium | Not Implemented |

---

### EY-T15: Insecure Inter-Agent Protocol → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Schema validation on all inter-agent messages | L3-P5 | Preventive | Medium | Medium | Not Implemented |
| Mutual authentication for inter-agent comms | L7-P1 | Preventive | Medium | High | Not Implemented |
| End-to-end encryption | CL-P4 | Preventive | Medium | Medium | Partially Implemented |
| Inter-agent traffic analysis | L7-D4 | Detective | High | Medium | Not Implemented |

---

### EY-T19: K8s Orchestration Compromise → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Code signing and image verification | L4-P4 | Preventive | Medium | High | Not Implemented |
| Orchestration integrity monitoring | L4-D4 | Detective | Medium | Medium | Not Implemented |
| Disaster recovery / rapid redeployment | L4-C4 | Corrective | High | High | Partially Implemented |

---

### EY-T21: Selective Log Manipulation → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Immutable audit logging (append-only storage) | L5-P1 | Preventive | Medium | High | Not Implemented |
| Log integrity protection (cryptographic chaining) | L5-P4 | Preventive | Medium | High | Not Implemented |
| Log completeness verification | L5-D3 | Detective | Medium | Medium | Not Implemented |
| Log reconstruction from redundant streams | L5-C1 | Corrective | Medium | Medium | Not Implemented |

---

### EY-T24: Dynamic Policy Enforcement Failure → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Runtime guardrails engine | L6-P3 | Preventive | High | High | Not Implemented |
| Policy violation alerting | L6-D1 | Detective | Medium | Medium | Not Implemented |
| Automatic policy re-enforcement | L6-C1 | Corrective | Medium | High | Not Implemented |

---

### EY-T26: A2A Memory Injection → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| External API input validation (treat A2A as untrusted) | L7-P3 | Preventive | Medium | Medium | Partially Implemented |
| Agent identity verification (behavioral fingerprinting) | L7-D1 | Detective | High | Medium | Not Implemented |
| Agent de-registration on identity failure | L7-C1 | Corrective | Low | High | Not Implemented |

---

### EY-T27: Malicious Agent Diffusion → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| HITL gate for all external communications | L5-P2 | Preventive | Medium | High | Partially Implemented |
| Agent behavioral profiling | L3-D3 | Detective | High | Medium | Not Implemented |
| Agent quarantine | L3-C3 | Corrective | Medium | High | Not Implemented |

---

### EY-T32: Context Window Poisoning → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Input length limits and context budget management | Custom | Preventive | Low | Medium | Not Implemented |
| System prompt reinforcement at end of context | L1-P4 | Preventive | Low | Medium | Partially Implemented |
| Output anomaly detection | L1-D1 | Detective | Medium | Medium | Not Implemented |

---

### EY-T34: OAuth Token Relay → Risk: High

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| End-to-end authorization propagation | L6-P2 | Preventive | High | High | Not Implemented |
| Privilege chain monitoring | CL-D3 | Detective | Medium | High | Not Implemented |
| Trust reset on chain compromise | L7-C3 | Corrective | High | High | Not Implemented |

---

## Medium Threat Mitigations (4 threats)

### EY-T14: Cross-Client MCP Interference → Risk: Medium

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Schema validation | L3-P5 | Preventive | Medium | Medium | Not Implemented |
| Session isolation in MCP servers | Custom | Preventive | Medium | High | Not Implemented |

### EY-T22: Insufficient Logging → Risk: Medium

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Log completeness verification | L5-D3 | Detective | Medium | Medium | Not Implemented |
| Define minimum logging requirements per component | Custom | Preventive | Low | Medium | Not Implemented |

### EY-T33: TOCTOU Permission Race → Risk: Medium

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Re-validate permissions at execution time, not just at request time | Custom | Preventive | Medium | High | Not Implemented |

### EY-T35: Observability Overload → Risk: Medium

| Mitigation | Catalog ID | Type | Cost | Effectiveness | Status |
|-----------|-----------|------|------|--------------|--------|
| Log rate limiting and deduplication | Custom | Preventive | Low | Medium | Not Implemented |
| Anomaly-based alerting (statistical, not volume-based) | L5-D4 | Detective | Medium | Medium | Not Implemented |

---

## Mitigation Gap Analysis

| Gap | Affected Threats | Severity | Recommendation |
|-----|-----------------|----------|----------------|
| **No end-to-end auth propagation (L6-P2)** | EY-T23, EY-T34 | Critical | Highest-priority architectural change — confused deputy affects every user interaction |
| **No PII detection/masking** | EY-T9, EY-T25, EY-T31 | Critical | Required for GDPR, EU AI Act compliance; addresses data residency and exfiltration |
| **No per-document ACL in vector DB** | EY-T9 | Critical | Architectural limitation; evaluate vector DB alternatives or implement proxy-level filtering |
| **No immutable audit logging** | EY-T21 | High | Prerequisite for HC-3 (full auditability); required for SOX compliance |
| **No cross-layer correlation** | EY-T29, EY-T30 | Critical | Without this, multi-stage attacks are invisible; implement SIEM-level correlation |
| **No circuit breaker / runaway detection** | EY-T13, EY-T10 | High | Low-cost, high-impact controls to prevent resource and cost exhaustion |
| **No config drift detection for MCP** | EY-T11 | Critical | Rogue MCP server redirect is undetectable without this |
| **HITL capacity not planned** | EY-T20 | Critical | Queue management and auto-throttle needed before production scale |

---

## Mitigation Summary

| Implementation Status | Count |
|----------------------|-------|
| Implemented | 1 (Vault secrets management) |
| Partially Implemented | 12 |
| Not Implemented | 72 |
| **Total unique mitigations** | 85 |

---

## Post-Phase Verification

All mitigation catalog IDs (L1-P1 through CL-R3) were verified against `playbook/09-mitigation-catalog.md`. All threat IDs referenced in mitigation cards exist in the Phase 6 threat register. No discrepancies found.

---

*This threat model was generated with AI assistance using the OWASP MAESTRO Playbook. It must be reviewed by a qualified security professional before use in production risk decisions.*
