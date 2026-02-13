# Advanced Subsystems

Beyond the core Brain Engine and Context/Memory systems, corteX includes fourteen advanced subsystems that implement deeper neuroscience and decision-theory patterns. These subsystems operate at higher levels of abstraction, building on the foundational weight engine, memory fabric, and context engine.

## Subsystem Map

| Subsystem | Brain Pattern | Priority | What It Does |
|-----------|--------------|----------|--------------|
| [Attentional Filter](attention.md) | Thalamic gating | P2 | Routes processing depth based on novelty and change |
| [Functional Columns](columns.md) | Cortical columns | P2 | Task-specialized processing units with competition |
| [Resource Homunculus](resource-map.md) | Somatosensory cortex | P2 | Dynamic resource allocation based on usage |
| [Concept Graphs](concept-graphs.md) | Distributed representations | P3 | Semantic concept network with spreading activation |
| [Continuous Calibration](calibration.md) | Metacognition | P1 | Confidence tracking with Platt scaling |
| [Targeted Modulation](modulation.md) | Optogenetics | P3 | Precision control over specific tools/behaviors |
| [Map Reorganization](reorganization.md) | Cortical plasticity | P3 | Territory reallocation based on usage patterns |
| [Population Coding](population.md) | Motor cortex ensemble | P0 | Multi-evaluator consensus decisions |
| [Component Simulator](simulation.md) | Mental simulation | P3 | What-if analysis and A/B testing |
| [Structured Output](structured-output.md) | Self-monitoring | P1 | LLM self-assessment signal extraction (zero extra calls) |
| [Content-Aware Predictions](content-prediction.md) | Prefrontal rehearsal | P1 | LLM-powered tool/response/sentiment prediction |
| [Game Theory Integration](game-integration.md) | Nash/Shapley | P2 | Equilibrium routing and fair credit attribution |
| [Context Summarization](context-summarizer.md) | Memory consolidation | P1 | L2/L3 progressive compression for long sessions |
| [Semantic Scoring](semantic-scorer.md) | Semantic memory | P1 | TF-IDF vector relevance and novelty scoring |

## Priority Levels

The subsystems are organized by implementation priority:

- **P0 (Foundation)**: Required for basic operation. Population Coding provides the ensemble decision infrastructure.
- **P1 (Core)**: Required for production quality. Calibration ensures confidence estimates are accurate.
- **P2 (Enhancement)**: Significant quality improvements. Attention and columns optimize resource allocation.
- **P3 (Advanced)**: Sophisticated capabilities. Concepts, modulation, reorganization, and simulation provide deep adaptive behavior.

## How They Integrate

```
User Message
     |
     v
[Attentional Filter] -- classifies processing priority
     |
     v
[Functional Columns] -- routes to specialized processing unit
     |
     v
[Population Coding] -- ensemble evaluates tool/model candidates
     |
     v
[Targeted Modulator] -- applies enterprise/user overrides
     |
     v
[Concept Graphs] -- enriches context with semantic associations
     |
     v
[LLM Call] -- executed with calibrated confidence
     |
     v
[Calibration Engine] -- tracks prediction accuracy
     |
     v
[Resource Homunculus] -- updates allocation for next turn
     |
     v
[Map Reorganizer] -- periodic territory rebalancing
```
