# GKE Cloud Architecture for corteX Agent Runtime

**Document Type:** Research & Planning
**Date:** February 2026
**Status:** Recommended Architecture - Pending Implementation
**Authors:** corteX Architecture Team

---

## Executive Summary

This document presents findings from four parallel research teams evaluating cloud
infrastructure options for deploying corteX agents at scale. The recommended
architecture is **Option D: Hybrid** -- corteX Brain engine + Direct Anthropic API +
MCP for infrastructure integration, deployed on Google Kubernetes Engine (GKE) with
the Agent Sandbox project for secure, multi-tenant agent execution.

The key conclusion: Claude Agent SDK is an **architectural conflict** (score 2.8/10)
and must NOT be adopted. corteX should use the Direct Anthropic API (score 9.0/10)
enhanced with MCP integrations (score 7.0/10), yielding a Hybrid score of 8.0/10.

---

## 1. Architecture Options Evaluation

### 1.1 Scoring Matrix

| Option | Description                        | Score  | Verdict         |
|--------|------------------------------------|--------|-----------------|
| A      | Claude Agent SDK                   | 2.8/10 | **Reject**      |
| B      | Direct Anthropic API               | 9.0/10 | Already works   |
| C      | MCP for infrastructure             | 7.0/10 | Enhancement     |
| **D**  | **Hybrid (B + C + corteX Brain)**  | **8.0/10** | **Recommended** |

### 1.2 Recommendation: Option D - Hybrid Architecture

```
+------------------------------------------------------------------+
|                     corteX Hybrid Architecture                    |
+------------------------------------------------------------------+
|                                                                    |
|  +------------------+    +------------------+    +---------------+ |
|  |  corteX Brain    |    |  Direct Anthropic|    |  MCP Client   | |
|  |  Engine          |--->|  API (Claude)    |    |  Adapter      | |
|  |                  |    |                  |    |               | |
|  |  - Weights       |    |  - Messages API  |    |  - GKE MCP    | |
|  |  - Goal Tracker  |    |  - Tool Use      |    |  - BigQuery   | |
|  |  - Plasticity    |    |  - Streaming     |    |  - Logging    | |
|  |  - Prediction    |    |  - Vision        |    |  - GitHub     | |
|  |  - Loop Safety   |    |                  |    |  - Filesystem | |
|  +--------+---------+    +--------+---------+    +-------+-------+ |
|           |                       |                      |         |
|           +----------+------------+----------+-----------+         |
|                      |                       |                     |
|              +-------v-------+       +-------v--------+           |
|              |  Orchestrator |       |  Tool Registry  |          |
|              |  (Prefrontal  |       |  (MCP + Native) |          |
|              |   Cortex)     |       |                 |          |
|              +-------+-------+       +-------+--------+           |
|                      |                       |                     |
|              +-------v-----------------------v--------+           |
|              |          GKE Agent Runtime              |          |
|              |  (Agent Sandbox + gVisor + Warm Pools)  |          |
|              +----------------------------------------+           |
+------------------------------------------------------------------+
```

The Hybrid approach preserves corteX's brain-inspired value proposition (weights,
plasticity, goal tracking, loop safety) while leveraging Anthropic's API directly
for LLM inference and MCP for standardized infrastructure access.

---

## 2. Why NOT Claude Agent SDK

Claude Agent SDK introduces fundamental architectural conflicts with corteX:

### 2.1 Competing Subsystems

| Subsystem          | corteX                          | Claude Agent SDK              | Conflict     |
|--------------------|---------------------------------|-------------------------------|--------------|
| Agentic Loop       | Brain-driven orchestration      | Own competing loop            | **Critical** |
| Tool Framework     | CapabilitySet + registry        | Own tool definitions          | **Critical** |
| Context Management | Synaptic weights + plasticity   | Own context window mgmt       | **High**     |
| Security Model     | Capability-based attenuation    | Binary allow/deny             | **High**     |
| Memory             | 4-tier (working/ST/LT/episodic) | Single conversation context   | **Medium**   |

### 2.2 Infrastructure Cost Penalty

| Deployment Model       | Cost/hr per Agent |
|------------------------|-------------------|
| Claude Agent SDK       | ~$0.05/hr         |
| GKE Container (corteX) | $0.01-0.02/hr     |
| **Difference**         | **2.5-5x higher** |

### 2.3 Strategic Risk

Adopting Claude Agent SDK would mean:
- Abandoning corteX's brain engine -- the core value proposition
- Locking into a single LLM vendor's execution model
- Losing provider-agnostic architecture (OpenAI, Gemini, local models)
- Replacing sophisticated capability attenuation with binary security

**Verdict:** Claude Agent SDK is a product for teams that have NO agent framework.
corteX already IS an agent framework. Using both creates redundancy and conflict.

---

## 3. GKE Agent Sandbox

### 3.1 Overview

The **kubernetes-sigs/agent-sandbox** project (launched November 2025) provides
purpose-built Kubernetes primitives for AI agent execution. It is maintained under
the Kubernetes SIGs organization and designed specifically for the agent workload
pattern corteX requires.

### 3.2 Custom Resource Definitions

```
+-------------------+     +--------------------+     +----------------+
| SandboxTemplate   |---->| SandboxWarmPool    |---->| Sandbox        |
| (defines agent    |     | (pre-warmed pods   |     | (running agent |
|  pod spec)        |     |  for fast startup) |     |  instance)     |
+-------------------+     +--------------------+     +----------------+
         |                                                   ^
         |                                                   |
         +---->  SandboxClaim  (tenant requests a sandbox) --+
```

| CRD               | Purpose                                            |
|--------------------|----------------------------------------------------|
| SandboxTemplate    | Defines container image, resources, security       |
| SandboxWarmPool    | Maintains N pre-created pods for instant allocation |
| SandboxClaim       | Tenant request for a sandbox (bound to template)   |
| Sandbox            | Running agent instance with lifecycle management   |

### 3.3 Key Capabilities

- **Sub-second startup** via warm pools (pods pre-created, waiting for claims)
- **gVisor runtime** (runsc) for userspace kernel isolation between agents
- **Pod Snapshots** for checkpoint/restore of agent state to Cloud Storage
- **Automatic cleanup** with configurable TTL and idle timeouts
- **Resource quotas** enforced at the sandbox level

### 3.4 Example: SandboxTemplate for corteX Agent

```yaml
apiVersion: sandbox.k8s.io/v1alpha1
kind: SandboxTemplate
metadata:
  name: cortex-agent
  namespace: tenant-acme
spec:
  runtimeClassName: gvisor
  containers:
    - name: agent
      image: gcr.io/questo/cortex-agent:latest
      resources:
        requests:
          cpu: "250m"
          memory: "512Mi"
        limits:
          cpu: "1"
          memory: "2Gi"
      securityContext:
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        allowPrivilegeEscalation: false
  networkPolicy:
    egress:
      - to:
          - ipBlock:
              cidr: 10.0.0.0/8  # internal only
        ports:
          - port: 443
            protocol: TCP
```

---

## 4. MCP Integration Path

### 4.1 MCP Server Ecosystem (Tiered by Trust)

| Tier | Source                      | Trust Level       | Examples                              |
|------|-----------------------------|-------------------|---------------------------------------|
| 1    | modelcontextprotocol org    | **Highest**       | Filesystem, Git, Memory, Fetch        |
| 2    | Vendor Official             | **High**          | Google (GKE, BigQuery), GitHub, Docker |
| 3    | Anthropic Registry          | **Vetted**        | Curated third-party servers           |
| 4    | Community (popular)         | Moderate          | High-star open-source servers         |
| 5    | Community (unvetted)        | **Use at own risk** | Unknown provenance                  |

### 4.2 Google Managed MCP Servers

Google provides official MCP servers for GCP services, usable from corteX agents:

| MCP Server     | Capabilities                                |
|----------------|---------------------------------------------|
| GKE            | Cluster management, pod operations          |
| Compute Engine | VM lifecycle, snapshots                     |
| BigQuery       | Query execution, dataset management         |
| Cloud SQL      | Database operations, backups                |
| Cloud Logging  | Log queries, alert management               |
| Cloud Monitor  | Metrics, dashboards, SLO tracking           |

### 4.3 Vendor and Reference MCP Servers

| Server                    | Source                       | Use Case                   |
|---------------------------|------------------------------|----------------------------|
| Filesystem                | modelcontextprotocol org     | File read/write in sandbox |
| Git                       | modelcontextprotocol org     | Repository operations      |
| Memory                    | modelcontextprotocol org     | Persistent key-value store |
| Fetch                     | modelcontextprotocol org     | HTTP requests              |
| github-mcp-server         | github/github-mcp-server    | GitHub API integration     |
| Docker MCP                | Docker official              | Container management       |
| Red Hat Kubernetes MCP    | Red Hat                      | K8s operations             |

### 4.4 corteX MCP Client Module

A new module `corteX/integrations/mcp/` will serve as the MCP client adapter:

```
corteX/integrations/mcp/
    __init__.py
    client.py          # MCP client protocol implementation
    registry.py        # Discovers and registers MCP servers
    security.py        # Tier validation + CapabilitySet enforcement
    adapters/
        google.py      # GCP MCP server adapters
        github.py      # GitHub MCP adapter
        filesystem.py  # Sandboxed filesystem adapter
```

The MCP client will:
1. Discover available MCP servers per tenant configuration
2. Validate server trust tier against tenant security policy
3. Map MCP tools to corteX's CapabilitySet for attenuation
4. Route MCP tool calls through the brain engine for weight-based selection

---

## 5. Multi-Tenancy: Namespace-per-Tenant

### 5.1 Architecture

```
+------------------------------------------+
|              GKE Cluster                  |
|                                           |
|  +-----------+  +-----------+  +-------+  |
|  | ns:       |  | ns:       |  | ns:   |  |
|  | tenant-   |  | tenant-   |  | cortex|  |
|  | acme      |  | tenant-   |  | system|  |
|  |           |  | globex    |  |       |  |
|  | [agents]  |  | [agents]  |  | [ctrl]|  |
|  | [secrets] |  | [secrets] |  | [api] |  |
|  | [quotas]  |  | [quotas]  |  | [mon] |  |
|  +-----------+  +-----------+  +-------+  |
|                                           |
+------------------------------------------+
```

### 5.2 Isolation Primitives

| Primitive          | Purpose                                          |
|--------------------|--------------------------------------------------|
| Namespace          | Logical boundary per tenant                      |
| RBAC               | Tenant-scoped roles, no cross-namespace access   |
| NetworkPolicy      | Default deny-all, explicit allowlists            |
| ResourceQuota      | CPU/memory/pod limits per tenant                 |
| LimitRange         | Per-pod resource bounds within tenant            |
| Workload Identity  | GCP service account mapping (no static keys)     |
| gVisor             | Kernel-level isolation between agent pods        |

### 5.3 Zero-Trust Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-acme
spec:
  podSelector: {}       # applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
  ingress: []            # deny all inbound
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: cortex-system   # only to control plane
      ports:
        - port: 443
          protocol: TCP
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0       # LLM API egress
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
          protocol: TCP
```

---

## 6. Security Stack

### 6.1 Defense in Depth

```
Layer 1: GKE Cluster  -->  Private cluster, authorized networks
Layer 2: Namespace     -->  RBAC + NetworkPolicy isolation
Layer 3: Pod           -->  gVisor runtime, read-only rootfs
Layer 4: Container     -->  Seccomp profiles, no privilege escalation
Layer 5: Network       -->  Egress proxy, TLS-only external
Layer 6: Identity      -->  Workload Identity (no static keys)
Layer 7: Application   -->  corteX CapabilitySet attenuation
```

### 6.2 Security Measures Detail

| Measure                    | Implementation                                  |
|----------------------------|--------------------------------------------------|
| Runtime isolation          | gVisor (userspace kernel via runsc)              |
| Syscall filtering          | Seccomp profiles (allowlist mode)                |
| Filesystem                 | Read-only root, tmpfs for scratch                |
| Network egress             | Controlled via proxy + NetworkPolicy             |
| Identity                   | Workload Identity Federation (zero static keys)  |
| Secrets                    | GCP Secret Manager, mounted as volumes           |
| Agent behavior             | corteX CapabilitySet enforcement per tenant      |
| Audit                      | Cloud Audit Logs + corteX event stream           |

### 6.3 MCP Safety Tiers and Enforcement

| Tier | Trust Level      | Enforcement                                       |
|------|------------------|---------------------------------------------------|
| 1    | Highest          | Auto-approved, full capability access              |
| 2    | High             | Auto-approved, logged, capability-scoped           |
| 3    | Vetted           | Tenant opt-in required, sandboxed execution        |
| 4    | Moderate         | Explicit admin approval, heavy sandboxing          |
| 5    | Use at own risk  | Disabled by default, requires security review      |

---

## 7. Cost Model

### 7.1 Per-Agent Cost Breakdown

| Component            | Spot Instance | On-Demand    |
|----------------------|---------------|--------------|
| Compute (e2-small)   | $0.012/hr     | $0.034/hr    |
| gVisor overhead      | $0.005/hr     | $0.005/hr    |
| Networking           | $0.005/hr     | $0.005/hr    |
| Storage (ephemeral)  | $0.002/hr     | $0.002/hr    |
| Monitoring           | $0.003/hr     | $0.003/hr    |
| **Warm pool amort.** | $0.015/hr     | $0.029/hr    |
| **Total per agent**  | **$0.042/hr** | **$0.078/hr**|

### 7.2 Monthly Infrastructure Estimates

| Scale        | Agents | Infra Cost/mo | With CUD (3yr) | Notes                    |
|--------------|--------|---------------|-----------------|--------------------------|
| 10 tenants   | ~50    | ~$1,000       | ~$750           | Startup phase            |
| 50 tenants   | ~250   | ~$3,200       | ~$2,400         | Growth phase             |
| 100 tenants  | ~500   | ~$5,150       | ~$4,100         | Scale phase              |
| 500 tenants  | ~2500  | ~$22,000      | ~$16,500        | Multi-region required    |

### 7.3 Cost Dominance Analysis

```
Infrastructure costs:  ~15-20% of total
LLM API token costs:   ~70-80% of total   <-- DOMINANT
Storage/networking:     ~5-10% of total
```

The dominant cost driver is LLM API tokens, not infrastructure.
Infrastructure optimization (Spot, CUD, autoscaling) yields meaningful but
secondary savings. Token optimization (caching, prompt compression, model
routing) has the highest ROI.

---

## 8. Implementation Phases

### Phase 1: Foundation (Weeks 1-4)

| Task                              | Priority | Effort  |
|-----------------------------------|----------|---------|
| GKE private cluster setup         | P0       | 3 days  |
| Agent Sandbox CRD installation    | P0       | 2 days  |
| SandboxTemplate for corteX agent  | P0       | 3 days  |
| Warm pool configuration           | P0       | 2 days  |
| gVisor runtime class setup        | P0       | 1 day   |
| CI/CD pipeline (Cloud Build)      | P1       | 3 days  |
| Basic monitoring (Cloud Monitor)  | P1       | 2 days  |
| **Milestone:** Single-tenant agent execution on GKE  |

### Phase 2: Integration (Weeks 5-8)

| Task                              | Priority | Effort  |
|-----------------------------------|----------|---------|
| MCP client module implementation  | P0       | 5 days  |
| Google MCP server integration     | P0       | 3 days  |
| Namespace-per-tenant setup        | P0       | 3 days  |
| RBAC + NetworkPolicy per tenant   | P0       | 3 days  |
| ResourceQuota + LimitRange        | P1       | 2 days  |
| Workload Identity configuration   | P1       | 2 days  |
| **Milestone:** Multi-tenant with MCP integration     |

### Phase 3: Resilience (Weeks 9-12)

| Task                              | Priority | Effort  |
|-----------------------------------|----------|---------|
| Pod Snapshots for state recovery  | P0       | 5 days  |
| Checkpoint/restore to GCS         | P0       | 3 days  |
| Prometheus + Grafana dashboards   | P1       | 3 days  |
| Alerting (PagerDuty/Opsgenie)     | P1       | 2 days  |
| Chaos testing (agent failures)    | P1       | 3 days  |
| Security audit + pen testing      | P0       | 5 days  |
| **Milestone:** Production-ready single region         |

### Phase 4: Scale (Weeks 13-16)

| Task                              | Priority | Effort  |
|-----------------------------------|----------|---------|
| Horizontal pod autoscaling        | P0       | 3 days  |
| Cluster autoscaling (node pools)  | P0       | 2 days  |
| Multi-region GKE (if needed)      | P1       | 5 days  |
| 500+ concurrent agent testing     | P0       | 5 days  |
| Cost optimization (Spot + CUD)    | P1       | 3 days  |
| Documentation + runbooks          | P1       | 3 days  |
| **Milestone:** 500+ agents, multi-region capable      |

### Phase Summary Timeline

```
Week:  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16
       |=========Phase 1=========|=========Phase 2=========|
                                  |=========Phase 3=========|=========Phase 4=========|
       [GKE + Sandbox + Pools]    [MCP + Multi-tenant]       [Snapshots + Monitoring]
                                                              [Scale + Multi-region]
```

---

## 9. Full System Architecture Diagram

```
                            +---------------------------+
                            |      Client / SaaS App    |
                            +-------------+-------------+
                                          |
                                     HTTPS/WSS
                                          |
                            +-------------v-------------+
                            |    GKE Ingress (HTTPS)    |
                            |    Cloud Armor WAF        |
                            +-------------+-------------+
                                          |
                   +----------------------+----------------------+
                   |                                             |
        +----------v-----------+                   +-------------v-----------+
        |   cortex-system ns   |                   |    tenant-{name} ns     |
        |                      |                   |                         |
        |  +----------------+  |    SandboxClaim   |  +-------------------+  |
        |  | corteX API     +--+------------------>+  | Agent Sandbox     |  |
        |  | (FastAPI)      |  |                   |  | (gVisor runtime)  |  |
        |  +----------------+  |                   |  |                   |  |
        |  +----------------+  |                   |  | +---------------+ |  |
        |  | Orchestrator   |  |                   |  | | corteX Brain  | |  |
        |  | Controller     |  |                   |  | | Engine        | |  |
        |  +----------------+  |                   |  | +-------+-------+ |  |
        |  +----------------+  |                   |  |         |         |  |
        |  | Warm Pool Mgr  |  |                   |  | +-------v-------+ |  |
        |  +----------------+  |                   |  | | MCP Client    | |  |
        |  +----------------+  |                   |  | +-------+-------+ |  |
        |  | Monitoring     |  |                   |  +---------|-----+---+  |
        |  +----------------+  |                   |            |     |      |
        +----------------------+                   +------------|-----|------+
                                                                |     |
                   +--------------------------------------------+     |
                   |                                                  |
        +----------v-----------+                        +-------------v------+
        |   MCP Servers        |                        |  Anthropic API     |
        |                      |                        |  (Claude)          |
        |  - Google GKE MCP    |                        +--------------------+
        |  - BigQuery MCP      |
        |  - GitHub MCP        |
        |  - Filesystem MCP    |
        |  - Cloud Logging MCP |
        +----------------------+
```

---

## 10. Key Decisions Log

| # | Decision                                    | Rationale                                     |
|---|---------------------------------------------|-----------------------------------------------|
| 1 | Reject Claude Agent SDK                     | Competing agentic loop, loses corteX value    |
| 2 | Direct Anthropic API as primary LLM path    | Already works (9.0/10), provider-agnostic     |
| 3 | MCP as infrastructure integration layer     | Standardized, vendor-supported, extensible     |
| 4 | GKE Agent Sandbox for runtime               | Purpose-built for AI agents, gVisor, warm pools|
| 5 | Namespace-per-tenant isolation              | Balance of security and operational simplicity |
| 6 | Workload Identity over static keys          | Zero static credentials in cluster             |
| 7 | gVisor over kata-containers                 | Lower overhead, GKE-native support             |
| 8 | Spot instances for agent workloads          | 60% cost reduction, agents are stateless       |
| 9 | Pod Snapshots for state recovery            | Checkpoint/restore without re-running agents   |
|10 | 4-phase rollout over 16 weeks               | Risk-managed, milestone-driven delivery        |

---

## 11. Risks and Mitigations

| Risk                                  | Likelihood | Impact | Mitigation                              |
|---------------------------------------|------------|--------|-----------------------------------------|
| Agent Sandbox API instability (alpha) | Medium     | High   | Pin CRD versions, maintain fork         |
| gVisor performance overhead           | Low        | Medium | Benchmark early, fallback to runc       |
| MCP server compatibility issues       | Medium     | Medium | Tier-based validation, integration tests|
| Spot instance preemption              | High       | Low    | Warm pools, checkpoint/restore          |
| LLM API cost overruns                 | Medium     | High   | Token budgets, model routing, caching   |
| Cross-tenant data leakage            | Low        | Critical | gVisor + NetworkPolicy + audit logs    |

---

## 12. Next Steps

1. **Provision GKE cluster** with private networking and gVisor node pool
2. **Install Agent Sandbox CRDs** from kubernetes-sigs/agent-sandbox
3. **Build corteX container image** optimized for sandbox execution
4. **Implement MCP client module** at `corteX/integrations/mcp/`
5. **Design tenant onboarding automation** (namespace + RBAC + quotas)
6. **Benchmark warm pool latency** (target: sub-500ms agent startup)

---

*This document is a living research artifact. Update as implementation progresses.*
*Last updated: February 2026*
