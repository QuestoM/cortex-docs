# API Reference

Complete auto-generated API documentation for the corteX SDK, built directly from source code docstrings.

## Public API

The main entry points for building agents with corteX:

| Class | Description |
|-------|-------------|
| [`Engine`](engine.md) | Root engine that manages LLM providers and global configuration. |
| [`Agent`](agent.md) | Configured agent template. Stateless -- creates sessions. |
| [`Session`](session.md) | Live conversation session with full brain integration. |
| [`Response`](response.md) | Result object returned from agent runs. |

## Engine Modules

The brain-inspired subsystems that power every session:

| Module | Description |
|--------|-------------|
| [`weights`](engine/weights.md) | Synaptic weight engine with Hebbian learning and Bayesian priors. |
| [`goal_tracker`](engine/goal-tracker.md) | Goal tracking with loop detection. |
| [`feedback`](engine/feedback.md) | 4-tier implicit feedback engine. |
| [`prediction`](engine/prediction.md) | Prediction and surprise engine (reactive). |
| [`plasticity`](engine/plasticity.md) | Neural plasticity rules (LTP, LTD, homeostatic). |
| [`adaptation`](engine/adaptation.md) | Sensory adaptation and habituation. |
| [`game_theory`](engine/game-theory.md) | Dual-process routing, reputation, minimax, Nash, Shapley. |
| [`context`](engine/context.md) | Cortical Context Engine for long-running workflows. |
| [`memory`](engine/memory.md) | Memory Fabric with working, episodic, and semantic stores. |
| [`bayesian`](engine/bayesian.md) | Bayesian conjugate priors and prospect theory math. |
| [`calibration`](engine/calibration.md) | Continuous calibration and metacognition monitoring. |
| [`columns`](engine/columns.md) | Cortical column architecture for task specialization. |
| [`concepts`](engine/concepts.md) | Distributed concept graph with spreading activation. |
| [`cross_modal`](engine/cross-modal.md) | Cross-modal association and Hebbian binding. |
| [`attention`](engine/attention.md) | Attentional filter with subconscious processing. |
| [`resource_map`](engine/resource-map.md) | Resource Homunculus for non-uniform allocation. |
| [`modulator`](engine/modulator.md) | Optogenetics-inspired targeted activation and silencing. |
| [`proactive`](engine/proactive.md) | Proactive prediction engine (predicts user's next action). |
| [`reorganization`](engine/reorganization.md) | Cortical map reorganization and territory management. |
| [`population`](engine/population.md) | Population coding and ensemble quality estimation. |
| [`simulator`](engine/simulator.md) | Digital twin component simulator for what-if analysis. |

## Enterprise

Administrative control plane for on-prem and multi-tenant deployments:

| Module | Description |
|--------|-------------|
| [`config`](enterprise/config.md) | Per-tenant settings, safety policies, and compliance. |
| [`licensing`](enterprise/licensing.md) | Per-seat license management with offline support. |
| [`updates`](enterprise/updates.md) | SDK update delivery for connected and air-gapped environments. |

## Tools

Developer tool framework for extending agent capabilities:

| Module | Description |
|--------|-------------|
| [`decorator`](tools/decorator.md) | Decorator-based tool registration (`@cortex.tool`). |
| [`executor`](tools/executor.md) | Safe tool execution with timeout and error handling. |

## LLM Layer

Multi-provider LLM routing and adapter layer:

| Module | Description |
|--------|-------------|
| [`router`](llm/router.md) | Multi-model LLM router (the system's thalamus). |
| [`base`](llm/base.md) | Abstract LLM provider interface. |
| [`openai_client`](llm/openai-client.md) | OpenAI-compatible provider (OpenAI, Azure, vLLM, Ollama). |
| [`gemini_adapter`](llm/gemini-client.md) | Google Gemini provider adapter. |
