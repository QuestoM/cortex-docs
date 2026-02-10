# Concepts

How corteX works under the hood.

---

## Architecture at a glance

```text
                        +-------------------+
                        |      Engine       |
                        | (provider config) |
                        +---------+---------+
                                  |
                          create_agent()
                                  |
                        +---------+---------+
                        |      Agent        |
                        | (name, prompt,    |
                        |  tools, config)   |
                        +---------+---------+
                                  |
                         start_session()
                                  |
                   +--------------+--------------+
                   |          Session             |
                   |   20 brain components        |
                   |   conversation state         |
                   |   14-step run() pipeline     |
                   +------------------------------+
```

**Engine** registers providers and creates agents. **Agent** is a stateless template (personality, tools, config). **Session** is a live conversation with full brain state.

---

## The 20 brain components

Organized by priority tier. All components initialize when a session starts and learn across turns.

### P0 -- Core (always active)

| Component | Neuroscience analog | Purpose |
|---|---|---|
| **WeightEngine** | Synaptic weights | Bayesian posteriors for behavior, tools, and models. |
| **GoalTracker** | Prefrontal cortex | Track user goal progress, detect drift and loops. |
| **FeedbackEngine** | Reward circuitry | Process implicit/explicit signals across 4 tiers. |
| **PredictionEngine** | Predictive coding | Predict outcomes, detect surprise, drive learning. |
| **PlasticityManager** | Synaptic plasticity | Apply Hebbian, homeostatic, and metaplasticity rules. |
| **MemoryFabric** | Hippocampus | Working, episodic, and semantic memory tiers. |
| **DualProcessRouter** | System 1/2 | Route to fast (worker) or slow (orchestrator) path. |
| **ReputationSystem** | Social trust | Track tool reliability with quarantine on failure. |
| **AdaptationFilter** | Sensory adaptation | Habituate to repetitive signals, amplify novelty. |
| **PopulationQualityEstimator** | Population coding | Estimate response quality from multiple perspectives. |

### P1 -- Context and calibration

| Component | Neuroscience analog | Purpose |
|---|---|---|
| **CorticalContextEngine** | Working memory | Hot/warm/cold context tiers for 10,000+ step workflows. |
| **ProactivePredictionEngine** | Motor planning | Predict next user action, pre-warm resources. |
| **CrossModalAssociator** | Association cortex | Learn co-occurrence patterns across modalities. |
| **ContinuousCalibrationEngine** | Metacognition | Platt scaling, confidence adjustment, calibration health. |

### P2 -- Specialization and attention

| Component | Neuroscience analog | Purpose |
|---|---|---|
| **ColumnManager** | Cortical columns | Specialized processing units that compete for tasks. |
| **ResourceHomunculus** | Somatosensory cortex | Non-uniform resource allocation by task type. |
| **AttentionSystem** | Attentional filter | Subconscious processing, change detection, priority routing. |

### P3 -- Advanced plasticity

| Component | Neuroscience analog | Purpose |
|---|---|---|
| **ConceptGraphManager** | Distributed representation | Spreading activation across concept networks. |
| **CorticalMapReorganizer** | Cortical plasticity | Territory merging and redistribution for tools/models. |
| **TargetedModulator** | Optogenetics | Force-activate or silence specific tools/components. |
| **ComponentSimulator** | Digital twin | What-if analysis and A/B testing against live state. |

---

## Learn more

- [Architecture deep-dive](architecture.md) -- the 14-step `run()` pipeline in detail.
- [LLM Routing](llm-routing.md) -- how the thalamus selects models.
