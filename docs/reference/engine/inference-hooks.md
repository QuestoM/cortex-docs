# Inference Hooks (Layer 2)

Inference-time brain modifications for local models. These modules modify attention patterns during the forward pass without requiring additional training.

## inference_hooks

Neuroscience-inspired hooks that modify attention patterns and outputs during the forward pass.

::: corteX.engine.inference_hooks
    options:
      show_source: true
      members_order: source

---

## neuro_adapter

LoRA (Low-Rank Adaptation) framework for brain-inspired fine-tuning. Manages adapter weights, merging, and neuroscience-motivated training specifications.

::: corteX.engine.neuro_adapter
    options:
      show_source: true
      members_order: source

---

## training_collector

Training data collector that captures brain-state-driven LLM interactions for supervised fine-tuning (SFT), Direct Preference Optimization (DPO), and brain-conditioned training.

::: corteX.engine.training_collector
    options:
      show_source: true
      members_order: source

---

## early_exit

System 1/2 dual-process inference within a single local model. Exit classifiers at intermediate transformer layers enable confident early exits (System 1) while uncertain inputs traverse the full layer stack (System 2).

::: corteX.engine.early_exit
    options:
      show_source: true
      members_order: source

---

## Module Summary

### inference_hooks

| Class | Description |
|-------|------------|
| `HebbianConfig` | Configuration for HebbianAccumulator parameters |
| `HabituationConfig` | Configuration for AttentionHabituation parameters |
| `TemperatureConfig` | Configuration for AdaptiveHeadTemperature parameters |
| `VotingConfig` | Configuration for PopulationWeightedVoting parameters |
| `InferenceHookConfig` | Master configuration for the full pipeline |
| `HebbianAccumulator` | Short-term Hebbian co-activation tracker (Eliasmith et al., 2024) |
| `AttentionHabituation` | Stimulus-specific adaptation for repeated attention patterns (Ulanovsky et al., 2003) |
| `AdaptiveHeadTemperature` | Per-head temperature scaling based on attention entropy |
| `PopulationWeightedVoting` | Confidence-weighted aggregation of attention head outputs (Georgopoulos, 1986) |
| `InferenceHookPipeline` | Orchestrates all inference-time hooks in sequence |

### neuro_adapter

| Class | Description |
|-------|------------|
| `LoRAConfig` | Configuration for Low-Rank Adaptation parameters |
| `NeuroAdapterConfig` | Full training configuration for a neuroscience-inspired LoRA adapter |
| `LoRAWeight` | A single LoRA adapter weight: A (rank x in_features), B (out_features x rank) |
| `AdapterManager` | Manages multiple named LoRA adapters (load, save, merge, inspect) |
| `NeuroscienceAdapterSpec` | Defines neuroscience modifications trainable as LoRA adapters |

### training_collector

| Class | Description |
|-------|------------|
| `TrainingExample` | A single training example from a brain-state-driven LLM interaction |
| `TrainingCollector` | Buffered collector that writes training examples to JSONL files |
| `BrainStateSerializer` | Serializes brain component state for embedding in TrainingExample objects |
| `TrainingDataPipeline` | Formats collected examples for SFT, DPO, and brain-conditioned training |

### early_exit

| Class | Description |
|-------|------------|
| `EarlyExitConfig` | Configuration for early exit inference pipeline |
| `ExitClassifier` | Two-layer MLP classifier at a transformer exit point |
| `ConfidenceCalibrator` | Platt scaling for confidence calibration with online learning and ECE |
| `DualProcessInference` | Orchestrates System 1/2 inference within a single local model |
| `AdaptiveComputationController` | Dynamically adjusts confidence threshold for early exit |
| `create_dual_process_pipeline()` | Factory function to build a fully configured early exit pipeline |
