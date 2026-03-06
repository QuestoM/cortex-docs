# Cloud Agent Operations: Running AI Agents at Scale

## Research Report - February 2026

**Objective**: Design an architecture for running hundreds of AI agents (virtual "employees") powered by corteX + Claude Code at scale on Google Cloud, each with individual identity, personalized configuration, and autonomous task execution.

---

## Table of Contents

1. [Claude Code Deployment Options](#1-claude-code-deployment-options)
2. [Cloud Infrastructure for AI Agents](#2-cloud-infrastructure-for-ai-agents)
3. [Multi-Agent Orchestration at Scale](#3-multi-agent-orchestration-at-scale)
4. [corteX Integration Architecture](#4-cortex-integration-architecture)
5. [Practical Implementation Plan](#5-practical-implementation-plan)
6. [Cost Analysis](#6-cost-analysis)
7. [Key Decisions and Risks](#7-key-decisions-and-risks)

---

## 1. Claude Code Deployment Options

### 1.1 Headless Mode (CLI)

Claude Code can run fully headless in cloud VMs and containers using the `-p` (or `--print`) flag. This is the foundational mechanism for cloud deployment.

**Key syntax:**
```bash
# Basic headless execution
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"

# JSON output for programmatic consumption
claude -p "Summarize this project" --output-format json

# Streaming JSON for real-time monitoring
claude -p "Explain recursion" --output-format stream-json --verbose

# Continue previous conversations
claude -p "Continue the analysis" --resume "$session_id"

# Custom system prompt for role specialization
claude -p "Review this PR" --append-system-prompt "You are a security engineer"
```

**Session management** is supported: capture `session_id` from JSON output, then use `--resume` to continue specific conversations. This is critical for agents that need persistent context.

**Sources:**
- [Claude Code Headless Docs](https://code.claude.com/docs/en/headless)
- [CI/CD and Headless Mode Guide](https://angelo-lima.fr/en/claude-code-cicd-headless-en/)

### 1.2 Claude Agent SDK (Programmatic API)

The **Claude Agent SDK** (renamed from "Claude Code SDK" in September 2025) is the production-grade way to run Claude Code programmatically. It exposes the same tools, agent loop, and context management as Claude Code, available in both **Python** and **TypeScript**.

**Python SDK (recommended for corteX integration):**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

async for message in query(
    prompt="Find and fix the bug in auth.py",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Edit", "Bash"],
        system_prompt="You are an expert Python developer",
        permission_mode="acceptEdits",
        cwd="/home/user/project",
        max_budget_usd=5.0,
        env={"ANTHROPIC_API_KEY": "sk-..."}
    ),
):
    print(message)
```

**Key capabilities:**
- **Built-in tools**: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
- **Custom tools**: Define domain-specific tools via `@tool` decorator and in-process MCP servers
- **Subagents**: Spawn specialized agents for focused subtasks via `AgentDefinition`
- **Hooks**: `PreToolUse`, `PostToolUse`, `Stop`, `UserPromptSubmit` for validation, logging, blocking
- **Sessions**: Maintain context across exchanges, resume/fork sessions
- **Permissions**: Fine-grained tool control, custom permission handlers, sandbox configuration
- **MCP integration**: Connect to external systems (databases, browsers, APIs)
- **Budget control**: `max_budget_usd` parameter caps spending per session

**Two client modes:**

| Feature | `query()` | `ClaudeSDKClient` |
|---------|-----------|-------------------|
| Session | New each time | Reuses session |
| Hooks | Not supported | Supported |
| Custom tools | Not supported | Supported |
| Interrupts | Not supported | Supported |
| Use case | One-off tasks | Continuous conversations |

**Installation:**
```bash
pip install claude-agent-sdk  # Python
npm install @anthropic-ai/claude-agent-sdk  # TypeScript
```

**Current versions** (Feb 2026): Python v0.1.34, TypeScript v0.2.37, with 1.85M+ weekly downloads.

**Sources:**
- [Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Python SDK Reference](https://platform.claude.com/docs/en/agent-sdk/python)
- [GitHub - claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [GitHub - claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

### 1.3 Authentication for Cloud Deployment

**API Key authentication** (recommended for cloud):
```bash
export ANTHROPIC_API_KEY=your-api-key
```

**Third-party provider support:**
- **Amazon Bedrock**: `CLAUDE_CODE_USE_BEDROCK=1` + AWS credentials
- **Google Vertex AI**: `CLAUDE_CODE_USE_VERTEX=1` + Google Cloud credentials
- **Microsoft Azure**: `CLAUDE_CODE_USE_FOUNDRY=1` + Azure credentials

**Important policy**: Anthropic does NOT allow third-party developers to offer claude.ai login or rate limits for their products. All agents must use API key authentication.

**Sources:**
- [Docker Authentication Guide](https://docs.docker.com/ai/sandboxes/claude-code/)
- [GitHub Issue #7100 - Headless Auth](https://github.com/anthropics/claude-code/issues/7100)

### 1.4 Subscription vs API Pricing

#### Subscription Plans (Claude.ai)

| Plan | Price | Claude Code Access | Usage |
|------|-------|-------------------|-------|
| Pro | $20/mo ($17 annual) | Yes | Base usage |
| Max 5x | $100/mo | Yes | 5x Pro usage |
| Max 20x | $200/mo | Yes | 20x Pro usage |
| Team Standard | $25-30/seat/mo | Yes | Team features |
| Team Premium | $125/seat/mo | Yes | 6.25x Pro usage |
| Enterprise | Custom ($500-15K+/mo) | Yes | Custom limits, SSO, SCIM, audit logs |

**Max plan limitations (critical):**
- 5-hour rolling window for rate limits
- Weekly caps to prevent account sharing
- Users report hitting limits after ~2 hours of continuous heavy use (Jan 2026)
- Token consumption running 20-30% higher than expected (Feb 2026)
- NOT suitable for fleet of agents - use API instead

#### API Pricing (per million tokens)

| Model | Input | Output | Batch Input | Batch Output |
|-------|-------|--------|-------------|-------------|
| Claude Opus 4.6 | $5 | $25 | $2.50 | $12.50 |
| Claude Opus 4.5 | $5 | $25 | $2.50 | $12.50 |
| Claude Sonnet 4.5 | $3 | $15 | $1.50 | $7.50 |
| Claude Sonnet 4 | $3 | $15 | $1.50 | $7.50 |
| Claude Haiku 4.5 | $1 | $5 | $0.50 | $2.50 |

**Cost optimization features:**
- **Batch API**: 50% discount, processed within 24 hours
- **Prompt caching**: Cache reads at 10% of input price (e.g., $0.30/MTok for Sonnet)
- **Combined**: Batch + caching can achieve up to 95% discount on input tokens
- **Long context (>200K tokens)**: 2x input price, 1.5x output price

**Verdict: For 50-500 agents, use API pricing, NOT subscriptions.** Subscriptions are per-seat and have unpredictable rate limits. API gives precise cost control, batch discounts, and scales predictably.

**Sources:**
- [Anthropic Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Claude Plans & Pricing](https://claude.com/pricing)
- [Claude Max Token Economics](https://redasgard.medium.com/claude-max-token-economics-the-invisible-meter-3db4de52c47d)
- [Claude API Rate Limits](https://platform.claude.com/docs/en/api/rate-limits)

### 1.5 Claude Code in CI/CD (GitHub Actions)

Anthropic provides an official GitHub Action: [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action).

**Capabilities:**
- Responds to `@claude` mentions in PRs and issues
- Automatic PR reviews on open/update
- Code changes implementation
- CI failure analysis and fixes
- Parallel agent workers ("ultrapilot" launches up to 5 workers)

**Key pattern for fleet operations**: Each agent can have its own GitHub Action workflow, acting as an autonomous contributor.

**Sources:**
- [Claude Code GitHub Actions Docs](https://code.claude.com/docs/en/github-actions)
- [GitHub Marketplace - Claude Code Action](https://github.com/marketplace/actions/claude-code-action-official)

---

## 2. Cloud Infrastructure for AI Agents

### 2.1 Docker Containers (Foundation Layer)

Claude Code has first-class Docker support via **Docker Sandboxes**.

**Official Docker sandbox:**
```bash
docker sandbox run claude ~/my-project
docker sandbox run claude -- "your prompt"  # Headless
```

**Base image features:**
- Ubuntu with Claude Code pre-installed
- Development tools: Docker CLI, GitHub CLI, Node.js, Go, Python 3, Git, ripgrep, jq
- Non-root `agent` user with sudo access
- `--dangerously-skip-permissions` enabled by default
- Private Docker daemon

**Custom Dockerfile for agent fleet:**
```dockerfile
FROM ubuntu:22.04

# Install Node.js and Claude Code
RUN apt-get update && apt-get install -y curl git python3 python3-pip
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs
RUN npm install -g @anthropic-ai/claude-code

# Install Python Agent SDK
RUN pip3 install claude-agent-sdk

# Install corteX SDK
COPY requirements.txt /app/
RUN pip3 install -r /app/requirements.txt

# Agent configuration
COPY agent_config/ /app/config/
COPY agent_runner.py /app/

# Environment
ENV ANTHROPIC_API_KEY=""
ENV AGENT_ID=""
ENV AGENT_ROLE=""

WORKDIR /app
CMD ["python3", "agent_runner.py"]
```

**Sources:**
- [Docker Sandboxes for Claude Code](https://docs.docker.com/ai/sandboxes/claude-code/)
- [Docker Blog - Run Claude Code Safely](https://www.docker.com/blog/docker-sandboxes-run-claude-code-and-other-coding-agents-unsupervised-but-safely/)
- [claudebox - Docker Dev Environment](https://github.com/RchGrav/claudebox)
- [claude-agent-sdk-container](https://github.com/receipting/claude-agent-sdk-container)

### 2.2 Google Cloud Run (Serverless Containers)

**Best for**: Task-based agent execution, event-driven workloads, cost efficiency for bursty usage.

**Pricing (us-central1):**
- vCPU: $0.000024/vCPU-second (~$0.086/hour)
- Memory: $0.0000025/GiB-second
- Requests: $0.40 per million
- Free tier: 180,000 vCPU-seconds/mo, 2M requests/mo

**Architecture pattern:**
```
Task Queue (Cloud Tasks/Pub/Sub)
    --> Cloud Run Service (autoscales 0-N)
        --> Agent Container
            --> Claude Agent SDK
            --> corteX Brain
            --> Task execution
        --> Results to Cloud Storage/Firestore
```

**Pros:**
- Scale to zero (no cost when idle)
- Automatic scaling 0 to 1000+ instances
- Pay only for actual execution time
- Built-in load balancing and HTTPS

**Cons:**
- Max request timeout: 60 minutes (Cloud Run Jobs: 24 hours)
- Cold starts (mitigated with min-instances)
- No persistent state between invocations
- CPU throttled when not handling requests (unless always-on)

**Estimated cost per agent-task:**
- 2 vCPU, 4GB RAM, 10-minute task: ~$0.03 compute + API tokens
- 100 tasks/day per agent: ~$3/day compute

**Sources:**
- [Cloud Run Pricing](https://cloud.google.com/run/pricing)
- [Cloud Run Documentation](https://cloud.google.com/run)

### 2.3 Google Kubernetes Engine (GKE)

**Best for**: Always-on agents, complex orchestration, persistent state, maximum control.

**Key features for agent fleets:**
- **Agent Sandbox on GKE**: Managed gVisor isolation, sub-second latency sandbox execution (90% improvement over cold starts)
- **Fleet Management**: Organize clusters by team/project, Team scopes for resource partitioning
- **Pod Autoscaling**: Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA)
- **In-place Pod Resize** (K8s 1.35 GA): Change CPU/memory without restart
- **Cost Allocation**: Built-in cost breakdown by cluster, namespace, labels

**Architecture pattern:**
```
GKE Cluster
├── Agent Namespace: "developers"
│   ├── Pod: agent-frontend-dev-01 (corteX + Claude SDK)
│   ├── Pod: agent-frontend-dev-02
│   └── Pod: agent-backend-dev-01
├── Agent Namespace: "analysts"
│   ├── Pod: agent-data-analyst-01
│   └── Pod: agent-report-writer-01
├── Shared Services Namespace
│   ├── Agent Registry (API)
│   ├── Task Queue (Redis/Pub/Sub)
│   ├── Memory Store (PostgreSQL/Redis)
│   └── Monitoring (Prometheus/Grafana)
└── Ingress / Load Balancer
```

**GKE pricing:**
- Autopilot: $0.0445/vCPU/hour + $0.0049/GB/hour
- Standard: $0.10/hour/cluster + node compute costs
- E2 nodes: ~$0.033/vCPU/hour, ~$0.004/GB/hour
- Committed Use Discounts (CUDs): up to 57% savings on 3-year commitments

**Sources:**
- [GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)
- [Agentic AI on GKE](https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke)
- [GKE Fleet Management](https://docs.cloud.google.com/kubernetes-engine/docs/fleets-overview)
- [GKE Cost Optimization](https://scaleops.com/blog/gke-cost-optimization-how-to-cut-kubernetes-spend-at-scale-in-2026/)

### 2.4 Google Compute Engine (Persistent VMs)

**Best for**: Long-running agents that need full VM access, development/testing.

**Pricing examples (us-central1, on-demand):**
- e2-small (2 vCPU, 2GB): ~$0.017/hour (~$12/month)
- e2-standard-2 (2 vCPU, 8GB): ~$0.067/hour (~$49/month)
- e2-standard-4 (4 vCPU, 16GB): ~$0.134/hour (~$97/month)

**Not recommended for fleet** due to:
- Fixed cost even when idle
- Manual scaling
- Higher operational overhead
- Better served by GKE or Cloud Run

### 2.5 Google Cloud Batch

**Best for**: Batch processing tasks, large-scale one-time operations.

**Pattern**: Submit agent tasks as batch jobs, Google manages compute provisioning.

**Pros:**
- No cluster management
- Spot VM support (60-91% cheaper)
- Automatic retry and scheduling

### 2.6 Alternative: AWS Bedrock

AWS Bedrock provides managed access to Claude models with:
- IAM-based authentication
- AWS Cost Explorer integration
- Provisioned Throughput for guaranteed capacity ($44/hour/model-unit, $22/hour with 6-month commit)
- Intelligent Prompt Routing between model sizes
- Batch processing at 50% discount
- No API key management (uses AWS credentials)

**Pricing**: Same as Anthropic API, but billed through AWS.

**Advantage**: If already on AWS, simpler billing and integrated monitoring.

**Sources:**
- [AWS Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [Claude on Amazon Bedrock](https://aws.amazon.com/bedrock/anthropic/)

---

## 3. Multi-Agent Orchestration at Scale

### 3.1 Claude's Built-in Multi-Agent Features

**Swarm mode** (experimental, Jan 2026): A team lead agent plans and delegates to specialist background agents. Agents share a task board, coordinate via messaging, work in parallel.

**Subagents** (Agent SDK): Define specialized agents programmatically:
```python
options = ClaudeAgentOptions(
    allowed_tools=["Read", "Glob", "Grep", "Task"],
    agents={
        "code-reviewer": AgentDefinition(
            description="Expert code reviewer",
            prompt="Analyze code quality and suggest improvements.",
            tools=["Read", "Glob", "Grep"],
            model="sonnet",  # Use cheaper model for this role
        ),
        "test-writer": AgentDefinition(
            description="Test generation specialist",
            prompt="Write comprehensive tests for the given code.",
            tools=["Read", "Write", "Bash"],
            model="sonnet",
        ),
    },
)
```

### 3.2 Community Orchestration Frameworks

**Claude Flow** (leading open-source orchestrator):
- 54+ specialized agent types
- Distributed swarm intelligence
- Shared memory and consensus
- 84.8% SWE-Bench solve rate
- 250% improvement in effective subscription capacity
- 75-80% token reduction
- ~500K downloads, ~100K monthly active users
- [GitHub: ruvnet/claude-flow](https://github.com/ruvnet/claude-flow)

**Oh My Claude Code**: Agent swarm orchestration
- [Medium: Agent Swarm Testing](https://medium.com/@joe.njenga/i-tested-oh-my-claude-code-the-only-agents-swarm-orchestration-you-need-7338ad92c00f)

### 3.3 Agent Identity and Configuration Management

Each agent needs:

1. **Unique identity**: Agent ID, name, role, department
2. **Personalized brain state**: corteX weights, memory, goal history
3. **Tool access**: Role-specific tool permissions
4. **Knowledge base**: Shared + role-specific knowledge
5. **Session history**: Persistent conversation context

**Proposed configuration structure:**
```yaml
# agent_config/agent-frontend-dev-01.yaml
agent:
  id: "agent-frontend-dev-01"
  name: "Avi"
  role: "Senior Frontend Developer"
  department: "Engineering"
  team: "Web Platform"

cortex:
  brain_weights: "profiles/frontend-dev-senior.json"
  memory_store: "postgresql://memory-db/agent-frontend-dev-01"
  goal_tracking: true
  loop_prevention: true

claude:
  model: "claude-sonnet-4.5"
  fallback_model: "claude-haiku-4.5"
  max_budget_per_task_usd: 2.0
  max_budget_per_day_usd: 50.0
  system_prompt_preset: "claude_code"
  system_prompt_append: |
    You are Avi, a senior frontend developer at Questo.
    You specialize in React, TypeScript, and Next.js.
    Follow the team's coding standards in CLAUDE.md.
  allowed_tools:
    - Read
    - Write
    - Edit
    - Bash
    - Glob
    - Grep
    - WebSearch

workspace:
  repo: "github.com/QuestoM/web-platform"
  branch_prefix: "agent/avi/"

limits:
  max_concurrent_tasks: 3
  max_daily_tasks: 50
  max_tokens_per_day: 5_000_000
```

### 3.4 Task Routing and Workload Distribution

**Architecture:**
```
                    ┌─────────────────────┐
                    │   Task Manager API   │
                    │  (FastAPI / Cloud    │
                    │   Functions)         │
                    └──────┬──────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼────┐ ┌────▼────┐
        │ Priority   │ │ Skill  │ │ Load    │
        │ Queue      │ │ Router │ │Balancer │
        │(urgent)    │ │(match) │ │(round-  │
        └─────┬──────┘ └───┬────┘ │robin)   │
              │            │      └────┬────┘
              └────────────┼───────────┘
                           │
                    ┌──────▼──────────┐
                    │  Agent Registry  │
                    │  (who can do     │
                    │   what, who is   │
                    │   available)     │
                    └──────┬──────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
    │ Agent 1 │      │ Agent 2 │      │ Agent N │
    │ (Dev)   │      │(Analyst)│      │ (QA)    │
    └─────────┘      └─────────┘      └─────────┘
```

**Task routing strategies:**
1. **Skill-based routing**: Match task requirements to agent capabilities
2. **Priority queues**: Urgent tasks get assigned first
3. **Load balancing**: Distribute evenly across available agents
4. **Affinity routing**: Route related tasks to same agent (context reuse)
5. **Cost-aware routing**: Use cheaper models (Haiku) for simple tasks

### 3.5 Shared Context and Knowledge Bases

**Tiered knowledge architecture:**

| Layer | Scope | Storage | Example |
|-------|-------|---------|---------|
| Company KB | All agents | Cloud Storage + Vector DB | Company policies, brand guidelines |
| Department KB | Department agents | Namespace-scoped storage | Engineering standards, design system |
| Team KB | Team agents | Team-scoped storage | Sprint goals, architecture decisions |
| Agent Memory | Individual agent | corteX memory stores | Personal task history, learned preferences |

**Implementation with corteX:**
- **Working memory**: Current task context (corteX `memory/working`)
- **Short-term memory**: Recent interactions (corteX `memory/short_term`)
- **Long-term memory**: Persistent knowledge (corteX `memory/long_term`)
- **Episodic memory**: Past task experiences (corteX `memory/episodic`)

### 3.6 Monitoring and Observability

**The state of AI agent observability in 2026:**
- 89% of organizations have implemented observability for agents
- Quality issues are the #1 production barrier (32%)
- Agent observability differs from traditional monitoring due to non-deterministic behavior

**Recommended observability stack:**

| Layer | Tool | Purpose |
|-------|------|---------|
| Infrastructure | Google Cloud Monitoring + Prometheus | VM/container metrics |
| Application | OpenTelemetry + Grafana | Agent execution traces |
| AI-specific | Braintrust or Helicone | LLM call monitoring, cost tracking |
| Logs | Cloud Logging + Loki | Agent decision logs |
| Alerts | PagerDuty / Cloud Alerting | Budget overruns, failures |
| Dashboard | Grafana | Fleet-wide agent status |

**Key metrics to track:**
- Token consumption per agent per day
- Task completion rate and duration
- API error rate and retry frequency
- Budget utilization (actual vs. allocated)
- Agent "drift" (deviation from goals)
- Cost per completed task

**Sources:**
- [Top 5 AI Agent Observability Platforms 2026](https://o-mega.ai/articles/top-5-ai-agent-observability-platforms-the-ultimate-2026-guide)
- [Best AI Observability Tools 2026](https://www.braintrust.dev/articles/best-ai-observability-tools-2026)
- [HBR: Companies Need Agent Managers](https://hbr.org/2026/02/to-thrive-in-the-ai-era-companies-need-agent-managers)

---

## 4. corteX Integration Architecture

### 4.1 corteX as the Agent Brain

Each AI agent in the fleet runs a corteX instance as its "brain", while the Claude Agent SDK provides the LLM interface and tool execution. This separation gives us:

```
┌──────────────────────────────────────────────┐
│                 AI Agent                      │
│                                              │
│  ┌────────────────────┐  ┌────────────────┐  │
│  │    corteX Brain     │  │  Claude Agent  │  │
│  │                    │  │     SDK        │  │
│  │  - Weights system  │  │               │  │
│  │  - Goal tracker    │  │  - LLM calls  │  │
│  │  - Loop prevention │  │  - Tool exec  │  │
│  │  - Feedback loop   │  │  - File I/O   │  │
│  │  - Prediction      │  │  - Web access │  │
│  │  - Plasticity      │  │  - Sessions   │  │
│  │  - Memory system   │  │               │  │
│  └────────┬───────────┘  └───────┬────────┘  │
│           │                      │           │
│           └──────┬───────────────┘           │
│                  │                           │
│         ┌────────▼─────────┐                 │
│         │   Orchestrator   │                 │
│         │  (Runtime layer) │                 │
│         └──────────────────┘                 │
└──────────────────────────────────────────────┘
```

### 4.2 Role-Based Brain Profiles

corteX's weight system enables each agent to have a personalized "brain" based on their role:

```python
# Developer Agent - higher weights on code-related pathways
developer_profile = BrainProfile(
    synaptic_weights={
        "code_analysis": 0.95,
        "test_generation": 0.85,
        "documentation": 0.70,
        "architecture": 0.80,
        "debugging": 0.90,
    },
    goal_tracking=GoalConfig(
        persistence=0.9,  # Very goal-focused
        drift_threshold=0.15,
    ),
    loop_prevention=LoopConfig(
        state_hash_depth=5,
        max_similar_actions=3,
    ),
    plasticity=PlasticityConfig(
        learning_rate=0.02,  # Slow, stable learning
        decay_rate=0.001,
    ),
)

# Analyst Agent - higher weights on data and reasoning
analyst_profile = BrainProfile(
    synaptic_weights={
        "data_analysis": 0.95,
        "pattern_recognition": 0.90,
        "report_writing": 0.85,
        "visualization": 0.80,
        "statistical_reasoning": 0.90,
    },
    goal_tracking=GoalConfig(
        persistence=0.85,
        drift_threshold=0.20,  # More exploration allowed
    ),
    plasticity=PlasticityConfig(
        learning_rate=0.05,  # Faster adaptation
        decay_rate=0.002,
    ),
)
```

### 4.3 Integrating corteX with Claude Agent SDK

The integration uses corteX as the decision-making layer and Claude Agent SDK as the execution layer:

```python
import asyncio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions, tool, create_sdk_mcp_server
from cortex import CortexSDK

# Initialize corteX brain
brain = CortexSDK(
    provider="anthropic",
    api_key=os.environ["ANTHROPIC_API_KEY"],
    config={
        "brain": {"weights_enabled": True, "plasticity_enabled": True},
        "memory": {"working": True, "episodic": True, "long_term": True},
        "safety": {"loop_prevention": True, "goal_tracking": True},
    }
)

# Define corteX tools for the Claude Agent SDK
@tool("cortex_evaluate_goal", "Evaluate if current action aligns with goal", {
    "action": str, "goal": str
})
async def evaluate_goal(args):
    result = await brain.engine.goal_tracker.evaluate(
        action=args["action"], goal=args["goal"]
    )
    return {"content": [{"type": "text", "text": f"Goal alignment: {result.score}"}]}

@tool("cortex_check_loop", "Check if agent is in a loop", {
    "state_description": str
})
async def check_loop(args):
    is_loop = await brain.engine.loop_detector.check(args["state_description"])
    return {"content": [{"type": "text", "text": f"Loop detected: {is_loop}"}]}

# Create MCP server with corteX tools
cortex_server = create_sdk_mcp_server(
    name="cortex_brain",
    tools=[evaluate_goal, check_loop],
)

# Configure Claude Agent SDK with corteX integration
options = ClaudeAgentOptions(
    mcp_servers={"brain": cortex_server},
    allowed_tools=[
        "Read", "Write", "Edit", "Bash", "Glob", "Grep",
        "mcp__brain__cortex_evaluate_goal",
        "mcp__brain__cortex_check_loop",
    ],
    system_prompt={
        "type": "preset",
        "preset": "claude_code",
        "append": """
        You have a corteX brain that helps you make better decisions.
        Before taking significant actions, use cortex_evaluate_goal to check alignment.
        If you feel stuck, use cortex_check_loop to verify you're not looping.
        """
    },
    max_budget_usd=5.0,
    permission_mode="acceptEdits",
)
```

### 4.4 On-Prem Advantage

corteX's on-prem architecture is critical for this fleet:

1. **No external dependencies**: Brain logic runs locally in each container
2. **Data sovereignty**: Agent memory and weights stay within the infrastructure
3. **Cost control**: Brain operations are free (only LLM API calls cost money)
4. **Customizability**: Each agent's brain can be fine-tuned without vendor lock-in
5. **Reliability**: Brain continues functioning even if LLM API has issues (degraded mode)

### 4.5 4-Tier Feedback System at Scale

corteX's feedback hierarchy enables organizational learning:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Tier 1: Direct Feedback                            │
│  Agent self-evaluation on task completion            │
│  (Immediate, per-task)                              │
│                                                     │
│  Tier 2: User Insights                              │
│  Human manager reviews agent output                 │
│  (Daily/weekly review cycles)                       │
│                                                     │
│  Tier 3: Enterprise Feedback                        │
│  Cross-agent learning within the organization       │
│  (Agent A's learnings shared with Agent B)          │
│                                                     │
│  Tier 4: Global Feedback                            │
│  Aggregated patterns across all Questo deployments  │
│  (Future: multi-tenant learning)                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 5. Practical Implementation Plan

### 5.1 Conceptual Architecture Diagram

```
                    ┌─────────────────────────────┐
                    │      Questo Cloud           │
                    │    (Google Cloud Platform)   │
                    └──────────┬──────────────────┘
                               │
            ┌──────────────────┼──────────────────────┐
            │                  │                      │
   ┌────────▼────────┐ ┌──────▼──────┐ ┌─────────────▼──────┐
   │  Control Plane   │ │  Data Plane │ │  Observability     │
   │                  │ │             │ │                    │
   │ - Agent Registry │ │ - GKE      │ │ - Cloud Monitoring │
   │ - Config Store   │ │   Cluster  │ │ - Grafana          │
   │ - Task Manager   │ │ - Cloud    │ │ - Braintrust       │
   │ - Budget Manager │ │   Run Jobs │ │ - Cost Dashboard   │
   │ - API Gateway    │ │ - Cloud    │ │ - Alert Manager    │
   │                  │ │   Storage  │ │                    │
   └────────┬─────────┘ └──────┬─────┘ └────────────────────┘
            │                  │
            └──────────┬───────┘
                       │
         ┌─────────────▼──────────────┐
         │       GKE Cluster          │
         │                            │
         │  ┌──────────────────────┐  │
         │  │  Agent Pool          │  │
         │  │                      │  │
         │  │  ┌────┐ ┌────┐      │  │
         │  │  │ A1 │ │ A2 │ ...  │  │
         │  │  └────┘ └────┘      │  │
         │  │  ┌────┐ ┌────┐      │  │
         │  │  │ A3 │ │ A4 │ ...  │  │
         │  │  └────┘ └────┘      │  │
         │  └──────────────────────┘  │
         │                            │
         │  ┌──────────────────────┐  │
         │  │  Shared Services     │  │
         │  │  - PostgreSQL        │  │
         │  │  - Redis             │  │
         │  │  - Vector DB         │  │
         │  │  - Message Queue     │  │
         │  └──────────────────────┘  │
         └────────────────────────────┘
                       │
                       │ API calls
                       ▼
              ┌────────────────┐
              │   Anthropic    │
              │   Claude API   │
              └────────────────┘
```

### 5.2 Phased Rollout Plan

#### Phase 0: Foundation (Weeks 1-2)
**Goal**: Prove the concept with a single agent

- [ ] Set up Google Cloud project with billing alerts
- [ ] Create Docker image with corteX + Claude Agent SDK
- [ ] Build agent runner script (Python) that:
  - Loads agent config from YAML
  - Initializes corteX brain with role-specific weights
  - Creates Claude Agent SDK session
  - Executes a task and reports results
- [ ] Test with one agent executing simple code review tasks
- [ ] Establish baseline metrics: cost per task, time per task, quality

**Estimated cost**: $50-100/month (1 agent, light usage, API costs)

#### Phase 1: Small Team (Weeks 3-6)
**Goal**: 5-10 agents working on real tasks

- [ ] Deploy GKE Autopilot cluster
- [ ] Build Agent Registry service (FastAPI)
- [ ] Build Task Manager with priority queues (Cloud Tasks or Redis)
- [ ] Create 3-5 role profiles (Developer, Reviewer, Analyst, Writer, QA)
- [ ] Deploy 5-10 agent pods with distinct configurations
- [ ] Implement basic monitoring dashboard (Grafana)
- [ ] Add budget controls (max per agent per day)
- [ ] Test with real Questo internal tasks

**Estimated cost**: $200-500/month (compute) + $500-2,000/month (API)

#### Phase 2: Department Scale (Weeks 7-12)
**Goal**: 50 agents across multiple departments

- [ ] Scale GKE cluster to handle 50 concurrent agent pods
- [ ] Implement shared knowledge base (Vector DB: Weaviate or Pinecone)
- [ ] Build cross-agent learning (Tier 3 feedback)
- [ ] Add task routing intelligence (skill-based + cost-aware)
- [ ] Implement agent health monitoring and auto-restart
- [ ] Build management dashboard for department heads
- [ ] Add Batch API integration for non-urgent tasks (50% savings)
- [ ] Implement prompt caching for repeated contexts (90% savings on cache hits)

**Estimated cost**: $1,000-2,000/month (compute) + $5,000-15,000/month (API)

#### Phase 3: Company Scale (Months 4-6)
**Goal**: 100-500 agents as a virtual workforce

- [ ] Multi-cluster GKE setup for reliability
- [ ] Advanced task orchestration (multi-step workflows)
- [ ] Agent-to-agent communication protocols
- [ ] Self-healing: agents detect and recover from failures
- [ ] Cost optimization: dynamic model selection (Haiku for simple, Sonnet for complex, Opus for critical)
- [ ] HR-like dashboard: agent performance reviews, utilization metrics
- [ ] Security audit and compliance review
- [ ] SLA monitoring per agent role

**Estimated cost**: $3,000-8,000/month (compute) + $15,000-75,000/month (API)

### 5.3 Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Container orchestration | GKE Autopilot | Agent pod management |
| Agent runtime | Python + Claude Agent SDK | LLM interaction |
| Agent brain | corteX SDK | Decision-making, memory, weights |
| Task queue | Cloud Tasks + Redis | Task distribution |
| Agent registry | FastAPI + PostgreSQL | Agent config and status |
| Shared memory | PostgreSQL + pgvector | Long-term knowledge |
| Cache | Redis | Session state, prompt cache |
| Object storage | Cloud Storage | Artifacts, logs, checkpoints |
| Monitoring | Cloud Monitoring + Grafana | Infrastructure metrics |
| AI observability | Braintrust or Helicone | LLM cost and quality tracking |
| CI/CD | Cloud Build + GitHub Actions | Agent image builds |
| Secrets | Secret Manager | API keys, credentials |

---

## 6. Cost Analysis

### 6.1 API Cost Estimates

#### Per-Agent Token Usage (estimated daily)

| Agent Role | Model | Input Tokens/Day | Output Tokens/Day | Daily API Cost |
|-----------|-------|-------------------|-------------------|---------------|
| Developer | Sonnet 4.5 | 2M | 500K | $13.50 |
| Code Reviewer | Sonnet 4.5 | 1.5M | 300K | $9.00 |
| Data Analyst | Sonnet 4.5 | 1M | 400K | $9.00 |
| Report Writer | Haiku 4.5 | 2M | 1M | $7.00 |
| QA Tester | Haiku 4.5 | 1M | 300K | $2.50 |
| Simple Worker | Haiku 4.5 | 500K | 200K | $1.50 |

#### Fleet Cost Projections (API only, monthly)

| Fleet Size | Mix | Est. Monthly API Cost | With Batch (50%) | With Caching (est.) |
|-----------|-----|----------------------|-------------------|---------------------|
| 10 agents | 5 dev, 3 analyst, 2 QA | $5,000-8,000 | $3,000-5,000 | $2,000-3,500 |
| 50 agents | 20 dev, 15 analyst, 10 QA, 5 writer | $20,000-35,000 | $12,000-20,000 | $8,000-15,000 |
| 100 agents | Mixed | $40,000-70,000 | $24,000-42,000 | $16,000-30,000 |
| 500 agents | Mixed | $150,000-300,000 | $90,000-180,000 | $60,000-120,000 |

**Cost optimization levers:**
1. **Model selection**: Use Haiku ($1/$5 MTok) for simple tasks, Sonnet ($3/$15) for complex, Opus ($5/$25) only for critical decisions = 40-60% savings
2. **Batch API**: 50% off for non-urgent tasks (target: 40% of workload)
3. **Prompt caching**: 90% off on cached context (target: high cache hit ratio with shared system prompts)
4. **Combined**: Batch + caching = up to 95% savings on input tokens
5. **Smart routing**: Don't send simple lookups to the LLM at all
6. **Enterprise negotiation**: Contact sales@anthropic.com for volume discounts

### 6.2 Infrastructure Cost Estimates (Google Cloud, monthly)

| Component | 10 Agents | 50 Agents | 100 Agents | 500 Agents |
|-----------|-----------|-----------|------------|------------|
| GKE Autopilot | $150 | $500 | $1,000 | $4,000 |
| PostgreSQL (Cloud SQL) | $50 | $100 | $200 | $500 |
| Redis (Memorystore) | $50 | $100 | $200 | $500 |
| Cloud Storage | $10 | $50 | $100 | $300 |
| Networking | $20 | $50 | $100 | $300 |
| Monitoring | $20 | $50 | $100 | $300 |
| **Total Infra** | **$300** | **$850** | **$1,700** | **$5,900** |

### 6.3 Total Cost Summary (monthly)

| Scale | API (optimized) | Infrastructure | Total Monthly | Per Agent/Month |
|-------|----------------|----------------|---------------|-----------------|
| 10 agents | $3,000 | $300 | $3,300 | $330 |
| 50 agents | $12,000 | $850 | $12,850 | $257 |
| 100 agents | $24,000 | $1,700 | $25,700 | $257 |
| 500 agents | $90,000 | $5,900 | $95,900 | $192 |

**Note**: These are estimates based on moderate usage. Actual costs depend heavily on task complexity, token usage patterns, and optimization effectiveness. The per-agent cost decreases at scale due to shared infrastructure and better optimization opportunities.

**Comparison**: A human employee costs $5,000-15,000+/month. An AI agent at $200-330/month represents a 15-75x cost advantage, even before considering 24/7 availability and no ramp-up time.

---

## 7. Key Decisions and Risks

### 7.1 Critical Decisions

| Decision | Options | Recommendation | Rationale |
|----------|---------|---------------|-----------|
| **LLM Access** | API vs Subscription | **API** | Precise cost control, batch discounts, scales predictably |
| **Container Orchestration** | GKE vs Cloud Run vs VMs | **GKE Autopilot** | Balance of control, autoscaling, cost efficiency |
| **Agent SDK** | Claude Agent SDK vs raw API | **Claude Agent SDK** | Built-in tools, sessions, hooks - don't reinvent |
| **Brain Engine** | corteX integrated vs separate | **Integrated** | Custom tools expose brain to Claude, single container |
| **Task Queue** | Cloud Tasks vs Redis vs Pub/Sub | **Cloud Tasks** | Managed, retry logic, integrates with Cloud Run |
| **Database** | Cloud SQL vs AlloyDB vs Firestore | **Cloud SQL (PostgreSQL)** | pgvector support, familiar, cost-effective |
| **Provider** | Anthropic API vs Bedrock vs Vertex | **Anthropic API** | Direct relationship, latest features, Bedrock as fallback |

### 7.2 Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|-----------|
| **API rate limits** | Agents blocked | High | Multiple API keys, request queuing, Bedrock fallback |
| **Cost overrun** | Budget exceeded | High | Per-agent budgets, alerts, automatic shutoff |
| **Agent loops** | Wasted tokens | Medium | corteX loop prevention, max_turns, budget caps |
| **API outage** | All agents stop | Medium | Bedrock fallback, graceful degradation, task queuing |
| **Quality drift** | Bad outputs | Medium | corteX goal tracking, automated quality checks, human review |
| **Security breach** | Data leak | Low | Sandboxed containers, network policies, secret rotation |
| **Anthropic policy change** | Service disruption | Low | Multi-provider support (Bedrock, Vertex), corteX provider-agnostic |

### 7.3 Next Steps (Immediate Actions)

1. **Get Anthropic API key** at Tier 2+ (at least $40 spend required for Tier 2)
2. **Set up Google Cloud project** with billing alerts
3. **Build Phase 0 prototype**: Single agent in Docker, running tasks via Claude Agent SDK + corteX
4. **Benchmark**: Measure actual token usage for typical tasks to validate cost estimates
5. **Contact Anthropic sales** (sales@anthropic.com) to discuss enterprise pricing and rate limits for fleet deployment

---

## Appendix A: Key Reference Links

### Official Documentation
- [Claude Code Headless Mode](https://code.claude.com/docs/en/headless)
- [Claude Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK - Python Reference](https://platform.claude.com/docs/en/agent-sdk/python)
- [Claude Agent SDK - TypeScript Reference](https://platform.claude.com/docs/en/agent-sdk/typescript)
- [Anthropic API Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Claude Plans & Pricing](https://claude.com/pricing)
- [Claude API Rate Limits](https://platform.claude.com/docs/en/api/rate-limits)
- [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Docker - Configure Claude Code](https://docs.docker.com/ai/sandboxes/claude-code/)
- [Claude Code Dev Containers](https://code.claude.com/docs/en/devcontainer)

### GitHub Repositories
- [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)
- [claude-code-action (GitHub Action)](https://github.com/anthropics/claude-code-action)
- [claude-flow (Multi-Agent Orchestrator)](https://github.com/ruvnet/claude-flow)
- [claudebox (Docker Environment)](https://github.com/RchGrav/claudebox)
- [claude-agent-sdk-container](https://github.com/receipting/claude-agent-sdk-container)

### Google Cloud
- [GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)
- [GKE Fleet Management](https://docs.cloud.google.com/kubernetes-engine/docs/fleets-overview)
- [Agentic AI on GKE](https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke)
- [Cloud Run Pricing](https://cloud.google.com/run/pricing)
- [Compute Engine Pricing](https://cloud.google.com/compute/vm-instance-pricing)

### AWS Alternative
- [AWS Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/)
- [Claude on Amazon Bedrock](https://aws.amazon.com/bedrock/anthropic/)

### Industry Analysis
- [Deloitte: The Agentic Reality Check](https://www.deloitte.com/us/en/insights/topics/technology-management/tech-trends/2026/agentic-ai-strategy.html)
- [HBR: Companies Need Agent Managers](https://hbr.org/2026/02/to-thrive-in-the-ai-era-companies-need-agent-managers)
- [Salesforce: 12 AI Agents Per Company Is Just the Beginning](https://beam.ai/agentic-insights/12-ai-agents-per-company-salesforce-2026-report)
- [AI Agent Observability Platforms 2026](https://o-mega.ai/articles/top-5-ai-agent-observability-platforms-the-ultimate-2026-guide)
- [Best AI Observability Tools 2026](https://www.braintrust.dev/articles/best-ai-observability-tools-2026)

---

*Report generated: February 14, 2026*
*For corteX project by Questo*
