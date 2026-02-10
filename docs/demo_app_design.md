# corteX Demo System -- Architecture & Design Document

**Version:** 2.0
**Date:** 2026-02-10
**Status:** Blueprint (pre-implementation, platform selected)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Demo Scenarios](#2-demo-scenarios)
3. [Architecture](#3-architecture)
4. [UI Design](#4-ui-design)
5. [Brain Dashboard Layout](#5-brain-dashboard-layout)
6. [Tech Stack](#6-tech-stack)
7. [Rate Limit Strategy](#7-rate-limit-strategy)
8. [File Structure](#8-file-structure)
9. [Implementation Phases](#9-implementation-phases)

---

## 1. Executive Summary

The corteX demo system is a standalone application that showcases the full depth of the corteX SDK's 20 brain-inspired components in a premium, minimalist interface. It serves three audiences:

- **Prospective customers** seeing corteX for the first time (quick demos, < 5 min)
- **Technical evaluators** assessing SDK depth (medium demos, 30 min)
- **Internal developers** running long soak tests and observing emergent behavior (long demos, 1-4 hours)

The demo must handle Gemini API Tier-1 rate limits (25 RPM / 250 RPD) gracefully, never crashing, and clearly communicating rate-limit state to the user.

### Design Principles

1. **Show, don't tell.** Every brain component has a live visualization.
2. **Apple-level aesthetics.** Grey/white/black palette, indicator colors only for status.
3. **Fail gracefully.** Rate limits are a feature of the demo (shows the SDK's resilience), not a bug.
4. **SDK-first.** The demo imports from `corteX.*` exactly as a customer would. Zero private API usage.
5. **Self-contained.** One `pip install` + `npm install` gets everything running locally.

---

## 2. Demo Scenarios

### 2.1 Quick Demos (< 5 minutes)

These run in under 5 minutes and require fewer than 15 Gemini API calls. Designed for live presentations.

#### Scenario Q1: "Learning Your Style" (Feedback + Adaptation + Plasticity)

**Duration:** 2-3 minutes | **API calls:** ~8 | **Components highlighted:** FeedbackEngine, AdaptationFilter, PlasticityManager, WeightEngine

**Flow:**
1. User sends a detailed question. Agent responds verbosely.
2. User responds with "just the code please" (frustration signal).
3. Dashboard shows: Tier1DirectFeedback detects `BREVITY_PREFERENCE` + `FRUSTRATION`. AdaptationFilter fires `is_novel=True`. PlasticityManager applies Hebbian rule. WeightEngine `verbosity` weight drops from 0.0 to -0.25.
4. Agent's next response is concise code-only.
5. User says "perfect" (satisfaction signal). LTP fires. Autonomy weight nudges up.
6. Dashboard shows weight convergence, critical period multiplier decaying, and homeostatic regulation preventing extreme weights.

**What it proves:** corteX learns user preferences implicitly without ever asking "was this helpful?" -- and the learning is visible, explainable, and principled (Bayesian + Prospect Theory).

#### Scenario Q2: "Tool Selection Under Uncertainty" (Bayesian + Thompson Sampling)

**Duration:** 2-3 minutes | **API calls:** ~10 | **Components highlighted:** WeightEngine (ToolPreferenceWeights), BayesianToolSelector, ProspectTheoreticUpdater, PopulationDecoder

**Flow:**
1. System registers 3 tools: `web_search`, `code_interpreter`, `calculator`.
2. A series of 8 tasks are submitted. For each task:
   - Thompson Sampling draws from Beta posteriors to select a tool.
   - Tool executes (simulated success/failure with realistic probabilities).
   - Beta posteriors update. Gamma latency posteriors update.
   - Prospect-theoretic loss aversion amplifies failure signals 2.25x.
3. Dashboard shows Beta distributions narrowing in real-time, Thompson samples exploring then exploiting, and the population decoder aggregating votes.
4. Final state: `code_interpreter` dominates for coding tasks, `web_search` for research, with uncertainty intervals clearly visible.

**What it proves:** corteX replaces heuristic tool selection with mathematically principled exploration/exploitation that converges to optimal policy.

#### Scenario Q3: "Prediction Error Learning" (PredictionEngine + Surprise)

**Duration:** 3-4 minutes | **API calls:** ~12 | **Components highlighted:** PredictionEngine, SurpriseSignal, DualProcessRouter, CalibrationTracker

**Flow:**
1. Agent predicts outcomes before each action (confidence, latency, quality).
2. First 5 tasks: predictions are rough (high surprise). Dashboard shows large prediction error bars.
3. Tasks 6-10: predictions improve (low surprise). Calibration ECE drops.
4. Task 11: An unexpected failure (tool timeout). Massive surprise signal.
5. DualProcessRouter escalates from System 1 to System 2 (visible routing change).
6. CalibrationTracker ECE spikes. Platt scaling adjusts. MetaCognitionMonitor fires.
7. Agent recovers with System 2 careful reasoning.

**What it proves:** corteX implements predictive coding -- the brain's fundamental learning algorithm. The system learns from prediction errors, not just outcomes.

---

### 2.2 Medium Demos (30 minutes)

These run for 30 minutes and use 40-80 API calls. Designed for technical deep-dives.

#### Scenario M1: "Multi-Tool Research Project" (Full Orchestration)

**Duration:** 30 minutes | **API calls:** ~60 | **Components highlighted:** GoalTracker, CorticalContextEngine, MemoryFabric, CrossModalAssociator, ProactivePredictionEngine, ResourceHomunculus

**Flow:**
1. User assigns a multi-step research goal: "Research the top 3 AI agent frameworks, compare their architectures, and write a summary report."
2. Agent decomposes into sub-goals (GoalTracker sets plan).
3. Over 15-20 steps:
   - Agent uses `web_search` to research each framework.
   - CorticalContextEngine manages the growing context: L0 verbatim for recent steps, L1 observation masking for older tool outputs, L2 summaries for early research.
   - CrossModalAssociator binds: `code:langchain` <-> `documentation:agent_patterns` <-> `error_pattern:memory_overflow`. Spreading activation retrieves related items.
   - ProactivePredictionEngine predicts: after "research framework A", user likely wants "research framework B". Pre-warms tools.
   - MemoryFabric promotes important findings from working memory to semantic memory.
   - ResourceHomunculus allocates more tokens to research (high-frequency) and less to formatting (low-frequency).
4. GoalTracker shows progress bar advancing, drift score staying low.
5. At step 15, a simulated loop is detected (agent revisiting same source). State hash collision triggers replan.
6. Final consolidation: weights are consolidated, memory fabric runs sleep-like consolidation.

**What it proves:** corteX can orchestrate complex multi-step workflows with context management that scales to thousands of steps, while cross-modal associations and proactive predictions reduce latency and improve accuracy.

#### Scenario M2: "Enterprise Safety Escalation" (Game Theory + DualProcess)

**Duration:** 20 minutes | **API calls:** ~40 | **Components highlighted:** DualProcessRouter, MinimaxSafetyGuard, ReputationSystem, NashRoutingOptimizer, ShapleyAttributor, TruthfulScoringMechanism, EnterpriseWeights

**Flow:**
1. Configure enterprise rules: `safety_strictness=0.8`, `data_sensitivity=0.7`, compliance rules `["SOC2", "GDPR"]`.
2. Series of tasks with increasing risk levels:
   - Task 1 (low risk): Simple question. System 1 handles it. No escalation.
   - Task 3 (medium risk): Data query touching PII. DualProcessRouter detects `enterprise_safety > 0.8`. Escalates to System 2.
   - Task 5 (high risk): Code execution that could modify production data. MinimaxSafetyGuard activates. Selects action minimizing worst-case loss.
   - Task 8: A tool starts failing repeatedly. ReputationSystem quarantines it after 3 consecutive failures. Trust score drops to 0.0.
3. Dashboard shows:
   - System 1/2 routing split ratio (aim for ~70/30).
   - Reputation trust scores with quarantine indicators.
   - Nash equilibrium convergence for model routing.
   - Shapley credit attribution for multi-tool outcomes.
   - TruthfulScoringMechanism credibility scores.

**What it proves:** corteX provides enterprise-grade safety through game-theoretic primitives, not just keyword filtering. The dual-process architecture mirrors human cognitive control.

---

### 2.3 Long Demos (1-4 hours)

These run for hours and use 100-250 API calls (respecting daily limits). Designed for soak testing and observing emergent behavior.

#### Scenario L1: "Deep Learning Curves" (Plasticity + Calibration + Concept Graph)

**Duration:** 2-4 hours | **API calls:** ~200 (paced at ~2 RPM) | **Components highlighted:** PlasticityManager (all rules), ContinuousCalibrationEngine, ConceptGraph, MapReorganizer, ComponentSimulator

**Flow:**
1. System processes a sustained stream of tasks across 5 domains: coding, debugging, documentation, testing, deployment.
2. Over hours, the demo shows:
   - **Plasticity curves:** LTP streaks forming for successful tool-task pairings. LTD weakening failed patterns. Critical period multiplier decaying from 2.0x to 0.5x.
   - **Calibration health:** ECE tracking across 5 domains. Platt scaling parameters (a, b) evolving. MetaCognitionMonitor detecting oscillation and recommending learning rate adjustments.
   - **Concept graph growth:** New concepts emerge from repeated co-occurrences. Grandmother cells form for stable, high-frequency concepts. Spreading activation reveals hidden connections between domains.
   - **Territory reorganization:** MapReorganizer detects that "debugging" territory has grown while "documentation" has shrunk (reflecting usage patterns). Triggers cortical reorganization -- like visual cortex being colonized by touch in blind individuals.
   - **Component simulator:** Digital twin runs parallel simulations of alternative weight configurations. A/B test results show which configuration would have performed better.
3. Every 20 minutes, the system runs a consolidation cycle (sleep analogy). Dashboard shows before/after weight snapshots.
4. Rate limiting is clearly visible: RPM/RPD counters, backoff indicators, and the system gracefully queueing requests during rate-limited periods.

**What it proves:** corteX exhibits genuine emergent learning behavior over extended interactions. The brain-inspired architecture produces measurable improvements in prediction accuracy, tool selection, and calibration quality over time.

#### Scenario L2: "Resilience and Recovery" (Rate Limits + Context Recovery + Adaptation)

**Duration:** 1-2 hours | **API calls:** ~150 (deliberately pushes rate limits) | **Components highlighted:** Rate Limit Handler, CorticalContextEngine (checkpointing + recovery), AdaptationFilter (habituation), AttentionalFilter, TargetedModulator

**Flow:**
1. System runs at an aggressive pace (~8 RPM) for the first 15 minutes, consuming a significant portion of the RPD budget.
2. At ~100 requests, rate limits trigger. The demo shows:
   - Exponential backoff with jitter kicking in.
   - UI badge turns yellow ("Rate Limited"), then red if sustained.
   - Queue management: pending requests visible in the UI.
   - Graceful degradation: cached responses served while waiting.
3. System recovers. Pace throttles to sustainable ~2 RPM.
4. Checkpoint recovery test: At the 1-hour mark, a simulated context corruption triggers rollback to the last checkpoint. Dashboard shows checkpoint loading and state restoration.
5. Adaptation habituation: After an hour of consistent behavior, the AdaptationFilter habituates to recurring signals. Novel signals still trigger strong responses. Dashboard shows habituated vs. active signals.
6. TargetedModulator: Operator manually activates/silences specific brain components (like optogenetic experiments). Shows how modulation affects behavior.

**What it proves:** corteX handles rate limits, context corruption, and operational overrides gracefully -- critical requirements for production enterprise deployment.

---

### 2.4 Scenario Summary Matrix

| Scenario | Duration | API Calls | Primary Components | Difficulty |
|----------|----------|-----------|-------------------|------------|
| Q1: Learning Your Style | 2-3 min | ~8 | Feedback, Adaptation, Plasticity, Weights | Easy |
| Q2: Tool Selection | 2-3 min | ~10 | Bayesian, Thompson, Population | Easy |
| Q3: Prediction Error | 3-4 min | ~12 | Prediction, Surprise, DualProcess, Calibration | Easy |
| M1: Research Project | 30 min | ~60 | GoalTracker, Context, Memory, CrossModal, Proactive | Medium |
| M2: Enterprise Safety | 20 min | ~40 | DualProcess, Minimax, Reputation, Nash, Shapley | Medium |
| L1: Deep Learning | 2-4 hr | ~200 | Plasticity, Calibration, Concepts, Reorganization, Simulator | Hard |
| L2: Resilience | 1-2 hr | ~150 | RateLimit, Context, Adaptation, Attention, Modulator | Hard |

All 20 brain components are covered across the 7 scenarios. Each scenario can run independently.

---

## 3. Architecture

### 3.1 High-Level Architecture

```
+------------------------------------------------------+
|                  Browser (React SPA)                  |
|  +------------------+  +---------------------------+ |
|  | Conversation     |  | Brain Dashboard           | |
|  | Panel            |  |  +-- Weight Inspector     | |
|  |                  |  |  +-- Concept Graph         | |
|  |                  |  |  +-- Territory Heat Map    | |
|  |                  |  |  +-- Calibration Health    | |
|  |                  |  |  +-- Timeline View         | |
|  +--------+---------+  +------------+--------------+ |
|           |                         |                 |
|           +-----------+-------------+                 |
|                       |                               |
|              WebSocket (real-time)                     |
|              + REST API (commands)                     |
+------------------------------------------------------+
                        |
+------------------------------------------------------+
|                  FastAPI Backend                       |
|  +--------------------------------------------------+|
|  | Demo Controller                                   ||
|  |  +-- ScenarioRunner   (orchestrates demo flows)   ||
|  |  +-- RateLimitManager (RPM/RPD tracking + queue)  ||
|  |  +-- BrainStateEmitter (WebSocket broadcaster)    ||
|  +--------------------------------------------------+|
|  +--------------------------------------------------+|
|  | corteX SDK (imported as a library)                ||
|  |  +-- Engine (all 20 brain components)             ||
|  |  +-- Runtime (Orchestrator)                       ||
|  |  +-- Core (LLM clients, contracts)               ||
|  |  +-- Tools (tool framework)                       ||
|  |  +-- Memory (pluggable backends)                  ||
|  +--------------------------------------------------+|
+------------------------------------------------------+
                        |
              Gemini API (external)
         25 RPM / 250 RPD (Tier-1)
```

### 3.2 Backend Architecture

**Framework:** FastAPI + uvicorn

**Module Layout:**

```python
# demo/server.py -- Main entry point

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="corteX Demo", version="1.0.0")

# REST endpoints
@app.post("/api/demo/start")       # Start a scenario
@app.post("/api/demo/step")        # Advance one step
@app.post("/api/demo/message")     # Send user message
@app.get("/api/demo/state")        # Get current brain state
@app.get("/api/demo/scenarios")    # List available scenarios
@app.post("/api/demo/modulate")    # TargetedModulator override
@app.get("/api/demo/rate-limit")   # Get rate limit status

# WebSocket endpoint
@app.websocket("/ws/brain")        # Real-time brain state updates
```

**Key Backend Components:**

```
demo/
  server.py              # FastAPI app + routes
  scenarios/
    __init__.py
    base.py              # BaseScenario abstract class
    q1_learning.py       # Quick: Learning Your Style
    q2_tool_select.py    # Quick: Tool Selection
    q3_prediction.py     # Quick: Prediction Error
    m1_research.py       # Medium: Multi-Tool Research
    m2_enterprise.py     # Medium: Enterprise Safety
    l1_deep_learning.py  # Long: Deep Learning Curves
    l2_resilience.py     # Long: Resilience and Recovery
  rate_limiter.py        # Sliding window RPM/RPD tracker
  brain_state.py         # Collects snapshots from all 20 components
  ws_broadcaster.py      # WebSocket connection manager
  simulated_tools.py     # Mock tools for demos (no external deps)
```

### 3.3 How the Demo Imports from the SDK

The demo imports corteX exactly as a customer would:

```python
from corteX import Engine, Agent, Session, WeightConfig, tool
from corteX.engine.weights import WeightEngine, WeightCategory
from corteX.engine.bayesian import BetaDistribution, BayesianToolSelector
from corteX.engine.game_theory import DualProcessRouter, EscalationContext
from corteX.engine.context import CorticalContextEngine, ContextConfig
from corteX.engine.prediction import PredictionEngine
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.plasticity import PlasticityManager
from corteX.engine.adaptation import AdaptationFilter
from corteX.engine.memory import MemoryFabric
from corteX.engine.population import PopulationDecoder
from corteX.engine.proactive import ProactivePredictionEngine
from corteX.engine.cross_modal import CrossModalAssociator, ContextEnricher
from corteX.engine.calibration import ContinuousCalibrationEngine
from corteX.engine.columns import FunctionalColumns  # P2
from corteX.engine.resource_map import ResourceHomunculus  # P2
from corteX.engine.attention import AttentionalFilter  # P2
from corteX.engine.concepts import ConceptGraph  # P3
from corteX.engine.reorganization import MapReorganizer  # P3
from corteX.engine.modulator import TargetedModulator  # P3
from corteX.engine.simulator import ComponentSimulator  # P3
```

### 3.4 WebSocket Protocol

The WebSocket at `/ws/brain` emits JSON messages at configurable intervals (default: 500ms during active steps, 5s during idle):

```json
{
  "type": "brain_state",
  "timestamp": 1739145600.0,
  "step": 42,
  "components": {
    "weight_engine": {
      "behavioral": {"verbosity": -0.25, "autonomy": 0.55, ...},
      "tool_preference": {"code_interpreter": {"preference_score": 0.82, ...}},
      "goal_alignment": {"current_progress": 0.45, "drift_score": 0.08}
    },
    "dual_process": {
      "current_route": "system1",
      "system2_ratio": 0.28,
      "escalation_reasons": []
    },
    "prediction": {
      "last_surprise": 0.12,
      "calibration_error": 0.08,
      "average_surprise": 0.15
    },
    "context": {
      "hot_items": 12,
      "warm_items": 45,
      "cold_items": 120,
      "utilization": 0.62
    },
    "memory": {
      "working": {"items": 23, "capacity": 100},
      "episodic": {"items": 8, "success_rate": 0.75},
      "semantic": {"items": 15}
    },
    "plasticity": {
      "multiplier": 1.45,
      "ltp_streaks": {"code_interpreter+coding": 5},
      "ltd_streaks": {}
    },
    "calibration": {
      "overall_health": "healthy",
      "overall_ece": 0.08,
      "alarming_domains": []
    },
    "cross_modal": {
      "total_links": 34,
      "total_nodes": 18,
      "avg_strength": 0.42
    },
    "proactive": {
      "predicted_next": "coding",
      "confidence": 0.72,
      "accuracy": 0.68,
      "pre_warming_hit_rate": 0.65
    },
    "population": {
      "last_value": 0.81,
      "agreement": 0.92,
      "voter_count": 5
    },
    "resource_homunculus": {
      "task_count": 5,
      "over_allocated": [],
      "under_allocated": ["testing"],
      "reorganization_count": 3
    },
    "goal_tracker": {
      "progress": 0.45,
      "drift": 0.08,
      "loops_detected": 0,
      "stall_turns": 0
    },
    "feedback": {
      "total_interactions": 42,
      "corrections": 2,
      "satisfaction_count": 8
    },
    "adaptation": {
      "habituated_signals": ["brevity"],
      "active_signals": ["frustration", "engagement"]
    },
    "reputation": {
      "trust_scores": {"code_interpreter": 0.92, "web_search": 0.78},
      "quarantined": []
    },
    "concept_graph": {
      "total_concepts": 24,
      "total_edges": 56,
      "grandmother_cells": ["authentication", "REST_API"]
    },
    "attention": {
      "change_detected": false,
      "priority_queue_size": 3,
      "suppressed_count": 12
    },
    "modulator": {
      "active_overrides": [],
      "silenced_components": []
    },
    "simulator": {
      "running_experiments": 0,
      "completed_experiments": 2,
      "best_alternative_improvement": 0.05
    }
  },
  "rate_limit": {
    "rpm_used": 8,
    "rpm_limit": 25,
    "rpd_used": 87,
    "rpd_limit": 250,
    "is_limited": false,
    "retry_after_seconds": 0,
    "queue_depth": 0
  }
}
```

---

## 4. UI Design

### 4.1 Design Philosophy

**Apple-style minimalism:** Every pixel earns its place. No decorative elements. Information density is high but visual clutter is zero. The interface should feel like a scientific instrument -- precise, calm, trustworthy.

### 4.2 Color Palette

| Token | Hex | Usage |
|-------|-----|-------|
| `--bg-primary` | `#FFFFFF` | Main background |
| `--bg-secondary` | `#F5F5F7` | Card backgrounds, sidebar |
| `--bg-tertiary` | `#E8E8ED` | Hover states, subtle borders |
| `--text-primary` | `#1D1D1F` | Primary text, headings |
| `--text-secondary` | `#86868B` | Secondary text, labels |
| `--text-tertiary` | `#AEAEB2` | Disabled text, timestamps |
| `--border` | `#D2D2D7` | Borders, dividers |
| `--status-green` | `#34C759` | Healthy, success, active |
| `--status-red` | `#FF3B30` | Error, alarm, quarantined |
| `--status-yellow` | `#FFCC00` | Warning, rate limited |
| `--status-blue` | `#007AFF` | Info, links, active selection |
| `--accent` | `#1D1D1F` | Primary buttons, active tabs |

**Dark mode variant** (optional, not in MVP):
- Swap `#FFFFFF` <-> `#1D1D1F`
- Swap `#F5F5F7` <-> `#2C2C2E`
- Status colors remain the same

### 4.3 Typography

| Element | Font | Weight | Size |
|---------|------|--------|------|
| Page title | Inter | 700 | 28px |
| Section heading | Inter | 600 | 20px |
| Component label | Inter | 600 | 14px |
| Body text | Inter | 400 | 14px |
| Metric value | Inter (tabular nums) | 500 | 16px |
| Metric label | Inter | 400 | 12px |
| Code / data | JetBrains Mono | 400 | 13px |
| Timestamp | JetBrains Mono | 400 | 11px |

Use `font-variant-numeric: tabular-nums` for all numerical displays to prevent layout jitter.

### 4.4 Layout: Three-Panel Design

```
+---------------------------------------------------------------+
| Header: corteX Demo    [Scenario: Q1 v]  [Step: 12]  [RPM .]  |
+---------------------------------------------------------------+
| Sidebar    |  Main Content Area                                |
| (Brain     |  +--------------------------------------------+  |
|  Component |  | Conversation Panel (60% height)            |  |
|  List)     |  |  User messages + Agent responses           |  |
|            |  |  Inline brain annotations                  |  |
| 20 items   |  +--------------------------------------------+  |
| scrollable |  | Brain Dashboard (40% height, scrollable)   |  |
|            |  |  Active component visualizations           |  |
|            |  |  Weight Inspector / Concept Graph / etc.   |  |
|            |  +--------------------------------------------+  |
+---------------------------------------------------------------+
| Status Bar: Rate Limit [|||||||...........] 8/25 RPM  87/250 RPD
+---------------------------------------------------------------+
```

**Sidebar (240px, fixed):** Lists all 20 brain components. Each item shows:
- Component name
- Status dot (green = active this step, grey = idle)
- Key metric (e.g., WeightEngine shows "autonomy: 0.55")

Clicking a component scrolls the Brain Dashboard to that component's detail view.

**Conversation Panel (top, 60%):** Standard chat interface with:
- User messages (right-aligned, dark bubble)
- Agent responses (left-aligned, light bubble)
- Inline brain annotations (small grey pills showing which components fired)
- Tool call indicators (monospace, collapsible)

**Brain Dashboard (bottom, 40%):** Tabbed or scrollable panel with component visualizations. Tabs: `Overview` | `Weights` | `Prediction` | `Memory` | `Context` | `Learning` | `Graph` | `Timeline`

### 4.5 Component Design Specifications

#### Weight Gauges
- Horizontal bar gauges, 200px wide, 8px height
- Color: gradient from `--status-red` (left, -1.0) through neutral `--bg-tertiary` (center, 0.0) to `--status-green` (right, 1.0)
- Marker: 3px wide black line at current value
- Label above, value to the right
- Animate with 300ms ease-out transitions

#### Beta Distribution Mini-Charts
- Sparkline-style, 120px x 40px
- Area fill in `--status-blue` at 10% opacity
- Line in `--status-blue`
- Show mean as vertical dashed line, 95% CI as shaded region
- Update in real-time with 200ms CSS transition

#### Concept Graph
- Force-directed layout using D3.js
- Nodes: 8-16px circles, size proportional to activation count
- Edges: 1-3px lines, opacity proportional to strength
- Active nodes pulse with a subtle `--status-green` glow
- Grandmother cells rendered with a double-ring border
- Interaction: hover shows tooltip with metadata, click pins/unpins

#### Territory Heat Map (Resource Homunculus)
- Grid of rectangles, one per task type
- Size proportional to token_budget allocation
- Color intensity proportional to frequency (darker = more used)
- Over-allocated tasks have a red border
- Under-allocated tasks have a dashed border
- Animate during reorganization (rectangles resize with 500ms spring animation)

#### Calibration Reliability Diagram
- 10-bin bar chart (one per calibration bin)
- X-axis: predicted probability (0.0 - 1.0)
- Y-axis: actual frequency
- Diagonal line shows perfect calibration
- Bars colored green (within tolerance) or red (calibration error > threshold)
- ECE score displayed above the chart

#### Timeline View
- Horizontal scrollable timeline
- Each step is a vertical tick mark
- Above: tool used (icon), model used (icon)
- Below: surprise magnitude (bar height), outcome (green/red dot)
- Current step highlighted with `--status-blue`
- Clickable: clicking a step shows that step's full brain state snapshot

#### Rate Limit Status Bar
- Full-width bar at the bottom of the screen
- Two progress bars side by side:
  - RPM: filling left-to-right, resets every 60s
  - RPD: filling left-to-right, persistent for the day
- Color changes: green (< 60%), yellow (60-80%), red (> 80%)
- When rate limited: pulsing red background, countdown timer "Retry in 12s"
- Queue depth indicator: "3 requests queued"

---

## 5. Brain Dashboard Layout

### 5.1 Overview Tab

The Overview tab shows a single-screen summary of all 20 components in a grid layout:

```
+------------------+------------------+------------------+------------------+
| WeightEngine     | DualProcess      | Prediction       | GoalTracker      |
| verbosity: -0.25 | Route: Sys1      | Surprise: 0.12   | Progress: 45%    |
| autonomy:  0.55  | Sys2 ratio: 28%  | Calibration: 0.08| Drift: 0.08      |
| [gauge bars]     | [S1/S2 split pie]| [surprise line]  | [progress ring]  |
+------------------+------------------+------------------+------------------+
| Feedback         | Plasticity       | Adaptation       | MemoryFabric     |
| Corrections: 2   | Multiplier: 1.45 | Habituated: 1    | Working: 23/100  |
| Satisfaction: 8   | LTP streaks: 5   | Active: 3        | Episodic: 8      |
| [signal bars]    | [learning curve] | [signal list]    | [3-tier stack]   |
+------------------+------------------+------------------+------------------+
| Population       | Proactive        | CrossModal       | Calibration      |
| Agreement: 92%   | Accuracy: 68%    | Links: 34        | Health: HEALTHY  |
| Voters: 5        | Next: coding     | Avg Str: 0.42    | ECE: 0.08        |
| [voter dots]     | [chain diagram]  | [mini graph]     | [reliability]    |
+------------------+------------------+------------------+------------------+
| Columns          | ResourceMap      | Attention        | ConceptGraph     |
| Active col: 3    | Reorgs: 3        | Changes: 2       | Concepts: 24     |
| Competition: 0.4 | Over-alloc: 0    | Suppressed: 12   | Edges: 56        |
| [column bars]    | [heat map]       | [priority queue]  | [mini graph]     |
+------------------+------------------+------------------+------------------+
| Reputation       | Modulator        | Simulator        | Context          |
| Quarantined: 0   | Overrides: 0     | Experiments: 2   | Hot: 12 items    |
| Avg Trust: 0.85  | Silenced: 0      | Improvement: +5% | Utilization: 62% |
| [trust bars]     | [switch toggles] | [A/B bars]       | [tier bars]      |
+------------------+------------------+------------------+------------------+
```

Each card is 250px x 140px. The grid is 4 columns on desktop, 2 on tablet, 1 on mobile. Clicking any card opens its detailed view.

### 5.2 Detailed Component Views

Each component's detail view includes:
1. **Header:** Component name, status badge (active/idle/alarm), last update timestamp
2. **Primary visualization:** The main chart or visualization (see Section 4.5)
3. **Metrics table:** All key metrics in a two-column layout (label: value)
4. **Event log:** Scrollable list of recent events from this component (last 20)
5. **Raw state toggle:** Expandable JSON view of the component's `to_dict()` / `get_stats()` output

### 5.3 Real-Time Updates

- All visualizations update in real-time via WebSocket
- CSS transitions smooth out numerical changes (300ms ease-out)
- New events appear with a subtle slide-in animation
- Changed values flash briefly (100ms highlight with `--status-blue` at 10% opacity)
- No full page reloads -- all updates are incremental DOM patches via React state

---

## 6. Tech Stack

### 6.1 Recommended Stack (Primary)

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend** | FastAPI 0.100+ | Already in corteX dependencies. Async-native. WebSocket support built-in. |
| **ASGI Server** | uvicorn | Already in corteX dependencies. |
| **Frontend** | React 18 + TypeScript | Existing `ui-kit/` uses React. Component model fits dashboard. |
| **CSS** | Tailwind CSS 3 | Existing `ui-kit/` uses Tailwind. Utility-first for rapid prototyping. |
| **Charts** | Recharts (simple) + D3.js (concept graph) | Recharts for standard charts (line, bar, gauge). D3 for force-directed graph only. |
| **WebSocket Client** | Native WebSocket API | No library needed. React hooks for connection management. |
| **Build** | Vite | Existing `ui-kit/` uses Vite. Fast HMR. |
| **Testing (Backend)** | pytest + pytest-asyncio | Already configured in `pyproject.toml`. |
| **Testing (Frontend)** | Vitest + React Testing Library | Vitest integrates with Vite config. |
| **E2E Testing** | Playwright | Already in corteX dependencies. |

### 6.2 Alternative Stack (Simpler, for quick prototype)

If rapid prototyping is preferred over polish:

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend** | Same FastAPI | Same |
| **Frontend** | Streamlit | Single Python file. Built-in charts. WebSocket via `st.experimental_rerun`. |
| **Charts** | Streamlit built-in + Plotly | Streamlit's `st.metric`, `st.plotly_chart`, `st.json` |

**Streamlit tradeoffs:**
- Pro: 10x faster to build, single language (Python), no npm
- Con: Less visual control, no Apple-style aesthetics, harder to do real-time WebSocket updates, no custom force-directed graphs

**Recommendation:** Start with React for the primary demo (matches existing ui-kit), but consider a Streamlit version as a "developer console" for internal use.

### 6.3 Dependencies (Backend Demo)

```toml
# Added to pyproject.toml under [project.optional-dependencies]
demo = [
    "fastapi>=0.100",
    "uvicorn[standard]>=0.20",
    "websockets>=11.0",
    "httpx>=0.25",       # For rate limit testing
    "rich>=13.0",        # Terminal output formatting for CLI mode
]
```

### 6.4 Dependencies (Frontend Demo)

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "recharts": "^2.10.0",
    "d3": "^7.8.0",
    "d3-force": "^3.0.0",
    "lucide-react": "^0.263.0",
    "clsx": "^2.0.0"
  },
  "devDependencies": {
    "@types/d3": "^7.4.0",
    "tailwindcss": "^3.3.0",
    "vite": "^4.4.0",
    "typescript": "^5.0.0",
    "@vitejs/plugin-react": "^4.0.0"
  }
}
```

---

## 7. Rate Limit Strategy

### 7.1 Gemini Tier-1 Limits

| Metric | Limit | Window |
|--------|-------|--------|
| Requests Per Minute (RPM) | 25 | Sliding 60s window |
| Requests Per Day (RPD) | 250 | Calendar day (UTC) |

### 7.2 Tracking Implementation

```python
# demo/rate_limiter.py

import time
import asyncio
import random
from collections import deque
from dataclasses import dataclass, field
from typing import Optional, Deque

@dataclass
class RateLimitState:
    """Current rate limit state, broadcast to UI."""
    rpm_used: int = 0
    rpm_limit: int = 25
    rpd_used: int = 0
    rpd_limit: int = 250
    is_limited: bool = False
    retry_after_seconds: float = 0.0
    queue_depth: int = 0
    last_request_time: float = 0.0
    total_retries: int = 0
    total_cached_responses: int = 0

class SlidingWindowRateLimiter:
    """
    Sliding window counter for RPM tracking.
    Uses a deque of timestamps, evicting entries older than window_seconds.
    """

    def __init__(self, max_requests: int, window_seconds: float):
        self._max = max_requests
        self._window = window_seconds
        self._timestamps: Deque[float] = deque()

    def _evict(self, now: float) -> None:
        cutoff = now - self._window
        while self._timestamps and self._timestamps[0] < cutoff:
            self._timestamps.popleft()

    def can_proceed(self) -> bool:
        self._evict(time.time())
        return len(self._timestamps) < self._max

    def record(self) -> None:
        self._timestamps.append(time.time())

    @property
    def used(self) -> int:
        self._evict(time.time())
        return len(self._timestamps)

    @property
    def remaining(self) -> int:
        return max(0, self._max - self.used)

    @property
    def time_until_available(self) -> float:
        """Seconds until the next request slot opens."""
        if self.can_proceed():
            return 0.0
        self._evict(time.time())
        if self._timestamps:
            return max(0.0, self._timestamps[0] + self._window - time.time())
        return 0.0
```

### 7.3 Request Queue

When approaching rate limits (> 80% capacity), new requests enter a FIFO queue instead of being rejected:

```python
class RequestQueue:
    """FIFO queue for rate-limited requests."""

    def __init__(self, max_queue_size: int = 20):
        self._queue: asyncio.Queue = asyncio.Queue(maxsize=max_queue_size)
        self._processing = False

    async def enqueue(self, request_fn, *args, **kwargs):
        """Add a request to the queue. Returns a Future with the result."""
        future = asyncio.get_event_loop().create_future()
        await self._queue.put((request_fn, args, kwargs, future))
        if not self._processing:
            asyncio.create_task(self._process_queue())
        return await future

    async def _process_queue(self):
        """Process queued requests respecting rate limits."""
        self._processing = True
        while not self._queue.empty():
            request_fn, args, kwargs, future = await self._queue.get()
            # Wait for rate limit to clear
            while not rate_limiter.can_proceed():
                await asyncio.sleep(0.5)
            try:
                result = await request_fn(*args, **kwargs)
                future.set_result(result)
            except Exception as e:
                future.set_exception(e)
        self._processing = False
```

### 7.4 Exponential Backoff with Jitter

When a 429 response is received from the Gemini API:

```python
async def request_with_backoff(request_fn, max_retries: int = 5):
    """Execute request with exponential backoff + jitter on 429."""
    for attempt in range(max_retries):
        try:
            result = await request_fn()
            return result
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            # Exponential backoff: 1s, 2s, 4s, 8s, 16s
            base_delay = 2 ** attempt
            # Full jitter: uniform(0, base_delay)
            jitter = random.uniform(0, base_delay)
            delay = base_delay + jitter
            # Broadcast to UI
            await broadcast_rate_limit_event(delay, attempt + 1)
            await asyncio.sleep(delay)
```

### 7.5 Graceful Degradation

When rate limited, the demo does not crash or stall. Instead:

1. **Cached responses:** For demo scenarios, pre-computed responses for common steps are served from a local cache. The UI shows a "(cached)" badge.
2. **Simulated mode:** Components that do not need LLM calls (WeightEngine, PlasticityManager, CalibrationEngine, etc.) continue to run and update the dashboard in real-time using simulated data.
3. **Queue visualization:** The UI shows a "queue depth" indicator and estimated wait time.
4. **RPD budget pacing:** For long demos, the system automatically calculates the sustainable RPM rate (e.g., if 150 RPD remaining and 2 hours left, pace at ~1.25 RPM) and throttles accordingly.

### 7.6 UI Rate Limit Indicators

| State | RPM Bar Color | Badge | Behavior |
|-------|--------------|-------|----------|
| Normal (< 60%) | Green | None | Requests proceed immediately |
| Approaching (60-80%) | Yellow | "Rate Awareness" | Queue starts filling proactively |
| Limited (> 80%) | Orange | "Approaching Limit" | New requests auto-queued |
| Rate Limited (429) | Red (pulsing) | "Rate Limited -- Retry in Xs" | Backoff active, countdown visible |
| Daily Exhausted | Red (solid) | "Daily Limit Reached" | Switch to simulated mode |

---

## 8. File Structure

### 8.1 Proposed Directory Layout

```
corteX/
  corteX/                  # SDK source (existing)
    engine/                # 20 brain components
    core/                  # LLM clients, contracts
    runtime/               # Orchestrator
    memory/                # Memory drivers
    plugins/               # Subsystems
    tools/                 # Tool framework
    server/                # Existing FastAPI server
    sdk.py                 # SDK entry point

  demo/                    # NEW: Demo application
    __init__.py
    server.py              # FastAPI app (imports from corteX.*)
    config.py              # Demo-specific config (colors, limits, etc.)

    scenarios/             # Demo scenario implementations
      __init__.py
      base.py              # BaseScenario abstract class
      q1_learning.py       # Quick: Learning Your Style
      q2_tool_select.py    # Quick: Tool Selection
      q3_prediction.py     # Quick: Prediction Error
      m1_research.py       # Medium: Multi-Tool Research
      m2_enterprise.py     # Medium: Enterprise Safety
      l1_deep_learning.py  # Long: Deep Learning Curves
      l2_resilience.py     # Long: Resilience and Recovery

    engine/                # Demo-specific engine wrappers
      rate_limiter.py      # RPM/RPD sliding window tracker
      brain_state.py       # Snapshot collector for all 20 components
      ws_broadcaster.py    # WebSocket connection manager + broadcasting
      simulated_tools.py   # Mock tools for offline/demo use
      response_cache.py    # Cached responses for rate-limited operation

    frontend/              # React frontend (separate npm project)
      package.json
      tsconfig.json
      vite.config.ts
      tailwind.config.js
      postcss.config.js
      index.html
      public/
        favicon.svg
      src/
        main.tsx
        App.tsx
        index.css            # Tailwind + custom Apple-style tokens
        types.ts             # BrainState TypeScript interfaces

        hooks/
          useBrainSocket.ts  # WebSocket hook for real-time updates
          useScenario.ts     # Scenario control hook
          useRateLimit.ts    # Rate limit state hook

        components/
          layout/
            Header.tsx
            Sidebar.tsx
            StatusBar.tsx
            ThreePanel.tsx

          conversation/
            ConversationPanel.tsx
            MessageBubble.tsx
            ToolCallIndicator.tsx
            BrainAnnotation.tsx

          dashboard/
            BrainDashboard.tsx
            Overview.tsx          # 5x4 grid of all 20 components
            WeightInspector.tsx   # Weight gauges + Bayesian distributions
            ConceptGraphView.tsx  # D3 force-directed graph
            TerritoryHeatMap.tsx  # Resource Homunculus visualization
            CalibrationChart.tsx  # ECE reliability diagram
            TimelineView.tsx     # Horizontal step timeline
            PredictionChart.tsx  # Prediction vs actual sparklines
            MemoryFabricView.tsx # 3-tier memory visualization
            PlasticityView.tsx   # Learning curves + LTP/LTD
            DualProcessView.tsx  # System 1/2 routing indicator
            PopulationView.tsx   # Population coding voter dots
            AdaptationView.tsx   # Habituation signal list
            CrossModalView.tsx   # Association mini-graph
            GoalTrackerView.tsx  # Progress ring + drift gauge
            FeedbackView.tsx     # Signal type bars
            ContextView.tsx      # Hot/warm/cold tier bars
            ReputationView.tsx   # Trust score bars
            AttentionView.tsx    # Priority queue
            ModulatorView.tsx    # Toggle switches
            SimulatorView.tsx    # A/B test results bars

          shared/
            WeightGauge.tsx      # Reusable horizontal gauge
            MetricCard.tsx       # Label + value + optional sparkline
            StatusDot.tsx        # Green/yellow/red dot
            BetaDistChart.tsx    # Beta distribution sparkline
            JsonViewer.tsx       # Expandable raw JSON view
            RateLimitBar.tsx     # RPM/RPD progress bar

    tests/                 # Demo-specific tests
      __init__.py
      test_rate_limiter.py
      test_brain_state.py
      test_scenarios.py
      test_ws_broadcaster.py

  tests/                   # SDK tests (existing)
  docs/                    # Documentation (existing)
  ui-kit/                  # Existing UI component library
```

### 8.2 Import Patterns

The demo never imports private internals. All imports follow the public SDK API:

```python
# CORRECT: Demo imports from public SDK
from corteX.engine.weights import WeightEngine
from corteX.engine.calibration import ContinuousCalibrationEngine

# WRONG: Never do this
from corteX.engine.weights import _clamp  # private function
```

### 8.3 Configuration Separation

Demo configuration is isolated from SDK configuration:

```python
# demo/config.py
from dataclasses import dataclass

@dataclass
class DemoConfig:
    # Rate limits
    rpm_limit: int = 25
    rpd_limit: int = 250

    # UI WebSocket
    ws_update_interval_active_ms: int = 500
    ws_update_interval_idle_ms: int = 5000

    # Colors (for server-side rendering if needed)
    color_bg_primary: str = "#FFFFFF"
    color_bg_secondary: str = "#F5F5F7"
    color_text_primary: str = "#1D1D1F"
    color_text_secondary: str = "#86868B"
    color_status_green: str = "#34C759"
    color_status_red: str = "#FF3B30"
    color_status_yellow: str = "#FFCC00"
    color_status_blue: str = "#007AFF"

    # Demo behavior
    default_scenario: str = "q1_learning"
    max_queue_size: int = 20
    cache_responses: bool = True
    simulated_mode: bool = False  # Run without Gemini API
```

---

## 9. Implementation Phases

### Phase 1: Foundation (Week 1-2)

**Goal:** Backend running, one quick scenario working, basic UI shell.

- [ ] Create `demo/` directory structure
- [ ] Implement `rate_limiter.py` with sliding window RPM/RPD tracking
- [ ] Implement `brain_state.py` -- snapshot collector for all 20 components
- [ ] Implement `ws_broadcaster.py` -- WebSocket connection manager
- [ ] Implement `server.py` -- FastAPI app with REST + WebSocket endpoints
- [ ] Implement `scenarios/base.py` -- BaseScenario abstract class
- [ ] Implement `scenarios/q1_learning.py` -- First demo scenario
- [ ] Scaffold React frontend with layout components (Header, Sidebar, StatusBar)
- [ ] Implement `useBrainSocket.ts` -- WebSocket hook
- [ ] Build basic Overview grid (20 cards with labels and status dots)
- [ ] Build RateLimitBar component

### Phase 2: Quick Demos (Week 3-4)

**Goal:** All 3 quick scenarios running with polished visualizations.

- [ ] Implement `scenarios/q2_tool_select.py` and `scenarios/q3_prediction.py`
- [ ] Build WeightGauge, BetaDistChart, MetricCard shared components
- [ ] Build WeightInspector, PredictionChart, DualProcessView
- [ ] Build ConversationPanel with MessageBubble and BrainAnnotation
- [ ] Build FeedbackView, PlasticityView, PopulationView
- [ ] Implement `simulated_tools.py` for offline demo mode
- [ ] Write tests for rate limiter and brain state collector
- [ ] Polish animations (300ms ease-out transitions)

### Phase 3: Medium Demos (Week 5-6)

**Goal:** Both medium scenarios running. Full dashboard operational.

- [ ] Implement `scenarios/m1_research.py` and `scenarios/m2_enterprise.py`
- [ ] Build GoalTrackerView, ContextView, MemoryFabricView
- [ ] Build CrossModalView, CalibrationChart, ReputationView
- [ ] Build TerritoryHeatMap (Resource Homunculus)
- [ ] Build TimelineView (horizontal scrollable timeline)
- [ ] Implement `response_cache.py` for graceful degradation
- [ ] Build request queue visualization in StatusBar
- [ ] Write E2E tests with Playwright

### Phase 4: Long Demos + Polish (Week 7-8)

**Goal:** Long scenarios running. All 20 component visualizations complete. Production-ready.

- [ ] Implement `scenarios/l1_deep_learning.py` and `scenarios/l2_resilience.py`
- [ ] Build ConceptGraphView (D3 force-directed layout)
- [ ] Build AttentionView, ModulatorView, SimulatorView
- [ ] Build AdaptationView
- [ ] Implement RPD budget pacing for long demos
- [ ] Implement checkpoint recovery demonstration
- [ ] Performance optimization (virtualized lists, memoized components)
- [ ] Mobile responsive adjustments
- [ ] Final visual polish and animation tuning
- [ ] Documentation: README for running the demo

---

## Appendix A: Brain Component Quick Reference

| # | Component | Source File | Key Method for State | Visualization |
|---|-----------|------------|---------------------|---------------|
| 1 | WeightEngine | `engine/weights.py` | `snapshot()` | Weight gauges |
| 2 | DualProcessRouter | `engine/game_theory.py` | `get_stats()`, `to_dict()` | S1/S2 split indicator |
| 3 | CorticalContextEngine | `engine/context.py` | `get_stats()`, `get_token_budget_status()` | Tier bars (hot/warm/cold) |
| 4 | ProactivePredictionEngine | `engine/proactive.py` | `get_stats()` | Chain diagram + accuracy |
| 5 | CrossModalAssociator | `engine/cross_modal.py` | `get_stats()`, `to_dict()` | Mini association graph |
| 6 | ContinuousCalibrationEngine | `engine/calibration.py` | `report()`, `get_stats()` | ECE reliability diagram |
| 7 | FunctionalColumns | `engine/columns.py` | `get_stats()` | Column competition bars |
| 8 | ResourceHomunculus | `engine/resource_map.py` | `get_stats()`, `get_cortical_map()` | Territory heat map |
| 9 | AttentionalFilter | `engine/attention.py` | `get_stats()` | Priority queue |
| 10 | ConceptGraph | `engine/concepts.py` | `get_stats()`, `to_dict()` | Force-directed graph |
| 11 | MapReorganizer | `engine/reorganization.py` | `get_stats()` | Before/after territory diff |
| 12 | TargetedModulator | `engine/modulator.py` | `get_stats()` | Toggle switches |
| 13 | ComponentSimulator | `engine/simulator.py` | `get_stats()` | A/B comparison bars |
| 14 | MemoryFabric | `engine/memory.py` | `get_stats()` | 3-tier stack |
| 15 | PlasticityManager | `engine/plasticity.py` | `get_stats()` | Learning curves |
| 16 | PopulationDecoder | `engine/population.py` | (decode result) | Voter dots + agreement |
| 17 | FeedbackEngine | `engine/feedback.py` | `get_signal_summary()` | Signal type bars |
| 18 | PredictionEngine | `engine/prediction.py` | `get_stats()` | Surprise sparkline |
| 19 | AdaptationFilter | `engine/adaptation.py` | `get_stats()` | Habituated/active signals |
| 20 | GoalTracker | `engine/goal_tracker.py` | `get_summary()` | Progress ring + drift |

Additional components used by the brain components (not independently visualized but their data flows through the above):
- BayesianToolSelector (`engine/bayesian.py`) -- data shown via WeightEngine
- ReputationSystem (`engine/game_theory.py`) -- shown via dedicated ReputationView
- MinimaxSafetyGuard (`engine/game_theory.py`) -- shown in DualProcess detail view
- NashRoutingOptimizer (`engine/game_theory.py`) -- shown in DualProcess detail view
- ShapleyAttributor (`engine/game_theory.py`) -- shown in Timeline step detail

---

## Appendix B: Tailwind Config Extensions

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        'cortex': {
          'bg': '#F5F5F7',
          'text': '#1D1D1F',
          'text-secondary': '#86868B',
          'text-tertiary': '#AEAEB2',
          'border': '#D2D2D7',
          'green': '#34C759',
          'red': '#FF3B30',
          'yellow': '#FFCC00',
          'blue': '#007AFF',
        }
      },
      fontFamily: {
        'sans': ['Inter', '-apple-system', 'BlinkMacSystemFont', 'sans-serif'],
        'mono': ['JetBrains Mono', 'SF Mono', 'Menlo', 'monospace'],
      },
      animation: {
        'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
        'slide-in': 'slideIn 200ms ease-out',
        'value-flash': 'flash 100ms ease-out',
      }
    }
  }
}
```

---

## Appendix C: Running the Demo

```bash
# 1. Install SDK + demo dependencies
pip install -e ".[demo]"

# 2. Set API key
export GEMINI_API_KEY="your-key-here"

# 3. Start backend
uvicorn demo.server:app --reload --port 8000

# 4. In another terminal, start frontend
cd demo/frontend
npm install
npm run dev

# 5. Open browser
#    http://localhost:5173  (Vite dev server proxies API to :8000)

# 6. Or run in simulated mode (no API key needed)
DEMO_SIMULATED=1 uvicorn demo.server:app --reload --port 8000
```

---

## 10. Platform Selection: Complex Customer Service Platform

### 10.1 Decision (February 2026)

The demo SaaS application will be a **complex customer service platform** with full CRM, ticketing, knowledge base, analytics, and multi-channel support. This replaces the original standalone demo concept with a realistic enterprise application that showcases corteX agents in production-like conditions.

### 10.2 Three-Layer Architecture (Confirmed)

| Layer | Purpose | Audience |
|-------|---------|----------|
| **Layer 1: SaaS App** | Complex CS platform (CRM + helpdesk + KB + analytics) | End users / customer support agents |
| **Layer 2: Developer Dashboard** | corteX DevTools (brain inspector, weight tuning, observability) | SaaS developers integrating corteX |
| **Layer 3: Internal Dashboard** | Sales tool showing all 20 brain components in real-time | Questo sales team / demos |

### 10.3 Platform Research Results

Three platforms evaluated as base templates:

#### Top Choice: erxes (Experience Operating System)

- **GitHub**: https://github.com/erxes/erxes (~3,500 stars)
- **Stack**: TypeScript/Node.js (GraphQL Federation), React 18, MongoDB, Redis, BullMQ
- **License**: GPL-3.0 + Commons Clause (OK for demo, not redistribution)
- **Features**: Team inbox, CRM, sales pipeline, automation workflows, knowledge base, lead gen, chatbot, analytics, plugin architecture
- **Integration**: Build a corteX AI Agent Plugin via XOS architecture. BullMQ provides long-running task infrastructure. React frontend matches ui-kit.
- **Long-running scenarios**: Continuous lead qualification, multi-step deal automation, bulk segmentation, campaign orchestration, KB auto-generation, periodic analytics

#### Alternative: Chatwoot + Twenty CRM

- **Chatwoot**: https://github.com/chatwoot/chatwoot (~6,300 stars, MIT license, Ruby on Rails + Vue.js)
  - Purpose-built Agent Bot API for AI integration
  - 10+ communication channels (chat, email, WhatsApp, Telegram, etc.)
  - Best licensing (MIT)
- **Twenty CRM**: https://github.com/twentyhq/twenty (~39,000 stars, React frontend)
  - Modern CRM with contacts, companies, pipelines
  - React frontend matches corteX ui-kit
- Integration via API/webhooks between the two

#### Fallback: Odoo Community Edition

- **GitHub**: https://github.com/odoo/odoo (~48,900 stars, LGPL-3.0)
- **Stack**: **Python** backend (only Python option), PostgreSQL, custom OWL frontend
- **Features**: 30+ integrated modules (CRM, helpdesk, sales, accounting, inventory, HR, manufacturing)
- **Integration**: Direct Python import of corteX SDK -- no API overhead
- **Tradeoff**: Extremely complex, steep learning curve, non-React frontend

### 10.4 Recommendation

**erxes** is the strongest single-platform choice for demonstrating corteX in a complex, realistic CS environment. Its plugin architecture, React frontend, and BullMQ job queue provide natural integration points.

For maximum licensing flexibility: **Chatwoot + Twenty CRM** combination (both MIT-compatible).

### 10.5 Long-Running Bot Scenarios (Hours-Long Operation)

The platform must support scenarios where the corteX bot works continuously for many hours:

1. **Bulk ticket triage**: Process hundreds of incoming support tickets, classify by urgency/topic, auto-respond to simple ones, escalate complex ones
2. **Campaign orchestration**: Manage multi-step outreach to customer segments across channels over hours/days
3. **Knowledge base generation**: Analyze resolved conversations and auto-generate FAQ articles
4. **SLA monitoring**: Continuously monitor ticket SLAs, predict breaches, auto-escalate
5. **Customer sentiment analysis**: Process conversation history, detect trends, generate reports
6. **Data enrichment**: Enrich customer profiles by cross-referencing CRM data with support history
7. **Predictive escalation**: Use prediction engine to identify tickets likely to require human intervention
8. **Automated follow-ups**: Schedule and execute follow-up sequences based on ticket resolution

These scenarios demonstrate all 20 brain components working together in a sustained, production-like environment.

### 10.6 Hosting

- **Repository**: Private GitHub repo at https://github.com/QuestoM/
- **Documentation**: docs.cortex-ai.com (GitHub Pages, domain may change)
- **Deployment**: Docker Compose for local development, cloud-ready for production demos
